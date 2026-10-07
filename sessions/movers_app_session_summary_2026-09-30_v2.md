# Movers App session summary, 2026-09-30 v2

**The product video session.** No app code. First marketing creative produced: a 22.5s, 9:16 product video built from real app screenshots with Claude Code (Opus 5.5) and Remotion, music bed added. Testing Mover seeded for demo data. Two app findings (S418, S419). Decision: marketing moves to its own Claude project from here.

## 1. What was produced

| Thing | Where | State |
|---|---|---|
| 9:16 master, silent | `dev/movers-app-video-claude/out/moversapp-master-9x16.mp4` | Done, 675 frames, 22.5s, H.264 1080x1920 |
| 9:16 master, music | `dev/movers-app-video-claude/out/moversapp-master-9x16-music.mp4` | Done, second Remotion composition, same scenes |
| Contact sheet | `dev/movers-app-video-claude/out/contact-sheet.png` | One frame per second, review artefact |
| Remotion project | `dev/movers-app-video-claude` (three feature commits + music commit, local git identity, not pushed) | Config-driven: copy, colours, asset paths, durations in `src/config.ts`. Remotion pinned 4.0.530 (4.0.531 ships a broken file). 16:9 stub registered, crops not tuned. |
| Codex attempt | `dev/movers-app-marketing` | Abandoned mid-polish (usage cap). Terra pass weak, Astra pass decent but unfinished. Keep or delete; the Claude Code build is the one in use. |
| Prompt template + filled prompt | `moversapp_product_video_prompt.md` (marketing project) | Reusable for the next video |

Storyboard that shipped: hook "Enquiries land while you're on a job." / "So the quoting waits for your evening." > phone reveal "Every enquiry answered and priced." > flagship close-up on £2,410, React action bar, Approve press to Sent, "A priced reply, in your voice." / "One tap to send. Anything unusual waits for you." > pipeline "Leads, quotes and jobs in one place." > calendar "Jobs, leave, van dates. One calendar." then fleet "MOT, service and insurance, flagged early." > close "Give yourself a night off. Every night." + mark + "moversapp.app · 30 days free".

## 2. Demo data seeded on Testing Mover (via MCP, all tagged `Demo seed Oct 2026`)

- **Calendar:** 26 October jobs on Main (one per working day plus Saturdays, one 2-day Nottingham to Edinburgh), 3 leave entries on Team Hours, vehicle MOT/service/insurance dates plus 7-day reminders on Vehicles as `source='manual'` so sync ignores them. Nottingham base, long-distance mix (Sheffield, Leeds, Bristol, Cambridge, Edinburgh, London, Manchester, Cardiff). Cornwall removed deliberately: not the target market and not realistic for long-distance.
- **Pipeline:** 9 cards (4 Upcoming, 3 Awaiting Deposit, 2 Active) with names matching the calendar, addresses, prices, deposits, scopes. Refs J-0013 to J-0021 via the normal trigger. The 6 placeholder cards (Tom Hilton, Downing St etc) archived into Archive > September 2026, not deleted.
- **Review queue:** one priced item cloned from a real run (Emma Hughes, 4-bed West Bridgford to Edinburgh, £2,410 inc VAT, £600 deposit, 94%). Hand-written draft. Not pipeline output. Do not approve.
- **Fleet:** Renault MOT set to 7 Oct through the app (banner cleared). Sync created real events; the manual Renault pair was deleted to avoid duplicates.
- **Avatar:** owner `avatar_url` nulled to drop the stock photo. Exposed S418.

## 3. Findings

- **Pricing in drafts is not built.** The app cannot yet put a price in a reply. Operator chose to show it anyway because the quote engine exists and the LLM hookup is imminent. Mitigation baked in: the price line is one config string, re-render if it slips. Landing page carries the same claim.
- **Initials avatar fallback is near-invisible** on the wash background. S418.
- **Mobile Review Queue action bar** may sit under the bottom nav. Unconfirmed. S419, investigate first.
- **Agent comparison:** Codex Terra weak; Codex Astra good storyboard, cut off by cap; Claude Code Opus 5.5 followed the storyboard gate, measured the assets itself (screenshots were 1320x2868, not the 1180x2560 in the brief, and it adjusted), fixed two frame issues unprompted, and delivered a contact sheet. Use Claude Code for creative builds.
- **Screenshots are mobile.** Master is therefore 9:16. Any 16:9 cut needs its crops retuned.
- **Variants** (owner-operator, multi-crew) are three config fields each, but two are new screenshots, which means reseeding. Parked. Next creative: the "app becomes your admin" angle, after quoting ships.

## 4. Process notes

- Storyboard-before-build gate paid for itself twice. Copy was corrected before render each time.
- Seeding real data beat editing images or letting the agent fake UI. Screenshots stay truthful and re-shootable.
- Codex usage cap hits mid-task; work stays on disk. Resume prompt pattern: check uncommitted edits compile, commit, continue from the numbered step.
- Marketing work does not fit the S##/F## slice rhythm. New Claude project for it, seeded with brand, product truth and a marketing log.

## 5. Next session (app side)

Unchanged from 09-30: the nav-guard cluster S32 + S80 + S100. S418 is a ten-minute add-on to any UI session. S419 needs a phone check before it gets a brief.
