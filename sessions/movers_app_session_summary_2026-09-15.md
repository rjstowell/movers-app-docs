# Movers App — Session Summary, 2026-09-15

**Shape:** launch-scope review → the new-tenant correctness cluster → a four-commit depot arc shipped and verified on prod → one customer-facing PR merged → two read-only investigations that killed one design and produced another. **Five ships, no migration.** main `9378ef2` → `52118cf` → `f5131b4` → `dc9f43c` → `c72fce9` (PR #119) → `cffe546`.

**Headline:** the depot arc is CLOSED end to end (onboarding capture, typed-address resolution, Settings parity, removal warning, and the auto-send hold that made a missing depot safe). **S199 DONE. S249 DONE.** The parked Settings stale-coordinates task is RETIRED, absorbed by the Settings port. And the session's last hour produced the design for **F51** (partial provisioning on skip), which pulls **F9** (dashboard setup checklist) into launch scope as its co-requisite.

---

## 1. Launch-scope review (opening)

Reviewed launch plan + driving vision + master backlog against "what actually needs building before paying customers", solicitor/legal work excluded by operator instruction. Cross-checked every candidate against the master backlog rather than the launch-plan tier tables, which lag in places.

**Verdict at session start:** real build work = the new-tenant correctness cluster (S199 + S153 + S154), F12(b) real-email inbound test, the LLM spend caps (S252 inert + S266 per-tier), and S244 Postmark suppression. **Non-code go-live switches** = S264 Stripe TEST→LIVE (the money gate), `BILLING_CUTOFF` reset, S253 OpenAI hard cap, S243 Postmark Platform upgrade, S265 auth-email SMTP. Three UI sessions (F39–F50) confirmed as correctly deferrable.

**Flagged contradiction, unresolved:** F11(d) reads DONE in the master backlog (2026-08-27 v2) but is still carried as open session 8 in the launch plan. Likely the threaded-send half is done and only F12(b) remains. **Two-minute operator check owed.**

---

## 2. The correctness cluster shrank on contact with the live DB

Before writing any brief, read the provisioning path live (read-only, MCP). This reframed the whole cluster.

- **System categories are the noise.** `escalate` and `ignore` are seeded to every tenant with `is_system = true`, correctly description-less and template-less. They accounted for **30 of 34** null-description categories and **30 of 36** template-less categories. **Any guard or backfill must exempt `is_system`.**
- **S153 real footprint: 4 rows**, all on two known test tenants (Testing Mover: storage, scilly; Newco Removals: feedback, scheduling_change).
- **S154 real footprint: 6 rows**, all on test tenants.
- **S199 confirmed at the RPC:** `provision_default_email_responder` takes `p_survey_drive_miles` but has NO depot parameter and never writes `depot_address`. Onboarding structurally could not capture a depot.
- **S199 live damage: ZERO.** All 5 tenants with a survey-distance rule already had depot address AND coordinates. Forward-looking only.

**Consequence:** S199 was the only member of the cluster guaranteed to hit a real tenant, and it was self-contained with nothing to backfill. Took it first. S153/S154 remain open but are demonstrably test-tenant artefacts, not a live provisioning defect.

---

## 3. The depot arc — four ships

### 3.1 `52118cf` — onboarding depot capture (S199)

Found **half-built already** (commit `01251fe`, 2026-08-17, added a depot step enforced browser-side only). Audited against the brief and finished the missing parts rather than rebuilding.

- New shared `saveDepotLocation` in `lib/tenant/depot.ts`; Settings and onboarding both call it.
- **Server check added** in `completeAgentOnboarding`: refuses to finish when survey is a route, mileage > 0, and no depot. Previously UI-only.
- **Save order fixed:** depot now saves BEFORE the provisioning call. Provisioning marks onboarding `completed`, so a failed depot write used to leave a finished tenant with no depot and no way back into onboarding.
- Depot field moved into the mileage step; required when mileage > 0, optional when blank/0.
- Stale coordinates dropped when a picked suggestion is then edited.
- **Brief correction, logged:** the brief asserted the distance rule needs coordinates. It does not — `depot_address` alone suffices; coordinates only skip a lookup, and `distance.ts` geocodes the address on demand. Instruction was stricter than reality; no harm.
- No migration, provisioning RPC untouched. 5 new tests + all 207 existing pass.

### 3.2 `f5131b4` — typed-address resolve, confirm, postcode fallback

Closed the hole the capture opened: the field accepted any freetext, so typed junk stored a depot that never geocodes — the S199 failure via a different door. Design agreed in chat before building: **resolve on Continue → confirm the match → block on no-match → pass through on outage.**

- **Google gate cleared with no new API.** Discovery found `AddressAutocomplete` never loads a Google SDK client-side; it calls the app's own `/api/places/autocomplete` + `/api/places/details` on `GOOGLE_MAPS_SERVER_KEY` (Places API New). Typed text reuses the same two routes. No operator toggle needed.
- **No-match vs outage distinguished by HTTP status** (Places New uses status codes, not the legacy `ZERO_RESULTS` strings): 200 with zero suggestions = no match = **block**; any service error (network, quota, auth, 502) = **passthrough, address saved without coordinates**.
- Confirm-the-match ("We found: X. Use this?") prevents a vague entry resolving silently to the wrong place and mis-measuring every survey.
- `completeAgentOnboarding` deliberately still requires address, NOT coordinates — address-only is a legitimate state.
- **Bonus:** both Places routes now log Google failures to Vercel, so a real outage passthrough is visible rather than silent.
- Verified against real Google ("asdfasdf" → zero suggestions; "LS1 1UR" → match). 227/227 tests.

### 3.3 `dc9f43c` — Settings depot parity + stale-coords RETIRED

Ported the same logic to Settings › General, because skippers and later-setup tenants configure depot there.

- **The stale-coords bug was confirmed real:** Settings' main Save (`updateGeneralSettings`) wrote only `depot_address` and never touched `depot_lat`/`depot_lng`. A typed save kept the OLD address's coordinates, and since `distance.ts` prefers coordinates, distances measured from the old place. Clearing the field also left coordinates behind.
- Fix: the main Save now writes depot ONLY through `saveDepotLocation`, which always writes address + coordinates together and clears all three on empty. Editing the text clears in-form coordinates.
- Shared decision core extracted to `typedAddressPlan.ts`; `planDepotContinue` kept its signature and behaviour, so onboarding tests passed unmodified (the stop condition in the brief was correctly not tripped).
- Settings depot is OPTIONAL (empty allowed), unlike onboarding's conditional requirement.
- **"Saved" no longer shows before a save has actually completed.**
- 245/245 tests.

### 3.4 `cffe546` — depot-removal warning (the item-2 follow-up)

Warn-and-confirm on a set→empty transition only. Not a hard block (pricing fails safe; clearing can be legitimate), and not on every save.

- Always shown: the pricing/auto-distance line. **Additionally** when the tenant has an active survey `drive_miles` rule: the survey line.
- `hasSurveyDistanceRule` computed server-side in the Settings page (editors only, one query), passed as a boolean prop.
- **Collision risk cleared:** the new check runs BEFORE the typed-address plan and fires only when the saved depot is set and the field is empty; the typed-address confirm only handles non-empty text. The two cannot overlap.
- Confirm clears all three fields; Keep depot restores and saves nothing; Save disabled while the box is open.
- 262 tests (9 new).

**Approved copy (operator directive, verbatim, now on both surfaces):** `Select your address. This is used to calculate quotes and survey distances` — replacing the two earlier helper lines (redundant + jargon-y). Confirmed accurate: depot genuinely feeds quote pricing, not just survey routing.

---

## 4. `c72fce9` (PR #119) — auto-send hold when depot missing (S249 DONE)

The read-only quote-engine investigation (section 5) surfaced a bigger risk than the warning we were designing, and it was promoted above it.

**The harm:** a tenant with a survey distance rule and NO depot cannot evaluate the rule, so the rule is skipped and the survey is offered to EVERY enquiry — and the auto-send hold covered only `no_postcode` and `lookup_failed`, so those offers **auto-sent to out-of-range customers, silently.**

- `no_depot` added to the hold set; such a draft goes to review and never auto-sends.
- **Structural fix, not just the instance:** the bug existed because the auto-send cron and the review queue kept SEPARATE hold-reason lists. Unified into one shared helper, `surveyDistanceHold()` in `surveyDistanceStatus.ts`, which both now read.
- Review-queue label "No depot set" + reason text: *"Auto-send diverted: No depot address, so survey distance couldn't be checked. This offer was held. Add a depot in Settings, or remove your survey distance limit."*
- Route choice, pricing and MANUAL send all unchanged — a held item can still be sent by hand, exactly like a `no_postcode` hold. **No send-anyway override**, by explicit decision (see section 7).
- Correctly a branch + PR (reason text, label, shared helper, new test file, customer-facing send behaviour). Merge tied to the reviewed commit `0fcb014`. 253 tests.

**Pre-merge safety check (MCP):** exactly ONE tenant was in the triggering state (Tommy Movers, test, survey rule + no depot), so the merge was CORRECTIVE, not risky — it fixed the one tenant already exposed. No real customers affected.

---

## 5. Read-only investigation: does depot feed quote pricing? (YES)

- Depot is the **start AND end** of the calculator's auto-filled route: **depot → collections → deliveries → depot** (a round trip). It drives fuel, the distance-band markup, AND travel labour (hours/minutes → labour).
- **But clearing it does not silently misprice.** With no depot, auto-calculation doesn't fire, miles stay 0, price renders `£—`, and save is BLOCKED until miles are typed by hand. **Pricing fails safe.** That is why the removal warning is warn-not-block.
- Real risks named: (a) **stale miles**, not zero — a quote whose route was calculated before the depot was cleared keeps its old miles, and neither failure path resets them; (b) the auto-responder over-offer, fixed by PR #119.
- Already-saved quotes are unaffected (price + miles stored and restored without recalculating).

---

## 6. Read-only investigations: Skip, and what killed live-partial

### 6.1 What Skip actually does

- **Skip is a WHOLE-onboarding skip, not per-step.** The button sits outside the per-step blocks so it shows on every step; "Skip anyway" calls `skipAgentOnboarding(tenantId)` and routes to `/dashboard`.
- It sends ONLY `tenantId`. One upsert: `agent_onboarding_status = 'skipped'`. **It never calls `completeAgentOnboarding` or the provisioning RPC.**
- **Therefore Skip CANNOT create the survey-rule-without-depot state.** The mileage exists only in React state and is discarded.
- **Saved on skip:** the status flag, plus `website_url` (written mid-flow on the website step's Continue — the only mid-flow write). **Lost:** services, tone, quote routes, mileage, depot, exclusivity, agent name, phone. No agent row, no categories, no templates, no essential fields, no route rule.
- The uneven skipped-tenant data seen in the DB comes from what tenants did AFTERWARDS on the `/agents` empty state (`cloneDefaultLibrary` / `approveAgentConfig`), **not from Skip.**
- Middleware only forces onboarding when status is NULL, so both `completed` and `skipped` let the owner into the app, and a skipped tenant can return to guided setup.
- **Tommy Movers explained:** it FINISHED (status `completed`, which Skip cannot set) between `e037703` (2026-08-12, when Finish began passing `p_survey_drive_miles` defaulted to "15" with no depot capture) and `52118cf` (2026-09-14, server check + depot write reordered). A pre-fix artefact, not a Skip artefact. **Question closed.**
- **Remaining doors to the broken state, neither of them Skip:** Settings › General clear (now warned, still permitted by design) and the **Local Rules editor**, which writes a `drive_miles` entry with no depot check and no warning → **new S297**.

### 6.2 The scoping pass that killed live-partial as first conceived

- **NOTHING is generated during onboarding.** Categories and templates come from a FIXED library in code (`movers.ts`, 9 categories / 7 templates), filtered by `selectedServices` inside `completeAgentOnboarding` at Finish. No AI. The only AI calls during onboarding are website prefill and name suggestions, and neither produces categories or templates.
- **So at Skip time there is no generated setup to rescue — only raw form values.** The whole library rebuilds from the service selection at any later point.
- The RPC WILL accept a partial payload (empty arrays insert nothing; it fails only on a missing inbound address or invalid status).
- **Three traps a naive live-partial would hit:**
  1. **Re-running only ADDS, never removes or updates** (categories `on conflict do nothing`; templates skipped by name; essential fields `do nothing`). Services dropped between a partial and a later Finish leave their categories/templates/fields ACTIVE and the classifier still routes to them. The one refresh block targets `quote_request`, renamed to `removals` in s151, so it matches nothing — **already-latent bug on the normal path too.**
  2. **Creating an agent HIDES the way back:** `/agents` shows `EmptyStateChooser` (holder of the only in-app "Guided setup" link) only when there is no agent.
  3. **The depot check lives only in `completeAgentOnboarding`,** so a partial path passing mileage without a depot would recreate the exact broken state this session fixed.
- Also: `completeAgentOnboarding` can't be reused as-is (rejects blank name + empty routes, always sets `completed`), and the RPC's config write **overwrites every field it covers including nulls** — a partial passing `p_onboarding_status: null` would reset the tenant and bounce them back into onboarding.
- **Either way the agent starts DISABLED** — same as Finish.

---

## 7. Decisions taken in chat

- **No "send anyway" override on the missing-depot hold.** A knowing override would keep a distance limit alive while silently not enforcing it. The agency-preserving exits already exist and leave the system coherent: add a depot, remove the limit, or send the held item by hand from the review queue.
- **Depot removal = warn + confirm, NOT hard block.** Pricing fails safe, and a tenant legitimately running no depot must be able to clear it.
- **Onboarding needs NO change for the skip/require tension.** Skip can't create a broken rule, and depot is already required on the Continue path when a survey mileage is entered. Hard-requiring a depot to finish would only punish the explorer, for no safety gain.
- **Live-partial as first conceived is rejected; the reconciling version is adopted** — see F51 below.

---

## 8. F51 — the design that came out of it (NEW, not built)

**Operator intent:** capture as much data as possible whether they skip or not, make the system work with whatever was collected, and mitigate the problems rather than discard the data. Every problem above answered:

| # | Problem | Solution |
|---|---|---|
| 1 | Re-run only adds, never removes (stale leftovers) | Make provisioning **reconcile**: insert-if-missing, reactivate-if-inactive, **deactivate** library items the current selection no longer wants (never delete; never touch tenant-created/edited rows — needs a library-origin marker if absent). Also fix the dead `quote_request` refresh block. **Fixes a latent bug on the normal Finish path too.** |
| 2 | Partial provision hides the "Guided setup" way back | **F9 is the way back.** Key the checklist and the guided-setup entry on `agent_onboarding_status = 'skipped'` / actual missing state, NOT on "no agent exists". |
| 3 | Mileage-without-depot recreates the broken rule | Block Skip ONLY in that one combo: survey is a chosen route AND a mileage was genuinely entered AND no confirmed depot. The gate Continue already enforces, applied to Skip. Item 1's hold remains the net. |
| 4 | Agent starts disabled | Not a problem — same as Finish, and it's a safety feature. Becomes a checklist item: "Review and enable your agent." |
| 5 | `completeAgentOnboarding` can't be reused | New lenient `provisionPartialOnboarding` action: tolerates blank name (default signature) and empty routes (no route rule), sets `skipped` not `completed`. |
| 6 | Config write null-overwrites; null status bounces them back | Partial action never sends nulls: call the RPC with `p_write_config = false` and write only entered fields via a targeted update (the pattern depot already uses). Set `skipped` explicitly. **No RPC signature change → no overload migration.** |
| 7 | Untouched defaults look like choices (tone, mileage "15", exclusive) | Track **visited steps**; the partial payload includes only values from steps actually reached. Untouched mileage = blank = DEC29 "no limit" → no rule, no depot needed, no block. |
| 8 | Depot may be unconfirmed at skip time | Only a **resolved-and-confirmed** depot counts. Unconfirmed = absent → checklist item (and, with #3, must be confirmed or the limit dropped before skipping). |

**Shape:** Slice A = reconcile-provisioning + partial action + visited-steps + skip block (reconcile FIRST, it's a live bug). Slice B = F9 itself. **F9 is a co-requisite** — without the checklist, problem 2 is real — so build F9 first or ship both together.

**F9 pulled into LAUNCH SCOPE** on its own merits: onboarding deliberately omits pricing (too fiddly), so without a checklist a paying tenant never learns their quote engine needs pricing set up.

---

## 9. Verification (all on prod, operator-run)

**Depot arc — 10/10 checks green.** Onboarding: single approved copy line; typed address resolves + confirms; junk blocked with postcode hint; dropdown pick straight through; Edit returns to field. Settings: same four, plus stale-coords retirement (edit a saved depot to a different address without picking → new coordinates written, old gone), plus clear-saves-empty. Removal warning: both lines on a survey tenant (S199 Movers), pricing line only on a no-survey tenant (Testing Mover), Confirm cleared all three fields (**MCP-verified null/null/null**), Keep depot left it intact, normal address change did not false-trigger.

**PR #119 — NOT exercised live. Accepted on tests (operator decision).** Five environmental blockers in a row, none of them the fix: (1) test emails sent to `s199-movers@inbound.moversapp.app`, which is **dormant** for that tenant — its live inbox is the connected mailbox `hello@cornwallmovers.co.uk` (source `mailbox`); (2) **OpenAI ran out of credits mid-session**, 429 on every classification across the tenant until topped up; (3) once routed correctly, extraction returned every field empty on that one run, so `distance_status` became `no_postcode` (masking `no_depot`) and routes were `list`+`photos` — no survey offered; (4) **S199 Movers has auto-send OFF**, so its drafts hold as `auto_send_off` regardless and it can NEVER demonstrate the no-depot hold as distinct; **only S146 Mover has `auto_send_enabled = true`**; (5) a **stale client bundle** made the shipped removal warning look broken until a hard refresh. The `no_depot` branch is directly unit-tested and the pipeline was proven working end to end (classify → draft → queue → hold), so accepted.

**Extraction scare closed:** S199 Movers extracted pickup postcodes cleanly on 09-12, 09-13 and twice on 09-14 (`distance_status = resolved`). The all-empty run was a fluke — most likely the just-dry OpenAI account or the line break inside the pasted test address. Not a bug.

---

## 10. Quirks worth logging

- **After a deploy, HARD REFRESH before testing.** A cached client bundle plus a new feature looks IDENTICAL to "the feature doesn't work" — cost a full false-bug detour on `cffe546`, including two real depot clears (both tenants' depots wiped) because the un-refreshed page saved immediately instead of showing the confirm.
- **The Claude Code harness `page.tsx` phantom fired on all four builds** (+40, +56, +49, +72, all `-0`) exactly as logged 2026-09-11 v2. Correctly ignored every time. Rule holds: trust `git diff --stat origin/main..main`, never the agent's file card.
- **A dormant letterbox and a live mailbox look the same from the operator's side.** `status = letterbox_dormant` means accepted-but-not-processed; check `to_address` + `source` before concluding a pipeline failure.
- **Restore-before-you-break.** Snapshot a tenant's depot before clearing it for a test. S199 Movers was recoverable; Testing Mover's original address was lost because it wasn't captured first.

---

## 11. Open tails from this session

- **F11(d) status contradiction** (backlog says DONE, launch plan carries it open) — operator check owed.
- **S153 / S154** still open; confirmed test-tenant-only, so no longer urgent. Real remaining question is whether the Advanced-tab add-category UI can produce a template-less category (likelier real-world trigger than provisioning).
- **OpenAI billing** — ran dry on prod mid-session and silently killed every classification. Reliable billing + **S253**'s ENFORCED org hard cap matters more than the backlog's "do soon" framing suggests.
- **S296 amendment:** the read-only Settings view renders `—` (em dash) for an empty company name or depot, against house style. Fold into S296's existing GeneralForm em-dash sweep.
- **DEC55 CANDIDATE (needs operator lock):** "Skip provisions a reconciled partial from visited steps only; the survey distance rule requires a confirmed depot; F9 is the completion surface." Flagged, not locked.
