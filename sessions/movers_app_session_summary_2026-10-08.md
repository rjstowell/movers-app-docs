# Movers App, Session Summary 2026-10-08
## THE SID WALKTHROUGH SESSION (S479 + S484 + S480 + S481)

Thursday 2026-10-08, afternoon and evening. Sid (Cornwall Movers) walked the app with the operator in the morning because Cornwall is still not running on it. Nine notes came out. Four were DNS / sending identity and went to the parallel chat that shipped F75 and S478. The other five came here. The session waited for F75 to land (16:14), then ran everything direct to main from one Codex chat, each item verified on the operator's phone or on /quote before the next brief. Numbering was claimed in chat across both chats (this one S479 to S484, F76 to F77; the F75 chat starts at S485) to avoid a 09-25 style collision.

## What shipped, in order

| When | Item | Ref | What |
|---|---|---|---|
| 16:40 | S479 | `65c2819` | Jobs board hold-to-drag on touch: `MouseSensor` distance 6 + `TouchSensor` delay 400ms tolerance 8 replace one `PointerSensor`; `.job-card` `touch-action:manipulation`, callout and selection off. Root cause was a 6px activation for touch and mouse alike, with `touch-action:none` so the browser never got the scroll. |
| 17:31 | S479 | `62b2740` | `100dvh` beside `100vh`, `overscroll-behavior-y:contain`. Not enough (see below). |
| 17:31 | S484 | `982d65d` | Same-list drag down landed one card short: ordering moved to `lib/jobs/reorderCards.ts`, `arrayMove` on original indexes, 7 tests. Pre-existing bug found by Sid's checks. |
| 20:50 | S479 | `8736d28` | Board sized by the shell: `body:has(.jobs-page){height:100dvh}` on phone, `.jobs-page` flex column, `.board-canvas` flex:1, no viewport calc. Harness-measured at 1280x720 and 375x812. |
| 21:08 | S479 | `5512abe` | iOS blue selection across two lists during a drag: `-webkit-user-select:none` on `.board-canvas` (unprefixed did nothing on iOS), inputs excepted; `.card-stack` becomes the flex scroller, Add card pinned. |
| 21:33 | S480 | `30ef942` | Setup checklist "See what your agent writes" ticks after the first paste test: `see_writes_used` marker in `tenant_setup_dismissals`, written only when the agent actually ran; no run persisted. 6 tests. |
| 22:07 | S481 | `6378ee2`, `fe80056` | Quote Engine Internal report (PNG) + (PDF) merged into one button with a menu; phone row fits one line at 375px, 44px tap targets. |

Full bodies in movers_app_backlog_archive.md, ARCHIVE APPEND 2026-10-08.

## Sid's nine notes, where each went

| Note | Outcome |
|---|---|
| Job cards move by accident on mobile | S479 DONE |
| Consider removing internal report PNG button | S481 DONE (merged into a menu rather than removed) |
| Test a reply should tick the dashboard checklist | S480 DONE |
| Multi-day: note by delivery date, more than one collection/delivery day, checkbox | F76 OPEN. Clarified late: the ask is extra dated days per job (3+ day jobs), not a label. Design chat first; DEC63 revisit. |
| Deposit received then updated confirmation: ask to remove the deposit request | F74 amended, same design session |
| Quote engine field order (customer name at top is wrong) | F43 amendment; get Sid's order before that rebuild |
| Is "send a test email to yourself" only for app-inbound tenants? | Closed: F12(b) already switches by mode (`app_only` / `inbox_only`) |
| Turn quote into a job card | S483 OPEN |
| Escalated reply box needs an LLM draft or a route to the Review Queue | F77 OPEN, lean "Draft with AI" in the composer; Review Queue route carries a DEC62-class auto-send risk |
| (DNS / sending identity, four notes) | F75 chat, S478 + F75 DONE there |

## What verification turned up

- **The 208px viewport guess was wrong and `overscroll-behavior:contain` hid the proof.** The phone shell adds a top bar, 68px + safe-area bottom padding on `<main>`, heading and tabs; the canvas overflowed by a little and contain stopped the page from scrolling to it, so Add card was cut off. Fix was structural (flex from the shell down), not a better number.
- **iOS ignores unprefixed `user-select`.** The first S479 commit set `user-select:none` on `.job-card`; iOS still started a text selection on the 400ms hold and extended it across lists as the finger moved. `-webkit-user-select` is required; inputs inside the canvas need `user-select:text` back or typing breaks.
- **Dropping below the card you drop on landed above it.** Remove-then-splice shifts every lower index by one. Moving up was fine, so nobody had noticed.
- **The Cornwall checklist vanished because it is complete.** Sending verified by F75 today was the last counted row (one real inbound test run already on record, Add team dismissed). Testing Mover and S199 show no checklist because the operator is admin there and the checklist is owner-only. "See what your agent writes" cannot hold the widget open because it does not count. Verified S480 on Sep2 Movers (owner login `richsmap05+030926@gmail.com`, checklist wide open, agent enabled first).
- **First S480 verification run wrote nothing** because it hit the pre-deploy build (push to live is 2 to 4 minutes); the second run wrote exactly one marker row, `agent_runs` unchanged.
- **Codex verifies CSS and layout in a throwaway harness page** (real components, shell classes copied, deleted before commit, stale `.next/types` entries removed by hand). Measurements were accurate for S479 and S481; the real-phone checks still found the two things a harness cannot (address-bar dvh behaviour, iOS selection gesture).

## Deferred, with the reason

- **S482** iOS scroll indicator on the card stack: nothing shows while scrolling on the iPhone; `scrollbar-width` is desktop-only; a custom one is JS. Wait for real list lengths.
- **S483** quote to job card: moderate, pairs with S423, not a Cornwall day-one blocker.
- **F76** extra collection/delivery days: data model change (days per card, events per day, template tags per day). Cornwall's live need is two-day jobs, which F72 covers.
- **F77** escalated reply draft: design first; route (b) could send customer email without an explicit action.
- **F9 question**: should the paste-test row count toward collapse? Not decided.

## Process notes

- Two chats numbering in one evening: the second chat claimed a block in chat and told the first where to start. Worked. Recorded in the backlog's working-rules amendment.
- Codex reports pasted as text throughout; screenshots only for the UI. Each brief ended with a phone check the operator could do, and each item was verified before the next brief went out.
- Lint baseline: 25 warnings before S479, none from tonight's files. F57 snapshot (`customerExport.test.ts`) still the only red test (S447). `UnsavedChangesProvider` anchor test flaked once under full-suite load (S469).

## Launch plan

No stage change. Cornwall's Jobs board is now usable on a phone (hold-to-drag, correct ordering, no page scroll, no selection), which was the blocker Sid raised for day-to-day use. Trello seed still held by the operator.
