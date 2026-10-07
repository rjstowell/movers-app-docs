# Movers App — Session Summary 2026-09-04 v2

**main 51837fa → ce57cb9 (F30 slice 1, PR #88 squash-merged). DEC43 LOCKED. One migration (f30_tenant_billing) applied live via MCP.** Session was two halves: (1) decide billing collection end-to-end, (2) build + verify F30 slice 1 (Stripe Checkout foundation) with Codex. Code reads/writes via file-paste (GitHub connector still unavailable in-chat). Live DB + migration via Supabase MCP.

**Session goal:** close the last open operator decision (Cand. D — billing) and, once it turned into real build scope, ship the first slice of that build verified end-to-end.

---

## Headline outcomes

1. **DEC43 LOCKED — Stripe subscriptions at launch.** Card-upfront + 30-day free trial → auto-convert; read-only lock on non-payment. Reverses the Cand. D "manual-invoice pilot = zero code" assumption. Tiers £29/£44/£99, seat-split ≤4/5–10/≤20, target avg ~£40/tenant, seats-only differentiator for launch.
2. **F30 slice 1 DONE (ce57cb9, PR #88)** — `tenant_billing` table + Stripe hosted Checkout, verified LIVE in sandbox.
3. **S250 (operator billing prereq) near-complete** — Future Apps LTD + Monzo Business + Stripe live account + sandbox products + env vars. Only payout-verification-cleared tail remains.
4. **Three follow-on items:** S251 (Case B trial dedup, deferred), trial-eligibility Case A (folds into slice 2), tier feature-copy (→ F21).
5. **Docs updated:** master backlog + launch plan swapped in; this summary written.

---

## 1. Billing decision (Cand. D → DEC43)

Split the question into **pricing** (what/how much) vs **collection mechanism** (how money moves — the actual Cand. D). Operator rejected the manual-invoice pilot outright: **time + zero recurring admin > signup volume**. Chose automated Stripe from the start, accepting the +1–2 session build.

**Trial:** 30-day free, card upfront, auto-convert. Operator's logic: longer trial → tenant beds in → less churn if the product works. Card-upfront chosen over no-card after walking the trade:
- Launch is concierge (first ~20 hand-walked) → the "no-card = more signups" benefit only pays at anonymous self-serve, which launch isn't.
- Every trial user consumes solo-operator support time → card upfront filters tyre-kickers.
- Cheaper build (one flow, Stripe auto-converts) vs no-card's extra lock/chase/prompt path.
- Reversible with data once live testing proves conversion → **no-card trial revisited post-live-testing at self-serve stage (the trigger).**

**Pricing:** three tiers created in Stripe — Starter £29 (≤4 seats) / Pro £44 (5–10) / Scale £99 (≤20). Seat-split only for launch; AI caps / own-domain / SMS differentiation deferred to F21. Note the middle tier moved 29/44/99 during setup (was briefly a 2-tier 29/99 in the DEC-draft); £44 middle anchors the ~£40 target.

**Fees checked (UK, not the US screenshot):** Stripe UK = **1.5% + £0.20** domestic card (not 2.9% — UK interchange is regulation-capped), + optional Stripe Billing 0.7%. ~£0.84 on a £29 tenant with Billing — on the £0.85 the cost model already assumed. Billing's 0.7% *is* the dunning/retry/read-only machinery, so keeping it is consistent with "want it handled" (rec: keep for launch; self-manage only if fees bite at scale). Alternatives weighed + rejected for launch: Stripe MoR "Managed Payments" (3.5% add-on — declined), BACS DD (cheaper but wrong flow for card-upfront trial — good later option), Paddle/Lemon Squeezy MoR (VAT-handling premium not worth it UK-only), PayPal (worse).

**VAT flag (operator/accountant):** Stripe UK fees may carry 20% VAT; Future Apps LTD is new + likely under the £90k threshold so can't reclaim yet — factor whether ~£0.84 → ~£1.00. Not a launch blocker; not my lane.

---

## 2. Operator billing setup (S250) — done live during the session

- **Legal entity:** registered **Future Apps LTD** (fresh LTD, £100, own co number) — NOT the dormant Legacy Growth Investments LTD (that's the planned property vehicle; reusing it would burn it + name-mismatch). Stripe binds to the entity, so entity had to be decided *before* Stripe application (switching later ≈ new account).
- **Bank:** **Monzo Business** (full UK sort code/acc no, Stripe-accepted).
- **Stripe:** account created → chose "Pick what you need" (NOT the 3.5% MoR "Let us handle it") → Subscriptions + Accept-online-payments → hosted Checkout → **sandbox** for the build → three GBP monthly products created → **live account activated** (no banners = charges enabled; payout-verification may still settle). Public payments profile skipped (optional, had a typo + exposes address — do deliberately later if ever).
- **Env vars:** STRIPE_SECRET_KEY + 3 price ids set Prod + Preview.
- **Owed tail:** confirm Stripe live payout-verification fully cleared. Did NOT copy sandbox products to live (correct — copy at go-live; live gets different ids; one-shot per product).

---

## 3. F30 slice 1 — build (Codex) + review

**Discovery first (read-only Codex pass)** mapped the ground since the app code isn't in the project files. Key findings that shaped the build:
- Tenant identity across `tenants` / `tenant_users` / `tenant_config` / `tenant_agents`; resolution via `middleware.ts` + `lib/tenant/active.ts` (`getActiveMembership`).
- Existing Billing UI is a stub (toggles `tenants.addons.multi_company` only); no Stripe code, `stripe` not a dependency.
- Server-action house pattern = `jobs/actions.ts` (`"use server"`, auth + membership + role gate + mutate + revalidate).
- Resend inbound webhook (`app/api/agents/inbound/route.ts`) = the template for the Stripe webhook (raw body via `request.text()`, signature verify).
- Secrets read directly from `process.env` (no wrapper).
- **Read-only enforcement has NO single choke point** — middleware skips `/api`, and some actions write via `createAdminClient()` bypassing RLS. → slice 3 needs middleware UI lockdown + a mutation gate in shared context helpers. This is the hard part; deliberately deferred.
- Migration reality = hand-written SQL applied manually (S130 drift; only the live DB is ground truth).

**Built (branch `codex/f30-slice1-stripe-checkout`):**
- `tenant_billing` migration (one-to-one on tenant_id; customer/subscription/price ids, status, trial_ends_at, current_period_end, read_only_at).
- `stripe` pkg + `lib/stripe/client.ts` + `lib/stripe/prices.ts` (tier→price-id from env, throws if missing).
- `createCheckoutSession` server action (owner/admin gate, service-role customer upsert, `metadata.tenant_id` on **both** customer + subscription [load-bearing for slice 2], `trial_period_days: 30`). Writes only `stripe_customer_id` — authoritative state is slice 2's webhook.
- Interim plan selector on Settings→Billing (INTERIM test trigger, NOT final UI) + neutral success/cancel pages.

**Review caught one real defect before push:** Codex first copied the **tenant_agents** RLS (owner/admin INSERT/UPDATE/DELETE). That would let a tenant UPDATE their own `status`/`read_only_at` and **self-unlock without paying** once slice 3 lands. Corrected to **service-role-write-only, tenant-read-only** (DEC25 / tenant_mailboxes posture) — dropped the three write policies; checkout write already used `createAdminClient()`. Held the push until reviewed, then merged.

**Process notes worth keeping:**
- Codex carried the discovery task's read-only rule into the build and blocked on the `stripe` install — needed an explicit "that constraint was discovery-only" to proceed.
- Held push+PR until I'd eyeballed the migration + action, then green-lit.

---

## 4. Live verification (S199 Movers, sandbox)

The debugging trail (each cost real time — logged so slice 2 doesn't repeat):
1. **Blank Billing page** — first tested as **crew**; the selector is owner/admin-gated (correct). Switched to owner.
2. **Still blank** — was on the **wrong preview deploy** (an S83 redeploy, not the F30 branch). Each branch has its own preview URL; used F30's.
3. **"Could not create checkout session"** — Vercel function log showed `No such price: 'prod_…'`: env vars were set to **product** ids (`prod_…`), not **price** ids (`price_…`). Price id lives one level down under the product's Pricing row. Fixed + redeployed.
4. **Working.** All three tiers → Stripe hosted Checkout ("Try Starter / 30 days free / Then £29.00 / Up to 4 team members", Sandbox badge) → test card 4242 → success page.

**DB confirmed (MCP):** `tenant_billing` row for S199 = `stripe_customer_id = cus_VCM4…` populated; `stripe_subscription_id / status / trial_ends_at` NULL — **correct for slice 1** (that's slice 2's webhook job, not a bug). Table verified: 10 cols, RLS enabled, single `agents_select` (SELECT-only) policy = writes service-role-only as intended.

**NB:** the f30_tenant_billing migration applied via MCP `apply_migration` **wrote a schema_migrations ledger row** (non-orphan, like S83) — reconcile with the S130 pile.

---

## 5. Trial-eligibility (raised, not built)

Trial is currently **hardcoded 30d, always** — no eligibility check. Two cases:
- **Case A — same tenant cancels + re-subscribes:** check `tenant_billing` history, set `trial_period_days:0` if used. **Folds into slice 2** (webhook writes trial state anyway).
- **Case B — same person, new signup/tenant:** the real leak. Stripe **Radar free-trial-abuse control** (dashboard toggle, free, catches card-reuse across accounts) + optional cross-tenant email/card-fingerprint dedup = **S251**, "build if abuse appears." Low priority at pilot volume.
- **Architecture note:** marketing site *advertises* the trial; the **app server decides eligibility** at checkout-session creation (only it knows tenant_billing history). Keeps eligibility in one trusted place wherever the button lives. Funnel placement (signup vs post-onboarding) stays open, doesn't gate the logic.

**Tier feature-copy** on the Billing page (what each tier includes, not just price) = **F21's job**, not a Billing-page patch — the current selector is interim; adding real pricing copy now = throwaway before F21.

---

## Numbering (as of this session)

Next free: **S251 · F31 · DEC44.** New: **DEC43** (billing, LOCKED), **F30** (slice 1 DONE; slices 2–3 owed), **S250** (operator prereq, near-complete), **S251** (Case B trial dedup, deferred). **S248's candidate DEC bumped DEC43→DEC44** (still candidate). S178/S185 remain retired.

---

## Carried forward → next chat (slice 2)

- **F30 slice 2** — Stripe webhook (pattern on `agents/inbound` raw-body+signature) → sync subscription state to `tenant_billing`, idempotent; **+ trial-eligibility Case A**. Branch + PR.
- **F30 slice 3** — read-only enforcement (the hard one: middleware UI lockdown + shared-context-helper mutation gate).
- **S250 tail** — confirm Stripe payout-verification cleared.
- **S251** — Case B dedup + Radar toggle (deferred).
- Slice-1 branch `codex/f30-slice1-stripe-checkout` preserved (not deleted).
