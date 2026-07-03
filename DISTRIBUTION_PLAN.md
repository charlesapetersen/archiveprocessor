# Distribution Plan — Guided "Bring Your Own Key" + iPhone app + App Stores

> **This is the ACTIVE plan.** It supersedes `MANAGED_ACCESS_PLAN.md` (managed/paid API access — dropped because taking revenue is too hard). Here we ease distribution the same way **minus revenue**: the app **walks each user through creating their own Gemini + Mistral keys**, made easy and bulletproof. Living document — update Status/Changelog as work proceeds. Created 2026-07-03.

## How to read
- Feasibility: 🟢 straightforward · 🟡 doable with care · 🟠 real risk · 🔴 blocker / not Claude-feasible.
- **[USER]** = human-only (accounts, signing, submission, screenshots, hosting, verifying provider UI). **[CLAUDE]** = Claude Code builds it.
- Status: `NOT STARTED` · `IN PROGRESS` · `DONE` · `BLOCKED`.

---

## ▶ Build progress — RESUME HERE (updated after every increment, for durability)
**Phase 1 (guided key onboarding, macOS) — `DONE` ✅ (build-verified; adversarially reviewed; pending push).** Small, build-verified, committed increments so an interruption never loses work.
- [x] **1a** `OCR/KeyValidator.swift` — live validation calls + `KeyStatus` + error→plain-English map ✅ build-verified
- [x] **1b** `Models/ProviderKeySpec.swift` — Gemini + Mistral specs (deep links, steps, notes, precheck) ✅ build-verified (Sendable + @Sendable closures)
- [x] **1c** `Views/ProviderKeyWizard.swift` — reusable wizard (explain → open page → paste → validate → status) ✅ build-verified
- [x] **1d** `Views/SettingsView.swift` — "Set up keys (guided)" button + wizard sheet + Validated/Saved chips; manual edit clears validated flag ✅ build-verified · committed `c6957d8`
- [x] **1e** first-run wizard (ContentView, when no key present; skipped in test mode) + `SampleOCRTester` (synthetic-image end-to-end OCR test) + wizard "Test OCR on a sample page" step ✅ build-verified
- [x] **1f** adversarial review (5-agent workflow) → fixed 2 confirmed UX bugs: (i) stale OCR-test result surviving a key change/re-validate; (ii) Settings field not syncing after the wizard saves + a latent bug where opening Settings reset the Validated flag (keyField.onChange now ignores programmatic reloads) ✅ build-verified
Convention per increment: add files → `xcodegen generate` (if new files) → `xcodebuild -scheme ArchiveProcessor -configuration Debug … build` must succeed → commit locally → tick the box + set NEXT ACTION here.
Live-grounded: Gemini validation endpoint returns 200 for a good key, 400/API_KEY_INVALID for a bad one (matches `KeyValidator`).
**NEXT ACTION:** push Phase 1, then start **Phase 2** polish — clipboard-detect paste banner, EEA/UK/CH locale pre-warn on Gemini, 429 batch throttle/backoff, `[USER]` screenshots/GIFs + verify provider wording; then **Phase 3** (extract `ArchiveCore` + iPhone companion). ⚠️ Still unverified: whether Mistral OCR needs the user's own card (free-tier OCR) — confirm via the wizard's Test-OCR against a real free Mistral key.

## 1. Context & goal
Adoption is blocked because non-technical users (historians/archivists) can't make their own API keys. Rather than the developer selling API access (too hard: revenue, tax, store cuts, backend, liability — see the superseded plan), the app will **guide each user to create their own free Gemini + Mistral keys in ~2–3 minutes each**, validate them, and store them in the Keychain. Keep the existing bring-your-own-key/gateway path (this *is* that path, upgraded). Also ship an **iPhone capture companion** alongside the Android one and get all apps into the stores.

## 2. Verdict — highly feasible, low risk 🟢
This is dramatically simpler than the managed plan and removes every hard part:
- **No backend, no proxy, no server keys, no metering, no spend caps.** Keys live only in the user's Keychain (already implemented). Each user's API cost is billed by Google/Mistral **directly to them**; the developer handles no money.
- **No payments, no IAP, no Play Billing, no store commission, no legal entity, no tax.** All three apps are **free** and sell nothing → Apple/Google billing rules don't apply at all.
- **Almost 100% Claude-buildable.** The only human tasks are the unavoidable logistics (dev accounts, signing, submission), plus supplying current **screenshots/GIFs** of the provider sign-up pages and a one-time **verification of provider UI wording**, and hosting a privacy-policy URL.

**The main risk is not code — it's provider drift:** sign-up UIs, free-tier limits, and data-use terms change (Google cut free limits in Dec 2025; Mistral OCR's free-tier availability is unclear). The design mitigates this by (a) validating keys live and mapping errors to plain-English fixes, and (b) keeping instructions provider-agnostic. A few facts are flagged **⚠️ VERIFY BEFORE SHIP** below.

## 3. Architecture — entirely client-side
No new services. The app gains a **guided key-onboarding layer** on top of the existing per-provider Keychain storage and OCR clients.
```
First run ─▶ Onboarding wizard ─▶ [Gemini step] [Mistral step]  (each independent, skippable)
                                     │ open provider page in browser
                                     │ paste key (clipboard-detected, trimmed)
                                     │ Validate (cheap live call → plain-English verdict)
                                     │ Test OCR on a bundled sample page (proves end-to-end)
                                     ▼
                               Keychain (existing, per-provider)  ─▶ OCR pipeline (unchanged)
Settings ▶ "API Keys" ▶ per-provider status chips + "Get a key"/"Validate"/"Test" (re-runnable)
```
BYO-key/gateway for power users stays intact. Nothing about the OCR/PDF/tagging pipeline changes.

## 4. CORE FEATURE — Guided key onboarding (detailed, execution-ready)

### 4.1 Reusable wizard, driven by a per-provider spec
Build **one** `ProviderKeyWizard` view parameterized by a `ProviderKeySpec` (never two forks). Instantiate for Gemini and Mistral.

`Models/ProviderKeySpec.swift` (new) — `[CLAUDE]`:
```
struct ProviderKeySpec {
    let account: String              // Keychain account = LLMProvider.rawValue ("Gemini"/"Mistral")
    let displayName: String          // "Google Gemini", "Mistral"
    let blurb: String                // plain-language what/why + time + "no credit card to start"
    let signInURL: URL               // deep link (below)
    let steps: [String]              // on-screen numbered instructions mirroring the live site
    let keyPrecheck: (String) -> Bool// loose format check (Gemini ^AIza…; Mistral len≥20 & no spaces)
    let costNote, privacyNote, cardNote: String
    // validation + sample-OCR handled by KeyValidator (below)
}
```

### 4.2 Validation + plain-English errors
`OCR/KeyValidator.swift` (new) — `[CLAUDE]`. Two probes per provider:
- **Auth-validate** (instant, ~free): Gemini `GET https://generativelanguage.googleapis.com/v1beta/models?key=KEY` (or `…/models/gemini-2.5-flash-lite:countTokens?key=KEY`); Mistral `GET https://api.mistral.ai/v1/models` with `Authorization: Bearer KEY` (also checks the list contains `mistral-ocr-latest`).
- **End-to-end OCR test** (final confirmation): run the real OCR path on a small **bundled sample image** — Gemini `generateContent` on `gemini-2.5-flash-lite`; Mistral `POST /v1/ocr` `mistral-ocr-latest`. This surfaces region/billing/plan problems *now*, not mid-batch.

`enum KeyStatus { works, invalidKey, needsBilling(url), ocrNotEnabled(url), rateLimited, offline, providerBusy }` → one plain sentence + one action button. **Never show raw JSON/HTTP codes** (log them for support). Treat **429** and **offline/5xx** as "not your fault" (not a bad key).

**Error map (from research):**
| Gemini | Meaning | Message + action |
|---|---|---|
| 200 | valid | ✓ "Your Gemini key works." |
| 400 `API_KEY_INVALID` | wrong/mistyped/expired/leaked | "That key isn't valid — re-copy it." (Paste again) |
| 400 `FAILED_PRECONDITION` | free tier not available in region / billing off | "Your region needs billing enabled — set it up in AI Studio, or use Mistral instead." (Open billing / fallback) |
| 403 `PERMISSION_DENIED` | key blocked / API not enabled / restriction | "That key isn't allowed — create a fresh key." |
| 429 `RESOURCE_EXHAUSTED` | quota/rate | treat as **success** in onboarding: "Key works; you're at the free-tier limit — it'll retry." |
| 5xx / offline | transient | "Google's busy / you're offline — Retry." |

| Mistral | Meaning | Message + action |
|---|---|---|
| 200 (+ `mistral-ocr-latest` present) | valid & OCR-capable | ✓ "Your Mistral key works." |
| 200 but OCR test → plan/billing error (402/403/400) | free tier lacks OCR | "Your key works, but OCR needs a paid plan — add your own card in Mistral (charges go to Mistral, not us)." (Open billing) |
| 401 `authentication_error` | invalid/expired | "That key isn't valid — re-copy it from console.mistral.ai." |
| 429 `rate_limit_error` | rate | treat as **success**: "Key works; free-tier rate limit — it'll retry." |
| 5xx / offline | transient | "Mistral's busy / you're offline — Retry." |

### 4.3 Wizard steps (per provider, from research)
1. **Explain** — plain language: what the key is ("your private access code"), ~3 min, no card to start, you control cost/privacy.
2. **Open the page** — one primary button → `NSWorkspace.shared.open` (macOS) / `UIApplication.shared.open` (iOS) to the deep link (§5). Numbered on-screen instructions mirroring the live site + (later) screenshots.
3. **Paste** — on return, detect a key on the clipboard (`NSPasteboard`/`UIPasteboard`) and offer one-tap paste; **auto-trim** whitespace/quotes/newlines; masked `SecureField` with a show/hide eye; show only last-4 after save. Store via existing `KeychainHelper` (per-provider account).
4. **Validate** — auth-validate → status chip.
5. **Test OCR on a sample page** — real OCR on a bundled image; show extracted text; only then mark provider **Ready**.
6. **Status & skip** — persist a 4-state chip per provider (Not started / Key saved / Validated / OCR-tested); each provider independent and skippable; the app is fully usable with **one** key.

### 4.4 Integration points (execution map)
- **New:** `Models/ProviderKeySpec.swift`, `OCR/KeyValidator.swift`, `Views/ProviderKeyWizard.swift`, a bundled `Resources/SampleDoc.jpg` (small archival page for the OCR test — `[USER]` may supply a nicer one; `[CLAUDE]` bundles a placeholder from `Test Files/`).
- **Enhance:** `Views/SettingsView.swift` `apiKeySection`/`keyField` (add per-row **"How to get a key"** + **"Validate"** + status chip; add a **"Set up keys"** button that opens the wizard). Reuse the existing `keyField` Keychain save + `apiKeyChanged` notification.
- **First run:** in `ContentView`/`ArchiveProcessorApp`, if no provider key is present, present the onboarding wizard (dismissible; re-openable from Settings).
- **Reuse:** `KeychainHelper` (done), existing OCR clients for the sample-OCR test.
- **iOS:** same wizard via shared `ArchiveCore` (Phase 3).
- `xcodegen generate` after adding files.

## 5. Provider playbooks (⚠️ = re-verify against the live site before shipping copy/screenshots)

### Gemini (recommended first — lowest friction)
- **Deep link:** `https://aistudio.google.com/apikey`
- **Steps:** sign in with any Google account → **Create API key** → choose **"Create API key in new project"** (AI Studio auto-creates the project; never send them to Cloud Console) → copy the `AIza…` key → paste.
- **Free tier:** no credit card; models `gemini-2.5-flash` / `-flash-lite` (good for OCR). Rate limits are **dynamic** (Google cut them ~Dec 2025, ≈10 RPM / ~250 RPD on 2.5 Flash) — **don't hardcode**; link `https://aistudio.google.com/rate-limit`, throttle/queue bulk jobs, and treat 429 as "going as fast as the free tier allows."
- **Privacy:** free tier **may train on inputs/outputs and humans may review them** — warn before OCRing sensitive/PII material; **paid tier does not train.** Offer a "Set up billing (private)" deep link.
- **Region:** **EEA/UK/Switzerland must use the paid tier** (Google terms) → those users hit `FAILED_PRECONDITION`; detect locale and pre-warn.
- ⚠️ **VERIFY:** one source claims unrestricted keys are rejected after **2026-06-19** and users must "Restrict to Gemini API." Treat as an optional nag until confirmed against the live console.

### Mistral (optional/secondary)
- **Deep links:** `https://console.mistral.ai/` (sign up) → `https://console.mistral.ai/api-keys` (create key).
- **Steps:** sign up (email or Google/Microsoft/Apple SSO — SSO easiest) → **verify email** → **verify phone via SMS (required)** → API Keys → **Create new key** → **COPY IT NOW (shown once)** → paste.
- ⚠️ **VERIFY (biggest open question):** whether `mistral-ocr` runs on the **free "Experiment" tier** or **requires the paid "Scale" plan + the user's own card**. Design defensively: try the free key; if OCR returns a plan/billing error, prompt "add your own card in Mistral (charges go to Mistral, not us)." Do **not** promise "free OCR."
- **Privacy:** free tier **may train** unless the user disables **Admin Console → Privacy → "Anonymous improvement data"**; paid opted out by default; 30-day retention; ZDR available. Surface the opt-out for sensitive material. EU-hosted by default.
- **Key format:** opaque, no prefix → validate only by API call.

## 6. iPhone app + shared `ArchiveCore` (from prior research; unchanged)
Ship an **iPhone capture companion at parity with Android** (Option A) — iOS has no Finder tags, and the Mac already runs the `CaptureServer` HTTP+Bonjour receiver the companion talks to (no Mac changes). Extract a multiplatform **`ArchiveCore`** Swift package (OCR clients, `ImageEncoding`, `NetworkSession`, `KeychainHelper`, `PDFGenerator`, and the new `KeyValidator`/`ProviderKeySpec` — all Foundation/ImageIO/CoreGraphics/Security; strip cosmetic AppKit imports; guard with `#if canImport(AppKit)`), ~40–60% reusable. New iOS app: SwiftUI + AVFoundation camera, QR pairing, `PhotosPicker`, HTTP client mirroring Android's `MacClient.kt`, plus the **same guided key wizard** so iOS users onboard identically. `[CLAUDE]` builds it all; `[USER]` does signing/submission. Info.plist: `NSCameraUsageDescription`, `NSLocalNetworkUsageDescription`, `NSBonjourServices`, `ITSAppUsesNonExemptEncryption`, `PrivacyInfo.xcprivacy`.
> **[USER] DECISION D2** — iPhone scope: capture companion (recommended) vs standalone iOS processor. `UNDECIDED`.

## 7. App-store distribution — now simple (nothing to sell)
Because all apps are **free and contain no purchases**, there is **no IAP, no Play Billing, no commission, no billing setup** anywhere — the single biggest simplification vs the managed plan.
- **macOS:** Mac App Store as a **free app** (best discoverability; no IAP issues now) **or** a notarized Developer-ID **DMG** (simpler signing, no review). `[USER]` decides (**D1**); `[CLAUDE]` preps entitlements/Info either way.
- **iOS (App Store):** `[USER]` signing, submission; `[CLAUDE]` drafts App Privacy nutrition labels, usage strings, `PrivacyInfo.xcprivacy`, export-compliance. Watch **2.1/4.2 "minimal functionality"** — the listing/first-run must state the companion needs the Mac app.
- **Android (Play):** `[USER]` Play Console ($25), upload, Data Safety form; `[CLAUDE]` sets **targetSdk 36** (mandatory ~2026-08-31) and drafts Data Safety answers. New personal Play accounts face a **closed-testing gate (12 testers / 14 days)** — plan for the delay.
- **Both stores** need a hosted **privacy-policy URL** (`[CLAUDE]` drafts text; `[USER]` hosts).

## 8. Phased plan
| Phase | What | Feasibility | Status |
|---|---|---|---|
| **1. Guided key onboarding (macOS)** — `ProviderKeySpec`, `KeyValidator` (validate + sample-OCR + error maps), `ProviderKeyWizard`, first-run + Settings integration, bundled sample. **The core of this plan; needs no user accounts to build.** | 🟢 [CLAUDE] | `DONE` ✅ |
| **2. Polish & bulletproofing** — clipboard detect/trim, locale-based EEA billing pre-warn, 429 throttle/backoff during batches, privacy notices, re-runnable "Test key", `[USER]` screenshots/GIFs + verify provider wording. | 🟢 [CLAUDE] / 🟡 [USER assets] | `NOT STARTED` |
| **3. `ArchiveCore` package + iPhone companion** — extract shared package; build iOS capture app + same wizard. | 🟢 [CLAUDE] build / 🟡 [USER] signing | `NOT STARTED` |
| **4. Store distribution** — Mac (MAS/DMG), iOS App Store, Android Play; privacy labels/Data Safety (Claude drafts), targetSdk 36. | 🟡 [CLAUDE drafts] / 🔴 [USER submits] | `NOT STARTED` |
| **0. Accounts (parallel, user)** — Apple Developer ($99/yr; check @stanford.edu nonprofit/edu waiver), Play Console ($25), host privacy policy. | 🟢 [USER] | `NOT STARTED` |

## 9. What Claude Code can vs cannot do
| Can build (🟢) | Cannot do (🔴 human-only) |
|---|---|
| Entire guided wizard, per-provider spec, live key validation, error→plain-English mapping, sample-OCR test, status chips, first-run flow, Settings integration | Enroll in Apple Developer / Play Console; pay fees |
| Clipboard detect/trim, locale EEA pre-warn, 429 throttle/backoff, privacy notices, deep-link opening | Interactive Xcode signing/provisioning/notarization; store submission; console forms; TestFlight/closed testing |
| `ArchiveCore` package + iOS companion + AVFoundation camera + QR pairing | Verify current provider sign-up **UI wording**; capture **screenshots/GIFs**; host the **privacy-policy URL** |
| Privacy labels / Data Safety / export-compliance **text**; Gradle targetSdk 36; entitlements/Info.plist | Create the user-facing sample image if a real archival page is preferred (Claude can bundle a placeholder from `Test Files/`) |

## 10. Risks & ⚠️ verify-before-ship
- **Mistral OCR free-tier availability** — unclear; may need the user's own card (Scale plan). Design probes + prompts; verify live. *(highest-impact unknown)*
- **Provider UI/limits/terms drift** — free-tier limits are dynamic (Dec-2025 cut); don't hardcode; screenshots go stale — keep copy provider-agnostic and `[USER]`-refreshable.
- **Gemini "restrict key after 2026-06-19"** claim — verify; add optional nag.
- **Privacy for archival PII** — free tiers may train on data; surface clear warnings + paid/opt-out paths before OCRing sensitive records.
- **EEA/UK/CH** — Gemini requires paid there; detect and pre-warn.
- **Sample-OCR test cost** — one tiny call per validation; negligible, but note it consumes a hair of the user's quota.

## 11. Open decisions
- **D1** macOS distribution: Mac App Store (free) vs notarized DMG — `UNDECIDED` (either is fine now; no billing either way).
- **D2** iPhone scope: capture companion (recommended) vs standalone — `UNDECIDED`.
- **D3** First-run behavior: force the wizard on first launch vs. a dismissible banner — recommend dismissible with a persistent "Set up keys" call-to-action. `UNDECIDED`.
- **D4** Bundle a real archival sample image for the OCR test (user-supplied) vs a placeholder — `UNDECIDED`.

## 12. Sources (re-verify before launch — fast-moving)
- Gemini API key / free tier / billing / terms / troubleshooting: aistudio.google.com/apikey · ai.google.dev/gemini-api/docs/billing · /terms · /docs/troubleshooting · /docs/rate-limits · aistudio.google.com/rate-limit
- Mistral console / keys / pricing / errors / privacy opt-out: console.mistral.ai · console.mistral.ai/api-keys · mistral.ai/pricing · docs.mistral.ai/resources/error-glossary · docs.mistral.ai/getting-started/quickstarts/studio/activate-and-generate-api-key
- Onboarding UX references: Raycast (Settings → AI → API Keys BYOK), MacWhisper.
- Store/iOS-port details: see `MANAGED_ACCESS_PLAN.md` §5–6 (still valid; billing parts now moot).

## 13. Changelog
- 2026-07-03 — Created from a 3-track web-research sweep (Gemini key flow, Mistral key flow, guided-onboarding UX). Supersedes MANAGED_ACCESS_PLAN.md. No code written yet; Phase 1 (the wizard) is ready to build and needs no user accounts.
