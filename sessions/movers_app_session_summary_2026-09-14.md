# Movers App — Session Summary 2026-09-14

*UI overhaul arc, part 3. Reskin completion + layout. Six merges to main, no migrations.*

## One-line

Finished the page-CSS reskin on the two pages the 09-11 arc had left half-done (Jobs + Dashboard), locked the page-CSS decision that was blocking them (DEC53), then reskinned the remaining daily surfaces: a styled Reports coming-soon page, a full Calendar restyle, and a new Dashboard layout. Arrived at a design principle for sparse pages along the way (DEC54 candidate: hand-placed layouts, not masonry).

## Shipped + merged to main

Six merges, all draft-PR-then-verified-on-preview, no migrations anywhere.

- **F44 — page-CSS reskin, Jobs + Dashboard.** Two PRs on one arc under DEC53.
  - Track A (PR #113, `e010afa`): the no-new-token subset. Dashboard widgets regained the lift shadow (removed the `.dashboard-card` override), health/inbox/address rows + Jobs Quote panel + timeline list became real `.panel-clean` wells, drawer-backdrop duplicate dropped, delete-button bg to `--card`. Job-card rest shadow was tried then reverted (it inverted prod's flat-rest/hover-shadow behaviour and read heavy); card-stack padding reverted with it.
  - Track B (PR #114, `96763a2`): the token work. Added the danger/success/info/neutral status-tone families to globals, converged all amber on the canonical `#92400e` (retinted the warning tokens, hit 13 files app-wide), consumed the tokens on every dashboard/jobs status spot, added a `.bar-status` full-bleed variant, added a solid `--fill-danger`/`--on-danger` button token, moved the Jobs kanban lane to `--input-fill`, and pointed the three Jobs menus at the popover surface (copied from `.tenant-switcher-menu`, no modal primitive exists). Sending-identity banner hit a STOP (its two-line layout does not fit `.bar-status`), so it took the amber tokens only and kept its own shape.

- **F45 — Reports coming-soon.** PR #115 (`9cd3571`) + PR #116 (`7974ee5`). Replaced the text placeholder with a styled preview: a "what reports will cover" card (4 tiles) + a "Preview" card (3 metric tiles, a brand-coloured booked-value bar chart, a quotes-outcome breakdown), triple-labelled as sample data (pill + footnote + aria-label). Added a "Coming soon" nav pill in the Dashboard-badge slot, neutral tone, both desktop rail and mobile drawer. Then subtle hover polish (tile lift + bar darken, reduced-motion respected, no fake-value tooltips). No data engine, all mock, no dependency.

- **F46 — Calendar restyle.** PR #117 (`84a660a`). Own code, no library, so a restyle not a rebuild. Fixed the fill-height problem (a flex height chain replacing the fixed 706px card, so the grid fills the viewport, 112px is now the row minimum not the fixed height). Variable week count (4/5/6 rows, only the weeks the month needs, no dead trailing row). Timed events render as dot+text on desktop, solid blocks on mobile (mobile needed the block width for readable titles); all-day/multi-day stay solid at all sizes. Reused the existing bin+Delete button on the event popover (the one agreed behaviour change, no more Edit-first-to-delete). Hardcoded colours to tokens, horizontal-only gridlines (verticals were briefed then removed at operator request), styled sidebar checkboxes, 6-row loading skeleton. Per-calendar colours (user data) left inline throughout.

- **F47 — Dashboard layout.** PR #118 (`9378ef2`). Root cause of the too-wide cards: PR #61 (Aug) switched the page to a single-column class that overrode the existing 2-col split, so nothing capped width. Fix: content capped ~1280px left-aligned, Inbox + System health top row, Recent quotes full-width (it is a 5-column table and needs the room), Automations full-width below. Breakpoint is a `@container` query on `.dashboard-shell` (first `@container` use in the repo; a screen-width `@media` would be ~180px off because the sidebar toggles 240px/60px). Removed the dead `.dashboard-main`/`.dashboard-side` machinery and the nested `<main>`. loading skeleton updated to match.

## Decisions

- **DEC53 LOCKED — page-CSS edits allowed, per page, discovery-first.** The reskin may edit page stylesheets (`dashboard.css`, `jobs.css`) to remove rules overriding the globals primitives, but per page, discovery-first (classify each rule: duplicates-a-primitive to strip vs does-real-work to retoken/migrate; never blind find-replace), and adding the primitive class to the owning component is in scope. `quote.css` stays out (rebuild case, F43). This was the fork blocking Dashboard + Jobs at the end of 09-11.

- **DEC54 CANDIDATE (needs operator lock) — sparse pages get hand-placed layouts, not masonry.** Pages with few cards of differing size/shape (Dashboard, Settings General, Settings Profile) get an intentional per-page arrangement. Masonry is rejected on evidence: the old attempt (PR #102, closed unmerged) used CSS multi-column, which reads down not across, reshuffles on data-height change, and needed ~15 full-width exceptions; and with ~4 cards it pays nothing. The F47 capped two-column suits the Dashboard's mix but must not be forced onto pages whose cards do not balance (General is one short + one tall form). Companion feature: F48.

## What did not fit, and why (the General question)

Tried to line up Settings General as the fast-follow to F47. Discovery had called it a good fit, but the real card heights (Company details short, Business info tall) mean side-by-side gives the same blank-space problem F47 left on the Dashboard. Concluded that sparse settings pages need bespoke per-page layouts (DEC54 / F48), not the generic two-column, and that this is proper design work per page, not a quick reskin. Stopped rather than rush one in.

## Method notes

- Every "shipped" claim verified on the Vercel preview by the operator before merge (real Testing Mover data where the state needed it: amber bars, status colours, event deletion). Codex cannot log in, so logged-in checks (real deletion, 403 branches, live colours) are operator-only by design.
- The phantom `page.tsx +NNN -0` harness card appeared on nearly every build (Codex spins up a throwaway login-harness to measure, deletes it before commit). Trusted `git diff --stat origin/main...HEAD`, not the file card, every time. Never once in a real diff.
- One process fix mid-session: briefs now say "draft PR, STOP, do not merge" explicitly. An earlier brief folded "verify on preview, squash-merge" onto one line and Codex read it as merge authorisation (F45 merged before the operator eyeballed it; no harm, UI-only). Every brief after was explicit.
- Two briefs shipped with a wrong assumption that discovery/preview caught: the amber "same colour" claim (the warning tokens were a paler amber than the bars, would have clashed), and the job-card rest shadow (inverted hover behaviour). Both backed out before merge. The STOP-rule and preview-first flow did their job.

## Bugs found this session (pre-existing on main, logged not fixed)

- Calendars/burger menu button leaks onto desktop and does nothing there (S294).
- Calendar Ctrl+wheel zoom logs a console error and may zoom the whole browser page (S295).
- Settings tenant-facing hardcoded values + em dashes in customer-facing copy (S296).
- (Event time range en dash was reviewed and left: a clock range is not house-style prose.)

## New backlog items

S288 convergence-cleanup (wells redeclare primitive values). S289 `--shadow-lift` weight revisit. S290 Jobs kanban lane-tone revisit. S291 Calendar secondary sidebar collapse. S292 Recent quotes widget trim. S293 Dashboard alignment revisit (left vs centre). S294 + S295 the two calendar bugs above. S296 Settings tenant-facing cleanup. F48 bespoke per-page sparse layouts. F49 Dashboard content expansion. F50 Jobs board visual-interest pass. DEC53 locked, DEC54 candidate.

## Numbering after this session

Next free S = **S297**. Next free F = **F51**. Next free DEC = **DEC55**. Check the active backlog before assigning; do not infer from chat.

## Launch plan

No launch-plan change this session. All work was pre-launch UI polish, not a gate (consistent with the 09-11 precedent). None of the bugs found are launch gates.

## Still open / carried forward

The category-vs-template investigation (from 09-11 v2, not touched this session) remains open. The prior UI-arc ranked list still has Mobile sweep, Calendar Week/Day polish, and Motion/flow polish untouched; Agents tab reordering and the post-onboarding tour remain blocked on the setup-completeness signal.
