# Movers App — Session Summary 2026-09-04

**main faaaa9b → 51837fa. Two items shipped: S247 (direct-to-main, 35382ec) + S83 (PR #87 squash, 51837fa). One migration applied live via MCP. no_depot discussed → PARKED as S249. Build-only session — no external Postmark/billing work.**

Verification done through the F12 test panel + live DB reads (Supabase MCP). GitHub connector unavailable all session (see §5); worked by file-paste.

---

## Headline outcomes

1. **S247 DONE (35382ec, direct to main).** Survey offer is now stripped from the draft when there's no usable pickup location (`distance_status = no_postcode`), so the standard "please send your details" reply SENDS instead of being held by the cron survey/distance guard.
2. **S83 DONE (51837fa, PR #87).** Reply-to override regex replaced with a validated domain ignore list. New `ignored_sender_domains text[]` column, migration applied + backfilled live. Legacy regex kept as a hidden backend fallback (Cornwall-safe).
3. **no_depot** (survey over-offer when a tenant has no depot) discussed and **PARKED → S249.** Not built; the "hard rule" shape is noted for when it's picked up.

---

## 1. S247 — strip survey when pickup location is missing

**Bug (from the 2026-09-03 v3 auto-fire session):** a no-postcode enquiry produced a draft that still offered an in-person survey, and the cron guard `hasSurveyRoute && hasUnresolvedDistance` then diverted it to `manual_hold` — holding the exact reply worth auto-firing fast (the "send us your addresses/postcode/date" ask).

**Fix (`stripSurveyWhenLocationMissing`, lib/agents/resolveQuoteRoutes.ts):** a pure helper applied at the `resolveQuoteRoutes` call site in `computeDraft.ts`. When `distance_status === "no_postcode"` it removes `survey` from the resolved routes. Because everything downstream (missing-info field filter, route sentence, persisted `quote_routes`) reads the one stripped array, the draft becomes coherent (asks for details, no survey offer) AND `hasSurveyRoute` is false at cron time, so the guard passes and the reply sends. route.ts was NOT touched (its `no_postcode` clause is now dead-but-safe, belt-and-braces).

**Scope:** `no_postcode` only (customer gave no location). `lookup_failed` (we had a location, our lookup broke = system fault) is unchanged — survey kept, cron still holds it. `no_depot` NOT touched → see S249.

**Verified (F12 panel, same input before/after):** pre-fix draft offered a survey in three places (route sentence "come out and take a look", the 3rd missing-info bullet "dates that suit you for a survey", and the MISSING ESSENTIAL INFO list). Post-fix all three dropped survey and self-healed the copy ("...whichever's easier"); everything else held (Removals, Quote request reply template, 90% confidence, still asks for both addresses + list/photos + access). 18/18 unit tests green.

---

## 2. S83 — reply-to regex → validated domain ignore list

**Why:** the regex field was a silent footgun — a mover typing `cornwallmovers.co.uk` without backslashes gets a regex where `.` matches any character, silently broadening the own-domain skip with no feedback.

**Shipped (PR #87, squash 51837fa, 9 files):**
- **Migration `20260904091527_s83_ignored_sender_domains.sql`** — `alter table public.tenant_agents add column ignored_sender_domains text[] not null default '{}'`. Applied to the live DB via Supabase MCP `apply_migration`; all **11 agent rows backfilled to `{}`**, verified. `database.types.ts` + `TenantAgent` updated.
- **`normaliseIgnoredSenderDomain` (loopPrevention.ts)** — pure, client-safe. Lowercases, strips protocol/mailto/path/trailing dot, extracts domain from a pasted email, validates against an RFC-ish domain regex. Returns `{ domain }` or `{ error }`.
- **`isOwnDomain` extended** — now also matches the sender domain against the ignore list, **exact OR subdomain** (`d === entry || d.endsWith("." + entry)`) — NOT substring, so no broadening. Contact-email match unchanged. **Legacy `reply_to_override_regex` branch kept as a hidden fallback** (Cornwall has a live one) — UI removed, backend behaviour preserved. Migrating Cornwall's regex to a list entry + deleting the branch is a separate later cleanup.
- **`updateIgnoredSenderDomains` server action** — owner-gated, normalises + dedupes before a whole-array write. Separate from the Overview form.
- **`updateAgentOverview` no longer writes `reply_to_override_regex`** (removed from payload + write). This is the anti-clobber fix — leaving it in would have nulled the column the first time a tenant saved the Overview form once the field was gone.
- **OverviewTab** — regex field removed (all state/dirty wiring), new "Ignored sender domains" card (add with inline validation, per-row Remove, own Save + status). Empty by default; helper copy tells the tenant their own domain is already skipped automatically, so they don't add it.
- **27 unit tests** (loopPrevention.test.ts) — normalise cases + exact/subdomain/non-substring matching + contact-email + legacy-regex-fallback.

**Storage decision:** column on `tenant_agents` (not `tenant_agent_local_rules`) because the own-domain skip is an early-exit BEFORE local_rules load — a column rides the existing agent select with zero extra query.

**No onboarding seed:** the tenant's own domain is already auto-skipped by the contact_email check, so the list is only for OTHER domains and correctly starts empty.

**Verified live:**
- Migration + 11/11 backfill (DB read).
- Anti-clobber (the one that protects Cornwall): set a unique sentinel `reply_to_override_regex` on Testing Mover via SQL, pressed the main Overview **Save** on the branch preview, read back — sentinel **unchanged**. Restored Testing Mover's original `cornwallmovers\.co\.uk` after.
- UI field removed (confirmed gone on the branch).
- Persist: added a domain via the new card, Save + reload → column held one clean bare domain (lowercased/normalised on write).
- **Live skip leg NOT run** — the F12 panel doesn't expose a From address, so `isOwnDomain` can't be exercised there. The 27 unit tests carry the match logic; a real inbound from an ignored domain is the only true live check and wasn't staged (not blocking).

---

## 3. no_depot — discussed, PARKED as S249

**The gap:** when a tenant offers survey with a mileage rule but has **no depot address**, distance can't resolve (`distance_status = no_depot`), so the draft offers a survey the app can't size — and the cron guard does NOT catch `no_depot` (only `no_postcode`/`lookup_failed`), so it currently sends with the survey offer. Different root from S247: the CUSTOMER gave a location; the TENANT is misconfigured.

**Partial existing guard:** S199 blocks onboarding Finish when survey + mileage are set with no depot. So the main door is plugged; the Rules-tab door (set mileage later, no depot) and the Skip-out-of-onboarding back door remain open. Rare, but reachable.

**The "hard rule" shape (noted, not built):** offer a survey ONLY when distance actually resolved OR the tenant runs surveys with no mileage limit; otherwise strip it. Cleanly closes `no_postcode` + `no_depot` + `lookup_failed` in one rule. **Must be scoped to avoid breaking DEC29** (blank mileage = no-limit tenants have no drive_miles rule → `distance_status = null` → survey must stay). In code that's "strip when `distance_status` is one of `no_postcode` / `no_depot` / `lookup_failed`" (i.e. non-null failed-resolution), which does not touch `null`.

**Open sub-decision (also parked):** if survey is stripped on `lookup_failed` too, the cron guard can't hold it, so it'd send. Choice A = all three send (simplest, but a transient geocode failure silently degrades a full-info reply). Choice B = strip on all three but keep `lookup_failed` HELD via a guard keyed on `distance_status` directly (system fault gets human eyes). Leaning B when picked up.

**Decision this session:** park. Logged as S249.

---

## 4. Migration tracking (S130 note)

The S83 migration was applied with MCP **`apply_migration`** (name `s83_ignored_sender_domains`), NOT the usual by-hand `execute_sql`. `apply_migration` **does write a `schema_migrations` ledger row** — so this may be the FIRST migration this project that is NOT an orphan. Repo file is `20260904091527_s83_ignored_sender_domains.sql`; the applied name/timestamp may differ. Reconcile against the S130 orphan pile when it's next touched.

---

## 5. GitHub connector — persistent failure (for future sessions)

GitHub Integration shows **connected** in Manage connectors (status ✓) but its tools **never load into any chat** — confirmed absent from the tool list this session, and the operator reports it's been this way for days across multiple new chats, including a fresh reconnect. Not a per-chat toggle (Tool access only offers load-timing) and not a stale-binding issue (persists across new chats). Reads as a product defect, not operator config. Worked around all session by pasting files/greps. Worth a bug report to Anthropic with the Manage-connectors + in-chat-menu screenshots. Don't burn session time re-toggling it.

---

## Numbering

- **S247 DONE** (35382ec, direct to main).
- **S83 DONE** (51837fa, PR #87).
- **S249 NEW** — no_depot survey over-offer (PARKED; hard-rule shape + lookup_failed send/hold sub-decision recorded above).
- Next free S = **S250**. F30 free. DEC43 free (still the S248 candidate; no_depot did not lock a DEC).

## State for next session

- **S247/S83 both live on main and merged** — no follow-up owed except the optional S83 live-skip check (real inbound from an ignored domain) if ever wanted.
- **S249 (no_depot)** parked — decide Choice A/B and the DEC29-safe strip scope when picked up.
- **Tier-1 still open** (unchanged by this session): F28/F29 final delivered-green (blocked on Postmark approval, S243), S244 (bounce/complaint suppression before heavy sending), S191 → S194 (failed-run observability), F21 (marketing site + privacy, blocked on billing), Cand. D (billing decision).
- **Greeting name-resolution observation** from 2026-09-03 v3 (possible S87/S90 regression) still un-investigated.
