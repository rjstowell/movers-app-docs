# Movers App — Session Summary 2026-08-31 (v3)

**main: db3c566** (six ships this session). No launch-plan restructure. Two migrations applied via MCP (`inbound_mode` reconcile for 3b-1; `forwarding_note_dismissed_at` for 3c). GitHub connector showed connected but its tools weren't reachable in-chat — code reads were done by paste; DB reads/writes by Supabase MCP.

This session finished **F23 slice 3a-2** and shipped **3b-1, 3b-2, 3c** plus a tsc fix — leaving **only slice 3b-4 (mode-switch teardown)** open in the whole of F23.

---

## What shipped (in order)

### 1. F23 3a-2 ignored-override half — DONE (PRs #79/#80, DEC39)
The second half of 3a-2. An "override" action on `ignored` Filtered rows = **escalate-to-queue for manual handling, not a draft**. Reasoning (DEC39): re-running an ignored row re-ignores it (same body/classifier/settings), and an ignored run has no category → no template → nothing to draft from. So override = pure `ignored`→`escalated` status flip.

- Owner/admin-gated (`requireOwner`), `.eq("status","ignored")` on the update = idempotency guard (double-click → 0 rows, no-op).
- No pipeline re-run, no dedup guard (escalate is inert + human-gated; worst case a human closes an already-handled item).
- DB discovery de-risked the build: an escalated row already carries `classification` non-null + category null (115/115 live), and `ignored` rows are the same shape (127/127) — so promoting `ignored`→`escalated` produces a row the Escalated tab already renders. **No migration, no special-casing.**
- Do NOT gate on body-present (unlike Recover) — no reprocessing, so null body is harmless.
- Verified on prod: escalated real junk rows (creditcontrol, eBay, +2), confirmed clean `status=escalated` / `escalation_resolved_at` null via MCP, then reverted all test rows to `ignored`.

**Slice 3a-2 is now fully closed.**

### 2. F23 3b-1 — door follows the toggle — DONE (PR #80, DEC40)
The behavioural core of 3b. **The inbound door now follows `tenant_agents.inbound_mode`, not mailbox presence** (amends DEC37).

Why it was needed: DEC37 derived the door from mailbox status, which silently demoted the toggle — a mailbox-connected tenant who flipped to app_only found the door didn't move. The toggle must be authoritative; mailbox becomes a *capability/precondition*.

- `getInboundDoor`: returns `mailbox` only when `inbox_only` AND an ok mailbox exists; else `letterbox` (fail-safe — mail never dropped).
- `syncTenantMailbox` guard: skip ingest unless `inbox_only`. **Keyed on MODE, not `getInboundDoor`** — an `inbox_only` tenant with a temporarily-broken mailbox must keep retrying to recover; keying on the door would compute `letterbox` and strand it.
- Reconcile migration: the one `app_only`+ok-mailbox tenant (`99c73f1b`) → `inbox_only`, making the merge behaviour-neutral (no tenant's door flips on deploy).
- `route.ts` needed no change (already routes off `getInboundDoor`).

**Verified on PROD, not preview** — see the masking-cron gotcha below. After merge, flipped S199 → app_only on prod and watched the cron stop advancing `last_success_at` (guard fired), then reverted.

### 3. F23 3b-2 — in-app toggle + config gating — DONE (PR #81)
The visible switch the 3b-1 backend was waiting for.

- Segmented toggle "How do you want to manage enquiries?" → In the app (`app_only`) / In my own inbox (`inbox_only`). Owner/admin-gated `setInboundMode` action.
- Config gating: letterbox **Inbound address** field hidden in inbox mode; **MailboxCard hidden in app mode** (UI-only — a previously-connected mailbox's config persists and reappears on switching back).
- Soft-warn (amber) when `inbox_only` without an ok mailbox; DEC37 note under inbox mode ("drafts are still approved in the app; turn on auto-send to work fully from your inbox").
- Reframed after review: heading changed from "How enquiries reach you" (read as status quo) to a forward management choice; two loose buttons → one joined segmented control.

### 4. F23 3c — app-mode forwarding note — DONE (PR #82)
Discovery collapsed 3c to a single piece: the agent-**setup wizard doesn't touch mail routing at all**, so there was no onboarding insertion point — the note *is* the slice.

- Dismissible amber note in the Setup tab: "In app mode, only mail sent to your inbound address is handled. Forward your business email there, or switch to managing in your inbox." Persisted via new `tenant_agents.forwarding_note_dismissed_at`.
- **Re-arm rule:** dismissal persists within an app-mode stint, but switching *into* app mode (from inbox) clears the flag so the note returns; a same-segment re-click is a no-op (doesn't re-arm). Reappears instantly on switch (local state reset, no refresh).
- Polish: note moved to the **top** of the tab; stray "Add more paths using AI" button removed (belongs on Generate); **Overview tab renamed "Setup"** (visible label only, `overview` key retained); inbound-address helper text sharpened to name the consequence.
- **3b-3 guardrail slice retired as covered/moot:** the connect-prompt is unreachable (app mode hides the MailboxCard, the only connect surface), and disconnect/broken-mailbox is already caught by the 3b-2 soft-warn.

### 5. S229 — tsc fix — DONE (direct to main)
The 3b-1 mailbox test files failed `tsc --noEmit` (Codex had called them "pre-existing" — they weren't). `inboundDoor.test.ts` used `mockAdmin.from.mockImplementation` → wrapped in `vi.mocked()`; `ingest.test.ts`'s `createMailboxRow` was missing required `tenant_mailboxes` Row columns → added them. Test-only + orthogonal → direct to main.

---

## Decisions locked

- **DEC39** — override of an `ignored` Filtered row = escalate-to-queue (not force-draft). Human-gated, no dedup guard, no reason column, no body gate.
- **DEC40** — inbound door follows `inbound_mode`; mailbox = capability/precondition, not the decider. Amends DEC37.

## New backlog item

- **F27 (OPEN, pending design decision — NOT decided)** — promote the forwarding note to a cross-tab / dashboard banner (the in-Setup-tab note is under-exposed for inbox-dwelling tenants). Open questions: above-tabs-only vs dashboard too; dismiss-per-surface vs shared flag; whether it earns stacking with the DNS banner; note it's a state-hoisting refactor, not a move.

---

## Gotchas that cost time (also logged in quirks)

- **Shared dev DB + prod-only Vercel cron masked the 3b-1 preview test.** Crons fire only on production, not previews. With prod and previews on the same Supabase project, the prod cron (old code) kept advancing `last_success_at` on the tenant we'd flipped on preview — looked like a broken guard; was the cron. Lesson: cron-touched behaviour can't be behaviourally tested on preview (see S223); for a behaviour-neutral backend change, verify via unit tests + code review then confirm on PROD after merge.
- **`Link` build failure.** A brief told Codex to drop an "unused" `Link` import; it was still used → `react/jsx-no-undef` → `next build` failed. `tsc` + vitest passed (neither runs ESLint). **Standing fix: briefs that touch imports/JSX must verify with `npm run build`/`next lint`, not just tsc + test.**
- **Data-modifying CTE read the pre-update snapshot.** `WITH flip AS (UPDATE … RETURNING) SELECT … FROM same_table` returned OLD values — a single query sees one snapshot. Use the UPDATE's own RETURNING, or a separate follow-up SELECT.
- **Codex "trust my own read" deviations recur** — verify claimed no-ops (it skipped an instructed same-mode guard claiming it was already handled; it happened to be right, but needed checking).

---

## State at session end

- **main db3c566, green** (`tsc --noEmit` clean, 58 tests, build passing).
- All tenants at correct `inbound_mode` (only the two mailbox tenants — S199 `8f31127f`, `99c73f1b` — on `inbox_only`; the rest `app_only`). S146 (`d7338ef9`) forwarding-note flag was nulled during testing and may be re-dismissed by the operator.
- **F23 remaining: ONLY slice 3b-4 (mode-switch teardown)** — discovery-first, not started. Plus **S218** (inbox-only auto-send delay note) still open.

## Next session

- **F23 3b-4 (mode-switch teardown).** Start with a discovery pass: what does an explicit mode switch actually strand? (in-flight pipeline runs — probably fine, finish under old door; parked `letterbox_dormant` rows; the sync cursor). Decide whether teardown is a real build or near-noop, then build against findings. **Do NOT scope blind.**
- Then **S218** (small) and **3c/F27** banner-promotion design if it earns a slot.
