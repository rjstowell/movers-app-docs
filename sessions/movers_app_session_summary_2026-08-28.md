# Movers App — Session Summary 2026-08-28

**main 5258490 → aed352f (1 PR: #72). F15 writeback slice COMPLETE. F25 escalation writeback shipped + verified A–D on prod.**

Cold-start job was the F15 Unit 3 "open bug" (provisioning + Responded-move silently no-op on S199). It did not reproduce — the slice was actually working. Session then closed the second writeback direction as **F25** (escalated-feed move), diagnosed a preview-vs-prod trap that hid it, and verified end-to-end on production.

## F15 writeback — Unit 3 verified, slice DONE

- Shadow Approve+Send on S199 (watch_folder `INBOX.Shadow`) created **both** folders (`MoversApp Responded` + `MoversApp Escalated`), moved the enquiry to Responded, and the move landed. No error.
- **DB confirmed** the backfill executed: `tenant_mailboxes` for S199 has `replied_folder = INBOX.MoversApp Responded`, `escalated_folder = INBOX.MoversApp Escalated` (both were null pre-session). Delimiter `.` discovered per-server, `INBOX` prefix correct.
- The three earlier acceptance criteria (folders create, move to Responded, columns populate) all green.

### `\Answered` — set in code, display gap only
- The moved message showed no answered indicator in Hostinger webmail. Investigated: `\Answered` is a standard IMAP system flag (RFC 3501), stored by any compliant server. Code DOES set it — `moveSelectedMessage` (imap.ts) fires `messageFlagsAdd(uid, ["\Answered"])` before the move; actions.ts passes `markAnswered:true` at the Responded call (confirmed line 2085). So the flag is on the message; **Hostinger webmail just doesn't render an answered glyph.**
- Verified from code, not webmail appearance. Product stance: folder membership is the responded signal; the flag is a nice-to-have. Logged as a quirk (§6) so it isn't re-chased.

### Cleanup
- Deleted the three smoke-script residue folders (`App Responded` / `App Escalated` / `App Review`) — leftovers from the Unit 1 throwaway test, unrelated to the live "MoversApp"-prefixed flow. Make's `Charlie Replied` / `Harlene's feed` left alone.

## F25 — escalation writeback (#72, aed352f)

**Goal:** when the live pipeline escalates a `source='mailbox'` run, move the original from the watch folder to the tenant's `escalated_folder` (model A's needs-human pile), no `\Answered`.

**Design fork resolved → option A (fire at escalation time, automatic, in the pipeline).** Rejected move-at-resolve: the folder must read as "needs attention," not "resolved." Walked A against failure/repeat/better-state — own catch never throws (escalation record safe), absent = no-op on re-run, already-moved = absent no-op. Holds.

**Required a refactor first.** The folder-ensure helper (`ensureTenantWritebackFolders` + `buildMailboxConfig` + `resolveInboxHierarchyDelimiter`) lived inside the `"use server"` actions file, which a lib module (pipeline.ts) cannot import. Extracted to new **`lib/mailbox/writeback.ts`** as one source of truth; actions.ts imports it back (Responded path byte-identical). Chose clean/F25 (branch + PR) over duplicating the ~15 lines, to avoid a split-brain on folder names/delimiter later.

**Changes (3 files, +182 −104):**
- `lib/mailbox/writeback.ts` (new, +138) — moved helpers + new `performEscalationWriteback(admin, run)`: guards non-mailbox / null message_id, loads the tenant mailbox, self-heals `escalated_folder` via ensure if null, moves by `message_id`, no `markAnswered`, own try/catch (logs `[agents/pipeline]` step, never throws up), absent = silent no-op.
- `lib/agents/pipeline.ts` (+16) — both escalate branches (`thread_reply` + `escalated`) call `await performEscalationWriteback(admin, run)` after status is committed.
- `app/(app)/agents/actions.ts` (−63 net) — helpers now imported from the new module; Responded behaviour unchanged.

Build passed (only pre-existing `useMemo` warning in AgentOnboardingFlow.tsx). PR #72, squash-merged as aed352f.

## The preview-vs-prod trap (cost the session's real diagnosis time)

First escalate test: run escalated in-app but **did not move** — sat in Shadow. DB ruled out the easy causes (source=mailbox, message_id populated, escalated_folder non-null, only one mailbox row, code path clean to the move). The pasted branch code was correct (verified local HEAD == branch SHA `58cdab2`, clean tree).

**Root cause: where the code ran, not the code.**
- Path A (Responded) worked because Approve+Send is a **user action** — ran on the branch-preview deploy (has F25).
- Path B (Escalated) runs inside the mailbox-sync **cron**, and **Vercel crons only fire on the production deploy** — which didn't have F25 yet. The sync that processed the thread-reply ran *old main's* pipeline (no hook). The escalated row showed in the preview UI only because both deploys read the same DB.
- Evidence: branch-preview logs showed page loads / `system-health` but **no `/api/mailbox/sync`**; the earlier sync host was the `movers-<hash>` prod deployment.

**Consequence:** a cron-path feature can't be verified on a preview deploy. Chose merge-then-verify-on-prod (A's regression had already passed; preview physically can't exercise the cron). Logged as a quirk (§7) and raised **S223** (manual "Sync now" trigger) to remove the gap permanently.

## Verification A–D (on prod, aed352f)

- **A — Responded regression:** unbroken. Reply in Sent, original in `MoversApp Responded`, columns intact.
- **B — escalate move:** thread-reply into the mailbox → next cron tick → original moved `Shadow → MoversApp Escalated`, no `\Answered`, shows in escalated feed. Moved before the operator even checked.
- **C — idempotent:** over an hour of cron ticks, single copy, `error` null, no duplicate run. Sync cursor never re-fetches; absent no-op if re-invoked.
- **D — DB:** escalated run `a2138c0a` (subject "Re: Move from Hayle"), status `escalated`, `error` null, reason `thread_reply`. Second live escalation (`ecae42b2`, the Jo Dambok thread) also clean. Two-for-two.

**Both writeback directions now live:** Responded move on send, Escalated move on escalation.

One harmless leftover: the 09:14 "Re: Moving from Penstraze into Truro" escalation ran pre-F25 on old main, so it never moved — test residue in Shadow, ignore or drag by hand.

## New items
- **S223** — in-app manual "Sync now" mailbox trigger (owner-only, same code path as the cron, operator-fired). Closes the can't-test-cron-on-preview gap for all future mailbox work. OPEN.

## Numbering
Next free S = **S224**. Next free F = **F26**. Next free decision = **DEC37**.

## Next session
**F23** (single-mode inbound system, LAUNCH-BLOCKING) — its F15-writeback dependency is now cleared. Fold in **S43 step 3** (INBOUND_EMAIL_DOMAIN flip + tenant-row UPDATE) where the address model is decided. Consider **S223** early if more cron-path testing is coming.
