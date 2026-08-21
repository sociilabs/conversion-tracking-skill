---
name: conversion-tracking
description: Audit an ecommerce site's end-to-end conversion tracking stack (Meta Pixel + CAPI, GA4, Google Ads Enhanced Conversions, TikTok, LinkedIn, Pinterest, Reddit, Stripe Checkout, Consent Mode v2, coupon/offer pages) and produce a versioned Markdown report with what's done, what's missing, exact code snippets, and prioritized next steps. Use this whenever the user mentions setting up conversion tracking, ad pixels, Meta/Facebook Pixel, Conversions API, CAPI, GA4 ecommerce events, Google Ads conversion tracking, Enhanced Conversions, TikTok Events API, LinkedIn Insight Tag, Pinterest Tag, Reddit Pixel, Stripe Checkout success/webhook setup, Consent Mode v2, coupon/offer pages, discount attribution, UTM tracking, event deduplication, tracking audit, or "why aren't my conversions being tracked." Also use when adding analytics to an ecommerce site or checking a launch-readiness checklist for a store.
metadata:
  version: 1.0.0
---

# Conversion Tracking

An end-to-end audit skill for the 2026 ecommerce conversion tracking stack. Aimed at **custom / vibe-coded websites** (Next.js, Nuxt, Astro, SvelteKit, plain React/Vue, static + serverless, custom PHP/Node/Python backends). Works in any coding agent that can read files and write Markdown — Claude Code, Cursor, Cline, Aider, Windsurf, etc.

You (the coding agent) are an expert in server-side and browser-side conversion tracking. Your job when this skill triggers: **inspect the codebase, check every layer of the tracking stack against the 2026 best-practice checklist, and produce a single Markdown report** the user can read, share with their team, and act on. Then version the report on every subsequent run so progress is visible.

---

## What this skill covers

Ten interlocking areas. Each has a dedicated reference file. Read the reference before writing findings for that area.

| # | Area | Reference file |
|---|---|---|
| 1 | **Meta Pixel + Conversions API** | `references/meta-pixel-capi.md` |
| 2 | **Google Analytics 4** (browser + Measurement Protocol) | `references/ga4-and-measurement-protocol.md` |
| 3 | **Google Ads Enhanced Conversions** | `references/google-ads-enhanced-conversions.md` |
| 4 | **TikTok Pixel + Events API** | `references/tiktok-pixel-events-api.md` |
| 5 | **LinkedIn Insight Tag + Conversions API** | `references/linkedin-insight-tag-capi.md` |
| 6 | **Pinterest Tag + Conversions API** | `references/pinterest-tag-capi.md` |
| 7 | **Reddit Pixel + Conversions API** | `references/reddit-pixel-capi.md` |
| 8 | **Stripe Checkout integration** (success/cancel pages, webhooks, session metadata) | `references/stripe-checkout-integration.md` |
| 9 | **Consent Mode v2 + privacy** | `references/consent-mode-v2.md` |
| 10 | **Coupon & offer landing pages** (attribution, UTM strategy) | `references/coupon-and-offer-pages.md` |

Plus two cross-cutting references you should read *first*:

- `references/event-id-and-datalayer.md` — the unified `event_id` and dataLayer pattern the whole audit hangs on.
- `references/report-template.md` — the exact Markdown structure to output.

---

## Trigger check (before you do anything)

Before running, confirm the target is an ecommerce site — has products, a cart, and/or a paid checkout flow. If it's a lead-gen site with no purchase event, tell the user: "This skill is tuned for ecommerce (purchases, cart, checkout). For lead-gen you'll want most of it but skip the ecommerce-events sections — proceed anyway?" Then proceed if they confirm.

Also ask, if not obvious from the codebase:
- **Which ad platforms are actually in use or planned?** Skip audits for platforms they're not running (the report will mark them "N/A — not in scope").
- **Region focus?** (EEA/UK triggers a much stricter Consent Mode section.)
- **Is Stripe the payment processor?** If they're on PayPal/Adyen/Square/Braintree/etc., note it — the Stripe section becomes "Payment processor integration" and you'll adapt the guidance.

Keep this to 1–2 quick questions max. If the codebase makes it obvious, just proceed.

---

## Workflow

### Phase 1: Detect

Read the codebase to figure out what already exists. Do this before writing anything.

Check for (adjust paths to the framework):

- **HTML `<head>` / root layout** (`app/layout.tsx`, `pages/_document.tsx`, `src/app.html`, `index.html`, `_app.tsx`, `nuxt.config.ts`, `astro.config.mjs`, etc.) — look for `gtag`, `dataLayer`, `fbq`, `ttq`, `_linkedin_partner_id`, `pintrk`, `rdt`, `googletagmanager.com`, `connect.facebook.net`, `snap.licdn.com`, `analytics.tiktok.com`, `s.pinimg.com`, `redditstatic.com`.
- **`.env` / `.env.example` / `.env.local`** — look for tracking IDs (`NEXT_PUBLIC_GA_ID`, `META_PIXEL_ID`, `META_CAPI_TOKEN`, `TIKTOK_PIXEL_ID`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, etc.).
- **`package.json`** — look for `@stripe/stripe-js`, `stripe`, `@next/third-parties`, `react-facebook-pixel`, `@lemonsqueezy/*`, GTM packages, Consent Mode / CMP libraries.
- **API/server routes** — look for `/api/webhooks/stripe`, `/api/checkout`, `/api/track/*`, or equivalent (`pages/api/`, `app/api/`, `server/api/`, `functions/`, `pages/[...].ts` for edge functions).
- **Success/cancel/thank-you pages** — look for routes like `/success`, `/thank-you`, `/order-confirmed`, `/checkout/success`, `/cancel`, `/checkout/canceled`.
- **Cart / checkout page** — grep for `checkout`, `cart`, `stripe.checkout.sessions.create`, `PaymentIntent`.
- **Product / PDP files** — grep for product ID patterns, `add_to_cart`, `AddToCart`, buy button handlers.
- **Cookie banner / CMP** — look for `cookiebot`, `onetrust`, `cookieyes`, `usercentrics`, `cookiehub`, `iubenda`, `didomi`, or a homegrown banner.

You are looking at a snapshot in time. If something is present but broken/misconfigured, note it as `⚠️ Partial` rather than `✅ Done`.

### Phase 2: Audit against the checklist

Open `references/audit-checklist.md` if it exists, otherwise walk the areas in the table above one by one. For each area:

1. Read the reference file for that area.
2. Compare what's in the codebase against the checklist inside that reference.
3. Note status: `✅ Done` / `⚠️ Partial` / `❌ Missing` / `➖ N/A`.
4. Gather concrete code snippets (from the reference file) for anything Missing or Partial. **Use the exact framework/stack the user is on** — if it's Next.js App Router, write Next.js App Router code; if it's plain HTML + PHP, write that.
5. Note anything Beyond Scope (things the user needs to do in a platform UI, not in code) with clear "where to click" instructions.

### Phase 3: Write the versioned report

The report goes in `docs/conversion-tracking/` in the user's repo (create the folder if it doesn't exist). File naming:

```
docs/conversion-tracking/
├── audit-2026-08-21-v01.md     ← first run
├── audit-2026-08-28-v02.md     ← second run a week later
├── audit-2026-09-15-v03.md     ← third run
└── latest.md → audit-2026-09-15-v03.md   ← symlink or copy
```

Rules for versioning:
- Date is today's date, YYYY-MM-DD.
- Version increments on **every run**, even the same day (`v01`, `v02`, ...). If two runs happen on the same day, both keep that day's date.
- Before writing, `ls docs/conversion-tracking/` to find the highest existing version for today (or the most recent overall) and increment.
- On runs 2+, add a **"Since last audit"** section at the top comparing to the previous version (what got fixed, what's new, what regressed). Read the previous file to do this properly.
- Always update `latest.md` (copy, not symlink — more portable across OSes) so the user has one predictable path.

Use the structure in `references/report-template.md`. Do not deviate from the section order — the whole point of pre-defined sections is that users know where to find things across runs.

### Phase 4: Wrap up

After writing the file, tell the user:

- Where the file is (`docs/conversion-tracking/audit-YYYY-MM-DD-vNN.md`).
- The one-line summary (score + top 3 priorities).
- **Do NOT dump the full report into the chat.** It's long. The user opens the file.

If code snippets in the report reference environment variables the user doesn't have set, list them at the end of your reply so they can add them to `.env` immediately.

---

## The one thing that matters most

Read `references/event-id-and-datalayer.md` before writing anything else. The single highest-leverage architectural decision in this whole audit is: **one unified `event_id` per business event, generated once, threaded through every browser tag, every server-side API call, and Stripe session metadata.** If the user gets that right, deduplication works automatically across all seven ad platforms + GA4 + Stripe. If they get it wrong, they will silently double-count conversions on every platform and optimize toward fiction. Every reference file assumes this pattern is in place.

---

## Report structure (summary — full template in `references/report-template.md`)

```
# Conversion Tracking Audit — vNN — YYYY-MM-DD

## Executive Summary
- Overall readiness score (X/100)
- Top 3 priorities for this week
- Estimated conversion recovery if all fixes applied
- One-paragraph plain-English summary

## Since Last Audit (v02+)
- Fixed since last run
- New gaps found
- Regressions

## Stack Detected
- Framework, hosting, payment processor, CMP, ad platforms in play

## 1. Meta Pixel + Conversions API
   Status | What's in place | What's missing | Snippets | Next steps
## 2. Google Analytics 4
## 3. Google Ads Enhanced Conversions
## 4. TikTok Pixel + Events API
## 5. LinkedIn Insight Tag + CAPI       (skip if not in scope)
## 6. Pinterest Tag + CAPI               (skip if not in scope)
## 7. Reddit Pixel + CAPI                (skip if not in scope)
## 8. Stripe Checkout Integration
## 9. Consent Mode v2 + Privacy
## 10. Coupon & Offer Pages

## Cross-Platform: event_id Strategy
- The unified dedup ID plan for this site.

## Environment Variables Needed
- List of .env keys the user needs to set for anything in this report to work.

## QA & Validation Checklist
- Per-platform test steps + tools.

## Priority Roadmap
- Ordered list, highest leverage first, with estimated effort tags [S/M/L].
```

---

## Output format rules

- **One file per run.** Not multiple files. Not code files as siblings — put snippets inside the report as fenced code blocks with the correct language.
- **Markdown only.** No HTML, no PDFs, no docx — the whole point is that it renders in every code editor, GitHub, and every markdown viewer.
- **Fenced code blocks must specify language** (` ```js `, ` ```tsx `, ` ```html `, ` ```bash `, ` ```env `, ` ```json `) so syntax highlighting works.
- **Every code snippet must be complete enough to paste in** — no `// ... rest of the code` placeholders. If it needs context, show the context.
- **Absolute file paths** where recommending code changes: `app/api/webhooks/stripe/route.ts` not "your webhook handler".
- **Status emoji legend** (used consistently through the report): `✅ Done` `⚠️ Partial` `❌ Missing` `➖ N/A` `🚨 Critical` `💡 Optional`.
- **No hyping.** Direct, founder-to-founder. If something is broken, say it's broken. If something is fine, say it's fine.
- **Cite official docs** at the end of each section — one or two links per platform to the canonical setup page.

---

## Portability notes

This skill is designed to run identically in any coding agent. The only assumed capabilities are:

1. Read files in a project.
2. Grep / search across files.
3. Write a Markdown file into a folder.
4. Optionally: list existing files (for versioning).

No Claude-specific tools required. If the agent supports web-search, it can optionally verify platform IDs against live documentation — but the reference files already contain everything needed for a full audit offline.

To install in another agent:
- **Cursor:** Drop the whole `conversion-tracking/` folder into `.cursor/rules/` and reference `SKILL.md` in a rule file.
- **Cline / Roo / Aider:** Drop the folder anywhere in the project and open with `@conversion-tracking/SKILL.md` when needed.
- **Claude Code:** Drop into `~/.claude/skills/conversion-tracking/` (personal) or `.claude/skills/conversion-tracking/` (project).
- **VS Code + Continue / Copilot Chat:** Reference the SKILL.md path in a `.continue/config.json` slash-command or chat context.

---

## What this skill deliberately does NOT do

- Doesn't automatically **install** any tracking code — it recommends and provides snippets, the user (or the user + their coding agent in a follow-up chat) makes the changes.
- Doesn't touch platform UIs (Meta Events Manager, Google Ads, TikTok Business Center, etc.) — it tells the user exactly what to do there.
- Doesn't do post-launch performance analysis (that's a different job — attribution modeling, cohort analysis, LTV).
- Doesn't cover **app-side** (iOS/Android) tracking — web only. Mention if detected but out of scope.
- Doesn't cover **email/SMS attribution** past the UTM structure — Klaviyo, Attentive, Postscript setup is out of scope, but their UTMs get audited in the coupon/offer section.

---

## Related skills

- `schema` — Structured data / rich results markup.
- `seo-audit` — Broader SEO health.
- `ai-seo` — Optimization for AI search / answer engines.

If the user asks for tracking *and* SEO, run each skill separately and produce separate reports. Don't merge them.
