# Movers App: Session Summary 2026-09-29

**Chat span:** 2026-09-28 (afternoon) to 2026-09-29 (morning). **main `1ea9e1b` -> `c92ebae`. Tests 1314 -> 1425.** Four merged PRs (#146 F64 slice A, #147 slice B, #148 slice C, #149 slice D), three direct-to-main commits (`16cee4b` S389 banner polish, `da89737` AGENTS.md, `89eca36` migration stamp), six migrations applied via MCP from Codex. Everything verified on production or the preview by the operator; the auto-close cron verified with SQL backdates from this chat.

## 1. The shape (agreed before any build)

Operator asked for a Support page: submit ticket, support chat, FAQ/help, plus a live AI helper that knows what page the user is on. Project context showed the helper already exists on paper: F6 Manager Agent v1.5 (global app-wide surface, explain + diagnose grounded in tenant state). Building a separate help bot would have violated the 2026-07-22 lock (one user-facing assistant identity). Shape agreed:

1. **Help articles**, static, searchable. Knowledge base for both humans and the assistant. Not competing with it.
2. **The assistant** = F6 phase 1 entered from a global launcher. Context = current route + page manifest + read-only tenant state. No writes. Grounded in the articles; otherwise "I don't know, raise a ticket".
3. **Tickets**, human-answered, in-app thread, email + push notifications.
4. **Live human chat: skipped.** Solo operator. Live chat with nobody on it is worse than none.

Handoff chain: article -> assistant -> ticket. Order: tickets (F64) -> articles (F65) -> assistant (F6 phase 1).

## 2. F64: support tickets, four slices, all live

**Slice A, PR #146 `097ab28`.** `/support` page (help link, ticket form, "Your tickets"), `support_tickets` table (own table, not `support_reports`: human-raised, many per tenant, tenant reads own), admin `/admin/support` list + `?ticket=` view, admin email via the existing Postmark sender, nav item. Codex replaced my `document.referrer` route capture with a `lastRoute` tracker in AppShell (client nav does not update referrer). First admin actions did not persist: the submit button's `name="status"` value never reached the server action, silent early return. Fixed with a client component that passes values explicitly, plus inline success/error on every operator control.

**Slice B, PR #147 `dfc39b2`.** Operator asked "how do you actually reply?!" My brief had said reply by email from Gmail; that was my miss, and the notification was admin-to-admin anyway. Rebuilt as an append-only `support_ticket_messages` thread: operator reply (emails tenant), internal note (`is_internal`, RLS-hidden from tenants at policy level, column-level too), tenant reply (emails admin), status rules (operator reply -> replied; tenant message -> open; any message on closed reopens; internal notes never change status). `admin_notes` backfilled into a first internal note, column dropped after the production deploy (migration held until then, Codex's call). Follow-ups in the same PR: "Earlier messages" section in both email directions (internal notes double-guarded out), Close/Reopen ticket moved to the bottom by the composers, tenant "Issue resolved?" self-close, closed-state banner copy, severity (Minor / Major / Critical, tenant-set; badge + triage sort in admin; `[SEVERITY]` in admin email subjects). Category and severity kept separate on purpose: what kind of help vs how fast.

**Slice C, PR #148 `3ab8b12`.** Tenant notification on operator reply: blue info banner on the dashboard ("Support replied to your ticket: X" / "to N of your tickets", no dismiss, clears on open), Support nav badge (fetched on navigation, thread open, and tab focus with a 5-second gap, no timer), push via the existing `sendPushToUser` (fire-once via `push_sent_at`, failure reasons stored, "no devices" not alarmed), `tenant_last_viewed_at` set server-side on thread open, raiser-only (a teammate opening the thread does not clear it). Follow-up: "New reply" pill + unread-first sort in the tenant list, multi-ticket banner lands on `/support#tickets`, unread tickets older than the 50-row window still fetched.

**Slice D, PR #149 `c92ebae`.** Auto-close. `awaiting_tenant_since` set on operator reply, cleared on tenant message or close; reminders at day 2 ("still need help? closes in 5 days") and day 5 ("closes in 48 hours"), close at day 7 with an "automatic" thread message; email + push each time; `closed_reason` operator / tenant / auto. Hourly cron `/api/cron/support-autoclose` (cron registry taught hourly schedules; heartbeat-monitored). Every action is a conditional update on `(id, status, awaiting_tenant_since, reminder field)` so re-runs and concurrent tenant replies lose cleanly. Verified on production with SQL backdates from this chat: day 2 fired once (second run 0), day 5 fired once, tenant reply reset everything and dropped the ticket from the count, day 7 closed with `closed_reason='auto'` and the thread message (reminders null, so close wins over missed reminders), tenant reply reopened. Preview cron test was blocked by Vercel Deployment Protection (login redirect), so it ran on production after merge; low risk because guards are unit-tested and only three tickets were in scope.

## 3. Also shipped

- **S389 `16cee4b`.** Support banner switched from amber warning to blue info style (amber = fix something, red = broken, blue = information; blue is the existing Open-pill colour, tenant theming does not touch status colours). All six dashboard banners moved onto one `.bar-status` layout: icon top-left, text right, link on its own line under 768px. Operator verified on the phone.
- **AGENTS.md `da89737`.** First repo rules file for Codex: clear harness leftovers under `.next/types` and typecheck clean before every commit; rename MCP-applied migration files to the recorded version; no em/en dashes in copy; no merges/migrations/pushes to main without an explicit operator instruction.

## 4. Things that cost time

- Postmark, not Resend, sends admin@moversapp.app mail. Twenty minutes in the Resend dashboard looking for emails that were never there. Now in the quirks log.
- Gmail folds same-subject replies into one conversation. Two "missing" operator replies were collapsed under the third.
- Codex named migration files with a 2027 prefix; the DB stamps its own version at MCP apply. Four renames before merge. Rule now in AGENTS.md. The repo has older 2027-prefixed files with the same drift (S394, folds into S130).
- Codex committed twice with typecheck errors from a deleted harness page's stale `.next/types` entry. Now a stop rule.
- Vercel preview URLs need a Vercel login, so a bearer-token cron call from PowerShell gets the login page. Bypass token (S397) or test on production.

## 5. Doc changes this pass

- Active backlog: new 2026-09-29 header; 09-28 header demoted to a pointer; numbering S399 / F66 / DEC61; new session block (completed F64 A-D + S389, operational decisions, S390-S398, F65 reserved with the article list); F6, F20, F33 amended.
- Archive append 2026-09-29: outgoing 09-28 header + numbering, DONE bodies for F64 slices A-D and S389.
- Launch plan: new baseline `c92ebae`; F64 added DONE to Tier 2 (a stranger tenant needs a support path before they pay); F65 and F6 phase 1 to Tier 3; estimate unchanged.
- Quirks: section 17, eleven lines.

## 6. Next chat: F65 help articles (handoff)

Fresh chat, this one is full of screenshots. Claude writes the articles from project docs; operator reviews for accuracy; Codex builds the renderer.

**Format.** One Markdown file per article under `content/help/` (or `app/(app)/help/` if Codex prefers the route colocated). Frontmatter: `title`, `slug`, `summary`, `routes` (app paths the article relates to, used by the F6 assistant to pick the article for the current page), `updated`. Body: short, one job per article, no em/en dashes, screenshots optional later. Codex builds: index page at `/help` with search (client-side over titles + summaries is enough), article page at `/help/<slug>`, the existing DNS guide folded in as an article, the `/support` "Help guides" panel pointing at the index.

**Starting article list (correct before writing):** getting started and the setup checklist; connecting your inbox (inbox-only vs app-only, the letterbox address, forwarding); sending domain and DNS (F20 content); how the agent classifies and replies (categories, escalation, what "review" means); templates and auto-send (the consent gate, the delay, per-template auto-send); review queue, escalated and filtered; voice settings; quotes and the quote engine (routes, survey distance, manual override, VAT, PDF); jobs, calendar and vehicles; billing, plans and seats; team members and crew; notifications and push (installing the PWA); support tickets (severity, replies, auto-close); account and security (password, sign-in, multi-company switcher).

**Not in F65:** the assistant itself. That is F6 phase 1, its own session, after the articles exist.
