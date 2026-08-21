# Meta Pixel + Conversions API

The 2026 default: **run both, deduplicate on `event_id`, monitor Event Match Quality.** Pixel-only setups miss 30–50% of purchases on iOS + ad-blocker traffic. Meta shipped 1-click CAPI on April 15, 2026 which resets the floor to $0 for standard events, but for anything custom (subscription renewals, `predicted_ltv`, cross-platform routing) you still want a direct integration.

---

## Audit checklist

- [ ] **Pixel installed** on every page — base code fires on all pageviews
- [ ] Pixel ID stored in an env var (not hardcoded)
- [ ] **CAPI access token** generated (Business Settings → Data Sources → Pixel → Generate access token)
- [ ] **Domain verified** in Meta Business Manager
- [ ] All ecom events firing on the Pixel: `ViewContent`, `AddToCart`, `InitiateCheckout`, `AddPaymentInfo`, `Purchase` (at minimum)
- [ ] All the same events also firing via CAPI from the server
- [ ] **`event_id` matches** between Pixel and CAPI for the same conversion
- [ ] **`event_name` matches** (case-sensitive — `Purchase`, not `purchase`)
- [ ] User data hashed (SHA-256) before sending to CAPI: `em` (email), `ph` (phone), `fn` (first name), `ln` (last name), `ct` (city), `st` (state), `zp` (zip), `country`
- [ ] Non-hashed context sent: `client_ip_address`, `client_user_agent`, `fbc` (from `_fbc` cookie or `fbclid`), `fbp` (from `_fbp` cookie)
- [ ] `action_source: 'website'` set on every server event
- [ ] `event_source_url` set to the actual page URL
- [ ] **Event Match Quality (EMQ) score ≥ 7/10** in Events Manager
- [ ] **Deduplication rate > 90%** in Events Manager
- [ ] Test Events tool verifies both Pixel and Server events for the same conversion
- [ ] Consent-gated — events don't fire until the user grants consent (EEA/UK)
- [ ] Marketing API version pinned to a current version (e.g., v25.0) — not deprecated

---

## Base Pixel install (Next.js App Router example)

```tsx
// app/layout.tsx
import Script from 'next/script';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        {/* Consent Mode v2 default state — MUST fire before Meta Pixel loads if EEA/UK */}
        <Script id="consent-default" strategy="beforeInteractive">
          {`
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('consent', 'default', {
              ad_storage: 'denied',
              analytics_storage: 'denied',
              ad_user_data: 'denied',
              ad_personalization: 'denied',
              wait_for_update: 500,
              region: ['EEA', 'GB', 'CH']
            });
          `}
        </Script>

        {/* Meta Pixel */}
        <Script id="meta-pixel" strategy="afterInteractive">
          {`
            !function(f,b,e,v,n,t,s)
            {if(f.fbq)return;n=f.fbq=function(){n.callMethod?
            n.callMethod.apply(n,arguments):n.queue.push(arguments)};
            if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
            n.queue=[];t=b.createElement(e);t.async=!0;
            t.src=v;s=b.getElementsByTagName(e)[0];
            s.parentNode.insertBefore(t,s)}(window, document,'script',
            'https://connect.facebook.net/en_US/fbevents.js');
            fbq('init', '${process.env.NEXT_PUBLIC_META_PIXEL_ID}');
            fbq('track', 'PageView');
          `}
        </Script>
        <noscript>
          <img
            height="1" width="1" style={{ display: 'none' }}
            src={`https://www.facebook.com/tr?id=${process.env.NEXT_PUBLIC_META_PIXEL_ID}&ev=PageView&noscript=1`}
            alt=""
          />
        </noscript>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## Firing standard events with `event_id`

```ts
// lib/tracking/meta-pixel.ts (browser-side wrappers)
declare global { interface Window { fbq: any } }

export function trackViewItem(product: { id: string; name: string; category?: string; price: number; currency: string }, eventId: string) {
  window.fbq?.('track', 'ViewContent', {
    content_type: 'product',
    content_ids: [product.id],
    content_name: product.name,
    content_category: product.category,
    value: product.price,
    currency: product.currency
  }, { eventID: eventId });
}

export function trackAddToCart(product: { id: string; name: string; price: number; currency: string; quantity: number }, eventId: string) {
  window.fbq?.('track', 'AddToCart', {
    content_type: 'product',
    content_ids: [product.id],
    content_name: product.name,
    value: product.price * product.quantity,
    currency: product.currency
  }, { eventID: eventId });
}

export function trackInitiateCheckout(cart: { items: any[]; value: number; currency: string }, eventId: string) {
  window.fbq?.('track', 'InitiateCheckout', {
    content_type: 'product',
    content_ids: cart.items.map(i => i.id),
    num_items: cart.items.reduce((s, i) => s + i.quantity, 0),
    value: cart.value,
    currency: cart.currency
  }, { eventID: eventId });
}

// This one fires on the success page after redirect from Stripe.
// eventId MUST equal the payment_intent.id that the server also uses.
export function trackPurchase(order: {
  paymentIntentId: string;
  value: number;
  currency: string;
  items: { id: string; name: string; quantity: number }[];
}) {
  window.fbq?.('track', 'Purchase', {
    content_type: 'product',
    content_ids: order.items.map(i => i.id),
    contents: order.items.map(i => ({ id: i.id, quantity: i.quantity })),
    num_items: order.items.reduce((s, i) => s + i.quantity, 0),
    value: order.value,
    currency: order.currency
  }, { eventID: order.paymentIntentId });
}
```

---

## Server-side CAPI (Node/Next.js API route example)

```ts
// lib/tracking/meta.ts
import crypto from 'node:crypto';

const PIXEL_ID = process.env.NEXT_PUBLIC_META_PIXEL_ID!;
const ACCESS_TOKEN = process.env.META_CAPI_ACCESS_TOKEN!;
const TEST_EVENT_CODE = process.env.META_CAPI_TEST_EVENT_CODE; // undefined in prod
const GRAPH_VERSION = 'v25.0';

function sha256(v: string) {
  return crypto.createHash('sha256').update(v.trim().toLowerCase()).digest('hex');
}

// Only hash if not already 64-char hex
function hashIf(v?: string | null) {
  if (!v) return undefined;
  return /^[a-f0-9]{64}$/i.test(v) ? v : sha256(v);
}

export type MetaEventInput = {
  event_name: 'Purchase' | 'AddToCart' | 'InitiateCheckout' | 'ViewContent' | 'Lead' | 'CompleteRegistration' | string;
  event_id: string;
  event_time?: number;                // unix seconds; defaults to now
  event_source_url?: string;
  user: {
    email?: string; phone?: string;
    first_name?: string; last_name?: string;
    city?: string; state?: string; zip?: string; country?: string;
    external_id?: string;
    ip?: string; user_agent?: string;
    fbp?: string; fbc?: string;
  };
  custom_data?: Record<string, any>;   // { value, currency, content_ids, contents, num_items, coupon, ... }
};

export async function sendMetaCAPI(input: MetaEventInput) {
  const user_data: Record<string, any> = {
    em: hashIf(input.user.email),
    ph: hashIf(input.user.phone?.replace(/\D/g, '')),
    fn: hashIf(input.user.first_name),
    ln: hashIf(input.user.last_name),
    ct: hashIf(input.user.city),
    st: hashIf(input.user.state),
    zp: hashIf(input.user.zip),
    country: hashIf(input.user.country?.toLowerCase()),
    external_id: hashIf(input.user.external_id),
    client_ip_address: input.user.ip,
    client_user_agent: input.user.user_agent,
    fbp: input.user.fbp,
    fbc: input.user.fbc
  };

  // Meta rejects undefined fields — clean them up
  Object.keys(user_data).forEach(k => user_data[k] === undefined && delete user_data[k]);

  const body: any = {
    data: [{
      event_name: input.event_name,
      event_time: input.event_time ?? Math.floor(Date.now() / 1000),
      event_id: input.event_id,
      event_source_url: input.event_source_url,
      action_source: 'website',
      user_data,
      custom_data: input.custom_data
    }]
  };

  if (TEST_EVENT_CODE) body.test_event_code = TEST_EVENT_CODE;

  const url = `https://graph.facebook.com/${GRAPH_VERSION}/${PIXEL_ID}/events?access_token=${ACCESS_TOKEN}`;
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  });

  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Meta CAPI ${res.status}: ${err}`);
  }
  return res.json();
}
```

### Calling it from the Stripe webhook

```ts
// Inside the checkout.session.completed handler in app/api/webhooks/stripe/route.ts
await sendMetaCAPI({
  event_name: 'Purchase',
  event_id: eventId,      // === payment_intent.id, same value the success page's Pixel used
  event_source_url: `${process.env.NEXT_PUBLIC_SITE_URL}/success`,
  user: {
    email: session.customer_details?.email ?? undefined,
    phone: session.customer_details?.phone ?? undefined,
    first_name: session.customer_details?.name?.split(' ')[0],
    last_name: session.customer_details?.name?.split(' ').slice(1).join(' '),
    city: session.customer_details?.address?.city ?? undefined,
    state: session.customer_details?.address?.state ?? undefined,
    zip: session.customer_details?.address?.postal_code ?? undefined,
    country: session.customer_details?.address?.country ?? undefined,
    external_id: session.customer as string,
    ip: session.metadata?.client_ip,
    user_agent: session.metadata?.client_ua,
    fbp: session.metadata?.fbp,
    fbc: session.metadata?.fbc
  },
  custom_data: {
    currency: session.currency?.toUpperCase(),
    value: (session.amount_total ?? 0) / 100,
    content_type: 'product',
    content_ids: lineItems.map(li => li.price?.product as string),
    contents: lineItems.map(li => ({ id: li.price?.product as string, quantity: li.quantity, item_price: (li.price?.unit_amount ?? 0) / 100 })),
    num_items: lineItems.reduce((s, li) => s + (li.quantity ?? 1), 0),
    order_id: eventId,
    coupon: session.total_details?.breakdown?.discounts?.[0]?.discount?.coupon?.id
  }
});
```

---

## Environment variables

```env
NEXT_PUBLIC_META_PIXEL_ID=1234567890
META_CAPI_ACCESS_TOKEN=EAA...
META_CAPI_TEST_EVENT_CODE=TEST12345    # leave blank in production
```

---

## Beyond code (Meta Business Manager)

- **Business Settings → Data Sources → Pixel → Settings → Conversions API → Generate access token.** Copy immediately; you can regenerate but shouldn't need to.
- **Domain Verification:** Business Settings → Brand Safety → Domains → add domain → verify by DNS TXT record or meta tag. Required for iOS 14+ tracking.
- **Aggregated Event Measurement:** Events Manager → Aggregated Event Measurement → Configure the 8 events per domain in priority order (Purchase should be #1 for ecom).
- **Test Events:** Events Manager → [Pixel] → Test Events → generate a test event code → paste it into `META_CAPI_TEST_EVENT_CODE` env var → make a test purchase → verify both browser and server events show up with matching IDs.

---

## Common failure modes

- **Missing `event_id` on Pixel firing.** Third arg to `fbq('track', ...)` is optional — easy to forget. Result: no dedup, everything doubles.
- **Different `event_name` case.** `purchase` ≠ `Purchase`. Case-sensitive.
- **Server event has hashed field with the wrong format.** Emails must be lowercased and trimmed before hashing. Phones must have only digits (no `+`, no spaces).
- **`_fbc` cookie not set from `fbclid`.** Meta expects a specific format: `fb.1.<timestamp>.<fbclid>`. If you just save the raw `fbclid`, EMQ suffers.
- **Sending events on a deprecated Marketing API version.** Meta silently degrades and eventually rejects.
- **In-app browser purchases (Facebook / Instagram in-app):** Pixel is unreliable inside these browsers. CAPI recovers most of it.

---

## Testing

1. Set `META_CAPI_TEST_EVENT_CODE` to a code from Events Manager → Test Events.
2. Make a $1 test purchase in Stripe test mode.
3. Watch Test Events: within seconds, you should see **both** "Browser" and "Server" events for the same Purchase, with matching event IDs. Deduplication indicator should show "Deduplicated".
4. **Remove** `META_CAPI_TEST_EVENT_CODE` from production `.env`.

---

## Docs

- CAPI reference: https://developers.facebook.com/docs/marketing-api/conversions-api
- Deduplication: https://developers.facebook.com/docs/marketing-api/conversions-api/deduplicate-pixel-and-server-events/
- Standard events: https://developers.facebook.com/docs/facebook-pixel/reference
- Server event parameters: https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/customer-information-parameters
