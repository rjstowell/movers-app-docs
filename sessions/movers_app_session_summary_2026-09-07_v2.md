# Movers App — Session Summary 2026-09-07 v2

**F21 MARKETING SITE — DESIGN PHASE.** Design + decisions + legal research. **No code shipped, no migration.** Six marketing pages designed as standalone HTML mockups on one shared design system; site architecture locked; Privacy + Terms drafted to UK-GDPR/DUAA-2025 + AI-SaaS best practice; new items + DECs logged.

Baseline unchanged in code: main at `1d7fb29` (F30 slice 3 complete, from the prior 09-07 session).

---

## What we did

### 1. Reviewed state → picked F21 as next
Read the recent sessions, backlog, launch plan, driving vision. Confirmed we're close to MVP: **F30 billing was the biggest remaining technical gate and it closed** (09-06/09-07 build run). Picked **F21 (marketing site + privacy policy)** as the next big item — the last Tier-1 *build*, now fully unblocked (brand DEC34, domain-on-Vercel S43-A, pricing DEC43, billing F30 all done). Flagged the launch plan as ~2 sessions stale (it predated the F30 slice-3 run) — fixed this session.

### 2. Design — six pages, one system
Locked the design feel via three taste anchors (light/airy, trust blue, clean sans), then built to it. One design system across all pages:
- Light/airy near-white base, **single disciplined trust-blue `#1D5FD6`** (CTA + one accent only, not blue everywhere), navy ink `#0E1B2C`, **Geist** sans, **hairline dividers over drop-shadows**, left-aligned asymmetric hero, one orchestrated page-load reveal.
- Anti-generic moves: no cream (the AI tell), no centered-hero default, no shadowed card-kit, boldness spent only on the hero.

Pages (all in `/mnt/user-data/outputs`, `_v1`/`_v2`):
1. **Home** (`moversapp_landing_v2.html`) — hero = customer enquiry → drafted reply glimpse; how-it-works; feature bands (Quote Engine, Jobs/CRM, Calendar & fleet); "everyday essentials" trio; closing CTA. V2 calmed the hero + broadened from email-only to the whole operation after operator feedback.
2. **Solutions** (`moversapp_solutions_v1.html`) — deep per-feature detail with mini-UI visuals: auto-responder review-queue + auto-send toggle, Quote Engine breakdown + van-fit, Jobs/CRM kanban, Calendar & fleet reminders, leads inbox; closing trio (roles, dashboard, data).
3. **Pricing** (`moversapp_pricing_v1.html`) — 3 vertical tiles (£29/£44/£99), Pro "Most popular", seat bands as headline differentiator, FAQ.
4. **Talk-to-sales** (`moversapp_talk_to_sales_v1.html`) — what-to-expect + a scheduler-embed placeholder + "rather just try it → free trial" escape hatch.
5. **Privacy** (`moversapp_privacy_v1.html`) — UK-GDPR/DUAA-2025 draft.
6. **Terms** (`moversapp_terms_v1.html`) — B2B, AI-output liability shield + UCTA-safe cap.

**Scope discipline:** feature content grounded in launch-BUILT features only. Roadmap items (customer portal, SMS, call transcription, Open Banking, photo→inventory) deliberately left OFF to avoid overselling.

Operator reaction: strong ("damn good start", "one-shotted almost perfect"). Two asks handled: calm the hero slightly, and show it does more than email (→ V2). Copy left as first-draft by choice.

### 3. Site structure decisions
- **DEC47 LOCKED — separate site on the apex, app on a subdomain.** Marketing built/deployed separately (not same-repo route-group), on `moversapp.app`; app moves to `my.moversapp.app` (recommended, not locked). Decouples copy edits from app risk, matches the subdomain split, keeps marketing near-static (mockups ≈ the site). **OPEN sub-question: the platform** — static-via-Codex now (fast, no non-dev editor) vs a visual builder (Framer/Webflow — self-editing, but rebuilt in-tool, separate platform + cost, Codex can't build it). Operator unsure; leaning static-now, migrate later if self-editing becomes a priority.
- **About page DROPPED** — footer carries legitimacy (Future Apps LTD + co number + contact) instead.
- **Pricing** — 3 tiles + Pro "Most popular"; per-tier split = the stated features. **FLAG: don't advertise a per-tier "AI reply allowance" unless the S252 token cap is enforced per tier — advertise what you enforce.**
- **Talk-to-sales** — embed a third-party scheduler. **Building it on our own calendar was REJECTED** (a public booking engine ≠ the internal ops diary; buy-don't-build). **Cal.com recommended, NOT locked.**
- **Trial funnel** — Start-free-trial → Pricing → pick tile → signup carrying the tier → in-app Stripe Checkout (F30). The trial can't start without a tier (Stripe needs a price); eligibility stays server-side (DEC43).

### 4. Privacy + Terms — researched, drafted
Researched current UK requirements (not from memory): **Data (Use and Access) Act 2025** (Royal Assent 19 Jun 2025) amended UK GDPR, with a **complaint-handling duty in force from 19 Jun 2026**. Key structural points baked in:
- **Dual controller/processor hats:** we're the controller for tenants' own account data; the **processor** for the tenants' end-customers' enquiry data (the firm is controller). Backbone of the privacy policy — generic templates miss this.
- **The LLM is a sub-processor AND a US transfer** — enquiry text → OpenAI needs disclosure + SCCs/UK IDTA; sits alongside Supabase, Resend, Stripe, Google.
- **Terms AI shield:** output is probabilistic/may be wrong; **customer must review before sending**; quotes are estimates, not binding offers; auto-send at own risk; liability capped ~12 months' fees; indirect loss excluded — **but** non-excludable carve-outs (death/PI/negligence, fraud) kept in, or UCTA voids the whole clause.

**These are DRAFTS, not legal advice** — on-page amber "draft" bar + blue `[bracketed]` fields. Must be solicitor-reviewed + fields filled before publish (→ S260). Also surfaced: we reference a **tenant DPA we don't have yet** (→ S261).

### 5. Remaining-to-live mapped
Grouped the mockup→live gap: **build** (assemble the 6 into one project, wire real links, working mobile nav, favicon/meta/OG/404) — Codex can do it; **decisions/content** (per-tier split, scheduler provider, footer legitimacy, copy); **legal gate** (S260 fill/review); **infra** (the subdomain cutover — the long pole); **funnel wiring** (Pricing-tile→signup tier handoff). Minimum critical path: cutover → assemble/wire → legal → per-tier → scheduler → funnel.

---

## Decisions

- **DEC47 — marketing site architecture: separate site on the apex, app on a subdomain. LOCKED.** Platform (static-via-Codex vs Framer) is an OPEN sub-question.
- **DEC48 — operator cross-tenant console launch-slice vs post-launch. CANDIDATE (not locked).**
- **DEC49 — adopt the F21 design system as the app's shared design language. CANDIDATE (not locked).**

## New items

- **S260** — Privacy/Terms solicitor review + fill [bracketed] fields. Gates functional launch + Google OAuth.
- **S261** — tenant-facing DPA (UK GDPR Art 28). Required before real stranger tenants.
- **S262** — marketing-site copy refinement pass. Post-launch polish.
- **S263** — subdomain cutover (app → my.moversapp.app). FIRST build session; gates F21 build.
- **F31** — operator cross-tenant console (superadmin insights + actions across tenants). Absorbs F2. Timing = DEC48.

## Naming untangle (logged so it stops recurring)
Three things wore the "control centre / admin" name: the Driving Vision's "control centre" = the per-tenant Advanced/agents tab; **F2** = master-prompt editing only (folds into F31); **F31** = the cross-tenant operator console (the actual gap). F2 is a tab of F31, not a standalone.

## Doc updates this session
- **Master backlog:** new top header + numbering block (next free S264 / F32 / DEC50); F21 entry updated (design phase complete); new 2026-09-07 v2 session section (DEC47 + S260–S263 + F31 + DEC48/49 candidates + F2→F31 amendment).
- **Launch plan:** new baseline (folds in the 09-06/07 F30 build run the old baseline predated); F30 → DONE (slice 3 complete); S252 → DONE; S250 → CLOSED; S256 → DONE; S254 → PART-DONE (Half A); S253 reframed/parked note; S217 session-row → DROPPED; F21 row → DESIGN-DONE; new Tier-1 rows S263/S260/S261; F31 added to post-launch; session-order + estimate refreshed (~4–6 sessions to launch-ready, F30 done).

## Next
**S263 subdomain cutover** — one focused, config-heavy session (Supabase Site URL + redirect allow-list, Google OAuth redirect URIs, Stripe redirect URLs, cookie domain, DNS). Stand the app on the subdomain + verify login/checkout/email-links end-to-end WHILE the apex still works, then repoint the apex at marketing last. **Settle the deploy-target (Vercel vs VPS) + site-platform (static vs Framer) decisions before the brief.** Then F21 assemble/wire/deploy.

## Operator to-do
1. **Download the 6 design mockups + this summary + the swapped backlog & launch plan from outputs; upload into the Project** (design files are the F21 build reference — without them the next session starts blind). Optional: rename design files to clean names on upload.
2. Settle deploy-target + site-platform decisions.
3. Parallel: chase S243 (Postmark Platform + approval).
