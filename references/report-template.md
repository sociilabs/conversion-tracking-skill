# Report Template

This is the exact structure every audit report must follow. Copy the skeleton, fill in each section based on what you find in the codebase, and save to `docs/conversion-tracking/audit-YYYY-MM-DD-vNN.md`.

**Section order is fixed.** Users read multiple versions of this report over time — they need to find things in the same place every run.

---

## Copy this skeleton (verbatim structure — fill in the `[...]` bits)

```markdown
# Conversion Tracking Audit — v[NN]
_Generated [YYYY-MM-DD] · Site: [domain.com] · Stack: [Next.js 14 App Router + Stripe + Vercel]_

## Executive Summary

**Overall readiness: [XX]/100** — [one-line qualitative label: "Foundational", "Partial coverage", "Solid, gaps at the edges", "Launch-ready"]

**Top 3 priorities this week:**
1. [Highest-leverage fix — e.g., "Implement Meta CAPI to recover ~30% of purchase events currently lost to browser blocking"]
2. [Next]
3. [Next]

**Estimated conversion recovery if all fixes applied:** [+X–Y% reported conversions across paid channels, based on published benchmarks — see individual sections]

**One-paragraph summary:** [3–5 sentences. Plain English. What's working, what's dangerous, what's next. Founder-to-founder tone. No hype.]

---

## Since Last Audit
_(Omit this section on v01. On v02+, compare to the previous file in `docs/conversion-tracking/`.)_

### Fixed since v[NN-1]
- ✅ [Concrete thing that got fixed]
- ✅ [Another]

### New gaps found
- ⚠️ [Thing that got added to the site but has a tracking gap]

### Regressions
- 🚨 [Thing that was working in v[NN-1] but is now broken — usually a code change disabled tracking]

---

## Stack Detected

| Layer | Detected |
|---|---|
| Framework | [Next.js 14 App Router / Astro / SvelteKit / plain HTML+PHP / etc.] |
| Hosting | [Vercel / Netlify / Cloudflare Pages / self-hosted / etc.] |
| Payment processor | [Stripe Checkout / Stripe Payment Element / PayPal / Lemon Squeezy / etc.] |
| CMP (cookie banner) | [Cookiebot / OneTrust / CookieHub / custom / none] |
| Tag manager | [GTM / server-side GTM / none — direct gtag] |
| Ad platforms in scope | [Meta, Google Ads, TikTok] |
| Ad platforms detected but not in scope | [LinkedIn (Insight Tag found but not requested)] |
| Region focus | [US / EEA+UK / Global] |

---

## 1. Meta Pixel + Conversions API

**Status:** [✅ Done / ⚠️ Partial / ❌ Missing / ➖ N/A]

### What's in place
- [Concrete list of what you found. Include file paths and pixel IDs where visible.]

### What's missing
- 🚨 [Critical gaps]
- ⚠️ [Non-critical gaps]

### Recommended code

_Place the exact snippet the user should add, with correct file path and language, complete enough to paste in. Reference `references/meta-pixel-capi.md` for full patterns._

```tsx
// app/layout.tsx — add Meta Pixel with consent-aware default
[snippet]
```

```ts
// app/api/webhooks/stripe/route.ts — send server-side Purchase event
[snippet]
```

### Beyond code (do in Meta Events Manager)
- [ ] Enable Conversions API access token: Business Settings → Data Sources → [Pixel] → Generate access token
- [ ] Set up Event Match Quality monitoring
- [ ] Configure Domain Verification for the domain

### Docs
- Official CAPI docs: https://developers.facebook.com/docs/marketing-api/conversions-api
- Test Events tool: Events Manager → [Pixel] → Test Events

---

## 2. Google Analytics 4

**Status:** [✅ Done / ⚠️ Partial / ❌ Missing]

### What's in place
- [...]

### What's missing
- [...]

### Recommended code

```tsx
// [file path]
[snippet]
```

### Beyond code (do in GA4 Admin)
- [ ] Mark `purchase` as a Key Event
- [ ] Enable Enhanced Measurement on the web data stream
- [ ] Link GA4 to Google Ads
- [ ] Set data retention to 14 months (or the max your plan allows)

### Docs
- GA4 Ecommerce guide: https://developers.google.com/analytics/devguides/collection/ga4/ecommerce
- Recommended events: https://support.google.com/analytics/answer/9267735

---

## 3. Google Ads Enhanced Conversions

**Status:** [✅ / ⚠️ / ❌ / ➖]

### What's in place
### What's missing
### Recommended code
### Beyond code (do in Google Ads)
- [ ] Turn on Enhanced Conversions (single unified toggle since June 15, 2026)
- [ ] Accept customer data terms
- [ ] Import GA4 key events OR create native Google Ads conversion actions

### Docs
- https://support.google.com/google-ads/answer/13258081

---

## 4. TikTok Pixel + Events API

**Status:** [✅ / ⚠️ / ❌ / ➖]

_[Same subsection structure as above.]_

### Docs
- https://ads.tiktok.com/help/article/events-api

---

## 5. LinkedIn Insight Tag + Conversions API

**Status:** [✅ / ⚠️ / ❌ / ➖]

_[Same subsection structure.]_

### Docs
- https://www.linkedin.com/help/lms/answer/a1655394

---

## 6. Pinterest Tag + Conversions API

**Status:** [✅ / ⚠️ / ❌ / ➖]

_[Same subsection structure.]_

### Docs
- https://help.pinterest.com/en/business/article/the-pinterest-api-for-conversions

---

## 7. Reddit Pixel + Conversions API

**Status:** [✅ / ⚠️ / ❌ / ➖]

_[Same subsection structure.]_

### Docs
- https://business.reddithelp.com/s/article/link-standard-events-to-conversion-events

---

## 8. Stripe Checkout Integration

**Status:** [✅ / ⚠️ / ❌]

### What's in place
- Checkout Session creation at [`app/api/checkout/route.ts`]
- Success page at [`app/success/page.tsx`]
- Cancel page at [`app/cancel/page.tsx`]
- Webhook endpoint at [`app/api/webhooks/stripe/route.ts`]

### What's missing
- 🚨 [e.g., "Webhook does not verify signature — vulnerable to spoofed events."]
- ⚠️ [e.g., "Success page relies on redirect only — customers who close the tab never get their purchase tracked."]

### Recommended code

```ts
// app/api/checkout/route.ts — create session with attribution metadata
[snippet]
```

```tsx
// app/success/page.tsx — retrieve session server-side, fire browser conversion events
[snippet]
```

```ts
// app/api/webhooks/stripe/route.ts — verify signature, dedupe, fire server-side conversions
[snippet]
```

### Environment variables needed

```env
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
```

### Beyond code (do in Stripe Dashboard)
- [ ] Create webhook endpoint pointing to `https://[domain]/api/webhooks/stripe`
- [ ] Subscribe to events: `checkout.session.completed`, `payment_intent.succeeded`, `invoice.paid`, `charge.refunded`, `checkout.session.expired`
- [ ] Copy the endpoint secret into `STRIPE_WEBHOOK_SECRET`

### Docs
- Fulfill orders: https://docs.stripe.com/checkout/fulfillment
- Custom success page: https://docs.stripe.com/payments/checkout/custom-success-page
- Webhooks: https://docs.stripe.com/webhooks

---

## 9. Consent Mode v2 + Privacy

**Status:** [✅ / ⚠️ / ❌ — usually 🚨 Critical if EEA/UK traffic and defaults are wrong]

### Consent Management Platform detected
[Cookiebot / OneTrust / custom / **none**]

### What's in place
### What's missing
- 🚨 [If EEA/UK: no default consent state fires before Google tag — advertising data is leaking.]

### Recommended code

```html
<!-- Put this BEFORE any Google tag loads -->
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('consent', 'default', {
    'ad_storage': 'denied',
    'analytics_storage': 'denied',
    'ad_user_data': 'denied',
    'ad_personalization': 'denied',
    'wait_for_update': 500,
    'region': ['EEA', 'GB', 'CH']
  });
  gtag('consent', 'default', {
    'ad_storage': 'granted',
    'analytics_storage': 'granted',
    'ad_user_data': 'granted',
    'ad_personalization': 'granted'
  });
</script>
```

### Beyond code
- [ ] Choose a Google-certified CMP (Cookiebot, CookieHub, OneTrust, Usercentrics — see full list in `references/consent-mode-v2.md`)
- [ ] Verify with Google Tag Assistant that `gcs=G1nn` (all granted) fires after acceptance
- [ ] Update privacy policy for the June 15, 2026 change (ad_storage is now sole gate for GA→Ads data)

### Docs
- Consent Mode overview: https://developers.google.com/tag-platform/security/guides/consent

---

## 10. Coupon & Offer Pages

**Status:** [✅ / ⚠️ / ❌ / ➖]

### Detected offers
| Offer | Landing page | Code | Channels using it |
|---|---|---|---|
| Summer 25% off | `/offer/summer-25` | `SUMMER25` | Meta, Instagram, Email |

### What's in place
### What's missing
- [ ] Each channel needs its own unique code (or dynamic code per creator/audience) so leakage doesn't break attribution.
- [ ] `coupon` field needs to flow into GA4 `purchase` event, Meta CAPI `custom_data.coupon`, and Stripe metadata.
- [ ] UTM parameters need to persist to a first-party cookie for late attribution (Stripe checkout can be days later on a different device).

### Recommended code

```tsx
// app/offer/[slug]/page.tsx — offer landing page with UTM persistence
[snippet]
```

### Docs
- See `references/coupon-and-offer-pages.md` for the full pattern.

---

## Cross-Platform: `event_id` Strategy

**Current state:** [Describe what dedup pattern is in place, if any.]

**Recommended pattern for this site:**

- **Pre-purchase events** (`view_item`, `add_to_cart`, `begin_checkout`, etc.): UUID generated in the browser, stored in dataLayer, reused across every ad-platform pixel firing for the same interaction.
- **Purchase**: **use the order ID** (or `payment_intent_id` from Stripe). Pass identically through: GA4 `transaction_id`, Meta Pixel `eventID`, Meta CAPI `event_id`, TikTok `event_id`, LinkedIn `eventId`, Pinterest `event_id`, Reddit conversion ID.

This is the single most impactful architectural decision in this whole audit. Full pattern: `references/event-id-and-datalayer.md`.

---

## Environment Variables Needed

Compile everything from every section into one block the user can drop into `.env`:

```env
# Meta
NEXT_PUBLIC_META_PIXEL_ID=
META_CAPI_ACCESS_TOKEN=
META_CAPI_TEST_EVENT_CODE=       # remove after testing

# Google Analytics 4
NEXT_PUBLIC_GA_MEASUREMENT_ID=
GA_MEASUREMENT_PROTOCOL_API_SECRET=

# Google Ads
NEXT_PUBLIC_GOOGLE_ADS_CONVERSION_ID=
GOOGLE_ADS_CONVERSION_LABEL_PURCHASE=

# TikTok
NEXT_PUBLIC_TIKTOK_PIXEL_ID=
TIKTOK_EVENTS_API_ACCESS_TOKEN=

# LinkedIn (if in scope)
NEXT_PUBLIC_LINKEDIN_PARTNER_ID=
LINKEDIN_CAPI_ACCESS_TOKEN=

# Pinterest (if in scope)
NEXT_PUBLIC_PINTEREST_TAG_ID=
PINTEREST_CAPI_ACCESS_TOKEN=
PINTEREST_AD_ACCOUNT_ID=

# Reddit (if in scope)
NEXT_PUBLIC_REDDIT_PIXEL_ID=
REDDIT_CAPI_ACCESS_TOKEN=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
```

---

## QA & Validation Checklist

Test each platform end-to-end **before** running paid traffic. In this order:

- [ ] **Stripe** — Use `stripe listen --forward-to localhost:3000/api/webhooks/stripe` and place a test purchase. Verify webhook fires, signature verifies, and no duplicate processing occurs on retry.
- [ ] **GA4** — DebugView shows `purchase` with `transaction_id`, `value`, `currency`, and non-empty `items[]`.
- [ ] **Meta** — Events Manager → Test Events shows `Purchase` from both Browser and Server sources, and Deduplication rate > 90%.
- [ ] **Google Ads** — Tag Assistant confirms Enhanced Conversions data is being sent (user_data with hashed email/phone).
- [ ] **TikTok** — Events Manager → Test Events shows `CompletePayment` from both Pixel and Events API, deduplication working.
- [ ] **LinkedIn / Pinterest / Reddit** — same test-events pattern in each platform's Events Manager.
- [ ] **Consent Mode** — In Chrome DevTools → Network, verify `gcs` and `gcd` URL parameters appear on Google requests. Accept banner and re-check.
- [ ] **Coupon flow** — Use a test discount code from an offer landing page; verify it appears in Stripe session metadata, GA4 `purchase` event, and Meta CAPI `custom_data.coupon`.

---

## Priority Roadmap

Ordered by leverage. Effort tags: **[S]** = < 2 hrs, **[M]** = 2–8 hrs, **[L]** = > 1 day.

### 🚨 This week
1. **[S]** [Highest-leverage single fix — often "fire signature-verified Stripe webhook and send server-side Purchase to Meta + GA4"]
2. **[M]** [Next]

### ⏭ This month
3. **[M]** [Next]
4. **[L]** [Next]

### 💡 When you have time
5. **[S]** [Optional / optimization]
6. **[M]** [Optional / optimization]

---

_This report was generated by the `conversion-tracking` skill. Re-run any time to produce v[NN+1] and see progress since this version._
```

---

## Section-by-section guidance for the agent filling this in

### Executive Summary — Score calculation

Rough rubric (100 points total, deduct for gaps):

| Area | Max points | Critical failure (auto-halve) |
|---|---|---|
| Meta CAPI + dedup | 15 | Pixel-only, no CAPI |
| GA4 core ecommerce funnel | 12 | Missing `purchase` event or `items[]` empty |
| Google Ads EC (if running Ads) | 10 | Conversion Linker missing |
| Each additional ad platform in scope | 8 each | Pixel-only, no CAPI |
| Stripe webhook (signed + idempotent) | 15 | Webhook missing or unsigned |
| Consent Mode v2 (if EEA/UK traffic) | 10 | Missing or misconfigured defaults |
| `event_id` strategy (unified across platforms) | 15 | No dedup — each platform silently double-counts |
| Coupon/offer attribution | 5 | — |
| QA/testing evidence | 5 | — |

Score modestly. A site that passed a first audit at 45/100 is normal. A launch-ready site is 85+.

### Executive Summary — Estimated recovery

Rough published benchmarks to draw from (from research doc):
- Meta CAPI vs Pixel-only: +15–20% ROAS improvement once EMQ is optimized, +20–30% recovered conversions.
- Google Ads Enhanced Conversions: median +5% on Search, +17% on YouTube.
- TikTok Events API vs Pixel-only: +30–50% recovered conversions on iOS/blocker-heavy traffic (site-dependent).
- Consent Mode Advanced vs no Consent Mode: recovers up to 65% of otherwise-lost EU conversion journeys via modeling.

Combine conservatively — don't add them all together (there's overlap). A reasonable range for a paid-media-heavy site going from client-only to full server-side + Enhanced Conversions is **+15–35% reported paid conversions**.

### "Since Last Audit" — how to build it

1. `ls docs/conversion-tracking/` — find the previous file.
2. Read it.
3. Compare each numbered section — did status change from ❌→✅ or ⚠️→✅?
4. Note any new areas that got added (e.g., site added Pinterest since last run).
5. Flag any regressions.

If there's no previous file, omit this section entirely.

### Language and tone

Direct, founder-to-founder. Assume the reader is technical enough to read code but doesn't want to waste time. Bad: "You may want to consider potentially exploring the implementation of Meta Conversions API which could yield some conversion recovery benefits." Good: "You're running Pixel-only. Add CAPI. You're leaving ~25% of purchase events on the table."

No emojis inside prose (the ✅/⚠️/❌/🚨/💡 status glyphs are the only allowed ones, used in the exact positions shown above). No corporate hedging.

Cite platform docs directly at the end of each section — one or two links max. Don't cite blog posts.
