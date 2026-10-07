# Session summary 2026-09-09 (v2)

Domain incident + go-live tail. The marketing site had vanished from `moversapp.app` and the app appeared gone from `my.moversapp.app`. Root cause: the F21 apex go-live was never completed, so the app Vercel project still owned all three custom domains. Fixed live by moving apex, then www, onto the marketing project. End state now matches DEC47: marketing on the apex, www 308 to apex, app isolated on `my`. Also surfaced an unresolved AND unbuilt design question (where in the funnel Stripe checkout fires) and proposed DEC51 (candidate). NO app code, NO migration, NO repo change this session: all Vercel dashboard + DNS.

## The incident

Symptom: `moversapp.app` served the app login, not the marketing site; operator also reported the app missing from `my.moversapp.app`.

Diagnosis (external DNS/HTTP probes + Vercel dashboard):
- App project (`movers-app`) held all three custom domains: `my.moversapp.app`, `www.moversapp.app`, and `moversapp.app` (apex 308 to www).
- Marketing project (`movers-app-marketing-site`) held only its `.vercel.app` URL, no custom domain. Marketing was healthy (200 on its own URL), just address-less.
- So the app had captured apex + www + my; marketing was public nowhere.

Root cause: NOT something breaking. The F21 apex go-live (repoint apex + www to the marketing project) had never been completed. It was the deferred go-live tail from 09-08. The app project was never stripped of apex + www, so those hosts kept serving the app. `my.moversapp.app` was in fact still serving the app correctly throughout (the operator's "app gone from my" was browser cache; incognito confirmed correct after each step).

Correction to an in-session guess: a Vercel redeploy CANNOT move domains (domains are project-level config, not per-deploy), so the 09-09 F33/F34 app deploys did not cause this. The apex go-live simply had not happened.

## What changed (both live, both verified)

### 1. Apex go-live completed
- Removed `moversapp.app` from the app project.
- Added `moversapp.app` to the marketing project, Production, serve direct. NOT "redirect apex to www": www still served the app at that moment, so a redirect would have bounced apex straight back into the app.
- Verified: `moversapp.app` returns 200 marketing HTML.

### 2. www cutover completed (finished the F21 go-live tail)
Pre-flight: an investigation-only brief produced a full host + external-endpoint inventory (`www-domain-move-host-endpoint-inventory-2026-09-09.md`). Findings, all confirming www was safe to move:
- Zero `www.moversapp.app` or `my.moversapp.app` references anywhere in the app repo.
- All three crons (`auto-send`, `auto-replay`, `mailbox/sync`) are Vercel-native in `vercel.json`, invoked against the deployment directly, so they do not route through the www domain and survive the move.
- Stripe (sandbox) webhook already targets `my.moversapp.app/api/stripe/webhook` (moved during the S263 cutover, not www as older docs implied). No live Stripe endpoint exists yet (test to live is S264, unbuilt).
- Resend inbound webhook targets the `.vercel.app` production alias (`movers-app-lake.vercel.app`), not www. Survives.
- All app self-URLs are request-origin-derived (billing checkout success/cancel/return, invite, password reset, OAuth callback), so the app self-rebases to whatever host serves it.
- Mail subdomains `inbound.moversapp.app` (MX to AWS SES eu-west-1) and `send.moversapp.app` (DKIM verified) are separate Porkbun records, independent of the Vercel project mapping. Apex nameservers unchanged (Porkbun). Confirmed resolving. Mail cannot be collateral of the domain moves.
- Supabase Auth: Site URL = `https://my.moversapp.app`, allow-list includes `my.moversapp.app/**`. Auth on `my` safe.

Move:
- Removed `www.moversapp.app` from the app project.
- Added `www.moversapp.app` to the marketing project as a 308 permanent redirect to `moversapp.app` (apex is canonical, per the og:image target). NOT "include apex and www variants" (would have re-touched the already-live apex).
- Verified: `www` 308 to `moversapp.app` to 200 marketing; apex still 200; `my` still the app (307 to /login).

### End state (DEC47 target reached)
- `moversapp.app` -> marketing (200)
- `www.moversapp.app` -> 308 -> apex -> marketing
- `my.moversapp.app` -> app

Nothing broke: mail, crons, Stripe sandbox, Supabase auth, Resend inbound all untouched.

## Discovery: trial funnel checkout placement is open AND unbuilt

Operator asked why the marketing "Start free trial" CTA lands on a bare `my.moversapp.app/signup` rather than Stripe.

- Not a regression. By design the flow is signup first, then in-app Stripe checkout (DEC43: "signup -> Stripe Checkout, card captured"; F21 funnel line: "signup-with-tier -> in-app Checkout"). `createCheckoutSession` is an owner-gated server action inside the authenticated app; the app server is the only place that knows the tenant and decides trial eligibility, so there is no pre-signup public Stripe page and never was one.
- The operator's memory of "hitting Stripe before verifying email" on the test run = the F30 billing test as an already-authenticated tenant (S199) hitting the in-app tier selector, then Stripe, directly. No signup / verify was in that path because the tenant already existed, so "before verify" is really "no verify step present", not a built pre-verify funnel.
- What is actually unbuilt: F21 slice 3 (Pricing-tile to signup tier handoff, plus the post-signup checkout wiring). Deferred, ranked "not a concierge-launch blocker". So today the CTA carries no tier and does not auto-advance into checkout. A fresh signup currently lands in the app with no forced card step, and DEC45 grandfathers a no-billing-row tenant as not-locked, so they can use the app with no card. Tolerable only because launch is concierge (operator sets each tenant up by hand).
- Underneath this: DEC43 explicitly PARKED "where in the funnel checkout fires (signup vs post-onboarding)" as an open placement decision. Never resolved, never recorded anywhere findable.

Limit: the app repo is not in the Project files, so the exact built signup -> verify -> onboarding -> app sequence is unconfirmed. All of the above is from DEC43 + DEC45 + the F21 plan + the billing investigation, not from reading the live signup route.

### Proposed DEC51 (candidate, NOT locked): funnel checkout placement
Lock where Stripe checkout fires in the real funnel, because slice 3 needs it as an input and will otherwise re-open the question cold. Options: before onboarding vs after onboarding (both relative to email verify).
- Leaning before onboarding: onboarding itself fires several LLM calls (classifier generation etc), and card-upfront (DEC43) is meant to filter tyre-kickers before they cost anything, so capture the card before that spend.
- After onboarding lowers bounce (value seen first) but risks a half-built tenant that generated cost then bailed at the card.
Operator to decide + lock. (Numbering note: the 09-09 auto-alert idea also eyed DEC51 but was never written; if both get logged, funnel-placement takes DEC51 and auto-alert becomes DEC52.)

## Launch-plan impact
- Apex go-live was a Tier-1 remaining item. Its domain half is now DONE. The only remaining piece of "apex go-live" is S264 (Stripe test to live + a LIVE webhook at `my`), which rides the separate billing-promotion work and is still owed before real billing.
- F21 slices 1 to 2 were live only on the `.vercel.app` URL; the site is now genuinely public on the real domain. Remaining F21: slice 3 (tier funnel) + S260 (legal fill).

## Owed / open (surfaced, not built)
- DEC51 candidate (funnel checkout placement): pending operator lock.
- S264 Stripe test to live + LIVE webhook at `my`: unchanged, now the only remaining piece of apex go-live.
- Dead Supabase allow-list entries (`www.moversapp.app/**`, `moversapp.app/**`): prunable, auth originates only from `my` now. Low priority.
- Unrelated flag (from the investigation): the Stripe webhook handler acks-and-drops `checkout.session.completed`, acting only on `customer.subscription.*` events. Possibly by design (subscription events carry the state), possibly a gap. Worth a look, not chased this session.

## Method / notes
- All changes were Vercel dashboard + DNS. NO app code, NO migration, NO repo change, no PR.
- Verified end to end via external DNS + HTTP probes (resolution, status, redirect chains, response bodies), Stripe + Resend read-only API (in the investigation brief), and the Supabase dashboard.
- Browser 308 caching: the operator saw the stale app on the apex in a normal tab after the fix; incognito confirmed correct each time. Hard refresh clears it. (Add to quirks if it recurs.)
