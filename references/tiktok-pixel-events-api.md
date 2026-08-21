# TikTok Pixel + Events API

Standard 2026 setup: Pixel (browser) + Events API (server) running together with shared `event_id` dedup. Pixel-only setups miss ~35% of conversions on iOS + blocker-heavy traffic.

---

## Standard events for ecom

`PageView`, `ViewContent`, `ClickButton`, `Search`, `AddToWishlist`, `AddToCart`, `InitiateCheckout`, `AddPaymentInfo`, `PlaceAnOrder`, `CompletePayment` (purchase equivalent), `Subscribe`, `Contact`, `Download`, `SubmitForm`.

Note: TikTok uses `CompletePayment` where every other platform uses "Purchase". Get this wrong and no purchase optimization data reaches TikTok's algorithm.

---

## Audit checklist

- [ ] Pixel ID + Events API access token stored in env vars
- [ ] Pixel installed on every page — `ttq.page()` fires on route changes (SPAs)
- [ ] All ecom events firing on the Pixel with correct standard names
- [ ] All the same events also firing via Events API from the server
- [ ] Shared `event_id` between Pixel and Events API for the same event
- [ ] Match keys sent server-side (hashed): `email`, `phone_number`, `external_id` + non-hashed `ttclid`, `ip`, `user_agent`
- [ ] `ttclid` captured from URL and stored in cookie on landing
- [ ] Deduplication rate healthy in TikTok Events Manager
- [ ] Test Events tool verifies both Pixel and Server events for the same conversion
- [ ] Consent-gated for EU

---

## Install the Pixel

```tsx
// app/layout.tsx (Next.js App Router)
import Script from 'next/script';

const TT_PIXEL_ID = process.env.NEXT_PUBLIC_TIKTOK_PIXEL_ID!;

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <Script id="tiktok-pixel" strategy="afterInteractive">
          {`
            !function (w, d, t) {
              w.TiktokAnalyticsObject=t;var ttq=w[t]=w[t]||[];ttq.methods=["page","track","identify","instances","debug","on","off","once","ready","alias","group","enableCookie","disableCookie","holdConsent","revokeConsent","grantConsent"],ttq.setAndDefer=function(t,e){t[e]=function(){t.push([e].concat(Array.prototype.slice.call(arguments,0)))}};for(var i=0;i<ttq.methods.length;i++)ttq.setAndDefer(ttq,ttq.methods[i]);ttq.instance=function(t){for(var e=ttq._i[t]||[],n=0;n<ttq.methods.length;n++)ttq.setAndDefer(e,ttq.methods[n]);return e},ttq.load=function(e,n){var r="https://analytics.tiktok.com/i18n/pixel/events.js",o=n&&n.partner;ttq._i=ttq._i||{},ttq._i[e]=[],ttq._i[e]._u=r,ttq._t=ttq._t||{},ttq._t[e]=+new Date,ttq._o=ttq._o||{},ttq._o[e]=n||{};n=document.createElement("script");n.type="text/javascript",n.async=!0,n.src=r+"?sdkid="+e+"&lib="+t;e=document.getElementsByTagName("script")[0];e.parentNode.insertBefore(n,e)};
              ttq.load('${TT_PIXEL_ID}');
              ttq.page();
            }(window, document, 'ttq');
          `}
        </Script>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## Fire events (browser)

```ts
// lib/tracking/tiktok-pixel.ts
declare global { interface Window { ttq: any } }

export function ttViewContent(product: { id: string; name: string; category?: string; price: number; currency: string }, eventId: string) {
  window.ttq?.track('ViewContent', {
    contents: [{ content_id: product.id, content_name: product.name, content_type: 'product', content_category: product.category, price: product.price, quantity: 1 }],
    value: product.price,
    currency: product.currency
  }, { event_id: eventId });
}

export function ttAddToCart(product: { id: string; name: string; price: number; quantity: number; currency: string }, eventId: string) {
  window.ttq?.track('AddToCart', {
    contents: [{ content_id: product.id, content_name: product.name, content_type: 'product', price: product.price, quantity: product.quantity }],
    value: product.price * product.quantity,
    currency: product.currency
  }, { event_id: eventId });
}

export function ttInitiateCheckout(cart: { items: any[]; value: number; currency: string }, eventId: string) {
  window.ttq?.track('InitiateCheckout', {
    contents: cart.items.map(i => ({ content_id: i.id, content_name: i.name, content_type: 'product', price: i.price, quantity: i.quantity })),
    value: cart.value,
    currency: cart.currency
  }, { event_id: eventId });
}

// This is TikTok's purchase equivalent — note the name.
export function ttCompletePayment(order: {
  paymentIntentId: string;
  value: number;
  currency: string;
  items: { id: string; name: string; price: number; quantity: number }[];
}) {
  window.ttq?.track('CompletePayment', {
    contents: order.items.map(i => ({ content_id: i.id, content_name: i.name, content_type: 'product', price: i.price, quantity: i.quantity })),
    value: order.value,
    currency: order.currency
  }, { event_id: order.paymentIntentId });
}
```

---

## Server-side Events API

```ts
// lib/tracking/tiktok.ts
import crypto from 'node:crypto';

const PIXEL_ID = process.env.NEXT_PUBLIC_TIKTOK_PIXEL_ID!;
const ACCESS_TOKEN = process.env.TIKTOK_EVENTS_API_ACCESS_TOKEN!;
const TEST_EVENT_CODE = process.env.TIKTOK_EVENTS_API_TEST_CODE;
const ENDPOINT = 'https://business-api.tiktok.com/open_api/v1.3/event/track/';

function sha256(v: string) {
  return crypto.createHash('sha256').update(v.trim().toLowerCase()).digest('hex');
}

export type TTEventInput = {
  event: 'CompletePayment' | 'AddToCart' | 'InitiateCheckout' | 'ViewContent' | 'PlaceAnOrder' | string;
  event_id: string;
  event_time?: number;
  event_source_url?: string;
  user: {
    email?: string; phone?: string; external_id?: string;
    ip?: string; user_agent?: string; ttclid?: string; ttp?: string;
  };
  properties?: {
    currency?: string; value?: number;
    contents?: { content_id: string; content_name?: string; content_type?: string; price?: number; quantity?: number }[];
    order_id?: string; description?: string;
  };
};

export async function sendTikTokEventsAPI(input: TTEventInput) {
  const context: any = {
    ad: { callback: input.user.ttclid },
    page: { url: input.event_source_url },
    user: {
      email: input.user.email ? sha256(input.user.email) : undefined,
      phone: input.user.phone ? sha256(input.user.phone.replace(/\D/g, '')) : undefined,
      external_id: input.user.external_id ? sha256(input.user.external_id) : undefined,
      ttp: input.user.ttp
    },
    ip: input.user.ip,
    user_agent: input.user.user_agent
  };

  const body: any = {
    event_source: 'web',
    event_source_id: PIXEL_ID,
    ...(TEST_EVENT_CODE && { test_event_code: TEST_EVENT_CODE }),
    data: [{
      event: input.event,
      event_time: input.event_time ?? Math.floor(Date.now() / 1000),
      event_id: input.event_id,
      user: context.user,
      properties: input.properties,
      page: context.page,
      ad: context.ad,
      ip: context.ip,
      user_agent: context.user_agent
    }]
  };

  const res = await fetch(ENDPOINT, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Access-Token': ACCESS_TOKEN
    },
    body: JSON.stringify(body)
  });

  if (!res.ok) throw new Error(`TikTok Events API ${res.status}: ${await res.text()}`);
  return res.json();
}
```

### Calling it from the Stripe webhook

```ts
await sendTikTokEventsAPI({
  event: 'CompletePayment',
  event_id: eventId,   // === payment_intent.id
  event_source_url: `${process.env.NEXT_PUBLIC_SITE_URL}/success`,
  user: {
    email: session.customer_details?.email ?? undefined,
    phone: session.customer_details?.phone ?? undefined,
    external_id: session.customer as string,
    ip: session.metadata?.client_ip,
    user_agent: session.metadata?.client_ua,
    ttclid: session.metadata?.ttclid
  },
  properties: {
    currency: session.currency?.toUpperCase(),
    value: (session.amount_total ?? 0) / 100,
    contents: lineItems.map(li => ({ content_id: li.price?.product as string, content_type: 'product', quantity: li.quantity ?? 1, price: (li.price?.unit_amount ?? 0) / 100 })),
    order_id: eventId
  }
});
```

---

## Environment variables

```env
NEXT_PUBLIC_TIKTOK_PIXEL_ID=C1234567890ABCDEF
TIKTOK_EVENTS_API_ACCESS_TOKEN=abc123...
TIKTOK_EVENTS_API_TEST_CODE=TEST12345      # leave blank in production
```

---

## Capture `ttclid` from URL and persist

Add this to your `persistClickIds()` function (see `references/event-id-and-datalayer.md`):

```js
// On every page load
const params = new URLSearchParams(window.location.search);
if (params.get('ttclid')) {
  document.cookie = `ttclid=${params.get('ttclid')};path=/;max-age=${60*60*24*90};samesite=lax`;
}
```

Then when creating the Stripe Checkout Session, pull it from the cookie and pass it in `metadata.ttclid`.

---

## Beyond code (TikTok Business Center + Ads Manager)

- **TikTok Ads Manager → Assets → Events → Web Events → [Pixel] → Set Up Events API** — copy the access token.
- **Test Events tool** — Ads Manager → Assets → Events → Web Events → [Pixel] → Test Events → generate test code → paste into `TIKTOK_EVENTS_API_TEST_CODE`.
- Match `event_name` between Pixel and Events API exactly — case-sensitive.

---

## Common failure modes

- **Using "Purchase" instead of "CompletePayment".** TikTok won't recognize it as a purchase event; optimization suffers.
- **Missing `event_id` on Pixel firing.** No dedup, everything doubles.
- **`ttclid` not captured.** Match Quality drops significantly — TikTok can't tie the conversion to the click.
- **Consent handling.** TikTok Pixel supports `holdConsent()` / `grantConsent()` methods — use them if you have a CMP.

---

## Testing

1. Set `TIKTOK_EVENTS_API_TEST_CODE`, make a test purchase.
2. TikTok Ads Manager → Events → Test Events shows both Pixel and Server events with matching IDs.
3. Deduplication rate should reach 90%+ after a few real conversions.
4. Remove `TIKTOK_EVENTS_API_TEST_CODE` from production.

---

## Docs

- Events API: https://ads.tiktok.com/help/article/events-api
- Get started: https://ads.tiktok.com/help/article/getting-started-events-api
- Standard events: https://ads.tiktok.com/help/article/standard-events-parameters
- Deduplication: https://ads.tiktok.com/help/article/event-deduplication
