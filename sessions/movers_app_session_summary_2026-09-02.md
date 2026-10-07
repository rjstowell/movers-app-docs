# Movers App — Session Summary 2026-09-02

**main: db3c566 → f6a5e4d.** Four items shipped (F28, S230, S217-close, S218), F23 fully closed (3b-4 = noop + DEC41), one real launch bug uncovered at the end (F29). One migration (S230 seed restore, via MCP + committed, orphan #~25). Supabase reads/writes via MCP; code reads via paste (GitHub connector still not reachable in-chat).

The session started as "close out the mailbox launch block" (F23 3b-4 + S218) and turned into two genuine bug discoveries — a mislabelled-as-done DEC35 mechanism (F28), a silently-drifted DB function (S230) — plus a live launch-blocker found while eyeballing an S218 copy line (F29).

---

## What shipped (in order)

### 1. F23 3b-4 discovery → CLOSED as noop (DEC41)
Discovery-first pass (read-only, code + live DB). An explicit `app_only`↔`inbox_only` mode switch strands nothing not already accepted:
- **Cursor replay** (inbox→app→inbox): sync resumes from the frozen `tenant_mailboxes.sync_cursor` and catches up (~50/run cap), but cross-source `message_id` dedup (`isDuplicateMessage`) stops any double-reply → wasted work only, never a double-send.
- **Parked `letterbox_dormant` rows**: only S199 had any (4, 3 unresolved); visible + manually actionable in Filtered, not orphaned. No auto-cleanup on switch by design.
- **In-flight at flip**: finishes under pre-flip semantics; same accepted class as the DEC35 residual window.
- **REJECTED the discovery's one suggested "build"** (gate `syncTenantMailbox` on `getInboundDoor` instead of raw mode) — that regresses DEC40: the asymmetry is deliberate (mode-keyed sync lets a broken inbox_only mailbox keep retrying/recover; door-keyed would strand it). Codex pattern-matched "mismatch = bug" without the DEC40 context.
**Verdict: no code. DEC41. F23 fully closed.**

### 2. F28 — mailbox auto-send parity (delivers DEC35). PR #83 (dafc5c1) + cleanup 91b8939
The scary find: **DEC35's safety mechanism was assumed built but wasn't.** DEC35 said mailbox auto-send is safe because auto-replies file to a "Responded" folder (disjoint from the human pile). Codex investigation found neither half wired: `pipeline.ts` force-blocked mailbox runs from auto-send (`run.source === "mailbox"`), and the cron `runAutoSend` sent but never called writeback (`performMailboxWriteback` was private in actions.ts, manual-send only). The system was "safe" only because mailbox auto-send was effectively off.

Fix, three parts:
- Removed the mailbox block → mailbox runs follow the same `result.autoSendEligible` gate as every source (confirmed it already reflects the tenant toggle + confidence threshold before removing).
- Extracted `performMailboxWriteback` → `lib/mailbox/writeback.ts` (exported), manual path re-points to it unchanged.
- Cron loads the mailbox + calls the extracted writeback after a successful send.

**Ordering = send-first** (send → `auto_sent` → file), chosen because send-then-file fails to a recoverable double-reply while file-then-send fails to a silent drop (customer gets nothing, filed as done) — the worse failure and the exact thing the app exists to prevent. Writeback failure non-fatal. Non-mailbox auto-sends no-op on the guard.

**VERIFIED end-to-end on prod (S199, manual mailbox send):** reply in Sent, original moved to INBOX/MoversApp Responded, DB status=sent/source=mailbox/resolved_by=operator. Proves the extracted writeback + manual path (which F28 re-plumbed, so this also clears the regression risk). **Auto-fire/cron path NOT verified — blocked by F29; rides with F29's fix.** The send→file "sliver" accepted for launch (Make tolerates it); F15 Problem B stays post-launch insurance.

### 3. S230 — restore tenant seed on creation + backfill. 78e0714, direct to main
Started as "seed 3 default calendars (Main/Vehicles/Team Hours) on new tenants — the seed hasn't carried forward." Discovery flipped the premise twice:
- The seeder (`seed_tenant_defaults`) DOES seed the 3 calendars, and it's on the creation path (called by `create_tenant_with_owner`). So not "never built."
- Live data showed the real pattern: every tenant ≤14 July has 3 calendars; every tenant from ~29 July on has 0. The seeder **worked then stopped.**
- The live `seed_tenant_defaults` source was intact + correct (right colours, idempotent). The `calendars` table schema matched the insert. So neither was broken.
- **Root cause: the LIVE `create_tenant_with_owner` had DRIFTED — it no longer called `seed_tenant_defaults`.** The repo's 10 July migration has the call; the deployed function didn't. No 14–29 July migration removed it → an out-of-band `CREATE OR REPLACE` on the live DB (the S130 disease). Calendars are the last block in the seeder, so they were the visible casualty while config/bands/boards mostly survived via other paths.

Fix (Option A — proper, durable, not another out-of-band change): committed `20260902133730_s230_restore_tenant_seed_call.sql` (CREATE OR REPLACE restoring the seeding version; signature verified identical) + ran live via MCP + backfilled every tenant <3 system calendars via the idempotent seeder in a loop. **Verified THREE ways:** backfill query (all tenants ≥3 now), live pg_proc contains the seed call, and a genuine new signup (Sep Movers) landed with 3 calendars + config + 4 bands + boards.

Colour-from-brand for calendars + onboarding brand capture were RAISED but correctly deferred (chicken/egg: no brand colour exists at seed time; brand capture is a post-launch feature).

### 4. S217 — CLOSED, not reproduced
Calendar-entries-not-appearing could not be reproduced on S199 (live prod URL, not preview): add → appears → persists on hard refresh → DB row confirmed; a new-calendar creation test was also clean. Original demo failure was a stale-preview view or mis-step. Dropped from launch gates per its own drop-out clause. (A mid-session false alarm — the operator briefly saw a 6-tab old-style Agents page and thought main was stale; it was a preview-branch URL, not prod. No bug.)

### 5. S218 — auto-send delay reassurance copy. b1adddc → 2f1818b → d1d42f0 → f6a5e4d
Investigation collapsed the scope: the delay is already per-tenant (`tenant_agents.auto_send_delay_seconds`, default 300s, 0–1800 slider), so no "inbox-only default" mechanism was needed — just copy. Final: one combined plain-language helper line under the delay slider, rendered whenever auto-send is on (`{autoSendEnabled === true}` — no mode/mailbox/consent dependency), "worker"/"tick" dropped, no em-dash. Verified live on S146.

Took 4 commits due to two build failures (see the standing rule below): raw JSX apostrophe, then a render-condition bug (copy didn't show), then the reword. Dismissibility / 30-day expire / tooltips raised but deferred to F27.

---

## New launch-blocker found: F29

While eyeballing the S218 copy, the operator hit a real bug: **a brand-new tenant cannot enable auto-send.** The generic-sender consent modal ("Use generic sending address?") accepts but loops — "No sender email configured" persists, toggle never sticks. `tenant_agents` has no sender/consent column; the auto-send gate reads consent/identity from other tables (F19's `tenant_users.ui_preferences` / `tenant_sending_identity`), which a fresh tenant has zero rows in. **This is the S201 hole, live.** Ties S208. **Also blocks F28's auto-fire verification.** Logged F29, LAUNCH-BLOCKING, own scoped session next.

---

## Decisions locked

- **DEC41** — (A) DEC35 is now actually delivered via F28 (was an untested assumption); (B) F23 3b-4 mode-switch teardown = accepted semantics, no build (the mode/door keying asymmetry is intentional per DEC40; cursor replay dedup-covered; dormant rows a manual queue by design).

## New backlog items

- **F29** — new-tenant auto-send consent/sender gate loops (LAUNCH-BLOCKING).
- **S231** — standing rule: JSX/import/hook briefs verify with `npm run build` (not just tsc+test).
- **S232** — desktop unsaved-changes toast to match mobile (cosmetic).
- **S233** — stale "Review Queue in this slice" inbound-mailbox helper copy (wrong post-F28).
- **S234** — persistent `AgentOnboardingFlow.tsx` useMemo missing-dep ESLint warning (tidy).

---

## Gotchas / process notes (also for quirks)

- **Codex's tsc+vitest does NOT run ESLint; Vercel's `next build` does.** THREE build failures this session (unused `Link` import, two `react/no-unescaped-entities` apostrophes) all passed tsc+test and only failed at deploy. This is why deploys "kept failing recently" — a verification gap. Standing rule S231: any JSX/import/hook brief verifies with `npm run build` before push.
- **Committed ≠ pushed.** S230 + S218 sat committed-locally-not-pushed for a while (F28 had deployed, so pushing was working earlier — these were just never pushed). Confirm push after each Codex ship, not just commit. Related: S230's DB fix was live via MCP before its migration file reached main — live DB and repo can be out of step.
- **Live DB functions can silently diverge from committed repo files** (S230). The repo migration had the seed call; the live function didn't. Neither the schema_migrations ledger nor the repo folder is a trusted mirror of the live DB — only querying the live DB directly is ground truth. Logged as an S130 amendment.
- **Preview-URL confusion** (twice): a `*-git-<branch>-*.vercel.app` URL runs that branch's code, which can look stale vs production (`moversapp.app`). Check the URL before diagnosing "the codebase is stale." One false alarm this session (6-tab Agents page).
- **The migration-apply path** is "commit .sql + run live via MCP `execute_sql`"; it does NOT write a schema_migrations ledger row. Consistent with the session's own convention; ledger reconciliation is S130.

## State at session end

- **main f6a5e4d.** All four items live or deploying; S218 render + reword verified on S146; F28 manual path + S230 verified on prod.
- **F23 fully closed.** Mailbox launch block core complete — but **F29 is now the live blocker** (new tenants can't enable auto-send; also gates F28 auto-fire verification).
- Two verify-tails outstanding: **F28 auto-fire** (needs F29 fixed first) and any final S218 toggle re-check.

## Next session

- **F29** (LAUNCH-BLOCKING) — scope the new-tenant auto-send gate. Trace the send-gate + consent-write + sending-identity provisioning; do NOT scope blind. Fixing it also unblocks the F28 auto-fire verification.
- Then the remaining Tier-1 cluster: F21 (marketing site, gated on billing), S191→S194 (failed-run home), the hardening batch.
