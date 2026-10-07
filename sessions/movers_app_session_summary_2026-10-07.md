# Movers App, Session Summary 2026-10-07
## THE ENQUIRY TO CARD SESSION (F73 + F71 + F72 + calendar batch)

Chat spanned Tuesday 2026-10-06 (afternoon) to Wednesday 2026-10-07. Opened with "what needs doing now", picked F71 (enquiry creates the job card), and the Step 0 investigation for it found two pre-existing faults that would have left Cornwall's main enquiry source invisible at cutover. Those became F73 and shipped first. Main `4ab1455` -> `d487d2a`. Four migrations applied live. Every ship verified on production by the operator with row checks via MCP from this chat.

## What shipped, in order

| When | Item | Ref | What |
|---|---|---|---|
| 10-06 | F71 Step 0 | report only | `docs/investigations/f71-enquiry-card-investigation-2026-10-06.md`. Found: own-domain skip drops website form mail on any tenant whose `contact_email` shares the form sender's domain (Cornwall, Testing Mover); customer identity in form mail lives only in body lines; Reply-To never captured; every reply path sends to From, which for form mail is the business itself |
| 10-07 | F73 | PR #167 `5c86a34`, mig `20261006162048` | Website form submissions as first-class enquiries: Reply-To captured on both ingest paths; three-part form detection (listed sender + Reply-To present + Reply-To off-domain) bypasses the own-domain skip; `Label: value` block parsed to `form_fields`; `customer_email/name/phone` written on every run before computeDraft; `resolveReplyRecipient` (Reply-To else From) at five send sites; Review Queue + Escalated show the customer; "Website form sender" field on the sender-rules card (`tenant_agents.form_sender_addresses`) |
| 10-07 | F71 | PR #168 `78b2e4f`, mig `20261007090006` | Enquiry creates a job card: `createEnquiryCard` hooked after the drafted status write, own try/catch; `cards.source_agent_run_id` + partial unique index (idempotent across every re-drive); soft duplicate by email on the same board within 30 days adds an activity row instead; `jobs_settings.enquiry_cards_enabled` + `enquiry_cards_list_id` with a Settings > Jobs card (toggle + board/list picker, default Leads > Website); system `card_activity` row (actor null, "Auto Responder"); drawer line "Created from an enquiry on <date>" |
| 10-07 | S464 | `a10827c`, rework `1147b43` | Calendar desktop edge nav. First cut put small chevrons in the 20px gutter; reworked to 40px overlay circles over the grid edge, only in the DOM while the edge zone is hovered, pointer-events off while fading, hidden during drag, never a tab stop |
| 10-07 | S465 | `858b514`, `8ff07af`, rename `ef4e960`, mig `20261007113052` | Team Hours green `#007a4a` (white 5.41:1). Brief said the seed was `#009058`; real seed was `#10b981` (white 2.54:1), only Testing Mover had `#009058`. Migration redefines both seed functions and moved all 9 Team Hours rows |
| 10-07 | S466 | `9d658e1`, fix `fce57bf` | Phone month view: tap anywhere in a cell opens the day, 5 event rows before "+N more". Follow-up: the grid became a vertical scroller on busy months and ate diagonal swipes; removed the 288px grid floor in phone portrait, `overscroll-y-contain`; page no longer scrolls when the month fits. Swipe thresholds (12px start, 45° axis, 90px or 20% commit) never changed |
| 10-07 | F72 | PR #169 `c8e2b5b`, mig `20261007140000` | Multi-day jobs: `cards.delivery_date/delivery_arrival_from/to`; second job event `source_subtype='job_delivery'` derived by the same sync; unique index now (tenant, source_ref, source_subtype); drawer "Delivery day" section with validation (delivery >= move date, server-checked); Change date on the delivery event edits the delivery day; `{{delivery_date}}` + `{{delivery_arrival_time}}` tags with the gap gate; delivery changes trigger "Send updated confirmation?" |
| 10-07 | polish | `a33ac66`, `749b443`, `d487d2a` | Remove / Add delivery day as real buttons; job-event pop-up says "Cancel job" not "Delete" (manual events keep Delete); drawer-header CSS rule was overriding `.btn-primary` text colour inside the card-actions popovers (Archive confirm, Move/Copy, drag-back confirm) |

Full bodies of every done item are in movers_app_backlog_archive_APPEND_2026-10-07.md.

## Design decisions made in chat

- **Reply-To is the spine for form mail, body parsing the backup (F73).** Every form plugin sets Reply-To; body layout is per-plugin. Confirmed on a real Cornwall "New Quote Request" source: `From: Cornwall Movers LTD <hello@cornwallmovers.co.uk>`, `Reply-To: <customer>`, WPMailSMTP. Three-part detection keeps the loop guard: our own BCC copies carry an own-domain Reply-To and no listed sender.
- **Identity written once on the run.** F71 and the Inbox read `customer_*` columns; nothing re-parses. The pipeline loads the run before the F73 write, so the write returns what it stored and the card helper reads that.
- **Idempotency key = agent_run_id (F71).** Every re-drive (F33 Retry, auto-replay, recoverFilteredRun, reactivateCapturedItem) re-runs the same `agent_runs` row; the S309 sweeper does not re-drive. Partial unique index makes a race a 23505 the helper treats as "already done". Proven on prod: direct SQL insert with the same run id -> 23505.
- **Drafted-only cards for v1.** Live data: every escalated run on S199 was `thread_reply` (303), `escalate` (52) or `ignore` (1), none with a quote category or extraction. One helper, so adding escalations later is one more call site.
- **Soft duplicate by email, same board, 30 days.** Second enquiry from the same address adds "Another enquiry from this customer" to the existing card; never touches the card itself. A returning customer a year later is a new lead.
- **Enquiry cards default off, explicit list id.** Rename-proof; deleted list shows an amber line and treats the feature as off. Legacy single-board tenants can pick any list.
- **One card per job, multi-day = a delivery date on the same card (F72, candidate DEC63).** Two cards rejected (breaks deposit, quote link, history, one confirmation). Date range rejected (crew need two distinct arrivals). Sequence chosen: build F72 first, then seed Cornwall (option 1).
- **Edge nav as overlay circles, not gutter chevrons.** The gutter version was too small to notice and framed the grid for a hover-only control. Overlay is the carousel pattern; interruption rules recorded on S464.
- **S465 migration widened in chat** to cover the real seed default `#10b981` as well as `#009058`, otherwise Cornwall's leave bars would have kept dark text.
- **F74 shape agreed, not built:** Book job preview editable in place (this send only, template untouched); a line whose tags all resolve empty is dropped from the render; the gate becomes a warning ("These lines will be left out: ...") with Send anyway / Add details; a line mixing a blank tag with real content still blocks. No LLM.

## Findings worth keeping

- **Board seed was broken 29 Jul to 3 Sep.** Every tenant created in that window (Joe's, A1A2, S146, S199, Sep, Sep2) has one untyped "Jobs" board with legacy lists and no Archive, so Archive on a card fails with "Something went wrong". `20260909210000_restore_typed_jobs_trigger_seed.sql` fixed the trigger; Cornwall (27 Sep) and Onb (28 Sep) are typed. Live trigger confirmed `jobs_seed_tenant`. No fix needed for the five test tenants; S199 retires at cutover. Recorded under S165 / S475.
- **S202 residual seen live.** "Quick question: does the quote include boxes?" from a known customer, no thread headers, classified removals 0.9 and auto-sent a full first-contact reply on Testing Mover. Exactly the fresh-email-about-an-existing-job gap. Now S476 with a concrete example.
- **Testing Mover's removals template has no `{{signature}}`**, so replies go out unsigned; the signature card says "appended via the variable", nothing warns. S472.
- **A reply to a form submission quotes the original beneath, attributed to the business** ("On ... Cornwall Movers LTD wrote"). Cosmetic. S473.
- **Verifying the own-domain guard needs contact_email on the same domain as the sender.** S199 is on richardstowell.com, so a hello@cornwallmovers.co.uk loop check classified instead of skipping; flipping contact_email for one send gave the real `skipped_own_domain`. Quirk recorded.
- **Supabase MCP `apply_migration` returned `cancelled` three times with no approval prompt shown.** Codex applied the F72 migration through the Management API SQL route and inserted the `schema_migrations` row by hand (version = filename timestamp, shape copied from an MCP-applied row). Quirk recorded.
- **A Supabase CLI now exists locally** (Codex ran `npx supabase gen types` with the access token in `.env.local`). S130's premise ("no CLI has ever existed") is dead; the orphan-migration repair is now possible. Re-scope recorded on S130.
- **`database.types.ts` drifts** because builds hand-add columns before the migration is applied. F73's regen also absorbed old drift (cards field order, the S415-dropped `jobs_delete_expired_archives` type). Quirk: regenerate after every applied migration.

## Measurements

- F73 adds one settings query per inbound run (the identity write needs `form_sender_addresses` before computeDraft loads the agent). Noted for S456.
- F71 helper: one settings read, one list read, one duplicate lookup (indexed), one insert, one activity insert, one sync. All inside the run's existing function budget.

## Test state left behind

Testing Mover `16e64782`: Enquiry cards ON -> Leads > Website. J-0027 "Richard" (direct email, has "Another enquiry" activity). Carlos J-0021 has a delivery day 9 Oct. Lucy J-0020 restored to Upcoming after the cancel test (activity log shows the full F72 run: delivery added, changed, removed, cancelled, uncancelled). "Moving v2" template now carries a `Delivery day: {{delivery_date}} from {{delivery_arrival_time}}` line. Auto-send is ON on this tenant: three test replies went to the operator's gmail. `default_job_block_minutes` 90.

S199 Movers `8f31127f`: `form_sender_addresses = {hello@cornwallmovers.co.uk}` (keep; it receives Cornwall form mail). Enquiry cards OFF. `contact_email` back to hello@richardstowell.com. J-0003 "Tess Card" archived by SQL (no Archive board). Two Tess drafts discarded; "Loop check" ignored row and "Loop check 2" skipped row remain, harmless.

## Cornwall cutover additions

Two settings on Cornwall's tenant when its mailbox connects: Agents > Overview > Website form sender = hello@cornwallmovers.co.uk, and Settings > Jobs > Enquiry cards ON -> Leads > Website. Without the first, every website enquiry is dropped as own-domain mail. Make's enquiry-to-card scenario is now replaceable. Cornwall Trello seed is unblocked (F72 done) and HELD by the operator for now.

## Open and next, in the order agreed

1. Cornwall Trello seeding (other chat), two-day jobs as one card with a delivery day, then archive the 8 manual job events by SQL. Held.
2. S450 Jobs tour + first-time explainers.
3. F74 editable confirmation + optional lines (design chat first, shape above).
4. Perf: S456 middleware (now also carries the F73 extra query), then S453, S457 to S459.
5. Small: S472 signature marker, S473 quoted attribution, S463, S467, S468, S469, S470, S447.
6. Quoting agent + Agents page differentiation: design chat before any number is assigned.

## Process notes

- Brief file lists were short twice (F73: five data-plumbing files; F71: one helper file); Codex stopped correctly each time and both were one-line resolutions. The expected-files list is worth keeping even when it is wrong.
- Brief facts from the backlog were wrong once (S465 seed colour). Codex checked prod before building. Backlog lines asserting data state should say which tenant they were read from.
- Investigate-before-build earned it again: F71's Step 0 surfaced F73, which is a Cornwall cutover gate nobody had on a list.
- Verification on S199 for anything own-domain needs contact_email flipped for the test, then reverted.
- Docs moved to git at session end. The project Context reader truncated the 554 KB active backlog at 262 KB with no warning and cannot edit in place; two hand-retyped files each carried a defect until rebuilt from the attached originals. `rjstowell/movers-app-docs` (private) is now the source of record, Context keeps search copies, and the active/archive split rule is unchanged (the archive is now one real append-only file instead of one `_APPEND_` upload per session). The 24 archive appends (2026-09-10 to 2026-10-06) and 45 Markdown session summaries (2026-08-27 on) were consolidated into the repo the same evening, each copied twice by independent agents and compared byte for byte before commit.
