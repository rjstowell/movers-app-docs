# Movers App: Session Summary 2026-10-02 v2 (THE DOMAIN CONNECT SESSION)

Chat spanned 2026-10-01 evening to 2026-10-02 afternoon. Ran alongside two other chats (the S428 invite-notify session and a sticky-save-bar continuation). Numbering read from the active backlog's 10-02 line before assigning: this session took **F69**, **DEC61**, **S433 to S440**. Next free: **S441 / F70 / DEC62**.

## 1. What shipped

- **F69 DONE.** Domain Connect automatic DNS setup for the sending identity. PR #161 squash-merged to main as `88757cd` (no migration, 37 new tests). Production has the code; the button stays hidden there until a DNS provider ingests our template.
- **Upstream template merged.** `moversapp.app.email-sending.json` accepted into Domain-Connect/Templates as PR #2108 within about an hour of opening (approved by pawel-kow, merge queue, 12:37 UTC). All bot checks green first time.
- **DEC61 LOCKED.** Domain Connect signed template is the auto-DNS route. Entri and a browser-agent plugin rejected.
- **Operator infrastructure:** RSA key pair generated, public key published as two TXT records at `_dck1.moversapp.app` (Porkbun, verified by `--check`), private key + state secret on Vercel Production and Preview, Preview-only `DOMAIN_CONNECT_SKIP_TEMPLATE_CHECK=1`, keys in password manager, local key folder deleted.
- **Testing Mover** sending identity switched from `yourcompany.co.uk` to `mail.richardstowell.com` (row deleted via MCP, re-added through the card; pending; generic consent not re-accepted).

## 2. The design conversation (why this exists)

Sid's signup stalled at DNS + inbox setup. Operator's read: busy movers see DNS records and IMAP settings, do nothing, the 30-day trial dies. Agreed framing:

- **Two different leaks.** Sending identity (DNS) is optional: the generic sender `send.moversapp.app` already works, so the leak is "looks broken". The inbound mailbox is the real gate: no mailbox, nothing arrives, product does nothing.
- **Browser plugin with an LLM driving the registrar UI:** possible, wrong. Blast radius on the tenant's live website/email DNS, extension-store overhead, no mobile, 70-85% reliability still needing a human.
- **Entri:** the commercial drop-in that does exactly this. $249/mo starter. Rejected.
- **Domain Connect:** the open standard underneath Entri. Free, IETF draft, supported by IONOS, GoDaddy, Cloudflare, Plesk, WordPress.com, NameSilo. Build our own: one template JSON + discovery + signed redirect + callback.
- **Five-layer help stack, tenant picks:** (1) F69 automatic, (2) F20 guided steps, (3) S437 "search how to do this at <registrar>" with a nameserver hint (operator pushed back on my dismissal; accepted: zero maintenance, long tail, opens in native browser), (4) S438 send-to-your-web-person email, (5) ask us / concierge during hand-held outreach. Per-registrar videos rejected: stale within months.
- **Mailbox side mirrors it:** F15 adapters 2/3 (Gmail/Microsoft OAuth, already planned) are the big lever; S439 host autodetect for IMAP tenants; S438 mailbox variant; S440 nudge sequence. Start Google restricted-scope verification early, it is calendar time.

## 3. How the build went

- Brief handed to Codex with a read-only Step 0. Step 0 came back with six corrections, all accepted: spec TXT key format (not DKIM syntax, my error), spec example as a verify-only test vector, template rules (prefixed variables, `k=rsa; p=%dkimkey%` like the wavelinker.io Postmark template, editor test links mandatory), new `DOMAIN_CONNECT_STATE_SECRET`, Testing Mover to `mail.richardstowell.com` (its `yourcompany.co.uk` placeholder is someone else's real domain; `richardstowell.com` is already verified so the button would never show), shared `reverifySendingIdentity`.
- Codex built, tested with a temporary harness page (deleted before commit), added the Preview-only skip flag so the button could be exercised before provider pickup, committed locally, pushed and opened PR #161 only when told.
- Verification on preview: button appeared ("Set up automatically with IONOS"); click landed on IONOS `/sync/404` (template unknown, expected); control run with an approved unsigned Mailjet template reached the IONOS consent page (URL shape proven, cancelled without applying). Signature remains untested until a provider serves our template.
- Merged. Day-0 probe after the upstream merge: IONOS, Cloudflare, GoDaddy all 404 for our template. Providers ingest on their own schedule; some review by hand.

## 4. Operator friction worth remembering (quirks logged)

- `vercel env add` with a piped value works for Production but silently fails for Preview: the pipe eats the "Git branch?" prompt. No "Added" line = not added. `vercel env ls` is the truth. Dashboard for Preview.
- Branch files (`docs/domain-connect/`, `scripts/domain-connect-keygen.ts`) vanish from Explorer when the checkout is on main because another chat switched it. `git branch --show-current` before hunting for files.
- Domain Connect Templates PR must be opened via "compare across forks" against `Domain-Connect/Templates`; the fork's own button opened PR #1 against the fork's master (a Merge button you can press = wrong target). Keep the prefilled PR template, tick the checkboxes, paste apex + subdomain editor links.
- IONOS `/sync/404` after login means template unknown, not a URL fault.
- The "/mnt snapshots at chat start" quirk did not hold here: this chat saw the parallel session's 10-02 swap-in and used it. Re-read numbering regardless.

## 5. Open after this session

- **S433 (do ASAP when the button appears):** IONOS pickup, real approval run on Production with Testing Mover, remove the Preview skip flag, Cloudflare manual onboarding request if needed.
- **S434** callback lands on the active company, **S435** iOS PWA round trip lands in Safari, **S436** GitHub org for the LTD.
- **S437, S438, S439, S440:** the rest of the DNS/mailbox help stack. Smalls, not launch-blocking, high value for trial conversion.
- F15 adapters 2/3 unchanged in scope; start the Google verification application early.

## 6. State of the world

- main `88757cd`. Production build from it. No migration.
- Sending identities: S199 Movers `richardstowell.com` verified; Cornwall `www.cornwallmovers.co.uk` pending (Sid); Testing Mover `mail.richardstowell.com` pending (ours, IONOS, used for S433).
- Vercel env: see backlog amendments. `NEXT_PUBLIC_APP_URL` still Production-only.
- GitHub: `rjstowell/Templates` fork exists (branch `moversapp-email-sending`); the mistaken `rjstowell/Templates#1` is closed.
- Launch state otherwise unchanged from the 10-02 header: Sid inactive, legal S260/S261 operator's, switch list S264/S381/`ACTIVE_TENANT_COOKIE_SECRET`.
