# Movers App: Session Summary 2026-10-02 v3 (PUSH LIFECYCLE + LOGOUT SCOPE)

Chat opened 2026-10-01 evening on a bug report, paused while two other chats finished (S428 invite-notify, F69 Domain Connect), resumed 2026-10-02 afternoon. Numbering read from the active backlog's 10-02 v2 line before assigning. This session took **S441 to S447**. Next free: **S448 / F70 / DEC62**. No new DEC; DEC55 amended.

## 1. What shipped

- **S441 DONE.** Push subscription row follows the login. PR #162 squash-merged as `20ee09a`. No migration, 9 new tests.
- **S444 DONE.** Logout scoped to this device + "Sign out of all devices". Direct to main `3ac636e`, follow-up `9bef57b`. No migration, 8 new tests.
- main `88757cd` -> `20ee09a` -> `3ac636e` -> `9bef57b`. Production built from `9bef57b`.
- Codex working tree: the uncommitted `docs/domain-connect/PR_DESCRIPTION.md` draft from the F69 chat is now `stash@{0}` "domain-connect PR_DESCRIPTION draft (waiting on IONOS)". `git stash pop` when S433 resumes.

## 2. The bug report and what it actually was

Operator, on the iPhone PWA logged in as rjstowell@hotmail.co.uk (owner of Cornwall Movers Ltd only; Cornwall has no inbound wired), received a push about another tenant's enquiry. Read as a cross-tenant leak needing a guard.

Three MCP queries settled it before any code was read:

- `push_subscriptions` had two rows. The iPhone row was created 2026-09-17 (F54 build day, phone then logged in as richjstowell@gmail.com) and `last_used_at` was 19:38 UTC on 10-01.
- rjstowell's memberships: Cornwall owner/active, Testing Mover crew/**pending**. Pending crew is correctly excluded from fan-out.
- Every `push_events` row that day was **S199 Movers** (not Testing Mover as assumed); the last one an escalation at 19:04 UTC. S199's owners are richjstowell + richsmap05+s199.

So: the row still named richjstowell at 19:04 (owner of S199), the push went to that user's one device, and at 19:38 the row re-keyed when Settings > Profile was opened. DEC55 fan-out was right. The gap was device lifecycle: the Notifications card mounting was the only thing that ever re-keyed a row, and logout never touched it. Any shared or handed-over device kept the previous user's pushes until someone opened Settings.

Lesson recorded: "cross-tenant" was the wrong frame. Pushes target users, never tenants; the question was "which user does this device row name".

## 3. S441 build

- Brief with a read-only Step 0. Codex found: three sign-out paths (ProfileForm, `/logout` route reached only by plain anchors from onboarding/checkout/profile-complete, DeleteAccountSection); no shared push helper, everything inline in NotificationsCard; POST already upserted on endpoint and overwrote `user_id`; DELETE matched endpoint + `user_id`, so a logout-as-B could never clear A's row; `last_used_at` written only by POST; `app/(app)/layout.tsx` is the authenticated shell. I checked the `auth.users` FK: `ON DELETE CASCADE`, so Delete account needed nothing.
- Built as specced: `lib/push/client.ts`, endpoint-only DELETE, device row released before `signOut` (ProfileForm + new `LogoutLink` for the three anchors), `PushRegistrationSync` in the shell.
- Two Codex deviations, both accepted: `getRegistration()` over `serviceWorker.ready` (`ready` hangs with no SW); Disable swallows a DELETE failure and unsubscribes anyway. One Codex addition (clear `sessionStorage` markers on logout) was reasonable for local logout and wrong for remote sign-out; removed under S444.
- My scope item 3 ("refuse at send time if the user is no longer a member") was dropped as redundant: fan-out already goes tenant -> members -> rows, so a non-member's row is never selected. Replaced by the endpoint-only DELETE, which was the real hole-closer.

## 4. S441 verification

Preview could not be tested on the installed PWA (bound to prod origin; iOS push needs install). Desktop Chrome on the preview URL instead, rows checked via MCP after each step:

1. Enable as richjstowell -> Chrome row (after allowing in the browser; first attempt showed "Not enabled" because permission had not been granted).
2. Sign out -> row gone.
3. Login as rjstowell, Dashboard only -> new row under rjstowell. Shell re-key proven. Side effect: login routed through `/profile/complete` and accepted rjstowell's pending Testing Mover crew invite ("Welcome to Testing Mover"); left in place.
4. Send test as richjstowell from Opera (incognito refuses notifications) -> Opera only.
5. Send test as rjstowell from Chrome -> Chrome (toast swallowed by Windows, found in the notification centre) + iPhone (same user), Opera silent.
6. Offline logout: unrunnable, `signOut` needs the server and the button spins until reconnect (S446). Property covered by step 3 + the POST-overwrite unit test.

Prod: iPhone PWA found already signed out (see section 5); login as richjstowell, Dashboard only -> the 17 Sep row flipped to richjstowell at 15:18 UTC.

## 5. S444: "deploys log me out" was desktop logout

Operator: the PWA seems to sign out on every prod push, which would be a trial killer. Pulled Supabase `auth_logs` via MCP before guessing.

- Nothing around the 15:08 deploy. The only event 15:05 to 15:18 was the 15:18 login.
- 2026-10-01 21:53:33 `logout` rjstowell from the desktop IP, 11 s later a login as richsmap05+s199 (the other session that night), then 22:00:42 two `Session not found` 403s from Vercel server IPs: middleware on the phone finding its session dead. Same pattern at 19:57 -> 20:14 -> 20:14:56 phone re-login.
- Supabase `signOut()` defaults to `scope: 'global'`. Codex grep: all three call sites passed no scope.

Fix: `scope: 'local'` on the two normal paths; Delete account stays global. Operator asked for "Sign out of all devices" now rather than later; agreed, with one addition: it must delete all the user's push rows, since a row is not tied to a session and signed-out devices would otherwise keep receiving pushes. Built as `lib/auth/logout.ts` with `signOutThisDevice` / `signOutAllDevices`, `useConfirm` dialog, `/api/push/subscribe` DELETE body `{ scope: "user" }`.

Prod verification (iPhone + desktop Chrome, both richjstowell): desktop Sign out -> phone stays in (PASS); desktop Sign out of all devices -> phone lands on /login (PASS); zero push rows for the user, which also removed the Opera preview row (PASS); phone login, Dashboard only -> **no iPhone row (FAIL)**.

## 6. The step 4 failure and the follow-up

Hypothesis: the shell sync's `sessionStorage` `push-sync:<userId>` guard. The phone never ran a local logout (it was revoked remotely), so the marker from the 15:18 login survived and the same user's next login skipped the POST. Operator asked whether this was "fairly sure" or "the only thing it can be". Fairly sure. One discriminating check without code: open Settings > Profile on the phone. Card said Enabled and a row appeared on mount, so browser subscription and POST path were healthy and only the shell sync had not fired. Cause confirmed.

Fix `9bef57b`: guard removed, `PushRegistrationSync` POSTs on every mount (ref per `userId` for StrictMode), marker-clearing code deleted, remount regression test added. Re-run with a clean start (desktop Sign out of all devices, phone login, Dashboard only): iPhone row back at 16:14:56 UTC. PASS.

Lesson: a client-side "already did this" marker cannot see server-side session changes. Anything that must re-run after a login cannot be gated by state that survives a remote revocation.

## 7. Operator friction worth remembering (all in quirks section 23)

- Push subscriptions are per origin; preview rows receive real pushes until deleted (Chrome preview row deleted via MCP at session end).
- Chrome incognito refuses notification permission; Windows can swallow Chrome toasts.
- Supabase auth log `referer` is the Site URL for every request; cannot distinguish prod from preview by it.
- `last_used_at` means last registered (S445).
- Do not design an offline logout test.

## 8. Open after this session

- **S442** tenant name in the push payload (small; multi-tenant owners cannot tell which business).
- **S443** deep link when the notification's tenant is not the active tenant (small; check what happens today first; same shape as S434).
- **S445** rename `last_used_at` -> `last_registered_at` (migration by hand, low).
- **S446** offline logout should fail fast with a message (low, pre-existing).
- **S447** F57 customer-PDF snapshot failing on main; refresh it so clean main reports 0 failures (low, noisy).
- Test fixture note: rjstowell@hotmail.co.uk is now active crew on Testing Mover and has a company switcher.
- Launch critical path unchanged.
