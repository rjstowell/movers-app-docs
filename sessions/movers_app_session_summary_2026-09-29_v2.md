# Movers App session summary, 2026-09-29 v2 (help articles shipped + the UI audit)

Session ran 2026-09-29 into 2026-09-30. Second session of 09-29; the first (support surface, F64) is in movers_app_session_summary_2026-09-29.md.

## 1. What shipped

- **F65 DONE.** Two PRs, both verified on preview and prod by the operator as owner and as Testing Mover crew.
  - **PR #150 `72ffebf`, feat(help): F65 help articles.** Fifteen Markdown articles in `content/help/`, one per topic, frontmatter `title`, `slug`, `summary`, `routes`, `updated`. `lib/help/articles.ts` reads them at build with a small strict frontmatter reader (gray-matter rejects colon-space titles); build fails on a missing field, empty routes, bad date, duplicate or malformed slug, or the reserved slug `dns-verification`. `/help` index with client-side search, `/help/<slug>` article page (react-markdown, raw HTML off, first `# title` line skipped), `/help/nope` 404. The existing `/help/dns-verification` page kept as is with a back link and listed as a card. Support > Help guides links to `/help` plus four articles. Crew already allowed (middleware is a blocklist). `content/help` bundled via `outputFileTracingIncludes`. Follow-up on the same PR: search matches article bodies too, title/summary hits rank first, formatting stripped before matching.
  - **PR #151 `1b0b7d1`, feat(help): grouped help index, role-aware Support tiles, article feedback footer.** `/help` in four sections (Getting set up / Your agent / Quotes and jobs / Account; `lib/help/groups.ts`, a slug outside the map fails the build), two-column cards, no Updated line on cards, search flattens to a ranked list, `?q=` follows the box via `replaceState`. Support > Help guides: live search as you type (six rows then "See all N results", empty state), role-picked tiles (owner/admin: getting-started, connecting-your-inbox, sending-domain-dns, templates-and-auto-send; crew: jobs-calendar-vehicles, notifications, account-security, support-tickets), Browse all help guides as `btn-ghost` (17:1 on a mint brand versus 1.52:1 for `btn-secondary`). Article page: 17px body, feedback footer "Was this useful?" Yes / No, thanks line only right after a click, question with the prior answer marked on the next visit. Table `help_feedback` (unique user + slug, RLS tenant-scoped, write as self only, no delete), migration **`20260929183926_help_feedback`** applied via MCP from chat, repo file renamed to match. Verified from chat: one row for the operator, slug `quotes`, No then Yes updated in place.
- **Docs tidy `9abf69e`**, chore(docs): `docs/investigations/` and `docs/handoffs/` subfolders, sixteen renames, README + s151 paths updated, F65 audit added, duplicate s173 removed. Surfaced that `docs/investigations` is gitignored with force-added files (S412).
- Tests 1425 -> 1469. main `c92ebae` -> `1b0b7d1`.

## 2. How the articles were written and checked

- Claude wrote all fifteen from the project docs, then applied two filters the operator set for every article: (1) nothing that reveals how the system works internally (no classifier, prompts, providers, cron, schema, formulas, catalogue size); (2) plain, quick, one job per article. Report per article, two lines, not a full read by the operator.
- Accuracy came from three sources, in order: operator screenshots of the real UI (Dashboard checklist, all Agents tabs, Quote, Pricing, Fleet, Vehicles, Profile), then a **five-agent read-only Claude Code audit** of every tenant page (labels verbatim with `path:line`, role gates, helper text, ~1400 lines; `docs/investigations/help-ui-audit-2026-09-29.md`), then the operator's six preview checks per PR. Screenshots for how it looks; the audit for what it says and who can reach it.
- Corrections that mattered: template switch is "Auto-send eligible" and Active lives on categories; quote routes and survey distance are not on Rules for Testing Mover (see S399); Review Queue actions are Approve + Send / Edit + Send / Discard with "Wrong response?" for corrections; deposit percentage is on Pricing > Adjustments; roles are Owner / Admin (with Billing, Pricing & fleet, Users, Reports permissions) / Crew; up to three owners; sign out is at the bottom of Settings > Profile; mailbox connect is IMAP host/port/user/password, Gmail app password, no Microsoft 365; plans Starter <=4 / Pro 5-10 / Scale <=20 seats; company delete archives for 90 days.
- Articles (slugs): getting-started, connecting-your-inbox, sending-domain-dns, how-the-agent-replies, templates-and-auto-send, inbox-folders, voice-and-signature, quotes, pricing-settings, jobs-calendar-vehicles, billing, team, notifications, support-tickets, account-security. Source of truth is the repo; the chat zip is a copy.

## 3. Audit findings -> backlog (S399-S412)

Fourteen new items, all in the active backlog with full bodies. Recommended batching for the next chat:

**Batch A, one "audit cleanup" brief, direct to main, no PR** (all small, orthogonal):
- S400 stale dev copy (Sending identity helper, Mailbox card header).
- S401 dash/label sweep (quote fit strings, vehicle calendar titles, "MoversApp", ellipsis, Website/Website URL, Default label, "60 months").
- S404 hide Start quote / Load in QE / Load snapshot for crew.
- S405 block renaming the three system calendars.
- S407 Resend + Revoke on pending invites; login-deletion sentence on Remove.
- S408 trial subtext by eligibility + "Trial ends {date}" on Billing.

**Batch B, one brief each, branch + PR:**
- S399 Quoting card on Agents > Rules (routes + survey distance always rendered). Tier 2. Then update the quotes article line "raise a ticket".
- S402 billing page + actions gated on the Billing permission. Tier 2.
- S406 plan prices from Stripe price objects, fallback to constants with a log. Tier 2, before S264.
- S409 brand-contrast rule, app-wide, measured at theme render. Tier 2.
- S403 field-level permission on `/vehicles` quote fields. Tier 3.

**Batch C, help follow-ups, one brief, branch + PR:** S410 counts on `/admin` + S411 comment on No (one migration, nullable `comment`).

**Operator decision, no code until decided:** S412 `docs/investigations` tracking. Recommendation: keep the ignore rule, `git rm --cached` the three force-added files.

## 4. Doc changes this pass

- Active backlog: new 09-29 v2 header, previous 09-29 header shortened to one line, numbering bumped (next S413 / F66 / DEC61), F65 marked DONE with a pointer, Completed / New items / Amendments blocks appended (S399-S412, F65 shape for the record, F6 inputs, F20 absorbed, S211 leftovers -> S401, S130 one more migration).
- Archive append `movers_app_backlog_archive_APPEND_2026-09-29_v2.md`: outgoing 09-29 header and numbering, F65 body.
- Launch plan: baseline `1b0b7d1`, F65 row DONE, estimate line naming S406 (with S264), S402, S408, S409.
- Quirks section 18: survey card render condition, calendar name-match, gray-matter colon titles, `outputFileTracingIncludes`, skipped H1, `btn-secondary` contrast, preview writes to prod DB, PowerShell `Move-Item` rename trap, Explorer address bar and Git Bash, `docs/investigations` ignore rule, parallel read-only audit as a method.
- Project instructions: operator added a Shell section (PowerShell for all commands, Git Bash only when a Unix one-liner is materially easier).

## 5. Open threads for the next chat

1. Batches A, B, C above. Suggested order: A (one afternoon), then S406 and S402 (stranger list), then S399, S409, S403, then C.
2. S412 decision.
3. F6 phase 1 design now has its inputs: article bodies, `routes` per article, `help_feedback`. Not started.
4. Quirk to watch: Codex/Claude Code still names migrations with a `2027…` placeholder; rename to the MCP stamp before merge (AGENTS.md rule held this session after one reminder).

## 6. Method notes

- Screenshots in chat worked for UI truth but eat context; the read-only audit brief (goal, per-page report shape, "quote labels verbatim", "flag what you could not determine", write to `docs/investigations/`) is the reusable pattern and cost one prompt.
- Six-check preview lists per PR, adjusted after each follow-up, kept verification to a few minutes per round.
- PowerShell note for the project instructions is in place; the two traps that cost time are in quirks 18.
