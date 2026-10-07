# Movers App — Session Summary 2026-09-25 v2

**Theme: OPERATOR SWITCHES + PRE-MANAGER READINESS + S375.** Not a build session. Two operator items closed from chat (S253, S330), one small shipped (S375, `a46fc07` + migration by hand), one read-only Codex investigation (multi-company), one seed check via MCP. main `37fd467` -> `a46fc07`. Tests 1229 -> 1239. Docs from the 09-25 morning session were added to the project files mid-session (my first pass ran on stale docs; corrected).

## 1. The question that opened the session

Operator priority: get Sid (Cornwall manager) onto the app as a PWA and let him use it properly. First answer assumed he would review S199 drafts; **wrong**. Corrected framing: Sid signs up FRESH, runs both small companies on the app for real (vehicles, calendar, jobs, agents, quote engine). He is the first real tenant and the F61 probe. The operator, not Sid, reviews S199 drafts and voice pairs.

Second correction: I wrote "drafts back to pre-tuning quality" for a fresh tenant. Also wrong. The 09-24 quality work is global (spine v5, extraction code, S358 timeout, S360/S361 logic, S360 hint migrated to all tenants and seeded, S363 labels seeded). Only business config is S199-specific (pricing, depot, local rules, template copy edits, tone, voice notes + F59 flag). The seed was then checked via MCP: `seed_tenant_defaults` writes Cornwall's v5 figures exactly, so Sid lands on his real pricing and F9 only asks him to confirm. S350 widened to a full fresh-tenant seed audit for strangers.

## 2. Readiness checklist, operator answers

Send-as collision: swap S199 -> new tenant same hour, no issue. Multi-company: investigate (done, section 5). Stripe test keys: Sid signs up with the 4242 card on the top tier; at S264 his row needs a comped edit (tier, status active, Stripe ids nulled) BEFORE live keys flip, with one grep to confirm the predicates read that as active. Invite flow: not an issue, invites arrive. S330: yes. S320: DONE 09-25 morning (my context was stale). S253: still open at session start, done this session. Trello: none. Calendar: Google Calendar OAuth sync not built; route = `.ics` export, parse in chat, insert via MCP. Make pause coincides with mailbox connect. Push on install. Tell Sid nothing; log stalls.

## 3. S253 DONE

OpenAI restored the closed LTD account. Walked through: prepaid $20, auto-reload $5->$20, **Monthly reload limit ON at $120** (the real ceiling in the prepaid model), org spend limit $120 with "Enforce a hard limit" ON, spend alerts at 40% (added) / 80 / 100 to admin@ (the "Email Subject Prefix" field rejects `@`; address goes in "Also send alerts to"). Project "Movers App", key owned by a **service account** not the user. Key swapped in Vercel prod+preview + local, redeployed. Verified: F12(a) panel drafted; MCP showed zero operator alerts and all five crons `ok` at 13:15; usage appeared on the new account after a lag (2 requests, 2,543 tokens).

The verification enquiry was a real Cornwall form submission. Pasting the Message field alone -> escalated `classifier_requested` (panel gives no finer reason -> S378). Pasting the whole form block -> Removals, Template 111, 90%, draft asked for the pickup postcode + move date, did NOT ask to confirm the item list. Make's live reply to the same enquiry asked "is that everything" AND both addresses (delivery was complete). Operator + Sid agree the app's no-confirm on a 15+ piece counted list is right, so S360 B2's rule text (hedge = partial) disagrees with the wanted behaviour -> S377. The postcode ask should have been suppressed by S361 (draft inferred Newquay) -> S376. Operator's wider idea (resolve both addresses like a human would, never ask) -> F62.

## 4. S330 DONE: prod had no backups

Supabase Free = no automated backups. Rather than Pro, nightly `pg_dump` from the operator's Plesk VPS (Ubuntu 24.04, IPv4-only -> session pooler). First SSH for the operator; DB password reset after `pg_stat_activity` showed only Supabase internals. Friction logged as quirks: `.pgpas` typo, a heredoc-stuck shell (Ctrl+C), a wrong-password run that reads "user postgres". Dump 17 MB; cron 03:00; 14-day retention; healthchecks.io dead-man check; restore into local Postgres 17 matched prod 948 / 21. Off-VPS second copy -> S382.

## 5. Multi-company (Codex, read-only)

Path is built end to end: add-on checkbox (Settings -> Billing -> Add-ons, owner, free, off on fresh signup), "+ Add another company" (Settings -> General -> Companies; invisible until the add-on is on), `createSecondCompany` -> full own onboarding + own checkout + own trial, `TenantSwitcher`, `tenant_users` link, `active_tenant_id` cookie. Flags: no consolidated billing (owner seat counted twice); **every extra company got a fresh 30-day trial** (fixed: S375); add-on read as any-membership + loose gate (S379); combined view placeholder; VAT seed 0.1 (S350). Operator direction: £7/mo per additional company as a line on the primary subscription -> DEC60 candidate + F63, post-launch. Trial-abuse signals: card fingerprint is the only reliable one; S380 post-launch, no auto-block; Stripe Radar defaults on at launch (S381).

## 6. S375 DONE

Trial keyed by checkout user. `user_billing_flags` table, service-role only; checkout gates on tenant AND user; webhook stamps the user via `checkout_user_id` metadata; members/invitees never stamped; backfill = 1 row. Migration pasted into chat and applied via MCP before Codex pushed. Two accepted edges documented on the entry. Manual check deferred to Sid's second company.

## 7. Invited members

Onboarding is tenant-keyed, so crew/admins skip the wizard and land on the dashboard cold. No first-run experience exists. S383 (confirm invite-accept landing + owner-invites-before-wizard) and S384 (role-based first-run, with F60). Not gates for Sid.

## 8. Lessons

- Read the docs the operator says were just added before answering readiness questions; two stale calls in one reply cost trust.
- "Tuned for S199" vs "tuned for all tenants": check where the change landed (code/spine/seed vs tenant rows) before claiming either.
- Free-tier infra under a paid product: the absence of a backup is a finding, not a setting to check.
- The prepaid-model money ceiling is the reload limit, not the spend limit. Both on.
- Test panels must be fed the real inbound shape or they test something else.

## 9. State at close

main `a46fc07`. Pre-Sid list cleared. Next: Sid signs up (fresh, top tier, test card), enables multi-company, adds company 2; operator invited as owner; `.ics` calendar import once his tenant id exists. Before his mailbox connects: S376, Postmark cap check, Make pause plan. Docs swapped in this pass: active backlog, launch plan, quirks (new §16), archive append 2026-09-25 v2, this summary.
