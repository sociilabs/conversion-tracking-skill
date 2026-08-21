# Google Consent Mode v2 + Privacy

**Mandatory since March 6, 2024** for EEA/UK traffic. Non-compliant = restricted remarketing, blocked conversion measurement, no modeling. And as of **June 15, 2026**, the `ad_storage` signal alone gates whether GA4 sends advertising data to linked Google Ads accounts — the old Google Signals toggle no longer restricts.

If your site serves any EU/UK traffic and doesn't have Consent Mode v2 correctly implemented, you're either non-compliant OR silently sending advertising data you thought was being held back.

---

## The four signals

| Signal | Controls |
|---|---|
| `ad_storage` | Ad cookies and identifiers (e.g., `_gcl_au`). **Sole gate for GA→Ads data since June 15, 2026.** |
| `analytics_storage` | Analytics cookies (`_ga`, `_ga_*`). When denied, GA4 sends cookieless pings for modeling. |
| `ad_user_data` | Whether user data can be sent to Google for advertising. |
| `ad_personalization` | Whether data can be used for personalized ads / remarketing. |

All four must be set. Setting only the v1 signals (`ad_storage` + `analytics_storage`) is no longer compliant.

---

## Basic vs Advanced

- **Basic:** Google tags don't load until the user grants consent. Nothing reaches Google before the banner interaction. Zero modeling for declined users.
- **Advanced (recommended):** Tags load with all signals defaulted to `denied`. Cookieless pings are sent for declined users, and Google models attribution from them. Requires:
  - A **Google-certified CMP** (Cookiebot, CookieHub, OneTrust, Usercentrics, Iubenda, Didomi are certified).
  - Threshold for modeling: typically **700 ad clicks per 7 days per country** per property.

Most ecom sites should run Advanced Mode — modeling recovers up to 65% of otherwise-lost EU conversion journeys.

---

## Audit checklist

- [ ] `gtag('consent', 'default', {...})` fires **before** Google tag / GTM loads (`beforeInteractive` in Next.js Scripts)
- [ ] All 4 signals present in the default: `ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization`
- [ ] Default state is `denied` for EEA/UK region (use `region: ['EEA', 'GB', 'CH']`)
- [ ] `wait_for_update: 500` set (prevents race with banner)
- [ ] `gtag('consent', 'update', {...})` fires immediately on banner interaction, same page load
- [ ] Google-certified CMP is used (if you want modeling)
- [ ] Consent state also gated for non-Google tags (Meta Pixel, TikTok, LinkedIn, Pinterest, Reddit) via CMP triggers or a `consent_granted` dataLayer event
- [ ] Google Tag Assistant shows `gcs=G111` (all granted) after accepting, `gcs=G100` (denied) before
- [ ] Privacy policy updated for the June 15, 2026 change (ad_storage is now sole gate for GA→Ads data)

---

## The correct implementation order (this is critical)

```html
<!-- BEFORE any other script in <head> -->
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}

  // 1. Region-specific defaults — denied for EEA/UK
  gtag('consent', 'default', {
    ad_storage: 'denied',
    analytics_storage: 'denied',
    ad_user_data: 'denied',
    ad_personalization: 'denied',
    wait_for_update: 500,
    region: ['EEA', 'GB', 'CH']
  });

  // 2. Region-specific defaults — granted for the rest of the world
  //    (change per your legal counsel)
  gtag('consent', 'default', {
    ad_storage: 'granted',
    analytics_storage: 'granted',
    ad_user_data: 'granted',
    ad_personalization: 'granted'
  });
</script>

<!-- Then load your Google tag / GTM -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXX"></script>
```

Then, when the user interacts with the cookie banner, your CMP calls `gtag('consent', 'update', {...})` with their choices.

### Next.js App Router version

```tsx
// app/layout.tsx
import Script from 'next/script';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
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
            gtag('consent', 'default', {
              ad_storage: 'granted',
              analytics_storage: 'granted',
              ad_user_data: 'granted',
              ad_personalization: 'granted'
            });
          `}
        </Script>
        {/* Only load Google tag AFTER consent defaults have been set */}
        <Script src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXX" strategy="afterInteractive" />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

---

## CMP integration (Cookiebot example)

Most Google-certified CMPs handle the consent update automatically once integrated. For Cookiebot:

```html
<script id="Cookiebot" src="https://consent.cookiebot.com/uc.js" data-cbid="YOUR-CBID" data-blockingmode="auto" type="text/javascript"></script>
```

Cookiebot handles the `gtag('consent', 'update', {...})` call automatically based on user choice. For a homegrown banner, fire this yourself:

```js
// In your banner's "Accept All" button handler:
gtag('consent', 'update', {
  ad_storage: 'granted',
  analytics_storage: 'granted',
  ad_user_data: 'granted',
  ad_personalization: 'granted'
});

// In "Reject All":
gtag('consent', 'update', {
  ad_storage: 'denied',
  analytics_storage: 'denied',
  ad_user_data: 'denied',
  ad_personalization: 'denied'
});
```

Also push a `consent_granted` event to dataLayer so non-Google tags can react:

```js
dataLayer.push({ event: 'consent_granted' });
```

Then set up your Meta / TikTok / LinkedIn / Pinterest / Reddit tag triggers to fire on this event, not on page load.

---

## Google-certified CMPs

The full list is at https://cmppartnerprogram.withgoogle.com, but common ones for ecom:
- **Cookiebot** — Denmark, feature-rich, good for EEA compliance
- **CookieHub** — Iceland, lightweight, developer-friendly
- **OneTrust** — enterprise-grade, complex
- **Usercentrics** — Germany, comprehensive
- **Iubenda** — Italy, popular for SMB
- **Didomi** — France, good for publishers
- **CookieYes** — India, budget-friendly

Any of these will handle the Consent Mode v2 signals correctly out of the box.

---

## The June 15, 2026 change (easy to miss)

Before June 15, 2026: two settings gated GA→Ads advertising data — the Google Signals toggle in GA4 Admin AND the `ad_storage` signal from Consent Mode.

After June 15, 2026: **`ad_storage` alone gates it.** Google Signals no longer restricts.

**What this means for you:** if you previously relied on Google Signals being off as a privacy safety net, and your Consent Mode default for `ad_storage` is `granted` (e.g., because you have no EEA banner and everything defaults to granted), you're now sending Google Ads the data that Google Signals used to hold back. **Under GDPR, this is likely a material change requiring proactive notification or fresh consent for third-party data sharing.**

Review your defaults, especially if you serve any EEA/UK traffic — it's easy to fix now, expensive to explain to a regulator later.

---

## Verify with Google Tag Assistant

1. Install Chrome extension: https://chrome.google.com/webstore/detail/tag-assistant-companion/jmekfmbnaedfebfnmakmokmlfpblbfdm
2. Open your site with an incognito window from an EEA IP (or use a VPN).
3. Open Tag Assistant → find your Google tag → check the payload.
4. Look for the `gcs` URL parameter on Google requests:
   - `gcs=G100` — all consent denied (before banner interaction)
   - `gcs=G111` — analytics + ads consent granted (after "Accept all")
   - `gcs=G101` — mixed
5. Also look for `gcd` — the granular consent state per signal.

If you never see `gcs=` in the URL, Consent Mode is not implemented.

---

## Common failure modes

- **Default consent fires AFTER Google tag loads.** Google tag has already committed to granted or has fired without consent state. Fix: `strategy="beforeInteractive"` in Next.js, or manual `<head>` script placement.
- **Only v1 signals set (ad_storage + analytics_storage).** Missing `ad_user_data` and `ad_personalization`. Google flags this in Diagnostics; features get restricted.
- **`region` parameter missing.** Defaults apply globally — non-EEA users unnecessarily start denied.
- **Non-Google tags not consent-gated.** Meta Pixel fires regardless of consent state. Result: legal exposure. Fix: fire tags only on `consent_granted` dataLayer event.
- **After June 15, 2026: `ad_storage` default is `granted` in EEA.** You're leaking advertising data GA→Ads that Google Signals used to hold back.

---

## Docs

- Consent Mode overview: https://developers.google.com/tag-platform/security/guides/consent
- Consent Mode reference (Google Ads help): https://support.google.com/google-ads/answer/13802165
- June 15, 2026 change: https://support.google.com/analytics/answer/16884324
- Simo Ahava's technical deep-dive: https://www.simoahava.com/analytics/consent-mode-v2-google-tags/
