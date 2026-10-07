# Movers App — Session Summary 2026-09-08

**S263 subdomain cutover DONE · F21 marketing site slices 1–2 BUILT + LIVE · Cal.com booking LOCKED (free) · Postmark approval cleared · doc pass.**

App now lives on `my.moversapp.app`. The marketing site is standing on its **own** Vercel project at `movers-app-marketing-site.vercel.app` — the apex is **not yet repointed** (that's the go-live tail). No app-repo code change and no migration this session: the cutover was dashboard/DNS config, and the marketing site is a separate repo.

Key handles set this session: marketing repo `rjstowell/MoversApp-Marketing-Site` (main `487ef14`, PR #1); Cal.com account `movers-app`, event slug `moversapp-sales-call` (30-min, Europe/London); brand blue `#1D5FD6`; public contact `support@moversapp.app`; Future Apps LTD No. 17438533, Unit 82a James Carter Road, Mildenhall, Bury St Edmunds, IP28 7DE.

---

## 1. S263 — subdomain cutover (DONE)

**Result:** app moved off the apex to `my.moversapp.app`, **config-only, zero code changes.**

**Discovery (read-only Codex pass) found the cutover needs no code:**
- Every absolute URL is **request-origin-derived** — login, Google-OAuth callback, password-reset, user-invite, and Stripe checkout/portal all build from the request host — so they self-rebase to the new subdomain automatically.
- Auth cookies are **host-only** (no `.moversapp.app` domain lock), so nothing to change; the only consequence is a fresh login on the new host.
- **The one exception:** signup-confirmation emails carry no app-set redirect, so they fall back to the Supabase **Site URL** — which is why that flips **last**.

**Mental-model correction (from the quirks log):** the app was **not** on the bare apex. Apex `308`→`www`, and `www.moversapp.app` was the canonical host — which is where the Stripe webhook was registered (a 308 counts as a **failed** Stripe delivery, so the webhook must sit on the canonical host).

**Execution order:** add `my` CNAME at Porkbun → allow-list **both** domains in Supabase → repoint the (test-mode) Stripe webhook to `my.moversapp.app/api/stripe/webhook` → verify on `my` while `www` still served → flip Supabase **Site URL** to `my` **last**.

**Verified on `my`:**
- Email + password login.
- Portal plan-switch (Starter→Pro on the S199 tenant) → the repointed webhook wrote `tenant_billing` — **DB-confirmed**: `updated_at` = 2026-09-08 08:49, status `active`. This proves the webhook works on the new host.
- Signup-confirmation email now lands on `my.moversapp.app/login?code=…`.

**Not verified — by choice / out of scope:**
- Password-reset + user-invite links — left untested deliberately (identical request-origin mechanism to the login that was verified).
- Google login — the flow exists in code (`app/login/page.tsx`) but the Supabase Google provider may not be configured. It's the Google-OAuth track (gated by the privacy policy / S260); the Supabase-managed callback is domain-independent, so the cutover needed nothing extra. Confirm provider setup separately.

---

## 2. F21 — marketing site slices 1–2 (DONE + LIVE)

Separate repo + separate Vercel project (per DEC47). Static, **no build framework**, Vercel preset "Other". Built and standing on `movers-app-marketing-site.vercel.app` — **not public** (apex repoint waits on slice 3 + S260).

**Setup:** created an empty private GitHub repo + a new Vercel project pointed at it. Order that matters: Codex populates the repo first, **then** Vercel imports (an empty repo can't deploy).

**Slice 1 (assemble):** the 6 mockups → clean routes (`index` / `solutions` / `pricing` / `talk-to-sales` / `privacy` / `terms`) + unified nav/footer + real inter-page links + a working **mobile hamburger** (the mockups only hid nav links <900px) + meta/favicon/404. "About" **dropped** (footer legitimacy instead). Initial populate went **direct to main** (greenfield — nothing to protect yet).

**Clean URLs:** added `vercel.json` (`cleanUrls: true`, `trailingSlash: false`) + extensionless internal links — `/pricing`, not `/pricing.html`.

**Slice 2 (PR #1, squash `487ef14`):**
- Pricing hero → **"A plan for every team"** (was the clunky "Simple pricing, priced by your team").
- Dropped "reply" from the two AI-allowance tier lines. **The lines were KEPT** (operator call — Pro/Scale tiles are thin otherwise, since the differentiator is seats-only). This copy is **not yet true** and must be backed → **S266**.
- **Cal.com inline embed** on Talk-to-sales, replacing the fake static calendar. Added `min-height:640px` to the embed div (Cal inline embeds collapse without a sized parent). Copy aligned to **30 min** (mockup said 20).
- Footer: `support@moversapp.app` + the Future Apps LTD legitimacy line.
- Favicon (SVG + 32px PNG + apple-touch) + OG image, all generated from the brand mark (`#1D5FD6`), wired on every page; `og:image` set to the destined apex URL.

**Follow-up fix (same branch):** the Talk-to-sales `.ts` section had no `max-width`, so it went full-bleed once the real embed stretched it. Constrained to `var(--maxw)` to match the rest of the site.

**Legitimacy note:** UK trading-disclosure law requires the registered company name + number + place of registration + registered office to appear on the site (footer or a legal page, easily found). The Ltd can't be hidden behind the brand — but a small footer line satisfies it while the brand still leads. Decision: keep the footer line rather than rely on Terms alone.

---

## 3. Cal.com booking (LOCKED = free tier)

Building our own booking engine **and** self-hosting were both **rejected**. A public booking engine (slot-locking, timezones, confirmations, calendar invites, reschedule/cancel, two-way calendar sync) is a different beast from the internal ops diary; free Cal.com solves it at £0. Cal.com seats scale with **who takes bookings**, not app users — so a single operator stays free indefinitely; you'd only pay when multiple people share sales-call availability (a later, revenue-backed problem). DIY/self-host parked as **F32** (post-launch, only if cost becomes non-noise).

**Setup:** account `movers-app` (business username; `moversapp` was taken), profile name "MoversApp", brand avatar from the mark. One 30-min event `moversapp-sales-call`, Europe/London. Booking calendar = a dedicated **"MoversApp Sales" Google calendar** (see gotchas — a personal-calendar block was hiding every weekday but Tuesday). Embed = HTML (iframe) inline, brand `#1D5FD6`.

---

## 4. Postmark / S243 (approval CLEARED)

Account approval came through this session (was a ~1–2 day review). Infra email `admin@moversapp.app` + both tokens live in Vercel prod+preview; sender verified. **Owed:** switch to the **Platform** plan at real-tenant time (Free/Basic cap sending domains).

---

## 5. Decisions settled (were gating the briefs)

- **DEC50 — deploy target = Vercel** (app + marketing). VPS rejected: managed SSL/CI/cron, zero migration, and the HTTP-generic workers keep the VPS door open later; static marketing is edge-cached and effectively free within the $20 credit. Vercel spend management: keep "Pause Projects" **off** for a live app (auto-pause = downtime); rely on the $200 on-demand cap + notifications.
- **Site platform = static-via-Codex** (Framer rejected — the mockups *are* the site; no self-edit need while launch copy stays placeholder). Resolves the DEC47 sub-question.
- **Cal.com free** locked as the Talk-to-sales scheduler (see §3).

---

## 6. New backlog items (logged this session)

- **S264** — Stripe **test→live** promotion. Prod billing currently runs on Stripe **test** keys/products (the sandbox £29/44/99 + `cus_VCM4…` were test-mode; "proven live" was proven in test, which is correct pre-launch). Go-live = live secret key + live products/prices + a **live** webhook at `my.moversapp.app/api/stripe/webhook`. S250 only did sandbox → this was untracked. **Tier 1** (rides the apex go-live).
- **S265** — custom auth-email SMTP. Supabase's default auth sender isn't aligned to moversapp.app, so confirmation/reset/invite emails hit **junk** (observed live). Point Supabase Auth → custom SMTP at Resend/Postmark on the branded domain. **Tier 2**, before real signups.
- **S266** — per-tier LLM token cap. The pricing page advertises "Higher/Highest AI allowance" per tier, but S252's guard is **one global daily ceiling identical across tiers** — so the copy isn't yet true. Make the `guardLLM` cap tier-dependent (the wrapper already reads the tenant → add a tier→cap lookup), Starter<Pro<Scale, all set **well above** honest heavy use (stays an abuse breaker, never throttles a real customer). Cross-repo (app-side). **Tier 2**, before apex go-live / real signups.
- **F32** — self-hosted / DIY booking engine. Rejected for launch; post-launch, revisit only if Cal.com cost becomes non-noise (then self-host Cal.com rather than build from scratch).
- **DEC50** — deploy target = Vercel (see §5).

**Numbering after this session: next free S267 / F33 / DEC51.**

---

## 7. Gotchas & learnings (candidate quirks-log entries)

- **Cal.com availability checkers disagree.** The marketing "username is still available" 404 page is an unreliable upsell widget; the **signup form** is authoritative (checks the full reservation table). Trust the form.
- **Cal.com hides all days if the conflict-check calendar is busy.** An all-day or recurring "busy" event on the connected Google calendar removes whole days from availability (symptom: only Tuesdays offered). Fix: point conflict-check at a **dedicated empty calendar**.
- **Cal inline embed height.** The embed `div` is `height:100%`, which collapses without a sized parent — set an explicit `min-height`.
- **App was on `www`, not the apex** (apex 308→www). Stripe webhooks must sit on the **canonical** host — a 308 redirect is a failed delivery to Stripe.
- **McAfee WebAdvisor** flags the `*.vercel.app` preview + the cross-origin Cal embed as "risky/blocked content" — benign, client-side (the extension), won't fire the same way on the reputable apex.
- **Codex asset dependency.** Codex can't commit/open a PR while referenced asset files (favicon/OG) are missing from the repo — the operator must place provided binaries in the repo root first.

---

## 8. State at session end / what's next

**Done:** S263 (app on `my.moversapp.app`), F21 slices 1–2 (live on its own Vercel URL), Cal.com free set up + embedded, Postmark approval, doc pass (backlog + launch plan swap-ins).

**Parked / next:**
- **F21 slice 3** — Pricing-tile → signup **tier handoff** (cross-repo; the app's signup must accept a tier param and carry it into F30 Checkout, surviving the email-confirmation round-trip). ~1 session. **Deferrable — not a concierge-launch blocker** (an operator can set a new tenant's tier during hand-held onboarding).
- **Apex go-live** (the tail): repoint apex + `www` → the marketing project, create the **live** Stripe webhook (S264). Waits on slice 3 + S260.

**Real Tier-1 critical path (higher priority than slice 3):** **S260** legal solicitor-fill (external/parallel) → **S191**→**S194** (persist pipeline errors → failed-runs get a visible home) → **F11(d)+F12(b)** (References-chain + real-email inbound) → **S244** (Postmark suppression). Estimate ~3–5 sessions to launch-ready on hard gates.

**Operator-parallel (no session cost):** S243 Platform upgrade, S265 auth SMTP, S266 per-tier cap.
