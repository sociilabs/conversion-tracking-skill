# Pinterest Tag + Conversions API

Pinterest's audience skews strongly toward planning purchases (home decor, fashion, food, DIY). Ad blocker use is moderate but growing. Pinterest recommends running Pinterest Tag + CAPI together and dedupes via `event_id`. Dynamic retargeting requires product IDs matching your Pinterest catalog.

---

## Standard events

`page_visit`, `view_category`, `search`, `add_to_cart`, `checkout` (Pinterest's purchase equivalent), `watch_video`, `signup`, `lead`, `custom`. Full list (20 types) at https://help.pinterest.com/en/business/article/track-conversions-with-pinterest-tag.

For ecom: `page_visit`, `view_category`, `add_to_cart`, `checkout`.

---

## Audit checklist

- [ ] Pinterest Tag ID and Ad Account ID stored in env vars
- [ ] Conversion Access Token generated from Pinterest Business Manager
- [ ] Pinterest base tag on every page
- [ ] Event-specific tags fire on the right pages (product page, cart, checkout success)
- [ ] `event_id` matches between Tag and CAPI for same event (use Order ID for `checkout`)
- [ ] Enhanced Match sending hashed email, external_id
- [ ] Product IDs on `view_category` / `add_to_cart` / `checkout` match your Pinterest catalog (for dynamic retargeting)
- [ ] Event Quality Score (EQS) in Events Manager is "Good"
- [ ] Events reach Pinterest within 1 hour of firing (else classified as offline)
- [ ] Test events verified in Ads Manager → Events → Test Events

---

## Install the base Pinterest Tag

```tsx
// app/layout.tsx
import Script from 'next/script';

const PIN_TAG_ID = process.env.NEXT_PUBLIC_PINTEREST_TAG_ID!;

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <Script id="pinterest-tag" strategy="afterInteractive">
          {`
            !function(e){if(!window.pintrk){window.pintrk=function(){window.pintrk.queue.push(Array.prototype.slice.call(arguments))};var
              n=window.pintrk;n.queue=[],n.version="3.0";var
              t=document.createElement("script");t.async=!0,t.src=e;var
              r=document.getElementsByTagName("script")[0];
              r.parentNode.insertBefore(t,r)}}("https://s.pinimg.com/ct/core.js");
            pintrk('load', '${PIN_TAG_ID}');
            pintrk('page');
          `}
        </Script>
        <noscript>
          <img height="1" width="1" style={{ display: 'none' }} alt=""
            src={`https://ct.pinterest.com/v3/?event=init&tid=${PIN_TAG_ID}&noscript=1`} />
        </noscript>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## Fire browser events with `event_id`

```ts
// lib/tracking/pinterest.ts
declare global { interface Window { pintrk: any } }

export function pinAddToCart(product: { id: string; name: string; price: number; quantity: number; currency: string }, eventId: string) {
  window.pintrk?.('track', 'add_to_cart', {
    event_id: eventId,
    value: product.price * product.quantity,
    order_quantity: product.quantity,
    currency: product.currency,
    line_items: [{ product_id: product.id, product_name: product.name, product_price: product.price, product_quantity: product.quantity }]
  });
}

export function pinCheckout(order: {
  paymentIntentId: string;
  value: number;
  currency: string;
  items: { id: string; name: string; price: number; quantity: number }[];
}) {
  window.pintrk?.('track', 'checkout', {
    event_id: order.paymentIntentId,     // <-- Use order/payment_intent ID as dedup key
    value: order.value,
    order_quantity: order.items.reduce((s, i) => s + i.quantity, 0),
    currency: order.currency,
    line_items: order.items.map(i => ({ product_id: i.id, product_name: i.name, product_price: i.price, product_quantity: i.quantity }))
  });
}
```

---

## Server-side CAPI

```ts
// lib/tracking/pinterest.ts (server)
import crypto from 'node:crypto';

const AD_ACCOUNT_ID = process.env.PINTEREST_AD_ACCOUNT_ID!;
const ACCESS_TOKEN = process.env.PINTEREST_CAPI_ACCESS_TOKEN!;
const ENDPOINT = `https://api.pinterest.com/v5/ad_accounts/${AD_ACCOUNT_ID}/events`;

function sha256(v: string) {
  return crypto.createHash('sha256').update(v.trim().toLowerCase()).digest('hex');
}

export async function sendPinterestCAPI(input: {
  event_name: 'checkout' | 'add_to_cart' | 'page_visit' | 'view_category' | 'signup' | 'lead' | string;
  event_id: string;
  event_time?: number;
  event_source_url?: string;
  action_source?: 'web' | 'app_ios' | 'app_android' | 'offline';
  user: { email?: string; phone?: string; external_id?: string; ip?: string; user_agent?: string; click_id?: string };
  custom_data?: {
    currency?: string;
    value?: string;             // must be string per Pinterest schema
    order_id?: string;
    order_quantity?: number;
    contents?: { id: string; item_price: string; quantity: number }[];
    num_items?: number;
    coupon?: string;
  };
}) {
  const body = {
    data: [{
      event_name: input.event_name,
      action_source: input.action_source ?? 'web',
      event_time: input.event_time ?? Math.floor(Date.now() / 1000),
      event_id: input.event_id,
      event_source_url: input.event_source_url,
      user_data: {
        em: input.user.email ? [sha256(input.user.email)] : undefined,
        ph: input.user.phone ? [sha256(input.user.phone.replace(/\D/g, ''))] : undefined,
        external_id: input.user.external_id ? [sha256(input.user.external_id)] : undefined,
        client_ip_address: input.user.ip,
        client_user_agent: input.user.user_agent,
        click_id: input.user.click_id
      },
      custom_data: input.custom_data
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

  if (!res.ok) throw new Error(`Pinterest CAPI ${res.status}: ${await res.text()}`);
  return res.json();
}
```

### Calling from the Stripe webhook

```ts
await sendPinterestCAPI({
  event_name: 'checkout',
  event_id: eventId,   // === payment_intent.id
  event_source_url: `${process.env.NEXT_PUBLIC_SITE_URL}/success`,
  user: {
    email: session.customer_details?.email ?? undefined,
    phone: session.customer_details?.phone ?? undefined,
    external_id: session.customer as string,
    ip: session.metadata?.client_ip,
    user_agent: session.metadata?.client_ua
  },
  custom_data: {
    currency: session.currency?.toUpperCase(),
    value: ((session.amount_total ?? 0) / 100).toFixed(2),
    order_id: eventId,
    order_quantity: lineItems.reduce((s, li) => s + (li.quantity ?? 1), 0),
    contents: lineItems.map(li => ({ id: li.price?.product as string, item_price: ((li.price?.unit_amount ?? 0) / 100).toFixed(2), quantity: li.quantity ?? 1 }))
  }
});
```

---

## Environment variables

```env
NEXT_PUBLIC_PINTEREST_TAG_ID=2612345678901
PINTEREST_AD_ACCOUNT_ID=549123456789012345
PINTEREST_CAPI_ACCESS_TOKEN=pina_XXXXXX...
```

---

## Beyond code (Pinterest Business Manager)

- **Business Manager → Ads Manager → Conversions → Events** — verify tag is receiving data. Use "Test Events" to validate payload before going live.
- **Business Manager → Ads Manager → Conversions → API for Conversions → Generate token** — copy the Conversion Access Token immediately.
- **Business Manager → Catalog** — upload your product catalog with product IDs that match what you send in `contents[].id`. Required for dynamic retargeting to work.
- **Event Quality Score** — Ads Manager → Events → [event] → Event Quality tab. Aim for "Good" or higher (add hashed email, external_id, click_id, and product IDs).
- **Deduplication verification** — Ads Manager → Events → [event] → Deduplication. Confirm `event_id` coverage above the resilient threshold.

---

## Common failure modes

- **Product IDs don't match catalog.** Dynamic retargeting silently doesn't work.
- **Value passed as number instead of string.** Pinterest CAPI expects `value: "129.99"` (string). Numbers may be silently rejected.
- **Missing `action_source`.** Defaults to `web` but explicit is safer.
- **Events arrive > 1 hour after they happened.** Pinterest classifies as "offline" and they don't power real-time retargeting.
- **Only Tag or only CAPI, not both.** You lose the durability that comes from the redundant setup.

---

## Testing

1. **Test Events** — Ads Manager → Conversions → Test Events → verify the event fires with correct `event_id`, currency, value.
2. **Event Diagnostics** — Ads Manager → Events → [event] → Event Diagnostics. Should show recent events with matching IDs from both sources.
3. **Deduplication** — after firing both browser and server events, check the deduplication tab — one conversion should show.

---

## Docs

- Overview: https://help.pinterest.com/en/business/article/the-pinterest-api-for-conversions
- Developer docs: https://developers.pinterest.com/docs/conversions/conversion-management/
- Best practices: https://developers.pinterest.com/docs/conversions/best/
- Testing: https://help.pinterest.com/en/business/article/validating-your-set-up-with-event-testing
