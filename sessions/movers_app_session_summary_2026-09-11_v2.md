# Movers App — Session Summary 2026-09-11 (v2)

*UI overhaul arc, part 2. Inbox rebuild + reskin rollout. Same calendar day as part 1; logged separately.*

## One-line

Rebuilt the Agents Inbox (Review Queue, Escalated, Filtered) into an email-client list/detail layout, rolled the locked clean primitives across the remaining pages, and added the panel-clean recessed primitive. Three merges to main.

## Shipped + merged

- **F39 Inbox list/detail** (PR #110, squash `e542719`). Review Queue, Escalated, Filtered converted from stacked scroll-cards to a shared list/detail shell. Built in slices on one branch, merged whole.
- **Clean-styling rollout** (PR #111, squash `d3d857b`). 6 pages (Dashboard, Quote, Jobs, Calendar, Vehicles, Reports) got the DEC51 clean-primitive class swaps. 20 files, +84/-84, no CSS, no shared components.
- **panel-clean primitive + S274 fix** (PR #112, squash `0e7e07f`). New recessed-panel primitive (DEC52) + 18 panels converted + S274 dangling token fixed. globals.css +10, rest .tsx swaps.

## F39 — the Inbox rebuild (detail)

Shared shell `InboxListDetail.tsx` (list column + reading pane on desktop; full-width list, tap-to-detail, back on mobile). Per-folder row content + action sets plug in.

- Desktop folder switcher = pills; mobile = labelled menu button (not a hamburger: the app already has one for app nav, and the label doubles as "you are here").
- Reading pane stacked: customer email on top, draft/action below.
- Load-bearing invariant preserved: switching rows with an unsaved draft/compose fires the discard/keep guard (the DEC51 nav-guard pattern applied to row selection). The existing page-level unsaved guard + realtime dirty-preserve in AgentsTabs were left untouched.
- Review Queue: all 5 row kinds (error / captured_locked / manual_hold / drafted / resolved), 6 modals, FeedbackWidget, live countdown preserved. Display toggles reduced to one ("Show confidence on list"); template moved to a muted pill; the "Why this category" reasoning line now shown (it was loaded but never displayed before).
- Escalated: reply composer, Mark done/Undo, Show done, plus a Date / Needs-action sort added. Filtered: Recover / Send to Escalated / Mark handled; banner + status message relocated folder-level; status message auto-fades (~6s).
- Shared `inboxRow.tsx` + `InboxSort.tsx` extracted (dedupe across folders).
- Deep links (`#review-queue-row-<id>`, `#escalated-run-<id>`, `#filtered-run-<id>`) fixed to select the linked row, since only the selected row mounts now.
- Sent tab NOT built (still F35).
- Em/en dashes removed from Inbox copy pre-merge (house style).

## panel-clean (S275 design closed, DEC52 locked)

Recessed-well primitive: `var(--bg)` fill, 1px `var(--border)`, 10px radius, no shadow (depth option B, chosen from a 3-option mock). Codifies the inline pattern already in use, does not restyle it. Applied to 18 panels (Inbox "Customer's email" wells, Vehicles service box + log rows, Agents/Settings panels). S274 (`--surface-card` dangling ref in FeedbackWidget) resolved to `var(--card)`, since it was a modal surface, not a recessed panel; the dead token was removed.

## Clean-styling rollout — low visible change, and why

The 6-page sweep was near-invisible: the app was already close to the primitives, so `surface-card` to `card-clean` etc changed class names, not pixels. The visible differences live in the spots the sweep PARKED: recessed wells, status colours, banners on Dashboard and Jobs. Those are blocked because `dashboard.css` / `jobs.css` / `quote.css` load after `globals.css` and win, so the primitives cannot reach them. That is the wall.

## Open decision — top of next session

**Should the reskin be allowed to edit the page stylesheets (`dashboard.css` / `jobs.css` / `quote.css`)?** Hit four times now (clean-styling sweep, then panel-clean). Until decided, Dashboard and Jobs recessed panels / status colours / cards cannot be finished; they stay half-reskinned. Options: allow page-CSS edits (per-page, discovery-first, start with the cheap `jobs.css .quote-panel` one-liner); migrate those stylesheets onto the primitives properly (bigger, closer to rebuilding them); or leave as-is (app never fully consistent). Candidate DEC53 once decided.

## Investigation logged (not decided) — category vs template

Operator questioned whether categories and templates are redundant (they look 1:1). Finding: NOT 1:1 by design. `template.category_id` is a non-unique FK, and a routing layer exists (`quote_route` local rule + `route_key` on essential fields). Cornwall is merely configured 1:1. Operator's own case for keeping the split: a planned second-line-enquiry agent (reply after gathering quote info) is a genuine one-category-two-templates case (info-gather reply vs quote-delivery reply). Confirming needs a read of the drafting-path code. Not F39-related.

## Quirk logged

Claude Code file-summary cards report a throwaway harness page (`page.tsx`) as a real edit whenever it spins up a visual test harness. Seen 4x this session (+173 / +178 / +191 / +174, all -0). NOT every run (the clean-styling sweep launched no harness and its card was accurate). Trust `git --no-pager diff --stat origin/main...HEAD`, not the card. First hit cost a full detour. Also reconfirmed: diff against `origin/main...HEAD`, not `main...HEAD` (stale local main showed already-merged F39 files as new).

## New backlog items

S281 spaced-hyphen copy cleanup. S282 Filtered Recover optimistic UI (15-20s wait). S283 Filtered Show-handled server-side fetch. S284 dangling CSS token/class audit. S285 prune merged branches. S286 list_completeness leak in reasoning text. S287 dead pref columns (showTemplate/showReasoning). F40 Dashboard inbound-address surface. F41 Inbox fetch-cap / paging. F42 Inbox search. F43 Quote Engine tab rebuild. DEC52 panel-clean (locked).

## Method notes

- Every "shipped" claim verified via `git --no-pager diff --stat origin/main...HEAD` before trusting the agent's card (caught the phantom page.tsx 4x).
- All three merges squash-merged, draft PR until live-verified, CI auto-fix left OFF throughout (hand-review each commit).
- F39 verified with real emails through Testing Mover (real escalated reply sent, real Recover, real deep-link); reskin + panel-clean eyeballed on the branch preview.

## Numbering after this session

Next free S = **S288**. Next free F = **F44**. Next free DEC = **DEC53**.
