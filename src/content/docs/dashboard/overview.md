---
title: Overview
description: Every KPI, chart and panel on the dashboard's Overview screen, with the exact definition the server uses for each number.
sidebar:
  order: 1
---

The Overview is the first screen after you sign in, at https://dashboard.adto.app/dashboard. It shows clicks, installs, conversion, revenue and attribution quality for your whole account, over one of three time ranges, and refreshes itself every minute while it is on screen.

Every number on this page is defined below exactly as the server computes it. Several of them do not reconcile with each other by design; the last section lists those cases.

## Tabs

The top bar reads **Overview**, with five tabs under it. The open tab is part of the address (`?tab=sources`), so a tab can be linked to; **Summary** is the default.

| Tab | What it holds |
|---|---|
| **Summary** | Four [KPI tiles](#kpi-tiles) and the [chart](#the-summary-chart) |
| **Sources** | [Installs by campaign](#installs-by-campaign), [Installs by source](#installs-by-source), [Top referrers](#top-referrers) and [Breakdown](#breakdown) |
| **Audience** | [Platforms](#platforms), [Top countries](#top-countries), [Retention](#retention), [Top devices](#top-devices), [Carriers](#carriers) and [Connectivity at install](#connectivity-at-install) |
| **Quality** | [How installs were matched](#how-installs-were-matched), [Match confidence & referrer health](#match-confidence--referrer-health) and [Apple Search Ads](#apple-search-ads) |
| **Activity** | [Recent events](#recent-events) and [Recent installs](#recent-installs) |

## Controls

Every tab opens with the [setup banner](#setup-banner), when there is a step left, and a date bar.

| Control | What it does |
|---|---|
| **Date range**: **7 days**, **30 days**, **90 days** | Selects the time range for every tab. The default is **7 days**. Every range is whole calendar days in UTC, today included: **7 days** starts at midnight UTC 6 days ago, **30 days** 29 days ago and **90 days** 89 days ago. The totals come from the daily rollups, so only today's rows are read row by row (see [Scheduled jobs](/guides/scheduled-jobs-and-retention/#daily-rollups-both-triggers)). |
| **Window** | The dates the range covers, such as "Nov 20 – Nov 26 UTC · against the 7 days before": the window, and the comparison window every tile's change is measured against. |
| Live refresh | "Live · updates every minute", at the end of the date bar. Every minute the page reloads the overview numbers, but only while its tab is on screen. A hidden tab does not refresh; coming back to it refreshes at once when the last refresh is a minute old. The recent events load each time the **Activity** tab opens, and the match-confidence read-out when the **Quality** tab opens or the range changes; neither takes part in the minute refresh. |
| **Set up with your agent** | In the top bar, on wide screens. Copies the prompt `Read https://adto.app/SKILL.md and help me get started with AdTo` to the clipboard. The label reads "Prompt copied — paste it into your agent" for two seconds. If the clipboard is blocked, a browser prompt shows the text instead. See [Automation and agents](/guides/automation-and-agents/). |

While the overview loads, four gray placeholder tiles show. If the overview request fails, a red alert with the server's error message replaces the tab's content under the date bar. If only the match-confidence request fails, that panel says so and the rest of the Quality tab still renders. If the recent-events request fails, the Recent events card shows its empty state.

## Setup banner

While an app still has setup steps left, a banner at the top of every tab, above the date bar, names the next one from the app's [Are we live yet?](/dashboard/apps/#are-we-live-yet) checklist, for example "Next step for My App: Save the app’s identifiers.", with the step's hint, how many steps are done, a link to the rest, and a button to where the step is done:

| Step | Button |
|---|---|
| Save the app's identifiers, Fix the Universal Links file, Fix the App Links file | **Open iOS settings** or **Open Android settings**: the app's editor opens at that section with the cursor in its first empty field |
| Install the SDK | **Install the SDK** ([Installation](/sdk/installation/), in a new tab) |
| Tap a link | **Open your links** |
| Register a first install | **Initialize the SDK** ([Initialize and handle links](/sdk/initialize-and-handle-links/)) |
| Attribute a first install to a link | **Test attribution** ([Testing](/sdk/testing/)) |
| Send a first event | **Send events** ([Events](/sdk/events/)) |

An account with no app sees **Register your app** instead. With several apps, the banner checks the first five in the order the Apps screen lists them and shows the first one with a step left. The optional webhook step is never shown.

**Later** hides the step for three days. Once the app has an install, the SDK, attributed-install and event steps show **Hide** instead, which hides that step for that app for good and moves on to the next one: a live app may never do them (an older SDK build that does not report its version, no install from a link yet, no events sent). Identifier and file steps always come back after three days, because links do not open the app until they pass. Hidden steps are remembered in this browser only.

Viewers do not see the banner. If loading it fails, the Overview shows without it.

## How deltas work

Under its number, each KPI tile shows the change against the previous window of the same length (for **7 days**, the 7 days before the current 7 days), followed by "vs previous {N} days".

- The change is relative, rounded to one decimal: `(current − previous) ÷ previous × 100`.
- When the previous window is 0, there is no percentage to show: the tile reads "No earlier period to compare" if the current value is above 0, and **0%** if it is 0 too.
- Green with an up arrow and a `+` means an increase; red with a down arrow and a `−` a decrease; gray **0%** with no arrow means no change.
- For **Conversion** the change is the relative change of the rate, not a difference in percentage points. A rate that moves from 2.0% to 3.0% shows +50%.

## KPI tiles

**Summary** shows four tiles, by default **Clicks**, **Installs**, **Opens** and **Revenue**. The pencil on a tile lists the nine KPIs below; picking one shows it in that tile, and a KPI already in another tile is marked "shown". The choice is kept in this browser only.

Selecting a tile charts its metric in [the chart](#the-summary-chart) under the tiles, and an orange dot marks the tile. **Unique visitors** and **Referrals** have no chart, so their tiles cannot be selected.

Counts use the server's own timestamps, so an install counts in the window in which the SDK reported it, not when the device installed the app.

| Tile | Caption | What the number is |
|---|---|---|
| **Clicks** | | Every click in the window: web taps on your links plus clicks reported by the SDK. Requests from bots never create a click. |
| **Unique visitors** | | The number of distinct hashed IP addresses among those clicks. It counts IP addresses, not devices or people: two phones on one Wi-Fi network are one visitor, and one phone that moves from Wi-Fi to cellular is two. |
| **Installs** | `first installs · {n} attributed · {n} flagged` | First installs reported in the window: installs on a device the server had not seen install the app before. "Attributed" is the subset the server matched to a click; "flagged" (shown when any) the ones that tripped a [fraud rule](/start/fraud-rules/). App updates are not installs: when an existing user updates into your first build with AdTo, the SDK reports the install as an app update. When the window has any, a second line reads `{n} app updates by existing users, not counted`. |
| **Reinstalls** | `on a device that had the app before` | Reinstalls reported in the window: the server had seen the device install the app before, or the SDK found its own first-install date on the device. They go through the same matching as any other install and are counted here instead of under Installs. |
| **Conversion** | `attributed installs ÷ clicks` | Attributed first installs divided by clicks, as a percentage with one decimal. Organic installs had no tap to convert, so they are left out. |
| **Revenue** | `tracked conversions` | Net revenue of the window: purchases minus refunds, each converted into your revenue currency (Settings) with the exchange rate of its day. Amounts whose rate is not known yet count as 0 until the daily conversion run. |
| **Leads** | `signup events` | Events whose type is `lead`. The automatic signup lead is recorded only on a first install, never on a reinstall or an app update, so existing users signing in after an update do not count. |
| **Opens** | `link re-opened the installed app` | App opens in the window: times a link opened an already installed app, as reported by the SDK. |
| **Referrals** | `referred users who signed up` | Referrals recorded in the window whose referred user signed up (status `converted`). See [Referral programs](/guides/referral-programs/). |

## The Summary chart

Under the tiles, a chart of one metric per day: one point per calendar day in UTC of the range, today included, so it covers the same days as the tiles. Days with nothing show 0. Labels are short dates such as "Oct 3". It reads the raw rows through [the analytics query](/api/admin-api/#the-analytics-query), as [Breakdown](#breakdown) does.

| Control | What it does |
|---|---|
| **Show** | The metric: **Clicks** (the default), **Installs** (first installs), **Reinstalls**, **Opens**, **Conversion** (attributed installs ÷ clicks of each day), **Leads** or **Revenue**. Selecting a tile picks its metric here too. |
| **Compare by** | **Nothing**, or any dimension [Breakdown](#breakdown) offers. The chart then draws the 5 largest values of that dimension over the window as lines, orange first, with a legend of each value and its total (no total for Conversion). When there are more, a note says "The 5 largest {dimension} values are shown." |
| **Add filter** | Opens a panel: **Filter on** (a dimension), **Values** (one or more of its values, searchable), then **Keep these** or **Leave these out**, and **Add filter**. Each filter becomes a chip such as "Country is EG, SA" or "Platform is not Desktop"; **×** removes it. One filter per dimension: adding another for the same dimension replaces it. Filters stay while you change the metric or the comparison, but are not saved. |

Empty state, over the chart: "Nothing in this window. Try a longer range.", or "Nothing in this window with these filters. Try a longer range or remove a filter."

## Sources tab

### Installs by campaign

Each bar stacks first installs and, lighter on top, reinstalls. The label above each bar is that bucket's conversion rate (first installs ÷ clicks); it is blank for buckets with no clicks.

- Each attributed install is grouped by the `utm_campaign` query parameter of the click it was matched to. `(direct)` collects matched clicks that carried no `utm_campaign`. A reinstall counts under a campaign only when a new tap won it; one routed to its first install's click is `Organic`.
- `Organic` collects installs that were not matched to any click. It has 0 clicks, so it never shows a conversion label.
- Clicks are grouped by their own `utm_campaign`, so a campaign with clicks but no installs still appears, with a bar of 0.
- Conversion per bucket is first installs ÷ clicks, one decimal.
- The chart shows the top 8 buckets, sorted by first installs, then reinstalls.
- The tooltip adds **Flagged for fraud** when any: first installs from a tap on that campaign that tripped a [fraud rule](/start/fraud-rules/), whether credited or not. An install whose credit a blocked rule withheld is counted under `Organic` for installs but under its tap's campaign here, so a campaign that only draws fraud still shows it.

This chart reads the `utm_campaign` parameter on the URL people tapped. It does not read the **Campaign** field in a link's Tracking section, nor `adto_campaign`. Those values are stored on the click and returned to the app, but they do not feed this chart. See [URL parameters](/links/url-parameters/).

### Installs by source

The same grouping, using `utm_source` instead of `utm_campaign`, with no top-8 limit.

### Top referrers

Subtitle: "The site of the page each click came from."

The top 8 referring sites by clicks in the window. AdTo stores only the origin of the referring page (`https://news.ycombinator.com`, never the full URL). Clicks without a referrer, and clicks referred from one of AdTo's own domains, count as `(direct/none)`. Hover a truncated value to see it in full; the bar is relative to the sum of the rows shown.

Empty state: "No clicks in this window."

## Breakdown

On the **Sources** tab, under the charts, **Breakdown** puts every metric side by side for one dimension: clicks, installs (first installs), reinstalls, conversion, opens, leads, revenue and LTV (the net revenue to date of the window's first installs, per install: what an install from that source has been worth so far; app revenue only, since a website purchase has no install), per link, campaign, country, platform, OS, language, referrer, channel (the referrer and UTM values read as one source: search, social, AI assistant, email and so on, listed under [the analytics query](/api/admin-api/#the-analytics-query)), UTM value, app version, match source, install store (the App Store, Google Play or another Android store), install source (what Google Play says brought an Android install: Instagram, a Google search, Gemini, an ad), routing rule (to compare A/B variants), page button (the button a visitor took on a link page) or event source (the app, the website through `adto.js`, or a store) (the **By** menu). It covers the range selector's window, cut into days in your browser's time zone, and loads on its own rather than on the minute refresh.

- Click a value to keep only it. A chip appears above the table, "Country is GB"; click another value to add it to the chip.
- Hover a row and click **−** to leave that value out instead: "Country is not US". A chip keeps values or leaves them out, not both; picking the other way for the same dimension replaces it.
- Click **is** or **is not** on a chip to turn it around, **×** to remove it, **Clear** to remove them all. Chips stay when you change **By**, so you can keep Germany and break it down by campaign.
- `(organic)` is what no link brought, such as installs no tap won; `(none)` is a link that carried no value for the dimension, such as a tap without a `utm_source`. A metric the dimension does not apply to shows `—`: taps have no app version, and only installs have a match source, an install store or an install source.
- An install counts under its winning tap's values, an event under the link it is credited to (the re-opening link inside its window, else the install's), the same rules as the [link stats](/dashboard/links/).

The table shows the 50 largest values by clicks. **Export CSV** downloads it with up to 1,000; **Raw rows…** downloads the clicks, installs, app opens or events of the window that pass the chips; the installs file has every install report, reinstalls and app updates included, and a chip the rows cannot have (clicks by match source) stops the download with a message. A file over 50,000 rows is written in the background when the deployment has an exports bucket: the card says so and turns into a download link, good for 7 days. See [Export raw data](/api/export/).

The numbers read the raw rows, so they reach back as far as the retention window keeps them. They can differ from the KPI cards at the window's edges: for 30 and 90 days the cards cut days in UTC and Breakdown in your time zone, and for 7 days the cards count the last 168 hours and Breakdown seven calendar days, today included. In a browser set to UTC the 30- and 90-day totals agree. The same query is in the [Admin API](/api/admin-api/#the-analytics-query) and in the MCP `analytics` tool.

## Audience tab

### Platforms

Subtitle: "Clicks by the platform they came from."

A donut of clicks in the window by device type, with a legend reading `{percent} · {count}`. The types are **iOS**, **Android**, **Desktop** and **Other** (a click whose device type is none of those). "Desktop" is any browser that does not identify as iPhone, iPad, iPod or Android. An iPad asking for desktop sites counts as iOS when the link's app is set up for iOS, and as Desktop otherwise.

### Top countries

Subtitle: "Clicks by the visitor's country."

The top 10 countries by clicks in the window, from the geolocation of the click, each a flag and the country's name, with a bar and the click count; hover a row for the full name. Clicks through Tor have no country and read **Tor**.

Empty state: "No clicks in this window."

### Retention

On the **Audience** tab, under Platforms and Top countries, **Retention** answers "did the users stay?" for the installs of the last 90 days, whatever the range selector says. Two boxes filter it: **All links** or one link (installs credited to it), and **All platforms**, **iOS** or **Android**. The card loads on its own and does not take part in the minute refresh.

The cohort is the first installs that carry a device id; reinstalls and app updates are left out, and a device counts once, on its first install day. A device is active on a day when the app started (the SDK's device ping, sent once a day), a link opened it, or it sent an event that day. Days are calendar days in UTC: day 1 is the UTC day after the install's, so an install at 23:30 that opens again at 00:30 is retained on day 1, and one that opens again at 23:50 the same day is not.

| Part | What it shows |
|---|---|
| **Day 1**, **Day 7**, **Day 30** | The share of installs at least that many days old that were active on that exact day after their install, with "{retained} of {installs} installs"; "no installs old enough yet" until one is |
| The weekly table | One row per install week (Monday to Sunday, UTC), newest first, with its installs, then for each of the next eight weeks the share active in that week, shaded darker from 10%, 30% and 60%. A week that has not started is `·`. Hover a cell for the counts. |
| The note under the table | How many installs carried no device id (the app turned collection off) and are not counted |

The figures read the raw rows, so they reach back as far as the retention window keeps them. See [Admin API](/api/admin-api/#analytics).

### Top devices

The top 8 device models among installs in the window that reported a model, with an install count. Models are stored lowercased. A model AdTo knows shows its marketing name with the code after it (`Samsung Galaxy S23 Ultra sm-s918b`, `iPhone 17 iphone18,3`), or the name alone when the SDK already sent the marketing name (`iPhone 17`); any other shows as `{model} · {manufacturer}`. Android names come from Google Play's supported-devices list, refreshed weekly for the models your installs report; iPhone names from a list in AdTo for the iPhone 8 to the iPhone 18 Pro Max. The count is first installs; when the model also had reinstalls the row adds `+{n} reinstalls`. App updates are left out.

Empty state: "No device data yet. Turn on `collectDeviceData` in the SDK." See [Device data and privacy](/sdk/device-data-and-privacy/).

### Carriers

Subtitle: "Mobile network codes (MCC/MNC), Android only."

The top 8 mobile carriers among installs that reported one, as `{mcc}/{mnc}` with an install count, reinstalls included and app updates left out. Only the Android SDK reports carrier codes.

Empty state: "No carrier data yet."

### Connectivity at install

The top 6 connection types reported at install, such as `wifi` or `mobile`, with install counts, reinstalls included and app updates left out.

Empty state: "No connectivity data yet."

## Quality tab

### How installs were matched

Subtitle: "A deterministic referrer or copied link beats the fingerprint."

Non-reinstall installs in the window, grouped by how they were matched, as bars reading `{percent} · {count}` in this order:

| Bar | Installs counted |
|---|---|
| **Install referrer (deterministic)** | Matched through an install referrer handed over by a store (Google Play, Huawei, Samsung, Xiaomi, vivo) or by Meta's on-device referrer. |
| **Copied link (deterministic)** | Matched through a link found on the device's clipboard: the iOS tap-to-store page copies it, or the user pasted it. |
| **Device fingerprint (probabilistic)** | Every other matched install, including rows that record no match source. |
| **Organic** | Installs not matched to a click, including those whose credit a blocked [fraud rule](/start/fraud-rules/) withheld. |

Empty state: "No installs in this window."

Inside the card, an amber box appears when at least one first install in the window carries fraud flags: `{n} first install(s) flagged for fraud`, then `· {n} acted on by a blocked rule` when an app's blocked rules acted on some, followed by a chip per rule reading `{label} · {count}` (and `({n} acted on)`). Hovering a chip explains the rule; clicking it opens [Installs](/dashboard/installs/) filtered to that rule. The total and the chips count the same first installs; one install can trip several rules. See [Fraud rules](/start/fraud-rules/).

## Match confidence & referrer health

On the **Quality** tab, under How installs were matched, this panel reads a separate endpoint and answers a different question from the cards above: not how many installs, but how reliably they were matched. Its intro text: "How reliably installs were matched in the last {N} days: the confidence the server assigned, how often the store handed back the install referrer, and whether the fingerprint agreed with the referrer when both existed."

:::caution[Different totals]
The panel counts every install in the window, including reinstalls and app updates. Its totals are higher than the Installs KPI, which counts first installs only.
:::

### Tiles

| Tile | Value | Caption | Color rule |
|---|---|---|---|
| **Credited installs** | Matched installs ÷ all installs. | `{m} of {n} matched to a click` | None. |
| **Android referrer coverage** | Android installs whose store install-referrer read returned `ok`, divided by all Android installs. | `{delivered} delivered · {missing} missing of {n}`, or `no Android installs in this window`. "Missing" counts the read statuses `service_unavailable`, `timeout`, `error` and `not_available`; other statuses count in neither number. | Amber when more than 5% of Android installs are missing. |
| **Fingerprint accuracy** | Among installs matched by a deterministic signal, the share where the fingerprint, run in the background, would have picked the same click with high confidence. | `shadow match agreed on {a} of {n} referrer installs`, or `needs referrer-matched installs to measure`. | Green at 90% or above. |
| **Click → install** | `{median} median`: the median time between the server-side click time and the install report, over up to the 5,000 most recent matched installs in the window. | `p90 {x} · {n} matched installs`, or `no matched installs yet`. | None. |

**iCloud Private Relay**, above the tiles whenever the window has iOS link taps, installs or not: the share of checked iOS link taps that came through Apple's relay, such as `7% of iOS link taps (20 of 283 checked) came through Apple's relay`. Safari sends an iCloud+ user's taps through the relay when Private Relay is on, while their apps use the phone's own address, so the fingerprint cannot match those taps; the tap-to-store page's copied link still can. Taps are checked against Apple's published list of relay addresses, which the server downloads when it is built. The line says instead when the server has no copy of the list, when no tap in the window was checked yet (right after the check was turned on, every tap is older), and when the list is over 30 days old, since taps through newer relay addresses then count as not through the relay. A team member with the viewer role does not see whether a single tap came through the relay.

Durations are formatted as `45s`, `2m 10s`, `1h 5m` or `3d 2h`. A `—` means there is nothing to measure.

### Confidence per platform

One bar per platform (the install's reported OS: **iOS**, **Android**, or the raw value), sorted by installs. The label on the right reads `{percent} credited · {n} installs`, and on Android adds `· referrer {percent}`.

Each bar is split into segments by the confidence the server assigned. Hover a segment for `{Label}: {count} ({percent}) — {hint}`:

| Segment | Color | Hint | Meaning |
|---|---|---|---|
| **Guaranteed** | dark green | `install referrer, copied link or paste` | A deterministic signal named the click. Credited. |
| **High** | light green | `fingerprint, corroborated and unique — credited` | The fingerprint found one clearly best click. Credited. |
| **Medium** | amber | `routed to the link, not credited` | The fingerprint found a plausible click. The app receives the deep-link route, but the install does not count as attributed. |
| **Low** | light gray | `two devices tied on different links, or contradicted — nothing` | A tie between taps on different links, or a contradiction. No route, no credit. |
| **No candidate** | near white | `no click from this network in the window` | Installs the server assigned no confidence to: no click from the same network within the window, or an install that skipped matching (app updates, replayed signals). |

The legend under the bars repeats the five labels, each with its hint on hover. What the levels mean, and the windows involved, are explained in [How attribution works](/start/how-attribution-works/).

### Install referrer reads

Chips reading `{status} · {count}` for every referrer-read status reported in the window, sorted by count. Hover a chip for its plain-English meaning. Statuses that indicate a failed read are amber.

| Status | Tooltip | Amber |
|---|---|---|
| `ok` | referrer delivered | |
| `organic` | store reported organic | |
| `feature_not_supported` | store too old for the API | |
| `service_unavailable` | store service unavailable | yes |
| `developer_error` | developer error | yes |
| `permission_error` | permission error | yes |
| `timeout` | timed out | yes |
| `error` | read error | yes |
| `not_available` | no store app | |
| `not_android` | iOS (no referrer API) | |
| `unknown` | not reported (older SDK) | |

Empty state: "No referrer telemetry yet."

### Kept out of matching

`{n} existing users who updated to an SDK build — recorded as app updates, never matched against the click pool.` This is the number of reports in the window whose install state was `update`: the app was already on the device before the SDK shipped, so there is no install to attribute.

### Signals on known matches

A table over the installs in the window that a store referrer, pasted link or Meta referrer matched to a recorded tap: for each signal, how often that tap and the install **Agree**, **Mismatch** or are **Missing** one side. The signals worth 0 points carry `· not scored`. Two last rows carry `· network`: **Same IP address** (whether that tap came from the install's address, which the fingerprint needs) and **Same IP version** (both IPv4 or both IPv6). Its note reads "Over {n} installs a store referrer, pasted link or Meta referrer proved: how often each signal agreed with that tap. A signal marked not scored earns points only once these numbers support it. The network rows say how often that tap came from the install’s address at all, which the fingerprint needs, and whether a change of address was a switch between IPv4 and IPv6." Empty state: "No install in this window had a deterministic match to compare with." See [Signals recorded for no points](/start/how-attribution-works/#signals-recorded-for-no-points).

### Rejected SDK requests

Calls to the `/api/v1` API that the server refused as malformed (400, 413 or 422) in the window, grouped by route and reason, most frequent first, at most 10 rows: the route (for example `POST /api/v1/events`), the error code and message, how often and when last. The server keeps the request's SHA-256 and size, never its body, and deletes the rows after 30 days. Authentication failures and rate limits are not listed. An SDK build sending a field the server rejects shows up here before anyone notices missing data.

### SDK versions in the wild

A table of the SDK builds that reported in the window: the build (`flutter_adto 1.0.0`), the **New installs** registered by it, the **Events** it sent, and when it was **Last seen**. New installs show what your current release ships; events show which older builds are still running. Up to 20 builds, most recently seen first. Empty: "No SDK build reported in this window."

### Empty and failed states

- "No installs in this window." when the window has no installs at all.
- "The attribution-quality read-out is unavailable right now." when the endpoint fails. The rest of the Overview still renders.

## Apple Search Ads

On the **Quality** tab, this card appears only once the window contains at least one install whose iOS SDK sent an AdServices token, whatever the outcome of the lookup. Its text: "iOS installs whose AdServices token Apple matched to one of your campaigns. Apple sends campaign ids, not names; they match your Apple Ads reports."

| Tile | Caption | Counts installs whose lookup status is | Color rule |
|---|---|---|---|
| **Attributed** | `installs Apple credited to a campaign` | `resolved`: Apple returned an attribution. | Green when above 0. |
| **No attribution** | `Apple found no matching record` | `no_attribution`. | None. |
| **Pending** | `waiting on Apple · {n} expired or invalid` | `pending`. The `{n}` in the caption adds up `expired` (no answer within 24 hours of the token), and `invalid` (Apple rejected the token, or none arrived). | Amber when above 0. |

Below the tiles, up to 8 rows list the campaigns behind the attributed installs: `campaign {id}` (or `campaign —` when Apple sent no id), a chip reading `view-through` for impression claims and `tap-through` otherwise, and the install count. Rows are grouped by campaign id and claim type, so one campaign can appear twice.

These counts include reinstalls. The lookup lifecycle (three inline attempts, hourly retries, the 24-hour token limit) is described in [Clicks and installs](/api/clicks-and-installs/).

## Activity tab

## Recent events

Subtitle: "Leads, conversions and custom events your app sent with the AdTo SDK."

The 12 most recent events of any type (`identify`, `lead`, `conversion`, `custom`), newest first. **This card ignores the time range**, and loads each time the Activity tab opens. Each row shows a type chip (green for `conversion`, orange for `lead`, gray otherwise), the event name (or the type when there is no name), `/{slug}` when the event is attributed to a link, and on the right either the amount in the event's own currency (a refund with a minus sign) when the event carries revenue, or the event's date.

Empty state: "No events yet. Call `identify` or `trackConversion` in the SDK." See [Events](/sdk/events/).

## Recent installs

On the **Activity** tab, next to Recent events. Its subtitle: "Open one to see why it was credited, signal by signal, with its tap, opens and events." The 12 newest installs, whatever the range: a chip with how each was matched (`referrer`, `clipboard`, `fingerprint`, or `organic`), the link it was credited to, the OS and device model, a fraud chip when rules fired, and the time. **All installs** opens the [Installs](/dashboard/installs/) screen, which lists every install with filters and a Watch mode for testing. Clicking one opens the **install detail**:

- **Verdict**: credited link, organic, or `credit to /{slug} withheld`, match source, confidence, reinstall, fraud flags.
- **Fraud checks**: every install rule with what it compared, flagged, clear or not checked (see [Installs](/dashboard/installs/#the-install-detail)).
- **Why**: the signal that named the click (store referrer, pasted link, Meta referrer, a late claim), or the fingerprint's best and runner-up candidates with each signal marked agree, mismatch or missing (device model, OS version, language, screen, network), their scores, the gap between them, the window that applied and how many repeat taps were merged. Under each candidate, **Recorded, not scored** marks the signals worth 0 points the same way: OS minor version, text size, dark mode, Increase Contrast, Reduce Motion, time zone (the UTC offset), language and region, and Android WebView build. **Show values** adds what the app and the tap each reported next to every signal (see [Installs](/dashboard/installs/#the-install-detail)). When a deterministic signal decided, a block **The tap the signal named, signal by signal** shows how that tap compares with the install, scored and unscored signals alike, and under **Network** whether it came from the install's address and IP version, when one was recorded (see [Attribution internals](/start/attribution-internals/#the-stored-explanation)), and the fingerprint's own pick is shown as well. When the fingerprint did not run, the reason is given.
- **Tap**: the credited (or routed) tap with its time, device and country, and any late claim that upgraded the verdict.
- **App opens** and **Events** of that install, then its **Screens**: the install's app sessions with the screens shown in each, as on a [customer](/dashboard/customers/#screens).
- **This device's data**: **Download everything stored (JSON)** and **Erase this device**, for a person's request to see or delete their data; see [A request from one person](/guides/privacy-and-data/#a-request-from-one-person). Erasing asks for confirmation, then shows what went ("Device erased: 1 install, 3 events…"). An install past the retention window has no device id left, and says so.

Installs recorded before match explanations existed show the verdict without the **Why** part.

## Numbers that do not add up to each other

These differences are by design. Knowing them saves you from chasing phantom discrepancies.

- **Installs vs the quality panel.** Reinstalls are matched like any other install; only app updates and replayed signals skip the fingerprint. The Installs KPI, the attribution-quality chart and the per-link install counts leave them out; the Reinstalls KPI, the campaign and source charts, Breakdown and the Links screen show them next to the first installs; the Match confidence panel, Carriers, Connectivity and Apple Search Ads count them in.
- **Conversion above 100%.** Organic installs count in the numerator.
- **Revenue.** Net of refunds and converted into one currency, here and on the Links and Customers screens. A purchase in a currency whose rate has not arrived yet counts as 0 until the next daily run.
- **Summary chart and tiles.** The chart reads the raw rows, as Breakdown does, and the tiles the daily rollups, so they can differ the way [Breakdown](#breakdown) and the tiles do.
- **Campaign and source charts.** They read `utm_campaign` and `utm_source` from the tapped URL, never the link's Tracking fields.
- **Recent events.** Always the latest 12, whatever the range.
- **Per-link numbers.** The [Links](/dashboard/links/) screen shows lifetime totals, so its columns do not match a 7-, 30- or 90-day Overview.
- **Fraud counts.** First installs only, in the total and in each rule's chip.
- **Withheld installs.** An install whose credit a blocked fraud rule withheld appears as Organic in the match sources and installs, even though the server still records which link it came from; the flagged counts place it under that link and campaign. See [Fraud rules](/start/fraud-rules/).

