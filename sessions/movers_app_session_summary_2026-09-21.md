# Session summary — 2026-09-21

**Big day. Seven ships, four merged PRs, two migrations by hand, a moat audit, and the Quote Engine IP work opened.** main `8603f65` -> `4279549` (S332, direct) -> `0792101` (PR #131, F31 PR 2b) -> `c556fa8` (S333, direct) -> `a8914f1` (PR #132, F31 PR 2c slice 1) -> `817b2ea` (S334, direct) -> `8e061dc` (S335, direct) -> `7f3a3f8` (PR #133, F31 PR 2c-2a). All verified on prod. Migrations applied via MCP: `20261223090000_f31_prompt_test_runs` and `20261224090000_f31_quote_tokens_used`.

## 1. PR A verification (S317 / S318, carried from 2026-09-18)

- **S317 pass on real traffic.** 59 mailbox runs since merge, ZERO with HTML-but-no-text, zero errors. Friday's check found no DMARC reports had arrived yet; the parked re-check today confirmed all four HTML-only DMARC reports since (09-19 to 09-21) carry derived text (684 chars each). "Re:" phone replies confirmed stripped (2098 -> 215).
- Emails with **no body at all** (2 DMARC-attachment-only + 2 Fuelcard) correctly ignored — that is S328's territory, not S317's.
- **S318 lines:** no new throttle rows, no new stranded runs, zero error runs with `error_cause` since merge — DEC57 backoff still unexercised on prod (no failures happened).
- **Anomaly parked against S327:** four unresolved S199 error runs (09-14/09-15 outage) have `auto_retry_count` 0, `last_auto_retry_at` null, NOT stranded — something excludes them from the auto-replay candidate query; cause unknown, harmless (3+ days old, shadow tenant), but "error run silently never retried and never stranded" is the exact hole S318 closes, so understand it when S327 runs.

## 2. F31 PR 2b — Prompts tab (PR #131, merged, `0792101`)

The console's first write surface. Draft / A/B test / publish / history / revert for the (then six, now nine) registry prompt keys.

- **`prompt_test_runs` table** = the spine must-test mechanism (mechanism detail under the DEC23 amendment, no new DEC). Append-only, FK to immutable `prompt_versions.id` so proof can never drift from the text published. `summary` jsonb holds outcome/category/confidence/wouldEscalate/missingFields/template per arm — NO enquiry text, NO draft text (rule enforced in app code, load-bearing). Publish of a `classifier_spine` VERSION requires >=1 test row for that exact version id; **null-version revert (code default) is exempt** — the emergency exit is never blocked.
- **Both resolver gaps closed:** empty/whitespace override falls back to code; candidates validated (missing token or empty -> ignored). Shared `validatePromptBody` (20k char cap) used by save, publish, and pre-test.
- **Metering bypass for operator tests:** AsyncLocalStorage `runWithMeteringDisabled` + two guards in openai.ts. Defaults false, no env/module switch, unit-tested that real traffic still meters. Operator test tokens spend at OpenAI but appear nowhere — accepted.
- **Tests can only run against a SAVED draft** (version id needed for the proof row). Save -> test -> publish flow stated in the UI explainer.
- **Live spine probe proven end to end on prod:** marker draft -> A/B test vs Testing Mover (test row written, summary clean) -> publish (in-app confirm, "EVERY tenant", last-tested shown) -> marker visible in assembled prompt -> real email recorded `classifier_spine_override_version_id` in `classification` JSON -> revert to code -> marker gone -> next email null override -> history shows both publications. Reject paths proven: missing `{{CATEGORY_ENUM}}` named on save; untested spine publish rejected with clear message. Token counter untouched by A/B test (arithmetic-verified).
- **Key learning during the probe: preview and prod share ONE database, so publishing an override on a PREVIEW makes it live on PROD instantly** (main's pipeline already reads overrides since PR 2a). Turned into the better test — proved main honours overrides — but it is a standing hazard. Quirks §15.
- **Clarity follow-up (same PR, b684c9b):** dedicated Revert-to-code-default button (visible only when an override is live), tenant picker relabelled "Test using this tenant's details" + helper line (publish is always global), in-app modal replacing window.confirm (Cancel default focus, mobile-checked), four-line "how this works" explainer (alternative line 2 for keys with no console test). Driven by operator confusion during verification — tenant picker read as scope, revert hidden in a dropdown.
- Squash-merged matching #130 convention. Post-merge suite 432 green, prod deploy success, prod /admin/prompts confirmed by hand.

## 3. S332 — date-sensitive test frozen (direct, `4279549`)

`autoReplay.test.ts` fixture used an absolute `created_at` that aged past the 24h strand threshold over the weekend — suite red on every branch from 2026-09-21 with no code change. Pinned NOW via the repo's existing `vi.spyOn(Date,'now')` pattern, fixtures made relative, restored afterEach; proven date-proof by shifting the pinned instant a month. Test-only, 407 green on main, merged into #131 branch cleanly before its merge. **Side-finding: GitHub checks on this repo are Vercel build only — "green PR" never meant tests pass.** Noted against S324 (CI does not gate on unit tests).

## 4. Moat audit (read-only) + S333

Operator asked how protected the codebase/IP is. Full read-only audit of everything reaching the browser.

- **Clean:** all agent pipeline prompts server-only and absent from every chunk; no source maps (local + prod confirmed); NEXT_PUBLIC_ set safe; /public images only; prompt/classifier tables RLS service-role-only; browser anon-key reads own-tenant only; `tenant_config`/`distance_bands`/`fleet_vehicles` RLS verified live by operator-side MCP (own-tenant policies present — the one thing the agent couldn't check).
- **Exposed (was):** entire Quote Engine in the client bundle — three LLM system prompts, 165-item cubic catalogue + 90 aliases, `computeQuote` pricing logic; `/api/openai` a client-controlled LLM proxy on our key (auth-gated but any trial, unmetered); three ungated `feedback-actions` server actions on the admin client; no leaked-instruction scan before auto-send (parked, see New items); calibrated pricing defaults seeded to every trial (accepted — a tenant must see their own numbers).
- **S333 (direct, `c556fa8`):** the three feedback actions (`resolveAiFeedbackConflict`, `getPendingCorrections`, `checkAiFeedbackContradiction`) now gate via `requireOwner(<row>.tenant_id)` derived server-side, matching sibling `deleteAiFeedback`. 9 new tests, 441 green. **Sweep of all 18 other "use server" admin-client files: every one already gated — this was the only hole.** Two low notes parked (see S338). Push initially missed — caught, pushed, deploy green, hand-verified on Testing Mover (correction save + remove).
- **Operator challenged the IP ranking and was right:** the pricing maths is spreadsheet-reproducible commodity arithmetic; the CATALOGUE + ALIASES are the hard-to-get part (the trade associate's own was inaccessible precisely because it lived server-side). This reversal reshaped slice 2's design (see §7).

## 5. F31 PR 2c slice 1 — quote prompts server-side, /api/openai locked (PR #132, merged, `a8914f1`)

- Three quote prompts moved verbatim into the registry as `quote_resolve` / `quote_sanity` / `quote_chat` — editable/publishable in the Prompts tab, NOT in TESTABLE_KEYS ("no console test available"; explainer line: check by running a quote on Testing Mover). Prompts list now nine keys.
- `/api/openai` is a **named-job endpoint**: client sends `{ kind, user|messages, context }` only; server resolves the system prompt via `resolvePrompt(key)` and assembles. Rejects any system/prompt/instructions field, any non-user/assistant chat role, unknown kind. `context` (chat's quote-data block) capped 8 KB; accepted instruction-stuffing residual (context sits in system position; rate limit + output cap + recording bound the damage). Auth, billing, 32 KB (applied to the fully assembled message, matching old behaviour), 20/min all kept.
- **Quote tokens counted, never enforced:** new `quote_tokens_used` column + sibling RPC (SECURITY INVOKER; EXECUTE revoked from PUBLIC/anon/authenticated on BOTH counter functions — closed a latent PostgREST-RPC door on the email one too). `enforceTenantTokenCap` never called on this route; email cap can never be tripped by quote use. Recording failure logs and still returns the result. Operator Spend tab shows quote tokens per tenant/day ("counted, never capped"). The old exemption comment literally described this fix as "a separate slice" — premise confirmed.
- Byte-identity proven via pre-change baseline snapshots for all three assembled prompts; grep of built chunks: all three prompt phrases + "CURRENT QUOTE CONTEXT" ZERO hits. `DEFAULT_PROMPTS` / `SET_PROMPT` / client prompt state deleted.
- Verified by hand on preview: 20-line paste -> local match -> AI resolve -> sanity -> chat, dev-tools source search clean, nine Prompts rows, Spend shows 4,193 quote tokens. Email counter confirmed still incrementing post-grant-change (5,626 -> 8,457 on a real email). 458 green, merged squash, prod deploy success.
- **Cost sanity for the operator:** GPT-4.1 mini at $0.40/M in, $1.60/M out => an email run ~0.13p, a full quote AI session ~0.25p, S199's whole shadow week ~35p. Rule of thumb ~600 emails per £1. The cap is for runaway bugs, not normal use.

## 6. S334 + S335 — second-line verifier UX (direct, `817b2ea`, `8e061dc`)

During slice-1 verification the operator could not find the verifier input: the message-history panel looked like the text box, the real input was borderless plain text below. Read-only check first proved NO gate exists in code (prod behaved identically) — pure UI. S334: input styled like every other input, history panel recessed, click-to-focus. S335 (operator direction): history panel hidden until first message, example moved into the input placeholder, **"or pricing" dropped from the section description — the verifier is for the ITEM LIST, not judging the price** (operator decision; the price still reaches the model in context, accepted — only the invitation was wrong). Both live.

## 7. F31 PR 2c slice 2 — discovery done, design agreed, NOT built

Read-only discovery (report in the moat-audit Claude Code chat). Corrected one wrong premise in my brief: **slice 1 did NOT move the resolve catalogue block server-side — ItemsCard still built the 165-item reference client-side and sent it as the user message.** That became 2c-2a (§8).

**Agreed shape (operator + discovery, do not re-derive):**
- **Pricing maths STAYS on the device** (operator's correct call: it is not the IP; instant totals + on-site usability win). Catalogue + aliases are the IP and move server-side.
- Three endpoints, tenant-gated + rate-limited (~60/min, min query length 2, <=25 results, no wildcard-returns-all): `search?q=` (typeahead + manual pick), `resolve` (batch paste, ported `match.ts` verbatim + snapshot-tested byte-identical), `by-ids` (rehydrate loaded quotes).
- **Items on the quote become self-describing** ({id,name,section,unit_ft3,specialist}) — the linchpin, since `state.grid` stores only {id,qty} and every volume is currently looked up from the in-browser index. Metadata written through to a **per-device IndexedDB cache**, version-stamped with `catalogue.version`, purged on mismatch (never price on stale volumes). computeQuote/totals/snapshot/PDF read cache + session, 100% local and instant.
- **Warm-on-first-load accepted:** first authenticated online load background-fills the cache, so any device that has opened the page once keeps today's offline parity. Fresh never-connected device off-grid: clear "connect once" state, never silent. The honest bar: an authenticated scraper can still enumerate 165 items over many capped queries — the win is closing the anonymous one-file download, a bar not a wall.
- `LOAD_RECORD` must hydrate from cache/by-ids/record (add `id` to persisted items); manual-pick dropdowns become one debounced search picker; `quote_sanity` unaffected (reads snapshot only).
- **PR split: 2c-2a (shipped, §8) -> 2c-2b atomic (endpoints + cache + self-describing + picker + JSON removal, MUST ship as one — endpoints-without-cache is strictly worse on-site than today, explicitly ruled out) -> 2c-2c polish.** 2c-2b is the next big build, fresh session.
- Operator asked whether a PWA/home-screen app could hold the catalogue "more safely" offline — answered no: same browser engine, same storage, dev tools see all of it; there is no third place between device (exposed) and server (needs signal). The cache path is the resolution.

## 8. F31 PR 2c-2a — resolve reference server-side (PR #133, merged, `7f3a3f8`)

Client now sends only `{ lines: [{raw, qty}] }` for `kind:"resolve"`; new server module `lib/quote/resolveReference.ts` imports the catalogue JSON server-only and reassembles the exact user message (reference block + verbatim middle block incl. the ≤/→/× code points + lines). 32 KB cap applied to the fully assembled text — identical pass/fail to before. Byte-identity proven against a baseline captured BEFORE editing ItemsCard (empty/one/many/quantity corpora). Grep of built chunks: reference-block header + instruction phrase ZERO hits; ItemsCard's builder and its sole `catalogueIdx.items` use gone. Response shape / grid application untouched (server preserves line order, `i` = array index). Hand-verified on preview: resolve behaves as prod; Network payload shows only the skinny lines. 465 green, merged squash, prod deploy success. **Catalogue item DATA still ships via the matcher import — that is exactly 2c-2b.**

## 9. Verification method notes

- Every merge ran the pre-merge gate (head unchanged, mergeable-clean, Vercel green, match prior merge convention) — caught nothing today but cheap.
- My own false alarm: flagged `prompts.ts +338/-84` as possible scope creep; the agent proved it was the editor's running edit tally, not the commit — git said new file +254/-0, no Quote Engine files touched. Editor tallies are not diffs (quirks §15).
- The `rm -rf` near-miss (agent typo'd a path into the OTHER checkout and deleted its app/ folder; fully recovered via git restore because the tree was committed-clean) drove the "movers-app-2 only, never movers-app" hard line now present in every brief. Real fix is the operator moving/deleting the stale checkout (quirks §15).
- S333's brief omitted "push + confirm deploy" and the commit sat unpushed — the standing brief line (quirks 2026-09-02) applies to every S-item brief, no exceptions.

## 10. Operator actions this session

- Applied both migrations via MCP after review; verified grants/RLS/columns live post-apply.
- Hand-verified: PR #131 full checklist incl. live spine probe (a-h) + both reject paths; #132 quote flow + dev-tools grep + Prompts/Spend tabs; #133 resolve + Network payload; S333 feedback flow; S334/S335 on prod.
- 404-as-non-operator confirmed using the S199 login.
- Fed the 20-line standard test paste (now the reusable corpus for quote checks).

## 11. New items raised (numbered in the backlog)

- **S336** — catalogue coverage review: "50 inch tv", "bookcase tall", "misc bags of clothes", "kingsize wardrobe" all unrecognised by local matching; everyday mover vocabulary. Review alongside which "ambiguous" prompts are deliberate size-splits. Sequence with/after 2c-2b (matcher moves server-side anyway).
- **S337** — leaked-instruction scan before auto-send (moat item 6): only pre-send content check today is the {{variable}} regex; nothing scans a draft for leaked system-prompt text or injected instructions. Prompt-extraction-by-email is the likeliest real IP theft vector post-launch.
- **S338** — `moveCategory` (~L1541) and `approveAgentAdditions` (~L2007) in agents/actions.ts read sibling rows filtered only by client-supplied `agent_id` (writes are tenant-scoped, so no cross-tenant write; a mismatched agentId could surface another tenant's category sort_order/keys into reorder logic). Tighten when those are next touched. From the S333 sweep.
- **S327 amended** — add the four never-retried-never-stranded error runs (§1) to its scope: find the exclusion cause before/while sweeping.
- **S324 noted** — GitHub CI is Vercel-build-only; the S332 red suite was invisible to PR checks.

## 12. Open threads / next

1. **F31 PR 2c-2b** — the atomic catalogue-server-side slice (§7). Biggest remaining moat item. Fresh session, design already settled.
2. DMARC/S317 needs nothing further — closed.
3. Held from 2026-09-18: S316, S327 (now bigger), rest of the open list. Launch path unchanged: go-live switches + S244 + F12(b) + F52.
4. 2c-2c polish + PR 2c "slice C" (config-helper prompts editable) queued behind 2c-2b.

## 13. Lessons

- **An operator challenge to a stated ranking ("is the pricing maths actually IP?") reversed the whole slice-2 design for the better.** State rankings as challengeable, not as findings.
- Read-only discovery corrected the brief's premise TWICE today (resolve-not-server-side; no verifier gate in code). The Step-0/discovery gate keeps paying.
- A "rejected" publish that was actually the note-required validation firing first is not the gate under test — drive past the first rejection to reach the one you mean to prove.
- Editor per-file tallies mislead; only git numbers are evidence.
- Preview + prod share one DB: any write surface on a preview is a prod write surface.
