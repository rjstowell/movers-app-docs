# Movers App — Session Summary 2026-08-31 v2

**F23 SLICE 3a-2 (recover-dormant half) COMPLETE — Recover action live on the Filtered tab. DEC38 LOCKED.**

main advanced via **PR #78** (squash `5e9100a`). No migration.

> Second session dated 2026-08-31 (system date). First was slice 3a-1 (the Filtered tab); this is 3a-2 part 1 (the Recover action). Marked "v2" to disambiguate same-day.

---

## What shipped

A **Recover** action on `letterbox_dormant` rows in the Filtered tab. It re-runs the existing pipeline on the stored row, tells the operator the outcome inline, and warns beforehand when the same email was likely already handled via the tenant's mailbox.

Files (PR #78): `actions.ts`, `page.tsx`, `AgentsTabs.tsx`, `FilteredTab.tsx`, `types.ts`.

Pieces:
- **`recoverFilteredRun(runId)` server action** — canEdit (owner+admin) gate; rejects non-`letterbox_dormant` rows; rejects null-body rows (returns `no_body`); calls `processInboundRun`; re-reads and returns the resulting status.
- **Recover button** on dormant rows only, **disabled on click** (blocks a double-fire producing two drafts).
- **Outcome → toast map:** `duplicate` → "Already handled via your mailbox — no reply drafted"; `drafted`/`auto_send_scheduled` → "Recovered — draft is in your Review Queue"; `escalated` → Escalated; `no_body` → can't recover.
- **Amber "Likely already handled via your mailbox" badge** — computed in the Filtered fetch: a dormant row that has a `source='mailbox'` sibling, same tenant, same `message_id`. Badge only; does **not** hide or block Recover (auto-hide could bury a genuine second enquiry — the DEC37 "C rejected" principle at the action layer).

---

## Step-0 findings that shaped the build (read-only, before any code)

Investigated the DB (via MCP) and three code files (`route.ts`, `pipeline.ts`, `loopPrevention.ts`) before building.

1. **`processInboundRun(admin, runId)` is row-shaped and has NO status guard.** It loads the `agent_runs` row fresh by id and runs. So Recover REUSES it as-is on a stored dormant row — no new reprocess endpoint, no payload reconstruction. Recovered rows change status and leave the Filtered filter set (`status IN ('ignored','letterbox_dormant')`) on their own.
2. **The double-reply guard was ALREADY LIVE.** `processInboundRun`'s first step is `isDuplicateMessage(tenant, message_id)` (`lib/agents/loopPrevention.ts`), which matches across ALL sources with no source filter. So a dormant ghost whose `message_id` equals a `mailbox` sibling self-resolves to `duplicate`, no draft. The slice added pre-click **visibility** (the badge) on top; it did not build the dedup.
3. **The pipeline trusts `run.body_text` — it never re-fetches the body.** So a null-body dormant row (the old pre-3a-1 rows) would garbage-classify. Recover is gated on body-present.
4. **`message_id` is populated on 100% of rows, both doors, same RFC822 namespace** (real Gmail/Outlook/domain-native ids). So exact cross-door match is viable. Dormant rows are always `source='resend_inbound'`; their only possible sibling is `source='mailbox'`.
5. **Gotcha (deferred fuzzy path):** `from_address` format differs by door — `mailbox` stores `"Name" <addr>`, `resend_inbound` stores bare `addr`. Any future from+subject fuzzy match must normalise to the bare address or it silently never fires.

---

## Verified end-to-end on prod (real emails, S199)

- Dormant row with body → **Recover** → Review Queue draft, row leaves Filtered (~30s, the pipeline runs synchronously).
- **Double-click** → button disabled after the first click, only one draft.
- **Twin test** (the fiddly one): manufactured a `source='mailbox'` row with the SAME `message_id` as the "Moving house"/Hos dormant row → the dormant row showed the amber "likely already handled" badge, AND Recover returned `duplicate` with the "already handled" toast and no draft. Test twin then deleted; the Hos row remains `duplicate` (genuine recovered state).
- Preview-verified before merge: Recover shows on dormant rows only (correctly absent on null-body rows), badge renders on the twinned row.

---

## Decisions

- **DEC38 LOCKED** — recover relies on the pipeline's existing `message_id` dedup to prevent double-reply; the UI badges the likely-handled case pre-click; fuzzy from+subject match deferred. See backlog DEC log for the full text.

## New backlog item

- **S228** — Recover UX: the synchronous ~30s pipeline wait feels dead. Fix = optimistic feedback (move the row out immediately + "Recovering — draft will appear in your Review Queue shortly", keep the button disabled, surface the rare failure via System Health). Post-launch polish; the shipped behaviour is correct, this is feel only.

## Still open / next

- **F23 slice 3a-2 IGNORED-OVERRIDE HALF (separate brief):** re-running an `ignored` row just re-ignores (same body/classifier/settings), and an ignored run has no category → no template to draft from. So "override" realistically = escalate-to-queue for manual handling, NOT force-draft — different plumbing (lands in Escalated, not a draft). Scoped out of this slice deliberately.
- Then **3b** (toggle + config gating + S218 delay note), **3c** (onboarding/form-pointing copy).

## Process notes

- **GitHub connector attempted this session but did not load into the chat** — code files were still pasted manually. Try a fresh chat next session; if the GitHub tools appear, code reads happen without the paste dance.
- Verify-after-merge was correct again here: the Recover *processing* only runs on prod (pipeline path), so the frontend pieces were preview-verified and the processing was merged-then-verified on prod.
