# Movers App, Session Summary, 2026-10-01

**THE NAV-GUARD SESSION + STICKY SAVE BAR SLICE 1 + ALIAS MATCHING.** Five ships: four merged PRs (#157, #158, #159, #160) and one direct-to-main (S424). main `3279098` -> `790c2e9`. Tests 1530 -> 1626 passing (one pre-existing failure, see S420). No migrations. S32, S100, S424, S426 DONE. S80 slice 1 of 4 DONE. New open: S420, S421, S422, S423, S425, S427, F67, F68.

---

## 1. State at session start

Launch build list cleared (09-23). Go-live switches operator-only. Legal (S260, S261) the long pole, operator's to do. Sid signed up four days ago, has not touched the app since, has not invited the operator as owner; DNS/inbox link waits on him. Postmark Platform plan already done (S243 closes). `ACTIVE_TENANT_COOKIE_SECRET` presence unverified (check Vercel env vars). S326 (dedicated admin@ login + 2FA for the operator console) still open.

Session pick: the nav-guard cluster S32 + S80 + S100, pinned twice already as next.

## 2. Ships

### PR #157, `d3dd619`: S32 slice 1 + S100
Investigation first (`docs/investigations/s32-s80-s100-nav-guard-investigation-2026-10-01.md`): one Agents-only guard with seven hand-wired flags; nothing outside `/agents` guarded; the company switcher POSTs `/api/tenant/switch` then `window.location.reload()`, so the link-click interceptor never saw it and the cookie had already moved when the browser asked.

Built: `lib/forms/UnsavedChangesProvider.tsx` (one provider in AppShell holding a set of dirty keys, one anchor-click interceptor, popstate re-arm, beforeunload, one `LeaveWithoutSavingModal`), `useUnsavedChanges(dirty, key?)` (always clears on unmount), `useRequestLeave()` for non-anchor navigations, `UnsavedChangesScope` per Agents tab. Switcher, company add/delete/restore, Profile sign out, Stripe redirects and the feedback widget's "go to Generate" now ask BEFORE the side effect. Fixed by construction: the VoiceSettingsPanel dirty leak (edit tone, leave Generate, whole page guard off) and the double-Back residue after a save.

Verified on preview: Agents guard, switcher confirm before cookie move (Stay = still company A, edit intact; Continue = company B), sign out with dirty display name. To test the switcher the operator invited `richjstowell@gmail.com` into S199 Movers as owner via Settings > Team; existing-account invite path works ("Existing account added to this company"), no email round trip. That login now sees two companies.

### PR #158, `2a7d8f4`: S32 slice 2a
`lib/forms/useFormDirty.ts` (FormData snapshot vs now, `markClean()` on save success, file picked = dirty). Registered: General company + business info, Fleet (asks only once values differ), Pricing + VAT, Distance bands, Deposit, Branding, Jobs retention, Vehicles reminders, Mailbox connect form, Sending identity add-domain, Describe flow pre-generate text. Four forms moved to `useActionSubmit` (Deposit, Business info, Branding, add-domain) so React 19's post-action reset does not fool the dirty check; side effect: a failed save keeps what was typed.

Two bugs found and fixed in the same PR:
- Overview delay slider was re-saving the whole form on blur, which silently stored a dragged-but-unsaved threshold. Delay is now Save-only; toggle stays instant; both sliders commit on Save. Operator: "more logical, the only thing that saves on its own is a toggle."
- `useFormDirty` set state inside the native `input` listener; in a real browser React re-rendered before its own `onChange` ran and wrote the old value back, so Address/Host/Port/Username in the mailbox card looked frozen (Password worked because it is excluded from the comparison). jsdom fires listeners synchronously so tests passed. Fix: defer the check a task. Quirks entry added.

### PR #159, `94c8bcf`: S32 slice 2b, closes S32
`useGuardedClose` for drawers/modals: backdrop and X ask when that overlay is dirty, explicit Cancel closes without asking, save marks clean then closes. Registered: job card drawer + comment composer, Quote Engine (existing `isDirty`; "Load another quote" now uses the shared modal), Vehicles add/edit modal + defect/damage logs + insurance date, Calendar event/calendar modals, Support ticket + reply, agent onboarding wizard (in-memory steps, branding colours included, cleared on Finish). Skipped and noted: board add/rename one-liners, operator console editors, account onboarding, profile complete, password form. 32 tests, keystroke-simulated.

### S424, `7dcb648`, direct to main
"Three-seater sofa" came back not recognised although `three seater` / `3 seater` are aliases. Root cause was NOT the hyphen: `parseQty` read a leading number word as a quantity (3 x "seater sofa"), and "three" was never equated with "3". Fix: `normKey()` (case, punctuation, number words one to ten -> digits) applied to alias keys and catalogue names at index build and to the line at match time; `matchLine()` runs the original matcher first, then normalised lookup, then plural-tolerant, then word overlap; `parseInput` keeps the number word in the item name when it precedes "seater"/"drawer" and the line did not match. Before/after sweep of 1,275 lines (every alias and catalogue name x5 variants): no previous match changed; 12 lines newly resolve, including the "N seater" aliases themselves (broken before), "tvs", "kids bikes", "childrens bikes"; "Mirrors", "Stools", "Desks" now resolve straight to the base item instead of asking. Verified on prod: "Three-seater sofa x1" + "2 x 3-seater sofa" -> 3 x Sofa - 3 seat, no AI button.

### PR #160, `790c2e9`: S80 slice 1 + S426
Investigation first (`docs/investigations/s80-sticky-save-bar-investigation-2026-10-01.md`). Key finding: the guard only records THAT something is dirty; no form had an `id`; six surfaces have no `<form>`; Mailbox needs a specific submitter. So a registration hook carrying save/discard/label was required.

Built: `SaveBarProvider` inside `UnsavedChangesProvider`, `useSaveBar({ label, save, discard, kind, primaryLabel?, beforeSave? })` which also registers with the guard, `StickySaveBar` mounted once in AppShell. Desktop: a row under `<main>`, covers nothing. Mobile: fixed above the bottom pill (height read from `--mobile-dock-height`, 56px measured), publishes `--save-bar-height`; setup banner, calendar FAB and Agents toasts stack above it. `<main>` bottom padding fixed (was 12px short, content hid under the pill on every mobile page). Overview: local mobile bar deleted; three registrations ("Replies & auto-send", "Email signature", "Ignored domains") so "+N more" is truthful; rail-section switches no longer discard (sections are hidden, not unmounted, so edits survive and the bar names them; leaving the tab still asks). Bar shows "Saved." for 1.5 s after a successful save, then hides or moves to the next dirty form. Guard-only dirty forms (Mailbox, add-domain) are counted in "+N more" and the link opens their section, scrolls and focuses; the bar then shows a per-form hint ("Verify or clear it in the form above", "Add or clear it in the form above") with no Save/Discard. S426: TemplateEditorDrawer z-30 -> z-50.

Three preview rounds with the operator; all checks pass on desktop and mobile widths.

## 3. Decisions locked (operational, not DEC-numbered)
- **Guard rule:** guard only where edits would be lost. Hidden-not-unmounted sections (Overview rail) are not guarded; the bar names what is unsaved.
- **Modal/drawer rule:** backdrop, X, Escape ask when dirty; an explicit Cancel/Discard button is an explicit discard, no prompt; save marks clean then closes.
- **Autosave rule:** toggles, selects and other discrete controls may commit instantly; sliders and free text commit on Save. Mixed rules inside one card are wrong. (S421 is the post-launch audit.)
- **Sticky bar shape:** per-form, most recently edited, "+N more" cycles through bar forms then guard-only forms; inline Save buttons STAY (bar mirrors, does not replace: keeps Enter-to-submit, tests, drawer footers); drawers and modals keep their own Save and hide the page bar while open; send surfaces (Review Queue, Escalated, Support, comments), Quote Engine and the onboarding wizard stay out of the bar.

## 4. New items (bodies in the backlog)
S420 (F57 PDF snapshot test fails on main, pre-existing), S421 (autosave for discrete controls audit, post-launch, after S80), S422 (Describe flow unreachable after onboarding + refresh, post-launch), S423 (Quote Engine address-only edit after save: no autosave, no guard, pre-existing), S425 (Quote Engine Items header line counter wrong/unclear), S427 (sticky footers inside drawers/modals), F67 (pricing in drafts: quote engine -> LLM reply hookup; marketing already claims it; design chat first), F68 (catalogue + list-to-cubic conversion semi-overhaul; feeds F67).

## 5. RESUME S80 (slices 2 to 4), for the next build session
Base: main `790c2e9` or later. Report: `docs/investigations/s80-sticky-save-bar-investigation-2026-10-01.md` section 2 (per-form table with save-trigger method, discard notes and gotchas) and section 5 (shape). Hook: `useSaveBar` in `lib/forms/`. One PR per slice, branch + PR each.
- **Slice 2, Settings forms with a form ref:** Business info, Deposit, Retention (controlled select first, else discard resets to mount value), Branding (discard via state: colours, revoke blob URL, clear file input), Pricing (VAT pill/scheme are useState, reset them; `beforeSave` opens the owning section + `reportValidity()`, else a hidden invalid field makes `requestSubmit()` fail silently), General (save can stop at the depot confirm; bar scrolls to it).
- **Slice 3, no `<form>`, lift the handlers:** Profile name (add ref or lift), Fleet and Distance bands (edit row + add row register as TWO keys), Reminder offsets stays guard-only.
- **Slice 4, Agents:** Voice, Templates (overlay kind, already z-50), Categories (needs a save-all-dirty-rows), Rules (survey distance confirm + keyword rows), Mailbox (`primaryLabel` "Verify" then "Save", discard needs a saved snapshot added).
- Then S80 closes. S427 (sticky drawer footers) and S421 (autosave audit) follow separately.

## 6. Operator notes
- Sid: waiting game; no action from him since signup. DNS link waits on his owner invite.
- Legal S260 + S261: operator's, ASAP.
- Go-live switches still open: S264 Stripe live (unblocked by S406), S381 Radar, S265 auth SMTP, `ACTIVE_TENANT_COOKIE_SECRET` (verify in Vercel env), S326 admin@ login.
- `richjstowell@gmail.com` is now owner of Testing Mover AND S199 Movers (for switcher testing). The Team invite Role select visually reset to "Crew" after submit; stored role was owner. Cosmetic, S66 class, no item.
- An unnamed GitHub check with no result showed on PRs #159 and #160; merges went on the Vercel status. Worth one `gh pr checks` look when convenient. No item.
- Next session options: run the two briefs waiting in other chats, then S80 slice 2; or F67 design chat.
