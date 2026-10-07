# Movers App — Session Summary 2026-09-17 (covers 2026-09-16 afternoon → 2026-09-17 afternoon)

**Headline:** the app became a phone app. F14 (installable PWA) and F54 (web push, four triggers) shipped and were verified end to end on a real iPhone; S248 gave failed sends a reason and a Retry; F35 added the Sent tab; S267 closed the escalated blocked-send dead end; the S-batch cleared every small launch-list item plus eleven polish fixes from the operator's mobile play-through; 15 test tenants were deleted. main `6877c34` → `a1b29b0`. Three migrations applied by hand (F54, S248, none for the rest).

**Numbering after this session:** next S = **S315**, next F = **F56**, next DEC = **DEC56**. DEC55 locked.

---

## 1. Launch-readiness review (start of session)

Read the launch plan, driving vision and master backlog and filtered to in-app work. Remaining before paying customers, grouped:

- **Money + keys go live:** S253 OpenAI hard cap (see correction below), `TENANT_DAILY_TOKEN_CAP` unset in prod so S252 is inert, S266 per-tier caps, S264 Stripe TEST→LIVE + live webhook, `BILLING_CUTOFF` reset.
- **Deliverability for strangers:** S265 auth SMTP, S244 Postmark bounce/complaint webhook, S243 Platform plan, F12(b) real-email inbound test.
- **Correctness holes:** F52 reconcile provisioning, S254 Half B seat-band downgrade, S248, S267, S201, S158, S208, S77, S165.
- **Optional pre-launch:** F21 slice 3, F53, S140.

F11(d) contradiction resolved: DONE 2026-08-27 v2 (PR #67 `baf6fc5`); the launch-plan row was stale and is now corrected.

**S253 correction:** not blocked on the Monzo card (it arrived). OpenAI closed the newly opened LTD account after repeated failed card attempts. Prod `OPENAI_API_KEY` is on the operator's personal account, ran dry 2026-09-15 (silent 429s), topped up $10. Path: re-register LTD account → hard cap → migrate key. Operator will do this before paying tenants.

**Manager trial verdict:** ready with guardrails. Prod Stripe is still TEST mode, so a normal signup with card `4242…` is the free sandbox. Two-tenant shape: cold signup on a fresh tenant (app-only mode, never the real Cornwall mailbox) + invite to S199 Mover (review-only, never Approve, Make still replies). Prod key confirmed on the personal account.

## 2. F14 — installable PWA (PR #122 `7ab8885`)

- Owner play-through in Safari with a no-back-button rule found nothing stranded (X buttons + burger nav cover the shell pages). Codex discovery then audited shell-less routes and found four dead ends: `/signup` submitted state, `/auth/callback` error, no `not-found.tsx`, no `error.tsx`; plus no logout on `/onboarding`, `/onboarding/checkout`, `/profile/complete`. All fixed with plain links.
- Manifest via `app/manifest.ts`, `webmanifest` added to the middleware matcher exclusion, icons generated from `moversapp_avatar_blue.png` (rounded "any" 192/512 + full-bleed maskable 192/512 + 180 apple-touch-icon, no transparency on the latter two), `public/sw.js` = skipWaiting + claim + no-op fetch handler (kept for old Android Chrome; Chrome warns it adds overhead — S314c), production-only registration, Apple meta added explicitly because Next now emits `mobile-web-app-capable` instead of the `apple-` prefixed tag.
- Verified: installed on iPhone from the preview then from prod, DevTools manifest + SW activated, Chrome shows Install. Android verification pending the manager's phone.
- Known platform limitation: iOS opens web links tapped in other apps in Safari, never in the installed PWA.

## 3. F54 — web push (PR #123 `5f706e6`, PR #124 `ce5262c`, `f710b76`, `e3f5dbb`)

- **Slice A:** tables `push_subscriptions` (endpoint-unique, per device), `user_notification_prefs` (per user, four booleans), `push_events` (unique on `agent_run_id + event_type`); VAPID keys in Vercel env; `sw.js` push + notificationclick (focus-or-open at `data.url`); `/api/push/subscribe` (POST/DELETE) and `/api/push/test`; `lib/push/send.ts` with 404/410 cleanup and never-throws; Notifications card in Settings › Profile (per-device Enable/Disable, four toggles saved via a server action following `updateProfile`, Send test, iOS-not-installed hint).
- **Slice B:** `queueNotifyEvent` at the four finalisation points (escalation in `pipeline.ts`, review hold in `pipeline.ts` gated to genuinely-held drafts, send failed ×2 and auto-sent in `auto-send/route.ts`), `waitUntil`-wrapped, insert-first idempotency; `.select("id").single()` added to the review-queue insert for the deep link; recipients = owner + admin, active.
- **DEC55:** recipients resolve by `tenant_users` membership, then all subscriptions for those users; `push_subscriptions.tenant_id` is informational. Found because the morning's push went nowhere: one phone = one endpoint, and the row records whoever last pressed Enable. Card now re-POSTs the existing subscription on mount so the row follows the logged-in user.
- **Copy:** titles "New draft ready for review" / "New escalated enquiry" / "A reply failed to send" / "Your agent replied to an enquiry"; body line 1 `{full name} · {category label}` (label from the same source as the inbox badge; falls back to sender email), line 2 = first 50 chars of the stripped enquiry at a word boundary (send_failed: the failure reason instead). Never "MoversApp" in a payload (iOS adds "from MoversApp" itself). 14 unit tests on the body builder.
- **Verified on iPhone:** test push; review hold; auto-sent; send failed (forced by setting the run's `from_address` to `not-an-email` during a 10-minute grace window, then restored); escalation ("Damaged wardrobe" complaint); toggle-off suppression. Latency ≈ inbound + classification (~1–2 min), push itself sub-second.

## 4. S248 — failed-send reason + Retry (`9608a9f`, sort `a63f48f`)

Discovery: only raw `agent_runs.error` existed; `approveQueueItem` refused any row with a resolution; failed rows hid the whole action block while system-health copy told users to "send it again". Built: migration `agent_review_queue.send_failure_code/reason` (by hand); shared `sendFailure.ts` classifier (sender_unverified / no_sender / provider_error / other; consent + billing stay holds); worker writes code + reason; approve path allows retry from `auto_send_failed` and returns "already resolved" distinctly; Review Queue shows red "Failed to send" + reason + "Retry send"; push line 2 = reason; failed rows now sort with open drafts in "Needs action". Verified live: push → tap → reason → Retry → sent, email received.

## 5. F35 — Sent tab (PR #125 `32759d0`, chips `b592f3d`)

Fourth Inbox folder after Filtered, no count badge. `sentData.ts` merges three queries in code (queue rows with `sent` / `edited_sent` / `auto_sent`, plus `agent_runs` with `manual_reply_sent_at`), prefixed ids `q-`/`run-`, 200 cap, newest first, realtime refresh. Chips: Agent / Approved / Edited & sent / Escalation reply (the "You · approved" form was dropped as noise). Read-only detail: reply on top, original enquiry below. Escalated-reply subject is reconstructed `Re: <original>` because it is not stored → S313. Verified on preview desktop + phone; Escalated tab untouched.

## 6. S267 — escalated blocked-send exit (`e276838`)

The Escalated composer showed "This reply can't be sent yet. Choose from the following:" with nothing to choose. Extracted the Review Queue's inline block JSX into `SendingIdentityBlockNotice` (message + Verify your domain link + Accept generic sending modal), used by both tabs; composed reply retained; `genericFromAddress` threaded into EscalatedTab; the two no-flag block shapes now carry the verify flag. Verified by flipping `autosend_consent_ack` off via SQL, seeing the notice, accepting via the modal, sending, and finding it in Sent as "Escalation reply".

## 7. Polish + hardening batches (nine + three + two commits, direct to main)

- S300 Edit+Send scrolls to, glows and focuses the textarea. S301 "Video Quote lives in the Quote page" card removed. S302 Quote pill centred (mobile override was `flex-start`). S303 "Parse list" → "Convert". S304 dashboard inbox icon is a keyboard-focusable link; card growth was the widget footer's safe-area padding, moved to `.dashboard-shell`. S305 Jobs Move menu contrast pinned.
- S306 share link `NEXT_PUBLIC_APP_URL ?? location.origin` (env set in Vercel Production only). S307 System Health: single failure deep-links to the row, several → failed-first sort; "Processing enquiries" href fixed. S308 Quote Engine AdminCard **deleted**: it was ungated (every tenant user could read the three LLM prompts behind a client-side `"cornwall"` passcode), duplicated Settings, and its Save was a stub. Prompts + `SET_PROMPT` plumbing stay as the seed for F31.
- S309 stuck runs: `received` older than 30 min are swept on the auto-replay tick into `error` with bucket `transient` (so Retry appears), health copy "N need retry". Verified: a run set back to `received` an hour old was swept and then re-driven by auto-replay on the next tick without any tenant action (self-heal), no duplicate push (idempotency guard).
- S310 Settings tab race: root cause was `settings/loading.tsx` on the redirect-only index letting the tab bar render before the server redirect resolved; boundary removed, comment left. S311 day/week calendar cards use full event colour with contrast text; month keeps dots.
- S158 hardcoded cookie-secret fallback → throw (chain: `ACTIVE_TENANT_COOKIE_SECRET` → service-role → anon); operator still to add the dedicated var. S208 all `from_email` reads removed. S77 crew dashboard errored on every login because `getCrewJobs` filtered by a non-existent `assignee_id`; filter dropped, crew see the tenant's upcoming week (verified live). S201 checked: hole closed. S165: 15 zero-activity tenants deleted by hand; 7 with history kept; Testing Mover has an active crew (`cornwallselectmotors@gmail.com`) and a pending admin invite for future role testing.
- `docs/investigations/` gitignored.

## 8. Operator actions this session

- Vercel env: `NEXT_PUBLIC_VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT=mailto:admin@moversapp.app` (prod + preview); `NEXT_PUBLIC_APP_URL=https://my.moversapp.app` (prod only). **Still to add:** `ACTIVE_TENANT_COOKIE_SECRET`. Flag `VAPID_PRIVATE_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_SECRET_KEY` as Sensitive (Vercel "Needs Attention").
- Migrations by hand via MCP `execute_sql`: F54 (three tables + RLS), S248 (two columns). S130 ledger unchanged.
- DB hand-edits for testing, all reverted or intentional: Testing Mover `autosend_consent_ack` (restored via the in-app modal), `from_address` on one run (restored), `send_after` pulled forward once, a July pipeline-error run and two stale `received` runs marked resolved, "Damaged wardrobe" run reset to `received` for the S309 test (self-healed).

## 9. Open threads / next

- **Next build session: F31 phase 1** (operator console): allowlist env var, `/admin`, read-only tenant list, DB-backed prompts tab. Design pass first.
- Then F55 (avatar vs logo uploads), S312 (`absoluteUrl()`), S313 (`manual_reply_subject`), S314 (F14 leftovers), F52, S254 Half B, S244.
- Launch tails unchanged and operator-owned: S253 (re-register LTD OpenAI), S264, S265, S243, `TENANT_DAILY_TOKEN_CAP`, `BILLING_CUTOFF`.
- Manager trial: install from `my.moversapp.app` on his Android (Chrome Install), log in inside the app, enable notifications in Settings › Profile. Android proves the no-op fetch handler question (S314c).
