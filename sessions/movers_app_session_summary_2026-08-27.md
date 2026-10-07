**Movers App — Session Summary 2026-08-27 (S43/S42 part-done · mailbox-model pivot · docs → .md)**

**main unchanged at e3fdbf0 — no code shipped. This was an infrastructure + decisions session.** Stood up the domain and inbound receiving on the owned domain (operator dashboards only), then a long design conversation reshaped the mailbox launch model. Two decisions locked (DEC35, DEC36), one feature raised (F23), one small item (S218), F15 writeback promoted to launch-critical, F15 Problem B race guard demoted to post-launch. Doc process changed to Markdown with surgical edits.

# What got built (infra, no code)

**S43 Part A — DONE.** moversapp.app added to Vercel: apex A record (216.150.1.1, from Vercel's panel) + www CNAME, both **Valid**, HTTPS confirmed. Apex 308-redirects to www (Vercel default) — canonical apex-vs-www choice is an F21 concern (the privacy-policy URL handed to Google). Replaced Porkbun's parking ALIAS on the apex; killed the recurring wildcard.

**S43 Part B step 1 — DONE.** Resend receiving on a dedicated subdomain **inbound.moversapp.app** (deliberately separate from send.moversapp.app — inbound MX and the sending feedback MX cannot share a host). MX inbound-smtp.eu-west-1 (priority 10, sole MX on its host), DKIM added. **Enable Sending toggled OFF** on this domain (receiving-only) — that was required to clear domain verification: Resend was gating Verified on the sending DKIM even though we only want receiving. Verified end-to-end on Testing Mover: email received in Resend + account-level email.received webhook returned 200 {"ok":true}. **No app row created** — because the tenant's stored inbound address is still @chualexua and the handler matches the full domain; that is exactly what step 3's UPDATE will fix. Expected, not a fault.

**S43 Part B step 3 + S42 — DEFERRED (on purpose).** The env flip (INBOUND_EMAIL_DOMAIN → new domain) + one-off UPDATE over tenant rows was NOT done, because the design conversation (below) is about to change the inbound address model (hide it, mode-gate it). Rewriting rows now = rewriting an about-to-change model. **Old @chualexua route left LIVE as a grace net; nothing half-migrated, no enquiry at risk.** S42 webhook is account-level and already fires on the new domain, so no repoint is needed to receive; any branded-URL repoint rides with step 3. Both fold into next chat's F23/writeback work where the address model is decided.

# The pivot — how the mailbox model got reshaped

Started from "does the inbound UUID address confuse non-technical tenants." Reasoning chain:

1. **The inbound address and "auto-send" are not two things.** The address is just the way in; auto-send is what the app is allowed to do with an enquiry, which depends on how it arrived. (Correction made mid-session — earlier framing conflated them.)
2. **Sending is independent of the inbound address.** The app receives via mailbox sync, drafts, and sends via the sending domain — the inbound letterbox is not involved in sending. So mailbox-only tenants need no letterbox at all.
3. **Why mailbox was held review-only = the race**, not a sending limit: when the app reads the same inbox the tenant reads, both can reply to the same customer (double-reply). A chosen gate, not a technical wall.
4. **What actually removes the race is separation, not a Sent-check.** Make is safe because it MOVES handled mail out; the human only works the escalated pile; app-handled and human-handled are disjoint. That's **folder writeback**, not the Sent-folder race guard. This was the key correction — the guard was over-weighted.
5. **Residual window** (fresh enquiry before the app files/sends it, ≤ sync tick + auto-send delay) is narrow, Make has it too, tolerated. Mitigated cheaply by a short inbox-only auto-send delay + a copy pointer near the timer (S218). The Sent-folder guard only shrinks the final seconds and carries its own fail-open hole, so it drops to post-launch insurance.
6. **Single-mode model:** don't run both inbound paths at once (duplicate ingestion). Tenant picks app-only OR inbox-only via an onboarding survey ("how do most enquiries reach you?" → direct email / website form; social = future, no integration built) and a big toggle that gates which config shows — driving-vision alignment (kill the config junction).

# Decisions locked

- **DEC35 — mailbox auto-send is IN launch scope** (supersedes DEC31's post-launch parking). Made safe by writeback separation, not the Sent-folder guard. Problem B race guard demoted to post-launch insurance for the residual window.
- **DEC36 — single-mode inbound model.** App-only OR inbox-only per tenant; survey + big toggle; config gated by mode; raw UUID address never shown to tenants. Guardrails: per-mode preconditions (guide/block when unmet), clean mode-switch teardown, loud "mailbox stopped syncing" signal for inbox-only (dashboard Errors & Health isn't enough — inbox-dwellers don't visit it; ties to F14 push). Onboarding placement: early but, given F9 is near-shell, a first-run settings choice with a default at launch; Manager-Agent plain-language version post-launch (F6).

# Items raised / re-scoped

- **F23 — single-mode inbound system (survey + toggle + mode-gated config). LAUNCH-BLOCKING.** Implements DEC36. Depends on the writeback slice.
- **F15 writeback slice — PROMOTED to launch-critical.** The separation that makes inbox-only safe + the two-pile "needs-me / done" view. Prereq F11(d) threaded send. Landmine: must not move mail into Cornwall's Make folders (INBOX.Charlie Replied / Harlene's feed).
- **S218 — inbound-only auto-send delay policy + copy pointer.** Small. The cheap residual-window mitigation.
- **F15 Problem B (Sent-folder race guard) — DEMOTED to post-launch insurance.** Build only if usage shows the window bites; Make's execution logs already measure the window with no code.

# Push notifications (raised, not scoped)

PWA push is possible: Android/Chrome/desktop work well; **iOS only if the tenant adds the app to the Home Screen** (no push from a plain Safari tab), so adoption isn't guaranteed. Not built — F14 (PWA) is its home. Re-check iOS behaviour when F14 comes up (Apple moves the goalposts). This is the "louder channel" DEC36 wants for the mailbox-sync-failed signal, but can't be the only channel.

# Process change — docs are now Markdown, edited surgically

Root cause of the recurring end-of-session file-type friction: the ".docx" files were never Word documents — they are plain Markdown text with a .docx extension (the project normalises uploads to markdown and renders that). So:

- **All these docs are now .md.** Extension matches content; the project renders them formatted (headings, bold) exactly as before.
- **Living docs (backlog, launch plan, DECs-in-backlog): edited SURGICALLY** — copy the real current file, str_replace only the targeted sections, everything else passes through byte-identical. **Never full-rewritten.** (This session's backlog swap-in changed 4 lines of 777 on purpose; the other 773 are byte-identical — verified.)
- **Session summaries: new .md file each time, additive.**
- **Readability:** view rendered (project / any markdown viewer). For a polished file to hand an outsider, export .md → .docx/PDF on demand only.

Rule to paste into the project Working Rules / instructions:
> Session docs are Markdown (.md). Backlog + launch plan + DEC log: edit surgically (targeted replaces on the real file, never regenerate the whole doc), swap in. Session summaries: new dated .md, additive. Readability via rendered markdown; export to docx/PDF only on demand.

# Quirks that bit (>5min each — for the quirks doc)

- **Porkbun wildcard *.moversapp.app → pixie.porkbun.com recurred** and hijacks undefined subdomain lookups until deleted (S209 repeat).
- **Resend gates domain-Verified on the sending DKIM even when only receiving is wanted** — fix is toggling Enable Sending OFF so only the receiving MX is required.
- **A 550 "mailbox unavailable" bounce was Resend rejecting because the domain was still Pending, NOT DNS/cache** — misdiagnosed as a Gmail negative-cache issue mid-session; DNS was fine (whatsmydns green). Corrected.
- **The /mnt mount snapshots project files at chat start** — files swapped in mid-session don't reach the file reader until a fresh chat. (Caused the backlog to read as stale/missing until re-added; only a new chat fully resyncs.)

# Numbering

Next free S-number **S219**, next free F-number **F24**, next free decision **DEC37**. New this session: S218, F23, DEC35, DEC36. Orphan migrations unchanged (S130 still blocks db push); no migration this session.

# Where the launch stands

Mailbox auto-send is now a launch feature, which adds F23 + the F15 writeback slice + F11(d) to Tier 1 and removes the Sent-folder guard from launch (net +1–2 sessions vs the 08-26 estimate). S43 Part A + B-step-1 are banked and permanent; S43 step 3 folds into the F23/writeback work. **Next chat opens on F23 + the F15 writeback slice** (single-mode system + the writeback that makes inbox-only safe), with S43 step 3 folded in once the address model is set.
