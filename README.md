# Movers App working docs

Source of record for the Movers App planning docs. Edited at the end of each build session by targeted line edits, committed as `docs: session <date>`.

| File | Role |
|---|---|
| `movers_app_master_backlog.md` | ACTIVE backlog: current header, numbering, open items, live DEC/quirk/lesson blocks |
| `movers_app_backlog_archive.md` | Append-only archive: past headers, past numbering blocks, DONE item bodies |
| `movers_app_launch_plan.md` | Path to launch, tiers and estimate |
| `movers_app_system_quirks.md` | Quirks and gotchas log |
| `sessions/` | One summary per session, dated, additive |

Rules: edit the real file, never regenerate it; archive is write-only; new items get the next free number from the backlog's numbering line, never inferred from chat.

## Doc pass (end of session)

1. Start of session: attach this repo (`add_repo rjstowell/movers-app-docs`, push access), clone it, read the backlog's numbering line from disk. Never read the backlog through the project Context reader: it caps a doc at 262 KB and truncates without warning.
2. Edit the active backlog, launch plan and quirks with exact line operations on the files on disk. Never regenerate a file. Never retype a doc from a tool result or from memory; if a file is not on disk, get it from the repo or an attached original, and diff before writing back.
3. Append this session's done items, the outgoing header and the outgoing numbering block to `movers_app_backlog_archive.md`. Append only; never read it to check context.
4. Write the session summary to `sessions/movers_app_session_summary_<date>.md`.
5. `git diff --stat`, commit `docs: session <date>`, push to `main`, report the hash. The diff is the record.
6. Project Context holds copies for search only. Refresh the backlog and launch plan there every few sessions; nothing depends on them being current.

If GitHub is unreachable at session end, write a delta file listing the exact edits and apply it next session. Never retype a doc to get around it.

History: docs lived in the project Context until 2026-10-07 (one `movers_app_backlog_archive_APPEND_<date>.md` per session because Context cannot edit in place). The 24 appends (2026-09-10 to 2026-10-06) were consolidated into `movers_app_backlog_archive.md` and the 45 Markdown summaries (2026-08-27 on) copied into `sessions/` on 2026-10-07, each file copied twice by independent agents and compared byte for byte. Older summaries (July to 2026-08-26) are .docx in the Context and were not migrated.
