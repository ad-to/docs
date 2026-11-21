---
title: URL parameters
description: The adto_* query parameters that change routing, set the in-app route or tag a click per share, the utm_* parameters the Overview charts use, and the parameters AdTo reserves.
sidebar:
  order: 2
---

Any link can be shared with query parameters that change what happens on that tap. Only keys prefixed `adto_` are read for routing and tracking; `utm_*` keys are stored for analytics. The parameters are not stored on the link, so the same link can be shared many ways:

```text
https://acme.adto.app/welcome?adto_campaign=newsletter&adto_deeplink=products/42&utm_source=email
```

The same parsing applies to clicks the SDK reports when a link opens the installed app: the SDK sends the opened URL's `adto_*` keys as `link_params`, and the server validates them exactly as it does for a browser tap.

## `adto_deeplink`: the in-app route

`adto_deeplink` replaces the link's **iOS deep link path** and **Android deep link path** for this tap, on both platforms.

| Accepted value | Example | Stored as |
|---|---|---|
| A bare route with an optional query | `products/42?color=black` | `products/42?color=black` |
| The route behind one of the app's own schemes (case-insensitive) | `myapp://products/42` | `products/42` |
| Any other scheme | `https://example.com/x` | Dropped |

The route may be at most 512 characters and may not contain `:`, whitespace, `#`, `;` or control characters. A leading `/` is stripped. The app the value is checked against is the link's app, or the default app.

Where the route is used:

- the Android `intent://` bounce (`intent://products/42?color=black&adto_click=<id>`);
- the interstitial's **Open app** button and the tap-to-store page's **Open in app** button (`myapp://products/42?color=black&adto_click=<id>`);
- the QR page's "Open in app" buttons (without `adto_click`).

The value is stored on the click as `link_params.deep_link` and returned to the app after an attributed install as `deep_link` (`AdtoAppInstallResponse.deepLink`, also on `routing.deepLink` for medium-confidence routing). On a warm open the SDK exposes it as `AdtoLinkClick.clickParams.deepLink`. Without a path, scheme links take the form `myapp://acme/welcome`.

## Redirect and fallback parameters

These override the destination for one tap. Each value must be an http(s) URL of at most 2048 characters; it is normalized and used as is. They are never stored.

| Parameter | Applies to | Effect |
|---|---|---|
| `adto_redirect_ios`, `adto_redirect_android` | That mobile platform | Replaces the store as the destination for a visitor without the app. If the URL's host is a store (`apple.com` and subdomains on iOS; `play.google.com` or a Huawei, Samsung or Xiaomi store host on Android), the attribution tag is added; otherwise the URL is used verbatim. |
| `adto_redirect_macos`, `adto_redirect_windows`, `adto_redirect_linux` | That desktop OS | The destination, ahead of `adto_fallback`, the link's desktop fallback, the app fallback and the QR page. |
| `adto_redirect` | Every platform without its own `adto_redirect_<platform>` | Same as above for the platform it lands on. |
| `adto_fallback` | Desktop, and mobile visitors with no destination at all | Used after the platform redirects and before the link's desktop fallback. Ignored on a mobile tap that has a store, custom redirect, app fallback or target. |

The platform is derived from the user agent:

| Visitor | Platform key |
|---|---|
| User agent contains `iPhone`, `iPad` or `iPod`, or an iPad asking for desktop sites (see [How a tap is routed](/links/how-a-tap-is-routed/)) | `ios` |
| User agent contains `Android` | `android` |
| macOS | `macos` |
| Windows | `windows` |
| Linux | `linux` |
| Anything else | None: only `adto_redirect` applies |

The full order in which these compete with the link's and the app's own settings is in [How a tap is routed](/links/how-a-tap-is-routed/).

## Tracking parameters

Five free-form values are stored on the click and handed back to the app after an attributed install:

| Parameter | Falls back to |
|---|---|
| `adto_label` | The link's **Label** |
| `adto_campaign` | The link's **Campaign** |
| `adto_adgroup` | The link's **Adgroup** |
| `adto_creative` | The link's **Creative** |
| `adto_tracker` | Nothing; it has no link-level default |

Each value is trimmed; an empty value or one with control characters is dropped; longer values are truncated to 255 characters. The link's defaults are the Tracking fields of a permanent link (temporary links have none). The six stored values (`deep_link` plus the five above) live in the click's `link_params` and come back as `click_params` on the verdict and on the webhook, and as `AdtoClickParams` in the SDK, with `deepLink`, `label`, `campaign`, `adgroup`, `creative` and `tracker`.

## `adto_referrer`: who shared the link

`adto_referrer` names the customer who shared this link, by the id your app passes to `identify`:

```
https://acme.adto.app/invite?adto_referrer=1042&adto_label=spring
```

It is stored on the click, not in `click_params`. An install credited to the tap gets it as `install.link?.referrerCustomerId`, and the new user's first `identify` records a referral and sends `referral.converted`, exactly as a [referral link](/guides/referral-programs/) does. On a referral link the link's own customer wins and `adto_referrer` is ignored. The value is trimmed; one longer than 200 characters or with control characters is dropped, never cut. Anyone can edit the URL to name another customer, so when that matters, give each customer a referral link instead.

## `adto_web_referrer`: the page that sent the visitor

A tap on a link placed on your own website arrives with your website as its referrer. `adto_web_referrer` names the page that sent the visitor to your website in the first place:

```
https://acme.adto.app/app?adto_web_referrer=https%3A%2F%2Fwww.google.com&utm_source=spring
```

[`adto.js`](/sdk/web/#carry-the-visits-source-into-the-link) adds it for you (`adto.link()` and the app banner). The tap stores its origin as the click's `referrer`, in place of the `Referer` header. Only an `http` or `https` URL is accepted; anything else is ignored and the header is used. It is never passed on to a destination.

:::caution[Tracking values do not feed the Overview charts]
The Overview's **Installs by campaign** and **Installs by source** charts group installs by the `utm_campaign` and `utm_source` of the matched click, not by `adto_campaign` or the link's Tracking fields. Use `utm_*` for dashboard charts and `adto_*` for values your app needs at runtime.
:::

## `utm_*` parameters

`utm_source`, `utm_medium`, `utm_campaign`, `utm_term` and `utm_content` are stored on the click verbatim. A key the tapped URL does not carry is taken from the link's own UTM values, or its template's, when it has them (see below). They are kept as long as the click (the retention window) and drive two Overview charts: installs grouped by the matched click's `utm_campaign` (`(direct)` when the click had none, `Organic` when the install matched nothing) and by its `utm_source`. The SDK sends the same five keys when it reports a click from an opened URL.

### Link-level UTM values

A link can carry its own `utm_source`, `utm_medium`, `utm_campaign`, `utm_term` and `utm_content` (API fields of the same names; **Tracking** in the link drawer), directly or from its [template](/api/links/#templates). They do two things:

- A click whose URL has no value for a key stores the link's.
- Every web destination of a tap gets the link's values added as query parameters, each only when the destination URL has none of its own: the desktop fallback, the custom redirect, the app's Fallback URL, the link's target, and a `adto_redirect` or `adto_fallback` URL. Store URLs never get them, and neither do scheme and `intent://` URLs.

So `https://shop.example.com/sale` behind a link with `utm_source=newsletter` opens as `https://shop.example.com/sale?utm_source=newsletter`, and the shop's web analytics name the same campaign as AdTo.

## Your own parameters

Keys without the `adto_` or `utm_` prefix, such as a share-time `ref` or an `item`, never change routing. They still reach the app and, when the link asks, the website.

**The app.** A warm open gives the app the whole opened URL (`click.queryParameters['productId']`). After a deferred install the verdict carries them as `click_params.query`, `{"productId": "42"}`. The rules for that copy:
- **Included:** the first value of each key, at most 20 keys and 2 KB.
- **Dropped:** a key over 64 characters or a value over 256, `utm_*` keys (they are on the click already), and AdTo's [reserved parameters](#reserved-parameters).

**The website.** Turn on **Pass the link's query parameters to website destinations** in the link drawer, under Template and tracking. Over the API, set `forward_query: true` on the link, or on a [template](/api/links/#templates). The tap's own parameters are then copied onto every web destination that `utm_*` values reach (see above):

- **What is copied:** the same keys the app gets, plus the tap's `utm_*`.
- **Precedence:** a tapped key wins over the destination URL's own value for it, and over the link's `utm_*`.
- **Never copied:** onto store URLs or scheme and `intent://` URLs. `adto_*` keys and the reserved parameters are never copied.

So a link to `https://shop.example.com/sale?campaign=summer`, tapped as `…/spring-sale?ref=anna&campaign=instagram`, opens `https://shop.example.com/sale?campaign=instagram&ref=anna`.

A route meant for the app belongs in `adto_deeplink` (`?adto_deeplink=products/42`), which the app gets on both paths as `deep_link` and which routes the intent and the "Open in app" buttons too. The prefix also keeps AdTo's keys apart from your website's own parameters. The reasons are in [Design decisions](/start/design-decisions/).

## Reserved parameters

| Parameter | Set by | Meaning |
|---|---|---|
| `adto_click` | AdTo | The click id, added to store URLs (inside `referrer` or `ct`), to scheme and `intent://` URLs and to the copied link. Do not add it to links you share; a reported click with an unknown id gets a fresh id. |
| `_ch` | AdTo | The marker of the one-shot client-hints redirect in front of the interstitial. |
| `_ipad` | AdTo | The answer of the touch check in front of a Mac Safari tap: `1` for an iPad asking for desktop sites, `0` for a Mac. |
| `_ref` | AdTo | The origin of the page that linked to a Mac Safari tap, carried past the touch check (whose own page would otherwise be the referrer). |

## Validation and what reaches the destination

Each invalid parameter is dropped on its own; the rest still apply, and routing continues as if the dropped one were absent. The visitor is sent to the destination URL as configured. AdTo adds only these:
- **Store tags:** `ct` on the App Store, `referrer` on Google Play and the OEM stores.
- **The click id:** `adto_click` on scheme and intent URLs. Web destinations get it too on a link with **Add the click id to website destinations** on (see [Web pixel](/sdk/web/)).
- **The link's own UTM values,** on web destinations.
- **The tap's own parameters,** on web destinations, for a link with **Pass the link's query parameters to website destinations** on (see [Your own parameters](#your-own-parameters)). Social-preview bots and the QR code ignore parameters entirely; see [Social previews and the QR page](/links/social-previews-and-qr/).
