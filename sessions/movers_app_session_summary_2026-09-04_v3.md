# Movers App — Session Summary 2026-09-04 v3

**main ce57cb9 → e125e11. Four F30 slices shipped + verified live (slice 2, slice-2 hotfix, slice 2.5, slice 2.6), read-only enforcement (slice 3) fully DESIGNED not built. Three migrations applied live via MCP (all non-orphan ledger rows → S130). GitHub connector still unavailable in-chat — worked via file-paste + MCP all session.**

Long build session, entirely on F30 (billing). Slice 1 (checkout) was already merged coming in; this session added the webhook that syncs subscription state, self-serve cancellation, and state-aware billing UI — then designed the read-only lock in full. The whole billing loop (subscribe → state shown → cancel → cancel-state shown → resubscribe) is now proven live in the Stripe sandbox on S199.

**Session goal:** push F30 as far as possible in the window — turned into slice 2 + 2.5 + 2.6 shipped and slice 3 designed to a discovery-backed build map.

---

## Headline outcomes

1. **F30 slice 2 DONE** (merged, squash 4548253, PR #89) — Stripe webhook → syncs live subscription state to `tenant_billing`; idempotent + order-independent (re-fetch live sub, full-state upsert); trial-eligibility Case A folded in (`trial_used_at` write-once stamp). Verified live: checkout → webhook 200 → row filled correctly.
2. **F30 slice 2 hotfix DONE** (direct to main, b2df8e2) — `trial_period_days: 0` is invalid to Stripe (min 1); repeat-trial checkout now OMITS the field instead. Caught live on the Case A test; confirmed against Stripe docs.
3. **F30 slice 2.5 DONE** (merged, PR #90) — Stripe Customer Portal: self-serve cancel + manage billing (the compliance hole). `createPortalSession` action + Manage-billing button. Cancellation events flow into the slice-2 webhook for free.
4. **F30 slice 2.6 DONE** (merged, squash e125e11, PR #91) — billing-page state display: current tier badged "Current", seat bands on all cards, NO checkout buttons when already subscribed (closes a double-billing hole), cancel-ending banner. Also added `cancel_at_period_end` capture (webhook + column).
5. **F30 slice 3 (read-only enforcement) FULLY DESIGNED** — capture-without-cognition model, reusable reactivation wall, ~13 enforcement points, the LLM-client wrapper as the near-universal chokepoint. Discovery pass done. NOT built. This is the remaining F30 build blocker.
6. **New backlog items** surfaced: LLM daily-token guard/counter (S252), OpenAI account hard-cap (operator, S253), seat-cap enforcement on plan-switch/downgrade (S254), dead-tier→portal-switch UX polish (S255), webhook cancel_at_period_end write-verify (S256). Numbers assigned in the backlog swap-in.
7. **DEC44 offered** — the read-only enforcement model. (S248's candidate DEC44 claim resolved — see numbering.)

---

## 1. Slice 2 — Stripe webhook (state sync)

**Built** (`app/api/stripe/webhook/route.ts` + `lib/stripe/webhook.ts` helpers):
- Raw body via `request.text()`, `runtime="nodejs"`, verify with `stripe.webhooks.constructEvent(rawBody, sig, STRIPE_WEBHOOK_SECRET)` — NOT svix (the inbound route's pattern was the shape reference only; its svix verify + `waitUntil` were deliberately NOT copied).
- **Synchronous processing, no `waitUntil`** — on failure returns non-2xx so Stripe retries. That retry-on-failure IS the reliability net; `waitUntil` would 200 before the work and defeat it. Stripe's 20s timeout leaves ample room for a retrieve + upsert.
- Handles `checkout.session.completed` + `customer.subscription.{created,updated,deleted}`; ignores others with 200.
- **Idempotency + ordering mechanism:** for every event, re-fetch the LIVE subscription (`stripe.subscriptions.retrieve`) and upsert its full current state — never apply event deltas. Duplicates/retries/out-of-order events all converge on live truth. No processed-events table needed at this volume.
- Tenant mapped via `subscription.metadata.tenant_id` (primary), `stripe_customer_id` lookup (fallback); unmapped → 200 (no retry loop).
- Writes via `createAdminClient()` (service-role; RLS is service-role-write-only). Does NOT touch `read_only_at` (slice 3's).
- `current_period_end` read off the subscription **item** (`item.current_period_end`) — correct location in the pinned Stripe API version, not the subscription object.

**Trial-eligibility Case A:** new `trial_used_at timestamptz` column (migration `f30_trial_used_at`, applied live via MCP). Webhook stamps it write-once (`UPDATE … WHERE trial_used_at IS NULL`) whenever the sub has/had a trial. `createCheckoutSession` reads it → repeat customer gets no trial.

**Verified live (S199 sandbox):** checkout → both events delivered 200 `{ok:true}` → row filled: `stripe_subscription_id` set, `status=trialing`, `trial_ends_at` exactly 30d out, `current_period_end` set (epoch→timestamptz correct, real date), `trial_used_at` stamped, `read_only_at` null. Two events converged to one consistent row = idempotency proven.

**LIVE DEBUG (cost real time, logged):** first deliveries failed **308** — the endpoint URL was registered as `https://moversapp.app/...` but the site 308-redirects apex→`www`, and **Stripe does not follow redirects on webhook delivery** (treats 3xx as failed). Fixed by registering the endpoint as `https://www.moversapp.app/api/stripe/webhook`. See quirks.

## 2. Slice 2 hotfix — trial_period_days: 0

The Case A path set `trial_period_days: billing?.trial_used_at ? 0 : 30`. **Stripe rejects `trial_period_days: 0`** (the param must be ≥ 1; "no trial" = OMIT the field). First checkout (30) worked; the repeat (0) threw → generic "Could not start checkout." Unit tests passed because they don't hit the live Stripe API. Fix: build `subscription_data` conditionally, spreading `trial_period_days` only for a first trial. Verified live: post-cancel re-checkout showed £29 charge, no trial line, no error. See quirks.

## 3. Slice 2.5 — Customer Portal (cancel / manage)

`createPortalSession()` in `settings/billing/actions.ts` — mirrors the checkout action (auth → getActiveMembership → owner/admin gate → admin-client reads `stripe_customer_id` → `stripe.billingPortal.sessions.create({ customer, return_url })` reusing `checkoutOrigin`). "Manage billing" button on the Billing page, owner/admin-gated, shown only when a customer exists.

**Operator config (Stripe sandbox → Settings → Billing → Customer portal — NOT the Billing data menu):** cancellation set to **"At end of billing period"** (not immediately), payment-method update on, invoice history on, plan-switch across the 3 prices on, quantity-change OFF (tiers are flat-priced seat bands, not per-seat quantity — quantity-on would let someone set qty 5 on the £29 price). Portal API 400s until saved once. Redirect link set to `https://www.moversapp.app/settings/billing`. Portal header + public business name should read **Movers App**, not the legal entity Future Apps LTD (legal entity stays; only the customer-facing display name changes).

**Verified live:** cancel via portal round-tripped, webhook synced the row (cancel-at-period-end → sub stays `active` until 4 Oct, which is correct Stripe behaviour). Portal does NOT auto-redirect after cancel — the return is a click by design.

## 4. Slice 2.6 — billing-page state display + cancel_at_period_end

**Surfaced by the slice-2.5 cancel test:** we weren't storing `cancel_at_period_end`, so a cancelled-but-active sub was indistinguishable from a healthy one. Added `cancel_at_period_end boolean not null default false` (migration `f30_cancel_at_period_end`, applied live via MCP) + one field to the webhook upsert.

**Billing UI now state-aware** (`page.tsx` server read + `BillingPlanSelector.tsx`):
- `hasActiveSubscription` = has a subscription id AND `status ∈ {active, trialing, past_due}` (past_due + trialing correctly count as subscribed — a past_due tenant can't spin up a second sub).
- All tiers show their seat band always (single source of truth `billingTierDetails[tier].seatBand` in `prices.ts`; reverse resolver `getTierFromPriceId`).
- No active sub → checkout buttons per tier. Has active sub → current tier badged "Current", NO checkout buttons anywhere (closes the double-billing hole), single "Manage billing" button.
- `cancel_at_period_end = true` → amber banner "Subscription cancelled, active until {date}."

**Verified live (S199):** all three tiers render, Starter badged "Current", no checkout buttons, Manage billing → portal, and (after flipping `cancel_at_period_end` via MCP — see note) the amber banner rendered correctly.

**NOTE — banner test was display-only:** S199 was cancelled EARLIER in the session, before the `cancel_at_period_end` column existed, so the webhook that synced that cancel had nothing to write there (`updated_at` predates the migration). The flag was set directly via MCP to prove the banner renders. This means **the webhook WRITE of `cancel_at_period_end` is NOT yet proven live** — low risk (one field on an upsert we've seen work), but flagged as S256: verify on the next real cancel event (or a fresh re-cancel).

**Blemish:** the banner copy uses an em dash ("cancelled — active until"). Feeds the S211 em-dash sweep.

---

## 5. Slice 3 (read-only enforcement) — FULL DESIGN (not built)

The core of the launch billing story: what a non-paying / cancelled tenant can and can't do. Locked design over several exchanges:

**Model = "capture without cognition."** A locked tenant keeps the cheap DB-only surfaces; loses the LLM-powered flagship value.

**Fully open (cheap, encourages reactivation):** Settings (must stay — Billing lives here), Vehicles, Jobs core, Calendar (incl. new entries), Dashboard. All are DB reads/writes, fractions of a penny.

**Blur-wall (reactivation wall — blurred page + centred "Reactivate to keep using" → Billing):** ONE reusable component in the app layout, wrapping `/quote` first. Reports IS built (`app/(app)/reports/page.tsx`) and inherits it; Video Quote is `/quote?tab=video` so it's covered by wrapping `/quote`. **Blur is cosmetic — the real lock is the server-side gate; the blurred page still has live data/buttons underneath, so it can never BE the enforcement.**

**Inbound = capture, then gate before cognition.** A locked tenant's incoming email is still received + stored + placed in the Review Queue as a RAW enquiry (zero LLM) — so no end customer's enquiry is silently dropped (the S194 harm), and the queue-full-of-raw-enquiries is a visible carrot. The **classify + draft LLM calls are skipped**. Guard sits in `pipeline.ts` after run-row creation, before `computeDraftForInput`.

**On reactivation:** raw captured enquiries stay raw; tenant manually triggers a draft per item. **No bulk-draft button** (200 clicks = human-rate-limited, no token spike; and it's deferred normal spend by a now-paying tenant, not a leak). Not just for launch — the better solution.

**Enforcement points (discovery: NO single mutation chokepoint — ≥6 side-effect entry points + 7 more LLM action paths, each with its own local membership check):**
- **The unlock:** nearly every LLM entry point funnels through the shared LLM client (`classify.ts`, `openaiClient.ts` for quote). So build a **`guardLLM(tenant_id)` wrapper at the shared LLM client** — checks read-only lock **AND** the per-tenant daily token counter (S252) **AND** increments — covering all ~10 LLM entry points at ONE site. This is simultaneously the LLM-half of read-only AND the abuse cap.
- **Non-LLM guards still needed per-site:** auto-send cron (`auto-send/route.ts` `runAutoSend`, `sendTenantReplyEmail` ~line 161 — guard already-scheduled rows); one-click send (`approveQueueItem` agents/actions.ts:2047) + edit-and-send (`editAndSendQueueItem` :2123) — both on local `requireOwner`, not a shared gate.
- **`isReadOnly(tenant_id)` helper** in `lib/tenant/active.ts` (its natural home) — the shared lock lookup. `read_only_at` maps from `status` (stamp on past_due/canceled/unpaid, clear on active/trialing) — **nothing sets `read_only_at` yet**; that mapping is slice-3 work alongside the guard.
- **Frontend wall** needs billing state in the layout — pages don't currently receive it; fetch server-side in the app layout, pass to the wall component.

**Suggested build order:** slice 3a = `isReadOnly` + `guardLLM` wrapper + counter table + `read_only_at` mapping (self-contained, highest value — it's the abuse cap AND the LLM lock). slice 3b = the 3 send/inbound guards + the reactivation wall + billing-state-into-layout.

**Cancel path (slice 2.5) sequences BEFORE enforcement** — a tenant must be able to exit cleanly before we lock them out. That's done.

---

## 6. Abuse / LLM-cost defence (designed)

Trigger: a trial account (valid card, £0 charged, full access) or bot could smash LLM usage in a day before cancelling. Two layers, both needed:
- **OpenAI account/project HARD spend cap** (dashboard budget limit + alert) — operator task, ~5 min, backstops EVERYTHING (trial abuse, bot, runaway bug, our own loops) regardless of app code. → **S253, do soon.**
- **Per-tenant daily TOKEN counter** (not call-count — a token-smash is few calls, giant payloads; token sum defends it, and `usage.total_tokens` is on every response so it's ~free at the wrapper). One generous ceiling = N× the heaviest plausible honest day (a circuit breaker, not a cost optimiser; NOT cost-pro-rata — too restrictive, usage varies wildly). **Hard block, "resets midnight."** Lives in `guardLLM`. Number = a 5-min calc at build time from the cost model. → **S252.**
- **Terms:** the fair-use daily-limit clause goes in the ToS (F21 owns terms) — the fallback for disgruntled-client conversations.

---

## 7. Seat-cap gap (surfaced, new item)

The Customer Portal lets a tenant self-downgrade (e.g. Scale→Starter) and Stripe just switches the price — **Stripe has no idea about the seat bands** (≤4 / 5–10 / ≤20). So a 15-seat tenant could self-downgrade to the £29 ≤4 plan and Stripe won't stop them. **The app must enforce seat caps** on plan-switch/downgrade; not built (DEC43 set seats as the differentiator but not the enforcement). → **S254.** Not launch-blocking, but real.

---

## Numbering (as of this session)

Next free: **S257 · F31 · DEC45.** New this session: **S252** (per-tenant daily LLM token counter/guard), **S253** (OpenAI account hard spend-cap — operator), **S254** (seat-cap enforcement on plan-switch/downgrade), **S255** (dead non-current tier cards → "Switch to X" via portal, UX polish), **S256** (verify webhook writes `cancel_at_period_end` on a real cancel event). **DEC44** = read-only enforcement model (offered/locked this session — resolves the S248 candidate claim on DEC44; S248's candidate DEC now floats to DEC45 if still wanted). **F30** slices 2/2.5/2.6 + hotfix DONE; slice 3 designed, the remaining F30 build blocker. S178/S185 remain retired.

Three migrations applied live via MCP `apply_migration` (all wrote non-orphan `schema_migrations` rows): `f30_trial_used_at`, `f30_cancel_at_period_end` (both this session), reconcile with the S130 pile.

---

## Carried forward → next chat

- **F30 slice 3a** — `isReadOnly` helper + `guardLLM` LLM-client wrapper + per-tenant daily token counter (S252) + `read_only_at` status-mapping. Self-contained; abuse cap + LLM lock in one.
- **F30 slice 3b** — auto-send/send/inbound guards + reactivation wall component + billing-state-into-layout.
- **S253** — OpenAI dashboard hard spend cap (operator, do soon — independent of code).
- **S256** — verify `cancel_at_period_end` webhook write on a real cancel (or re-cancel S199 fresh).
- **S250 tail** — confirm Stripe payout-verification cleared (the sandbox "capabilities paused" banner is this).
- **S254** — seat-cap enforcement. **S255** — tier-card UX polish. **S211** — banner em-dash.
- Branches `codex/f30-slice2-stripe-webhook`, `codex/f30-slice2_5-customer-portal`, `codex/f30-slice2_6-billing-state` preserved (not deleted).
