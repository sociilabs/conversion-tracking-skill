# The Unified `event_id` and dataLayer Pattern

**Read this first.** Every other reference file in this skill assumes this pattern is in place.

The single highest-leverage architectural decision in the whole tracking stack is: **generate one `event_id` per business event, at the moment the event happens, and pass it through every downstream tag** — browser pixels, server-side APIs, and Stripe session metadata alike. Get this right and deduplication works automatically across all seven ad platforms + GA4. Get it wrong and every purchase silently double-counts on every platform, campaign optimization learns from fiction, and you can't tell whether tracking is broken or the ads just aren't working.

---

## The rule

**One `event_id` per business event.** Not per user. Not per session. Per event.

- Pre-purchase events (`view_item`, `add_to_cart`, `begin_checkout`, `add_shipping_info`, `add_payment_info`) → generate a UUID once at the moment of the event, store in dataLayer, reuse across every tag firing for that same interaction.
- **Purchase → use the order ID** (or Stripe `payment_intent_id` — whichever is more stable). It's naturally unique per transaction, meaningful to the business, and lets your database and Stripe use the same key.

---

## Dedup mechanics across platforms — a summary table

| Platform | Field name | Dedup window | Winner |
|---|---|---|---|
| Meta Pixel / CAPI | `event_id` (Pixel `eventID`) | ~2 hours | Browser (Pixel) if it arrives first |
| GA4 | `transaction_id` (purchase only) | Session-level | Dedupes within session |
| Google Ads | `transaction_id` (browser) + same in Enhanced Conversions payload | — | Associates the two |
| TikTok | `event_id` | 48 hours | First received |
| LinkedIn | `eventId` | Same day | Insight Tag over CAPI |
| Pinterest | `event_id` (`dedup_key` in some docs) | Same day | First received |
| Reddit | Conversion ID | 1 hour | Merged |
| Stripe | `metadata.event_id` | (bridges browser session to webhook) | — |

Notice: the *field name* varies slightly, but the *value* should be identical. Get the field-name mapping right in each platform's tag config; the value is always the same string.

---

## The dataLayer contract

Every ecom event on the site should push to a single dataLayer with a consistent shape. Every ad platform tag reads from this one place. Change once, deployed everywhere.

### View item

```js
dataLayer.push({
  event: 'view_item',
  event_id: crypto.randomUUID(),
  ecommerce: {
    currency: 'USD',
    value: 79.99,
    items: [{
      item_id: 'SKU-001',
      item_name: 'Widget',
      item_brand: 'YourBrand',
      item_category: 'Widgets',
      price: 79.99,
      quantity: 1
    }]
  }
});
```

### Add to cart

```js
dataLayer.push({
  event: 'add_to_cart',
  event_id: crypto.randomUUID(),
  ecommerce: {
    currency: 'USD',
    value: 79.99,
    items: [{ item_id: 'SKU-001', item_name: 'Widget', price: 79.99, quantity: 1 }]
  }
});
```

### Begin checkout

```js
dataLayer.push({
  event: 'begin_checkout',
  event_id: crypto.randomUUID(),
  ecommerce: {
    currency: 'USD',
    value: 129.99,
    coupon: 'SUMMER25',
    items: [...]
  }
});
```

### Purchase (this is the important one)

```js
dataLayer.push({
  event: 'purchase',
  event_id: 'ORDER-12345',    // <-- REUSE the order ID here
  ecommerce: {
    transaction_id: 'ORDER-12345',    // <-- same value
    currency: 'USD',
    value: 129.99,
    tax: 10.40,
    shipping: 5.00,
    coupon: 'SUMMER25',
    items: [{
      item_id: 'SKU-001',
      item_name: 'Widget',
      item_brand: 'YourBrand',
      item_category: 'Widgets',
      price: 129.99,
      quantity: 1
    }]
  },
  user_data: {
    email: 'user@example.com',   // hashed downstream by each platform
    phone: '+15551234567',
    first_name: 'John',
    last_name: 'Doe',
    city: 'Lahore',
    country: 'PK',
    external_id: 'CUSTOMER-789'
  },
  attribution: {
    // Read from cookies at push time — these are click IDs from ad platforms
    fbp: getCookie('_fbp'),
    fbc: getCookie('_fbc'),   // or from ?fbclid= on landing
    gclid: getCookie('gclid'),
    gbraid: getCookie('gbraid'),
    wbraid: getCookie('wbraid'),
    ttclid: getCookie('ttclid'),
    li_fat_id: getCookie('li_fat_id'),
    rdt_cid: getCookie('rdt_cid'),
    ga_client_id: getGACID()   // from _ga cookie
  }
});
```

---

## Server-side: the same `event_id` on the webhook

When the Stripe webhook fires for `checkout.session.completed`, your server sends duplicate copies of `Purchase` to Meta CAPI, TikTok Events API, LinkedIn CAPI, etc. **Every one of those uses the same order ID as `event_id`.** Because it matches the browser event fired on the success page, all platforms dedupe and count the purchase exactly once.

The bridge is Stripe session **metadata** — set it when creating the session, read it back on the webhook.

### Setting metadata when creating the Checkout Session

```ts
// app/api/checkout/route.ts (Next.js App Router example)
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2026-07-29.dahlia' });

export async function POST(req: Request) {
  const body = await req.json();
  const { items, attribution } = body;

  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: items,
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/cancel`,
    metadata: {
      // Attribution — populated from the browser before /api/checkout is called
      utm_source: attribution.utm_source ?? '',
      utm_medium: attribution.utm_medium ?? '',
      utm_campaign: attribution.utm_campaign ?? '',
      utm_content: attribution.utm_content ?? '',
      utm_term: attribution.utm_term ?? '',
      ga_client_id: attribution.ga_client_id ?? '',
      fbp: attribution.fbp ?? '',
      fbc: attribution.fbc ?? '',
      gclid: attribution.gclid ?? '',
      ttclid: attribution.ttclid ?? '',
      li_fat_id: attribution.li_fat_id ?? '',
      rdt_cid: attribution.rdt_cid ?? '',
      // The final event_id will be the order ID, generated after payment.
      // But we can also pass a pre-purchase ID here if needed for cross-session dedup.
      pre_purchase_event_id: attribution.event_id ?? ''
    }
  });

  return Response.json({ url: session.url });
}
```

### Reading metadata + firing server-side conversions on the webhook

```ts
// app/api/webhooks/stripe/route.ts (Next.js App Router example)
import Stripe from 'stripe';
import { sendMetaCAPI } from '@/lib/tracking/meta';
import { sendGA4MP } from '@/lib/tracking/ga4';
import { sendTikTokEventsAPI } from '@/lib/tracking/tiktok';
// ... etc.

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2026-07-29.dahlia' });

export async function POST(req: Request) {
  const body = await req.text();
  const sig = req.headers.get('stripe-signature')!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return new Response(`Webhook Error: ${(err as Error).message}`, { status: 400 });
  }

  // Idempotency: dedupe on event.id (Stripe retries)
  if (await webhookAlreadyProcessed(event.id)) {
    return new Response(null, { status: 200 });
  }
  await markWebhookProcessed(event.id);

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = await stripe.checkout.sessions.retrieve(
        (event.data.object as Stripe.Checkout.Session).id,
        { expand: ['line_items', 'customer_details', 'payment_intent'] }
      );

      // THE unified event_id for this purchase. Use payment_intent.id — stable, unique, exists after payment.
      const eventId = typeof session.payment_intent === 'string'
        ? session.payment_intent
        : session.payment_intent?.id;

      const purchaseData = {
        event_id: eventId,
        transaction_id: eventId,
        value: (session.amount_total ?? 0) / 100,
        currency: (session.currency ?? 'usd').toUpperCase(),
        coupon: session.metadata?.coupon,
        items: session.line_items?.data.map(li => ({
          item_id: li.price?.product as string,
          item_name: li.description ?? '',
          price: (li.amount_unit_amount ?? li.price?.unit_amount ?? 0) / 100,
          quantity: li.quantity ?? 1
        })) ?? [],
        user: {
          email: session.customer_details?.email,
          phone: session.customer_details?.phone,
          first_name: session.customer_details?.name?.split(' ')[0],
          last_name: session.customer_details?.name?.split(' ').slice(1).join(' '),
          city: session.customer_details?.address?.city,
          country: session.customer_details?.address?.country,
          ip: session.metadata?.client_ip,
          user_agent: session.metadata?.client_ua
        },
        attribution: {
          fbp: session.metadata?.fbp,
          fbc: session.metadata?.fbc,
          gclid: session.metadata?.gclid,
          ttclid: session.metadata?.ttclid,
          li_fat_id: session.metadata?.li_fat_id,
          rdt_cid: session.metadata?.rdt_cid,
          ga_client_id: session.metadata?.ga_client_id
        }
      };

      // Fan out to all platforms in parallel — same event_id everywhere.
      await Promise.allSettled([
        sendMetaCAPI(purchaseData),
        sendGA4MP(purchaseData),
        sendTikTokEventsAPI(purchaseData),
        // sendLinkedInCAPI(purchaseData),   // if in scope
        // sendPinterestCAPI(purchaseData),  // if in scope
        // sendRedditCAPI(purchaseData)      // if in scope
      ]);

      break;
    }

    case 'charge.refunded': {
      // Fire GA4 refund + platform refund events with the same event_id
      break;
    }
  }

  return new Response(null, { status: 200 });
}
```

---

## Common failure modes

### 1. Different `event_id` in browser and server
The most common failure. Someone generates a fresh UUID on the browser purchase, and a fresh UUID on the server webhook. Both platforms count both. **Fix:** use `payment_intent.id` (or order ID) as the event_id in both places.

### 2. Reusing `event_id` across separate purchases
Two customers buy on the same day; devs use `Date.now()` as event_id; two Purchase events collide; one gets suppressed as a duplicate. **Fix:** always use per-event identifiers, never per-user or per-day.

### 3. Browser event fires *before* the ID is known
On the success page, someone fires the Pixel with `eventID: 'purchase-' + Date.now()` and then the server webhook fires with `event_id: session.payment_intent`. Two different values. **Fix:** on the success page, read the session ID from the URL, hit an API route to look up the `payment_intent.id`, use *that* as the eventID on the Pixel.

### 4. Server event arrives before the browser event
Meta expects browser first, then server, within the dedup window. If the server event arrives first (unlikely for purchases, common for lead-gen events that fire from CRM), Meta may count both. **Fix:** for cases where the server is the only reliable trigger, don't fire the browser event at all — server-only is fine.

### 5. `event_id` value differs by case, whitespace, or encoding
`Order-12345` and `order-12345` and `ORDER-12345 ` are three different strings to the dedup engine. **Fix:** always normalize (`.trim().toUpperCase()`) before sending. Pick a convention and stick to it.

### 6. Not sending `event_id` on the Pixel because "the Pixel handles it"
Meta Pixel's `fbq('track', 'Purchase', ..., { eventID: '...' })` — the third argument is optional and easy to forget. Without it, the browser event has no ID and dedup is impossible. **Fix:** always pass the third argument for events that also fire from CAPI.

---

## Where to get identifiers in the browser

```js
// Get UTM params from URL, store in first-party cookie for later attribution
function persistUTMs() {
  const params = new URLSearchParams(window.location.search);
  const utm = {};
  ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term'].forEach(k => {
    if (params.get(k)) utm[k] = params.get(k);
  });
  if (Object.keys(utm).length) {
    document.cookie = `_utm=${encodeURIComponent(JSON.stringify(utm))};path=/;max-age=${60*60*24*90};samesite=lax`;
  }
}
persistUTMs();

// Grab click IDs from URL, persist to cookies
function persistClickIds() {
  const params = new URLSearchParams(window.location.search);
  const CLICK_IDS = { fbclid: '_fbc', gclid: 'gclid', gbraid: 'gbraid', wbraid: 'wbraid', ttclid: 'ttclid', li_fat_id: 'li_fat_id', rdt_cid: 'rdt_cid' };
  Object.entries(CLICK_IDS).forEach(([urlKey, cookieName]) => {
    const val = params.get(urlKey);
    if (val) {
      // Meta's _fbc has a specific format: fb.1.<timestamp>.<fbclid>
      const cookieVal = urlKey === 'fbclid' ? `fb.1.${Date.now()}.${val}` : val;
      document.cookie = `${cookieName}=${cookieVal};path=/;max-age=${60*60*24*90};samesite=lax`;
    }
  });
}
persistClickIds();

// Read a cookie
function getCookie(name) {
  return document.cookie.split('; ').find(c => c.startsWith(name + '='))?.split('=')[1];
}

// Get GA4 client_id from _ga cookie
function getGACID() {
  const ga = getCookie('_ga');
  return ga ? ga.split('.').slice(-2).join('.') : null;
}
```

Fire `persistUTMs()` and `persistClickIds()` on every page load, at the top of your root layout. Read the cookies when creating the Stripe Checkout Session so the attribution rides through.

---

## The unified send-to-platform library skeleton

Instead of repeating platform-specific logic all over the code, centralize into `lib/tracking/`:

```
lib/tracking/
├── event-id.ts        # generateEventId(), for pre-purchase events
├── hash.ts            # SHA-256 helper (Node crypto or Web Crypto API)
├── datalayer.ts       # trackViewItem(), trackAddToCart(), etc. — the browser-side pushes
├── meta.ts            # sendMetaCAPI(purchaseData)
├── ga4.ts             # sendGA4MP(purchaseData)
├── google-ads.ts      # sendGoogleAdsConversion(purchaseData)
├── tiktok.ts          # sendTikTokEventsAPI(purchaseData)
├── linkedin.ts        # sendLinkedInCAPI(purchaseData)
├── pinterest.ts       # sendPinterestCAPI(purchaseData)
└── reddit.ts          # sendRedditCAPI(purchaseData)
```

Each platform module exports one function that takes the unified `purchaseData` object and does whatever platform-specific transformation is required. Reference files 1–7 each show what that transformation looks like.

---

## Docs / references

- Meta: https://developers.facebook.com/docs/marketing-api/conversions-api/deduplicate-pixel-and-server-events/
- TikTok: https://ads.tiktok.com/help/article/event-deduplication
- LinkedIn: https://learn.microsoft.com/en-us/linkedin/marketing/conversions/deduplication
- Pinterest: https://help.pinterest.com/en/business/article/validating-your-set-up-with-event-testing
- Reddit: https://business.reddithelp.com/s/article/link-standard-events-to-conversion-events
- Stripe: https://docs.stripe.com/webhooks
