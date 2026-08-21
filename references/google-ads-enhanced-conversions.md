# Google Ads Enhanced Conversions

**2026 milestones (this is the biggest change of the year in this space):**
- **April 2026:** Google Ads accepts user-provided data from website tags, Data Manager, AND API connections simultaneously — no more "pick one implementation method."
- **June 15, 2026:** Enhanced Conversions for Web + Enhanced Conversions for Leads combined into a **single on/off toggle**. Existing setups auto-migrated.
- **June 15, 2026:** Offline conversions + enhanced conversions for leads uploads moved to the **Data Manager API**; blocked in the legacy Google Ads API.

Median reported lift once enabled: **+5% on Search, +17% on YouTube**.

---

## Two implementation choices

You can pick either or both. Both is fine and Google now accepts both simultaneously.

1. **Native Google Ads conversion action** — you create a conversion in Google Ads, install a Google Tag + event snippet, and add user-provided data on top. Best for direct optimization signal.
2. **Import from GA4** — you create a GA4 Key Event (`purchase`), link GA4 to Google Ads, and import it. Easier but less immediate signal for bidding. Note: **GA4 imports don't support Enhanced Conversions** — for EC you need native Google Ads conversion actions.

For an ecom store spending real money on Google Ads: **use native conversion actions with Enhanced Conversions.**

---

## Audit checklist

- [ ] Google Tag (`gtag.js` or GTM) installed on every page
- [ ] **Conversion Linker** fires on every page (essential — without it, `gclid` is lost)
- [ ] Native Google Ads conversion action created for **Purchase** (and any other conversions)
- [ ] Conversion tracking ID + label stored in env vars
- [ ] Purchase conversion tag fires on the success page with `transaction_id`, `value`, `currency`
- [ ] **Enhanced Conversions toggle ON** (single toggle since June 15, 2026)
- [ ] Customer data terms accepted in Google Ads
- [ ] `gtag('set', 'user_data', {...})` fires **before** the conversion event, with at least one of: email, phone, or name+address
- [ ] User data is hashed (or Google's tag will hash it client-side automatically)
- [ ] Consent Mode v2 signals firing (`ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization`)
- [ ] For subscriptions/upsells: use the same `transaction_id` between the conversion event and any Data Manager API upload

---

## Base install (Google Tag + Conversion Linker + Conversion event)

```tsx
// app/layout.tsx (Next.js App Router)
import Script from 'next/script';

const ADS_ID = process.env.NEXT_PUBLIC_GOOGLE_ADS_CONVERSION_ID!;  // "AW-XXXXXXXXX"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        {/* If you already have GA4 loading gtag.js, don't load it again — just add the Ads config */}
        <Script id="google-ads" strategy="afterInteractive" src={`https://www.googletagmanager.com/gtag/js?id=${ADS_ID}`} />
        <Script id="google-ads-init" strategy="afterInteractive">
          {`
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', '${ADS_ID}');
            // Conversion Linker is automatic when you use gtag.js this way — no extra tag needed.
          `}
        </Script>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

If you're already loading GA4 with `gtag.js`, add the Ads ID with a second `gtag('config', 'AW-XXX')` call in the same script — no need to load the library twice.

---

## Fire the Purchase conversion (success page)

```tsx
// app/success/page.tsx (server component that reads the session, then renders a client component that fires the conversion)
import { CheckoutSuccess } from './CheckoutSuccess';
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2026-07-29.dahlia' });

export default async function SuccessPage({ searchParams }: { searchParams: { session_id?: string } }) {
  if (!searchParams.session_id) return <div>No session</div>;
  const session = await stripe.checkout.sessions.retrieve(searchParams.session_id, {
    expand: ['line_items', 'customer_details', 'payment_intent']
  });
  return <CheckoutSuccess session={JSON.parse(JSON.stringify(session))} />;
}
```

```tsx
// app/success/CheckoutSuccess.tsx  (client component)
'use client';
import { useEffect } from 'react';

const ADS_ID = process.env.NEXT_PUBLIC_GOOGLE_ADS_CONVERSION_ID!;
const PURCHASE_LABEL = process.env.NEXT_PUBLIC_GOOGLE_ADS_CONVERSION_LABEL_PURCHASE!;

export function CheckoutSuccess({ session }: { session: any }) {
  useEffect(() => {
    if (typeof window === 'undefined' || !window.gtag) return;

    const paymentIntentId = typeof session.payment_intent === 'string'
      ? session.payment_intent
      : session.payment_intent?.id;

    // 1. Enhanced Conversions: send user-provided data FIRST
    window.gtag('set', 'user_data', {
      email: session.customer_details?.email,
      phone_number: session.customer_details?.phone,
      address: {
        first_name: session.customer_details?.name?.split(' ')[0],
        last_name: session.customer_details?.name?.split(' ').slice(1).join(' '),
        street: session.customer_details?.address?.line1,
        city: session.customer_details?.address?.city,
        region: session.customer_details?.address?.state,
        postal_code: session.customer_details?.address?.postal_code,
        country: session.customer_details?.address?.country
      }
    });

    // 2. Fire the conversion event
    window.gtag('event', 'conversion', {
      send_to: `${ADS_ID}/${PURCHASE_LABEL}`,
      transaction_id: paymentIntentId,
      value: (session.amount_total ?? 0) / 100,
      currency: (session.currency ?? 'usd').toUpperCase()
    });
  }, [session]);

  return (
    <main>
      <h1>Thanks for your order!</h1>
      <p>Order #{session.payment_intent}</p>
    </main>
  );
}
```

Notes:
- Google's tag hashes email/phone/address client-side automatically. You do **not** need to SHA-256 them yourself.
- `send_to` combines conversion ID + label: `AW-1234567890/AbC-D_efG-h12_34-XY`. Get the label from the conversion action's tag setup in Google Ads.

---

## Environment variables

```env
NEXT_PUBLIC_GOOGLE_ADS_CONVERSION_ID=AW-1234567890
NEXT_PUBLIC_GOOGLE_ADS_CONVERSION_LABEL_PURCHASE=AbCdEfGhIjK-1234-XY
```

---

## Beyond code (Google Ads UI)

1. **Create the conversion action** — Google Ads → Goals → Conversions → New conversion action → Website → Enter your site URL → "Manually enter conversion action". Set:
   - Goal category: Purchase
   - Value: "Use different values for each conversion" (dynamic per transaction)
   - Count: "One" (avoid double-counting from the same order)
   - Click-through window: 30 days (default is fine)
   - View-through window: 1 day
   - Attribution model: Data-driven (default since 2024)
2. **Copy the conversion ID + label** from the tag setup screen — paste into env vars.
3. **Enable Enhanced Conversions** — Goals → Settings → Customer data use → toggle ON → accept customer data terms. (Since June 2026, this is one toggle for web + leads.)
4. **Link GA4** — Tools → Data manager → Product links → Google Analytics (GA4) → link. Optional if you're going native, but useful for cross-reporting.

---

## Common failure modes

- **Conversion Linker missing.** Without it, `gclid` is lost from the URL, and enhanced conversions can't match. Fix: `gtag('config', 'AW-XXX')` at the top of every page (or the Conversion Linker tag in GTM).
- **User data set AFTER the conversion event.** Order matters — `gtag('set', 'user_data', ...)` must fire before `gtag('event', 'conversion', ...)` on the same page.
- **User data set on a page where the conversion doesn't fire.** EC only associates data with conversions fired on the same page load. Set both together.
- **GA4-imported conversions used with EC.** Not supported. Use a native Google Ads conversion action for EC.
- **Consent denied but no Consent Mode default.** Result: conversions from EU users disappear entirely instead of being modeled. Fix: implement Consent Mode v2 defaults before the Google tag loads.

---

## Testing

1. **Google Tag Assistant** ([Chrome extension](https://chrome.google.com/webstore/detail/tag-assistant-companion/jmekfmbnaedfebfnmakmokmlfpblbfdm)) — install, browse to your site, open Tag Assistant, verify:
   - Google tag fires
   - Conversion Linker fires
   - Conversion event fires on success page with correct `transaction_id`, `value`, `currency`
   - User-provided data is present in the payload (should show `em`, `ph`, `addr` hashes)
2. **Google Ads diagnostic** — Goals → Conversions → [your Purchase conversion] → Enhanced Conversions diagnostic. Should show "Enhanced conversions is receiving data" within 48 hours.
3. **Conversion action status** — should move from "Recording conversions" (green) within 24 hours of a real purchase.

---

## Docs

- Enhanced Conversions for Web (gtag): https://support.google.com/google-ads/answer/13258081
- Enhanced Conversions for Web (GTM): https://support.google.com/google-ads/answer/13262500
- Updates to your enhanced conversions settings (April/June 2026): https://support.google.com/google-ads/answer/16884284
- Consent Mode reference: https://support.google.com/google-ads/answer/13802165
- Data Manager API: https://developers.google.com/data-manager
