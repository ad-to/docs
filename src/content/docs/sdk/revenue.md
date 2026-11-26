---
title: Revenue
description: Report purchases, renewals and refunds with their store transaction so revenue counts once, renewals stay with the link that won the user, and every amount is summed in one currency. Verify App Store purchases with StoreKit 2, and let the App Store and Google Play report renewals and refunds themselves.
sidebar:
  order: 7
---

`trackConversion` puts revenue on the link that brought the user. Three things make that number trustworthy: the store's transaction id, so a purchase reported twice counts once; the subscription's first purchase on every renewal, so the renewal stays with the same link; and one currency for every total.

## Purchases

Pass the transaction id the store gave you and the store:

```dart
await adto.trackConversion(
  revenue: 9.99,
  currency: 'EUR',
  productId: 'pro_monthly',
  transactionId: purchase.transactionId,
  store: AdtoStore.appStore,       // or AdtoStore.play
  purchaseType: AdtoPurchaseType.subscription,
);
```

The server stores each transaction once per store and type. A purchase you report again, after a restore, a reinstall or a retry, answers as a duplicate and changes nothing. Without a `transactionId` every call is a new purchase.

## Renewals

A renewal is a purchase with its own transaction id that names the subscription's first purchase as `originalTransactionId`:

```dart
await adto.trackConversion(
  revenue: 9.99,
  currency: 'EUR',
  productId: 'pro_monthly',
  transactionId: renewal.transactionId,
  originalTransactionId: renewal.originalTransactionId,
  store: AdtoStore.appStore,
  purchaseType: AdtoPurchaseType.subscription,
);
```

The renewal is credited to the link and install of that first purchase, even when it is reported from another device or after a reinstall.

## Refunds and cancellations

```dart
await adto.trackConversion(
  revenue: 9.99,                      // the amount refunded
  currency: 'EUR',
  transactionId: refunded.transactionId,
  store: AdtoStore.appStore,
  transactionType: AdtoTransactionType.refund,
);
```

A refund is subtracted from the link's net revenue and is credited to the same link as the purchase it refunds. A `AdtoTransactionType.cancel` records that a subscription will not renew; it moves no money and needs a `revenue` of 0.

## Google Play transaction ids

On Google Play, pass the order id as `transactionId` and the purchase token as `originalTransactionId`. The token stays the same across a subscription's renewals, so it plays the part Apple's original transaction id plays, and Play's notifications name it:

```dart
// in_app_purchase on Android
await adto.trackConversion(
  revenue: 4.99,
  currency: 'EUR',
  productId: purchase.productID,
  transactionId: purchase.purchaseID,                                  // GPA.…
  originalTransactionId: purchase.verificationData.serverVerificationData, // the purchase token
  store: AdtoStore.play,
  purchaseType: AdtoPurchaseType.subscription,
);
```

## Store-verified App Store purchases

With StoreKit 2, pass the transaction's `jwsRepresentation` as `appStoreJws`. With `in_app_purchase` on iOS it is the purchase's `verificationData.serverVerificationData`:

```dart
await adto.trackConversion(
  revenue: 9.99,
  currency: 'EUR',
  store: AdtoStore.appStore,
  appStoreJws: purchase.verificationData.serverVerificationData,
);
```

The server checks Apple's signature against Apple Root CA - G3 and that the transaction belongs to one of your apps (its iOS bundle ID), then takes the transaction id, original transaction id, product, price and currency from the transaction instead of the values you passed. The row is marked as store-verified. A JWS that does not verify, or names an app you do not have, rejects that event: it is listed in the batch's `rejected` answer and the queue drops it. A sandbox transaction is recorded like a production one, with `store_environment: "Sandbox"` in its metadata.

## Store notifications

The stores can report purchases, renewals and refunds themselves, including renewals that happen while the app is closed and refunds the app never hears about. Turn it on per app in the dashboard: **Apps → your app → Store notifications → Turn on**. It shows two URLs; each contains a token that names the app, and **New URLs** replaces them.

### App Store

In App Store Connect, open the app, then **App Information → App Store Server Notifications**, choose **Version 2** and paste the App Store URL for Production. For Sandbox, paste the URL of the same app in your [test environment](/dashboard/settings/#test-environment): switch the dashboard to **Test**, then **Apps → your app → Store notifications → Turn on**. Sandbox purchases then count in the test environment; sent to the live URL, they would count as live revenue (tagged `store_environment: "Sandbox"`). The server verifies each notification and the transaction inside it against Apple Root CA - G3 and refuses one for another bundle ID than the app's, with a `422` so Apple sends it again: once you fix the app's bundle ID, the next attempt is recorded. A recorded purchase is also sent to your webhooks as `event.conversion` with `source: "store"`.

| Notification | Recorded as |
|---|---|
| `SUBSCRIBED`, `DID_RENEW`, `ONE_TIME_CHARGE`, `OFFER_REDEEMED`, `DID_CHANGE_RENEWAL_PREF` with `UPGRADE` | a purchase, at Apple's price |
| `REFUND` | a refund of that transaction |
| `REFUND_REVERSED` | the refund is removed; the purchase counts again |
| `DID_CHANGE_RENEWAL_STATUS` with `AUTO_RENEW_DISABLED` | a cancel (counts nothing) |
| `TEST` | logged only |
| anything else | logged and ignored |

### Google Play

Play publishes real-time developer notifications to a Pub/Sub topic (Play Console → **Monetization setup**). In Google Cloud, create a **push** subscription on that topic with the Play URL as the endpoint, turn on authentication, and pick a service account. Enter that account's email under **Pub/Sub service account** in the dashboard: the server refuses a push whose Google-signed token is for another account or another URL, and a notification for another package than the app's.

Play's notifications carry no price and, for renewals, no order id. So:

| Notification | Recorded as |
|---|---|
| a voided purchase (refund or chargeback) | a refund of the order the app reported, at that order's amount |
| subscription `REVOKED` | a refund of the latest purchase the app reported for that token |
| subscription `CANCELED` | a cancel of that purchase |
| subscription or one-time `PURCHASED` | the purchase the app reported for that token is marked as store-confirmed |
| renewals and everything else | logged only; report renewals from the app |

A notification that names a token or order the app never reported is logged as `unmatched`.

### Counted once

Store rows and the app's reports are the same rows: one per store, transaction id and type. A transaction the app already reported is confirmed by the store's notification (the store's amount wins, and the row is marked store-verified) instead of being counted again. A transaction the store reported first takes the link and install of the app's report when it arrives. A renewal or refund is credited to the link of the subscription's first purchase. The stores resend a notification until it is acknowledged; a resend changes nothing.

The last 20 notifications of the app, with what each did (`recorded`, `updated`, `duplicate`, `ignored`, `unmatched`, `test`, `rejected`, `sandbox`), are listed under the URLs. The log is kept for 90 days.

## RevenueCat

If the app sells through [RevenueCat](https://www.revenuecat.com/), RevenueCat can report purchases, renewals, cancellations and refunds instead of the stores. Prices then arrive for Google Play too, and the app doesn't need to call `trackConversion`. RevenueCat's webhooks come with its Pro plan.

An app uses either store notifications or RevenueCat, not both: the two can name a Google Play purchase differently, and it would count twice. In the dashboard, **Apps → your app → Store notifications → Use RevenueCat instead** turns RevenueCat on (and the store notification URLs off). It shows a URL and an authorization value. The value is shown only then, so copy it straight away; **New URL and value** makes new ones and stops the old ones.

In RevenueCat, open **Integrations → Webhooks**, add a webhook with the URL, paste the value as the **authorization header value**, and send it production events. Use **Send test event** to check it: the dashboard lists it as `test`. A request without the value is refused with `401`. Sandbox events are logged as `sandbox` and never count as revenue.

### Which link gets the credit

RevenueCat sends the subscriber's **app user ID**, the ID the app logs in to RevenueCat with. Pass the same ID to [`identify`](/sdk/events/) as the customer ID, and AdTo knows which installs belong to the subscriber:

1. A renewal, cancel or refund goes to the link of its subscription's first purchase, as with store notifications.
2. A new subscription goes to the subscriber's newest install from before it started. A reinstall, or a second phone, that came later never takes the credit. If that install came from no link, the link of an earlier install on the same phone keeps it.
3. A subscriber AdTo doesn't know as a customer yet, such as one still on an app version without the SDK, is recorded under their app user ID, without an install. Once the app identifies them, their earlier rows take the install and link that rule 2 gives, and their next renewals follow. If the app reports the same transaction with `trackConversion`, the row takes the app's link and install.

RevenueCat's anonymous IDs (`$RCAnonymousID:…`) are never matched. A purchase made before the app logs the user in to RevenueCat is matched once RevenueCat has linked the two IDs, from its next event on.

| RevenueCat event | Recorded as |
|---|---|
| `INITIAL_PURCHASE`, `RENEWAL`, `NON_RENEWING_PURCHASE` | a purchase, at the price in the customer's currency (a free trial is 0) |
| `CANCELLATION` with `cancel_reason` `CUSTOMER_SUPPORT` | a refund of that transaction, when AdTo counted its purchase; otherwise logged as `unmatched` |
| any other `CANCELLATION` | a cancel (counts nothing) |
| `UNCANCELLATION` | the subscription's cancel is removed |
| `REFUND_REVERSED` | the refund is removed; the purchase counts again |
| `TEST` | logged only |
| anything else, and stores other than the App Store and Google Play | logged and ignored |

A RevenueCat row never changes a row the stores verified, and fills in the amount of a row the app reported first. Its `event.conversion` webhook carries `source: "revenuecat"`. RevenueCat retries an event it couldn't deliver for about 80 minutes; a resend changes nothing.

## One currency

Every total in the dashboard is in your account's revenue currency. A new account starts in the currency of the country it was opened from (EGP from Egypt), or EUR when that country is not known, and an owner can change it in [Settings](/dashboard/settings/#revenue-currency). Each amount is converted with the euro rate of its day: the European Central Bank's reference rate for the 30 currencies it publishes, and the [exchange-api](https://github.com/fawazahmed0/exchange-api) feed's for every other current ISO 4217 currency but one, 124 of them, such as the Egyptian pound (EGP) or the Saudi riyal (SAR). The rate is the one published on that day, or the last one before it (the ECB skips weekends and holidays). The event keeps the amount and currency you sent; only the totals are converted.

The server fetches the rates once a day. An amount in a currency whose rate it does not have yet counts as 0 until the next daily run converts it. A currency with no rate at all never converts: the Syrian pound (SYP), left out because the feed's rate for it is about a hundred times other sources', and codes that are no longer currencies, such as BGN since Bulgaria adopted the euro. Changing the revenue currency converts every stored amount again.

## What the server stores

| Field | From |
|---|---|
| `transaction_id`, `original_transaction_id` | `transactionId`, `originalTransactionId`, up to 128 characters |
| `product_id` | `productId` |
| `store` | `app_store` or `play` |
| `purchase_type` | `one_time` or `subscription` |
| `quantity` | 1 to 10000 |
| `transaction_type` | `purchase`, `refund` or `cancel` |
| `revenue_base` | The amount in the revenue currency, filled by the server |
| `source` | `store` when a verified StoreKit 2 transaction or a store notification vouches for the row |

The wire format is on [Events and attribution](/api/events-and-attribution/).
