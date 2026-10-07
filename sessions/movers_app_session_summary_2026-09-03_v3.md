# Movers App — Session Summary 2026-09-03 v3

**main unchanged at faaaa9b. NO code shipped — investigation + live-test session.** No PRs, no commits, no migrations, no DEC locked. All findings via read-only DB queries (Supabase MCP) + Codex investigation runs + operator live-UI testing.

**Session goal:** run the owed **F28 auto-fire (cron) mailbox auto-send verification** — the end-to-end check that was blocked by F29 and left outstanding when F29 shipped. Get it done while a genuinely-verified own-domain tenant (S199) is set up.

---

## Headline outcomes

1. **F28 auto-fire (cron) path VERIFIED to the provider handoff.** The full auto-send lane proved out on S199: draft → `auto_send_scheduled` (+`send_after`) → cron pickup on time → send-time guard → transport call. The only thing not yet seen is a completed *delivery*, blocked externally by Postmark Test mode (S243), not by code. Re-run once approval clears for the final green.
2. **S241 CLOSED — not-a-bug (seed artifact).** Hardening spun out as S245.
3. **Incident found + contained:** S199 auto-send was ON by accident (~1–2 days). **Blast radius = ZERO.** Flag forced off.
4. **Four new items exposed: S245, S246, S247, S248.**
5. **One observation parked:** greeting name-resolution looks like a possible S87/S90 regression — flagged for next session, not yet re-numbered.

---

## 1. S241 — investigated, CLOSED (not-a-bug)

S241 was the toggle-persistence anomaly from F29 State 3 (verified tenant showed auto-send ON in UI but `auto_send_enabled` stayed false, `updated_at` bumped) — untestable until a genuinely-verified tenant existed.

**Codex trace + live DB verdict: not a real verified-tenant bug.**
- Genuine verified S199 persists `auto_send_enabled = true`.
- DB-wide query: **zero** verified tenants stuck in false.
- Verified write path (`OverviewTab.tsx` checkbox → `saveAutoSend` → server action `setAgentAutoSendEnabled` → `actions.ts` `update({ auto_send_enabled }).eq(...)`) writes the column directly. No verified-only branch omitting it, no F29-style subset upsert, no duplicate rows (unique `(tenant_id, kind)` constraint).
- The original anomaly was on a **hand-seeded** row → seed artifact.

**Decision:** close S241 as not-a-bug. Park the optional hardening in Tier-3 as **S245** (read-back-after-write). Codex mis-numbered it "S242" (taken — Supabase URL fix, DONE); corrected to S245.

**NB — the divergence became LIVE later this session:** during incident cleanup, the operator flipped auto-send OFF in the S199 UI; Codex checked and found `auto_send_enabled` **still true** in the DB, and had to force it false via a direct update. So the toggle-vs-DB divergence S241 named is real and reproducible on a real write — S245 is no longer hypothetical.

---

## 2. Incident — S199 auto-send left ON by accident

While setting up the auto-fire test, discovered S199 (live customer mailbox `hello@cornwallmovers.co.uk`) had `auto_send_enabled = true`, left on unintentionally ~1–2 days, duration/count unknown at first.

**Blast radius measured = ZERO.** Live DB: **0 rows** with `status = auto_sent` + `source = mailbox` for S199 across the whole window. Earliest/latest auto-send timestamps: null. The 6 mailbox rows with `status = sent` (26 Aug–2 Sep) were all **manual** sends (operator pressed send). No customer received an unseen auto-reply.

**Why zero, despite the flag being on for days:** a SECOND guard. See §3.

**Contained:** auto-send forced OFF at the DB level (confirmed `false`, `updated_at` bumped). No cleanup, no customer follow-up needed.

*(Transport note corrected mid-session: S199 sent via Resend for the accidental window; Postmark was only connected today. The earlier "everything went via Postmark" statement was wrong.)*

---

## 3. The second guard — template `auto_send_eligible` (why blast radius was 0)

Codex traced the routing decision in `computeDraft.ts` / `pipeline.ts`. The auto-send scheduling condition is:

```
agent.auto_send_enabled === true &&
selectedTemplateRow.auto_send_eligible === true &&
classification.confidence >= Number(agent.confidence_threshold)
```

Evaluated in `computeDraft.ts`; only an eligible result gets `send_after` + `reason = "scheduled_auto_send"` + `status = "auto_send_scheduled"` in `pipeline.ts`. Everything else becomes a `drafted` review-queue row.

For S199: agent threshold **0.85**, delay **300s**. **Every one of its seven templates has `auto_send_eligible = false`** — including `Quote request reply` and `Man and van reply`. So the tenant flag was on, but no draft could ever reach the auto-send lane. That is exactly why the accidental flag fired nothing.

Codex's snapshot of the 59 unresolved review-queue drafts: 52 mailbox/removals + 5 mailbox/man_and_van + 2 resend-inbound/removals, **all blocked by template eligibility** (3 also below 0.85).

**→ New item S246** (dashboard warning: "auto-send is ON but none/N of your templates are set to auto-send") — the silent no-op deserves a signal.

---

## 4. Isolated test rig + confirming old drafts inert

**Rig (operator):** on S199, disable the INBOX→Shadow duplication (the app only reads the Shadow folder), so the app sees only hand-placed test mail; arm auto-send + flip the removals template `auto_send_eligible = true`; drop the delay low. Own-domain verified → routes via Postmark. Isolated because the only inbound is the operator's own test email → only auto-reply goes to a controlled address.

**Safety check before arming (Codex, read-only):** confirmed flipping the template flag + tenant flag does **NOT** retroactively arm the 59 existing drafts. Eligibility is stamped **once at draft time** (`processInboundRun()` writes `send_after` + status then); the cron only sweeps rows with `send_after IS NOT NULL` and due. The 59 are `drafted` / `send_after = NULL` → cron-invisible. No trigger re-stamps them (only the `letterbox_dormant` recovery path reprocesses, not `drafted` rows). **This is the safe design** — the dangerous alternative (re-evaluate all history on a setting change) is exactly what would let one toggle blast a backlog. It doesn't.

---

## 5. Test run 1 → `manual_hold` divert (postcode/distance guard) → S247

First test email had **no postcode**. Run `ece516c9` ("Moving house"):
- `distance_status = no_postcode`, `postcode_missing = true`, `distance_miles = null`
- `quote_routes = [list, survey, photos]`, category `removals`, confidence 0.85
- Draft = generic "please send your addresses/postcode/date" info-request (no price)

It **scheduled**, the cron picked it up at the tick (UI showed "Sending…"), then a **send-time safeguard diverted it to manual review**: cleared `send_after`, set queue `reason = manual_hold`, reverted run to `drafted`. No email sent (`outbound_email_id` null; Postmark Activity zero). No re-fire (send_after null → cron-invisible).

Codex confirmed the guard: `hasSurveyRoute && hasUnresolvedDistance`, evaluated in `app/api/cron/auto-send/route.ts` **before** the provider call. Appended reason: *"Auto-send diverted: a survey was offered but the distance to the pickup address could not be worked out."*

**Operator flag (correct):** this holds auto-send exactly when the draft is *asking for the missing info*. It conflates "can't compute a price" with "shouldn't reply" — but the info-request reply is the single most valuable thing to auto-fire fast (customer emails with nothing → bot instantly asks for details). Real customers regularly email with no info. **Must be fixed before onboarding.**

**→ New item S247** (Tier-1, pre-onboarding): split "can't price → hold" from "no info yet → send the standard ask-for-details reply."

---

## 6. Test run 2 → cleared guard → `auto_send_failed` (Postmark Test mode) → S248

Second test email included **both postcodes** (PL14 → TR1). Run `3ba75695` ("Quote for house move"):
- `distance_status = resolved`, 51.9 mi → `hasUnresolvedDistance = false` → **guard passed**
- `quote_routes = [list, photos]` (no survey), status `auto_send_scheduled`, `reason = scheduled_auto_send`, `send_after` set + due

Cron picked it up on time, evaluated the guard (passed), and **called the transport** — then **failed at the provider send**: `status = auto_send_failed`, `error = "Email delivery failed. Please try again."` (vendor-neutral copy from S221), `outbound_email_id` / `resend_email_id` both null → nothing delivered. Queue `resolution = auto_send_failed` (resolved → no cron retry). No re-fire.

**Root cause = Postmark Test mode.** Postmark dashboard: **"Test mode" + "We're reviewing your account"** (S243 approval requested today, ~1–2 day review). In Test mode Postmark only delivers to your own verified addresses; sending to an external gmail is blocked. **Not a code fault.**

**This CLOSES the F28 auto-fire verification to the provider handoff** — every step of our code (schedule → cron pickup → guard → transport call) is proven live. Only the external delivery is gated. Re-run the two-postcode email once Postmark approval clears for the delivered green.

**Operator flag → New item S248:** the failed send tells the tenant "please try again" with **no retry button** and no reason. Prior art: **S61** deliberately made `auto_send_failed` a *resolved / non-retryable* row ("no human watching") vs `manual_send_failed` which stays unresolved/retryable. That assumption breaks — the failed auto-send is sitting in the Review Queue where a human IS looking, wearing misleading copy. Fix = (a) plain-language failure reason (extend `errorMessages.ts`), (b) a retry/manual-send button. **Reason copy is load-bearing** — a naive retry loops on external/transient causes (retrying now fails again until Postmark approves). Lightly reverses the S61 design → candidate **DEC43** (not locked this session). Tier-2.

---

## 7. Observation parked — greeting name resolution (possible S87/S90 regression)

Operator signed a test email "Jammy Dodger Man the Great" (From display name "Richard"). The Movers app draft greeted **"Hello Richard"** (From display name); the operator's existing make.com automation greeted **"Hi Jammy"** (signoff), which the operator judges more correct.

**Prior art:** this is the **S87/S90 greeting defect** — the rule "greeting uses the signoff/signature name, not the From display name" — recorded as shipped + verified 2026-08-14. What was seen looks like a **possible regression**, OR the extractor rejecting the absurd signoff as a non-name and falling back to From. (Compare S72: "Hello Johnson" was NOT a bug — extraction was correct there.)

**Status: parked, NOT re-numbered.** Maps to existing S87/S90. Investigate next session before assigning anything new: pull `GREETING_SYSTEM_PROMPT` behaviour + the extractor's From-vs-signoff precedence on a display-name-vs-signature conflict.

---

## New items (full detail in numbering block)

- **S245** — auto-send toggle-persistence hardening (read-back-after-write; UI from persisted boolean; regression test). Tier-3. *Divergence observed live this session.*
- **S246** — dashboard warning: auto-send ON but no/only-some templates `auto_send_eligible`. Tier-3.
- **S247** — send-time postcode/distance guard over-holds the standard info-request reply. **Tier-1, pre-onboarding.**
- **S248** — failed-send retry action + human-readable failure reason. Tier-2. Candidate DEC43.

Next free S = **S249**. DEC43 free (S248 candidate). F30 free.

## Closed / status changes

- **S241** — CLOSED, not-a-bug (seed artifact). Hardening → S245.
- **F28 auto-fire (cron) verification** — effectively CLOSED to the provider handoff; final delivery parked on S243 (Postmark approval).

---

## State for next session

- **F28/F29 final green:** once Postmark approval clears (~1–2 days), re-run the two-postcode Shadow test on S199 → expect `outbound_email_id` populated, `status = auto_sent`, reply delivered, original filed to `INBOX/MoversApp Responded`.
- **S247 is pre-onboarding** — the postcode/distance guard fix should land before Cornwall onboarding.
- **S248 + candidate DEC43** — decide whether `auto_send_failed` rows become human-actionable (retry + reason) at the same time.
- **Greeting observation** — investigate S87/S90 precedence next session.
- **Tier-1 still open** beyond the above: S244 (Postmark bounce/complaint suppression — before heavy sending), S191 → S194 (failed-run observability), F21 (marketing site + privacy, blocked on billing), Cand. D (billing decision).
- **Operator end-of-session cleanup (done):** auto-send + template auto-send-eligibility turned OFF on S199; INBOX→Shadow duplication re-enabled. S199 back to safe live state.
