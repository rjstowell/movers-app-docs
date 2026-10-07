# Movers App session summary, 2026-09-30

**Audit cleanup + the upload bug + the cleanup that never ran + S66 closed.** Eleven ships, every one verified on preview or prod by the operator. main `1b0b7d1` -> `3279098`. Tests 1469 -> 1516. One migration by hand via MCP.

## 1. Shipped, in order

| Item | How | Where | What |
|---|---|---|---|
| S400, S401, S404, S405, S407, S408 | direct, 7 commits | `1b0b7d1`->`fd8ce37` | Batch A from the 09-29 v2 audit: stale copy, dash/label sweep, crew quote links hidden, system calendars unrenameable, invite Resend/Revoke + login-deletion sentence, trial copy by eligibility on Billing AND onboarding checkout, "Trial ends {date}". |
| S413 | direct | `4d9d215` | Retention: "Never delete" dropped, GDPR helper line, null -> 60 months. Part 2 found nothing had ever purged an archived card. |
| S414 | PR #152 | `fe7f052` | iPhone photo upload 500. Root cause: Next 1 MB Server Action body limit (Vercel caps at 4.5 MB anyway). Job card attachments now go browser -> Storage via signed URL. |
| S415 | PR #153 | `5125e9d` | Archived-card cleanup as an app cron (03:17 UTC, S320 heartbeat, operator alert on partial failure). Dead edge functions and SQL function removed. Proven end to end on J-0012. |
| S66 | PR #154 | `db76331` | Two-month-old save-then-revert. Real cause: React 19 auto `form.reset()` after `<form action>`. Shared `useActionSubmit` helper on six forms. Member row now closes with "Saved.". |
| S406 | PR #155 | `d790419` | Plan prices from Stripe price objects, constants as fallback with a warn log, Billing + onboarding. Ready for S264. |
| S402 | PR #156 | `803b1f3` | Billing page + checkout/portal actions gated on `canManageBilling` (the tab's own check, exported). |
| S417 | direct | `8965423` | Member row reopen-on-saved-role fix. The reported "stuck Saving..." was Vercel skew protection serving an old build. |
| S412 | direct | `3279098` | `docs/investigations` fully untracked (21 files, not 3), ignore rule kept. |

## 2. Findings worth remembering

- **Nothing had ever deleted an archived card.** Two half-built cleanups (edge function never deployed; SQL function waiting on pg_cron that is not installed), neither heartbeated. The retention setting was a promise with no purge behind it. Fixed by S415. Only one card was old enough to matter (July, 24 months), so no data was overdue.
- **S66 was never a props/cache/race bug.** React 19 resets a `<form action>` after the action, and a `<select>` only reads `defaultValue` at mount. Reproduced in jsdom. The second-save hazard (next Save writes the reverted value) made it a data bug, not cosmetic.
- **The photo upload had nothing to do with compression.** Server Actions cannot carry files over 1 MB; the "20 MB document limit" label was unreachable by design. Direct-to-Storage is the only shape that makes the label true.
- **Vercel skew protection cost one false bug report.** An open tab kept hitting the 12:28 build for three hours. Fresh tab before every prod check.
- **Stripe prices are immutable**, so the fallback path can only be unit-tested.
- **Preview + prod share one DB**, so hitting a destructive cron on a preview runs it across all tenants. Confirm candidates with a read-only query first.

## 3. Verification notes (operator, prod unless stated)

- Batch A: checks 1, 3, 4, 5, 6 pass; 2 and the mailbox header trusted; 7 and 8 (trial states) deferred to the manager's second-company test because S199 has a live sub and Testing Mover predates the cutoff.
- S414 on preview: owner iPhone photo 1.9 MB (would have failed before), 5-20 MB PDF, opened on mobile + desktop, no 413.
- S415: retention 6 on Testing Mover, J-0012 `archived_at` moved to 30 Jan via MCP, curl on preview through `vercel curl`: deleted 1, then 0; row + attachment + object gone; J-0001 kept; heartbeat ok; /admin/alerts Crons card lists it; prod route 401s without the secret; cron registered on the prod deployment. Retention reset to 24.
- S66 on preview: Jobs retention, VAT scheme, Team role (closes with Saved.), auto-send toggle, General. Mailbox skipped (none connected).
- S406 on preview: Billing shows the three test prices; no `[Billing] plan price fallback` line in logs.
- S402 on preview: Charlie as admin without the permission sees the read-only card by URL; with it, plan cards back; owner unchanged; Charlie back to crew.
- S417 on prod, fresh tab: crew -> admin, tick Billing, admin -> crew, all close with Saved.; reload shows crew.

Retention "Saved." snap-back seen during S415 setup was S66, confirmed by DB read (6 saved). Not a new bug.

## 4. Doc changes this pass

- Active backlog: new 2026-09-30 header; numbering (next S418 / F67 / DEC61); S66 body removed; S32 amended (cluster S32 + S80 + S100, General confirmed, S66 done separately); S400/401/402/404/405/406/407/408/412 bodies removed from the 09-29 v2 block; new session block with F66 (archived-card export, post-launch) and amendments to S211, S264, S375, S76, S331, S130, plus S414 gap and housekeeping notes.
- Archive append 2026-09-30: outgoing 09-29 v2 header + numbering, all done bodies including S413/S414/S415/S417 (raised and closed in-session).
- Quirks: section 19 (uploads, React 19 form reset, skew protection, `vercel curl`, previews share DB, Stripe prices immutable, cron registry daily, retention null mapping); docs/investigations line updated.
- Launch plan: baseline 2026-09-30, papercuts row minus S66, estimate note.

## 5. Next session: nav-guard cluster S32 + S80 + S100

Not a new item (S416 was not created). Three existing entries, one piece of work: S32 app-wide guard audit, S80 Fleet inline editing has no guard (+ sticky-save-bar rollout question), S100 company switcher bypasses the guard. Scope per the S80 note: **every surface holding unsaved user input**, not every form. Confirmed holes so far: Settings > General, Settings > Fleet, company switcher.

Investigation first, read-only, then a fix brief. Paste to Claude Code in a new chat:

```
Investigation, no code changes. Work ONLY in movers-app-2.
Nav-guard cluster S32 + S80 + S100: unsaved edits are lost on navigation without a warning on some surfaces; others warn.
Report:
1. The guard itself: which hook/component implements the unsaved-changes warning, where it lives, how a surface registers dirty state with it, which navigation paths it intercepts (in-app links, tabs, folders, router.push, browser back, tab close via beforeunload, company switcher, logout).
2. Every surface holding unsaved user input, app-wide (settings forms, Agents tabs, templates, categories, rules, Review Queue draft editor, job card drawer, quote engine, vehicles, calendar edit pop-up, onboarding steps, support ticket compose, profile/password). Table: surface, file, has a dirty-state notion (yes/no), registered with the guard (yes/no), which navigation paths still lose the edit.
3. S100 specifically: what the company switcher does on select (router.push / server action / hard reload) and why the guard misses it.
4. Which surfaces save instantly (no Save button) and therefore need no guard; list them so they are not counted as holes.
5. One recommended general shape: a single useUnsavedChanges(dirty) hook every surface calls, covering in-app navigation + beforeunload + the switcher, and whether the F8 sticky save bar should roll out with it (S80) or stay separate. State trade-offs in 3 lines. Do not implement.
```

Also open from the 09-29 v2 audit: S399 (quote routes editable after onboarding, Tier 2), S409 (brand contrast, Tier 2), S403, S410, S411. `deno.json` orphan in `supabase/functions/` for the next tidy.
