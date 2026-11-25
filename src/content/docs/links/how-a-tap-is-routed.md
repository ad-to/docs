---
title: How a tap is routed
description: Every decision the redirect engine makes between a tap on an AdTo link and the destination, on mobile, on desktop and for bots, and when the click is recorded.
sidebar:
  order: 1
---

A link such as `https://acme.adto.app/welcome` is served by the redirect engine. This page lists its decisions in the order the server takes them.

## Entry points

| Request | Handled when |
|---|---|
| `https://{username}.adto.app/{slug}` | Subdomain mode. Exactly one path segment; two or more return JSON `404 {"error":{"code":"not_found","message":"Not found"}}`. |
| `https://{custom domain}/{slug}` | The account's [custom domain](/links/custom-domains/) while it is ready, active, moved or within the 7 days after its removal. Its root goes to the app's Fallback URL. |
| `{base}/r/{username}/{slug}` | Any host, any mode; on a custom domain only for the account it belongs to (another account's link is a 404). |
| `{base}/?user={username}/{slug}` | Any host except the dashboard host and custom domains. |
| `https://{username}.adto.app/` | No slug: a known username is sent to the dashboard, an unknown one to `UNKNOWN_HOST_REDIRECT`. |

## Decision order

| # | Check | Result |
|---|---|---|
| 1 | The username does not exist | `302` to `UNKNOWN_HOST_REDIRECT` (`https://adto.app/` on the hosted service), or the 404 page when that variable is unset. Applies in path mode too. |
| 2 | No permanent or temporary link has this slug, or the link is deleted | `404` page |
| 3 | The link is paused | `410` "Link paused" page, for bots as well, **no click recorded** |
| 4 | The link has expired: a temporary link's `expires_at`, or a permanent link's when it has one, has passed | `302` to the link's expired URL when it has one, else the `410` "Link expired" page; for bots as well, **no click recorded** |
| 5 | The request is a `HEAD`, or the user agent is a bot | `200` social preview page, **no click recorded**. See [Social previews and the QR page](/links/social-previews-and-qr/). |
| 6 | Safari's Mac user agent without `_ipad` in the URL, and the link's app (the link's app, or the account's default, oldest one) is set up for iOS: a bundle ID, an App Store URL or an iOS scheme | `200` touch check, **no click recorded**. The page goes straight on to the same URL with `_ipad=1` when the browser reports touch (an iPad asking for desktop sites) or `_ipad=0` (a Mac), plus the referring page's origin as `_ref` and, when the request carried the no-track header, `adto_nt=1`; that request continues below. |
| 7 | Otherwise | A click row is built. The app is the link's app, or the account's default (oldest) app. |
| 8 | The user agent is iOS or Android | The mobile chain below |
| 9 | Anything else, or a mobile visitor with no destination at all | The shared tail below |

An archived link routes like any other. See [Links: pause, expire, archive and delete](/api/links/#pause-expire-archive-and-delete).

"iOS" means the user agent contains `iPhone`, `iPad` or `iPod`, or it is an iPad asking for desktop sites, which it does by default: Safari's Mac user agent with `_ipad=1` from the touch check, or a Mac user agent carrying `CriOS`, `FxiOS` or `EdgiOS` (Chrome, Firefox and Edge for iOS). "Android" means it contains `Android`, tablets included. Everything else is desktop, including an iPad's Safari tap on a link whose app is not set up for iOS. An iPhone set to request desktop websites passes the touch check too and is recorded as a tablet.

## The click row

The click gets a 16-character id (the `adto_click` value), `source: web`, the link kind, device type, OS (the Android major version comes from the client hint when present; the iOS version from Safari's `Version/` token, because Safari 26 and later freeze the OS token at 18.6 or 18.7, and an 18.6 or later OS token without one, as in-app browsers send, is stored without a version), the iOS minor version from the same `Version/` token, browser, device model (from the `Sec-CH-UA-Model` hint or a WebView `Build/` token), the user agent (512 characters), `Accept-Language` (128 characters), country, region, city and ASN from Cloudflare, the salted IP hash plus the raw IP (unless the deployment sets `STORE_RAW_IP=0`), the referring page's origin, or the package of the Android app that opened the link (null when the referrer is AdTo itself), the `utm_*` values and the `adto_*` tracking values. The interstitial and tap-to-store pages add the screen size, pixel ratio, touch points and connection type later, with the device's text size (iOS and iPadOS only), dark mode, Increase Contrast and Reduce Motion settings and its UTC offset. No CPU architecture is recorded: no phone browser sends one. Permanent links also increment their lifetime click counter; temporary links have none.

## Which value applies

Each routing field can come from several places. For one tap the first one that sets it wins:

| # | Source | Fields it can set |
|---|---|---|
| 1 | The tapped URL's `adto_*` parameters | the custom redirect (`adto_redirect`, `adto_redirect_<os>`), the desktop fallback (`adto_fallback`), the in-app route (`adto_deeplink`), the tracking values. See [URL parameters](/links/url-parameters/). |
| 2 | The link's first matching geo or language rule | `target`, `redirect_url`, `fallback_url`, the iOS and Android deep-link paths. See [Geo and language routing](#geo-and-language-routing). |
| 3 | The link | every routing field, the tracking defaults and UTM values, the tap-to-store page, the interstitial and the click id on web destinations |
| 4 | The link's app (or the account's default app) | the store URLs, the Fallback URL, the schemes, the tap-to-store page and the interstitial |
| 5 | The deployment | the branded QR page, `UNKNOWN_HOST_REDIRECT` |

The tapped URL's own `utm_*` beat the link's on the click row; the link's UTM values are added to every web destination that lacks them. See [URL parameters](/links/url-parameters/#link-level-utm-values).

### Geo and language routing

A permanent link can carry up to 20 rules (`routing_rules`; the **Geo and language routing** block in the link drawer). A rule names countries (ISO 3166 two-letter codes, from Cloudflare's view of the visitor), languages (the primary tag of the browser's first `Accept-Language` entry, such as `de` for `de-AT`), or both, and the link fields it replaces. Rules are tried in order; the first whose every condition matches stands in for the link's own values of the fields it sets, for that tap only. A tap that matches no rule is routed exactly as without rules: a rule never blocks a visitor. Bots and link unfurlers see the link's own preview and target.

A rule's iOS or Android deep-link path for the tap's platform is also stored on the click, so an install that click brings receives it as `deep_link` (unless the URL had a `adto_deeplink`).

A rule can also hold for a time or for a share of visitors:

- **Scheduled routing:** `starts_at` and `ends_at` (ISO times) bound the rule, for a sale page that takes over on Black Friday and steps back on Monday.
- **A/B splits:** `percent` gives the rule a share of visitors, 1 to 100. Shares stack in rule order and add up to at most 100: rules of 50 and 50 split everyone; 50 and 25 leave a quarter on the link itself. A visitor's share comes from their hashed IP, so they get the same variant on every tap from the same network.

The rule that routed a tap is recorded on the click as `routing_rule` (its `name`, else `rule 2` and so on). The Overview's Breakdown by **Routing rule**, or the analytics query's `routing_rule` dimension, compares the variants by clicks, installs, conversion, leads and revenue.

## Mobile: choosing the destination

The server resolves one destination for a visitor without the app. The first non-null entry wins:

| # | Candidate | Notes |
|---|---|---|
| 1 | A custom redirect whose host is a store, re-tagged | Custom redirect means, in order: `adto_redirect_ios` / `adto_redirect_android`, then `adto_redirect`, then a matching rule's `redirect_url`, then the link's **If the app isn't installed (mobile) > Custom URL** (`redirect_url`). Store hosts are `apple.com` and its subdomains on iOS; `play.google.com` and the Huawei, Samsung and Xiaomi store hosts on Android. The attribution tag is added. |
| 2 | A custom redirect, verbatim | Any other custom redirect URL. Not an install path, so no tag. |
| 3 | The tagged store URL | iOS: the app's App Store URL with `ct=adto_click=<id>`. Android: the visitor's OEM store when configured, else the Play Store URL, with `referrer=adto_click=<id>&adto_l={username}/{slug}`. See [App stores and OEM routing](/links/app-stores-and-oem-routing/). |
| 4 | The **app's** Fallback URL | This outranks the link's own target on mobile. |
| 5 | The link's target (its **Fallback URL**), or a matching rule's `target` | |
| 6 | Nothing | The visitor falls through to the shared tail: `adto_fallback`, then the link's desktop fallback, then the QR page. |

A permanent link's deep-link paths and a `adto_deeplink` parameter do not change the destination; they only change the in-app route used by the intent bounce and the "Open in app" buttons.

## Mobile: delivering it

With a destination in hand, the response depends on the app's settings and the browser. The first matching row applies:

| # | Condition | Response | Click recorded |
|---|---|---|---|
| — | iOS, inside an in-app browser (Facebook, Instagram, Messenger, LINE, WeChat, Snapchat, TikTok or X, or any app's own WebKit view whose user agent has no `Safari/` token), the app has a Team ID and bundle ID, the link is on another host than the API, and the tap is not on its way back (see below) | `302` to the [in-app browser page](#the-in-app-browser-page-ios) on the API host | Before the response |
| a | iOS, **iOS tap-to-store** is on for the link (its own setting, else the app's), and the destination is the App Store (a tagged `apple.com` custom redirect, or no custom redirect and an App Store URL) | `200` tap-to-store page | Before the response |
| b | Android, inside the Facebook or Instagram in-app browser (`FB_IAB`, `FB4A`, `FBAV` or `Instagram` in the user agent), the app has an **Android app scheme** and an **Android package**, and a destination exists | `200` opening page that tries the intent and falls back by itself (see below) | Before the response |
| c | **Mobile beacon interstitial** is on for the link (its own setting, else the app's) and a destination exists | `200` interstitial page. Hint-less Chromium first gets a one-shot `302` to the same URL with `?_ch=1` and `Critical-CH`, which inserts nothing. | Before the response |
| d | Hint-less Chromium (see below) | `302` with `Critical-CH` | **Not yet.** Chrome retries the navigation with the model hint and the retry records it. |
| e | Android intent bounce applies | `302` to an `intent://` URL | Before the response |
| f | Any other destination | `302` | After the response, in the background |
| g | No destination | Shared tail | After the response, in the background |

### Android intent bounce

The bounce opens the installed app directly from Chrome when App Links verification has not happened yet, and lands everyone else on the store. It is used only when all of these hold: the device is Android; the app has both an **Android app scheme** and an **Android package**; the user agent contains `Chrome/` (Samsung Internet and Edge qualify); the user agent is not an in-app browser or WebView (`; wv)`, `FB_IAB`, `FB4A`, `FBAV`, `Instagram`); and a destination exists.

```text
intent://{path or username/slug}?adto_click=<id>#Intent;scheme={scheme};package={package};S.browser_fallback_url={encoded destination};end
```

`{path}` is `adto_deeplink`, else the link's **Android deep link path**. The SDK reads `adto_click` from the opened URL and converts the web click into the app's own click instead of logging a second one. Verified App Links bypass the browser, and the server, entirely.

### Facebook and Instagram on Android

Their in-app browsers open an installed app from an `intent://` URL but ignore its `S.browser_fallback_url`, so a visitor without the app would stay on a blank page. They get the interstitial page instead, whatever the interstitial setting: after 450 ms it navigates to the same `intent://` URL, and when the page is still visible 1.5 seconds later (the app did not take over), it goes to the destination itself. With the app installed the tap lands in the app; without it, on the tagged store page. Other WebViews (`; wv)`) keep the plain `302` to the destination.

### The interstitial page

Turned on per app with **Mobile beacon interstitial**, and on or off for a single link under **Mobile landing pages** in the link drawer (`mobile_interstitial: true | false`; `null` follows the app). The page is titled "Opening the app…", carries `noindex`, and on load sends `{c, w, h, dpr, t, e, fs, cs, hc, rm, tz}` to `/_/collect`, which updates the click row once: the click id, the screen width and height, the pixel ratio, touch points and connection type, then the text size (`fs`, the computed px of `font: -apple-system-body`, sent only when the browser resolves the iOS system text style), the color scheme (`cs`, `dark` or `light`), Increase Contrast (`hc`, 1 or 0 from `prefers-contrast: more` or `no-preference`), Reduce Motion (`rm`, 1 or 0 from `prefers-reduced-motion`) and the UTC offset in minutes (`tz`). A key the browser cannot answer is left out; a value out of range is stored as null. It then routes automatically: to the intent bounce after 450 ms when the bounce applies, otherwise to the destination after 300 ms. A custom scheme is never tried automatically. After 1.6 seconds it shows "Taking too long?" with an **Open app** button; that tap copies `https://acme.adto.app/welcome?adto_click=<id>` to the clipboard, tries the scheme URL when the app has one, and after 2 seconds goes to the destination if the page is still visible. Without JavaScript a plain **Continue** link remains. When iOS tap-to-store is also on, iPhone taps take the tap-to-store page instead, so the switch affects Android only.

### The in-app browser page (iOS)

Instagram, Facebook and the other in-app browsers never open an app from a link on the page's own host, so an existing user's tap there used to reach the App Store instead of the app. For an app with a Team ID and bundle ID (its Universal Links), such a tap first goes to `https://api.adto.app/_/continue/<click id>`, which carries the tap's query along. That page, titled "Continue in the app", has an **Open in app** button that is a plain link back to the tapped URL on the link's own domain, with `adto_click=<click id>&adto_b=1` added. A tap on it crosses domains, so iOS can open the app through its Universal Link; the SDK reads `adto_click` and turns the web click into the app's own, as with the Android intent bounce. It is the page's only button: an app's custom scheme made iOS ask for permission, or did nothing, inside these browsers. The page copies nothing to the clipboard: the SDK reads it only after a page that copies.

Without the app, the in-app browser loads the link again. `adto_b=1` never goes to the page a second time, and an `adto_click` of the same link from the last hour, tapped from the same network, is the same click, so nothing more is counted; then the tap routes as in the table. The tap-to-store page copies that click, and on this way back it leads with **Get {app name}**, since the visitor has just tried the app. A visitor without the app therefore sees one more page. The page answers `404` for an unknown click or one more than an hour old. Taps with `adto_nt=1`, and links on the API host itself, never take it. On an iPhone with the app installed, **Open in app** opened it from Instagram, Facebook and Messenger, and those went on to the App Store without it; TikTok, LINE, WeChat, Snapchat and X are not confirmed yet, which is why the setting is marked Beta. Safari and WhatsApp never take this page: a tap there opens the app through its Universal Link before AdTo sees it.

### The iOS tap-to-store page

Turned on per app with **iOS tap-to-store**, and on or off for a single link under **Mobile landing pages** in the link drawer (`ios_tap_to_store: true | false`; `null` follows the app), so a paid campaign link can show it while a support link opens the store directly. On the hosted service a tap through iCloud Private Relay gets the page even with the app's switch off, when the link leads to the app's App Store page; only a link's `false` keeps it from those taps too. iOS has no install referrer and Safari only writes the pasteboard from a user gesture, so this page replaces the automatic redirect with one button. It is titled "Get {app name}" and shows **Get {app name}** plus, when the app has an iOS scheme, a secondary **Open in app** button. Inside the Facebook, Instagram, Messenger, LINE, WeChat, Snapchat, TikTok and X in-app browsers, and only when a scheme exists, the heading becomes "Continue in the app" and **Open in app** comes first. The Get tap shows a "Link saved" notice, writes `https://acme.adto.app/welcome?adto_click=<id>` to the pasteboard and opens the tagged store URL; a 600 ms safety timer opens the store even if the write never settles. **Open in app** only navigates to the scheme URL. There are no timers before the tap, and the same beacon as the interstitial fires on load. Without JavaScript the page shows an **Open the App Store** link.

### Client hints and hint-less Chromium

Modern Chrome freezes its Android user agent, so the device model only arrives through the `Sec-CH-UA-Model` client hint. Every engine response except the bot, 404, 410 and unknown-user ones carries `Accept-CH: Sec-CH-UA-Model, Sec-CH-UA-Platform-Version`, so later taps on the same subdomain arrive with hints. A request is "hint-less Chromium" when the model header is absent (an empty value counts as present), the user agent has no `Build/` token and no in-app marker, and `Sec-CH-UA` is present with `Sec-CH-UA-Mobile: ?1`. Such a request is answered with `Critical-CH`, which makes Chrome repeat the navigation with the hints; only the repeat records the click. A client that never retries leaves no web click, but an install can still be attributed through the store referrer, which names the link.

## Desktop and the shared tail

Desktop visitors, and mobile visitors with no destination, take this chain. The first non-null entry wins:

| # | Candidate |
|---|---|
| 1 | `adto_redirect_macos`, `adto_redirect_windows` or `adto_redirect_linux` for the visitor's OS, else `adto_redirect` |
| 2 | `adto_fallback` |
| 3 | A matching rule's `fallback_url`, else the link's **If the app can't open the link (desktop) > Custom URL** (`fallback_url`), unless the desktop page is the QR page |
| 4 | The app's Fallback URL, unless the desktop page is the QR page |
| 5 | The branded QR page |

**The desktop page.** On a computer, the link's **Desktop visitors see** setting (`desktop_qr`), else its app's **Desktop** setting, chooses between the website and the QR page. The app's setting is **Fallback URL** unless you change it. With **QR page**, rows 3 and 4 are skipped, so a desktop visitor gets the QR page even when a fallback URL is set. The page's "Continue to {host} on desktop" link then goes where the redirect would have: the link's fallback URL, else the app's, else the link's target. A phone is never sent to the QR page by this setting: it sees the page only with no destination at all.

Desktop never redirects to the link's target automatically. The target only appears on the QR page as the "Continue to {host} on desktop" link, when no fallback URL comes before it. The QR page encodes the link URL with the tap's parameters, lists the store buttons and the "Open in app" buttons, and is described in [Social previews and the QR page](/links/social-previews-and-qr/).

## Error pages

| Status | Page | When |
|---|---|---|
| 404 | "Link not found" / "This AdTo link does not exist." | Unknown slug, a deleted link, or unknown username when `UNKNOWN_HOST_REDIRECT` is unset |
| 410 | "Link paused" / "This link is paused and not available right now." | A paused link |
| 410 | "Link expired" / "This link has expired and is no longer available." | An expired link without an expired URL; for a temporary link until the daily delete (after that: 404) |

Both pages carry `noindex`, a status chip, a **Go to AdTo** button pointing at `https://adto.app/` and the footer "Check the link for typos, or ask whoever shared it for a new one." They are in the visitor's language like every link page (see [The link pages](#the-link-pages)). Two or more path segments on a tenant host return JSON `404 {"error":{"code":"not_found","message":"Not found"}}` instead.

## The link pages

The QR page, the interstitial, the tap-to-store page and the error pages share these things.

- **Language.** English, German, French, Spanish or Italian, the first of them in the browser's `Accept-Language` by weight; English when none matches. `<html lang>` says which.
- **Content-Security-Policy.** Each response has its own random nonce, and only the page's own script and style blocks carrying it run: `default-src 'none'; script-src 'nonce-…'; style-src 'nonce-…'` (plus `font-src 'self'` on the pages that use Outfit, served by the same host at `/_/fonts/`), `img-src data:`, `connect-src 'self'` for the beacon, `base-uri 'none'`, `form-action 'none'`, `frame-ancestors 'none'`. Nothing on the page can be framed, and nothing injected into it can run.
- **The app's icon.** The interstitial, the tap-to-store page and the in-app browser page show the app's icon from its store listing above its name. The server fetches it, never the visitor's browser: within an hour of an app being added or its identifiers changing, AdTo looks the app up on the App Store by its bundle ID (or the App Store URL's id, when the bundle ID matches), else on Google Play by its package (through unavatar.io when Google's listing page gives none), and keeps a small copy. It is refreshed weekly. An app with neither listing shows its name alone.
- **Apple's Smart App Banner.** On an iPhone, the tap-to-store page and the interstitial carry Apple's `apple-itunes-app` tag, so Safari shows its banner across the top: **Open** when the app is installed, **View** (the App Store page) when it is not. The banner passes the app this click's URL, `https://acme.adto.app/welcome?adto_click=<id>`, the same one the page copies, so the SDK knows the link and the click. It needs an App Store URL with an id on the app, and in-app browsers (Instagram, Facebook, TikTok) do not show it. A tap that goes straight to the App Store shows no page, so no banner.
- **Which way out.** A tap on a store badge or the **Get** button, an **Open in app** button, or the QR page's website link sends `/_/collect` `{c: <click id>, a: "cta_ios" | "cta_android" | "cta_open" | "cta_web"}` with `sendBeacon`, which survives the navigation the tap starts. The first one is kept on the click as `cta` with `cta_at`, from the same network within an hour of the tap. The Overview's Breakdown shows it as **Page button**, the analytics query as the `cta` dimension: how many QR-page visitors went to a store, to the app or to the website.


## Check where a link goes

The link drawer's **See where this link goes** (`GET /api/admin/links/:slug/route-check`) runs the saved link through this page's decision once for each of five devices (six on the hosted service) and lists what each visitor gets:

| Device | What it stands for |
|---|---|
| iPhone, Safari | An iPhone in its own browser |
| iPhone, Safari through iCloud Private Relay | The same iPhone with Private Relay on (iCloud+); listed only on a server that checks taps against Apple's list (the hosted service). With iOS tap-to-store off it still gets the tap-to-store page when the link leads to the app's App Store page |
| iPhone, inside Instagram | An iPhone in an app's built-in browser |
| Android, Chrome | An Android phone in Chrome |
| Android, inside Facebook | An Android phone in Facebook's or Instagram's built-in browser |
| Computer | A Windows computer |

Each row says what the visitor sees (a straight redirect, the [tap-to-store page](#the-ios-tap-to-store-page), the [interstitial page](#the-interstitial-page), a page that tries the app first, or the page with the QR code), where they end up, the in-app route the app would open, and the routing rule that decided, by name. A paused, expired or deleted link shows as not routed.

**Visitor's country** (two letters, such as `DE`) and **Language** (such as `fr`) are optional: fill them in to see what a visitor a [geo or language rule](#geo-and-language-routing) names would get.

- Nothing is counted: no click, no webhook, and the link's numbers do not move.
- It checks the saved link, so save your edits first.
- It shows the browser's path. A phone that already has the app opens it from the link itself, through Universal Links or App Links, without reaching AdTo; see [Entry points](#entry-points).
- An A/B rule picks its variant from the visitor's address, and the check is one visitor, so it shows one variant.

## Link health

**Link health**, at the top of the link drawer (`GET /api/admin/links/:slug/health`, or the `link_health` [agent tool](/guides/automation-and-agents/)), scores the saved link from 0 to 100 and lists what is wrong first, each with a link to where it is fixed. Every check reads what AdTo actually answers for the link: the association files from the link's own host (your custom domain while it is live, else your AdTo subdomain), the routes from [Check where a link goes](#check-where-a-link-goes), and the preview as Facebook's crawler sees it, image fetched. Nothing is counted.

| Check | Weight | Passes when |
|---|---|---|
| Custom domain | 10 | Your custom domain is the one links use. Only scored when you have one that is not being removed. |
| iOS identifiers | 15 | The app has a Team ID and a bundle ID in the shapes Apple accepts. |
| Universal Links file | 15 | The link's host serves `apple-app-site-association` as Apple reads it: answered 200 as JSON, under 128 KB, listing the app, every path covered. Checked once the iOS identifiers pass, since the file is built from them. |
| iPhone without the app | 10 | The app has an App Store URL, or the link sends phones without the app somewhere of its own. |
| Android identifiers | 15 | The app has a package and at least one SHA-256 signing fingerprint. |
| App Links file | 15 | The link's host serves `assetlinks.json` granting `handle_all_urls` to the package with every saved fingerprint. Checked once the Android identifiers pass. |
| Android without the app | 10 | The app has a Google Play URL, or the link sends phones without the app somewhere of its own. |
| Title | 10 | The shared link has a title of its own (the link's or the app's), not "Open in the app". |
| Description | 5 | The link or the app has a description. |
| Image | 10 | The image answers as an image to Facebook's crawler, 8 MB or less. |

The score is the weight of the passing checks over the weight of those that apply. A platform the app has no field for is left out; an app set up for neither platform fails both. A paused, expired or deleted link scores 0 and says so, since nobody is routed.

Below the checks, a table says what a tap does on an iPhone, an Android phone and a computer, with and without the app. Without the app is the route check's answer. With the app depends on half AdTo cannot see: the app's build must still list the link's host in Associated Domains (iOS) or verify it in an intent filter (Android). So the table says the app opens *when the build declares the host*, and shows the last time the app reported opening the link. That shows the app handles the link, though not which way it arrived: the SDK reports an open by Universal Link, App Link or scheme alike.

DNS records and the certificate of a custom domain are not scored here: the custom domain card in Settings checks them (see [Custom domains](/links/custom-domains/)).

## When the click is recorded

| Path | Click |
|---|---|
| `HEAD` or bot user agent | None |
| 404, 410, unknown username | None |
| `x-adto-no-track: 1` header or `?adto_nt=1` on the link | None. Routing is exactly that of a real tap; use it for QA taps and uptime monitors. The parameter never reaches the destination. |
| Tap-to-store page, interstitial page (also the Facebook and Instagram one), intent bounce | Written before the response is sent |
| Hint-less Chromium plain redirect | None on the first request; the hinted retry writes it |
| Every other redirect and the QR page | Written in the background after the response |
| A verified Universal Link or App Link opening the installed app | The browser never contacts the server. The SDK reports the open as a click with `source: mobile_app` and as an app open. |

Each tap nets at most one click: when the SDK reports a click that carries a `adto_click` id the server already has, the existing web row is converted to `source: mobile_app` instead of inserting another. A web click that was never written can still be recreated from the store referrer at install time; see [How attribution works](/start/how-attribution-works/).
