# Managed API Access + Multi-Platform Distribution — Long-Term Plan

> ⛔ **SUPERSEDED 2026-07-03 by `DISTRIBUTION_PLAN.md`.** Decision: the developer will **not** take revenue / manage paid API access (too hard — revenue, tax, store cuts, backend, liability). Instead the app **guides each user to create their own Gemini + Mistral keys**. This doc is kept for the rationale and the still-valid store/iOS-port research (§5–6); its billing/proxy/payments sections are moot.

> **Living document.** Update the Status column and the Changelog as implementation proceeds.
> Created 2026-07-03. Owner: Charles Petersen (@stanford.edu). Implementation: Claude Code.

## How to read this doc
- **Feasibility** ratings: 🟢 straightforward · 🟡 doable with care · 🟠 hard / real risk · 🔴 blocker or not-Claude-feasible.
- **[USER]** = a step only the human can do (accounts, signing, console forms, payments, legal). Claude Code cannot do these.
- **[CLAUDE]** = Claude Code can implement this in code.
- Status values: `NOT STARTED` · `IN PROGRESS` · `DONE` · `BLOCKED` · `DROPPED`.

---

## 1. Context & goal

The main barrier to wide adoption is that most users can't create their own LLM API keys. To fix that, the app should let users **pay for OCR through the app** using API access the developer manages — **Gemini + Mistral only**. This is **not for profit**; the aim is the **widest, easiest distribution**. The existing **bring-your-own-key / API-gateway** option stays for power users. In scope: an **iPhone app alongside the Android app**, and getting all apps into **Apple's and Google's stores**. It must **work at scale and be bulletproof**.

**Overall feasibility verdict: WORKABLE 🟢, with caveats.** Every piece of *software* here is a standard, well-documented pattern that Claude Code can build (proxy, metering, receipt/payment validation, StoreKit/Stripe, iOS app, shared Swift package). The real gating factors are **not engineering**:
1. **Business/economics** — store commissions vs. a not-for-profit pass-through, and you fronting everyone's API bill.
2. **Ops & liability** — you become an operator of a paid, multi-tenant service (uptime, abuse, key custody, taxes, ToS/privacy, possibly a legal entity).
3. **Human logistics** — accounts, signing, submission, legal review.

If those are acceptable, proceed. If running a paid backend service (with real money at stake for every user) is not something you want to own operationally, **do not pursue the managed path** — keep BYO-key/gateway only. Nothing below is blocked by Claude Code's abilities; the honest risk is the ongoing operational/financial commitment.

---

## 2. The one pivotal decision (decide before building)

**How is the Mac app distributed, and therefore how do users pay?** This drives everything downstream.

| Path | Platform cut | Reach | Notes |
|---|---|---|---|
| **A. Mac app as a notarized direct-download DMG (outside the Mac App Store) + Stripe web checkout for credits** ⭐ recommended | **0% Apple cut** (Apple IAP rules don't apply outside the Mac App Store) | Global | Best fit for "not for profit, widest reach." Loses Mac App Store discoverability. Stripe ~2.9%+30¢. |
| **B. Mac App Store + StoreKit consumable IAP for credits** | **15%** (Small Business Program, <$1M/yr) or 30% | Global | Requires grossing-up credit prices to still cover the API bill. Full App Review + IAP setup. |

Sources: Apple Guideline 3.1.1 requires IAP for in-app "credits"; a Mac app distributed **outside** the Mac App Store is not bound by Apple IAP at all (developer.apple.com/app-store/review/guidelines/; developer.apple.com/documentation/storekit/choosing-a-receipt-validation-technique).

**Recommendation: Path A** (notarized DMG + Stripe). It avoids the store tax entirely, works for all users globally, and keeps money off the mobile apps (see §3). Path B is only worth it if Mac App Store presence matters more than the ~15% cut and price gross-up. **Build the code so the credit source is pluggable** (Stripe module *and* a StoreKit module behind a build flag) so this can change later.

> **[USER] DECISION D1 — Mac distribution & payment path:** `UNDECIDED` (recommend A).

---

## 3. Recommended architecture (the big picture)

Keep **all money and all API keys off the client apps.** Three ideas make this clean and bulletproof:

1. **A server-side "OCR proxy" holds the master Gemini/Mistral keys.** Clients never see them. The proxy authenticates the user, **meters** usage, **hard-caps** spend per user and globally, and forwards the OCR call. Shipping a master key in a distributed app would leak it — the proxy is mandatory, not optional. (Both providers expect this: Gemini's docs recommend a backend proxy; Mistral formally sanctions the "End User Account" pattern where your users consume via your key and you're responsible for them.)

2. **The mobile apps (Android + new iPhone) are FREE capture companions with NO in-app purchase.** They photograph documents and stream them to the Mac; **the Mac does the OCR** (via the proxy). Because the phones sell nothing, **Google Play Billing and Apple IAP simply don't apply to them** — the single cleanest way to sidestep store billing. (Google Payments policy: a free app selling no in-app digital goods has no billing obligation.)

3. **Credits are bought and consumed on the Mac side.** On Path A that's a Stripe web checkout; the Mac app redeems a server-tracked balance. The BYO-key/gateway path stays as an alternative for power users.

```
[iPhone companion]  [Android companion]        (free; no purchases; no keys)
        \                 /
         \  stream photos (existing CaptureServer: /ping, /photo, /session/complete)
          v               v
        [  macOS app  ]  --- OCR request (user token) --->  [ OCR PROXY (your server) ]
             ^  buy credits (Stripe web / or StoreKit IAP)        |  holds Gemini+Mistral keys
             |                                                     |  meters + per-user & global caps
        [ Stripe / RevenueCat ] --- validated purchase --->  [ credit ledger DB ]
                                                                   |
                                                     Gemini API (paid) / Mistral API (paid)
```

**Stack (solo-dev-friendly, all buildable by Claude Code):**
- **Proxy**: Cloudflare Workers (+ Durable Objects for race-free balances/rate-limits) with a D1/Postgres ledger, *or* Firebase Functions + Firestore. Master keys stored as platform secrets.
- **Payments**: Stripe (Path A) and/or RevenueCat in front of StoreKit/Play Billing (Path B / future). RevenueCat validates Apple ASSN v2 / Google RTDN receipts and offers a built-in "Virtual Currency" credits ledger.
- **Providers**: Gemini (Google Cloud, **paid tier**) + Mistral (**Scale plan**, training off).
- **Clients**: existing macOS app + Android companion, new iOS companion, sharing a new **`ArchiveCore`** Swift package.

---

## 4. Phased plan

### Phase 0 — Decisions, accounts, legal  ·  Status: `NOT STARTED`  ·  🟡
Foundational, mostly **[USER]**. Nothing to code yet.
- **[USER]** Resolve **D1** (Mac distribution/payment path, §2).
- **[USER]** Create + fund provider accounts, **paid tiers**, with hard budget caps:
  - Google Cloud project → enable **billing** (paid Gemini tier; required for privacy + terms). Set a Cloud Budget alert + cap.
  - Mistral account/workspace → **Scale plan**; Admin Console → Privacy → training toggle **OFF**; set a monthly workspace spend cap.
- **[USER]** Apple Developer Program ($99/yr) — **check the nonprofit/edu fee waiver** given your @stanford.edu affiliation. Google Play Console ($25 one-time).
- **[USER]** Stripe account (Path A) with KYC/bank/tax — and/or plan StoreKit (Path B).
- **[USER] / legal**: decide the **entity** that collects money and fronts the API bill (personal vs. an LLC/nonprofit), and get **light legal/accounting review** of the not-for-profit pass-through, sales-tax/VAT on digital goods, and reliance on any store exceptions. 🟠 This is the least "Claude-feasible" area and the biggest real-world commitment.
- **[USER]** Write/host a **public privacy policy URL** (both stores require it) — **[CLAUDE]** can draft the text; you host it.
- **[USER]** Confirm with **Mistral** (email sales/support) that your multi-user "End User Account" OCR app is authorized (not resale) — get it in writing before scaling. Gemini has no equivalent sign-off but note the April-2026 "for professional/business, not consumer use" clause and **position toward archivists/researchers**.

**Feasibility note:** 🟢 code-wise (nothing to build yet); 🟠 real-world (this phase is all human commitment, money, and a little legal). If Phase 0 stalls, the whole managed path stalls — that's expected and is the honest go/no-go gate.

### Phase 1 — OCR proxy backend (MVP)  ·  Status: `NOT STARTED`  ·  🟢 [CLAUDE builds] / 🟡 [USER deploys]
The heart of the system. Provider-agnostic; Gemini + Mistral + a pass-through mode for BYO-key/gateway.
- **[CLAUDE]** Worker/Function that holds keys as secrets and forwards OCR (`generateContent` for Gemini; `/v1/ocr` for Mistral); EU/US endpoint selection for Mistral.
- **[CLAUDE]** **Metering**: count tokens (Gemini) / pages (Mistral) per user; persist usage.
- **[CLAUDE]** **Credit ledger** with atomic **reserve → commit → refund** and **idempotency keys** (retries never double-charge).
- **[CLAUDE]** **Hard caps + circuit breaker**: per-user daily/lifetime spend cap, global daily cap, and a breaker that trips if provider spend crosses a threshold. *This is the #1 safeguard — one shared billing account pays for everyone, so a leaked token or abusive user must not be able to drain the budget.*
- **[CLAUDE]** Per-user **rate limiting** + request **queueing/backoff** on 429s (both providers throttle per-account; Mistral especially — limits rise only with cumulative spend).
- **[CLAUDE]** **Auth**: anonymous device identity now, with optional Sign in with Apple / Google upgrade later so a reinstall doesn't lose credits.
- **[CLAUDE]** **Batch API** routing for non-urgent bulk jobs (Gemini Batch = 50% off; Mistral Batch = ~50% off but no ZDR) — keep live/interactive OCR on the standard API.
- **[USER]** Create the Cloudflare/Firebase account, set DNS/custom domain, paste secrets, approve deploys, set the global monthly budget.

**Feasibility:** 🟢 to build (standard patterns). The only "gotcha" is that correctness here is financially load-bearing — hence adversarial review + tests before real keys.

### Phase 2 — Payments / credits  ·  Status: `NOT STARTED`  ·  🟢 [CLAUDE] / 🟡 [USER]
Grant credits **only after server-side validation**; sell consumable **"page-credit" packs** (not subscriptions).
- **Path A (Stripe, recommended):** **[CLAUDE]** Stripe Checkout + webhook that credits the ledger after `checkout.session.completed`; **[USER]** Stripe account/keys, tax settings. Works globally, ~0% platform cut beyond Stripe's fee. On iOS/Play this must **not** appear as an in-app purchase flow (see §5) — it lives on the Mac/web.
- **Path B (StoreKit/Play IAP, if D1=B or a future MAS build):** **[CLAUDE]** StoreKit 2 consumable flow + App Store Server API validation + **App Store Server Notifications v2** webhook (`ONE_TIME_CHARGE` → grant; `CONSUMPTION_REQUEST` → fraud reply); optionally via **RevenueCat** to offload receipt validation and get a credits ledger. **[USER]** create IAP products, ASSN v2 endpoint, sign Paid Apps Agreement + banking/tax, enroll Small Business Program (15%).
- **[CLAUDE]** **Refund/chargeback handling**: consume Apple/Google/Stripe refund webhooks, claw back or zero balances (accept some loss on already-consumed credits).
- **[CLAUDE]** Pricing is **config-driven**: pack price = `API_cost / (1 − platform_cut)` + a variance margin (Gemini per-token cost varies with document length, so a flat "per page" credit can under-recover on long pages).

**Feasibility:** 🟢 code; 🟡 the economics decision is yours (§8).

### Phase 3 — Wire the macOS app to managed access  ·  Status: `NOT STARTED`  ·  🟢 [CLAUDE]
- **[CLAUDE]** Add a third OCR mode next to "own key" and "gateway": **"Managed (pay-as-you-go)"** — routes OCR through the proxy with the user's token; shows a **credit balance + buy button** (Stripe web on Path A). Reuse the existing provider/model plumbing; the proxy speaks the same request/response shape.
- **[CLAUDE]** Keep **BYO-key/gateway** fully intact (explicit requirement).
- **[CLAUDE]** Handle out-of-credits, network, and cap-exceeded states gracefully in the UI (with `?` help + gray-out per the settings convention).

### Phase 4 — iPhone capture companion + shared `ArchiveCore`  ·  Status: `NOT STARTED`  ·  🟢 [CLAUDE builds] / 🟡 [USER signs/submits]
The macOS differentiator (Finder tags) doesn't exist on iOS, so **Option A: an iPhone capture companion at parity with Android** (recommended). It streams to the Mac's existing `CaptureServer` (`/ping`, `/photo`, `/session/complete`) — **no Mac-side changes needed**.
- **[CLAUDE]** Extract a multiplatform Swift package **`ArchiveCore`** from the reusable core (OCR clients, `ImageEncoding`, `NetworkSession`, `KeychainHelper`, `PDFGenerator` — all Foundation/ImageIO/CoreGraphics/Security; only cosmetic AppKit imports to strip). Guard platform code with `#if canImport(AppKit)` / `#if os(iOS)`. Roughly **40–60%** of the Swift core is reusable.
- **[CLAUDE]** New iOS app: SwiftUI + **AVFoundation** camera, **QR pairing** (AVCaptureMetadataOutput), `PhotosPicker` import, an HTTP client mirroring Android's `MacClient.kt`, Keychain-backed session token. Mirror the recent Android UX (segment-transfer animation, no pile-up).
- **[CLAUDE]** Info.plist strings (`NSCameraUsageDescription`, `NSLocalNetworkUsageDescription`, `NSBonjourServices`), `ITSAppUsesNonExemptEncryption`, `PrivacyInfo.xcprivacy`.
- **[USER]** Xcode signing/provisioning, App Store Connect record, TestFlight, submit. (macOS app project uses XcodeGen — mirror that for the iOS target / SPM.)

> **[USER] DECISION D2 — iPhone app scope:** Option A (capture companion, recommended) vs Option B (standalone iOS processor). Status: `UNDECIDED`. Option B reuses more code but discards the Finder-tags differentiator, needs a tags-in-PDF/sidecar substitute, and **triggers Apple IAP** for any on-device paid OCR (30%/15%). Only pursue B for Mac-less users.

### Phase 5 — Store distribution  ·  Status: `NOT STARTED`  ·  🟡 [CLAUDE drafts content] / 🔴 [USER-only submission]
- **macOS:** Path A → **[USER]** notarize + staple the DMG (Developer ID), host the download. Path B → Mac App Store submission.
- **iOS (App Store):** **[USER]** signing, App Privacy nutrition labels, export-compliance, review. **[CLAUDE]** drafts all privacy/label/answer text. Risk: an app "useless without a paired Mac" can draw **2.1/4.2 minimal-functionality** scrutiny — the listing + first-run must clearly state it needs the Mac app.
- **Android (Play):** **[USER]** Play Console, Data Safety form, **targetSdk 36** (mandatory for new apps/updates by 2026-08-31), signing, upload. New personal Play accounts face a **closed-testing gate (12 testers / 14 days)** before production — plan for the delay. **[CLAUDE]** sets Gradle targetSdk and drafts Data Safety answers.

**Feasibility:** 🔴 for the submission mechanics themselves (Claude cannot click console forms, sign, or submit) — but 🟢 for all the *content* Claude prepares.

### Phase 6 — Scale-hardening, monitoring, legal  ·  Status: `NOT STARTED`  ·  🟡
- **[CLAUDE]** Observability (usage/spend dashboards or logs), anomaly/abuse detection, per-user suspension, an in-app **ToS/acceptable-use** so one bad user can't get your master provider account suspended (their content runs under your key).
- **[USER]** Request **quota increases early** (Gemini Tier 2/3; Mistral before Tier 4) *before* you have many concurrent users. Renew ASSN/notification certs. Watch provider price/model changes (keep routing config-driven).
- **[USER] / legal:** privacy compliance (US state laws effective 2026), tax on digital sales, and re-verify store guidelines + the Apple external-link fee status (SCOTUS ruling expected ~June 2027) before launch.

---

## 5. Store-billing rules — the crux (why the design is what it is)
- **Apple:** In-app "credits"/AI processing consumed in the app = **IAP required** (Guideline 3.1.1); the 3.1.3 "consumed outside the app" exception is **physical** goods only, not digital OCR. **US storefront only**, post-Epic, you *may* link out to Stripe with no entitlement — but that's US-only and the allowed commission is **legally unsettled** (SCOTUS cert granted 2026-06-30). ⇒ Don't build a Stripe-only *in-app* flow for a Mac App Store/iOS build outside the US. **A notarized Mac DMG avoids all of this.**
- **Google:** Play Billing applies to in-app digital purchases. A **free companion that sells nothing has no billing obligation** — the clean path. US alternative-billing exists but is in flux (Jan-2026 settlement hearing) and still costs fees.
- **Net:** Keep purchasing **off the phones**; sell on the Mac (Stripe, Path A). Both mobile apps stay free → no store billing at all.

## 6. Provider notes
**Gemini** (Google Cloud, **paid tier mandatory** — free tier trains on data & is disallowed for EEA/UK/CH): multi-tenant proxy is standard and permitted; no per-end-user keys; paid tier is **not** trained on and supports ZDR. Scaling ceiling is **quota** (Tier 1 ~150–300 RPM → Tier 3 4,000+ RPM with spend + a form), not law. Watch the "not for consumer use" clause → position as professional/archival. Budget (per 1M tokens, paid standard; ~½ for Batch — **re-verify before launch**): flash-lite $0.10/$0.40, flash $0.30/$2.50, pro $1.25/$10 (≤200k ctx).
**Mistral** (**Scale plan**, training off): the **"End User Account"** pattern (users consume via your key; you're responsible for them) is explicitly sanctioned — but §2.2(h) **prohibits selling/transferring keys**, so the proxy is mandatory and never expose the key; don't frame it as resale. `mistral-ocr` ≈ **$2–4 / 1,000 pages** (OCR 4 ~2× OCR 3 — pin a dated snapshot, re-verify). EU-hosted default; ZDR available for real-time (not Batch). Single-workspace rate limit is the main ceiling; **get written authorization** before scaling.

## 7. What Claude Code can vs. cannot do
| Can build (🟢) | Cannot do (🔴 human-only) |
|---|---|
| The whole OCR proxy: metering, ledger, idempotency, per-user + global caps, circuit breaker, queueing/backoff, Batch routing | Create/fund Google Cloud, Mistral, Apple, Google Play, Stripe, RevenueCat, Cloudflare/Firebase accounts |
| StoreKit 2 IAP + App Store Server API/ASSN v2 validation; Stripe Checkout + webhooks; RevenueCat integration | Configure IAP/Play products, pricing, agreements, banking/tax, Small Business Program |
| iOS capture companion, AVFoundation camera, QR pairing; shared `ArchiveCore` SPM package; wire Mac/Android to the proxy | Interactive Xcode signing/provisioning/notarization; TestFlight; store submission; console web forms |
| Privacy policy / App Privacy / Data Safety / export-compliance **text**; Gradle targetSdk 36 | Host the privacy policy; DNS/secrets install; deploy approvals; legal/tax/entity decisions; DUNS/identity verification |

## 8. Economics (not-for-profit pass-through)
- **Path A (Stripe/DMG):** user pays ≈ API cost + Stripe ~2.9%+30¢. Cleanest.
- **Path B (IAP):** platform takes 15% (SBP) / 30% **of gross**, so prices must gross-up (`net = gross × (1 − cut)`), e.g. to net $10 of API cost at 15% you charge ≈ $11.76. Apple gives no nonprofit relief on consumables.
- **Structural risk:** you front **everyone's** bill; flat "per-page" credits can under-recover on long/verbose pages. Meter at **cost + small buffer**, enforce caps, keep pricing config-driven.

## 9. Risks & things to re-verify before launch
- Shared-billing cost runaway → **hard caps + circuit breaker + provider budgets** are load-bearing (build first).
- Refund-after-consume fraud → refund webhooks + clawback.
- Anonymous balances lost on reinstall → add Sign in with Apple/Google linking before wide launch.
- Gemini "not for consumer use" gray area; Mistral written authorization; Apple external-link fee (SCOTUS ~2027); Play US alt-billing (Jan-2026 hearing) — **re-verify all store guidelines & provider terms/pricing immediately before launch** (they changed repeatedly through 2024–2026).
- Ops/liability: uptime, abuse handling, ToS, privacy law (2026 US state laws), sales tax/VAT — a genuine ongoing commitment, not a one-time build.

## 10. Open decisions (fill in as resolved)
- **D1** Mac distribution/payment path (A recommended) — `UNDECIDED`
- **D2** iPhone scope: companion (A) vs standalone (B) — `UNDECIDED`
- **D3** Backend host: Cloudflare Workers vs Firebase — `UNDECIDED`
- **D4** Payments: Stripe-only vs RevenueCat(+IAP) vs both — `UNDECIDED`
- **D5** Legal entity for collecting payments / fronting API cost — `UNDECIDED`

## 11. Sources (verify before launch — fast-moving)
- Apple App Store Review Guidelines 3.1.1/3.1.3 — developer.apple.com/app-store/review/guidelines/
- Apple Small Business Program (15%) — developer.apple.com/app-store/small-business-program/
- App Store Server Notifications v2 — developer.apple.com/documentation/appstoreservernotifications
- Apple external-purchase-link status — macrumors.com/2025/12/11/apple-app-store-fees-external-payment-links/
- Google Play Payments policy & alternative billing — support.google.com/googleplay/android-developer (Data Safety, billing)
- Gemini API terms/pricing — ai.google.dev / ai.google.dev/gemini-api/docs/pricing
- Mistral commercial terms & pricing — legal.mistral.ai/terms/commercial-terms-of-service ; mistral.ai/pricing
- RevenueCat (cross-platform IAP + Virtual Currency) — revenuecat.com ; Cloudflare Workers/Durable Objects — developers.cloudflare.com

## 12. Changelog
- 2026-07-03 — Initial plan created from a 6-track web-research sweep (Apple billing, Google billing, Gemini terms, Mistral terms, backend+payments architecture, iOS port + store submission). No code written yet; awaiting decisions D1–D5.
