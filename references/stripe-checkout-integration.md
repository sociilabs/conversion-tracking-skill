# Stripe Checkout Integration

Stripe is the payment layer that ties the whole tracking stack together. Your Stripe Checkout Session is where attribution metadata gets stored, and your Stripe webhook is where the source-of-truth Purchase event fires for every ad platform.

**The two-track pattern is the whole ballgame:**
1. **Webhook = source of truth.** Fulfillment, revenue attribution, and analytics-that-must-not-be-missed happen on `checkout.session.completed`. A customer can close the browser after paying and never see the success page — the webhook is the only reliable trigger.
2. **Success page = UX + best-effort client-side analytics.** Fires browser Pixel events, gives the user their receipt, and re-hydrates the ad-platform view. But if browser fails, the webhook has already covered it.

---

## Audit checklist

- [ ] Stripe secret key + publishable key + webhook secret all in env vars
- [ ] Stripe API version pinned (e.g., `2026-07-29.dahlia` or `2026-01-28`)
- [ ] Checkout Session creation passes attribution metadata (UTMs, click IDs, `event_id`)
- [ ] Success URL includes `?session_id={CHECKOUT_SESSION_ID}` placeholder
- [ ] Success page retrieves the Session server-side (with `line_items` expanded) and fires browser conversion events with `payment_intent.id` as the unified `event_id`
- [ ] Webhook endpoint verifies `Stripe-Signature` header with endpoint secret
- [ ] Webhook returns 2xx within 20 seconds
- [ ] Webhook is idempotent: dedupes on `event.id` (Stripe retries for 3 days)
- [ ] Subscribed to: `checkout.session.completed`, `payment_intent.succeeded`, `invoice.paid` (subs), `charge.refunded`, optionally `checkout.session.expired`
- [ ] Webhook fires server-side Purchase to every ad platform CAPI with matching `event_id`
- [ ] Amounts converted from cents (Stripe) to major units (÷100) before sending to platforms

---

## Create the Checkout Session (with attribution metadata)

```ts
// app/api/checkout/route.ts (Next.js App Router)
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2026-07-29.dahlia' });

export async function POST(req: Request) {
  const { items, attribution } = await req.json();

  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: items,   // [{ price: 'price_xxx', quantity: 1 }, ...]
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/cancel`,
    // Attribution rides through here — set once, read on webhook.
    metadata: {
      utm_source: attribution.utm_source ?? '',
      utm_medium: attribution.utm_medium ?? '',
      utm_campaign: attribution.utm_campaign ?? '',
      utm_content: attribution.utm_content ?? '',
      utm_term: attribution.utm_term ?? '',
      ga_client_id: attribution.ga_client_id ?? '',
      fbp: attribution.fbp ?? '',
      fbc: attribution.fbc ?? '',
      gclid: attribution.gclid ?? '',
      gbraid: attribution.gbraid ?? '',
      wbraid: attribution.wbraid ?? '',
      ttclid: attribution.ttclid ?? '',
      li_fat_id: attribution.li_fat_id ?? '',
      rdt_cid: attribution.rdt_cid ?? '',
      client_ip: attribution.client_ip ?? '',
      client_ua: attribution.client_ua ?? '',
      coupon: attribution.coupon ?? ''
    }
  });

  return Response.json({ url: session.url });
}
```

The browser client should call this route and then `window.location.href = session.url` to redirect.

---

## Success page (fires browser conversion events)

```tsx
// app/success/page.tsx
import Stripe from 'stripe';
import { CheckoutSuccess } from './CheckoutSuccess';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2026-07-29.dahlia' });

export default async function SuccessPage({ searchParams }: { searchParams: { session_id?: string } }) {
  if (!searchParams.session_id) return <div>Invalid session.</div>;

  const session = await stripe.checkout.sessions.retrieve(searchParams.session_id, {
    expand: ['line_items', 'customer_details', 'payment_intent']
  });

  // Pass only what the client component needs; do NOT pass the whole session to the browser
  const paymentIntentId = typeof session.payment_intent === 'string' ? session.payment_intent : session.payment_intent?.id;

  return (
    <CheckoutSuccess data={{
      eventId: paymentIntentId ?? searchParams.session_id,
      value: (session.amount_total ?? 0) / 100,
      currency: (session.currency ?? 'usd').toUpperCase(),
      items: session.line_items?.data.map(li => ({
        id: li.price?.product as string,
        name: li.description ?? '',
        price: (li.price?.unit_amount ?? 0) / 100,
        quantity: li.quantity ?? 1
      })) ?? [],
      coupon: session.metadata?.coupon
    }} />
  );
}
```

```tsx
// app/success/CheckoutSuccess.tsx (client component)
'use client';
import { useEffect } from 'react';
import { trackPurchase as metaPurchase } from '@/lib/tracking/meta-pixel';
import { ga4Purchase } from '@/lib/tracking/ga4';
import { ttCompletePayment } from '@/lib/tracking/tiktok-pixel';
import { liPurchase } from '@/lib/tracking/linkedin';
import { pinCheckout } from '@/lib/tracking/pinterest';
import { rdtPurchase } from '@/lib/tracking/reddit';

export function CheckoutSuccess({ data }: { data: any }) {
  useEffect(() => {
    // Dedupe: don't fire twice on refresh
    if (sessionStorage.getItem(`purchase-fired-${data.eventId}`)) return;
    sessionStorage.setItem(`purchase-fired-${data.eventId}`, '1');

    // Fire every ad-platform browser event with the SAME event_id (= payment_intent.id)
    metaPurchase({ paymentIntentId: data.eventId, value: data.value, currency: data.currency, items: data.items });
    ga4Purchase({ transaction_id: data.eventId, value: data.value, currency: data.currency, coupon: data.coupon, items: data.items.map((i: any) => ({ item_id: i.id, item_name: i.name, price: i.price, quantity: i.quantity })) });
    ttCompletePayment({ paymentIntentId: data.eventId, value: data.value, currency: data.currency, items: data.items });
    // liPurchase(data.eventId);        // if LinkedIn is in scope
    // pinCheckout({ paymentIntentId: data.eventId, ... }); // if Pinterest in scope
    // rdtPurchase({ paymentIntentId: data.eventId, ... }); // if Reddit in scope
  }, [data]);

  return (
    <main>
      <h1>Thanks for your order!</h1>
      <p>Order ID: {data.eventId}</p>
    </main>
  );
}
```

---

## Webhook (source-of-truth: verify, dedupe, fan out to all CAPIs)

```ts
// app/api/webhooks/stripe/route.ts
import Stripe from 'stripe';
import { sendMetaCAPI } from '@/lib/tracking/meta';
import { sendGA4MP } from '@/lib/tracking/ga4';
import { sendTikTokEventsAPI } from '@/lib/tracking/tiktok';
// ... imports for LinkedIn / Pinterest / Reddit if in scope

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2026-07-29.dahlia' });

// Simple in-memory dedup for demo — use Redis/DB in production
const processed = new Set<string>();

export async function POST(req: Request) {
  const body = await req.text();
  const sig = req.headers.get('stripe-signature');
  if (!sig) return new Response('No signature', { status: 400 });

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return new Response(`Webhook Error: ${(err as Error).message}`, { status: 400 });
  }

  // Idempotency
  if (processed.has(event.id)) return new Response(null, { status: 200 });
  processed.add(event.id);

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = await stripe.checkout.sessions.retrieve(
        (event.data.object as Stripe.Checkout.Session).id,
        { expand: ['line_items', 'customer_details', 'payment_intent'] }
      );
      const eventId = typeof session.payment_intent === 'string' ? session.payment_intent : session.payment_intent?.id;
      if (!eventId) break;

      const lineItems = session.line_items?.data ?? [];
      const value = (session.amount_total ?? 0) / 100;
      const currency = (session.currency ?? 'usd').toUpperCase();

      // 1. Business fulfillment (email receipt, provisioning, etc.)
      // ...

      // 2. Fire every ad-platform CAPI in parallel with the SAME event_id
      const commonData = {
        session,
        eventId,
        lineItems,
        value,
        currency,
        contentIds: lineItems.map(li => li.price?.product as string),
        contents: lineItems.map(li => ({
          id: li.price?.product as string,
          quantity: li.quantity ?? 1,
          item_price: (li.price?.unit_amount ?? 0) / 100
        }))
      };

      await Promise.allSettled([
        sendMetaCAPI({
          event_name: 'Purchase',
          event_id: eventId,
          user: {
            email: session.customer_details?.email ?? undefined,
            phone: session.customer_details?.phone ?? undefined,
            first_name: session.customer_details?.name?.split(' ')[0],
            last_name: session.customer_details?.name?.split(' ').slice(1).join(' '),
            country: session.customer_details?.address?.country ?? undefined,
            ip: session.metadata?.client_ip,
            user_agent: session.metadata?.client_ua,
            fbp: session.metadata?.fbp,
            fbc: session.metadata?.fbc
          },
          custom_data: { currency, value, content_type: 'product', content_ids: commonData.contentIds, contents: commonData.contents, order_id: eventId, coupon: session.metadata?.coupon }
        }),
        sendGA4MP({
          client_id: session.metadata?.ga_client_id ?? '',
          transaction_id: eventId,
          value, currency,
          coupon: session.metadata?.coupon,
          items: lineItems.map(li => ({ item_id: li.price?.product as string, item_name: li.description ?? '', price: (li.price?.unit_amount ?? 0) / 100, quantity: li.quantity ?? 1 }))
        }),
        sendTikTokEventsAPI({
          event: 'CompletePayment',
          event_id: eventId,
          user: {
            email: session.customer_details?.email ?? undefined,
            phone: session.customer_details?.phone ?? undefined,
            external_id: session.customer as string,
            ip: session.metadata?.client_ip,
            user_agent: session.metadata?.client_ua,
            ttclid: session.metadata?.ttclid
          },
          properties: { currency, value, contents: commonData.contents.map(c => ({ content_id: c.id, content_type: 'product', quantity: c.quantity, price: c.item_price })), order_id: eventId }
        })
        // ... LinkedIn / Pinterest / Reddit if in scope
      ]);

      break;
    }

    case 'invoice.paid': {
      // Subscription renewals — fire Purchase again with the invoice ID as event_id
      break;
    }

    case 'charge.refunded': {
      // Fire GA4 refund + platform refund events, same event_id as original purchase
      break;
    }
  }

  return new Response(null, { status: 200 });
}
```

Notes:
- **Do NOT** use Next.js `app/api/webhooks/stripe/route.ts` request body parsing — call `await req.text()` and pass the raw string to `constructEvent`, or signature verification will fail.
- Store `processed` event IDs in Redis or a database column for real idempotency across server restarts.

---

## Environment variables

```env
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
NEXT_PUBLIC_SITE_URL=https://yoursite.com
```

---

## Beyond code (Stripe Dashboard)

- **Developers → Webhooks → Add endpoint** → URL: `https://yoursite.com/api/webhooks/stripe`
- **Events to send** — subscribe to: `checkout.session.completed`, `payment_intent.succeeded`, `invoice.paid`, `charge.refunded`, `checkout.session.expired`
- **Copy the endpoint secret** (`whsec_...`) into `STRIPE_WEBHOOK_SECRET`
- **API version** — under Developers → API version. Pin explicitly rather than following Stripe's default.

---

## Common failure modes

- **Signature verification fails.** Cause: request body was parsed as JSON before verification. Fix: use `req.text()` (raw string) and pass to `constructEvent`.
- **Webhook takes > 20 seconds.** Stripe times out and retries — you get duplicate processing. Fix: return 2xx immediately, do the CAPI fan-out in a background job.
- **Not idempotent.** Stripe retries for 3 days. Every retry re-fires all ad-platform CAPI events. Fix: check `event.id` against a stored set before processing.
- **Success page purchase fires on refresh.** Fix: `sessionStorage` guard OR `transaction_id` on the event (GA4 dedupes automatically).
- **Amounts sent in cents to ad platforms.** Meta shows `$12,999.00` instead of `$129.99`. Fix: divide by 100 before every send.

---

## Testing

1. **Local webhook forwarding:**
   ```bash
   stripe listen --forward-to localhost:3000/api/webhooks/stripe
   ```
   Copy the `whsec_...` it prints into your local `.env`.
2. **Test purchase:** Use card `4242 4242 4242 4242`, any future date, any CVC, any zip.
3. **Verify:** webhook fires, signature verifies, no duplicates on retry, all ad platform events show up in their respective Test Events tools.

---

## Docs

- Checkout Sessions API: https://docs.stripe.com/api/checkout/sessions
- Fulfill orders: https://docs.stripe.com/checkout/fulfillment
- Custom success page: https://docs.stripe.com/payments/checkout/custom-success-page
- Webhooks reference: https://docs.stripe.com/webhooks
- Analyze conversion funnel: https://docs.stripe.com/payments/checkout/analyze-conversion-funnel
