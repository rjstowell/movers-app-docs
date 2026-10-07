# Movers App — Session Summary 2026-09-07

**Headline:** F30 **slice 3 (read-only enforcement) COMPLETE** — the last piece, slice 3b-2c, shipped and proven live end-to-end. Four defects surfaced during testing and all fixed live (S257 the notable one). S256 proven, S250 closed, S253 reframed+parked, **S254 Half A shipped**. Two merges to main: `1d7fb29` (PR #97 — 3b-2c + S257/S258/S259) → `f87ab5d` (S254 Half A, direct).

Environment: GitHub connector still unavailable in-chat — worked via file-paste + local git + Supabase MCP all session. Codex hit its usage limit late; final S254 merge done by hand in PowerShell.

---

## F30 slice 3b-2c — per-item reactivation drafting (DONE, PR #97)

**What it does.** When a tenant is read-only (billing locked), inbound enquiries are captured raw (`captured_locked`) with no LLM touch. After they pay, each captured row gets a **Generate reply** button that re-drives the pipeline on the existing raw row.

**Reuse discovery.** `recoverFilteredRun` (re-drives `processInboundRun`) was the template. The one gap reuse didn't cover: a `captured_locked` row already has a queue row, so a naive re-drive inserts a *second* → the action deletes the old row, but **only on advance** (kept on error/re-lock, so a failed re-draft stays visible + retryable). Guard-before-pipeline (blocks Generate reply while still locked). Action keys off the queue **`reason`**, not run status.

**UI (ReviewQueueTab).** captured rows show a red **"No draft. Billing issue."** pill + explanatory note + **Generate reply/Discard only** (Approve+Send/Edit+Send suppressed — they'd have sent a blank email off the null-draft row), and the misleading default "Drafted" status badge is suppressed.

**Live proof (S199, real Stripe cus_VCM4…):**
- A — capture renders correctly (pill/note/buttons, no Send).
- B — Generate reply while `canceled` → inline "Reactivate your subscription…", no draft.
- C — flip active → Generate reply → real LLM draft, `run_status=drafted`, exactly one queue row (`auto_send_off`), old captured row deleted (no duplicate).

Staging note: to get a real `captured_locked` row, S199 needed BOTH `tenant_billing.status='canceled'` AND `tenant_agents.inbound_mode='app_only'` — in `inbox_only` mode app-inbound mail diverts to `letterbox_dormant` (Filtered) before the read-only gate. All overrides snapshotted + restored.

---

## S257 — the live-only bug (DONE, migration)

Capture left `agent_runs.status='received'` (should be `captured_locked`). **Root cause was NOT the code:** pipeline.ts already issued `update({status:'captured_locked'})` — it failed **silently** because `agent_runs_status_check` didn't include `'captured_locked'`, and the update had no error check. The queue insert survived (its `reason` CHECK was fixed back in PR #94; the runs-table CHECK was missed).

Codex twice reported "already done, no changes needed" from reading the source. The **DB disproved it both times** (fresh live captures showed `received`). Fix = a CHECK-constraint migration adding the value — applied live via MCP `f30_agent_runs_status_captured_locked` (v20260907093722) + a committed file (`20260907000000_…`, idempotent DROP-IF-EXISTS+ADD; dup ledger entry → **S130 reconcile**). Proven on a fresh **v3** capture: `run_status=captured_locked`.

**Lesson reinforced:** unit/green tests can't hit a DB CHECK; only live DB-verify catches a silent constraint rejection.

## S258 — portal live-verify (DONE)
LockedBanner "Reactivate" → `createPortalSession` → real Stripe portal against cus_VCM4…. Wrinkle resolved: the sub was `cancel_at_period_end` (not dead) so the portal offers "Don't cancel subscription" = reactivation — no fresh-checkout dead end.

## S259 — portal-error UX (DONE, PR #97)
Both banners wrap `createPortalSession` in try/catch + no-url guard → inline error, button stays enabled. Success redirect intact (verified in diff). Code-reviewed only (can't fire where the portal works).

## S256 — webhook writes cancel_at_period_end (DONE, proven)
S199 portal "Don't cancel" → `customer.subscription.updated` webhook wrote `cancel_at_period_end` true→false (`updated_at` matched the click, not MCP). Also confirms the webhook reaches a live endpoint.

---

## S254 Half A — seat-add cap (DONE, main f87ab5d)

Numeric `maxSeats` (starter 4 / pro 10 / scale 20) + `label` added to `billingTierDetails` (was display copy only). `inviteMember` blocks a new seat at the tier cap — counts active+pending+invited (role-agnostic, service-role), grandfathers no-billing tenants (null tier → uncapped), guard after the existing-member early-return so re-invites aren't blocked.

**Live proof (S199):** filled to Starter's 4 via 3 borrowed mailinator test users → 5th invite blocked with "Your Starter plan allows up to 4 team members…" → filler removed.

**Still open:**
- **Half B (downgrade-below-seats)** — warn-only, NOT cut. A portal downgrade is a fait accompli; hard-cutting seats would log out real people/owners mid-job. Compute over-seat on read + nudge banner. Needs `app/api/stripe/webhook/route.ts`. Lower stakes pre-launch (Half A stops growth past the band).
- **TOCTOU race** — count-then-insert isn't atomic; two simultaneous invites could both pass at cap-1. Human-rate pre-launch; DB-level check only if it ever matters. (Codex flagged it rather than silently "fixing" — correct.)

---

## Operator items

**S250 — CLOSED.** Live Stripe account fully verified, Monzo bank linked for payouts (payments balance + Treasury, default GBP). The sandbox "capabilities paused" banner was **sandbox-only** and used placeholder owner data (Marie Dupont / Mark Medina — Stripe's fictitious test data, per Stripe docs) — not the live blocker. Distinct from the Monzo *debit card* (still pending → S253). Caution logged: the **live** verification, when it triggers, needs real accurate UBOs/directors (real KYC), unlike the sandbox dummies.

**S253 — reframed + parked on the Monzo card.** Personal-org 30-day spend = **$0.60**, so a **$120/mo hard cap ≈ 200×** honest load = a fine catastrophe ceiling. Post-2026 OpenAI change: the $ number alone is **alerts-only** — must toggle **"Enforce a hard limit" ON** (org level) + a spend alert ~$50–80 below. Now bundles **LTD OpenAI provisioning**: LTD account → "Movers App" project → migrate `OPENAI_API_KEY` (Vercel+local) + re-verify a live draft → then the cap. **Launch-time DEC candidate:** an org cap is shared-fate (a coordinated trial-abuser could aggregate to it and 429 all tenants); mitigations = keep it high, S252 per-tenant counter, DEC43 card-upfront; structural fix = per-project caps (heavy, overkill pre-launch).

---

## Method note
Every Codex claim was DB/live-verified this session. That discipline caught S257's silent constraint failure that both green tests and Codex's own word missed — twice. Trust-but-verify against the DB, not the report.

## Open after this session
F30 slice 3 done. Remaining Tier-1: **F21** (marketing site + privacy policy — now unblocked by the billing decision), the correctness/hardening cluster (S191→S194 etc.), S244 (bounce/complaint suppression). Billing tail: **S254 Half B**, S255. Small: S211 (em-dash sweep — includes both billing banners), S228. Blocked on Monzo card: S253.
