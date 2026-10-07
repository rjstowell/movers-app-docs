# Movers App — Session Summary, 2026-09-16

**Shape:** built and shipped **F51 end to end** across two PRs, plus its co-requisite **F9**. Two features, one migration (applied by hand to the single dev/prod DB). main `cffe546` → `4b27c9d` (PR #120, F51 Slice A, squash) → `6877c34` (PR #121, F9 / Slice B, squash). Verification loop = real onboarding runs on dev tenant `4d4ab07b-8a9b-4050-9139-6f0cd6f369c3` + Supabase MCP reads/wipes.

**Headline:** **F51 DONE** (partial provisioning on Skip) and **F9 DONE** (dashboard setup checklist + pricing confirm banner). **Reconcile was CARVED OUT of Slice A into its own future feature (F52)** once discovery proved it isn't a live bug and has no safe foundation yet. A long skip-block "failure" chase turned out to be a committed-vs-placeholder `"15"` default, not a code defect. Two decisions still owed the operator: **DEC55** (revised by the reconcile split) and the **F11(d)** contradiction.

---

## 1. F51 slicing + the reconcile split (the important reframe)

The Slice A brief carried "reconcile FIRST, it is a live bug." A read-only discovery gate proved that **wrong**, and the gate did its job by stopping before any build:

- **The dead `removals` block is a subject/body REFRESH, not a removal.** There is **no deactivation logic anywhere today**, so "reconcile" (deactivate deselected library rows) is **net-new behaviour**, not a repair. The urgency premise evaporated.
- **No library-origin marker exists.** `is_system` on `tenant_agent_categories` flags only the protected `ignore`/`escalate` classifier categories (s34); seeded service categories are `is_system=false`, identical to tenant-created. `tenant_agent_templates` has no marker (`created_by` NULL-for-seeded is incidental, not designed, and blind to *edited* rows). `tenant_agent_essential_info_fields` has none. Reconcile cannot safely deactivate without one.

**Consequence:** reconcile split OUT of Slice A into its own item (**F52**). It's the heaviest piece (marker → migration → backfill → edited-row semantics), the least urgent (nothing re-runs onboarding until F9 makes it easy), and nothing else depended on it. Slice A shipped as the four ready pieces. **Marker decision for F52 when we build it:** an explicit origin marker AND an *edited* signal (a seeded row the tenant later edits must survive deactivation). Codex's option 2 (scope to the known finite library key set) was **rejected** — it can't protect a tenant-edited library row whose key collides with a library key.

## 2. F51 Slice A — SHIPPED (PR #120, squash `4b27c9d`)

Four pieces, no migration:
- **`provisionPartialOnboarding`** — lenient Skip action, reuses the existing additive RPC with `p_write_config=false`, then writes only entered fields to `tenant_config` via a targeted upsert with `status='skipped'`. No RPC signature change. A failed depot write leaves the owner in onboarding to retry.
- **Visited-steps + `mileageTouched`** — the partial write includes a value only if its step was reached; the default `"15"` is never persisted unless the owner actually typed a mileage.
- **Skip-block** — Skip is blocked only in the one unmeasurable combo (survey route + genuinely-entered mileage + no confirmed depot), with an in-dialog warning. Everything else skips freely.
- **contact_email into onboarding** — folded in (cheap, we were already editing the flow); written on both Finish and partial paths.

**Doc correction found in discovery:** Skip writes only `agent_onboarding_status='skipped'` — NOT also `website_url` as the old backlog/summary claimed.

## 3. The "15" chase — three "fails" that weren't (test-method lesson)

Skip-block read as broken across several manual runs. Root cause was **not** the code: the survey-distance field shipped a **committed value of `15`**, not a placeholder. Untouched, that reads to a tester (and the owner) as "my cap is 15", but the system correctly treats untouched-as-blank (`mileageTouched=false` → no `drive_miles` → DEC29 no-limit) and so does not block. Every "it let me skip" run was correct behaviour with an unentered mileage. Once a value was genuinely **typed** (distinct number, not the default), the block fired exactly as designed.

**Fixes (follow-up commits on the same branch, in PR #120):**
- Survey-distance field made honestly empty, `"15"` as **placeholder only** — a DEC29 reaffirmation (blank = no-limit stays the safe default; a silent 15-mile cap under-offers, the invisible failure).
- Skip-block warning copy tightened ("Confirm a depot address or clear the distance before skipping.", stray commas removed) + spacing off the buttons.

**Lesson:** a committed default that displays as a chosen value but is treated as blank is a test-cycle trap. Placeholder, not committed value, when untouched means "unset".

## 4. F9 / Slice B — SHIPPED (PR #121, squash `6877c34`)

Setup checklist widget + pricing confirm banner. **Migration `20261220090000_f9_setup_checklist.sql`**: `tenant_config.pricing_approved_at` + new `tenant_setup_dismissals` table (RLS copied from `tenant_sending_identity`). Applied by hand to the dev DB via MCP.

**Row model (locked in design, verified live):**
- **Detected** (auto-tick from real state): onboarding (`agent_onboarding_status='completed'`), pricing (`pricing_approved_at` not null), vehicles (`fleet_vehicles`>0), email sending (`getTenantSendingIdentityStatus().verified`), business details (depot + contact present).
- **Detected-or-dismissible:** review & enable agent (`tenant_agents.enabled=true` OR dismissed; either satisfies).
- **Dismissible prompt:** add your team (persisted per-tenant dismissal; no real signal).
- **Non-counted links:** "see what your agent writes" (links the F12 preview, never ticks — F12 has no persisted done-state), Connections ("coming soon", greyed).
- **Collapse:** widget renders nothing when all *counted* rows are done + all dismissibles cleared. Gone = the signal.
- **Recovery entry:** the onboarding row's not-done state IS the guided-setup link, keyed on `agent_onboarding_status != 'completed'` (owner-only; admins are bounced from the onboarding page). This is the fix for the hidden-way-back problem F51's partial provision would otherwise create.

**Pricing confirm banner (1C):** above the rail on `/settings/pricing`, shown only while `pricing_approved_at IS NULL`; confirm is a **timestamp-only, idempotent** write that never touches the four rail tabs' values; banner gone forever once stamped (only a full account reset clears it). Not tab-gated. This is the only "pricing is set up" signal in the schema — `tenant_config.pricing` + `distance_bands` are seeded at tenant creation, so "rows exist" is always true.

## 5. F9 iteration (manual testing → follow-up fixes, all on PR #121)

- **Resume onboarding at the right step** — the recovery link restarted at step 1, contradicting its "pick up where you left off" copy. Fixed via **approach A: derive resume position from persisted `tenant_config`** (open the step after the last one with saved data), no new column. Note: data set later via Settings (e.g. `contact_email`, `website_url`) counts as a completed step on resume — accepted as correct (finish setup, don't replay the wizard).
- **"Dismiss" → "Mark done"** on both dismissible rows (label only).
- **"See what your agent writes"** now deep-links to `/agents?tab=generate#test-enquiry` and highlights the tester card (reused `useHashHighlight`). Still non-counted.
- **Card width** — capped to one tile's width, left-aligned, on its own full-bleed row above the Inbox + System health row (NOT a grid slot, which would leave an empty gap and orphan its partner on collapse).
- **Connections "coming soon" pill** — reuses the sidebar Reports pill classes (dropped the hand-rolled one).
- **Card accent** — 1px `--brand-primary` border + a soft low-opacity blue outer glow (`box-shadow` layered on `--shadow-lift`), neutral background (no tint — a fill would read as an error state next to System health).

## 6. Collapse verification + the email-sending dead end (quirk)

Couldn't flip the email-sending row by inserting a `verified` `tenant_sending_identity` row: it stayed "To do", and the app's own "sending from a generic address" banner agreed. Comparison against the one genuinely-verified tenant showed the difference is a real numeric provider domain id + DNS snapshot; **`getTenantSendingIdentityStatus().verified` is a LIVE provider check keyed on `provider_domain_id`, not the DB `verification_status` column.** So a hand-inserted row can never read as verified — real DNS/provider setup would be needed. **F9 is correct** (reads the same signal as the app banner). **Collapse accepted as unit-test-verified** (`buildSetupChecklist` covers it) plus 6 of 7 counted rows flipping live; faking a live provider verify to eyeball the last row wasn't worth it.

## 7. Decisions, deferrals, still-owed

- **DEC29 REAFFIRMED** — survey-distance blank = no-limit; the field carries a placeholder, never a committed default.
- **D1 → F53 (own feature, NOT folded into F9):** first-time agent-enable walks the owner through a "confirm you've checked everything" flow before the toggle switches on. Lives at the enable control on the Agents page; F9 only reflects on/off state. Needs a short design pass before a brief.
- **Glow-pulse** — operator wants the tester-card highlight to pulse in/out (3× slow) rather than a static glow. Spec parked onto the **existing glow/highlight backlog item**; `useHashHighlight` consumers (incl. "see what your agent writes") inherit it once built.
- **DEC55 STILL A CANDIDATE, needs operator lock** — the reconcile split changes its wording: Skip provisions a partial from visited steps only (reconcile deferred to F52); the survey rule requires a confirmed depot; F9 is the completion surface.
- **F11(d) contradiction STILL OWED** — backlog reads DONE (2026-08-27 v2), launch plan carries it open (session 8). Not resolved this session.
- **No theming exists in the app** (single `:root`, no dark mode anywhere) — the F9 border/glow use fixed brand tokens; re-check if a dark theme is ever added.
- **`schema_migrations` desync** — the F9 migration was applied by hand to the dev DB; only one Supabase project exists (`movers-app-dev`), so it is also prod, nothing separate to apply. Register it on whatever the prod migration path is so a future deploy doesn't try to re-run (it's idempotent: `add column if not exists` / `create table if not exists`).

## 8. New / updated backlog items

- **F51 DONE** (Slices A `4b27c9d` + B `6877c34`).
- **F9 DONE** (delivered as F51 Slice B, `6877c34`).
- **F52 NEW** — reconcile provisioning (deactivate deselected library rows). Needs an origin marker + an edited-signal (migration). Should land before F9's guided-setup makes re-running onboarding common. OPEN, not started.
- **F53 NEW** — guided first-time agent-enable review flow (D1). OPEN, design pending.
- **S298 NEW** — remove the now-uncalled `skipAgentOnboarding` (replaced by `provisionPartialOnboarding`, left in place; strip so it can't be re-wired by mistake). OPEN, small.
- **S299 NEW** — optional polish: the survey-distance empty-field warning is now always-on when empty (a side effect of the placeholder fix). Covers the Continue-with-blank case, so kept; only tighten if it nags. OPEN, optional.
- **Open question for F52/F9:** `drive_miles` does not persist on a partial-skip-with-depot (the survey rule carries routes but no mileage value). May be intended (partial writes routes, not the mileage sub-value) or a gap. Confirm during F52.
- **Candidate item (parked from the F9 test):** an end-to-end live email self-test (send a real message, confirm receipt) — overlaps F12(b)'s receive half. Not built, competes for launch scope on its own.
