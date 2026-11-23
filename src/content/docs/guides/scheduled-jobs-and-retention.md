---
title: Scheduled jobs and retention
description: AdTo's two scheduled jobs, what each one runs, how the daily rollups are built, and exactly which rows and columns the daily retention sweep deletes, nulls or keeps.
sidebar:
  order: 4
---

AdTo runs three scheduled jobs, on UTC time:

| Expression | Fires | Runs |
|---|---|---|
| `* * * * *` | Every minute | Webhook retries only |
| `7 * * * *` | Every hour at :07 | Apple Search Ads retry, the daily rollups and webhook retries |
| `17 3 * * *` | Daily at 03:17 UTC | The same, then the expired temporary links' delete, the deleted-link anonymization, the webhook delivery-log purge, the exchange rates and, once the rollups are done, the retention sweep |

The code branches on the expression: `* * * * *` runs the webhook retries and returns; `7 * * * *` returns after the Apple Search Ads pass and the rollups; any other expression runs everything. The jobs run inside `waitUntil`, so a failure in one does not stop the others, except that the retention sweep waits for the rollup build. There is no job-level logging beyond console warnings.

## Apple Search Ads retry (both triggers)

`retryPendingAsaTokens` in `apps/worker/src/asa.ts`:

1. Selects up to 100 installs with `asa_status = 'pending'` and a non-null `asa_token`, oldest `asa_token_at` first (index `idx_installs_asa_pending`).
2. For each one, if the token arrived more than 24 hours ago (`ASA_TOKEN_TTL_MS`), the row is settled as `expired` without calling Apple. Apple answers 404 forever for such tokens.
3. Otherwise it makes exactly one `POST https://api-adservices.apple.com/api/v1/` with the token as a `text/plain` body and increments `asa_attempts`:
   - `200` with `attribution: true`: status `resolved`, Apple's record stored in `asa_attribution` (snake_case: `campaign_id`, `ad_group_id`, `keyword_id`, `ad_id`, `conversion_type`, `claim_type`, `click_date`, `impression_date`, `country_or_region`, `supply_placement`), and `app.install_updated` goes to the webhooks that take the install's app.
   - `200` with `attribution: false`: status `no_attribution`.
   - `400`: status `invalid`.
   - `404` (minted too recently): stays `pending`.
   - `5xx`, a network error or a malformed body: stays `pending`.
4. Whenever a status is settled, `asa_resolved_at` is set and `asa_token` is nulled: the token is a one-time credential and is never kept.

More than 100 pending tokens are spread over successive passes. The three inline attempts the install request itself makes, five seconds apart, are described on [Attribution internals](/start/attribution-internals/).

## Daily rollups (both triggers)

`buildRollups` (`apps/worker/src/rollups.ts`) fills `link_daily`: per account, link, UTC day and platform (`ios`, `android`, `desktop`, `other`), the clicks, first installs and the credited ones, app opens, leads, net revenue in the account's currency, and the events credited to a link because it re-opened the app. Installs and events no link earned sit under the slug `''`. The rows are rebuilt from the raw rows, a whole day at a time and never on the request path, so a tap costs no extra write.

Each account has a `rollup_state` row: `through_day`, the last built day, and `frozen_through` (below). A run takes the accounts furthest behind and builds up to 40 steps of 31 days across them, so an account with years of data backfills itself over the next runs; nothing needs to be run by hand. Reads take every day after `through_day` from the raw rows, so the dashboard is exact while the rollups catch up.

A change to a raw row of a built day moves `through_day` back to the day before it, and the next run rebuilds from there: a late claim that credits an older install, a store notification that changes an amount or reverses a refund, an exchange rate that arrives late, a new re-engagement window on an app (every unfrozen day), a deleted app, a new account currency, and a permanent link delete, which also removes the link's rollup rows.

After `link_daily`, `buildBreakdowns` (`apps/worker/src/breakdowns.ts`) fills `breakdown_daily` with the Overview's breakdowns per account, UTC day and value: clicks by country, referrer, campaign and source; installs by the campaign and source of their tap, device model, carrier, connectivity and match source; flagged installs by fraud rule. Without it, every Overview load read every raw click and install of its window for these lists. Its progress is its own, `breakdowns_through` on the same `rollup_state` row, so a deployment whose `link_daily` was built before it existed builds it over the next runs, and the Overview reads the raw rows until then. The same changes move it back. A value keeps its first 256 characters, built or read raw: a campaign or a source comes from the tapped URL. Its rows go at the retention cutoff with the raw rows, below. Unique visitors are not in it: a visitor on two days is one visitor, so that count still reads the raw taps.

What reads the rollups: the Overview's clicks, installs, reinstalls, opens, leads and revenue, its clicks per day and per device, for windows over 7 days (which are whole UTC days, today included), and the lifetime installs, reinstalls, opens, leads and revenue of every link on the Links screen.

## Webhook retries (every trigger)

`retryDueDeliveries` (`apps/worker/src/webhooks.ts`) sends again the pending deliveries whose `next_attempt_at` has passed and whose webhook is on. It runs every minute, and on the hourly and daily triggers too, so a deployment that runs only those still retries, at the next :07.

How a run works:
- **Order and caps:** oldest first, up to 20 per webhook and 300 in all. A receiver with a backlog cannot hold the others back.
- **Claiming:** each delivery is claimed before it is sent, by moving its `next_attempt_at` 2 minutes out. Two runs that overlap never send it twice.
- **Sending:** ten at a time, with a 10-second timeout each. New sends start only in the first 45 seconds of a run. A claimed delivery left unsent is due again 2 minutes later, and a larger backlog waits for the next run.
- **Schedule:** a failure schedules the next attempt 1 minute, 5 minutes, 30 minutes, 2 hours, 5 hours, 10 hours and 12 hours after the previous one, then daily, 13 attempts in all. Then the delivery is marked failed.

The Node server logs a run of the minute trigger only when it fails. See [Webhooks](/api/webhooks/#delivery-and-retries).

## Device names (daily)

`refreshDeviceModels` (`apps/worker/src/devicemodels.ts`) downloads Google Play's supported-devices list (`storage.googleapis.com/play_public/supported_devices.csv`, about 5 MB) and stores the marketing names of the device models your installs reported in the last 400 days, never the whole list, in `device_models`. It runs when the last refresh is a week old, or a day old while installs report models the table lacks. A download that fails, is not the list or has fewer than 10,000 rows changes nothing. The Overview's Top devices joins the names at read time.

## Webhook delivery log (daily)

```sql
DELETE FROM webhook_deliveries WHERE created_at < <now minus 30 days>
```

## Expired temporary links (daily)

```sql
UPDATE temp_links SET deleted_at = <now>
WHERE expires_at < <now minus 24 hours> AND expired_url IS NULL AND deleted_at IS NULL
```

A temporary link therefore keeps answering for up to a day after it expires: the redirect shows the 410 page, `GET /api/v1/resolve/:user/:slug` answers `410 link_expired`, and `GET /api/v1/temp-links/:slug` still works. A link with an expired URL is never deleted this way: it keeps sending taps there until its owner deletes it. After the delete both answer 404 and the link leaves the lists (`?include_deleted=1` lists it). It is a soft delete, like the dashboard's **Delete**: the link's clicks, installs and events stay, and its slug stays reserved (`409` with `deleted: true`) until the link is restored or deleted permanently, which takes its history too, so a new link never inherits it. Its taps lose their device data 30 days later, as for every [deleted link](#deleted-links-daily). The dashboard's **Delete expired** button and `DELETE /api/admin/temp-links/expired` do the same at once for every expired link. See [Temporary links](/links/temporary-links/).

## Exchange rates (daily)

Downloads the day's euro rates into `fx_rates` (`date`, `currency`, `rate`, plus `EUR` = 1), each under its source's date: the ECB's reference rates, and the [exchange-api](https://github.com/fawazahmed0/exchange-api) feed's for the currencies the ECB does not publish, from the feed's `currency-api.pages.dev` address or, when that fails, from jsDelivr, whose copy can be a day behind. The feed never replaces an ECB rate. Then it fills `events.revenue_base` for up to 2,000 amounts that have none and a rate can convert: the amount in the account's revenue currency, converted with the rate of the event's day (the last rate published on or before it, else the first after it). A failed download is logged and retried the next day, and the other source's rates are stored regardless; amounts keep waiting with `revenue_base` null and count as 0 in totals. An amount in a currency neither source publishes waits the same way, without holding up the others. The same download also runs when an owner picks a revenue currency no rate is stored for yet, so the totals do not read 0 until the next run.

## Anonymization sweep (daily)

The cutoff is now minus 400 days. An account with a shorter window of its own (Settings → **Data retention**, 1 to 400 days; `PUT /api/admin/settings/retention`) is then swept again with its own cutoff, in the same order; a longer one has no effect.

Two tables can keep less than that, because they are most of what a busy deployment stores and nothing reads them back as far: `app_sessions` (the screen views) for `SESSION_RETENTION_DAYS` and `device_days` (one row per device per day it started the app) for `DEVICE_DAY_RETENTION_DAYS`, never under 30 days, the window monthly active users are counted over. Each applies when it is shorter than the window being swept; unset, the table follows the window. The hosted service keeps sessions 90 days and daily app starts 180. The screens a link led to and the retention read-out cover at most what their table keeps, and say which window they used.

First, for every account whose rollups are built through the cutoff's day, the clicks, installs, events and app opens older than the cutoff are deleted, except events with a `transaction_id` (purchases, renewals, refunds), which stay and are nulled as below, so a restored purchase still counts once and a refund finds its purchase. `frozen_through` becomes the cutoff's day: those days' totals now live only in `link_daily` and are never rebuilt (a later change of account currency converts them with each day's exchange rates instead). This is what keeps storage at about the retention window's worth of rows.

Then `anonymiseExpiredDeviceData` runs one `db.batch`, so every statement commits or none does. Each account's pass is on its own: one that fails is logged (`[adto] retention: sweep failed`) and runs again the next day, and the others, and the purges in the table, still run. The sweep also runs when the day's rollup build failed, since it deletes only the days the rollups hold. Rows of accounts whose rollups are not that far yet are kept for aggregate analytics with their personal columns set to `NULL`, as below.

### What is deleted, nulled and kept

| Table | Action | Condition | Columns affected |
|---|---|---|---|
| `ad_identifiers` | DELETE rows | `created_at < cutoff` | whole row |
| `customer_installs` | DELETE rows | `created_at < cutoff` | whole row |
| `breakdown_daily` | DELETE rows | `day` before the cutoff's day, for every account | whole row |
| `device_days` | DELETE rows | `day` before the cutoff's day, or `DEVICE_DAY_RETENTION_DAYS` ago when shorter | whole row |
| `app_sessions` | DELETE rows | `started_at < cutoff`, or `SESSION_RETENTION_DAYS` ago when shorter | whole row |
| `rejected_requests` | DELETE rows | `created_at` older than 30 days, whatever the retention setting | whole row |
| `inspector_entries` | DELETE rows | `created_at` older than `INSPECTOR_RETENTION_DAYS` (7 by default), every hour, 2,000 rows at a time | whole row |
| `events` | SET NULL | `created_at < cutoff` | `device_id`, `customer_id`, `metadata`, `ip_raw` |
| `customers` | SET NULL | `updated_at < cutoff` (last identify, not creation) | `name`, `email`, `phone`, `first_install_id` |
| `app_installs` | SET NULL | `created_at < cutoff` | `device_id`, `device_model`, `device_manufacturer`, `device_kind`, `cpu_type`, `os_build`, `screen_width`, `screen_height`, `screen_density`, `text_size`, `dark_mode`, `high_contrast`, `reduce_motion`, `utc_offset_min`, `connectivity_type`, `carrier_mcc`, `carrier_mnc`, `ip_raw`, `ip_hash`, `web_user_agent`, `meta_referrer`, `asa_token` |
| `app_opens` | SET NULL | `created_at < cutoff` | `device_id`, `install_id`, `click_id`, `ip_hash` |
| `clicks` | SET NULL | `clicked_at < cutoff` | `device_id`, `device_model`, `ip_raw`, `ip_hash`, `ua_raw`, `accept_language`, `region`, `city`, `asn`, `as_org`, `launch_referrer` |
| `clicks` (screen profile and device settings) | SET NULL | `clicked_at` older than 30 days, only rows that still carry a value | `screen_width`, `screen_height`, `pixel_ratio`, `max_touch_points`, `text_size`, `dark_mode`, `high_contrast`, `reduce_motion`, `utc_offset_min` |
| `rate_limits` | DELETE rows | `window_start` older than the current 60 s window minus 2 | whole row |
| `erasures` | DELETE rows | `expires_at` passed: 8 days after a device or customer was erased | whole row |
| `webhook_deliveries` | DELETE rows | an account whose own window is under 30 days: older than its cutoff (otherwise the 30-day job below) | whole row |

The sweep also deletes what an erasure missed: for every tombstone still standing, an event of the erased device or customer stored in the minute after the erasure (the life of an edge location's copy of the account, `SDK_AUTH_CACHE_SECONDS`) with a client time from before it; see [Privacy and data](/guides/privacy-and-data/#a-request-from-one-person).

The screen profile and the device settings are dropped early because the fingerprint reads them only inside its window, which never exceeds 30 days (720 hours). `enriched_at` is kept on purpose: it is the "first beacon wins" gate, and a null would let a late beacon rewrite the row.

### Deleted links (daily)

`anonymiseDeletedLinks` gives the clicks and app opens of a link deleted more than 30 days ago (`links.deleted_at` or `temp_links.deleted_at`) the same treatment as above, whatever their age: the `clicks` personal columns, screen profile and device settings, and the `app_opens` columns, are set to `NULL`, only on rows that still carry a value. Thirty days is the cap of the attribution window, so an install from a tap made before the delete can still match its click until then. The link's installs and events belong to your app's users rather than to the link, and keep the retention schedule. A deleted link's counts stay; a permanent delete removes the rows instead.

### What stays after the cutoff

- **Clicks:** `id`, `user_id`, `username`, `link_slug`, `link_kind`, `source`, `device_type`, `os`, `os_major`, `browser`, `country`, `session_id`, `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`, `metadata`, `clicked_at`, `referrer` (origin only), `enriched_at`, `lang`, `os_minor`, `connection_type`, `bot_score`, `link_params`, `client_click_time`, `origin`.
- **Installs:** everything not listed above, including `install_referrer` (which contains `adto_click=<id>` and `adto_l=<username>/<slug>`), `metadata`, `asn`, `country`, `timezone`, `language`, `lang`, `os`, `os_version`, `os_major`, `os_minor`, `cpu_family`, `clipboard_slug`, `package_name`, `installer_package`, `is_physical_device`, the install and referrer timestamps, `fraud_flags`, the whole attribution verdict (`matched`, `matched_slug`, `matched_click_id`, `match_source`, `referrer_source`, `match_confidence`, `match_score`, `candidate_count`, `routing_click_id`, `fp_shadow_*`), `install_attempt_id`, `created_at_client`, `sent_at_client` and `asa_attribution`.
- **Events:** `type`, `event_name`, `revenue`, `currency`, `install_id`, `matched_slug`, `client_timestamp`, `created_at`.
- **Customers:** `customer_id` (the id your app chose), `created_at`, `updated_at`.
- **App opens:** `username`, `slug`, `app_version`, `os`, `created_at`.

### Tables the sweep never touches

`users`, `apps`, `links`, `temp_links` (the expiry delete only sets `deleted_at`), `audit_events` (kept as long as the account), `blog_categories`, `blog_posts`, `contact_messages` and `password_resets`. Contact-form messages and used or expired reset tokens (hashes only) therefore accumulate until someone deletes them: owners can delete messages in the dashboard.

### Limits

- An account can shorten its window, never lengthen it past 400 days.
- Past the cutoff, raw rows are deleted once their day is in the rollups, and only anonymized until then. Within the retention window, a tenant can delete a link permanently, or erase one device or customer (see [Privacy and data](/guides/privacy-and-data/#a-request-from-one-person)); a plain delete keeps the rows.
- The sweep runs once a day at 03:17 UTC. Data older than the cutoff can exist for up to 24 hours before it is anonymized.
