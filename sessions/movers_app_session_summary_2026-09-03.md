# Movers App — Session Summary 2026-09-03

**main f6a5e4d → bbbca2b.** Two fixes shipped + merged (F29, S143), two papercuts done (S233, S234), one item verified-closed (S201), one production config bug fixed (S242) that turned out to be the root cause of a long false-trail, and a cluster of stale-provisioning bugs surfaced (S237, S238). S221 upgraded to launch-blocking. No new DEC locked; no migration.

## Shipped + merged

### F29 — new-tenant auto-send consent loop (deff265, PR #84). DONE.
The launch-blocker from 2026-09-02. A brand-new tenant accepting the generic-sender consent modal looped on "No sender email configured" and the auto-send toggle never stuck.

Root cause was **not** a read/write table mismatch (both revolve around `tenant_sending_identity`). The read gate correctly treats `RESEND_FROM_EMAIL` as a valid sender fallback, so a fresh tenant is *offered* the generic-consent modal. But the consent WRITE helper `acknowledgeGenericSendingConsent` had a backwards precondition — `if (!fromEmail) return "No sender email configured."` — and was fed only `contact_email` (no env fallback). A fresh tenant with no business email → write refuses → loop. Demanding a personal sender email to consent to *using the shared address instead of a personal one* is a false gate.

Fix: dropped the precondition; consent upsert now writes only `autosend_consent_ack` + `_ack_at`, **non-clobbering** on conflict (a verified tenant's `sending_domain`/`from_email`/`verification_status` are preserved). Generic-send consent is now a tenant-level flag; the generic address resolves from env at SEND time. Sending itself is still gated by `resolvedFrom` being non-empty, so consent granted ≠ blind send.

Verified end-to-end on the prod DB:
- **State 1 (fresh tenant, no business email):** consent sticks, `auto_send_enabled` true, row created with `from_email` null + `sending_domain` "".
- **State 2 (existing row, not-yet-consented):** consent writes, toggle sticks, and the existing `sending_domain`/`from_email` are **preserved** — proves the on-conflict clause is DO UPDATE of only the consent columns (not DO NOTHING, not a clobber). This was the canary; it passed.
- **State 3 (verified tenant):** surfaced an unrelated anomaly → **S241** (below). The non-clobber held (verified fields untouched).

Closes the S201 consent-loop hole. Confirms S208 (`from_email` stays null/dead on generic consent). The underlying decision — generic consent = tenant-level flag, generic address from env at send — will lock as **DEC42** but was not formally taken, so it remains pending.

### S143 — false "no need for a visit" fallback (bbbca2b, PR #85). DONE.
`resolveQuoteRoutes` fell back to `['list']` when a tenant's quote-route policy was unknown, rendering "We quote by email, so there's no need for a visit" to customers — a false claim for a mover who surveys, and dismissive to the many customers who *expect* a visit.

Fix: fallback returns `[]` (empty) for null / `{}` / empty-routes / subtraction-to-empty, so **no route sentence is emitted** rather than a false claim. Confirmed the `{{route_sentence}}` `<p>` node strips clean (no gap, no dangling text). Route-sentence STRINGS were deliberately left untouched — those belong to the copy session (S69). Unit test flipped to expect `[]`; render check confirmed all three cases (populated / old `['list']` / empty).

**Verified by unit test + render check + merge. The live-email test was inconclusive** — both available test tenants are broken in different ways: Sep2 is unprovisioned (skipped onboarding → run `ignored`, no draft), and Sep Movers has the route sentence **baked into its template body** as literal text instead of the `{{route_sentence}}` placeholder, making it immune to all route logic (→ S238). Confirmed the resolution order the hard way: **quote_route local rule → `agent_quote_route_policy.routes` → fallback** — both upstream sources must be empty to reach the fallback.

The **nudge half** of S143 (tell an under-configured tenant their customers are only being asked for an address) is split out as **S235**. This fix kills the false claim; it does not yet warn the tenant they're thin.

### S233 — inbound-mailbox card copy (c7d6da9). DONE + verified prod.
The card said mailbox runs "always land in Review Queue" — false after F28 (mailbox auto-send parity). Replaced with copy reflecting that mailbox runs follow the tenant's auto-send settings like any other source. Verified live on the Setup tab.

### S234 — exhaustive-deps warning (d8a0507). DONE.
The persistent `react-hooks/exhaustive-deps` warning on `AgentOnboardingFlow.tsx`'s `useMemo` (missing `depotNeeded`). Codex chose the cleanest fix: inline the `depotNeeded` condition into the memo so the closure disappears entirely — a behavioural no-op (`depotNeeded` was derived only from values already in the deps). Warning gone, confirmed by grepping the build log.

## Verified-closed

### S201 — auto-send with no sender. VERIFIED-CLOSED.
Read-only trace confirmed no send path (manual approve, edit+send, cron auto-send) reaches Resend with an empty/invalid From — all resolve via `resolveSendFrom` (`contact_email || RESEND_FROM_EMAIL`) and guard before `sendReplyEmail`. The cron re-checks the sender at send time even if the toggle was enabled earlier, and marks `auto_send_failed` (visible in Review Queue) on failure. The consent-loop half was closed by F29.

**Residual (not fixed, near-unreachable in prod):** the manual path with an empty sender *from the start* fails gracefully (inline error, no send) but does not write `manual_send_failed`, so observability is weaker there than on cron. Safe — no blind send, nothing marked-sent-with-nothing. Only fires if both `contact_email` and `RESEND_FROM_EMAIL` are missing, and `RESEND_FROM_EMAIL` is set in prod (DEC33/S209). Logged as a residual note only.

## Config fix — the root cause of today's detour

### S242 — Supabase auth Site URL + redirect allow-list. DONE (config, no code).
The Supabase Auth **Site URL** was pointing at an ephemeral S58 preview branch (`movers-app-git-codex-s58-...vercel.app`). So when the operator confirmed a signup email, the confirmation link dropped them onto **stale branch code**, and they did onboarding there without realising.

This caused a long false-trail: onboarding *appeared* broken (survey-distance step "missing", a dead "When do you offer each?" screen). Both were **S58 ghosts** — on `main` the survey-distance step is present and "When do you offer each?" is correctly gone (verified by re-onboarding on prod). Codex's HEAD traces were correct throughout; the operator was simply on the wrong deploy. Lesson: preview *browsing* ≠ preview *email processing*, and a stale Site URL silently routes real auth flows to a dead branch.

Fixed: Site URL → `https://www.moversapp.app` (canonical www — the apex redirects to www, and matching it avoids a token-dropping redirect hop on confirmation/reset links). Redirect allow-list corrected: added `https://www.moversapp.app/**` + `https://moversapp.app/**`, removed the S58 preview entry. `localhost:3000/auth/callback` kept for dev.

## Launch-blocker upgraded

### S221 — own-domain send transport. UPGRADED to launch-blocking + re-scoped.
Own-domain sending is essential from launch (operator decision — reverses the earlier "not a launch blocker"). The real problem is Resend's pricing MODEL: it prices per verified DOMAIN ($20/10, ~$90/1000), which scales badly for multi-tenant own-domain. Live symptom found this session: the "Add domain" field 403s on the plan cap (raw error to tenant = S220's leak). The wiring works — it's a quota refusal, not missing code.

Direction (vendor **not** locked): switch the own-domain path to a per-EMAIL transport, HYBRID with Resend kept for the generic path. SES is the frontrunner (10k domains free, $0.10/1k emails; sandbox → production-access is a pre-launch gate; infra not managed). Alternatives (Postmark/Mailgun/SparkPost) to be compared before locking. Logged as a **DECISION PENDING** (will lock as DEC42 when the vendor is chosen). Ties S220/S219/S208.

## New items raised

- **S235** — dashboard warning for a tenant with empty/null quote-route policy (the nudge half of S143). Persistent until routes set. Relates to F9.
- **S236** — "Skip for now" discards all onboarding data entered before the skip point (provisioning only runs on Finish). Should persist or partially provide.
- **S237** — inbound-address provisioning inconsistent: some tenants get the old `inbound-{uuid}@…resend.app` format instead of the F23-slice-1 company-slug on `inbound.moversapp.app`. Sep Movers (old) vs Sep2 (new), both same era. Leaks UUID + non-branded domain.
- **S238** — stale templates with the route sentence baked in as literal text instead of `{{route_sentence}}` (Sep Movers confirmed). Such tenants are immune to route logic — the claim can't be corrected by config. Same stale-provisioning family as S237.
- **S239** — survey-distance onboarding step has no `canContinue` guard; a tenant can select survey, clear the field, and finish with a null distance.
- **S240** — DECIDE: mute/lock main navigation during onboarding so users can't wander off mid-flow.
- **S241** — auto_send/verified toggle-persistence anomaly (F29 State 3): toggle showed ON but `auto_send_enabled` stayed false on a hand-seeded verified tenant. Off the F29 path; may be a seed artifact; untestable until a genuinely-verified own-domain tenant exists — which needs the S221 transport. **Sequence this check right after the transport is decided and set up.**
- **S69 REVIVED** — the copy-review session: the "no need for a visit" rewrite (route sentences shouldn't imply a visit is unnecessary, since many customers expect one), a postcode-vs-full-address audit against DEC20 (Codex's rendered example asked for postcodes; DEC20 decided full addresses — confirm live copy), and a review of all 7 route sentences / 6 bodies / missing-info block.

## Notes / lessons

- **Codex was wrong three times on the onboarding "ghosts"** — but its HEAD traces were accurate; the disagreement was that the operator was on S58 preview code (the Supabase Site URL trap, S242), not main. Don't trust preview *browsing* to reflect main when auth routing is misconfigured.
- **Skipped-onboarding tenants aren't provisioned** (no templates/categories/agent) — they can't process inbound mail and can't render a reply, so they're useless as render test beds.
- **Copy architecture (re-confirmed, DEC20):** reply copy is hand-written seed copy shared across tenants — 6 body variants + 7 route sentences, assigned per tenant by stable tenant-id hash. LLM runs at onboarding time only (services, category descriptions) + per-inbound extraction; inbound reply rendering is deterministic, no LLM. So the seeded sentences are what real customers get.
- **Standing rule (S231 family) extended:** briefs that commit direct-to-main must confirm the PUSH (`git log origin/main`), not just the commit — a fix sat committed-not-pushed twice this session.

## State for next session

- **F29 auto-fire (cron) verification still owed** — now unblocked (a tenant can finally enable auto-send). Needs auto-send ON on a mailbox tenant.
- **Transport decision (S221)** — compare SES vs Postmark/Mailgun/SparkPost, lock a vendor (DEC42), then set it up. **Then run S241.**
- **Copy session (S69)** — its own pass, with `docs/reference/auto-responder-template-copy-reference.md` open, DEC20 as the yardstick.
- **Stale-provisioning cluster (S237 + S238)** — investigate the provisioning path; likely correlated tenants; may need a template/address migration.
- Onboarding guards (S239 + S240) and the skip-data-loss fix (S236) when convenient.
