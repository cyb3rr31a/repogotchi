---
inclusion: manual
---

# Demo prep (run as the `/demo-prep` slash command)

Use this when preparing the hackathon video or a live demo.

## Set up

1. Run `node scripts/make-demo-repo.mjs ../haunted-demo`. It creates a local-only repo with:
   - a small TypeScript project with ~25 TODO/FIXME markers,
   - a `coverage/lcov.info` at 18% line coverage,
   - 6 unmerged branches backdated 40–200 days,
   - 10 days of lazy commit history ("wip", "fix", one 900-line commit),
   - `DEMO_SECRET.txt` (untracked), holding a randomly generated, AWS-shaped fake key to paste on camera,
   - `src/broken.ts`, which produces several type errors once opened.
2. Open `../haunted-demo` in a second Kiro window with the extension running from the Extension Development Host. Check that the Repogotchi view sits at the bottom of the Explorer sidebar, so the reaper enters from the edge facing the editor.
3. Set `repogotchi.spookiness` to `full` and `repogotchi.reaperInEditor` to `slipping`.

## Shot list

1. **The reaper reveal.** Open `src/broken.ts`. After about 10 s the fever climbs. The pet slips to Sick, and the reaper leans in from the panel edge. The pet glances at it, looks back at the camera, sweats and goes blue. If timing is unreliable, use **Repogotchi: Summon Reaper (Demo)**.
2. **Poison.** Paste the fake key from `DEMO_SECRET.txt` into `src/config.ts`, then `git add` it. Within ~5 s the pet is poisoned, the reaper looms with its hourglass, and the warning appears. Click "Show me", delete the line, and unstage it. The antidote plays, followed by the rotate-your-key reminder.
3. **Death.** Run **Repogotchi: Time Travel (Demo)** +72h. The hourglass empties, then you see the death, the ghost, and the graveyard epitaph.
4. **Redemption.** Walk deletes the stale branches (show the modal confirm). Clean jumps to TODOs. Run `scripts/demo-heal.sh ../haunted-demo`, then Feed. Health climbs. **Resurrect**: generation 2.
5. **Flourishing.** Time Travel through a week of tested commits (or prepare a second demo repo with a 7-day tested streak). Show the ghost-flowers, the Lantern Glow trophy and the Fitness page.

## Before recording

- The extension runs from a packaged `.vsix` installed in a clean Kiro profile, not only from the dev host. This proves the install works.
- **Never push the demo repo anywhere.** The key is fake, but it's secret-shaped, and secret scanners and push protection will (rightly) flag it.
- The Kiro spec panel and hooks panel are visible in at least one shot, to show how Kiro was used.
