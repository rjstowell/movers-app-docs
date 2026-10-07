# Movers App, Session Summary 2026-10-06
## THE BOOKING AUTOMATION SESSION (F70 + drawer and calendar polish)

Chat spanned Friday 2026-10-02 to Tuesday 2026-10-06. Started beside the S441/S444 chat and waited for its doc pass before assigning numbers (three sessions in a row had published "next F70 / DEC62" without consuming them, which looked like a clash and was not). Main `9bef57b` -> `4ab1455`. Tests 1711 -> 1854; the only red test throughout is the known S447 snapshot.

## What shipped, in order

| When | Item | Ref | What |
|---|---|---|---|
| 10-02 | F70 slice 1 | PR #163 `bfefbaf`, mig `20261002184659` | List roles, card arrival window + vehicles/team, job events on Main derived from the card's list, ⏳/✅, locked lists with tap popover, job events read-only in the calendar pop-up |
| 10-05 | S449 | `22fca09` | Calendar search across all dates, jump + open |
| 10-05 | S452 + S449 follow-up | `b0d1205`, `84b729c` | Toggles client-only (were URL state, 4 serial queries); search index client-side, instant; colour dot; clear on pick; S451 closed in passing |
| 10-05 | S448 | `1cf064e` | Drawer Save/Discard in the sticky header, Ctrl+S, onSubmit instead of action= |
| 10-05 | S454 | `a5f8296` | Save latency measured (~20 serial trips) and cut; sync parallel + conditional; no revalidate |
| 10-05 | S455 | `008863a`, `45dff4b` | Drawer phone layout: no overflow, four paired rows, bottom save bar above the keyboard |
| 10-05 | S460 / S461 / S462 | `34c6248`, `2acad7c`, `1def0f2` | Month nav + drag without selection; start times on desktop bars; WCAG text colour |
| 10-05 | F70 slice 2 | PR #164 `b2a738a`, mig `20261005135310` | Booking templates table, renderer, LLM paste-to-template, Settings > Jobs editor + preview |
| 10-05 | F70 slice 3a | PR #165 `49401a0`, mig `20261005152403` | Book job modal + send (HTML + text, greeting, signature), Deposit received + Undo with confirm, pills, activity |
| 10-06 | F70 slice 3b | PR #166 `1fe67c1`, mig `20261006080245` | Drop on Awaiting Deposit opens Book job, booked-card drag-back confirm, rebooked flow, Change date on the event pop-up, Cancel job both sides |
| 10-06 | S471 | `4ab1455` | Crew read-only card as selectable text, Copy + Open in Maps, tel/mailto |

Full bodies of every done item are in movers_app_backlog_archive_APPEND_2026-10-06.md.

## Design decisions made in chat (not DEC-numbered unless stated)

- **Buttons send email; lists drive the calendar (DEC62, LOCKED).** Make sent the booking email on a card move. Here a drag never emails on its own; the "Book job" button and the drop-on-Awaiting-Deposit modal are the two explicit paths, and both go through the same modal. The calendar icon derives from the card's list every time, no option. Every control on a job event acts on the card.
- **Lists need identity, not names.** Tenants can rename/reorder/delete lists, so `lists.role` was added and role'd lists are locked (operator's call: a renamed list gets misused while the DB still treats it as Awaiting Deposit). Only two roles; Active/Pending left unroled because nobody could say what it means to the crew.
- **Main, not a new Jobs calendar.** Main is the diary you look at; jobs fill it; provisional jobs belong there too. Vehicles has its own calendar because it is a different data type.
- **Cancel over refusal.** Delete on a job event opens "Mark job cancelled?" instead of refusing; same action from the card. Shipped in 3b (3a carried a temporary refusal).
- **`confirmation_sent_at` and `booked_at` are distinct.** Sent = invitation to pay the deposit; booked = deposit confirmed. State = list; timestamps for display and idempotency; history in `card_activity` (`confirmation_sent`, `booked`, `deposit_undone`, `rebooked`, `cancelled`, `uncancelled`). No `rebooked_at` columns: rebooking N times is N activity rows.
- **Arrival window, never all-day, never a duration estimate.** `arrival_from` required to book, `arrival_to` optional; a fixed start uses `default_job_block_minutes` (60) purely so the grid can draw it. Template tag `arrival_time` renders "9:00am" or "between 12:00pm and 3:00pm".
- **Templates, not job types.** N templates per tenant with a default and a picker shown only when >1. Moving / man-and-van / single-item are three templates the tenant writes. Gate is template-driven: a tag only blocks a send if the template uses it, so a tenant who never mentions vans never sees a vans warning. `vehicles`/`team` live on the card anyway for the crew and calendar.
- **LLM once at save, not per send.** Paste the existing email, convert once to merge tags, review, save; sending is deterministic. Markdown-lite (**bold** within a line, `- ` bullets) reproduces Cornwall's formatted confirmation; greeting is a visible first line, not hidden magic; the tenant's Email signature is appended the same way replies do it.
- **Resend never un-books.** A send on a card already on Awaiting Deposit or Upcoming does not move it, even with "always move" on (Codex caught this).
- **Cut:** drag remember-choice + `drag_to_awaiting_deposit_sends`. Dropped the day after locking because a one-tap "Not now" is cheaper than a setting that turns a gesture into an email. Drag itself stays (it is how every other list works) and the drop opens the same modal.
- **BCC stays.** Booking sends BCC `contact_email` like replies; the operator thought this was gone. It is S190, scheduled for app-wide removal when writeback covers Sent.
- **Main deploys to Production.** Only PR branches get a Vercel Preview; S-items are live when CI passes. Test S-items on production as Testing Mover.

## Deviations by Codex that were accepted

- Click-driven lock popover instead of the brief's "reuse the board's popover" (the board's is a CSS hover panel, dead on iOS).
- `export const DEFAULT_BLOCK_MINUTES` from the sync (one constant, two readers).
- "Not set" instead of a dash for empty crew fields (AGENTS.md dash rule).
- `updateCardDates` instead of calling `updateCard` with three fields (would blank the card).
- `deposit_undone` logged by the move core, not the Undo button, so drag/Move/Undo each log once.
- Calendar job-event buttons tightened to the jobs permission (`context()` requireManager) from the calendar's broader edit rule.
- Preview height from grid stretch after the ResizeObserver approach stuck at 307px.
- Undo convert button added without being asked.

## Measurements worth keeping

- One Ctrl+S before S454: ~20 serial Supabase trips (5 middleware, 10 action incl. a duplicate `context()` and 5 inside the sync, ~5 on the revalidate render). Local 2170ms; prod 2 to 3s with cold functions. After: 985 to 1110ms local, prod felt instant. Sync 586 -> 242ms.
- Calendar toggle before S452: 595 to 966ms local (URL push, full server render); after 22 to 33ms, no request. Search keystroke 557 to 827ms -> 16 to 19ms.
- Prod functions run in lhr1, Supabase in eu-west-2, so per-trip cost is small; serial count is the enemy, not distance.

## Test state left behind (Testing Mover `16e64782-f2ee-4cd9-b3c9-7426b2640750`)

- `tenant_sending_identity.autosend_consent_ack = true` by SQL (domain `mail.richardstowell.com` still pending). Sends go from `hello@send.moversapp.app` via Resend, BCC to contact email.
- Owner richjstowell@gmail.com; admin rjstowell@hotmail.co.uk; crew cornwallselectmotors@gmail.com, richardrockon@hotmail.co.uk. Crew are read-only on cards.
- Templates: "Moving" (default), "Moving v2" (bold + bullets). `deposit_received_confirm = never`. `default_job_block_minutes` possibly 90 from a test. `book_button_moves_card` was toggled to always during a test; check before relying on it.
- Lucy Morgan J-0020 and Carlos Diaz J-0021 have been through every flow; their activity logs are a worked example. A demo-seed manual event for Lucy (10 Oct) sits beside her job event.

## Cornwall (live)

Jobs board has no cards. 13 seeded calendar events are manual (8 real jobs incl. two multi-day, 5 visits/unavailability/queries). The Trello seeding chat was held for F70 and is unblocked now. After seeding, archive the 8 manual job events in favour of synced ones (SQL, operator + Claude). Multi-day jobs -> F72 before the seed if the two two-day jobs should land as one card each.

## Open and next, in the order agreed

1. Cornwall Trello seeding (other chat), then archive the manual job events.
2. Calendar batch, one direct-to-main brief: S464 edge nav, S465 Team Hours green, S466 phone cells tap-to-day + 5 rows.
3. S450 Jobs tour + first-time explainers (describes finished F70 behaviour).
4. F71 enquiry -> card (scoped in the backlog), with the S199 form Reply-To check.
5. F72 multi-day jobs design.
6. Perf: S456 middleware first, then S453, S457 to S459.
7. Small: S463 safe-area, S467 console prompt, S468 generic-sending button, S469 flaky test, S470 permission name, S447 snapshot.
8. Payments feed: own session, not prioritised (MacroDroid today; open banking later).

## Process notes

- Paste Codex output as text; screenshots only for UI. Roughly a quarter of the tokens and greppable.
- One chat per S-item; F-slices share a chat. Investigation brief (Step 0, report only) before a build brief when file names are unknown; it paid for itself on slice 1.
- Merge prompts list the expected files and tell Codex to stop on anything extra; it stopped twice (once correctly for an unlisted export, once for my own short list) and both were resolved in one line.
- Verification steps must say which board, which login and which deployment; two mis-steps this session came from "Preview" (meant production) and "drag" (cross-board drag is impossible).
