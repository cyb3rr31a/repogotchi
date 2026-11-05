# Repogotchi: Kiro spec pack

A virtual pet that enforces good coding habits. It lives in the Kiro/VS Code sidebar and thrives or dies based on **test coverage** (food), **open TODOs** (mess), **stale branches** (ghosts), **Problems-panel errors** (fever) and **commit habits** (fitness). A committed secret **poisons** it instantly, positive **streaks** make it flourish, and when you start slipping the **grim reaper** peeks in from the side while the pet sweats and turns blue. This pack holds everything Kiro needs to build it. You drive the build in Kiro, so your Kiro usage is real and visible to judges.

## What's in the pack

```
.kiro/
├── steering/
│   ├── product.md        always on. What it is, the principles, what's out of scope.
│   ├── tech.md           always on. Stack, commands, testing rules.
│   ├── structure.md      always on. Folder layout and dependency direction.
│   ├── spooky-ui.md      loads when editing webview/ or src/ui/. Creatures, reaper, effects, accessibility.
│   └── demo-prep.md      manual. Run it as the /demo-prep slash command before recording.
├── specs/repogotchi/
│   ├── requirements.md   17 requirements in EARS format, numbered for traceability
│   ├── design.md         architecture, data model, formulas, state machine, reaper choreography, 14 correctness properties
│   └── tasks.md          22 tasks with checkpoints, each traced to requirement IDs
└── hooks/
    ├── core-tests-on-save.json   PostFileSave: runs core tests
    ├── test-buddy.json           PostFileSave: agent adds missing tests
    ├── todo-guard.json           PostFileSave: agent resolves or logs new TODOs
    ├── verify-after-task.json    PostTaskExec: typecheck and test after each spec task
    └── pet-checkup.json          Stop: prints Repogotchi's own vitals
```

## How to run it in Kiro

1. Create an empty folder (`repogotchi/`), `git init`, and copy the `.kiro/` folder into it. **Commit `.kiro/` and never gitignore it.** It's your proof of Kiro usage.
2. Open the folder in Kiro. Check that the steering files appear in the Kiro panel and the hooks appear under **Agent Hooks**.
3. Open `.kiro/specs/repogotchi/requirements.md` and ask Kiro to review it. Tweak anything you disagree with *before* building (weights, the death timer, creature names). Kiro refines the design and tasks from there.
4. Open `tasks.md` and click **Start task** on task 1. Work through the tasks in order. Stop at the two checkpoints (tasks 8 and 15) and look at the results yourself.
5. Tasks marked `*` (3.4, 4.2, 5.4, 6.6: property tests) are optional, but they're strong evidence of spec-driven rigour. Keep them if time allows.
6. Use **vibe mode** for visual polish in tasks 16–17 (for example "make the reaper's lean-in slower and creepier" or "give Webster a bigger sweat drop"), and note which parts you vibe-coded versus spec-coded. Judges like to see both used deliberately.

If your Kiro version shows the hooks as invalid or legacy, recreate them from the **Agent Hooks** panel using the same trigger, file pattern and prompt text from each JSON file. Kiro's hook schema changed between versions. The same goes for `fileMatchPattern` in `spooky-ui.md`: if the brace pattern isn't picked up, split it into separate patterns.

## Category pitch: Resurrection

> The virtual pet died in the late 90s. Your neglected codebase is dying right now. Repogotchi resurrects the 1996 pocket pet as an editor companion that enforces modern best practice: coverage is food, TODOs are mess, stale branches are ghosts, compiler errors are a fever, sloppy commits are a couch-potato life, and a leaked API key is poison. Slip, and the grim reaper starts peeking in. Keep up good habits, and the pet flourishes.

Resurrection is the best fit because the dead technology is unmistakable and the modern problem is real. The haunted pixel-art spider and its grim reaper also give you an edge on polish.

Visual reference: the "Repogotchi Spider Mockup" design canvas in your Claude artifacts shows the editor view, every mood and the life cycle. Keep it open while Kiro builds tasks 16–17.

## Demo video (about 3 minutes)

| Time | Shot |
|---|---|
| 0:00–0:15 | Hook: a 90s pocket pet beeping, then a hard cut to the sidebar as the grim reaper leans in. The pet glances at it, then looks straight at the camera, sweating and blue. |
| 0:15–0:35 | Install from `.vsix` in a clean Kiro window. The egg hatches on the first scan. |
| 0:35–1:05 | Slipping: open a file with errors and the fever climbs to 39 °C. The reaper peeks, then lurks. Click it for "Why is the reaper here?" |
| 1:05–1:30 | Poison: paste a fake key and stage it. The pet is poisoned, the reaper looms with its hourglass, then "Show me", delete, antidote, and the rotate-your-key reminder. |
| 1:30–1:50 | **Time Travel +72h**: the hourglass empties, then death, the ghost and the epitaph. |
| 1:50–2:20 | Redemption: Walk, Clean, Feed, health climbs, the reaper shrugs away, the pet wipes its brow. **Resurrect**, generation 2. A flash-forward shows a 7-day tested streak with ghost-flowers and the Lantern Glow trophy. |
| 2:20–2:50 | How Kiro built it: the spec panel with ticked tasks, a hook firing on save, the steering files, the pet check-up hook printing Repogotchi's own vitals. |
| 2:50–3:00 | End card: name, category, repo link. |

## Submission checklist

- [ ] Public repo with `.kiro/` committed and an open-source license
- [ ] `.vsix` attached to a GitHub release. The README install steps work in a clean Kiro profile.
- [ ] README has a "How Kiro was used" section: spec-driven tasks, hooks (what each caught), steering, vibe vs. spec, and anything Kiro got wrong that you corrected
- [ ] The video shows the app working exactly as described (functionality rule)
- [ ] Check the hackathon rules page for the deadline, video length limit and any required fields

## Things to watch

- **Naming:** "Tamagotchi" is a Bandai trademark. Keep it out of the extension name, icon and marketplace listing. In the write-up, say "90s virtual pet" or mention the toy only descriptively. All creatures in the steering are original designs. Keep them that way.
- **Check that the name is free** on Open VSX and the VS Code Marketplace before publishing. "Repogotchi" is a working name. The "-gotchi" suffix is widely used, but if you want zero trademark risk, pick something like "Hauntling" or "Repo Wisp".
- **The reaper stays inside the extension's own space.** Extensions can't draw over the editor window itself, so the reaper leans in from the edge of the pet's panel (the edge facing your code) and can also peek from the editor gutter. On camera this reads as peeking from the side of the IDE.
- **Secret detection is a nudge, not a guarantee.** Say so in the README, and recommend a dedicated scanner (for example gitleaks) or GitHub push protection for real security.
- **Cross-platform hooks:** every hook command runs through `node` or `npx`, so the hooks work on Windows, macOS and Linux.
