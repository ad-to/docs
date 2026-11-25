---
title: Links
description: Create, read, update and delete permanent and temporary links over the REST API, and resolve any link without an API key.
sidebar:
  order: 2
---

A link is a short URL on your account's host, `https://<username>.adto.app/<slug>`. Permanent links live until you delete them or give them an expiry, and carry routing, social-preview and tracking settings. Temporary links expire after a TTL and carry only a target and a social preview. Both kinds can be paused, archived, deleted and restored ([Pause, expire, archive and delete](#pause-expire-archive-and-delete)).

Every route on this page except `resolve` needs the Bearer key and the request signature described in [API overview](/api/overview/). All routes are scoped to the account that owns the key: you can only see and change your own links.

## The Link object

Every permanent-link route returns the full object, including fields you never set (they come back as `null`).

```json title="Link"
{
  "id": "6f1c2a0e-3b7d-4c8e-9a11-2f5d8c0b4e77",
  "slug": "spring-sale",
  "name": "Instagram bio, spring",
  "group_ids": ["5a7e0c12-…"],
  "target": "https://shop.example.com/sale",
  "url": "https://acme.adto.app/spring-sale",
  "app_id": "0d4e5f66-7a8b-4c9d-8e0f-1a2b3c4d5e6f",
  "click_count": 0,
  "created_at": "2026-10-12T09:00:00.000Z",
  "updated_at": null,
  "metadata": { "screen": "sale", "campaign": "spring" },
  "og_title": null,
  "og_description": null,
  "og_image_url": null,
  "ios_deep_link_path": "sale",
  "android_deep_link_path": "sale",
  "redirect_url": null,
  "fallback_url": null,
  "label": null,
  "campaign": "spring",
  "adgroup": null,
  "creative": null,
  "ios_tap_to_store": null,
  "mobile_interstitial": null,
  "desktop_qr": null,
  "append_click_id": null,
  "forward_query": null,
  "routing_rules": null,
  "utm_source": null,
  "utm_medium": null,
  "utm_campaign": null,
  "utm_term": null,
  "utm_content": null,
  "external_id": null,
  "referrer_customer_id": null,
  "expires_at": null,
  "expired_url": null,
  "disabled_at": null,
  "disabled_reason": null,
  "archived_at": null,
  "deleted_at": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | string | UUID, assigned by the server. |
| `slug` | string | The path segment of the link. |
| `name` | string or null | Your own name for the link, so a generated slug stays recognisable. Never used in routing or attribution. |
| `group_ids` | array of strings | The [link groups](#link-groups) the link is in, in group-name order; `[]` when none. |
| `target` | string or null | The web destination. `null` means "inherit the app's fallback URL at tap time". |
| `url` | string | The public link URL, `https://<username>.adto.app/<slug>`. |
| `app_id` | string or null | The app whose store URLs, schemes and fallback apply to this link. `null` means the account's default app (its oldest app). |
| `click_count` | number | Lifetime counter. It grows on web taps, on SDK-reported clicks and on clicks synthesized from a store referrer. It never goes down. |
| `created_at`, `updated_at` | string or null | ISO 8601 UTC. `updated_at` stays `null` until the first `PUT`. |
| `metadata` | JSON or null | Your own data. Delivered to the app on attributed installs as `custom_params` and inside `link.metadata`. |
| `og_title`, `og_description`, `og_image_url` | string or null | Social preview overrides for bots and unfurlers. Fall back to the app's values. See [Social previews and QR](/links/social-previews-and-qr/). |
| `ios_deep_link_path`, `android_deep_link_path` | string or null | The app route a custom-scheme open uses on that platform: `scheme://<path>?adto_l=<username>/<slug>` instead of `scheme://<username>/<slug>?adto_l=…`. Stored without a leading slash. |
| `redirect_url` | string or null | Mobile visitors without the app go here instead of the store. A store URL here is still tagged with the click id. |
| `fallback_url` | string or null | Where desktop visitors go instead of the QR page, and the last mobile fallback. |
| `label`, `campaign`, `adgroup`, `creative` | string or null | Tracking defaults copied onto every click of this link. A `adto_<field>` query parameter on the tapped URL wins for that tap. See [URL parameters](/links/url-parameters/). |
| `ios_tap_to_store`, `mobile_interstitial` | boolean or null | This link's iOS tap-to-store page and mobile interstitial. `null` follows the app's setting, and on the hosted service still shows the tap-to-store page to iCloud Private Relay taps when the app's switch is off; `ios_tap_to_store: false` keeps it from those taps too. See [How a tap is routed](/links/how-a-tap-is-routed/#mobile-delivering-it). |
| `desktop_qr` | boolean or null | What desktop visitors get: `true` the QR page (even with a fallback URL, which becomes the page's link on to the website), `false` the fallback URL first. `null` follows the app's setting. See [How a tap is routed](/links/how-a-tap-is-routed/#desktop-and-the-shared-tail). |
| `append_click_id` | boolean or null | Whether the link adds `adto_click` to its web destinations, for [`adto.js`](/sdk/web/). `null` is off. |
| `forward_query` | boolean or null | Whether the link copies the tapped URL's own query parameters onto its web destinations. See [Your own parameters](/links/url-parameters/#your-own-parameters). `null` is off. |
| `routing_rules` | array or null | Geo and language rules, first match wins. See [Geo and language rules](#geo-and-language-rules). |
| `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content` | string or null | Added to the link's web destinations and stored on clicks whose URL has none. See [Link-level UTM values](/links/url-parameters/#link-level-utm-values). |
| `external_id` | string or null | Your own id for the link (a product or CMS row id), unique in the account. See [Upsert and external ids](#upsert-and-external-ids). |
| `referrer_customer_id` | string or null | Set on a customer's referral link. See [Referral programs](/guides/referral-programs/). |
| `created_via` | string or null | Where the link was made, read-only. The values are `dashboard` (a signed-in session), `api` (an account key), `sdk` (an SDK key) and `mcp` (the MCP server). Links made before this field took it from the account's audit log. `null` means the audit log no longer holds the link's creation. |
| `expires_at`, `expired_url` | string or null | When the link expires, and where taps go after that instead of a `410` page. |
| `disabled_at`, `disabled_reason` | string or null | Paused since, and your note on why. A paused link answers `410` and records nothing. |
| `archived_at` | string or null | Filed away in the dashboard. The link keeps working. |
| `deleted_at` | string or null | Deleted. Only listed with `include_deleted`. |

## Slug rules

- Allowed characters: `A-Z`, `a-z`, `0-9`, `_` and `-`. Length 1 to 128.
- Optional when you create a link, permanent or temporary: absent, `null` or `""`, the server generates one, 5 random letters and digits such as `Xk3pQ`. It is drawn again while your account holds it (live, deleted or temporary), and again if another create stores it first, instead of answering `409`; should 5-character draws keep colliding, it is 6 characters long. A generated slug is short, not secret: anyone can try 5-character slugs, so a link that must stay private needs a long random slug of your own.
- Reserved: `healthz`, `readyz`, `bulk` and `upsert` (any case), and any slug starting with `ext_`, which names a link by its external id.
- The slug is used exactly as sent. Leading or trailing whitespace makes it invalid. Slugs are case-sensitive: `Sale` and `sale` are two different links. A generated slug mixes both cases, so share it as a link or QR code rather than reading it out.
- One namespace per account for permanent **and** temporary links. Creating either kind with a slug the other kind holds returns `409 {"error": {"code": "conflict", "message": "Slug already exists", "slug": "<slug>"}}`.
- A slug becomes free again only when the link is deleted permanently. The daily job's delete of an expired temporary link is a soft delete, so it keeps its slug. A deleted link keeps its slug, because its history is filed under it: creating that slug answers `409 {"error": {"code": "conflict", "message": "Slug belongs to a deleted link: restore it, or delete it permanently to reuse the slug", "slug": "<slug>", "deleted": true}}`.

## Create a link

`POST /api/v1/links` returns `201` with the Link.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `slug` | string | no | See the slug rules above. Absent, `null` or `""`: a generated slug, 5 random letters and digits. |
| `name` | string | no | Trimmed; at most 200 characters; no control characters. `""` stores `null`. |
| `group_ids` | array of strings | no | Up to 50 of your [link group](#link-groups) ids. An id that is not one of yours answers `400 Unknown group_id: …`. On update it replaces the link's groups; `[]` or `null` takes the link out of every group, and leaving it out keeps them. |
| `target` | string or null | no | An http(s) URL. `null`, `""` or absent stores `null` (inherit the app's fallback URL). |
| `app_id` | string | no | The id of one of your apps. Default: your oldest app, or `null` when the account has no app yet. |
| `metadata` | JSON object | no | Your own key-value data. `null` or `""` stores `null`; any other non-object value (a string, number, boolean or array) answers `400 {"error": {"code": "invalid_request", "message": "metadata must be a JSON object"}}`. |
| `og_title`, `og_description` | string | no | Trimmed; at most 150 characters each. `""` stores `null`. |
| `og_image_url` | string | no | An http(s) URL. `""` stores `null`. |
| `ios_deep_link_path`, `android_deep_link_path` | string | no | A relative route such as `products/42?color=black`. At most 512 characters; no `:`, whitespace, `#`, `;` or control characters. A leading `/` is stripped. `""` stores `null`. |
| `redirect_url`, `fallback_url` | string | no | http(s) URLs. `""` stores `null`. |
| `label`, `campaign`, `adgroup`, `creative` | string | no | Trimmed; at most 255 characters; no control characters. `""` stores `null`. |
| `ios_tap_to_store`, `mobile_interstitial` | boolean or null | no | `true` or `false` overrides the app's setting for this link; `null` or absent follows it. Anything else answers `400 Invalid ios_tap_to_store: must be true, false or null (follow the app)`. |
| `desktop_qr` | boolean or null | no | `true` or `false` overrides the app's desktop setting for this link; `null` or absent follows it. Anything else answers `400 Invalid desktop_qr: must be true, false or null (follow the app)`. |
| `append_click_id` | boolean or null | no | `true` adds `adto_click=<click id>` to the link's web destinations (never stores, MMP links or schemes); `false` or `null` does not. Anything else answers `400 Invalid append_click_id: must be true, false or null`. |
| `forward_query` | boolean or null | no | `true` copies the tapped URL's own parameters (not `adto_*` or AdTo's markers; `utm_*` included) onto the link's web destinations, over the destination's own value and the link's `utm_*`. It never copies them onto stores or schemes. `false` or `null` does not. Anything else answers `400 Invalid forward_query: must be true, false or null`. |
| `routing_rules` | array or null | no | Up to 20 rules; `null` or `[]` stores none. See below. |
| `template_id` | | | No longer accepted: link templates were removed. A non-empty value answers `400 Link templates were removed: set the fields on the link itself`; `null` and `""` are ignored. |
| `utm_source` … `utm_content` | string | no | Trimmed; at most 255 characters; no control characters. `""` stores `null`. |
| `external_id` | string | no | 1 to 255 characters, no control characters. Taken by another link: `409 {"error": {"code": "conflict", "message": "external_id belongs to another link", …}}`. |
| `expires_at` | string | no | An ISO 8601 time such as `2026-12-31T23:59:00Z`, stored in UTC. A time in the past creates an already-expired link. |
| `expired_url` | string | no | An http(s) URL taps go to once the link has expired. |
| `disabled`, `disabled_reason` | boolean, string | no | `true` creates the link paused. The reason is at most 255 characters. |
| `archived` | boolean | no | `true` creates the link archived. |

Checks run in this order and stop at the first failure:

| Status | Body |
|---|---|
| 400 | `{"error": {"code": "invalid_request", "message": "Invalid JSON body"}}` |
| 400 | `{"error": {"code": "invalid_request", "message": "Invalid slug: only letters, numbers, hyphens and underscores are allowed"}}` |
| 400 | `{"error": {"code": "invalid_request", "message": "Slug 'healthz' is reserved"}}`: `healthz` and `readyz` (any case) are answered by the service itself on every host, so they are refused for permanent and temporary links |
| 400 | `{"error": {"code": "invalid_request", "message": "Invalid target: must be a valid http(s) URL"}}` |
| 400 | `{"error": {"code": "invalid_request", "message": "disabled must be true or false"}}`, `"archived must be true or false"`, `"expires_at must be an ISO 8601 time, e.g. 2026-12-31T23:59:00Z"`, `"expired_url must be a valid http(s) URL"`, `"disabled_reason: max 255 characters"` |
| 409 | `{"error": {"code": "conflict", "message": "Slug already exists", "slug": "spring-sale"}}`, or the deleted-link variant above |
| 400 | `{"error": {"code": "invalid_request", "message": "Unknown app_id"}}` (the id is not one of your apps) |
| 400 | `{"error": {"code": "invalid_request", "message": "Invalid label: max 255 characters"}}`, `{"error": {"code": "invalid_request", "message": "Invalid campaign: control characters are not allowed"}}`, `{"error": {"code": "invalid_request", "message": "Invalid adgroup: must be a string"}}` (checked in the order `label`, `campaign`, `adgroup`, `creative`) |
| 400 | `{"error": {"code": "invalid_request", "message": "Invalid ios_deep_link_path: must be a relative path, not a full URL (no \":\": no scheme, no spaces, no \"#\"/\";\")"}}`, `{"error": {"code": "invalid_request", "message": "Invalid android_deep_link_path: deep link path is too long (max 512 chars)"}}` |
| 400 | `{"error": {"code": "invalid_request", "message": "Invalid redirect_url: must be a valid http(s) URL"}}`, same for `fallback_url` |

The server also rejects `og_title` or `og_description` longer than 150 characters and an `og_image_url` that is not an http(s) URL, with the `400` strings listed under [Update a link](#update-a-link).

```bash title="Create a link"
adto POST /api/v1/links '{
  "slug": "spring-sale",
  "target": "https://shop.example.com/sale",
  "metadata": { "screen": "sale" },
  "ios_deep_link_path": "sale",
  "android_deep_link_path": "sale",
  "campaign": "spring"
}'
```

`adto` is the signing helper from [API overview](/api/overview/).

## Create many links

`POST /api/v1/links/bulk` with `{"links": [ … ]}` creates up to 100 links, each with the create fields above. Every link succeeds or fails on its own, and the call answers `200`:

```json title="POST /api/v1/links/bulk"
{
  "created": [ { "slug": "spring-1", "url": "https://acme.adto.app/spring-1", … } ],
  "errors": [ { "index": 1, "slug": "spring-1", "code": "conflict", "error": "Duplicate slug or external_id in this batch" } ]
}
```

`created` holds full Link objects; `errors` names each failed item by its position in `links`, with the same `code` and message a single create would give. A slug or external id that appears twice in the batch fails from its second appearance. Items without a slug get generated ones, drawn for the batch together: they never repeat each other or a slug another item names. An empty array, more than 100 links, or a body without `links` answers `400`. The dashboard's **Import CSV** and the MCP `create_links` tool use this route.

## Upsert and external ids

A link created with `external_id` can be named by it anywhere a v1 link route takes a slug: `GET`, `PUT` and `DELETE /api/v1/links/ext_<id>` and `POST /api/v1/links/ext_<id>/restore`, so a system that knows its own ids never has to store slugs.

`PUT /api/v1/links/upsert` makes a sync idempotent. It finds the link by `external_id` when the body has one, else by `slug`:

| Found | Result |
|---|---|
| A live link | Updated with the `PUT` rules (absent keeps, `null` or `""` clears); `200` with the Link. A `slug` in the body that differs from the found link's answers `400`: a link's slug never changes. |
| A deleted link | `409` with `"deleted": true`: restore it first |
| Nothing | Created with the create rules; `201` with the Link. Named by `external_id` alone, it gets a generated slug, which later calls find it by. |

A body with neither `external_id` nor `slug` answers `400 {"error": {"code": "invalid_request", "message": "slug or external_id is required"}}`: it would create another link on every call.

```bash title="Sync one product's link"
adto PUT /api/v1/links/upsert '{"external_id": "product-42", "slug": "p42", "target": "https://shop.example.com/p/42", "campaign": "catalog"}'
```

## Link groups

A group is a named set of your permanent links: a campaign, a partner, a channel. A link can be in any number of groups; set them with the link's `group_ids`. Groups never change how a tap is routed or credited.

| Route | Response |
|---|---|
| `GET /api/v1/link-groups` | `200 {"data": [Group, …], "next_cursor": null}`, by name |
| `POST /api/v1/link-groups` | `{name}` → `201` Group; a name already in use answers `409` |
| `GET /api/v1/link-groups/:id` | `200` Group or `404 Group not found` |
| `PUT /api/v1/link-groups/:id` | `{name}` → `200` Group, renamed |
| `DELETE /api/v1/link-groups/:id` | `200 {"success": true}`; its links stay, in their other groups |

```json title="Group"
{
  "id": "5a7e0c12-…",
  "name": "Influencers",
  "link_count": 8,
  "created_at": "2026-11-23T09:00:00.000Z",
  "updated_at": null
}
```

`name` is 1 to 100 characters. `link_count` counts the group's links that are not deleted. Groups use the links scopes, reading needs `links:read` and changing `links:write`, and need an account key: names and groups are your private notes, so an SDK key (`adto_pub_…`, which ships inside your app) gets `name: null` and `group_ids: []` on every link. Filtering with `group` or reading a group answers `403 account_key_required`. An SDK key's changes are checked for scope first: without `links:write`, changing a group answers `403 insufficient_scope`, as any `/api/v1/links` change does. With `links:write`, changing a group, or sending either field to a create, change or upsert, answers `403 account_key_required`; in a bulk create only the links that send them fail, each with an `errors` entry whose `code` is `account_key_required`, and the call still answers `200`. A deleted link keeps its groups for a restore; a link deleted permanently leaves them.

## List and read links

| Route | Response |
|---|---|
| `GET /api/v1/links?limit=&cursor=&include_deleted=&group=` | `200 {"data": [Link, …], "next_cursor": "…" \| null}`, newest first. `group` (a group id) lists only that group's links. `limit` is 1 to 200 (default 50); pass `next_cursor` back as `cursor` for the next page until it is `null`. A `limit` out of range, or a `cursor` this API did not return, answers `400`. Paused, expired and archived links are listed; deleted ones only with `include_deleted=true`. |
| `GET /api/v1/links/:slug` | `200` with the Link, or `404 {"error": {"code": "not_found", "message": "Link not found"}}`. A temporary link's slug and a deleted link also answer 404 here. |

## Update a link

`PUT /api/v1/links/:slug` applies a partial update and returns `200` with the full Link. Unknown slugs and deleted links return `404 {"error": {"code": "not_found", "message": "Link not found"}}`. The lifecycle fields of the create body work here too; see the next section.

| In the body | Effect |
|---|---|
| Key absent | The stored value is kept. This includes the `og_*` fields. |
| `null` | Clears the field: `target`, `metadata`, `og_*`, the deep-link paths, `redirect_url`, `fallback_url` and the four tracking fields all become `null`. |
| `""` | Clears the field, `metadata` included. |
| `slug`, `app_id` | Ignored. Neither can be changed after creation. |

Validation is the same as on create. Errors come back as `400` with the same strings, plus `{"error": {"code": "invalid_request", "message": "og_title must be at most 150 characters"}}`, `{"error": {"code": "invalid_request", "message": "og_description must be at most 150 characters"}}` and `{"error": {"code": "invalid_request", "message": "og_image_url must be a valid http(s) URL"}}`.

```bash title="Clear the target so the link inherits the app fallback"
adto PUT /api/v1/links/spring-sale '{"target": null, "label": "launch-week"}'
```

## Geo and language rules

A rule replaces some of the link's fields for taps from given countries or browser languages. Rules are tried in order and the first match wins; a tap no rule matches is routed as usual, and the URL's `adto_*` parameters still win over a rule. See [How a tap is routed](/links/how-a-tap-is-routed/#geo-and-language-routing).

| Rule field | Type | Rules |
|---|---|---|
| `name` | string | Up to 64 letters, digits, spaces and `. _ : -`; recorded on each click the rule routes (`routing_rule`), else `rule N` |
| `countries` | array of strings | ISO 3166 two-letter codes, any case, stored upper case |
| `languages` | array of strings | Language tags; only the primary tag is kept, lower case (`de-AT` becomes `de`) |
| `percent` | integer | An A/B share of visitors, 1 to 100. Shares stack in rule order; together at most 100. A visitor's share follows their hashed IP. |
| `starts_at`, `ends_at` | string | ISO 8601 times bounding the rule (scheduled routing); `ends_at` after `starts_at` |
| `target`, `redirect_url`, `fallback_url` | string | http(s) URLs that replace the link's own for a matching tap |
| `ios_deep_link_path`, `android_deep_link_path` | string | In-app routes, with the same rules as the link's |

A rule needs a condition (`countries`, `languages`, `percent`, `starts_at` or `ends_at`; a tap must meet every one a rule has) and at least one of the destination fields. Unknown fields, more than 20 rules, shares over 100, or a bad value answer `400`, for example `{"error": {"code": "invalid_request", "message": "routing_rules[0] needs a condition: countries, languages, percent, starts_at or ends_at"}}`.

```bash title="German-speaking visitors get the German sale page and in-app route"
adto PUT /api/v1/links/spring-sale '{"routing_rules": [
  {"countries": ["DE", "AT", "CH"], "fallback_url": "https://shop.example.de/sale", "ios_deep_link_path": "de/sale", "android_deep_link_path": "de/sale"},
  {"languages": ["fr"], "fallback_url": "https://shop.example.fr/soldes"}
]}'
```

```bash title="Half the visitors see the new landing page; compare by routing_rule"
adto PUT /api/v1/links/spring-sale '{"routing_rules": [
  {"name": "new-landing", "percent": 50, "fallback_url": "https://shop.example.com/new", "redirect_url": "https://shop.example.com/new-m"}
]}'
```

## Pause, expire, archive and delete

A link that is not live records nothing: no click row, no click count, for people and bots alike. The SDK's click and app-open reports for it are refused the same way, after the app has already opened.

| State | Set with | A tap gets | `resolve`, SDK clicks and opens |
|---|---|---|---|
| Paused | `PUT {"disabled": true, "disabled_reason": "…"}`; `{"disabled": false}` resumes and clears the reason | A `410` "Link paused" page. The reason is never shown. | `410 {"error": {"code": "link_paused", …}}` |
| Expired | `expires_at` in the past, on create or `PUT`; `null` or `""` removes it | A `302` to `expired_url`, or a `410` "Link expired" page without one | `410 {"error": {"code": "link_expired", "expired_url": "…" \| null, …}}` |
| Archived | `PUT {"archived": true}`; `false` unarchives | The usual routing: archiving only files the link away in the dashboard | As a live link |
| Deleted | `DELETE /api/v1/links/:slug` | A `404` page, as for a slug that never existed | `404 not_found` |

Pausing a paused link keeps its first `disabled_at`. A paused link that has also expired answers as paused.

```bash title="End a sale at midnight and send late taps to a page"
adto PUT /api/v1/links/spring-sale '{"expires_at": "2026-12-31T23:59:00Z", "expired_url": "https://shop.example.com/sale-over"}'
```

### Delete, restore, delete permanently

`DELETE /api/v1/links/:slug` returns `200 {"success": true}`. Unknown slugs, and links already deleted, return `404 {"error": {"code": "not_found", "message": "Link not found"}}`.

Deleting is a soft delete. The link leaves the list (unless `include_deleted=true`), its URL answers 404, and `GET` and `PUT` answer 404. Its clicks and the installs, events and app opens attributed to it stay, so the Overview and every past verdict keep their numbers. The link keeps its slug. Thirty days after the delete, the daily job anonymises its clicks and app opens (IP hash, device id, user agent and the like), as retention would do after `RETENTION_DAYS`; installs and events keep the normal schedule.

`POST /api/v1/links/:slug/restore` brings a deleted link back and returns `200` with the Link; restoring a live link changes nothing. An unknown slug answers 404.

### Clone

`POST /api/v1/links/:slug/clone`, with an optional body `{"slug": "summer-sale"}`, makes a new permanent link from an existing one. Left out, the slug is generated. Answers:
- **`201` with the copy.** Its `created_via` is where the clone was made.
- **`400`** when the body is not a JSON object (an empty body is fine).
- **`404`** when the source is unknown or deleted.
- **`409`** when the slug is taken.

The copy is checked like a create.

**Copied:** the target, the app (exactly as the source has it, so a link with none gets none), metadata, social preview, deep-link paths, redirect and fallback URLs, label, campaign, adgroup and creative, UTMs, the page settings, `append_click_id`, `forward_query`, the routing rules and the link groups.

**Not copied:**
- the name;
- the history and counters;
- `external_id`, which is unique;
- the expiry, the pause and the archive;
- a referral link's customer.

The dashboard's **Clone** on a link's row and the MCP `clone_link` tool use the same route.

`DELETE /api/v1/links/:slug?permanent=true` deletes the link and its history in one atomic batch, on a live or a deleted link: the events of installs it attributed and events denormalized to its slug, its app opens and referrals, the installs attributed through its clicks (customers that pointed at those installs keep their identity but lose `first_install_id`), its clicks, then the link. The dashboard's totals drop accordingly, and the slug is free again. This cannot be undone; see [Dashboard: Links](/dashboard/links/). Only an owner deletes permanently: `?permanent=true` needs the account key: an SDK key is refused with `403 account_key_required`, since team members can read SDK keys in Settings. An SDK key still soft deletes.

:::note
The dashboard's cookie-session route `DELETE /api/admin/links/:slug` (with `?permanent=1` for the hard delete, which answers `403 insufficient_role` to a member or viewer) answers `{"deleted": true}` even for a slug that does not exist. Only the v1 route returns 404.
:::

## Temporary links

Temporary links expire after a TTL. They have no app binding, no deep-link paths, no tracking defaults and no click counter; taps use the account's default app for store routing and fallback. See [Temporary links](/links/temporary-links/) for how they behave when tapped.

```json title="TempLink"
{
  "id": "9a7b5c3d-1e2f-4a6b-8c9d-0e1f2a3b4c5d",
  "slug": "Xk3pQ",
  "target": "https://example.com/build/1234",
  "url": "https://acme.adto.app/Xk3pQ",
  "expires_at": "2026-10-12T10:00:00.000Z",
  "created_at": "2026-10-12T09:00:00.000Z",
  "ttl_seconds": 3600,
  "og_title": null,
  "og_description": null,
  "og_image_url": null,
  "expired_url": null,
  "disabled_at": null,
  "disabled_reason": null,
  "archived_at": null,
  "deleted_at": null
}
```

### Create a temporary link

`POST /api/v1/temp-links` returns `201` with the TempLink.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `target` | string or null | no | Same rules as for permanent links. |
| `slug` | string | no | Same rules as for permanent links. When absent, `null` or `""`, the server generates one as for permanent links: 5 random letters and digits, for example `Xk3pQ`. |
| `ttlSeconds` | number | no | Lifetime in seconds. Default 900 (15 minutes). |
| `og_title`, `og_description`, `og_image_url` | string | no | Same rules as for permanent links. |
| `expired_url`, `disabled`, `disabled_reason`, `archived` | | no | As on permanent links. The expiry is always the TTL: `expires_at` in the body answers `400`. |

How `ttlSeconds` is interpreted:

- It must be a JSON number when present, otherwise `400 {"error": {"code": "invalid_request", "message": "ttlSeconds must be a number"}}`.
- The value is floored to a whole number. `0`, or anything that floors to `0` (for example `0.5`), becomes `900`.
- Everything else is clamped to the range 60 to 2,592,000 (30 days): `30` becomes `60`, `-5` becomes `60`, `1e15` becomes `2592000`.
- `expires_at` is `created_at` plus the resulting `ttl_seconds`.

Other errors: `400 {"error": {"code": "invalid_request", "message": "Invalid slug"}}`, `400 {"error": {"code": "invalid_request", "message": "Invalid target: must be a valid http(s) URL"}}`, `409 {"error": {"code": "conflict", "message": "Slug already exists", "slug": "…"}}`. Invalid `og_*` values are rejected as on permanent-link create.

```bash title="A link for a CI build that expires in one hour"
adto POST /api/v1/temp-links '{"target": "https://example.com/build/1234", "ttlSeconds": 3600}'
```

### Read, update and delete

| Route | Response |
|---|---|
| `GET /api/v1/temp-links/:slug` | `200` TempLink, or `404 {"error": {"code": "not_found", "message": "Temporary link not found"}}`. |
| `PUT /api/v1/temp-links/:slug` | Partial update of `target`, the three `og_*` fields and the lifecycle fields; absent keeps, `null` or `""` clears. Returns `200` TempLink, or `404 {"error": {"code": "not_found", "message": "Temporary link not found"}}`. The TTL and `expires_at` cannot be changed. |
| `DELETE /api/v1/temp-links/:slug` | `200 {"success": true}` after a soft delete, as for a permanent link; `?permanent=true` deletes the link with the same history cascade, and needs the account key: an SDK key is refused with `403 account_key_required`, since team members can read SDK keys in Settings. `404 {"error": {"code": "not_found", "message": "Temporary link not found"}}` for a slug that is not yours. |
| `POST /api/v1/temp-links/:slug/restore` | `200` TempLink after restoring a deleted temporary link, or `404`. |

There is no list route for temporary links in the v1 API. The dashboard's `GET /api/admin/links` returns them under `temp_links`.

### Lifecycle after expiry

1. From `expires_at` on, a tap goes to the link's `expired_url`, or answers the `410` "Link expired" page, for bots too. No click is recorded. The link still exists.
2. `resolve` answers `410 link_expired`; `GET /api/v1/temp-links/:slug` keeps answering `200`.
3. The daily job (03:17 UTC) deletes temporary links whose `expires_at` is more than 24 hours in the past, except those with an `expired_url`, which keep sending taps there until you delete them. Their clicks and installs are kept, and the slug is free again. From then on the link URL answers 404 and the API answers `404`.
4. The dashboard's "Delete expired" button (`DELETE /api/admin/temp-links/expired`) removes every expired temporary link of the account immediately, also keeping their history.

## Resolve a link (public)

`GET /api/v1/resolve/:username/:slug` needs no API key, no signature and is not rate limited. It is what the SDK calls when it opens a link, and it works for any account's links. CORS allows every origin.

```json title="GET /api/v1/resolve/acme/spring-sale"
{
  "id": "6f1c2a0e-3b7d-4c8e-9a11-2f5d8c0b4e77",
  "slug": "spring-sale",
  "target": "https://shop.example.com/sale",
  "url": "https://acme.adto.app/spring-sale",
  "type": "permanent",
  "created_at": "2026-10-12T09:00:00.000Z",
  "click_count": 12,
  "expires_at": null,
  "ttl_seconds": null,
  "metadata": { "screen": "sale", "campaign": "spring" },
  "ios_deep_link_path": "sale",
  "android_deep_link_path": "sale"
}
```

Every key is always present. Which ones carry values depends on the kind of link:

| Field | Permanent link | Temporary link |
|---|---|---|
| `type` | `"permanent"` | `"temporary"` |
| `click_count` | number | `null` |
| `expires_at` | the expiry, or `null` | value |
| `ttl_seconds` | `null` | value |
| `metadata` | JSON or `null` | `null` |
| `ios_deep_link_path`, `android_deep_link_path` | string or `null` | `null` |

`id`, `slug`, `target`, `url` and `created_at` are filled for both kinds. Only a live link resolves: a paused one answers `410 {"error": {"code": "link_paused", "message": "This link is paused"}}`, an expired one `410 {"error": {"code": "link_expired", "message": "This link has expired", "expired_url": "…"}}` (`null` without one), and an unknown username or slug, or a deleted link, `404 {"error": {"code": "not_found", "message": "Link not found"}}`. The `og_*`, routing and tracking fields are not exposed here.
