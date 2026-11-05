---
inclusion: always
foundationalType: structure
---

# Project structure

```
repogotchi/
├── .kiro/                     # specs, steering, hooks. Keep committed; judges look here.
├── src/
│   ├── extension.ts           # activate/deactivate, wires everything, no logic
│   ├── core/                  # PURE. No vscode import. 100% unit-testable.
│   │   ├── types.ts           # Metrics, Vitals, PetState, Stage, Mood, GraveEntry
│   │   ├── clock.ts           # Clock interface, SystemClock, OffsetClock (time travel)
│   │   ├── scoring.ts         # metrics -> vitals -> health
│   │   ├── mood.ts            # health -> mood band
│   │   ├── lifecycle.ts       # state machine: egg -> ... -> adult, dying, dead, resurrect
│   │   ├── evolution.ts       # adult form chosen from lifetime care
│   │   ├── epitaph.ts         # cause of death + epitaph text
│   │   ├── secrets.ts         # secret rules, entropy, masking (raw values never leave this file)
│   │   ├── exercise.ts        # fitness from commits, isClearMessage
│   │   ├── streaks.ts         # tested / clear / healthy streaks with grace days
│   │   ├── trophies.ts        # streak milestones -> cosmetic trophies
│   │   ├── causes.ts          # top causes by health points lost ("why is the reaper here?")
│   │   ├── reaper.ts          # reaper level 0-3 from mood, poison and 24h trend
│   │   └── names.ts           # pet name generator (seeded)
│   ├── collectors/            # read the repo. Thin I/O wrappers around pure parsers.
│   │   ├── git.ts             # execFile wrapper with timeout and typed errors
│   │   ├── branches.ts        # stale branch detection
│   │   ├── contentScan.ts     # one pass over `git ls-files`: TODO markers + secrets
│   │   ├── commits.ts         # git log --numstat parser for my commits
│   │   ├── diagnostics.ts     # pure summarizeDiagnostics (Problems panel -> counts)
│   │   └── coverage/
│   │       ├── index.ts       # discover report and dispatch to parser
│   │       ├── lcov.ts
│   │       ├── istanbulSummary.ts
│   │       └── cobertura.ts
│   ├── services/
│   │   ├── petService.ts      # orchestrates scans, owns PetState, emits change events
│   │   ├── scanScheduler.ts   # debounce, interval, git HEAD/index watcher
│   │   ├── diagnosticsWatcher.ts # fever fast path: settle timer -> applyDiagnostics
│   │   └── storage.ts         # workspaceState persistence plus schema version migration
│   └── ui/
│       ├── petViewProvider.ts # WebviewViewProvider for the sidebar pet
│       ├── graveyardPanel.ts  # WebviewPanel listing past pets
│       ├── statusBar.ts       # compact mood and health in the status bar
│       ├── reaperGutter.ts    # editor gutter cameo of the reaper
│       ├── actions.ts         # feed / clean / walk / treatFever / exercise / antidote / whyReaper / rename / resurrect
│       ├── protocol.ts        # typed host <-> webview messages (shared with webview)
│       └── strings.ts         # all user-facing copy
├── webview/                   # code that runs inside the webview (bundled separately)
│   ├── main.ts
│   ├── render.ts              # draws the reaper scene, the bare LCD with the spider, stat bars and action row
│   ├── expressions.ts         # composites eyes, mouths, sweat, fear/fever/poison overlays at face anchors
│   ├── reaperDirector.ts      # reaper timelines and the pet's reaction choreography
│   ├── sprites.ts             # original pixel-art bitmaps as data (no image files needed)
│   └── styles.css
├── test/
│   ├── core/                  # vitest + fast-check
│   ├── collectors/
│   ├── fixtures/              # coverage reports, source with TODOs, fake secrets (allowlisted path)
│   ├── helpers/makeRepo.ts    # builds throwaway git repos with backdated branches
│   └── integration/           # @vscode/test-electron
├── scripts/
│   ├── make-demo-repo.mjs     # builds the haunted demo repo used in the video
│   ├── demo-heal.sh           # heals the demo repo on camera
│   └── self-scan.mjs          # prints Repogotchi's own vitals (used by a Kiro hook)
├── media/
│   ├── icon.png
│   └── reaper-gutter.svg      # 16x16 original reaper for the editor gutter
├── package.json
└── README.md
```

## Dependency direction

`ui` → `services` → `collectors` + `core`. `collectors` → `core/types` only. `core` → nothing.
A lower layer must never import from a higher one. `webview/` may import only `src/ui/protocol.ts` and `src/core/types.ts`.

## Naming

- Files use camelCase (`petService.ts`). Types use PascalCase. Commands use the `repogotchi.` prefix.
- Test files mirror source paths: `src/core/scoring.ts` → `test/core/scoring.test.ts`.
- Property tests are named `*.prop.test.ts`.
