# Movers App — Session Summary 2026-09-03 (v2)

**main bbbca2b → faaaa9b.** One launch-blocker shipped + merged: **S221** (Postmark own-domain transport, hybrid with Resend for the generic path). This kills the Resend per-DOMAIN pricing/cap model that 403'd the live "Add domain" flow. Transport vendor decided → locks as **DEC42** (Postmark own-domain, Resend generic, SES documented as fallback). One migration (`provider_domain_id` rename). Three new items raised (S243, S244, F30 candidacy). No F29/S241 work this session (Codex usage limit + operator's call to stop after S221).

## Shipped + merged

### S221 — Postmark own-domain transport (faaaa9b, PR #86). DONE.
The launch-blocker from earlier 2026-09-03. Own-domain sending (tenant-verified) + ALL own-domain verification now go through **Postmark**; generic app-domain sends stay on **Resend**, unchanged. Hybrid dispatch keyed on the EXISTING verified-state derivation — no second source of truth added.

**Why Postmark, not Resend/SES/others.** Resend prices + caps per verified DOMAIN ($20/10 domains, ~$90/1000) — wrong model for multi-tenant own-domain; the live "Add domain" field 403s on the plan cap (= S220's raw-error leak). Compared SES / Postmark / Mailgun / SparkPost:
- **SparkPost (Bird)** — eliminated: post-acquisition pricing opaque, enterprise-sales-gated, vendor-churn risk.
- **Mailgun** — Foundation $35/mo, 1000 domains; middle on everything, weaker shared-IP deliverability, no standout reason.
- **SES** — 10k identities, ~$0.10/1k (10× cheaper at scale). But: sandbox→production-access approval gate, and you must BUILD bounce/complaint (SNS) + suppression + reputation monitoring yourself. One bad tenant can threaten the whole account.
- **Postmark (chosen)** — least build (managed suppression + webhooks, no SNS plumbing), adapter nearly clones the existing Resend domain flow, best deliverability (deliverability failures are silent tenant-harm, the worst failure class for this product). **CAP CORRECTION mid-session:** the initial read that Postmark Pro = unlimited domains was WRONG — live pricing page: Basic $15 = 5 domains, Pro $16.50 = 10 domains, **Platform $18 = UNLIMITED**. Pro reproduces the Resend trap at 10. **Only Platform clears the cap** → Platform is the floor tier, not an upsell. $18/mo flat + $1.20/1k over 10k emails.

**Scale math (correct tier):** 100 tenants/10k emails = $18/mo; 1000 tenants/100k = ~$126/mo. SES stays ~10× cheaper at high volume but the gap is immaterial until several hundred tenants — and switching vendors later re-publishes DNS for every verified tenant, so decide-for-scale-now was chosen while switching is still ~free (near-zero real verified tenants today).

**Step-0 investigate (Codex) confirmed the design before any build:**
- The own-domain/generic BRANCH already existed in all three send paths (manual, edit+send, cron) — `resolveSendFrom` → `getTenantSendingIdentityStatus` derives verified → verified uses tenant from, else `resolveGenericSendFromAddress`. **Missing piece was only transport dispatch** (transport was hard-wired to Resend at import sites).
- Verified gate (`lib/agents/sendingIdentity.ts`: `fromDomain === sendingDomain && status === "verified"`) + generic fallback are centralized/reusable.
- One real coupling: provider-specific `resend_domain_id` column → resolved by rename (below).

**What shipped (11 files, +310/-29 pre-tweaks):**
- `lib/postmark/domains.ts` (new) — account-token Domains API wrapper (create/get/verify/**list**; list added for idempotent re-add — the Resend wrapper lacks it, NOT backfilled). Stores the exact DNS records Postmark returns into `dns_records_snapshot`. DOMAIN-level (DKIM + Return-Path CNAME), not per-address Sender Signatures.
- `lib/postmark/sending.ts` (new) — Postmark sender mirroring `sendReplyEmail`'s `SendReplyEmailArgs` shape + same threading headers + quoted-original HTML.
- `lib/agents/replySender.ts` (new) — the DISPATCHER. `sendingIdentity.verified ? "postmark" : "resend"`. `"resend"` calls the existing `sendReplyEmail` unchanged. Wired into all three send sites (agents/actions.ts manual + edit+send, cron/auto-send/route.ts).
- Own-domain card flow (SendingIdentityCard.tsx → settings/general/actions.ts) → Postmark wrapper for add/verify/status. Generic path untouched.
- Tenant-facing error copy (**FOLDS S220**) → vendor-neutral; grep confirms zero "Resend"/"Postmark" in tenant-facing strings.
- `POSTMARK_SERVER_TOKEN` (sending) + `POSTMARK_ACCOUNT_TOKEN` (domains API) — both server-side only, both live in Vercel prod+preview.

**MIGRATION — `resend_domain_id` → `provider_domain_id` (option A, vendor-neutral rename).** Rides the PR (F-rule). Chosen over a new `postmark_domain_id` column (would create a second dead column, the S208 grievance) or an N-provider `provider`+`provider_domain_id` pair (YAGNI — own-domain is only ever Postmark now). Existing rows keep old Resend ids harmlessly (stale under any option, re-resolve on Postmark re-verify). Every read/write site updated; grep clean except the mandatory mention inside the rename migration.

**VERIFIED live end-to-end:**
- **Generic path:** unverified S199 tenant → dispatcher chose `resend` → sent from `hello@send.moversapp.app` → Resend message ID returned. Hybrid fallback proven.
- **Own-domain path:** S199 configured with `hello@richardstowell.com`; Postmark domain 8083095 created, DNS records stored in snapshot; idempotent re-add returns same id. After DNS published on richardstowell.com (Ionos), DKIM + Return-Path verified, `tenant_sending_identity.verified_at` set. Live send fired **via Postmark as `Sam <hello@richardstowell.com>`**, stream `outbound`, DKIM selector `20260903124811pm`, Postmark Activity: **Processed** (then Hard Bounce — because `hello@richardstowell.com` is not a real mailbox; transport + DKIM signing are the S221 bar, and both passed). Received-header check waived (no mailbox to receive); Postmark Activity rendered message is the evidence.

## DNS gotchas hit this session (all operator-side, cost real time)
1. **Wrong zone first.** Postmark records were published on `moversapp.app` instead of `richardstowell.com` (the domain Postmark generated them for + S199's configured sender). Moved to richardstowell.com (Ionos), strays removed from moversapp.app.
2. **Doubled/wrong host suffix.** The TXT DKIM host was entered as `20260903124811pm._domainkey.moversapp.app` (Ionos stores the host bare; the domain is appended automatically). Corrected to bare `20260903124811pm._domainkey`.
3. **Incomplete DKIM value.** The TXT value was published as `p=MIGf...` — missing the required `k=rsa; ` prefix. Postmark requires the full `k=rsa; p=...`. Corrected → DKIM verified.
Lesson for the tenant-facing DNS help + future providers: host is bare (provider appends the zone), and the DKIM value must be copied WHOLE including any `k=...;` prefix. (Ties F20 DNS how-to.)

## Operator prerequisite state (S243 — partially done)
- Infra email created: `admin@moversapp.app` (Porkbun forward → new Gmail `moversappadmin@gmail.com`), intended as the single owner for all app infra accounts. **Recommendation going forward: register all infra accounts (Postmark, Resend, Supabase, Vercel, Porkbun) under this address.** (Self-hosting a mailbox on the Ionos VPS was considered + rejected — port-25/reverse-DNS/deliverability upkeep cuts against the "less to run" direction that drove the Postmark-over-SES choice.)
- Postmark account created (Free tier for now — fine for dev/verify).
- `admin@moversapp.app` sender signature **verified** ✓.
- Both tokens live in Vercel (prod + preview) ✓.
- **Account approval REQUESTED** ✓ (Postmark manually reviews new accounts; ~1–2 days, off critical path). Form answers positioned the product correctly: transactional-only, reply-to-inbound-only, recipient-initiated, no lists, DKIM-verified domains, per-message bounce/complaint handling via webhooks.
- **STILL OWED:** Platform plan (switch when taking real tenants — Free/Basic cap domains); confirm approval comes back clean.

## New items raised
- **S243** — Postmark account/infra setup (operator prereq; partially done per above; PLATFORM PLAN + approval-clearance outstanding).
- **S244** — Postmark bounce/complaint webhook + suppression handling. DEFERRED out of the S221 brief deliberately. Stated as INTENT on the approval form but NOT built. Needed before any heavy/real sending — hard bounces + complaints must auto-suppress or reputation degrades. Ties S243.
- **F30 candidacy** — S221 became multi-file architectural feature work; consider promoting S221's lineage to F30. (Left as an S-number for now; flag for decision.)

## Notes / lessons
- **Env vars need a redeploy to take effect** — Vercel prompts for it after adding vars; redeploying current main to pick up the tokens is expected + harmless (same code). A token env change does NOT auto-rebuild.
- **The dispatcher correctly chose Resend while the tenant was unverified** — confirms the hybrid fallback is real, not just the happy path.
- **Postmark two-token split:** SERVER token = sending only; ACCOUNT token = Domains API (list/create/verify DKIM+Return-Path). The account token is account-wide + more powerful — server-side only, guard it.
- **Standing rule held:** PR verified with `npm run build` (not just tsc+vitest), PUSH confirmed via `git log origin/main` after merge.

## State for next session
- **S241** (auto_send/verified toggle-persistence anomaly) — was untestable until a genuinely-verified own-domain tenant existed. **S199 is now that tenant.** Needs a Codex run. Sequence this next.
- **F29 auto-fire (cron) verification** — still owed (enable auto-send on a mailbox tenant, confirm cron sends + files to Responded). Unblocked since F29 shipped.
- **Migration-timestamp reconciliation** — repo file `20260903114316` vs applied Supabase history `20260903124445`. Folds into the S130 pile; reconcile with the normal Supabase migration workflow before it compounds.
- **S243/S244** — Platform plan at real-tenant time; build bounce/complaint suppression before heavy sending.
- **DEC42** — locks this session (see backlog): Postmark own-domain, Resend generic, SES documented fallback.
