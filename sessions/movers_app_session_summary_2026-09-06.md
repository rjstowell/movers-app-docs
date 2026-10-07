# Movers App — Session Summary 2026-09-06

**Focus:** F30 Slice 3 (read-only / non-payment enforcement). Shipped the full read-only backend + display layer, redesigned the lock model, fixed two real bugs caught in live verification.

## Shipped + verified live (all on prod `main`)

### 1. Slice 3a — token cap + counter (merged `845ad0c`)
- `guardLLM` wraps `callLLM` (the single LLM chokepoint in `lib/llm/openai.ts`); covers all LLM entry points at one site.
- `tenant_id` made a **required** param on `callLLM` → compiler-enforced coverage. Caught 2 LLM call sites Phase 0's manual scan missed (8 → 10: `normaliseOnboardingServices`, `generateOnboardingNames`).
- `tenant_llm_daily_usage` counter table, service-role-write-only RLS (tenant can't self-reset).
- `isReadOnly` helper + `read_only_at` webhook mapping (later superseded — see #4).
- Verified live: test enquiry classified/drafted, counter incremented, no accidental lock.

### 2. Counter-accumulate fix (PR #93, merged `0e7268c`)
- **Bug** (found during 3b-1 verification): counter **overwrote** instead of summing — `tokens_used` went 2548 → 431 within the same day. Root cause: supabase-js `.upsert()` *replaces* on conflict; it can't do `col = col + value`. This silently defeated the S252 daily cap (never accumulated toward the ceiling).
- **Fix:** atomic Postgres function `increment_tenant_token_usage()` (`INSERT … ON CONFLICT DO UPDATE SET tokens_used = tokens_used + EXCLUDED`), called via `.rpc()`. Preserved null-safe / success-only / non-fatal properties.
- Verified: isolated MCP test (431 → 1931 on +1000/+500, exact) and a real enquiry (431 → 2886).

### 3. Slice 3b-1 — backend enforcement (commit `3ffeb19`)
- Inbound **capture-without-cognition**: guard before `computeDraftForInput`, zero LLM under lock, inserts a raw queue row.
- Send guards: auto-send cron reverts locked rows to `drafted` + clears `send_after`; `approveQueueItem` / `editAndSendQueueItem` blocked with "Reactivate your subscription to send."
- ⚠️ **Process:** pushed straight to `main`, no PR (rule violation — see process notes).
- Check 1 (not-locked / Cornwall path safe): PASS.
- Check 2 (lock fires): **FAILED first pass** — `reason='captured_locked'` violated the `agent_review_queue_reason_check` CHECK constraint → enquiry dropped to `error` (the exact S194 "lost enquiry" harm). Live-only constraint bug; unit tests structurally can't catch it. Fixed in #4.

### 4. Read-only grace model + reason-CHECK fix (PR #94, merged `0496347`) — DEC44 refinement
- **reason CHECK:** added `captured_locked` to the allowed set (fixes the drop-to-error bug).
- **Grace model (replaces "lock on any non-active/trialing"):**
  - `active` / `trialing` → not locked
  - `past_due` → **7-day grace** from first past_due; not locked until grace expires
  - `canceled` / `unpaid` → locked immediately
  - no billing row → not locked (grandfathers Cornwall / test tenants)
- `tenant_billing.grace_started_at` column added; `isReadOnly` rewritten **compute-on-read** (no cron — nothing fires at the 7-day mark, so a stored flag can't do it). `GRACE_PERIOD_DAYS = 7` shared constant.
- Webhook now manages `grace_started_at` (write-once on first past_due; cleared on active/trialing recovery; not reset on dunning retries). `read_only_at` column retained but no longer authoritative.
- Re-verified Check 2 live: grace-fresh → cognition runs (tokens climbed); grace-expired → raw `captured_locked` capture, zero tokens, no error; UI send-block confirmed. All pass.

### 5. Slice 3b-2a — grace banner (PR #95, merged `b0f6cdf`)
- `getBillingDisplayState()` returns `{ locked, inGrace, graceDaysLeft }`.
- `GracePeriodBanner` (red) renders only when `inGrace`: "Billing issue — resolve within N day(s)…" + "Resolve now" → `createPortalSession`.
- Verified on preview: 7 days, 1 day (singular), no-row → no banner.

### 6. Slice 3b-2b — locked banner (PR #96, merged `61c245f`)
- `LockedBanner` (slate, PauseCircle) renders when `locked && !inGrace`: "Your service is paused — reactivate…" + "Reactivate" → `createPortalSession`.
- Mutual exclusivity guaranteed at source (`inGrace` only sets when `!locked`).
- Verified on preview: grace-expired lock, canceled lock, no-row hides.

## Outstanding — Slice 3b-2c (last piece of Slice 3)
- **Per-item reactivation drafting** — re-draft `captured_locked` rows after payment (re-invokes the LLM pipeline on existing rows; functional + riskier).
- **`captured_locked` run-status fix** — capture currently leaves `agent_runs.status = 'received'` (only the queue `reason` is set). Set the run status too, and make reactivation key off the queue **`reason`**, not run status.
- **Portal wiring verification against a real Stripe customer** (S199) — deferred. Test rows use fake `cus_TEST_*` ids, so "Resolve now"/"Reactivate" error at Stripe; the portal action itself was proven in slice 2.5. Verify end-to-end as part of 3b-2c.

## Process notes (Codex behaviours to watch — quirks candidates)
- **Straight-to-main relapse:** 3b-1 pushed without a PR despite the brief. Next briefs need blunt "no direct main push; branch + PR only; STOP if you can't PR."
- **Hardcoded secret:** Codex printed the Supabase service-role key in an ad-hoc `node -e` script. Key was **rotated** this session (local + Vercel). Codex should use MCP for DB checks, never hardcode secrets.
- **Over-claiming "done":** claimed `status='captured_locked'` was set (it wasn't); claimed counter "counting works" off a single reading (it was overwriting); claimed guard "before cognition" (needed live confirmation). Verify against DB, not the report.
- **Live-only bugs:** unit tests pass but can't hit live DB CHECK constraints (reason CHECK) — always run Check 2 live.
- **Migration timestamps future-dated** (`20261219…`) — S130-adjacent debt.

## Note
`movers_app_master_backlog` was not present in the project file mount this session — backlog updates are provided as a separate apply-list rather than a swap-in.
