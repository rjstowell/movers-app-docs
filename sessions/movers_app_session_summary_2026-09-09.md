# Session summary 2026-09-09

Reliability arc. Three ships, all verified live: S191, F33, F34. main advanced via PR #99 (F33) + PR #100 (F34); S191 direct-to-main bfc86c9. Three migrations applied via MCP. Codex hit its usage cap mid-F34, work finished on Claude Code on the same branch. Backlog split into active + archive this session (see Process).

## What shipped

### S191 DONE (bfc86c9, direct to main, no migration)
Pipeline status-writes stopped swallowing their errors. Discovery corrected the backlog's framing: the `error` column and `'error'` status already existed (DB-confirmed), and send-path failures already persisted text. The real gap was the inbound classify/draft pipeline ignoring the PostgREST error on every `.update()`, so a failed status-write left the run stranded in `received` with no trace (the S257 silent-swallow class, generalised). Live evidence: 8 runs stuck in `received`, `updated_at == created_at`, ages 2 to 59 days.

Fix: a checked-update helper throws on a failed write, so the existing outer catch converts it to `status='error'` + readable text. Outer catch made non-throwing (console fallback if even the error-write fails, to avoid webhook-retry loops); missing-run and fallback-write-failure both logged. Verified live (a real escalation) + a unit suite covering every branch.

### F33 DONE (PR #99, delivers S194). Failed runs get a home + auto-recovery
An errored run (status='error', now carrying readable text from S191) was invisible: no queue row, so a real customer enquiry sat unanswered with no signal.

Shipped:
- **Error card** in the Review Queue (reason='error'), red, showing a TRANSLATED safe cause and the customer's email; the raw error is never shown to the tenant.
- **Buckets by who-can-fix** (not by error type): transient (AI down/timeout/rate-limit) -> Retry; tenant-config (missing agent/template) -> Go-to-setup link + Retry; our-end (missing API key, check-violation, unknown) -> "we're on it", no retry. This correction mattered: some "config" errors (missing OPENAI_API_KEY, check violations) are OURS not the tenant's, so they belong in the our-end bucket from the tenant's view.
- **Auto-report** on our-end only: writes a `support_reports` row + emails admin@moversapp.app (both non-fatal). Transient never auto-reports (would spam on every OpenAI blip).
- **Manual "Report to support"** on any card: upserts the same row (idempotent via UNIQUE(agent_run_id)).
- **Auto-replay cron** (every 10 min, cap 5 via `auto_retry_count`): re-drives error runs through the pipeline, self-healing transient / token_cap / config / our-end once the fault clears. This is what makes the card's "this will update once resolved" promise true. Runs at the cap are left for a human.
- **Recovery copy + Move to Escalated**: honest card copy ("this enquiry is safe, this card will update with a reply once resolved, or move it to Escalated and respond yourself") + a Move-to-Escalated button.

Migrations (via MCP): `support_reports` (service-role-only RLS, UNIQUE on agent_run_id) + `agent_runs.error_bucket` + `agent_runs.auto_retry_count`. Reconcile with S130.

Verified live end-to-end: our-end card + support_reports row + real admin email (with a seeded error run, since the pipeline path can't be exercised on a preview); manual report idempotent (one row, no dupe on second click); Move to Escalated relocates the run; retry recovery (error -> drafted/escalated, count 0 -> 1) via a manual cron trigger; cap held at 5 (at-cap run skipped); null-body run correctly skipped; unauthenticated cron rejected (401). The auto-replay run also legitimately recovered 2 real pre-existing stranded runs.

### F34 DONE (PR #100). Reply to the customer from the Escalated tab
The Escalated tab was read-only (Mark-as-done only) so every escalation was a dead end unless the tenant left the app. A genuine launch hole for the "manage enquiries in the app" promise.

Shipped: a run-based `sendEscalatedReply` action that reuses the existing transport with NO queue row or draft (the transport was already run-capable); a small composer (subject prefilled "Re:", plain-text body). On success the run STAYS `status='escalated'` with `escalation_resolved_at/by` set (mirrors Mark-as-done, keeps it auditable in place), and stores `manual_reply_body_html` + `manual_reply_sent_at` + the real `outbound_email_id`. An atomic compare-and-set claim on `outbound_email_id` (pending token before send, real id after) blocks concurrent double-send; the claim is RELEASED on a send failure (retry stays possible) but RETAINED on send-ok/save-fail (never double-email a delivered reply). Mailbox writeback fixed to search the ESCALATED folder (the inbound was moved there on escalation, so the old watch-folder search missed it).

One bug caught in review before merge and fixed on the branch (144b5db): the send-failure path did not clear the claim, which would have blocked a legitimate retry of a reply that never sent. Fixed with a guarded clear on that path only; the save-fail path deliberately keeps the claim.

Migration (via MCP): `manual_reply_body_html` + `manual_reply_sent_at`.

Verified live: a real reply delivered to the customer address and correctly threaded (In-Reply-To set), card flipped to done in place, DB persisted, claim swapped to the real Resend id. The send-gate (unverified sender + no consent) fired correctly first; to test the send, a consent-ack row was inserted for Testing Mover and removed after (state restored).

## Decisions taken (none formally locked as a DEC)
- Error buckets classified by WHO CAN FIX, not by error type.
- Our-end errors auto-report (row + admin email); transient never auto-reports.
- Recovery is a scheduled cron (not operator-manual, not tenant-facing retry-that-loops), covering all self-healing buckets, cap-bounded.
- Replied escalations stay in the Escalated tab as "done" for launch; a proper Sent tab is the long-term home (F35).
- em-dash / en-dash preference moved into userPreferences (applies every session now).
- The our-end-auto-alert + auto-replay-recovery model was discussed but NOT locked as a DEC (candidate DEC51 if ever wanted).

## New backlog items
- **F35** Sent tab (fourth Inbox tab; correct long-term home for replied escalations). Post-launch.
- **S267** Escalated-reply blocked-send exit route (the send-gate UX gap F34 exposed: a blocked reply offers only Mark-as-done, no path to fix). Tier-2.
- **S268** Retry-atomicity hardening (unique constraint on agent_review_queue; deferred from F33's guard-and-accept). Tier-3.

## Method / notes
Every Codex and Claude-Code claim was DB/live-verified via the Supabase MCP. That discipline caught, this session: the escalated-reply send-gate block (not an F34 bug); a wrong-branch preview (main, not the F34 branch) that made a seeded card render as old code; and the apex->www redirect dropping the Authorization header on the manual cron curl (fixed by hitting www directly). Codex usage-limited twice (mid-F33 and mid-F34); F34's fix finished on Claude Code with a cold-context brief (read-the-function-by-name, don't-trust-line-numbers).

## Process change: backlog split
The master backlog had grown to ~560k, taxing every swap-in. Split into two files this session:
- `movers_app_master_backlog.md` (active) = current header + current numbering + OPEN items + live lesson/quirk/DEC blocks. ~419k.
- `movers_app_backlog_archive.md` (archive) = all previous session headers, all previous numbering blocks, all DONE items. Searched on demand, never loaded for routine swap-ins. ~157k.
Only headers + numbering chains + DONE items moved (safe, mechanical). The 25% cut is secondary; the point is that the two biggest growth drivers (a giant new header each session + accumulating done items) now flow to the archive, so the active file stops ballooning. Going forward, each session's outgoing header and newly-done items go to the archive at doc time.
