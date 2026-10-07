# Movers App — Session Summary 2026-08-28 (v3)

**F23 slice 2 — inbound routing respects mode. Shipped + verified end-to-end on prod.**
main `e34791e` → `ec26a51` (2 PRs: #75 gate, #76 `letterbox_dormant` status). One migration, applied to prod via Supabase MCP. One data fix, also via MCP.

---

## What shipped

**Slice 2 = one inbound door per tenant, derived (not chosen).**

- **#75 — the door gate.** New `lib/mailbox/inboundDoor.ts` → `getInboundDoor(admin, tenantId)`: returns `'mailbox'` if the tenant has any `tenant_mailboxes` row with `status='ok'`, else `'letterbox'`. Fail-open to letterbox on query error (dropping mail silently is the worse failure). Wired into the Resend webhook (`app/api/agents/inbound/route.ts`, `recordAndProcessInbound`) right after the `tenant_agents` match, before body fetch: if the door is `mailbox`, record the run and return without processing. 3/3 vitest on the helper.
- **#76 — distinct status.** Wrong-door mail was first written as `status:'ignored'` + `classification.ignore_reason:'letterbox_dormant'`. That conflates it with the classifier's real `ignore` category (spam, invoices, one-word “Testing”). Added **`letterbox_dormant`** to the `agent_runs_status_check` constraint and switched the gate to write it directly (no classification payload). Migration re-tagged the one existing test row.

**Mailbox sync: no code change.** The door is defined as “has an `ok` mailbox row”, and the cron/S223 path already only produces runs for connected mailboxes — so there was nothing to gate there. Stated explicitly in the #75 PR so it doesn't read as an omission.

---

## Data fix (pre-merge, via MCP)

S146 and S199 both held `hello@cornwallmovers.co.uk` — the same message ingested into two tenants (the exact DEC36 cross-tenant duplicate). Deleted S146's row. S199 keeps it (it has the writeback folders). S146 now has no mailbox → letterbox door → becomes the slice-3 “switch to mailbox via toggle” test case.

Post-fix `tenant_mailboxes`: 2 rows, no duplicate address — S199 (`hello@cornwallmovers.co.uk`), Joe's (`testing2026@mwbv.co.uk`).

---

## Verification — all on prod, real emails

1. **Testing Mover** (no mailbox → letterbox door) letterbox → processed normally, `received` → `escalated` (multi_match: removals + quote-with-no-addresses — correct classifier behaviour, not a gate effect). Proves the open door still works.
2. **S199** (mailbox → letterbox dormant) letterbox → `ignored`+`letterbox_dormant` (pre-#76), immediate, no body fetch, no draft, no Review Queue row. Proves the gate fires and short-circuits.
3. **`hello@cornwallmovers.co.uk`** + Sync now on S199 → ONE run, S199 only, `source:mailbox`, `drafted`. No S146 row (its mailbox is gone). Proves no cross-tenant dup.
4. **Post-migration re-send** to S199's letterbox → new row `status:'letterbox_dormant'`, `classification:null`, body empty. Proves the DEPLOYED gate (not just the migration) writes the new status clean.

Migration self-checks: constraint includes `letterbox_dormant` = true; stale rows (`ignored` + reason) = 0; S199 test row flipped.

---

## The design correction (the session's main thinking)

The build surfaced that **two concepts were fused in DEC36**: *where mail enters* and *where the tenant manages*. Long back-and-forth untangled them into **DEC37**:

- **Door** = where enquiries enter the pipeline. Derived from mailbox existence, not a preference. Mailbox verified → mailbox door, letterbox dormant. No mailbox → letterbox door. One-directional: only the letterbox is ever blocked, only when a mailbox is live.
- **Toggle (`inbound_mode`)** = where the tenant manages (app vs own inbox). The real choice — but it only bites once auto-send is decided. With auto-send OFF, drafts live only in the app, so approval is in the app regardless. “Mailbox-managed” = mailbox door + auto-send on; not a separate stored surface.

Decisions inside DEC37 worth not relitigating:

- **We do NOT auto-flip auto-send when a mailbox connects.** Considered (“mailbox position nudges auto-send on”), rejected — can't make that call for the tenant. Mailbox toggle position shows a one-line “drafts approve in the app; turn on auto-send to work fully from your inbox” note instead.
- **Data lives in both places even when managed from one.** Responsibility for the data shouldn't sit solely with the app. App-managed tenants still get a full copy in their own mailbox (→ **S226** Sent-append + writeback filing); mailbox-managed have the full record in the app.
- **The letterbox is NOT redundant post-mailbox-sync.** It's the door for tenants we can't read yet (Gmail/Outlook, pending F15 OAuth) or who won't share creds — and a selling point (run with no external mailbox at all). Mailbox door is a superset for the syncable; letterbox is the fallback + no-mailbox path.
- **`app_only`/`inbox_only` is now a slight misnomer** (`app_only` really = “no mailbox / letterbox door”). Rename deferred.
- **Option C (re-route dormant-letterbox mail into the mailbox pipeline) REJECTED.** It can't distinguish “only copy” from “duplicate of what's also in the mailbox”, so it re-opens the DEC36 double-reply. The fix is visibility (S224), not re-routing.

---

## The known gap (by design)

A real move enquiry sent to S199's letterbox became `letterbox_dormant` and **appeared on no screen** — because the view that shows filtered/non-drafted runs (**S224**) doesn't exist yet; it's slice 3. This is not a slice-2 bug: slice 2's job was the gate (stop the dup, record recoverably), and surfacing was always S224/slice-3 scope.

**Operator call: this must be fixed before launch.** Wrong-door mail is recorded but invisible until S224, so a mis-pointed form would drop enquiries silently. Only inbox_only tenant today is S199 (test), so the risk window is empty — keep it empty by shipping S224 before any real inbox_only tenant onboards. Recorded as a pre-launch gate on both the F23 and S224 backlog entries.

---

## New items

- **S226 — Sent-folder append after send (mailbox-connected tenants).** IMAP-append the outgoing reply into the tenant's `\Sent` so a full copy lives in their own mailbox. **Replaces the S190 BCC**, which drops to a no-mailbox-only fallback. The “data in both places” half of DEC37. OPEN.
- **S227 — provider pre-check at mailbox connect.** Detect Gmail/Outlook and warn “not supported yet” BEFORE the tenant enters creds, so they don't set up something that can't sync. Ties to F15 OAuth adapters. OPEN.

## Decisions

- **DEC37 LOCKED** — inbound door derived from mailbox status; toggle = management surface; refines DEC36. Full text in the backlog DEC block.

## Flagged, no number change

- **F15 OAuth adapters to pull forward (post-slice-4)** so Gmail/Outlook tenants become syncable. Existing F15 scope, not a new item.

---

## Slice 3 (next, FRESH chat)

Bigger than 1 or 2 and a different kind of work — first slice with real tenant-facing UI. Three distinct pieces, expect sub-slicing 3a/3b/3c:

- the app/mailbox management toggle + per-position config gating (+ “connect a mailbox first” prompt, + the auto-send note)
- the **S224** filtered-runs view (Review-Queue/Escalated sibling) surfacing `letterbox_dormant` + `ignored`/no-queue-row runs, with a recover/reply action and a “your form points at the wrong address” nudge — **pre-launch gate**
- onboarding/survey wiring so the form points at the right door per mode (root-cause fix for wrong-door mail)

First move when it opens: **step-0 the settings UI + where the toggle plugs in.** Wrong headspace to start in this chat's slice-2 backend context.

---

## Process notes

- **Step-0 investigate gate earned its keep again.** It surfaced (a) no clean “verified” discriminator — `replied_folder`/`escalated_folder` only populate on first send, so “folders exist” would deadlock a new inbox tenant; `status='ok'` is the only usable gate; and (b) the S146/S199 shared-mailbox duplicate, caught before it shipped.
- **The classify pipeline is ~18s received→drafted.** A run on `received` for <20s is normal, not stuck. `letterbox_dormant` by contrast is immediate (no pipeline) — a useful tell that the short-circuit fired.
- **Codex ran low on session quota mid-build; flipped to Claude Code for the merge/deploy of #76.** Same brief format worked unchanged.
- **One out-of-brief edit on #76** was checked in the diff before merge: a `console.error` string `ignored`→`letterbox_dormant`. Internal log only, not tenant-facing. Clean.
- **Migration applied via MCP, not by the deploy.** Vercel deploy ships code; the constraint change is a separate `apply_migration` step against prod. Don't assume merge = migration live.
