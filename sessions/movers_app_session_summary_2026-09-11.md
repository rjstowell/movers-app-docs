# Movers App — Session Summary 2026-09-11

*UI overhaul arc, part 1. Design-language + master-detail rebuild. Ran across the evening of 09-10 into the morning of 09-11; treated as one session, dated 09-11.*

## One-line

Reskinned the app to a cleaner design language, rebuilt the two densest config pages (Settings > Pricing, Agents > Setup) as a master-detail layout (desktop rail + mobile accordion), and caught + fixed a launch-blocking jobs-board seeding regression along the way.

## Shipped + merged to main

All UI work verified on preview and merged. Commits/merges:

- **Token + colour swap** — `c3f4d99` direct to main. Neutral palette drabness fixed (`--bg` `#f5f5f0` -> `#FAFBFD`, cooler border/text), default accent teal -> marketing blue `#1D5FD6`, added `--shadow-lift`. App already on Geist (no font change). Tailwind straggler sweep converted hardcoded neutrals to tokens.
- **Branding default-accent fallback fix** — `lib/tenant-branding.ts` teal `#0d9488` -> blue `#1D5FD6` (dark `#143F97`). ROOT CAUSE: the `:root` default only reaches non-tenant auth pages; every tenant page injects the accent inline via `app/(app)/layout.tsx` from `tenant-branding.ts`, whose hardcoded fallback was still teal. Tenants with empty `branding: {}` now show the blue default. Tenant custom accents unaffected.
- **Jobs-board seeding regression — FIXED** (see its own section below). Branch `fix/jobs-board-seed-regression`, `806c9bb`, migration `20260909210000_restore_typed_jobs_trigger_seed.sql`, merged.
- **F37 Pricing master-detail** — PR #106, merge `41eb3bb` (`ba44db9` build + `1a05f8f` Full-view removal). `PricingForm.tsx` -> `PricingWorkspace.tsx`.
- **F38 clean-styling, Settings + Agents** — PR #107, merge `96c4de2` (`c7284c5` styling pass across 51 files + `1763df7` `--input-fill` token). Field fill locked at `#F6F8FC`.
- **F37 Agents Setup master-detail** — PR #108, merge `172af9e` (5 commits: rebuild + Enabled-trap fix + Inbound-save + Agent-active instant-save + nav-guard/discard).
- **F37 mobile rail accordion** — PR #109, merge `9192dae` (`b13014f` accordion + `663c7aa` independent toggle / multi-open).

## Key decision — DEC51 (master-detail pattern)

**Master-detail is the layout pattern for dense config pages only, at a ~4+ heavy-section threshold.** Below that (Profile, General, Fleet table, Team, Billing, Jobs tab), pages keep their existing layout and get clean styling only. The app-wide layer is the **clean primitives**, not the rail: `.card-clean` / `.input-clean` / `.label-clean` / `--input-fill` / `--shadow-lift` + the warning tokens (`--bg-warning` / `--border-warning` / `--text-warning`, `.banner-warning` / `.text-warning` / `.pill-warning`) + `.table-clean`.

Pattern specifics:
- Desktop: left rail of section bars (with status dots), one section open at a time, pane swaps.
- Mobile: accordion — bars stack full-width, tap to toggle, **multiple open at once**, independent close.
- **No "Full" view.** Tried on Pricing, dropped — Clean-only. (This killed the view-mode persistence requirement entirely.)
- Dots are static/neutral for now; real pending/done state is the separate setup-completeness thread.

## Load-bearing invariants (do not break in future work)

1. **Save guarantee.** Both Pricing and Agents Setup are single-form pages where the action reads the whole FormData and falls back to defaults for anything missing. So closed sections must stay **MOUNTED**, never unmounted — desktop hides via `display:none` (`data-closed`), mobile via a `1fr->0fr` grid-row + `visibility:hidden` on the inner. Closed inputs still submit. Verified via a DOM harness across all mode x section combos (FormData keys identical).
2. **The Enabled/"Agent active" trap.** `enabled` MUST remain a hidden `<input name="enabled">` inside the overview form. If it leaves the form, `updateAgentOverview` reads `enabled=false` and silently disables the agent, AND clamps `confidence_threshold` to 0.5 (not the 0.85 fallback — `Number(null)->0->isFinite` is true), AND nulls `escalation_whatsapp` / `sender_display_name`. Agent Active now instant-saves via a targeted `setAgentEnabled` action (mirrors `saveAutoSend`), but the hidden input stays synced so a later form Save still reads the right value.
3. **Nav-away guard.** Rail section-switching now routes through the existing `useUnsavedChangesGuard` / `requestLeave` convention (same in-app modal used elsewhere, NOT the browser dialog). Discard genuinely reverts fields to stored values (`discardUnsavedEdits`). Guard covers overview form + signature + ignored-domains dirtiness. **Mobile accordion collapse does NOT route through the guard** — collapsing isn't navigation and would silently eat edits.
4. **Deep-link hashes** (`#email-signature`, `#mailbox-card`, `#sending-identity`) open their owning section via a synthetic `hashchange` re-dispatch. The shared `useHashHighlight` hook was left untouched on purpose (4 other surfaces depend on it) — the re-dispatch is the contained workaround.

## Jobs-board seeding regression (launch bug, caught + fixed)

Spotted when a new tenant (0809 Movers) showed one Jobs board with `board_type = NULL` and generic lists (New Lead..Paid) instead of the correct 5 typed boards. DB investigation:
- **Correct set** (tenants 2026-07-06..07-14): 5 typed boards — Jobs / Leads / Quotes / Completed / Archive.
- **Broken set** (2026-07-29 onward, 9 tenants, all test): 1 board, `board_type NULL`.
- **Cause:** an `AFTER INSERT` trigger `tenants_seed_jobs` runs the LEGACY `jobs_seed_tenant()` (single null-type board), which fires first and makes a board, so `seed_tenant_defaults` (guarded "only if no boards exist") then skips. Live schema drift ~07-29 replaced the trigger fn with the old 07-07 definition after the typed 07-08 version existed. S230's 09-02 seed-restore checked board *existence*, not correctness — false comfort.
- **Fix:** migration `20260909210000` points `jobs_seed_tenant()` at `jobs_seed_for_tenant()` (which creates all 5 typed boards + current-month buckets). Verified: fresh tenant gets 5 boards, 0 null types.
- **Not backfilled:** the 9 broken are all test tenants, being detonated soon; no customer impact. If ever backfilled, must delete the null board first (`jobs_seed_for_tenant` has no existing-board guard, would add 5 alongside the junk one).
- **Fragility flag:** two separate board-seed paths (trigger + `seed_tenant_defaults`) is what let this happen. Worth consolidating later so it can't recur.

## New backlog items raised (S273–S280)

- **S273** — Granular agent resets. Rail "Reset" is a single full-nuke; offer individual resets (clear templates / rules / categories) alongside. New actions to build, not copy. OPEN.
- **S274** — `FeedbackWidget.tsx:438` references `--surface-card` which doesn't exist in globals.css, silently falls back to `#fff`. Dangling token. OPEN.
- **S275** — `.panel-clean` primitive for the ~30 recessed inner panels the clean-styling pass left alone (converting them to `.card-clean` would make wells white-on-white). OPEN.
- **S276** — Ignored-domains "Add" should commit directly; currently Add stages, then a separate Save commits (two-step). OPEN.
- **S277** — Escalation WhatsApp: currently a disabled "Coming soon" placeholder (feature not built; nothing consumes `escalation_whatsapp` — confirmed via `generateClassifierPrompt.ts` which declares but never reads it). Build the real feature. DEFERRED/OPEN.
- **S278** — `isRunningInitialSetup` ("Setting up…" banner) is near-dead code now that enabling left the form (keys off `enabled && !savedEnabled` at submit). Verify nothing else triggers it (e.g. fresh provisioning) then remove or restore. OPEN.
- **S279** — Pricing Distance-bands table starts internal scroll ~960px viewport instead of ~720px because the rail eats 250px. Pre-existing, minor. OPEN.
- **S280** — `0.5px` hairline borders render as 1px on 1x (non-retina) monitors; only truly hairline on scaled/retina. Inherited from the primitives. Cosmetic note. OPEN.

## Remaining UI arc — RANKED (next session picks from the top)

*Not pre-numbered — assign each an F/S number from the active backlog's next-free when it's picked up, per numbering discipline.*

1. **Inbox -> list/detail (email-client layout).** Biggest daily-use surface, currently heavy scroll-through cards. All sub-tabs (Review Queue, Escalated, Filtered, + unbuilt Sent). Feature-scale.
2. **Clean-styling roll-out to the remaining pages** (Dashboard, Quote, Jobs, Calendar, Vehicles, Reports). Mechanical sweep applying the F38 primitives page-by-page (NOT a blind regex — card roots vary, some tables have no card class; report stragglers). This is F38 continued.
3. **Quick wins** (small, visible, low-risk): Dashboard card width (too wide, wasted middle space) · Jobs list colour picker (3-dot menu -> faded per-list background) · Reports "coming soon" upgrade + visual preview.
4. **Agents tab reordering** — Setup first until setup complete, then drops to 4th; Inbox promotes. Depends on the setup-completeness signal existing.
5. **Calendar restyle + hover-to-add-event** (plus resolving the click-day-vs-add conflict). Real work.
6. **Motion / flow polish** — attention glow on cards needing attention, general dynamism. Finishing layer.
7. **Mobile sweep** — the rest of the app at phone width (deferred all session; only the master-detail pages got mobile attention).

**Two spun-off threads (feature-scale, own sessions):**
- **Setup-completeness signal** — one "is this area configured?" model that powers: the master-detail bar dots (pending/done), F9 (dashboard setup checklist widget, already in backlog), and the tour's targeting. Build once, don't triplicate.
- **Post-onboarding tour** — guided "take a tour" animated walkthrough pointing at unconfigured areas. Needs the completeness signal to exist first.

## Numbering after this session

- **F37 DONE** (master-detail: Pricing / Agents Setup / mobile accordion slices).
- **F38** — clean-styling: Settings + Agents DONE; remaining pages OPEN.
- **DEC51 LOCKED** (master-detail pattern + clean primitives as app-wide layer).
- New S: **S273–S280** (tails above).
- **Next free: S281, F39, DEC52.** Check the active backlog before assigning; do not infer from chat.

## Launch plan

No launch-plan change made this session (UI work is pre-launch polish, not a gate). One thing worth a one-line touch if desired: the jobs-board seeding regression was launch-relevant (every new signup got a broken Jobs page) and is now fixed — worth noting under the relevant launch-plan section so it's not re-flagged.
