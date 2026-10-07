# Movers App — Session Summary 2026-09-22 (covers the 2026-09-21 evening parity work too)

**Quote Engine verification, the manual override, VAT done properly, a customer-facing quote, and the moat's big slice.** Ten ships, three merged PRs, no migrations. main `7f3a3f8` -> `162d285` (S339, direct) -> `5abe711` (S340, direct) -> `51dc646` (PR #134, F56) -> `825b4c2` (S341, direct; helper-text follow-up after) -> `763eea7` (S342) -> `49af8dc` (S343) -> `8ba8cfa` (S344) -> `1750864` (PR #135, F57) -> `52aafed` (PR #136, F31 PR 2c-2b). Test suite 441 -> 844. Operator hand-verified F56, S341, S343, F57 and 2c-2b on preview.

## 1. Quote Engine parity, answered by machine (S339 + S340, closes S127)

S127 had sat open since 2026-08-09: does the app reproduce the v5 workbook? Nobody had checked. Answered this session without the operator running through anything by hand.

- **Oracle from the spreadsheet itself.** 56 scenarios written into `Cornwall_Movers_Quote_Engine_v5.xlsx` cells, recalculated headless in LibreOffice, outputs read back. Baseline reproduced the Excel-saved values, so the recalculated sheet is a trustworthy answer key. Coverage: team 1 to 6, all four access types, every band edge (0/50/51/139/140/299/300/301), min and max profit clamps, van combos, the status ladder, rounding traps (travel landing exactly on 2.75h and 5.5h, weight on .5).
- **S339 (`162d285`)**: `computeQuote.parity.test.ts`, 52 core scenarios through `buildEngineConfig` (DB-shaped rows) then `computeQuote`. **52/52 to the penny.** The DB-row mapping is under test, not just the maths. Four "edge" scenarios reported, not asserted.
- **S340 (`5abe711`)**: settings sensitivity (32 fields, each bumped in turn, each moves its output; **no dead settings**), catalogue reference (165/165 items match the Cubic tab on volume, dimensions, specialist flag; 90/90 aliases valid), parser snapshot (40 inputs, characterisation for the 2c-2b move), and a read-only wiring trace from screen to engine.
- **Wiring trace findings:** miles are round trip in both; travel time is round trip in both (the "(one way)" label was wrong in the sheet and the app; formulas never doubled it); route-lookup miles are integer-rounded but hand-typed miles are not; band lookup falls to the LAST band on no match.
- **Pre-v5 sheet.** The operator surfaced the "Ultimate Moving Quote Calculator" (Google Sheet) that v5 was built from. Compared cell by cell: same bands/markups/clamps/VAT/travel/wages; v5 changed fuel 0.55 -> 0.60, replaced a hand-typed efficiency table with a full ladder, replaced hand-typed fractional vans with volume fill x access, added Cubic + Bridge (weight). All deliberate except possibly fuel and efficiencies, both tenant settings. The hand-typed vans figure became F56.

**Verification lesson locked:** `computeQuote` is pure, so "run it by hand" adds nothing over 52 fixtures. What CAN differ is the wiring around it, and that is what Part A traced. The manual-run rule stays for pipelines with external services.

## 2. F56 vehicle-loads manual override (PR #134, `51dc646`)

Cornwall's manager surveys jobs himself and types a vehicle figure (0.7, 1.3, 2.5) straight into the old sheet. The app forced him through the item list. Shipped in three commits on one branch:

- **Engine:** `overrideOn` + `vehicleLoads: Record<vehicleId, number>`. Active only when on AND at least one ticked vehicle has loads > 0. `loadUnload = ceil(sum_v(base_hours_v / ((team/2)·eff) · loads_v) · accessMult)`. One ceil on the total (matches the sheet oracle). Per-vehicle, so an 18t lorry and two 3.5t vans each carry their own loads. Fuel, payload, weight, fill, band, clamps unchanged from ticked vehicles. Off path byte-identical; S339/S340 untouched.
- **First cut was a single "van loads" number with a mean-of-fleet time per vehicle.** Operator caught it before merge: a mean is meaningless for a mixed fleet. Reworked to per-vehicle before merge. Also renamed van -> vehicle throughout (tenants have lorries).
- **UI:** "Manual override" `PillSwitch` (new shared component, none existed) at the top of Vehicles & fit; loads input beside each ticked vehicle with a 1.5s primary-colour highlight on flick-on; items section and volume/fill readouts fade (still editable, "Recorded on the quote, not used for pricing"); summary shows "Load basis: Manual" plus one line per vehicle. Owner/admin only via the same `canEditQuoteParams` rule Settings > Pricing uses; ignored server-side for other roles. Customer PDF never prints loads or basis; items always print (they form the contract).
- **Fixtures:** 17 override scenarios generated from the Ultimate sheet via LibreOffice (`qe_override_fixtures_v1.json`), each run twice (no items vs 300 ft3), plus mixed-fleet and non-Standard-access formula asserts.
- **Post-merge fixes (`eb64271`):** the price gate `V > 0 && miles > 0 && vehicles > 0` hid the price, blocked save and export for override-only quotes; replaced with shared `quoteHasPrice` (V > 0 OR surveyed) at all four surfaces. Travel labels relabelled "Travel time" / "Travel hours" (round trip is the truth). `travelOneWay` engine field name kept (parity tests).
- Design rule carried forward: everything the engine needs lives in one plain quote state so a future LLM auto-quote path fills the same structure and leaves the override blank.

## 3. S341 VAT settings (`825b4c2` + helper-text follow-up)

The Pricing form showed "vat_rate 0.1". Two different numbers were being conflated: the HMRC cost the engine deducts (flat rate = X% of gross; standard = one sixth of gross) and the 20% a customer sees. Shipped: `vat_registered` switch (PillSwitch), scheme radio (Flat rate with % input, default 10; Standard 20%), `vat_rate` now DERIVED on save (0 / pct/100 / 0.166667). `normalizePricing` backfills old rows from `vat_rate` (Cornwall 0.1 -> registered, flat, 10; no price change). Engine and parity tests untouched. Operator tested flat / standard / off: internal and customer documents both correct, and different where they should be.

Noted, not changed: the seed migration still writes `vat_rate: 0.1`, so a new tenant starts VAT-registered by default (S350). The customer-facing 20% is hardcoded (UK standard rate) (S349).

## 4. Job-card carry-over polish (S342, S343, S344)

- **S342 (`763eea7`)**: amber "Manual quote" pill on the board row and card drawer when the linked quote's `loadBasis` is surveyed. Discovery: `quotes.created_by` already exists; quotes and cards are linked both ways; `payload.quote.loadBasis` is queryable. So per-user override rate for Reports needs no schema change (S351).
- **S343 (`49af8dc`)**: Customer name / ref was a borderless input (missing `.num` class rule); fixed. Job card created from a quote now carries `total_pence` from the price.
- **S344 (`8ba8cfa`)**: deposit prefilled on create from `jobs_settings.deposit_percentage` (46608 @ 25% -> 11652). Drawer onBlur behaviour untouched; create path leaves deposit null when the percentage is unset (drawer falls back to 50%: a known, accepted difference).

## 5. F57 customer quote PDF, tenant branded (PR #135, `1750864`)

The only report printed base cost, profit, band, fill, and was hardcoded "Cornwall Movers" / "Quote Engine v5". Shipped: internal report de-Cornwalled (company name from Business Info, `catalogueVersion` dropped, Q ref in the header, UUID demoted to the footer); new "Customer quote (PDF)" button: logo (`branding.logo_url`, public bucket, no migration) + company + address/phone/email/website, Quote/ref/date, customer + addresses + move date, inventory grouped by room with specialist markers, original list, then Net / VAT (20%) / Total if registered or Total only if not. Nothing internal, no "Quote Engine", no "AI". `vat_registered` captured in the quote payload at save so old documents keep the VAT position they were saved under. Shared render extracted to `pdfRender.ts`.

## 6. F31 PR 2c-2b: catalogue, aliases and matching server-side (PR #136, `52aafed`)

The moat's big slice, built to the 2026-09-21 design in a fresh Claude Code chat.

- **Server:** `lib/quote/catalogue.ts` (server-only; `match.ts` ported verbatim + JSON), four gated routes under `app/api/quote/catalogue/` via a shared `_gate.ts` (auth + tenant + billing): `search?q=` (min 2 chars, max 25, empty/wildcard 400), `resolve`, `by-ids` (max 200), `warm` (full alias-free item list for the device cache, 5/hour per user). 60/min per user via the existing `checkRateLimit`. Client `match.ts` and both JSON imports deleted.
- **Client:** grid items self-describing `{id,name,section,unit_ft3,specialist,qty}`; `cubicV`/snapshot/specialist/PDF/LLM context read the item, no index on the device. IndexedDB `qe-catalogue` (`catalogueCache.ts` + `useCatalogue.ts`), version-stamped, purged and re-warmed on drift. `LOAD_RECORD` hydrates new items directly, legacy `{id,qty}` via cache then by-ids, else an "item details unavailable" row. Manual add is one debounced `ItemSearchPicker` (alias-aware online, name-only offline).
- **Invariants held:** `computeQuote` untouched; S339/S340 fixture tests untouched and green; parser snapshot byte-identical (only the import moved); catalogue reference 165/165 via the server module. **Bundle grep of `.next/static`: zero hits** for "Treadle sewing machine", "box-unsized-default", "settee". The anonymous one-file download is closed.
- **Operator offline pass (desktop DevTools Offline):** picker serves from cache; Convert fails offline (matcher is server-side by design; copy changed from raw "Failed to fetch" to "Convert needs a connection. Offline, search items by name below." plus an offline note under the search box). "settee" offline = no match (aliases stay on the server, name-only offline: accepted). A page RELOAD offline does not load: the F14 PWA does not cache the quote app shell (F58). The connect-once state cannot be forced from DevTools (an open page keeps the catalogue in memory past a site-data clear); unit-tested only.
- **Honest bar, restated:** an authenticated user can still enumerate the catalogue via `warm` or many capped searches. The win is closing the anonymous download.

## 7. Housekeeping and process

- S339 and S340 were committed to local main and never pushed; they rode inside PR #134 until caught. Pushed main, force-pushed the branch to resync the PR. **Rule restated in every brief since: direct-to-main items commit AND push.**
- Claude Code builds throwaway login-free harness pages to smoke-test UI (`page.tsx +81` etc). It deleted them each time, but every merge prompt now asks for the PR file list to prove it.
- Squash merges are the repo convention; merge prompts say so.

## 8. Backlog changes (see master backlog for wording)

- **DONE:** S127 (via S339/S340), S339, S340, S341, S342, S343, S344, F56, F57, F31 PR 2c-2b.
- **NEW OPEN:** S345 (band-gap + hand-typed miles), S346 (parser oddities: grand piano -> baby grand, bare fridge -> medium, "3, seater" -> qty 1), S347 (fleet/band rows uncoerced into `buildEngineConfig`; confirm int8/float8 not numeric), S348 (e2e harness with Testing Mover login: save/load/offline/override/PDF), S349 (customer VAT 20% hardcoded; becomes a setting for non-UK or a rate change), S350 (new-tenant VAT seed defaults to registered; decide), S351 (Reports: override rate per user from `loadBasis` + `created_by`), F58 (offline app shell for the quote page: F14 PWA does not cache it). S336 unblocked (matcher is server-side now).
- No new DEC. Candidate DEC58: "pricing maths stays on the device; catalogue, aliases and matching are server-side" (built as designed 2026-09-21; not formally locked).

## 9. Next

1. 2c-2c polish, then the config-helper prompts slice, PR 2d, PR 3 (F31 remainder).
2. S336 catalogue coverage review (now cheap: server-side matcher).
3. S348 e2e harness would have saved most of today's manual clicking.
4. Cornwall non-email cutover is no longer gated by S127. Remaining gates are the launch-plan ones.
