# Movers App — Session Summary 2026-09-10

**SIGNUP BILLING HOLE CLOSED + AN UNAUTHENTICATED OPENAI PROXY SHUT. Three ships, all live + verified on prod (`my.moversapp.app`).** F36 (signup billing gate, slices 1+2), S269 (`/api/openai` lockdown), S270 (checkout-step styling). main: `65f3c19` (F36 s1) → `120d11c` (S269) → `271c3d6` (F36 s2) → `6e940d1` (S270). No migration this session. First real exercise of the read-only blocked path (a fresh post-cutoff signup is the first genuinely blockable tenant).

**NUMBERING MISTAKE, logged so it doesn't confuse git-vs-backlog later.** The commits shipped under the WRONG codes because the operator's briefs carried guessed numbers: "F32" (guessed from chat as F31+1) and `S###` left for Codex to fill (Codex can't read the Claude-side backlog, so it grabbed S260/S261 from its own repo view). All three collide with real, already-assigned items. Canonical numbers assigned here; commit labels kept as historical aliases (merged history not rewritten).

| Canonical | Shipped-under label (alias) | The label's REAL owner (untouched) |
| --- | --- | --- |
| **F36** | "F32" | F32 = DIY booking engine (post-launch, rejected) |
| **S269** | "S260" | S260 = Privacy/Terms solicitor review |
| **S270** | "S261" | S261 = tenant-facing DPA |

Process fix going forward: the number comes from the active backlog into the brief title; never leave `S###`, never infer an F-number from chat.

---

## 1. The problem found (start of session)

Two holes in the signup path, plus a worse third found during discovery:

1. **No payment gate on signup.** A visitor could reach `/login` → "Sign Up" (or the marketing "Start free trial") and land in the app with no plan and no card. The intended funnel (DEC47: signup-with-tier → in-app Checkout) was designed but the funnel WIRING was never built — DEC43's notes had left "where checkout fires (signup vs post-onboarding)" as an open placement decision. Checkout only ever fired from the interim Settings→Billing selector, manually.
2. **The read-only enforcement (F30 slice 3, already live) did NOT catch a never-paid tenant.** DEC45's grace model treats **no billing row → not locked** (a carve-out for Cornwall / pre-billing test tenants). A fresh signup has no row, so it read as fully unlocked, LLM value included, indefinitely.
3. **`/api/openai` was an open, unauthenticated general-purpose LLM proxy on the OpenAI key** (found during the F36 discovery pass). No session, no secret, no rate limit, no token accounting; caller controls the system prompt and unbounded input; spend hits `api.openai.com` directly, invisible to the per-tenant counter. Anyone with the path could drain the key, no signup needed. Live on prod, protected only by nobody knowing the path.

**Decision: option B (server-side gate) over option A (fix the door).** A door-fix leaves the no-row-is-unlocked grandfather open to any future ungated tenant-creation path; the server predicate sits behind every door and doesn't care whether the funnel wiring is correct. Confirmed harder when the marketing "Start free trial" turned out to also skip Stripe (the in-app checkout step simply didn't exist yet).

---

## 2. Discovery pass (read-only Codex)

Report: `signup-billing-gate-discovery-2026-09-09.md`. Headline findings that shaped the build:

- The gate does not exist on the page path: `middleware.ts` checks auth → membership → profile → onboarding, stops at line 230. Seam for a billing gate = 230→232.
- A tenant's first `tenant_billing` row is created only inside `createCheckoutSession` (`actions.ts:72`), not at signup / `create_tenant_with_owner` / `seed_tenant_defaults`. Live at the time: 19 tenants, 1 with a billing row.
- The DEC45 grandfather branch is one line: `lib/tenant/active.ts:195` `if (error || !data) return false;` — collapses "never subscribed" and "query failed" into the same answer, load-bearing for the seat cap and send pipeline. New gate needs its OWN predicate; do not touch line 195.
- **The leak is off the page path, where no redirect reaches:** the cron processes unpaid tenants' inbound mail and auto-sends LLM replies (`pipeline.ts:109`, `auto-send/route.ts:81`, both keyed off `isReadOnly` which returns false for no-row); seats uncapped (`settings/users/actions.ts:99` derives cap from `stripe_price_id`, no row → no tier → no limit); nine `/api` routes billing-blind, seven covered by one insert in `app/api/cards/_context.ts`.
- No tier hand-off exists (zero hits for a tier/plan query param or cookie in `app/` or `lib/`); marketing is a separate repo and a query param wouldn't survive the Supabase email redirect.
- Grandfather recommendation: a `created_at` cutoff (column exists, no migration, correct for all future tenants; an allowlist can't be authored from a dev DB with no Cornwall in it).

---

## 3. F36 slice 1 — billing entitlement leak-plug (PR #103, `65f3c19`)

Server-side only, no UX, no migration.

- **Predicate:** `isBillingBlocked(client, tenantId)` in `lib/tenant/active.ts`, beside `isReadOnly` (line 195 untouched, composes on top). Returns true when `isReadOnly` is true OR (no billing row AND `created_at >= BILLING_CUTOFF`). So: false for active/trialing/in-grace, false for grandfathered no-row (`created_at < cutoff`), true for canceled/unpaid/past-due-expired, true for a new no-row tenant. Any query error / unreadable `tenants` row fails **open** (documented).
- **`BILLING_CUTOFF`** = `"2026-09-10T00:00:00.000Z"` at `lib/tenant/active.ts:263`, carries `TODO(deploy)`. It was the first midnight UTC after the newest tenant at authoring time, so all 19 are grandfathered. **Went stale overnight** — it is now in the PAST, so on any deploy it enforces immediately against anything created since 00:00 UTC today. Harmless now (verified zero tenants stranded, below), but must be reset to the real deploy moment before public signups open.
- **Chokepoints:** pipeline (capture-without-cognition), auto-send (no send), seat invites (blocked before the tier cap, which is otherwise unchanged), and both API context helpers (`cards/_context.ts` + `quotes/_context.ts`) returning a shared 402 `billing_required`.
- **Deviations (accepted, all improvements):** two context files not one (discovery under-counted); the 402 authored once in the cards context and re-exported; `quoteContext` also covers three Google-spend routes (`/api/places/autocomplete`, `/api/places/details`, `/api/routes/distance`) — same leak class, free from the shared insertion. `/api/tenant/switch` deliberately NOT gated (grants no paid resource, only sets the active-tenant cookie; gating it would deadlock a blocked owner out of the very workspace they must reach checkout from). `/api/openai` correctly out of scope for this slice (no tenant to run the predicate against) — became S269.
- Verified `tsc`/lint/build clean, 198/198 tests (+11 for the predicate, cutoff fixtures derive from `BILLING_CUTOFF` so retuning won't break them).

---

## 4. S269 — `/api/openai` lockdown (PR #104, `120d11c`)

The open proxy. Confirmed reachable in production, single legitimate caller (the quote page's `callOpenAI` used by ChatCard/ItemsCard/SanityCheck, already behind the auth wall so the session cookie is present but the route never read it). Only guard was an `OPENAI_API_KEY` presence check; caller controlled the system prompt, input unbounded (~1M tokens/request possible), no rate limit, spend invisible to `tenant_llm_daily_usage`.

- Shipped: **401** (no user) → **403** (no membership) → **402** (billing-blocked, reusing `billingBlockedResponse`) → **429** (rate limit, `checkRateLimit` keyed `openai:${tenantId}` at 20/min, tighter than Places lookups because an LLM call costs more) → **500** (no key, moved BELOW auth so anonymous callers learn nothing about config) → **413** (`MAX_BILLED_CHARS`, 32KB, sized from the measured worst case: ItemsCard's catalogue block at ~165 items / ~6KB, so 3-4x headroom, turns a ~1M-token request into ~8k). `body.system` passthrough left as-is (constraining the proxy shape is a separate P2, out of scope).
- **Token accounting DEFERRED (operator call).** Investigation found `tenant_llm_daily_usage` is PK `(tenant_id, usage_date)` with NO source/bucket column, so any write from this route consumes the same daily budget the agents pipeline enforces against — a genuinely separate bucket needs a migration, out of scope. Deferring means quote-page spend stays invisible to `TENANT_DAILY_TOKEN_CAP` as today; the bound is now the rate limit + the input ceiling, not a daily cap. Recorded as a `DEFERRED` comment in-route. → **S272** (the migration: own bucket + source column).
- Rebased onto main after #103 squash-merged (dropped #103's pre-squash commits; diff verified as ONLY the two `/api/openai` files before merge). Both branches deleted after merge.
- **Cosmetic flaw not fixed (flagged):** `billingBlockedResponse()` returns `error` as a string, but the quote page's `extractJSON` reads `data.error.message`, so a blocked tenant on the quote page would see "API error" not the subscription text. Kept the shared 402 contract intact (slice 2 keys off `code: "billing_required"`). Likely moot once slice 2 gates blocked tenants off the quote page entirely; revisit if not. (Logged as a note, not yet numbered.)
- Verified 204/204 tests (+6), refusal cases assert `fetch` was never called (proves no spend, not just a non-200). **Prod-verified: anonymous `curl` → HTTP 401**, `X-Matched-Path: /api/openai` (the route answering, not Vercel).

---

## 5. F36 slice 2 — force signup → checkout before onboarding (PR #105, `271c3d6`)

Target sequence: signup → verify email → sign in → onboarding name/company (tenant created) → **choose plan + Stripe Checkout** → rest of onboarding (provisioning) → app. Plus a one-time "Email verified" banner.

Confirmations that shaped it:
- Flow is a hybrid: tenant creation is its own route (`/onboarding`) which redirects to `/agents/setup/onboarding` rendering `AgentOnboardingFlow.tsx` (one client component, internal `StepKey` state). Route-per-step at the seam this slice needs, internal state after — so the checkout step went in as its own route (no surgery on the 775-line component) and BOTH enforcement halves were built.
- Tenant creation = `app/onboarding/actions.ts:22` `create_tenant_with_owner`; the redirect on line 28 is the insertion point.
- Provisioning = `completeAgentOnboarding` (`agents/actions.ts:721`, one RPC that provisions AND sets status completed). **`skipAgentOnboarding` (:708) is a SECOND completion path** that sets "skipped" and satisfies the middleware check identically — gating only completion would have left skip an open door. Both gated.

Two findings that changed the plan:
- **`createCheckoutSession` already seeds a `tenant_billing` row (`actions.ts:72-79`) with just `stripe_customer_id`, BEFORE Stripe.** Slice 1's predicate treated "has a row" as "let `isReadOnly` decide", and `isReadOnly` returns false for a null status — so merely OPENING checkout (not completing it) would have unlocked the product permanently. Fixed: post-cutoff tenants now require an ENTITLING status, not merely a row. `isReadOnly` / line 195 / `BILLING_CUTOFF` all untouched; pre-cutoff branch unchanged.
- **The confirmation email never reaches `/login`** — it goes to `/auth/callback`, which auto-signs the user in and drops them at `/onboarding`; a banner there would never render, and its params (`code`, `token_hash`, `type`) are shared with OAuth/reset (the exact confusion warned about). Fix: signup now sets `emailRedirectTo: ${origin}/login?verified=1`, producing the target sequence literally. **Deploy step: that URL had to be in the Supabase Redirect URLs allow-list** — DONE this session (added; `https://my.moversapp.app/**` already covered it anyway).

Built: checkout step reusing the interim selector's mechanism (`/onboarding/checkout` route); middleware gate at 231; both completion paths refuse while `isBillingBlocked`; success page now server-side `stripe.checkout.sessions.retrieve(session_id)` and upserts `tenant_billing` (admin client, status trialing) to kill the return-from-checkout race (webhook stays authoritative reconciler); email-verified banner on `/login` keyed off `verified=1`. One self-caught bug: the Stripe return paths sit under `/settings`, an onboarding-entry path, so a returning tenant was briefly bounced to onboarding before the success route could reconcile — both return paths exempted (three gates now run in sequence, exemption sets must agree).

Verified 207/207 tests (+3 for the abandoned-checkout hole). **Prod-verified end-to-end on a fresh signup:** email → "Email verified" banner on `/login` → name/company → checkout step (nothing else on the page, so no skip/jump possible) → test card 4242 → real onboarding → app. Clean flow. This is the **first real exercise of the blocked path** (a fresh signup is post-cutoff).

---

## 6. S270 — onboarding checkout styling (direct to main, `6e940d1`)

Cosmetic, single component, presentation only. The checkout step reused the interim selector and rendered as a bare page; restyled to the app's own design system (`.agent-onboarding-shell` / `-content` / `-step` / `-kicker` / `-copy`; cards `.surface-card`; buttons `.btn` / `.btn-primary` / `.btn-secondary`), NO marketing CSS lifted (separate repo, static, different stack). Price + seats only, **no per-tier feature bullets** (that copy is unverified F21/S266 territory; kept out of the paid flow). Pro highlighted with the existing "Most popular" pill + brand border + primary button. Logic (tier→price, `createCheckoutSession`, Stripe redirect, gate) byte-identical. Verified in-browser via a throwaway route, 207/207 tests. Prod-confirmed visually: "Start your trial" kicker, three styled cards £29/£44/£99, Pro highlighted, no feature bullets.

---

## 7. DEC45 amendment (new-tenant billing predicate)

DEC45's "**no billing row → not locked**" grandfather is retained for existing tenants but no longer applies to NEW ones. The new-tenant rule lives in a SEPARATE predicate (`isBillingBlocked`), NOT in `isReadOnly` / line 195: a tenant with `created_at >= BILLING_CUTOFF` and no ENTITLING status (active/trialing, not merely a row) is blocked from paid resources and from completing onboarding. Grandfather is drawn by a `created_at` cutoff (column exists, no migration, correct for all future tenants; allowlist rejected — unauthorable from a dev DB without Cornwall). `billing_exempt` stays a purely additive later option (`created_at < CUTOFF || billing_exempt`) if a comp account is ever needed. Logged as an amendment to DEC45; no new DEC number consumed.

---

## 8. Verification + open tails

- **`BILLING_CUTOFF` past-dated but harmless.** MCP query on prod: zero tenants with `created_at >= 2026-09-10T00:00:00Z`, so the past cutoff strands nobody. It only bites when a real signup lands before it's reset. **Reset to a real forward date before public signups open** (deploy task, `TODO(deploy)` intact).
- **`TENANT_DAILY_TOKEN_CAP` unset in prod.** Codex flagged the cap "fails open when the env var is unset." Confirmed unset → **S252's per-tenant daily cap is currently INERT in prod** (agents pipeline included), so there is no daily token ceiling anywhere right now. Fold into the still-pending cap-setting session. This makes **S253** (enforced hard cap on the LTD OpenAI org, parked on the Monzo debit card) the only global spend backstop until the cap session lands — pull forward if possible.
- Blocked-path scenarios beyond the fresh-signup run still await ordinary launch traffic; the fresh signup covered the core (capture-without-cognition + skip-block + checkout redirect).

---

## New / updated backlog items

- **F36 DONE** (signup billing gate, slices 1+2; PRs #103/#105; `65f3c19` + `271c3d6`; no migration). Alias "F32" in commits.
- **S269 DONE** (`/api/openai` lockdown; PR #104; `120d11c`; no migration). Alias "S260". Token accounting deferred → S272.
- **S270 DONE** (onboarding checkout styling; direct `6e940d1`). Alias "S261".
- **S271 NEW OPEN** — seat-count DISPLAY to tenants (X of Y seats used) in Settings→Users / the invite form. Enforcement works (slice 1); the number is invisible, so a tenant hits the cap blind. Cosmetic-ish, orthogonal, own item.
- **S272 NEW OPEN** — `/api/openai` token-accounting migration: add a source/bucket column to `tenant_llm_daily_usage` so quote-page LLM spend meters + caps in its OWN bucket (avoids coupling quote spend to the agents-pipeline daily budget, which could starve inbound auto-replies on a heavy quoting day). Deferred from S269.
- **DEC45 AMENDED** — new-tenant predicate (see §7). No new DEC number.
- **F21 note** — the marketing→app tier hand-off and the full styled checkout screen with real per-tier feature copy are both already inside F21's scope (F21 lists "Pricing-tile→signup tier handoff" + "per-tier feature decision"). This session gave them concrete shape: the in-app checkout step now EXISTS (F36), so F21's hand-off is "skip the in-app plan screen when the tier is already known", not "build checkout"; and the checkout screen currently ships price/seats-only styling (S270) pending F21's feature-copy call. The in-app plan screen can't be deleted even once the hand-off lands (the login→signup door has no tier).
- **Quirk candidate** — `TENANT_DAILY_TOKEN_CAP` (and the per-tenant cap generally) fails OPEN silently when the env var is unset; no daily ceiling is enforced in that state, and nothing signals it.

## Method

Every Codex/Claude-Code claim DB/prod-verified: the anonymous-curl 401, the fresh-signup end-to-end, the `BILLING_CUTOFF` zero-stranded query (MCP). Codex caught four things that would have shipped wrong: the open-checkout-unlocks-permanently near-miss, the skip-completion second door, the email banner never rendering on `/login`, and a self-introduced checkout-return loop. The numbering collision was the operator/brief side's mistake, corrected here.
