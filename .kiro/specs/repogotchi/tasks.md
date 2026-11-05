# Implementation Plan

- [ ] 1. Scaffold the extension project
  - Create `package.json` with name `repogotchi`, `engines.vscode ^1.90.0`, activation events `workspaceContains:.git` plus the view and commands, and all scripts from the tech steering
  - Contribute the pet as a webview view in the Explorer container: `contributes.views.explorer: [{ "type": "webview", "id": "repogotchi.pet", "name": "Repogotchi" }]`, so it docks in the left sidebar below the file tree
  - Add `tsconfig.json` (strict), an esbuild build for `dist/extension.js` and `dist/webview.js`, ESLint config, and Vitest config with v8 coverage writing `coverage/lcov.info` (jsdom environment for `test/webview/**`)
  - Stub `src/extension.ts` with empty `activate`/`deactivate` and an output channel named "Repogotchi"
  - Verify that `npm run build`, `npm run typecheck` and `npm test` (with zero tests) all pass
  - _Requirements: 12.1, 12.2_

- [ ] 2. Core types and clock
  - [ ] 2.1 Implement `src/core/types.ts` exactly as in design.md
    - _Requirements: 4.1, 5.1, 6.4_
  - [ ] 2.2 Implement `src/core/clock.ts` with `Clock`, `SystemClock`, `OffsetClock`, a `FakeClock` test helper in `test/helpers/`, and a pure `localDay(t, tzOffsetMinutes)` returning `YYYY-MM-DD`
    - Unit-test the offset arithmetic and day boundaries around midnight
    - _Requirements: 11.1, 11.3, 11.4_

- [ ] 3. Scoring, mood and causes
  - [ ] 3.1 Implement `scoring.ts` for all five vitals, the temperature conversion, and `healthFrom` with base, streak bonus and poison cap in the order from Req 4.5
    - Table tests for every example in the design scoring tables (for example 1 error → Calm 82, Calm 37 → 39.4 °C)
    - _Requirements: 1.6, 1.7, 2.3, 2.4, 2.7, 3.4, 4.1, 4.5, 13.3, 13.4, 14.5, 16.3, 16.7_
  - [ ] 3.2 Implement `mood.ts` including Flourishing and the poison override, with boundary tests at 19/20, 39/40, 59/60, 79/80
    - _Requirements: 4.2_
  - [ ] 3.3 Implement `causes.ts` (top three causes by health points lost, poison first)
    - _Requirements: 17.7_
  - [ ]* 3.4 Property tests `scoring.prop.test.ts` for correctness properties 1, 2, 9 and 11
    - _Requirements: 4.1, 14.5, 16.3_

- [ ] 4. Secret detection (core)
  - [ ] 4.1 Implement `secrets.ts`: the rule table, Shannon entropy, placeholder skipping, the `repogotchi:allow-secret` line marker, custom-rule compilation with invalid-regex dropping, and `scanSecrets(path, text, rules)` that masks at the source
    - One positive, one placeholder and one near-miss test per rule. Fake secrets are built at runtime by string concatenation in the tests, so the repo never contains a literal match
    - _Requirements: 14.2, 14.4, 14.6_
  - [ ]* 4.2 Property test `secrets.prop.test.ts` for correctness property 10 (no field or log line ever contains more than 4 characters of a matched value)
    - _Requirements: 14.6_

- [ ] 5. Exercise, streaks and trophies (core)
  - [ ] 5.1 Implement `exercise.ts`: `isClearMessage`, regularity, smallness, quality, and Fitness with the no-commits rule
    - _Requirements: 15.2, 15.3, 15.4, 15.5_
  - [ ] 5.2 Implement `streaks.ts`: Tested, Clear and Healthy streaks, grace days, today's grace, and `longest`
    - Example tests: 7 tested days in a row; a weekend gap inside grace; an untested active day resets; the Healthy streak broken by a poisoned day
    - _Requirements: 16.1, 16.2_
  - [ ] 5.3 Implement `trophies.ts` with milestones 3/7/14/30, once per trophy per pet
    - _Requirements: 16.4_
  - [ ]* 5.4 Property tests for correctness properties 13 and 14
    - _Requirements: 15.5, 16.2_

- [ ] 6. Life cycle reducer
  - [ ] 6.1 Implement `lifecycle.ts` `step()` for `scan`: hatching, stage advance by age, the critical-illness timer (including poison), death, history append and downsampling, `dayLog` updates, poison and antidote transitions, and the effects list
    - Example tests: egg hatches on first scan; Baby→Child at exactly 24h; timer resets at Health 20; death at exactly `deathGraceHours`; poisoning a healthy pet starts the timer at once
    - _Requirements: 5.1, 5.2, 5.3, 5.5, 6.1, 6.2, 6.3, 14.5, 14.8_
  - [ ] 6.2 Implement the `diagnostics` fast-path event (updates Calm, Health, mood and timer only; history sample at most every 5 minutes)
    - _Requirements: 13.5_
  - [ ] 6.3 Implement the resume rule with the `lastChance` effect
    - _Requirements: 6.7_
  - [ ] 6.4 Implement `evolution.ts` (time-weighted average plus the streak boost) and wire it into the Teen→Adult transition
    - _Requirements: 5.4, 16.6_
  - [ ] 6.5 Implement `epitaph.ts` (all six causes) and `names.ts`, both seeded and deterministic, plus the `resurrect` and `rename` events. Record trophies and longest streaks in the grave entry
    - Resurrect is rejected (no-op plus an effect explaining why) when Health < 60
    - _Requirements: 6.4, 6.5, 6.6, 7.6, 16.5_
  - [ ]* 6.6 Property tests `lifecycle.prop.test.ts` for correctness properties 3–8
    - _Requirements: 5.3, 5.5, 6.2, 6.5, 11.4_

- [ ] 7. Reaper level (core)
  - Implement `reaper.ts` `reaperLevel()` with the mood, poison and 24h-drop rules, returning 0 for egg and dead pets
  - Property test for correctness property 12
  - _Requirements: 17.1_

- [ ] 8. Checkpoint: core complete
  - Run `npm run test:coverage`. `src/core/` must be ≥ 90% line coverage. Fix gaps before continuing.

- [ ] 9. Git and branch collector
  - [ ] 9.1 Implement `git.ts` `runGit` with typed errors (`not-installed`, `not-a-repo`, `timeout`, exit code)
    - _Requirements: 8.4_
  - [ ] 9.2 Create `test/helpers/makeRepo.ts`: temp repo with commits, merged and unmerged branches, backdated commits via `GIT_COMMITTER_DATE`/`GIT_AUTHOR_DATE`, a configurable `user.email`, and an optional bare `origin` remote
  - [ ] 9.3 Implement `branches.ts`: default-branch detection, pure `parseForEachRef`, pure `classifyStale`, local/remote de-duplication
    - Fixture-string tests for the parser; temp-repo tests for detection and classification, including the no-default-branch fallback
    - _Requirements: 3.1, 3.2, 3.3, 3.5_

- [ ] 10. Content scan: TODOs and secrets
  - [ ] 10.1 Implement the pure `scanText(path, text, tokens)` and a tiny glob matcher for `excludeGlobs`, `secretAllowGlobs`, `testGlobs` and `lockfileGlobs`
    - Tests: whole-word matching (`TODOS` and `mytodo` don't count), case sensitivity, CRLF line endings, line numbers
    - _Requirements: 2.2, 2.5, 2.6_
  - [ ] 10.2 Implement `contentScan.ts`: `git ls-files -z --cached`, `.env` path findings, size and binary filtering, batched reads with event-loop yielding, one pass for markers and secrets, non-empty line counting
    - Temp-repo tests: a staged-but-uncommitted secret is found; `.env.example` is ignored; an allowlisted path is ignored. Perf test with 5,000 generated files in under 3 s (skippable in CI with an env flag)
    - _Requirements: 2.1, 2.3, 8.3, 14.1, 14.3, 14.4_

- [ ] 11. Commits collector
  - Implement `commits.ts`: identity from `user.email` with the all-authors fallback, the `git log --all --no-merges --numstat` call, pure `parseLogNumstat`, binary numstat handling, lockfile exclusion, and `touchesTest`/`touchesSource`
  - Temp-repo tests with backdated commits by two authors, a lockfile-only commit and a binary file
  - _Requirements: 15.1, 16.1_

- [ ] 12. Coverage collector
  - [ ] 12.1 Implement the `lcov.ts`, `istanbulSummary.ts` and `cobertura.ts` parsers with fixtures in `test/fixtures/coverage/`, including malformed files
    - _Requirements: 1.2, 1.3, 1.4, 1.8_
  - [ ] 12.2 Implement `coverage/index.ts`: choose the newest candidate by mtime, return `missing`/`unparseable`/`found`, and compute the stale-food flag against `lastCommitTime`
    - _Requirements: 1.1, 1.5, 1.7, 1.9_

- [ ] 13. Fever: diagnostics
  - [ ] 13.1 Implement the pure `collectors/diagnostics.ts` `summarizeDiagnostics` (Error and Warning only, inside the repo, not excluded, with first error line per file)
    - _Requirements: 13.1_
  - [ ] 13.2 Implement `services/diagnosticsWatcher.ts` with the settle timer, mapping `getDiagnostics()` to plain entries, and disposal
    - Unit-test the settle behaviour with fake timers (rapid changes give exactly one update)
    - _Requirements: 13.2, 13.5_

- [ ] 14. Services
  - [ ] 14.1 Implement `storage.ts`: per-repo key, `schemaVersion`, pure `migrate`, backup on failure, export to JSON
    - _Requirements: 10.1, 10.2, 10.3_
  - [ ] 14.2 Implement `petService.ts`: parallel collection with per-collector error boundaries (a secret-scan error never clears existing poison), the `step()` call, `applyDiagnostics` with persistence throttled to once a minute, the `onDidChange` event, and the latest markers and findings kept in memory only
    - _Requirements: 4.3, 4.4, 8.4, 8.5, 13.5, 14.6_
  - [ ] 14.3 Implement `scanScheduler.ts`: single-flight with one queued follow-up, interval, save debounce (30 s), and a `.git/HEAD`, `.git/index`, `.git/refs/**` watcher debounce (5 s), plus disposal
    - Unit-test single-flight with fake timers
    - _Requirements: 8.1, 8.2_

- [ ] 15. Checkpoint: headless pet works
  - Add `scripts/self-scan.mjs`, which builds the core and collectors with esbuild into a temp file and prints this repo's five vitals, temperature, poison status, streaks, mood, stage and reaper level as a small ASCII card. Run it against this repo and confirm the numbers are plausible and that Repogotchi is **not** poisoned by its own test fixtures.

- [ ] 16. Webview: the spider on its LCD
  - [ ] 16.1 Implement `protocol.ts` with type guards, shared by the host and the webview
    - _Requirements: 9.1_
  - [ ] 16.2 Implement `petViewProvider.ts` for the Explorer view `repogotchi.pet` (left sidebar, below the file tree) plus `repogotchi.showPet`: CSP with nonce, local resource roots, a `ready` handshake that re-sends state, the empty states, and pausing when the view is hidden
    - _Requirements: 8.4, 8.6, 9.1, 9.6, 9.8_
  - [ ] 16.3 Draw original pixel-art spiders in `webview/sprites.ts`: egg sac, Speck, Skitter, Weaver, Moonweaver, Dustcrawler, Gloomfang and the ghost spider, with face anchors; the corner web (intact and torn), silk thread, caught bug, ghost-flowers, gravestone, skull bottle and lantern; the shared eye and mouth sets, the reaper (idle, drum fingers, hourglass, shrug), the tombstone prop, trophies, and the LCD pixel font
    - _Requirements: 9.1, 16.4, 17.5_
  - [ ] 16.4 Implement `webview/render.ts` and `styles.css` with **no toy case**: the scene layer behind a bare LCD at the top of the view; a compact panel (about 380 px) showing only the health line (`Health 73 · Webster is spinning happily.`) and two dropdown buttons, `Stats ▾` and `Actions ▾`; a **Stats pop-up** over the LCD (also opened by clicking the LCD) holding the FOOD/WEB/GHOSTS/TEMP/FIT bars; an **Actions pop-up** with plain-label buttons (Run tests, TODOs · N, Branches · N, Errors · N; Remove secret when poisoned; Resurrect when dead), the top-cost action highlighted, and the "health only changes when your code does" footer. Only one pop-up is open at a time, × or Escape closes it, and the choice is remembered per workspace; web holes, caught bugs and ghost edge icons; mood effects; fever, poison and flourishing overlays; and the DEMO TIME badge. Match the "Repogotchi Spider Mockup" canvas
    - _Requirements: 9.1, 9.3, 9.7, 9.9, 9.10, 11.2, 13.4, 15.6_
  - [ ] 16.5 Implement effect gating for spookiness and reduced motion, capping flashes at 3 per second
    - _Requirements: 9.4, 17.10_
  - [ ] 16.6 Make the dropdown toggles and pop-up buttons real accessible buttons (at least 44 px tall, `aria-expanded` on toggles, focus trapped in an open pop-up) and add an aria-live description below the screen, including reaper, fever and poison announcements
    - _Requirements: 9.3, 17.11_

- [ ] 17. The grim reaper and the pet's reactions
  - [ ] 17.1 Implement `webview/expressions.ts`: composite eyes and mouths at face anchors; sweat drop on the side away from the reaper; fear tint gradient over the forehead; fade timings
    - jsdom tests check that anchors place eyes correctly for every creature
    - _Requirements: 17.5, 17.6_
  - [ ] 17.2 Implement `webview/reaperDirector.ts`: the timeline engine, pure `nextAppearance(level, rng)` with ±30% jitter, the level 1/2/3 sequences from the design table, the hiding pose behind the tombstone, the reaper standing behind the LCD so only its hood, eyes and blade show until it looms, the hourglass driven by `criticalRemaining`, and the retreat plus relief when the level drops to 0
    - Fake-timer tests for each sequence's cue order, and that motion gating collapses cues to end frames
    - _Requirements: 17.2, 17.3, 17.4, 17.5, 17.6, 17.10_
  - [ ] 17.3 Resolve the reaper side from `repogotchi.reaperSide` and `workbench.sideBar.location`, and add the focusable reaper button that posts `whyReaper`
    - _Requirements: 17.7, 17.8_
  - [ ] 17.4 Implement `ui/reaperGutter.ts` with the original 16×16 `media/reaper-gutter.svg`, first-error-line placement, hover message, throttled refresh and cleanup
    - _Requirements: 17.9_

- [ ] 18. Status bar, actions and commands
  - [ ] 18.1 Implement `statusBar.ts` with the codicon per state (including `$(beaker)` poisoned and `$(sparkle)` flourishing) and a tooltip listing all vitals, temperature and streaks
    - _Requirements: 9.2_
  - [ ] 18.2 Implement the Feed action (test command, then default test task, then picker) and rescan on task end
    - _Requirements: 7.1_
  - [ ] 18.3 Implement the Clean quick pick grouped by file
    - _Requirements: 7.2_
  - [ ] 18.4 Implement the Walk quick pick with Checkout, a modal-confirmed `git branch -d`, and Copy name. Never `-D`, never remote
    - _Requirements: 7.3, 7.4, 7.5_
  - [ ] 18.5 Implement Treat Fever, Exercise, Antidote (masked quick pick plus the rotate-your-key footer) and Why Reaper
    - _Requirements: 13.6, 14.9, 15.7, 17.7_
  - [ ] 18.6 Implement Rename, Resurrect, Scan, Export, Time Travel, Reset Time and Summon Reaper (Demo), and register every command in `package.json` under the "Repogotchi" category
    - _Requirements: 6.6, 7.6, 10.3, 11.1, 11.3, 11.5_
  - [ ] 18.7 Wire notifications: mood drops (6-hour throttle), `lastChance`, `poisoned` (with "Show me" and "How to fix"), `antidote`, and trophy celebrations
    - _Requirements: 4.4, 6.7, 14.7, 14.8, 16.4_

- [ ] 19. Graveyard panel
  - Implement `graveyardPanel.ts` with tombstone cards (name, generation, form, lifespan, epitaph, trophies, longest streaks) and an inline SVG health sparkline, using the same CSP rules
  - _Requirements: 9.5, 9.6, 16.5_

- [ ] 20. Integration smoke tests
  - Use @vscode/test-electron to check activation in a fixture git repo, that every contributed command is registered, that the webview resolves, that a non-git folder shows the empty state with no pet stored, and that injecting an Error through a `DiagnosticCollection` lowers Calm after the settle time
  - _Requirements: 8.4, 12.2, 13.1, 13.2_

- [ ] 21. Demo tooling
  - [ ] 21.1 `scripts/make-demo-repo.mjs <dir>`: about 25 markers, an 18% `coverage/lcov.info`, 6 unmerged branches backdated 40–200 days, 10 days of history with lazy "wip"/"fix" commits, a ready-to-paste fake AWS-shaped key generated at random in `DEMO_SECRET.txt` (outside git, never a real credential), and a `src/broken.ts` snippet that introduces type errors
  - [ ] 21.2 `scripts/demo-heal.sh <dir>`: swap in an 85% report, remove most markers, and fix `src/broken.ts`. Stale branches are cleaned through the Walk action on camera
  - _Requirements: 11.1, 11.5_

- [ ] 22. Package and document
  - [ ] 22.1 Finish the existing `README.md`. Keep it for people using Repogotchi only: the pitch, demo plan and submission notes live in the gitignored `pitch.md`. Add a GIF of the reaper scene, check the install steps and settings table against the built extension, and fill in "How it was built with Kiro" from `KIRO_LOG.md`
    - _Requirements: 12.3_
  - [ ] 22.2 Add the icon, `LICENSE`, `CHANGELOG.md` and `.vscodeignore`. Run `npm run package`, install the `.vsix` in a clean Kiro profile, and walk through the demo checklist in `/demo-prep`
    - _Requirements: 12.1_
