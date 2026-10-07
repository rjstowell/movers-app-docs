# Movers App: Session Summary 2026-09-18 (covers 2026-09-17 evening to 2026-09-18 evening)

**Headline:** the operator got eyes, and the first thing they saw was a silent outage. F31 phase 1 (operator console at `/admin`) shipped read-only behind a fail-closed gate, then grew the assembled-prompt view (S107). Its first screen exposed the 2026-09-15 OpenAI out-of-credit outage: eight hours, no alarm, every affected run stranded, and 13% of mailbox mail being classified on the subject line alone. Two outage-hardening PRs fixed that (S317, S318). The day closed with the plumbing for operator-editable prompts (F31 PR 2a), proven byte-identical with nothing overridden.

**main:** `a1b29b0` -> `bb952de` [PR #126, F31 PR 1] -> `707bfd2` [PR #127, F31 PR 1b] -> `b2e7271` [PR #128, S317] -> `6f3381e` [PR #129, S318] -> `8603f65` [PR #130, F31 PR 2a]. Five PRs, two migrations by hand, suite 407 green with zero known failures.

**Numbering after this session:** next S = **S332**, next F = **F56**, next DEC = **DEC58**. DEC48 locked, DEC23 amended, DEC56 and DEC57 locked.

---

## 1. Launch-readiness review (start of session, legal excluded)

No big feature is left on the critical path. What remains is one go-live switch-over, three real builds and one small batch, roughly 5 to 6 sessions.

- **Go-live switches (operator action + one verify session):** S253 (LTD OpenAI account + hard cap + key migration; prod key sits on a personal account and ran dry 09-15), `TENANT_DAILY_TOKEN_CAP` (unset in prod, so S252 is inert), S264 (Stripe test to live + live webhook + reset `BILLING_CUTOFF`), S265 (auth SMTP, then test signup confirm / reset / invite on `my`), S243 (Postmark Platform plan), `ACTIVE_TENANT_COOKIE_SECRET`.
- **Builds, ranked:** S244 (bounce / complaint webhook + suppression; shared sending reputation), F12(b) (real-email inbound test), F52 (reconcile provisioning), then S266 + S254 Half B as one small batch.
- **Off the critical path, operator's call:** F31 (chosen and started this session), F21 slice 3, config reachability (S163 / S134 / S175 + F10 2e).
- **Not tracked anywhere before this session, now logged:** backups + restore drill (S330), cron-death alerting (S320), cancelled-tenant data deletion (S331).
- **Vision note:** F6 (Manager Agent) stays post-launch; launch ships the config-heavy app. Conscious gap.

## 2. F31 PR 1: operator console (PR #126 `bb952de`)

- **Discovery first** (12 questions, read-only). It changed the design in several places; the full list is in the archive append. The ones that matter going forward: every per-tenant computation the console needs already accepts an arbitrary tenant id; `support_reports` had no reader; LLM failures and cron ticks were not recorded anywhere; Quote Engine prompts still ship to every browser.
- **Gate:** `OPERATOR_USER_IDS` (comma-separated Supabase user ids). Unset = nobody. Non-operators get 404. Logged-out users go to `/login`. Enforced in the layout AND as the first line of every data function. One middleware line (`/admin` added to `ONBOARDING_EXEMPT_PATHS`). No `/api/admin` routes because middleware passes `/api/` through ungated.
- **Views:** Tenants (owner, tier, billing state, seats, onboarding, agent, inbound mode, sending identity, last inbound), tenant drill-in, Trouble (60-minute cross-tenant error burst, tenants with problems, unresolved error runs, failed sends, open support reports, mailbox sync errors), Spend (red "CAP UNSET" line, tokens today / 7 days per tenant).
- **Rules:** zero writes, no customer content in v1.
- **Login:** there is no separate operator account type. One email = one login = one user id for ever; tenant membership is separate. Operator access currently attaches to `richjstowell@gmail.com`. Dedicated `admin@` login + 2FA = S326.
- **Verified** on preview and prod: fail-closed 404 before the env var existed, 7 tenants, non-operator 404 on all four URLs, normal app untouched.

## 3. F31 PR 1b: assembled prompt view, delivers S107 (PR #127 `707bfd2`)

- Decision: one shared loader used by the pipeline and the console, shipped alone in a small PR so the pipeline-touching diff could be read line by line. A console-only copy would drift and the view would lie.
- Verified: identical test-panel output on preview and prod; prompt matched the live DB config; a live description edit showed up immediately; one real email drafted normally after merge.
- Reading a real assembled prompt answered parts of S315 for free: descriptions are substituted verbatim as match conditions; sensitivity keywords are a case-insensitive substring match; a NULL description falls back to the bare label.

## 4. S317: mailbox HTML fallback (PR #128 `b2e7271`)

- 79 of 612 mailbox emails (13%) had no text part and were classified blind. None was ever drafted; errored ones never retried. Checked what happened to them: 48 ignored were all real junk, 27 escalated were mostly customer replies that a human saw. Nothing lost, but a real gap.
- Fix at ingest, shared with the Resend path: server-side `htmlToPlainText` (cheerio), never throws, falls back to the old tag-strip. The old lightweight helper stays for the browser-side draft editor.
- Early check on real traffic passed (a phone reply arrived with text, and the quoted thread was cut from 2,780 to 761 characters). **Still to do: re-check after overnight DMARC reports ("check PR A").**

## 5. S318: outage hardening (PR #129 `6f3381e`)

- Policy is DEC57. In short: honest cause classification, quota / auth as `our_end`, one alarm email per cause per hour, retries that back off over 24 hours when the provider rejected the call (and for daily-token-cap errors, which now heal after the midnight reset), loud stranding, a candidate query that cannot be starved.
- Review catches before merge are listed in the archive append. The one that mattered most: the detector would have missed the exact failure this PR exists for, because the real 09-15 message ("429 You have no credits remaining") is not the classic quota shape. It is now a test case, and future alarms carry the provider's status, code and type.
- Proven on production: first cron tick stranded the six old S199 runs, wrote six reports, sent exactly one email with five suppressed.

## 6. F31 PR 2a: prompt override plumbing (PR #130 `8603f65`)

- Operator decisions that shaped it: the spine must be editable (rare, after a diagnosis, never casually); a test facility in the console is a must before any spine change goes live; extraction and the Quote Engine prompts must be editable too; full history with revert weeks later. Per-tenant forks were proposed and replaced by additive tenant notes (DEC56).
- Shipped with no UI and no write path. With empty tables every prompt is byte-identical to before, proven by snapshots captured before the refactor.
- The planned SQL-only live probe was dropped: it needs the byte-exact spine text and a correct fingerprint, which only the repo has. PR 2b's first verification step does the same proof through the real UI.

## 7. Operator actions this session

- Added `OPERATOR_USER_IDS` to Vercel Production + Preview.
- Two migrations applied by hand via Supabase MCP `execute_sql` and verified by SELECT (columns, tables, RLS, grants, functions, smoke test of the claim function with its test row deleted afterwards).
- "Piggies" description probe on Testing Mover: added, seen in the prompt, removed.
- Test emails sent through Testing Mover (auto-send on, generic consent): all drafted and sent normally.

## 8. PR 2b BRIEF INPUTS (everything a fresh chat needs to write the brief)

**State to build on.** `main` = `8603f65`. Tables `prompt_versions` and `prompt_publications` are live and EMPTY (columns and rules in the archive append). Registry `lib/prompts/registry.ts`: six keys with `getCodeDefault`, `getRequiredTokens`, `codeDefaultHash`. Resolver `lib/prompts/resolvePrompt.ts`: `resolvePrompt(key, { candidates })` returns `{ text, source: "candidate" | "override" | "code", versionId }`, never throws, never caches. `computeDraftForInput` accepts `input.promptCandidates` and threads it to classifier, extraction and greeting. `assembleClassifierSystemPrompt(input, { candidates })` is shared by pipeline and console. F12(a) test panel = `lib/agents/testEmail.ts` (runs `computeDraft` with no persistence). Console lives in `app/admin/` (layout, `ui.tsx`, pages) with data in `lib/admin/` (`data.ts`, `classifierPrompts.ts`, `requireOperator.ts`, `operatorGate.ts`, `CopyButton`).

**Goal.** A Prompts tab in the operator console. This is the FIRST WRITE SURFACE behind the gate.

**Locked workflow (DEC23 amendment).**
1. List page: every registry key with label, where it is used, what is live (code or override), live version, last published, and a "code default changed since your edit" badge when the live version's `based_on_code_hash` differs from the current `codeDefaultHash`.
2. Detail page per key: code default (read-only), live text, editor.
3. Save as draft = new `prompt_versions` row (`based_on_code_hash` = current code hash, note, `created_by`). A draft is never live.
4. Test = A/B in the console: pick a tenant, paste an enquiry, run the pipeline twice with no persistence (live prompts vs the candidate for this key) and show side by side: category, confidence, would escalate, missing info, template, draft preview.
5. Publish = new `prompt_publications` row pointing at the version. Global scope only (`tenant_id` null) in this PR. Note required.
6. History = every version and every publication, newest first, who / when / note.
7. Revert = publish an older version, or publish a null version (code default). Always a new row. Works weeks later.

**Validation (on save, on publish, and on a candidate before a test run).** Non-empty after trim; a sane length cap; every required token present; clear rejection message otherwise.

**Two resolver gaps to close in this PR.** (a) Treat an empty or whitespace-only override body as invalid and fall back to the code default. (b) Candidates currently skip the required-token check; validate them before running.

**Spine rule.** A `classifier_spine` version must have at least one test run before it can be published, enforced server-side, and the publish confirmation must say it affects every tenant. Recommended mechanism (decide at Step 0): a small append-only `prompt_test_runs` table (version id, tenant id, who, when, a summary with NO customer content). It doubles as an audit trail. Additive migration, applied by hand before merge.

**Test harness rules.** Operator-only. Never persists runs, queue rows or emails. Reuse `testEmail.ts` internals, do not fork pipeline logic. Runs against the chosen tenant's real config with the service role. **Token usage from operator tests must NOT count against that tenant's daily cap** (Step 0: how `guardLLM` / `enforceTenantTokenCap` take the tenant id, and the cleanest bypass). Not subject to the tenant panel's 30-per-hour limit. v1 = one pasted enquiry. Replaying a candidate against a tenant's last N real enquiries is the stronger tool for spine changes but puts customer content in the console: later item.

**Write-surface rules.** Server actions only (no `/api/admin`, middleware passes `/api/` through ungated). `requireOperator()` is the first line of every action. Writes touch ONLY the prompt tables (plus the test-run table if adopted). `created_by` / `published_by` = the operator's auth user id. Self-check by grep before reporting done.

**IP rule (DEC1 / DEC12).** Prompt text must never reach a tenant-facing route or bundle. The editor page necessarily sends prompt text to the operator's browser; that page must be gated, force-dynamic and noindex like the rest of `/admin`.

**Out of scope for 2b.** Quote Engine prompts and the composite setup-helper prompts (PR 2c). Tenant note slot (PR 2d). Operator edits of tenant descriptions (PR 3). Diff view (nice to have).

**Verification plan (operator).** Non-operator gets 404 on the Prompts pages and the actions reject. Save a draft of `greeting_system`, see it in history, confirm nothing changed live. **The live probe:** draft a `classifier_spine` version = code default plus one harmless marker line; run the A/B test; publish; the Testing Mover assembled-prompt view shows the marker; one real test email records a non-null `classifier_spine_override_version_id`; publish a revert to the code default; marker gone; next real email records null. History shows both publications. Try to save a spine with a placeholder deleted: rejected. Confirm an operator test run did not move the tenant's token counter.

**Housekeeping to mention in the brief.** Migration filename: continue the repo's `202612xx` sequence (next `20261223090000`), see quirks doc.

### Queued after 2b

- **PR 2c.** Own discovery first: how the quote page calls the LLM, whether API routes accept prompt text from the client (today `DEFAULT_PROMPTS` ships in the client bundle with `SET_PROMPT` reducer plumbing: a live DEC1 / DEC12 leak). Move prompts server-side, add registry keys, remove them from the bundle, then wire the composite setup-helper prompts (`CONFIG_HELPER_SYSTEM_PROMPT` + scope rule + formatting rules + add-mode builder). Decide what "test" means for quote prompts.
- **PR 2d.** Tenant note slot (DEC56): new spine placeholder, spine version bump + tests, empty note = section omitted so prompts stay byte-identical for tenants without notes; resolver learns tenant scope; same workflow on the tenant drill-in.
- **PR 3.** Operator edits a tenant's enquiry-type descriptions. Today `updateCategory` binds the session tenant, guards role, requires a non-empty description, has no length cap, and can never edit system categories. Build a tenant-id-parameterised path through the SAME logic, plus a change-log table (first slice of DEC46: tenant, entity, field, before, after, actor, actor kind operator / tenant, reason, time). Reason required. Show the assembled prompt before and after. Warn that "Not X" exclusion clauses are load-bearing. Support script: "Do you want me to go in and correct this? It will change the description in your config."

## 9. Open threads / next

- **"Check PR A":** re-query the live DB after overnight DMARC reports; expect zero new mailbox runs with HTML but no text.
- **Next build:** F31 PR 2b, from section 8. Then 2c, 2d, PR 3, or return to the launch critical path (S244, F12(b), F52, small batch, go-live switches).
- **Held small fixes:** S316 (crew Notifications card), S327 (old S199 error runs).
- **Watch items, no action yet:** auto-replay processes up to 50 runs sequentially per tick (pre-existing; rotation makes a timeout harmless but watch function duration after a big outage); the billing-wording fallback could misread a free-tier rate-limit message that mentions billing (result would be one throttled alarm and an `our_end` card, mild); a real quota or auth failure has still never exercised the new alarm path in production (unit-tested against the real SDK error type and the real 09-15 message).

## 10. Lessons

- **The stop rule earned its cost four times:** company-name fallback had three levels, the HTML helper was weaker than the brief assumed, `error_resolved_at` is dormant, and the spine has four placeholders. Every one was a wrong "fact" in a brief, caught before code.
- **Test against the failure you actually recorded, not the textbook one.** The preview test could not run, so the real 09-15 message became the test case, and it exposed a detector that would have missed it.
- **"No row yet" inside two minutes is not a failure.** Inbound lag was 95 seconds on the provider's side; theorising started too early.
- **Moving a filter from SQL into code changes what LIMIT means.** That is how starvation appears.
- **Anything compared against a cron period needs tolerance.** Ticks arrive early as often as late.
- **The console paid for itself on its first page load.**
- **Check the migrations folder before naming a migration.** The repo's filenames run ahead of the calendar as a sequence; a "wrong date" correction this session broke the sequence (harmlessly).
