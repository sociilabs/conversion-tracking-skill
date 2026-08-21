# Coupon & Offer Landing Pages

Two goals sit in tension: **attribution fidelity** (every code trackable to its channel) and **SEO / discoverability** (coupon queries drive real traffic if pages are structured right). This section covers both.

---

## The attribution problem

Most ecom sites use one generic coupon code across all channels. When someone converts, you can't tell if they came from the Instagram ad, the podcast sponsor, the newsletter, or a random deal aggregator that scraped your code. That's a huge attribution hole.

The fix is a two-layer strategy:
1. **Unique codes per channel / per campaign / per creator** — so the code itself is the attribution signal.
2. **UTM parameters preserved through the checkout** — so the source signal survives even if the code doesn't.

Get both right and you can attribute revenue to individual creators, ad sets, and email flows without a CDP.

---

## Audit checklist

- [ ] Each marketing channel gets its own unique discount code (or dynamically generated per-creator/per-recipient code)
- [ ] Codes follow a naming convention: `CHANNEL_CAMPAIGN_YEAR` (e.g., `INSTA_BFCM25`, `PODCAST_LAUNCH24`)
- [ ] Offer landing pages (`/offer/<slug>`) exist for each active campaign
- [ ] UTM parameters captured from URL on landing and **persisted to a first-party cookie** for late attribution
- [ ] Coupon code flows into the GA4 `purchase` event's `coupon` parameter
- [ ] Coupon code flows into Meta CAPI `custom_data.coupon`
- [ ] Coupon code stored in Stripe Checkout Session metadata
- [ ] Newsletter/popup codes have their own UTM tags so they don't override channel attribution
- [ ] Landing pages don't tag their own internal links with UTMs (breaks session attribution in GA4)
- [ ] For public deal sites: separate codes / URLs so they don't cannibalize paid campaign attribution
- [ ] If SEO-heavy: category hub pages indexed, individual code pages `noindex`

---

## Attribution patterns

### Pattern 1: Unique code per channel (simple, most common)

Create a code per campaign/channel/creator. Send each code as `coupon` on the `purchase` event. Attribution follows the code.

```
EMAIL15         → newsletter subscribers
INSTA15         → Instagram organic
INSTA_ADS15     → Instagram paid ads
PODCAST_JOE15   → Joe Rogan podcast sponsor
JANE_TT_BFCM25  → Jane's TikTok, Black Friday campaign
```

Simple. Works everywhere. Doesn't require URL infrastructure.

### Pattern 2: URL-embedded discount + UTM (Shopify-style)

Use a URL that both applies the discount and carries UTMs:

```
https://yoursite.com/discount/SUMMER25?utm_source=meta&utm_medium=cpc&utm_campaign=summer_launch&redirect=/products/widget
```

Shopify supports this natively at `/discount/<code>?...`. For custom sites, build a route that reads the code and UTMs, applies the discount to the cart, and redirects to the product/landing page.

### Pattern 3: Dedicated offer landing page

`/offer/summer-25` — own page per campaign, own set of tracking, own conversion events. Cleaner UTM segmentation and per-offer analytics. Ideal for larger campaigns with multiple creatives.

---

## Offer landing page implementation

```tsx
// app/offer/[slug]/page.tsx (Next.js App Router)
import { OfferLandingClient } from './OfferLandingClient';

const OFFERS = {
  'summer-25': { code: 'SUMMER25', discount: 25, headline: 'Summer Sale — 25% Off Everything' },
  'welcome-15': { code: 'WELCOME15', discount: 15, headline: 'Welcome — 15% Off Your First Order' },
  // ...
};

export default async function OfferPage({ params }: { params: { slug: string } }) {
  const offer = OFFERS[params.slug as keyof typeof OFFERS];
  if (!offer) return <div>Offer not found</div>;

  return <OfferLandingClient offer={offer} slug={params.slug} />;
}
```

```tsx
// app/offer/[slug]/OfferLandingClient.tsx
'use client';
import { useEffect } from 'react';

export function OfferLandingClient({ offer, slug }: { offer: any; slug: string }) {
  useEffect(() => {
    // 1. Persist UTMs from URL to first-party cookie (90-day)
    const params = new URLSearchParams(window.location.search);
    const utm: any = {};
    ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term'].forEach(k => {
      if (params.get(k)) utm[k] = params.get(k);
    });

    if (Object.keys(utm).length) {
      document.cookie = `_utm=${encodeURIComponent(JSON.stringify(utm))};path=/;max-age=${60*60*24*90};samesite=lax`;
    }

    // 2. Also persist the offer code, so checkout picks it up automatically
    document.cookie = `_offer=${offer.code};path=/;max-age=${60*60*24*30};samesite=lax`;
    document.cookie = `_offer_slug=${slug};path=/;max-age=${60*60*24*30};samesite=lax`;

    // 3. Fire view_item_list / view_promotion for GA4
    window.gtag?.('event', 'view_promotion', {
      creative_name: slug,
      creative_slot: 'offer-landing',
      promotion_id: offer.code,
      promotion_name: offer.headline
    });
  }, [offer, slug]);

  return (
    <main>
      <h1>{offer.headline}</h1>
      <p>Use code <code>{offer.code}</code> at checkout — or click below and we'll apply it automatically.</p>
      <a href={`/shop?discount=${offer.code}`}>Shop the sale</a>
    </main>
  );
}
```

---

## Threading the coupon code through the whole stack

**At checkout session creation:**

```ts
const coupon = getCookie('_offer');   // from the offer landing page

const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [...],
  discounts: coupon ? [{ coupon: /* Stripe Coupon ID */ }] : undefined,
  success_url: `${SITE_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url: `${SITE_URL}/cancel`,
  metadata: {
    ...utmMetadata,
    coupon: coupon ?? ''    // <-- carries through to webhook
  }
});
```

**On the success page (browser):**

```ts
window.gtag('event', 'purchase', {
  transaction_id: paymentIntentId,
  value: value,
  currency: currency,
  coupon: order.coupon,     // <-- flows into GA4 purchase event
  items: [...]
});
```

**On the Stripe webhook (server, to Meta CAPI):**

```ts
await sendMetaCAPI({
  event_name: 'Purchase',
  event_id: eventId,
  // ...
  custom_data: {
    // ...
    coupon: session.metadata?.coupon   // <-- flows into Meta reporting
  }
});
```

Now every conversion carries its coupon code across every platform. You can filter/compare performance by code in every dashboard.

---

## Anti-patterns to avoid

- **Generic public codes** posted on Reddit / deal sites. They leak, attribution breaks, and organic buyers get "promotional" attribution.
- **Tagging internal links with UTMs.** Overrides the original session source in GA4 — a user who arrived from Meta now shows as coming from your homepage.
- **Losing UTMs in redirects.** Klaviyo, custom link shorteners, and some CMPs strip them. Fix: pass UTMs through in query strings, or capture at first touch and pass server-side.
- **Newsletter popup that fires a "welcome" code without a UTM tag.** Overrides the click-through UTM, making your Meta campaign look worse than it is.
- **Using the offer code as the event_id.** Two different orders using the same code = one gets deduped away. Fix: `event_id` should always be per-transaction.

---

## SEO structure (if coupon queries drive traffic)

For sites that want to rank for `<brand> coupon codes` and similar queries:

- **Category hub pages** (indexed, 3,500+ words) — comprehensive guides, buying tips, all active codes in context. These rank.
- **Individual code pages** (`noindex, follow`) — for tracking, deep-linking, analytics. Thin content, don't try to index them.
- Never index thin individual code pages — thin-content penalties are why 80% of coupon sites tank.

Detailed SEO structure is outside this skill's scope — see the `seo-audit` skill if you need it.

---

## Docs / references

- Shopify discount URL format: https://help.shopify.com/en/manual/discounts/managing-discount-codes/sharing-discount-links
- GA4 `purchase` event with coupon: https://developers.google.com/analytics/devguides/collection/ga4/reference/events#purchase
- Meta CAPI custom_data: https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/custom-data
- Stripe Coupons API: https://docs.stripe.com/api/coupons
- UTM best practices: https://support.google.com/analytics/answer/10917952
