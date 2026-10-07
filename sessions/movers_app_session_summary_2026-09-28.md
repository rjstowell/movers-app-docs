# Movers App: Session Summary 2026-09-28

**Chat span:** 2026-09-25 (evening) to 2026-09-28. **main `a46fc07` -> `1ea9e1b`. Tests 1239 -> 1314.** Two merged PRs (#144 F55 slice A, #145 F61 slice A), eight direct-to-main commits, no migrations, one MCP data reset (Sep2 Movers `branding`).

## 1. Launch pin-down (no build)

Read launch plan + active backlog + driving vision. Result, unchanged by the rest of the session:

- **Sid trial: ungated.** Fresh signup, top tier, test card, multi-company add-on, company 2, invite operator. Hold until his mailbox connects: S376 (Bedowan Meadows trace), S359 (bound LLM calls), S243 Postmark cap check, Make pause plan.
- **Real paying strangers: seven items, zero build.** S264 Stripe test->live (+ live portal configs, `BILLING_CUTOFF` reset, live webhook) with S381 Radar; S265 auth SMTP; S243 Postmark Platform; `ACTIVE_TENANT_COOKIE_SECRET`; S326 operator login; S260 legal (solicitor + ICO); S261 DPA. Items 1 to 5 = one afternoon; 6 and 7 external, start now. Then one verify session.
- **Broad self-serve launch (not gates for hand-held outreach):** F61, F60, F21 slice 3, S323.
- **Marketing/outreach starts now.** No card taken from strangers until S264 + S260 + S261.

## 2. Comp billing call

Question: how do Sid's companies (and Cornwall, partners) run on the app without paying, without a hacked DB state?

Codex read-only investigation (09-26) of the billing gates: `tenant_billing` has no `tier` column (tier derives from `stripe_price_id`); `isBillingBlocked` reads `status` only; a spoofed `status='active'` row with NULL Stripe ids passes every gate and gets the right cap/seat band if `stripe_price_id` is set, but the Billing page then lies (three Choose buttons, no Manage billing, trial copy) and Choose opens a real paid checkout.

Decision: **Stripe-native coupon.** Real card at normal Checkout; operator applies a 100%-off-forever coupon from the Dashboard (Subscription -> Actions -> Apply coupon) before day 30. **No `allow_promotion_codes`** on Checkout: a visible code box invites code-hunting and abandonment. No SQL spoof. Coupon created in test mode now, again in live at S264. Sid on test card now is fine: at the flip, NULL his test Stripe ids via MCP, he re-checkouts on live, coupon applied; `trial_used_at` already stamped is irrelevant under 100% off.

## 3. The onboarding run (unplanned, all prod-verified by the operator)

Three operator corrections opened it: logo should be asked during onboarding; Skip should skip the step, not the wizard; palette extraction never found the brand colours. Codex read-only grep first (wizard is one 969-line component; Skip exited via F51 `provisionPartialOnboarding`; no per-step skip existed; colorthief defaults kept greys; onboarding never touched `branding`; `BrandingForm` not reusable). Then in order:

- **S386 `3fe5256` + `eb4d684`.** Skip = clear this step, advance. Hidden with a reason on the three required steps. No dialog, no server call. Tone Skip keeps the default. F51 partial-provision path now unreachable from UI, kept in code.
- **S385 `6d292e3`.** Own `extractPalette` (background/outline filtering, saturation-weighted ranking, dedupe, additive fallback). colorthief removed. Operator: "working a lot better".
- **S387 `6a4d772`.** After S386, a skip-everything Finish read Done on the F9 row and lost the link back. Now "Business setup: X of Y steps done" with the wizard link until every step has data; `?resume=1` admits completed tenants; re-entry Finish is **config-only** (`saveOnboardingEdits`) because re-running the provisioning RPC would overwrite template bodies, signature and the quote-route rule (Codex caught this and asked). Routes/mileage/exclusivity read-only on re-entry.
- **F55 slice A, PR #144 `050a77a`.** `branding` step after `name`. Shared `BrandingPicker`; `uploadTenantLogo` action; upload on select; 1024px client downscale; bucket-path URL validation (Codex addition); progress counts the step. First prod pass did not show the step because Sep2 Movers already had a Settings logo; `branding` reset via MCP and it rendered.
- **F61 slice A, PR #145 `bd2e0d8` + `b5a5384` + `8c22fe6` + `0b45bfc` + `1ea9e1b`.** Operator reframed the wizard from "agent setup" to business setup: intro screen (first entry only), per-step eyebrows (a uniform "Setup" eyebrow was tried and rejected as pointless text), "Business setup" row, name step "Who signs your replies?". Branding step trimmed to logo + suggested swatches, then rebuilt on operator feedback: live re-theme of the wizard itself as colours are picked (brand CSS variables on the wizard root; pure mapping moved to `lib/brand-tokens.ts` because `tenant-branding.ts` pulls the service-role client), native colour pickers with pipette badges and a caption, Undo under the picker row, short copy. Codex surfaced a default mismatch (app blue vs Settings teal): fixed by saving colours only when picked plus one default `#1D5FD6`. Final operator verdict on prod: "perfect".

Wrong call of mine during this run, corrected by the operator: I claimed brand colours only reached the quote PDF; they drive app accents and buttons. Copy was rewritten accordingly.

## 4. Doc changes this pass

- Active backlog: new 2026-09-28 header; 09-25 v2 header demoted to a pointer; numbering S389 / F64 / DEC61; F55 and F61 entries amended; new session block with decisions, S388, and amendments (billing gate findings, F51/F9 notes, wizard shape for the record).
- Archive append 2026-09-28: outgoing header + numbering, DONE bodies for S385, S386, S387, F55 slice A, F61 slice A.
- Launch plan: new baseline `1ea9e1b` with the three-tier pin-down and the comp-billing call; estimate line unchanged in substance.
- Quirks: six lines (two brand-default copies; `tenant-branding.ts` is server-only; saved-logo re-entry; 1 MB server-action cap; CRLF/harness gotchas; harness cannot sign in).

## 5. Next

- Sid signs up. Test-mode coupon applied to his subscription.
- Operator: solicitor + ICO (S260), DPA (S261), outreach.
- Next chat: support page (tickets and/or live chat, help section). Fresh chat; this one is full of screenshots.
- Open small from this run: S388 (name change on re-entry does not update the signature).
