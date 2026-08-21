# Google Analytics 4 (GA4)

GA4 is the analytics baseline every other ad platform integration builds on. The whole `event_id` + dataLayer pattern in this skill is compatible with the standard GA4 recommended-events model.

---

## The ecommerce event canon (11 events)

GA4 recognizes 11 recommended ecommerce events. Missing any one breaks the funnel report at that step.

| Event | Fires when | Feeds report |
|---|---|---|
| `view_item_list` | User views a product listing / category | Shopping behavior |
| `view_item` | User views a product detail page | Product performance |
| `select_item` | User clicks a product in a list | Product performance |
| `add_to_cart` | User adds a product | Shopping behavior, Purchase Journey |
| `view_cart` | User views the cart | Shopping behavior |
| `remove_from_cart` | User removes a product | Shopping behavior |
| `begin_checkout` | User starts checkout | Checkout Journey |
| `add_shipping_info` | User selects/enters shipping | Checkout Journey |
| `add_payment_info` | User selects/enters payment | Checkout Journey |
| `purchase` | Transaction completes | Ecommerce Purchases, Monetization |
| `refund` | Transaction refunded | Monetization |

**Minimum viable:** `view_item`, `add_to_cart`, `begin_checkout`, `purchase`.
**Full checkout funnel visibility:** add `add_shipping_info`, `add_payment_info`.
**Cart abandonment analysis:** add `view_cart`, `remove_from_cart`.

---

## Every event needs an `items` array

```js
items: [{
  item_id: 'SKU-001',           // must match your product catalog
  item_name: 'Widget',
  item_brand: 'YourBrand',
  item_category: 'Widgets',
  item_variant: 'Blue-Large',
  price: 79.99,
  quantity: 1,
  currency: 'USD'               // required if item is in different currency than event
}]
```

If `items[]` is empty, the transaction still records but item-level revenue is $0. Silent failure mode.

---

## Audit checklist

- [ ] GA4 property created; Measurement ID (`G-XXXXXX`) in an env var
- [ ] Google tag installed on every page (via gtag.js directly, GTM, or `@next/third-parties`)
- [ ] **Enhanced Measurement** enabled on the web data stream
- [ ] All 11 ecommerce events firing (or at minimum the "4 required": `view_item`, `add_to_cart`, `begin_checkout`, `purchase`)
- [ ] Every event has a populated `items[]` with `item_id` matching your catalog
- [ ] `purchase` event has `transaction_id`, `value`, `currency`, `items[]`
- [ ] `purchase` and other key events marked as **Key Events** in GA4 Admin
- [ ] Google Ads linked (Admin → Product Links → Google Ads)
- [ ] Data retention set to 14 months (Admin → Data Collection → Data Retention)
- [ ] GA4 receives consent state (`analytics_storage`) from CMP
- [ ] For Stripe redirect flows: **Measurement Protocol backup** fires `purchase` from the webhook with the same `transaction_id`
- [ ] For EU: regional endpoint `region1.google-analytics.com` used

---

## Install (gtag.js — direct, no GTM)

```tsx
// app/layout.tsx (Next.js App Router)
import Script from 'next/script';

const GA_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID!;

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        {/* Consent Mode v2 defaults MUST come first — see consent-mode-v2.md */}
        <Script id="ga4" strategy="afterInteractive" src={`https://www.googletagmanager.com/gtag/js?id=${GA_ID}`} />
        <Script id="ga4-init" strategy="afterInteractive">
          {`
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', '${GA_ID}', {
              send_page_view: true,
              cookie_flags: 'SameSite=None;Secure'
            });
          `}
        </Script>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## Firing ecommerce events

```ts
// lib/tracking/ga4.ts (browser)
declare global { interface Window { gtag: any; dataLayer: any[] } }

type Item = { item_id: string; item_name: string; item_category?: string; item_brand?: string; price: number; quantity: number };

export function ga4ViewItem(items: Item[], currency = 'USD') {
  window.gtag?.('event', 'view_item', {
    currency, value: items.reduce((s, i) => s + i.price * i.quantity, 0), items
  });
}

export function ga4AddToCart(items: Item[], currency = 'USD') {
  window.gtag?.('event', 'add_to_cart', {
    currency, value: items.reduce((s, i) => s + i.price * i.quantity, 0), items
  });
}

export function ga4BeginCheckout(items: Item[], value: number, coupon?: string, currency = 'USD') {
  window.gtag?.('event', 'begin_checkout', { currency, value, coupon, items });
}

export function ga4Purchase(order: {
  transaction_id: string; value: number; currency: string;
  tax?: number; shipping?: number; coupon?: string;
  items: Item[];
}) {
  window.gtag?.('event', 'purchase', order);
}
```

---

## Measurement Protocol (server-side backup for Purchase)

**Why:** Stripe redirect breaks browser context roughly 5–15% of the time. A user pays, closes the tab before landing back on `/success`, and no browser purchase fires. Measurement Protocol closes that gap.

**How:** Fire the purchase server-side from the Stripe webhook, with the **same `transaction_id`**. GA4 dedupes on `transaction_id` — if the browser also fired, the second one is dropped.

```ts
// lib/tracking/ga4.ts (server)
const MEASUREMENT_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID!;
const API_SECRET = process.env.GA_MEASUREMENT_PROTOCOL_API_SECRET!;

// Use EU endpoint if serving EU traffic and required by your DPO:
// const ENDPOINT_HOST = 'https://region1.google-analytics.com';
const ENDPOINT_HOST = 'https://www.google-analytics.com';

export async function sendGA4MP(input: {
  client_id: string;                    // from _ga cookie, passed via Stripe metadata
  user_id?: string;
  transaction_id: string;
  value: number;
  currency: string;
  coupon?: string;
  items: { item_id: string; item_name: string; price: number; quantity: number }[];
  consent?: { ad_user_data: 'GRANTED' | 'DENIED'; ad_personalization: 'GRANTED' | 'DENIED' };
}) {
  if (!input.client_id) {
    console.warn('GA4 MP: no client_id available — event will be attributed as new user');
  }

  const body = {
    client_id: input.client_id || crypto.randomUUID().replace(/-/g, ''),
    user_id: input.user_id,
    non_personalized_ads: input.consent?.ad_personalization === 'DENIED',
    consent: input.consent,
    events: [{
      name: 'purchase',
      params: {
        transaction_id: input.transaction_id,
        value: input.value,
        currency: input.currency,
        coupon: input.coupon,
        items: input.items
      }
    }]
  };

  const url = `${ENDPOINT_HOST}/mp/collect?measurement_id=${MEASUREMENT_ID}&api_secret=${API_SECRET}`;
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  });

  // MP always returns 2xx (even for bad payloads) — use the debug endpoint to validate.
  return res.status;
}

// Debug endpoint for testing — returns validation errors
export async function sendGA4MPDebug(input: any) {
  const url = `${ENDPOINT_HOST}/debug/mp/collect?measurement_id=${MEASUREMENT_ID}&api_secret=${API_SECRET}`;
  const res = await fetch(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(input) });
  return res.json();
}
```

---

## Environment variables

```env
NEXT_PUBLIC_GA_MEASUREMENT_ID=G-XXXXXXXXXX
GA_MEASUREMENT_PROTOCOL_API_SECRET=abc123        # Admin → Data Streams → [stream] → Measurement Protocol API secrets
```

---

## Beyond code (GA4 Admin)

- **Admin → Data Streams → [Web stream] → Configure tag settings → Manage automatic event detection** — enable Form interactions (needed for Enhanced Conversions later).
- **Admin → Data Streams → [Web stream] → Measurement Protocol API secrets → Create** — copy the value into `GA_MEASUREMENT_PROTOCOL_API_SECRET`.
- **Admin → Events → Key Events → Mark `purchase` as key event.** Do this for any other conversion actions (lead form submissions, subscriptions, etc.).
- **Admin → Product Links → Google Ads → Link.** Required for Google Ads to import GA4 conversions.
- **Admin → Data Collection and Modification → Data Retention → 14 months.**

---

## Common failure modes

- **Empty `items[]` on `purchase`.** Transaction records with $0 item revenue. Reports show sales but no products.
- **Purchase fires on every reload of the success page.** Fix: use `transaction_id` (GA4 dedupes automatically) *and* store an "already-fired" flag in `sessionStorage`.
- **`client_id` mismatch between browser and MP.** Browser purchase and server MP purchase are attributed as two different users, breaking user-level attribution. Fix: pass GA4 client_id (`_ga` cookie) into Stripe session metadata; use the same value in MP.
- **MP always returns 2xx.** Silent failure mode — bad payloads are accepted but never processed. Use the `/debug/mp/collect` endpoint during setup to validate.
- **Consent Mode signals not sent to MP.** Fix: pass `consent` object in the MP payload for EEA users.

---

## Testing

1. **DebugView** (Admin → DebugView) — the fastest way to see events fire in real time. Enable debug on your browser via the `?debug_mode=true` URL parameter or the [GA Debugger Chrome extension](https://chrome.google.com/webstore/detail/google-analytics-debugger/jnkmfdileelhofjcijamephohjechhna).
2. **Realtime report** — Reports → Realtime. Shows events with a 30-sec delay.
3. **Measurement Protocol debug** — send test events to `/debug/mp/collect` and check for validation errors.

---

## Docs

- Ecommerce: https://developers.google.com/analytics/devguides/collection/ga4/ecommerce
- Recommended events: https://support.google.com/analytics/answer/9267735
- Measurement Protocol: https://developers.google.com/analytics/devguides/collection/protocol/ga4
- Consent Mode + GA4: https://developers.google.com/tag-platform/security/guides/consent
