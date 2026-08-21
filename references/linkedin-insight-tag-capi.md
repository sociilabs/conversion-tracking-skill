# LinkedIn Insight Tag + Conversions API

LinkedIn skews B2B, higher-value, longer sales cycles. Its audience also skews toward heavy ad-blocker use (30–50%), so a Pixel-only setup loses a substantial share of qualified conversions. CAPI + Insight Tag with `eventId` dedup + `li_fat_id` capture is the 2026 standard.

Note: Meta launched free 1-click CAPI in April 2026 — **LinkedIn has no equivalent**. LinkedIn CAPI still requires a direct integration, server-side GTM, or a paid third-party tool.

---

## Standard conversion categories

`PURCHASE`, `LEAD`, `SIGN_UP`, `ADD_TO_CART`, `BOOK_APPOINTMENT`, `START_CHECKOUT`, `SUBSCRIBE`, `KEY_PAGE_VIEW`, `QUALIFIED_LEAD`, `MARKETING_QUALIFIED_LEAD`, `SALES_QUALIFIED_LEAD`, `CLOSE_WON`, `CLOSE_LOST`. For ecom: `PURCHASE`, `START_CHECKOUT`, `ADD_TO_CART`.

---

## Audit checklist

- [ ] Partner ID stored in env var
- [ ] Insight Tag installed on every page
- [ ] Conversion rules created in Campaign Manager for each key event (browser rule + CAPI rule per event)
- [ ] Enhanced Conversion Tracking enabled on the Insight Tag
- [ ] `li_fat_id` captured from URL and cookie on landing pages
- [ ] Access token generated (Campaign Manager → Data → Signals manager → Direct API or GTM → Generate token)
- [ ] CAPI events firing with matching `eventId`
- [ ] Match keys sent server-side (hashed): email, first_name + last_name, `li_fat_id`
- [ ] Conversion associated with the relevant campaigns (this trips people up — creating a conversion rule alone doesn't do anything until it's linked to a campaign)
- [ ] Attribution windows set: 30-day click / 7-day view for ecom; 90-day click / 90-day view for lead

---

## Install the Insight Tag

```tsx
// app/layout.tsx
import Script from 'next/script';

const LI_PARTNER_ID = process.env.NEXT_PUBLIC_LINKEDIN_PARTNER_ID!;

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <Script id="linkedin-insight" strategy="afterInteractive">
          {`
            _linkedin_partner_id = "${LI_PARTNER_ID}";
            window._linkedin_data_partner_ids = window._linkedin_data_partner_ids || [];
            window._linkedin_data_partner_ids.push(_linkedin_partner_id);
            (function(l) {
              if (!l){window.lintrk = function(a,b){window.lintrk.q.push([a,b])};
              window.lintrk.q=[]}
              var s = document.getElementsByTagName("script")[0];
              var b = document.createElement("script");
              b.type = "text/javascript";b.async = true;
              b.src = "https://snap.licdn.com/li.lms-analytics/insight.min.js";
              s.parentNode.insertBefore(b, s);
            })(window.lintrk);
          `}
        </Script>
        <noscript>
          <img height="1" width="1" style={{ display: 'none' }} alt=""
            src={`https://px.ads.linkedin.com/collect/?pid=${LI_PARTNER_ID}&fmt=gif`} />
        </noscript>
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## Fire conversion events (browser)

```ts
// lib/tracking/linkedin.ts
declare global { interface Window { lintrk: any } }

const CONV_ID_PURCHASE = process.env.NEXT_PUBLIC_LINKEDIN_CONV_ID_PURCHASE!;

export function liPurchase(eventId: string) {
  // LinkedIn Insight Tag fires conversion by conversion_id — event_id is a payload attribute for dedup
  window.lintrk?.('track', {
    conversion_id: CONV_ID_PURCHASE,
    event_id: eventId
  });
}
```

---

## Server-side CAPI

```ts
// lib/tracking/linkedin.ts (server)
import crypto from 'node:crypto';

const ACCESS_TOKEN = process.env.LINKEDIN_CAPI_ACCESS_TOKEN!;
const CONV_ID_PURCHASE = process.env.LINKEDIN_CONV_RULE_PURCHASE!;
const CAPI_ENDPOINT = 'https://api.linkedin.com/rest/conversionEvents';

function sha256(v: string) {
  return crypto.createHash('sha256').update(v.trim().toLowerCase()).digest('hex');
}

export async function sendLinkedInCAPI(input: {
  event_id: string;
  event_time_ms: number;
  conversion_urn: string;             // e.g., urn:lla:llaPartnerConversion:<CONV_RULE_ID>
  amount?: { amount: string; currencyCode: string };
  user: {
    email?: string;
    first_name?: string; last_name?: string;
    company?: string; title?: string; country?: string;
    li_fat_id?: string;
  };
}) {
  const userIds: any[] = [];
  if (input.user.email) userIds.push({ idType: 'SHA256_EMAIL', idValue: sha256(input.user.email) });
  if (input.user.li_fat_id) userIds.push({ idType: 'LINKEDIN_FIRST_PARTY_ADS_TRACKING_UUID', idValue: input.user.li_fat_id });

  const user: any = { userIds };
  if (input.user.first_name || input.user.last_name || input.user.company || input.user.title || input.user.country) {
    user.userInfo = {
      firstName: input.user.first_name,
      lastName: input.user.last_name,
      companyName: input.user.company,
      title: input.user.title,
      countryCode: input.user.country
    };
  }

  const body = {
    conversion: input.conversion_urn,
    conversionHappenedAt: input.event_time_ms,
    conversionValue: input.amount,
    eventId: input.event_id,
    user
  };

  const res = await fetch(CAPI_ENDPOINT, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${ACCESS_TOKEN}`,
      'Content-Type': 'application/json',
      'LinkedIn-Version': '202601',    // current stable version — update as needed
      'X-Restli-Protocol-Version': '2.0.0'
    },
    body: JSON.stringify(body)
  });

  if (!res.ok) throw new Error(`LinkedIn CAPI ${res.status}: ${await res.text()}`);
  return res.json();
}
```

### Calling it from the Stripe webhook

```ts
await sendLinkedInCAPI({
  event_id: eventId,     // === payment_intent.id
  event_time_ms: Date.now(),
  conversion_urn: `urn:lla:llaPartnerConversion:${process.env.LINKEDIN_CONV_RULE_PURCHASE}`,
  amount: {
    amount: ((session.amount_total ?? 0) / 100).toFixed(2),
    currencyCode: (session.currency ?? 'usd').toUpperCase()
  },
  user: {
    email: session.customer_details?.email ?? undefined,
    first_name: session.customer_details?.name?.split(' ')[0],
    last_name: session.customer_details?.name?.split(' ').slice(1).join(' '),
    country: session.customer_details?.address?.country ?? undefined,
    li_fat_id: session.metadata?.li_fat_id
  }
});
```

---

## Environment variables

```env
NEXT_PUBLIC_LINKEDIN_PARTNER_ID=1234567
NEXT_PUBLIC_LINKEDIN_CONV_ID_PURCHASE=12345678       # for the browser Insight Tag
LINKEDIN_CAPI_ACCESS_TOKEN=AQV...                    # from Campaign Manager
LINKEDIN_CONV_RULE_PURCHASE=87654321                 # conversion rule ID for CAPI
```

---

## Capture `li_fat_id`

Add to your `persistClickIds()`:

```js
const params = new URLSearchParams(window.location.search);
if (params.get('li_fat_id')) {
  document.cookie = `li_fat_id=${params.get('li_fat_id')};path=/;max-age=${60*60*24*90};samesite=lax`;
}
```

When Enhanced Conversion Tracking is enabled on the Insight Tag, LinkedIn automatically appends `li_fat_id` to landing page URLs. Capture from URL; also read from a first-party cookie (`li_fat_id` — LinkedIn sets it automatically if the Insight Tag runs).

---

## Beyond code (Campaign Manager)

- **Create conversion rules** — Campaign Manager → Account Assets → Conversion Tracking → Create Conversion. Do this **twice per event**: once with source "Insight Tag" (browser), once with source "Conversions API" (server). They should have the same name (e.g., "Purchase - Insight Tag" and "Purchase - CAPI") for reporting clarity.
- **Enable Enhanced Conversion Tracking** — In each Insight Tag conversion → Settings → Enhanced Conversion Tracking → ON.
- **Associate conversions with campaigns** — critical step people forget. Conversion → Associate to campaigns → select the campaigns that should use this conversion for optimization.
- **Generate CAPI access token** — Data → Signals manager → Direct API (or Google Tag Manager) → Generate token.
- **Attribution windows** — Conversion → Settings → Attribution → 30-day click, 7-day view for ecom; 90-day click, 90-day view for lead.

---

## Common failure modes

- **Conversion rule created but not associated with a campaign.** Silent. No conversions ever report.
- **CAPI conversion sent to the browser rule's URN.** Wrong source; may or may not dedupe correctly.
- **`li_fat_id` not captured.** Match rate drops dramatically for anonymous visitors.
- **Attribution window too narrow** for a long B2B cycle. Use the max where it makes sense.
- **Not using both browser + CAPI.** For a 30% blocker audience, this is 30% of your data lost.

---

## Testing

1. Fire a test conversion via the Insight Tag and observe it in Campaign Manager → Conversion Tracking → [conversion] → status "Active".
2. Fire a matching CAPI event with the same `eventId` and confirm dedup: only the Insight Tag conversion should show in reporting; the CAPI conversion count is deducted.
3. Check the `li_fat_id` is populated in your CAPI payload for improved match rates.

---

## Docs

- Conversions API overview: https://www.linkedin.com/help/lms/answer/a1655394
- Set up CAPI: https://www.linkedin.com/help/lms/answer/a1657171
- Best practices: https://www.linkedin.com/help/lms/answer/a5538676
- Deduplication: https://learn.microsoft.com/en-us/linkedin/marketing/conversions/deduplication
- Direct API: https://learn.microsoft.com/en-us/linkedin/marketing/conversions/conversions-api
