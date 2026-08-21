# Reddit Pixel + Conversions API

Reddit's audience skews privacy-heavy and ad-blocker-heavy — browser pixel loss is materially higher on Reddit traffic than on other channels. **Server-led CAPI setup is especially high-leverage on Reddit.** In many cases you can run CAPI-only (browser Pixel optional) and still get complete tracking.

Reddit's setup trap: **the Pixel ID and the Conversion Access Token are two different credentials from two different places in Reddit Ads Manager, and the token is shown only once.**

---

## Standard events

`PageVisit`, `ViewContent`, `Search`, `AddToCart`, `AddToWishlist`, `Purchase`, `Lead`, `SignUp`, `ViewCategory`, `Custom`.

---

## Audit checklist

- [ ] Pixel ID stored in env var (from Events Manager)
- [ ] Conversion Access Token stored in env var (from Events Manager → Conversions API section — shown only once, copy immediately)
- [ ] Reddit Pixel installed on every page (optional if going CAPI-only)
- [ ] All ecom events firing via CAPI at minimum
- [ ] Shared conversion ID between Pixel and CAPI (use Order ID for Purchase — 1-hour dedup window)
- [ ] Match keys sent: at least one of — `click_id` (`rdt_cid`), email (hashed), IP + user_agent + screen_dimensions, IDFA, AAID
- [ ] `rdt_cid` captured from URL and persisted
- [ ] Test events verified in Events Manager

---

## Install the Pixel (optional if going CAPI-only)

```tsx
// app/layout.tsx
import Script from 'next/script';

const RDT_PIXEL_ID = process.env.NEXT_PUBLIC_REDDIT_PIXEL_ID!;

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <Script id="reddit-pixel" strategy="afterInteractive">
          {`
            !function(w,d){if(!w.rdt){var p=w.rdt=function(){p.sendEvent?p.sendEvent.apply(p,arguments):p.callQueue.push(arguments)};p.callQueue=[];var t=d.createElement("script");t.src="https://www.redditstatic.com/ads/pixel.js",t.async=!0;var s=d.getElementsByTagName("script")[0];s.parentNode.insertBefore(t,s)}}(window,document);
            rdt('init','${RDT_PIXEL_ID}');
            rdt('track', 'PageVisit');
          `}
        </Script>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## Fire browser events

```ts
// lib/tracking/reddit.ts
declare global { interface Window { rdt: any } }

export function rdtPurchase(order: {
  paymentIntentId: string;
  value: number;
  currency: string;
  items: { id: string; name: string; price: number; quantity: number }[];
}) {
  window.rdt?.('track', 'Purchase', {
    conversionId: order.paymentIntentId,     // dedup key
    value: order.value,
    currency: order.currency,
    itemCount: order.items.reduce((s, i) => s + i.quantity, 0),
    products: order.items.map(i => ({ id: i.id, name: i.name, category: 'ecom' }))
  });
}
```

---

## Server-side CAPI

```ts
// lib/tracking/reddit.ts (server)
import crypto from 'node:crypto';

const PIXEL_ID = process.env.NEXT_PUBLIC_REDDIT_PIXEL_ID!;
const ACCESS_TOKEN = process.env.REDDIT_CAPI_ACCESS_TOKEN!;
const ENDPOINT = `https://ads-api.reddit.com/api/v2.0/conversions/events/${PIXEL_ID}`;

function sha256(v: string) {
  return crypto.createHash('sha256').update(v.trim().toLowerCase()).digest('hex');
}

export async function sendRedditCAPI(input: {
  event_type: 'Purchase' | 'AddToCart' | 'ViewContent' | 'Lead' | 'SignUp' | string;
  conversion_id: string;
  event_at?: string;    // ISO 8601
  user: {
    email?: string; external_id?: string;
    ip?: string; user_agent?: string; click_id?: string;
    screen_width?: number; screen_height?: number;
    idfa?: string; aaid?: string;
  };
  event_metadata?: {
    currency?: string;
    value_decimal?: number;
    item_count?: number;
    products?: { id: string; name?: string; category?: string }[];
  };
}) {
  const user: any = {
    email: input.user.email ? sha256(input.user.email) : undefined,
    external_id: input.user.external_id ? sha256(input.user.external_id) : undefined,
    ip_address: input.user.ip,
    user_agent: input.user.user_agent,
    aaid: input.user.aaid,
    idfa: input.user.idfa,
    screen_dimensions: input.user.screen_width ? { width: input.user.screen_width, height: input.user.screen_height } : undefined
  };

  const body = {
    events: [{
      event_at: input.event_at ?? new Date().toISOString(),
      event_type: { tracking_type: input.event_type },
      click_id: input.user.click_id,
      user,
      event_metadata: input.event_metadata,
      conversion_id: input.conversion_id
    }]
  };

  const res = await fetch(ENDPOINT, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${ACCESS_TOKEN}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(body)
  });

  if (!res.ok) throw new Error(`Reddit CAPI ${res.status}: ${await res.text()}`);
  return res.json();
}
```

### Calling from the Stripe webhook

```ts
await sendRedditCAPI({
  event_type: 'Purchase',
  conversion_id: eventId,   // === payment_intent.id
  user: {
    email: session.customer_details?.email ?? undefined,
    external_id: session.customer as string,
    ip: session.metadata?.client_ip,
    user_agent: session.metadata?.client_ua,
    click_id: session.metadata?.rdt_cid
  },
  event_metadata: {
    currency: session.currency?.toUpperCase(),
    value_decimal: (session.amount_total ?? 0) / 100,
    item_count: lineItems.reduce((s, li) => s + (li.quantity ?? 1), 0),
    products: lineItems.map(li => ({ id: li.price?.product as string, name: li.description ?? undefined }))
  }
});
```

---

## Environment variables

```env
NEXT_PUBLIC_REDDIT_PIXEL_ID=a2_abc123xyz
REDDIT_CAPI_ACCESS_TOKEN=eyJ...
```

---

## Capture `rdt_cid`

Add to your `persistClickIds()`:

```js
const params = new URLSearchParams(window.location.search);
if (params.get('rdt_cid')) {
  document.cookie = `rdt_cid=${params.get('rdt_cid')};path=/;max-age=${60*60*24*90};samesite=lax`;
}
```

---

## Beyond code (Reddit Ads Manager)

- **Ads Manager → Events Manager → Create Pixel** — copy the Pixel ID.
- **Ads Manager → Events Manager → [Pixel] → Conversions API → Generate Access Token** — copy immediately; shown once, non-expiring.
- **Test Events** — Ads Manager → Events Manager → [Pixel] → Test Events → make a test purchase → verify event arrives.

---

## Common failure modes

- **Reddit's token is shown only once — losing it means generating a new one and updating everywhere.**
- **Missing conversion_id.** Reddit merges duplicates within 1 hour based on conversion_id. Without it, every event counts separately.
- **Only browser Pixel, no CAPI.** Reddit's privacy-conscious audience blocks it disproportionately. Server-side recovery is high-leverage.
- **Match keys too sparse.** At least one of `click_id`, `email`, `IP + UA + screen_dims`, `IDFA`, `AAID` is required.

---

## Testing

1. **Ads Manager → Events → Test Events** — should show real-time events with `conversion_id`.
2. Trigger both Pixel and CAPI Purchase with same `conversion_id` — verify only one conversion appears (deduped within 1-hour window).
3. Look at deduplication rate in Events Manager after 24h of live traffic.

---

## Docs

- Conversions API: https://ads-api.reddit.com/docs/v2/#tag/Conversions
- Setup guide: https://business.reddithelp.com/s/article/set-up-conversions-api
- Deduplication: https://business.reddithelp.com/s/article/link-standard-events-to-conversion-events
- Event Metadata reference: https://ads-api.reddit.com/docs/v2/#tag/Conversions
