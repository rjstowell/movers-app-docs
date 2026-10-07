# Movers App — Session Summary 2026-10-02

**S428 DONE (existing-user invite notification) · S265 DONE (Supabase auth sender off the Cornwall domain) · Postmark hygiene · new S429-S432.**

Side session, started the evening of 10-01 and finished the morning of 10-02, run while PR #161 (F69 Domain Connect) was open in another chat. Both ships went direct to main from a fresh checkout; no F69 files touched. Numbers taken here: S428, S429, S430, S431, S432. No F or DEC. main `790c2e9` -> `8550bfd`.

---

## 1. What started it

Operator invited `rjstowell@hotmail.co.uk` to Testing Mover and S199 Movers to reach every test account from one login, and got no email. Same address had also been added to Cornwall Movers Ltd by Sid the day before, also with no email.

Read-only DB queries sorted it in one pass:
- The address had been an auth user since 2026-09-10 (Testing Mover crew invite, never accepted, row still `pending`).
- Cornwall: existing-user path, row inserted `active`, nothing sent. By design, and the gap.
- Testing Mover: already a member, early return, no message to the inviter.
- S199: no row; the invite had gone to a mistyped gmail address.

So one real product gap (existing users are added silently) plus two operator slips. The gap became S428.

## 2. S428: notify existing users on invite

**Investigate gate paid off.** Brief assumed Resend; Codex's read-only pass found every system email already goes through Postmark (`sendPostmarkEmail`, `admin@moversapp.app`) and that `lib/resend/sending.ts` only holds the tenant reply path. Transport switched to Postmark before any code was written.

**Shipped `aa04c39`:**
- `lib/team/notifications.ts`: `sendAddedToCompanyEmail`, Postmark, never throws, logs + `{ ok: false }` on failure.
- `inviteMember` existing-user path: insert first, email second, inviter sees "We have emailed them." or "The notification email could not be sent."
- Already-member returns are now specific: active -> "already a member"; pending + unconfirmed -> "invite already pending, use Resend"; pending + confirmed elsewhere -> row activated + emailed (keeps the old role, S429).
- Unconfirmed invitees get a "use Forgot password on the login page first" line.
- 12 new tests. No existing `inviteMember` tests existed before.

**Shipped `8550bfd`:** `lib/postmark/systemFrom.ts` with `SYSTEM_FROM` = `"MoversApp" <admin@moversapp.app>`, used by the team notice and both support emails. Separate file because tests mock `@/lib/postmark/sending` wholesale.

**Verified live** across two Hotmail inboxes: confirmed user added to S199 + Testing Mover (email + active rows, DB-confirmed), already-member message with no send, unconfirmed invitee got the Forgot-password line, From reads `MoversApp <admin@moversapp.app>`.

**False failure on first verify:** the tab had been open since before the deploy; Vercel skew served the old build (old success copy, nothing in Postmark Activity at 08:12). Second bite of the S417 quirk. Hard refresh before every prod verification.

## 3. S265: auth mail was going out as cornwallmovers.co.uk

Found by accident while comparing junk placement: the Supabase invite email arrived from `Movers App <noreply@cornwallmovers.co.uk>`. S265 had been logged as a junk problem with Supabase's default sender blamed. Wrong on both counts. On 2026-07-10 Supabase Auth was pointed at Resend SMTP to lift the 2/hour cap, Cornwall was the only verified Resend domain, and every confirmation / reset / invite since had carried it. Cross-company domain leak sitting in the go-live polish bucket.

**Fix, config only, operator:**
- Postmark: sender signature `noreply@moversapp.app` (MoversApp), signature name on `admin@` fixed from the operator's personal name to MoversApp, dead unverified domains removed, Return-Path CNAME `pm-bounces -> pm.mtasv.net` added at Porkbun and verified.
- Supabase Auth SMTP: `smtp.postmarkapp.com`, 587, Server API Token as both username and password, sender `MoversApp <noreply@moversapp.app>`. One typo caught (`smtp.postmark.com`).
- Verified live: new-user invite and password reset from the new sender. Signup confirmation unverified (same SMTP, different template); check on the next fresh tenant.

Postmark chosen over verifying moversapp.app in Resend: zero DNS work, matches all other system mail, keeps Resend's domain slots for tenant sending. Resend verification of moversapp.app belongs to S209 when the generic sender is built.

## 4. Hotmail junk: do not chase

With DKIM, Return-Path and DMARC all passing, Hotmail still junked first-contact mail from the domain, inconsistently (invite junked, reset delivered, S428 notice junked once and delivered once in the same inbox). Young .app domain, near-zero volume. Levers: branded HTML (S430), DMARC `rua` (S431), volume, time. Nothing else worth session time.

DMARC today: `v=DMARC1; p=none;`, no reporting. SPF is Porkbun's default include. Both fine until S209 puts Resend on the domain.

## 5. New items

- **S429** stale-pending activation should apply the chosen role (tiny).
- **S430** branded HTML template for Postmark system mail + Supabase auth templates (polish, deferred since July).
- **S431** DMARC `rua` reporting now, `p=quarantine` only after S209 aligns Resend (operator).
- **S432** `.gitattributes` line-ending normalisation; `notifications.ts` landed CRLF.

Unnumbered: push/toast variant of the S428 notice for a user already signed in (edge case, revisit if multi-company users become common); verify signup confirmation on the new SMTP sender at the next fresh tenant.

## 6. Process notes

- Two chats open at once worked. The F69 chat published its numbering (S428 up, F70 up, DEC62 up) before this chat briefed; this chat took S428-S432 and recorded F69/DEC61 as reserved by the other session in the numbering line. The other chat records F69 + DEC61 at its own doc pass.
- Read-only MCP queries against `auth.users` + `tenant_users` turned a "strange bug" into three plain facts in two calls. Cheap, do it first.
- "Verified" needs the provider log as ground truth when the UI copy is the only visible difference. Postmark Activity settled the skew question in one look.
- Codex's step-0 investigate gate changed the transport before a line was written. Keep it on every brief that names a library or service.

## 7. Test accounts after this session

`rjstowell@hotmail.co.uk` (user `73f3015b-7312-4557-b376-daafa583c8b0`): owner Cornwall Movers Ltd, admin S199 Movers, admin Testing Mover. Second multi-company login alongside `richjstowell@gmail.com`. The wrongly-invited `rjstowell@gmail.com` row on S199 was removed (sole membership, auth user deleted).
