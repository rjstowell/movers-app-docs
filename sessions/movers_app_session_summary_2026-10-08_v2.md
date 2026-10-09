# Movers App: Session Summary 2026-10-08 v2 (THE SENDING IDENTITY + ENQUIRY ROUTES SESSION)

Chat spanned 2026-10-08 morning to 2026-10-09 evening, parallel to the Sid walkthrough chat (which pushed its doc pass first as `db9d197`). Numbering read from disk before assigning: this chat took **S478** and **F75** (agreed across both chats on 10-08), then **F78**, **S485**, **DEC64**, and at the doc pass **S486**, **S487**, **F79**. Next free: **S488 / F80 / DEC65**.

## 1. What shipped

| Item | What | Where |
|---|---|---|
| S478 | One sending domain per tenant; unowned Postmark domains deleted and recreated, never reused | `b536b08`, migration `20261008123300` |
| F75 | Sending domain derived from the Business Info email; DNS host detection ("Your DNS is managed at Cloudflare"); phone record fields; guide links in context; honest pending copy; background re-check cron | PR #170 `72cfd65` |
| F78 | DEC64 enquiry routes: business email Yes/No then app/inbox; a healthy mailbox is read in both modes and decides the door; checklist and test enquiry follow the answers; BCC off by default with opt-in copies; booking confirmations in both Sents in full; app-sent tag header | PR #171 `fcc2213`, three migrations |
| S485 | Booking confirmation template cannot be saved until converted (a tag other than `customer_name`); no double greeting on paste | `2e1ebbf`, `6af9959`, `1522d9b`, `0ccd7c0` |
| DEC64 | Locked. Reverses DEC40. | backlog |

**Cornwall is now sending-verified for real.** Records added in Cloudflare at about 16:05, nobody pressed Check verification, the new cron flipped it green at 16:15. First end-to-end proof of the whole F75 chain.

## 2. How it started

Cornwall's sending domain had sat Pending for an hour. Two causes, both the product's fault: the tenant had typed `www.cornwallmovers.co.uk` (the form accepted it; it can never match `hello@cornwallmovers.co.uk`), and the records had been added at IONOS, the registrar, while the live nameservers are Cloudflare. The operator's framing held for the whole session: these are things the code must handle, not things a mover should know.

A read-only Step 0 then found that `sending_domain` and the from-address were two unlinked inputs (3 of 4 live rows mismatched), and, worse, that Postmark domains were looked up by name across the whole account, so a free-trial tenant could inherit another business's verified domain and send as them with valid DKIM. S478 shipped first, on its own.

## 3. Decisions and reversals

- **DEC64 (F78).** Forwarding the business email into the app was designed in the morning, built, and rejected by the operator the same day: one-way, an extra step, no filing, and the Gmail confirmation code would land in the app as an enquiry. Replaced by "connect the mailbox for everyone with a business email, in both modes". The "In the app / In your inbox" toggle stays as the tenant's working preference; it no longer decides whether a connected mailbox is read.
- **DEC40 reversed.** The toggle used to decide the door. Now a healthy mailbox decides it. The DEC40 problem (flipping the toggle did not move the door) cannot recur because the toggle no longer moves the door.
- **BCC rule.** Copies of sent mail are off by default; an opt-in setting with an optional different address that must confirm by link. Found along the way: every reply from the generic sender had been bouncing on its own BCC (`send.moversapp.app` has no mail server), 11 of 11 in Resend over 60 days, invisible in the app.
- **Domain Connect onboarding corrected.** A merged template is not picked up by any provider; each one needs an email. Sent 10-08 to IONOS, Cloudflare and GoDaddy from `admin@moversapp.app` (Gmail send-as over Postmark SMTP, set up during the session). The quirks note from 10-02 ("providers ingest on their own schedule") was wrong.

## 4. What the operator found while testing

- Sid's four DNS notes (guide link prominence, phone scroll, "you can leave this page", how long it takes) all went into F75.
- Cornwall's dashboard checklist vanished while the tenant could not receive a single customer email: the test enquiry had ticked via the Movers App address. That is what became F78.
- Booking confirmations appeared in neither Sent (tab or folder). Folded into F78 and built first.
- A pasted, unconverted booking template could be saved. S485.
- Guided setup is owner-only; the operator's hotmail login is admin on Testing Mover, so `?resume=1` bounced to Agents with no message. Quirk logged.
- The Quoting step on resume shows "Routes and survey rules can't be changed here yet" and locks the options. S487.

## 5. Follow-ups left open

- **Bounce fix unobserved:** no generic-sender reply has gone out since the F78 deploy. Next one: Codex reads Resend and reports delivered or bounced.
- **Cornwall mailbox connect:** the real inbound test for F78's door rule, when Sid is ready.
- **Provider replies** to the Domain Connect emails (S433 amended). Cloudflare was asked whether `redirect_uri` is honoured given `syncRedirectDomain` is unsupported there.
- **S486** proper admin@ mailbox before launch. **F79** inbound mailbox guide, next design session. **S487** onboarding nit.
- Old Postmark domain `8145051` (`www.cornwallmovers.co.uk`) and the two IONOS `www` records: harmless, operator to delete.

## 6. Process notes

- Every brief this session opened with a read-only Step 0 and Codex came back with corrections each time (from-address source, bounce confirmation, greeting edge case). Keep doing it for anything touching sending or inbound.
- Two chats numbering at once worked with the rule: second to push re-reads the line, inserts its header above, edits the line in place, appends its archive block after. Recorded in quirks 27.
- Vercel preview protection blocks cron endpoints; verified on production within one interval instead of adding a bypass secret.
- Codex twice did more than the brief and was right both times (the "already added" guard in S478; no BCC when writeback already filed a booking). Twice its first cut was too thin and the operator caught it (summary-only bookings in the Sent tab; the greeting-only guard).

## 7. State of the world

- main `0ccd7c0`. Production built from it. Migrations applied: `20261008123300`, `20261009120000`, `20261009160000`, `20261009180000`.
- Sending identities: S199 `richardstowell.com` verified; **Cornwall `cornwallmovers.co.uk` verified (10-08 16:15)**; Testing Mover `mail.richardstowell.com` pending (IONOS, S433 test); S146 `test.richardstowell.com` pending.
- Routes: Cornwall, S199, Joe's, S146 `own_email`; Testing Mover reset to null for testing (owner login `richjstowell@gmail.com`); A1A2, Sep, Sep2, Onb null. Copy setting off for all.
- Business Info emails moved for testing: Testing Mover `hello@mail.richardstowell.com`, S146 `hello@test.richardstowell.com`.
- `admin@moversapp.app` sends via Gmail send-as (Postmark SMTP). Provider onboarding emails sent 10-08 14:24.
- Docs: this summary, backlog header + numbering + amendments + new items, archive append 2026-10-08 v2, quirks 27, launch plan rows.
