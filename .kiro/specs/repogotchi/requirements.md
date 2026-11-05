# Requirements Document

## Introduction

Repogotchi is an editor extension for Kiro and VS Code. It shows a virtual pet that enforces good engineering habits. Its wellbeing comes from real signals in the open git repository and editor: test coverage, open TODO-style markers, stale unmerged branches, errors in the Problems panel, and commit habits. A committed secret poisons it instantly. Positive streaks make it flourish. When the developer starts slipping, a grim reaper peeks in from the side of the pet's panel, and the pet reacts with visible fear. The pet grows through life stages, changes mood with repo health, can die from prolonged neglect, and can be resurrected once the repo recovers. Care actions in the UI map one-to-one to real developer actions.

## Glossary

- **Workspace repo:** the git repository at the root of the first workspace folder.
- **Vitals:** Fullness, Cleanliness, Spirit, Calm and Fitness, each an integer from 0 to 100 where higher is better.
- **Health:** the weighted combination of vitals, plus any streak bonus and minus any poison cap, an integer from 0 to 100.
- **Marker:** a case-sensitive match of a configured token (default `TODO`, `FIXME`, `HACK`, `XXX`) as a whole word in a tracked text file.
- **Stale branch:** a local or remote-tracking branch that is not the default branch, not the current branch, not merged into the default branch, and whose tip commit is older than `repogotchi.staleBranchDays`.
- **Diagnostic:** an Error or Warning entry in the editor's Problems panel for a file inside the workspace repo.
- **Secret finding:** a match of a built-in or configured secret rule in a tracked or staged file, or a tracked `.env`-style file.
- **Poisoned:** the condition of the pet while at least one unallowed secret finding exists.
- **My commits:** non-merge commits on any local branch whose author email equals `git config user.email`. If that is unset, all non-merge commits.
- **Test file:** a path matching `repogotchi.testGlobs`.
- **Active day:** a local calendar day (by the Clock) with at least one of my commits.
- **Streak:** a run of active days that each meet a streak rule, where gaps of up to `repogotchi.streakGraceDays` idle days do not break the run.
- **Reaper level:** 0 (absent), 1 (peeking), 2 (lurking) or 3 (looming), derived from health and its trend.
- **Scan:** one collection of the repo signals plus recomputation of pet state. Diagnostics update through a faster path between scans.
- **Clock:** the time source used by the extension. It may be offset in demo mode.

## Requirements

### Requirement 1: Test coverage feeds the pet

**User Story:** As a developer, I want my test coverage to determine how well fed my pet is, so that writing tests feels rewarding.

#### Acceptance Criteria

1.1 WHEN a scan runs THE SYSTEM SHALL search the workspace repo for a coverage report using the glob list in `repogotchi.coverageFiles` (default: `coverage/lcov.info`, `coverage/coverage-summary.json`, `coverage.xml`, `coverage/cobertura-coverage.xml`, `**/lcov.info` excluding `node_modules`).
1.2 WHEN an LCOV report is found THE SYSTEM SHALL compute line coverage as the sum of `LH` divided by the sum of `LF` across all records.
1.3 WHEN an Istanbul `coverage-summary.json` is found THE SYSTEM SHALL use `total.lines.pct`.
1.4 WHEN a Cobertura XML report is found THE SYSTEM SHALL use the root `line-rate` attribute multiplied by 100.
1.5 WHEN more than one report is found THE SYSTEM SHALL use the most recently modified report.
1.6 THE SYSTEM SHALL set Fullness equal to line coverage percentage, rounded to the nearest integer and clamped to 0–100.
1.7 IF no coverage report is found THEN THE SYSTEM SHALL set Fullness to `repogotchi.noCoverageFullness` (default 25) and show the hint "No food found. Generate a coverage report to feed me."
1.8 IF a coverage report cannot be parsed THEN THE SYSTEM SHALL treat it as not found, log the parse error to the Repogotchi output channel, and continue the scan.
1.9 WHEN a coverage report is older than the latest commit on the current branch by more than 7 days THE SYSTEM SHALL mark Fullness as "stale food" in the UI without changing its value.

### Requirement 2: Open TODOs make the pet's home dirty

**User Story:** As a developer, I want open TODO markers to lower my pet's cleanliness, so that I'm nudged to resolve tech debt.

#### Acceptance Criteria

2.1 WHEN a scan runs THE SYSTEM SHALL enumerate tracked files using `git ls-files` and scan only text files no larger than 1 MB.
2.2 THE SYSTEM SHALL count every marker occurrence from `repogotchi.todoPatterns` as a whole word.
2.3 THE SYSTEM SHALL compute marker density as markers per 1,000 non-empty lines across scanned files.
2.4 THE SYSTEM SHALL compute Cleanliness as `round(100 × e^(−density / repogotchi.todoTolerance))` where `todoTolerance` defaults to 4.
2.5 THE SYSTEM SHALL record, for each marker, the file path, line number, token and trimmed line text, so that the Clean action can list them.
2.6 THE SYSTEM SHALL exclude paths matching `repogotchi.excludeGlobs` (default: `**/node_modules/**`, `**/dist/**`, `**/vendor/**`, `**/*.min.*`).
2.7 IF the repo has zero non-empty lines scanned THEN THE SYSTEM SHALL set Cleanliness to 100.

### Requirement 3: Stale branches haunt the pet

**User Story:** As a developer, I want abandoned branches to lower my pet's spirit, so that I keep my branch list tidy.

#### Acceptance Criteria

3.1 THE SYSTEM SHALL determine the default branch from `refs/remotes/origin/HEAD` if present, else from `init.defaultBranch`, else the first existing of `main` or `master`.
3.2 WHEN a scan runs THE SYSTEM SHALL list branches with `git for-each-ref` over `refs/heads` and `refs/remotes`, de-duplicating a local branch and its upstream into one entry.
3.3 THE SYSTEM SHALL classify a branch as stale when it meets every condition in the glossary definition, using the Clock to measure age.
3.4 THE SYSTEM SHALL compute Spirit as `max(0, 100 − repogotchi.ghostWeight × staleCount)` where `ghostWeight` defaults to 15.
3.5 IF the default branch cannot be determined THEN THE SYSTEM SHALL skip merge checks, treat only age as the stale criterion, and show a one-time warning.

### Requirement 4: Health and mood

**User Story:** As a developer, I want one clear health number and a visible mood, so that I can tell at a glance how my repo is doing.

#### Acceptance Criteria

4.1 THE SYSTEM SHALL compute base Health as the weighted mean of Fullness, Cleanliness, Spirit, Calm and Fitness using `repogotchi.weights` (default 0.25 / 0.20 / 0.15 / 0.20 / 0.20), normalising weights to sum to 1, rounded to the nearest integer.
4.2 THE SYSTEM SHALL map Health to mood: Flourishing (≥ 80 and at least one streak of 7 or more days), Thriving (≥ 80), Content (60–79), Grumpy (40–59), Sick (20–39), Dying (< 20). Poisoned overrides mood as described in Requirement 14.
4.3 WHEN any vital increases between consecutive scans THE SYSTEM SHALL play a positive reaction (eat, tidy or cheer) for that vital.
4.4 WHEN Health crosses a mood boundary downward THE SYSTEM SHALL show one non-modal notification naming the main cause, at most once per 6 hours per mood.
4.5 THE SYSTEM SHALL apply adjustments in this order: base Health, then streak bonus (Requirement 16), then the poison cap (Requirement 14), then clamp to 0–100.

### Requirement 5: Life cycle and evolution

**User Story:** As a developer, I want my pet to grow up over time, so that consistent care feels like progress.

#### Acceptance Criteria

5.1 WHEN the extension activates in a repo with no saved pet THE SYSTEM SHALL create a pet in the Egg stage with a generated name and generation number 1.
5.2 WHEN the first successful scan completes THE SYSTEM SHALL hatch the Egg into the Baby stage.
5.3 THE SYSTEM SHALL advance stages by age since hatching: Baby (< 1 day), Child (1–3 days), Teen (3–7 days), Adult (≥ 7 days).
5.4 WHEN a pet becomes Adult THE SYSTEM SHALL choose its form from lifetime time-weighted average Health: Moonweaver (≥ 75), Dustcrawler (45–74), Gloomfang (< 45).
5.5 THE SYSTEM SHALL keep a health history of at most 500 samples per pet, downsampling older entries, for evolution and the history sparkline.

### Requirement 6: Death and resurrection

**User Story:** As a developer, I want neglect to have consequences and recovery to be celebrated, so that the stakes feel real.

#### Acceptance Criteria

6.1 WHILE Health remains below 20 THE SYSTEM SHALL track the continuous duration of critical illness using the Clock.
6.2 WHEN critical illness lasts `repogotchi.deathGraceHours` (default 72) THE SYSTEM SHALL mark the pet as Dead and show the ghost.
6.3 WHEN Health returns to 20 or above before the grace period ends THE SYSTEM SHALL reset the critical-illness timer.
6.4 WHEN a pet dies THE SYSTEM SHALL add a graveyard entry with name, generation, form, birth time, death time, cause of death (`poison` if poisoned at death, otherwise the lowest vital at death) and a generated epitaph that cites the real metric (for example "Drowned in 214 TODOs" or "Poisoned by a committed AWS key").
6.5 WHILE the pet is Dead THE SYSTEM SHALL keep scanning and display current Health, and SHALL offer the Resurrect action only when Health is 60 or above.
6.6 WHEN the user triggers Resurrect THE SYSTEM SHALL play the resurrection animation and create a new Egg with the generation number increased by one, keeping the graveyard.
6.7 WHEN the extension activates and the saved pet was critically ill at its last scan THE SYSTEM SHALL count the time the editor was closed toward the critical-illness duration, but SHALL cap it so that at least `repogotchi.lastChanceHours` (default 24) of grace remain, and SHALL show the message "You came back. {name} is barely hanging on."

### Requirement 7: Care actions are real actions

**User Story:** As a developer, I want the pet's buttons to do real work, so that caring for the pet improves my repo.

#### Acceptance Criteria

7.1 WHEN the user triggers Feed THE SYSTEM SHALL run `repogotchi.testCommand` if set, else the workspace's default test task, else show a picker of available tasks, and SHALL rescan when it finishes.
7.2 WHEN the user triggers Clean THE SYSTEM SHALL show a quick pick of markers grouped by file, sorted by file marker count descending, and SHALL open the selected location.
7.3 WHEN the user triggers Walk THE SYSTEM SHALL show a quick pick of stale branches with age, last author and last commit subject.
7.4 WHEN the user chooses "Delete local branch" on a stale branch THE SYSTEM SHALL ask for modal confirmation and then run `git branch -d` (never `-D`), and SHALL show git's error if the branch is not fully merged.
7.5 THE SYSTEM SHALL NOT delete remote branches, modify source files or create commits.
7.6 WHEN the user triggers Rename THE SYSTEM SHALL accept a name of 1–16 printable characters and persist it.

### Requirement 8: Scanning behaviour

**User Story:** As a developer, I want the pet to stay current without slowing my editor down.

#### Acceptance Criteria

8.1 THE SYSTEM SHALL scan on activation, on the `repogotchi.scan` command, every `repogotchi.scanIntervalMinutes` (default 15), WHEN a file is saved (debounced 30 s), and WHEN `.git/HEAD` or `.git/refs/**` changes (debounced 5 s).
8.2 THE SYSTEM SHALL never run two scans concurrently. A request during a scan SHALL queue at most one follow-up scan.
8.3 THE SYSTEM SHALL complete a scan of a 5,000-file repo in under 3 seconds on a typical laptop, and SHALL yield to the event loop between file batches.
8.4 IF the workspace has no git repo THEN THE SYSTEM SHALL show an empty state ("No repo to haunt. Open a git repository.") and SHALL NOT create a pet.
8.5 THE SYSTEM SHALL make no network requests.
8.6 WHILE the pet view is hidden THE SYSTEM SHALL keep computing state but SHALL NOT run webview animations.

### Requirement 9: Haunted, accessible interface

**User Story:** As a developer, I want a memorable spooky pet that's still readable and comfortable to use.

#### Acceptance Criteria

9.1 THE SYSTEM SHALL contribute a "Repogotchi" webview view to the Explorer view container (`contributes.views.explorer`, id `repogotchi.pet`), so that it appears in the left sidebar below the file tree. The view SHALL render the pixel-art spider on a bare LCD screen, with no toy case or physical-style buttons, as described in the spooky-ui steering.
9.2 THE SYSTEM SHALL show a status bar item with mood and Health (for example `$(flame) Webster 72`) that focuses the pet view when clicked.
9.3 THE SYSTEM SHALL render every vital as text alongside its graphic, with accessible labels.
9.4 WHERE `repogotchi.spookiness` is `off` or the OS requests reduced motion THE SYSTEM SHALL disable flicker, noise, fog and screen shake.
9.5 THE SYSTEM SHALL provide a Graveyard panel listing past pets with epitaphs, lifespan and a health sparkline.
9.6 THE SYSTEM SHALL use a strict webview Content Security Policy with a nonce, loading only local bundled resources.
9.7 THE SYSTEM SHALL remain legible in light, dark and high-contrast themes.
9.8 THE SYSTEM SHALL provide `repogotchi.showPet`, which reveals and focuses the pet view, and SHALL NOT take focus on activation.
9.9 THE SYSTEM SHALL keep the panel compact. By default it shows only the LCD, a health line that reads health first and then the comment (for example "Health 73 · Webster is spinning happily."), and two dropdown buttons, "Stats" and "Actions". The stat bars and the action buttons SHALL stay hidden until the developer opens one of the dropdowns (clicking the LCD also opens Stats). Each opens as a pop-up over the screen, only one at a time, closed with × or Escape. The last choice SHALL be remembered per workspace.
9.10 THE SYSTEM SHALL label action buttons in plain words with live counts: "Run tests", "TODOs · N", "Branches · N", "Errors · N", plus "Remove secret" when poisoned and "Resurrect" when dead. It SHALL highlight the action that would recover the most health, and SHALL state in the Actions pop-up that health changes only when the code does.

### Requirement 10: Persistence

**User Story:** As a developer, I want my pet to survive editor restarts.

#### Acceptance Criteria

10.1 THE SYSTEM SHALL persist pet state and graveyard in `workspaceState`, keyed per repo root, with a `schemaVersion`.
10.2 WHEN stored state has an older `schemaVersion` THE SYSTEM SHALL migrate it, or, if migration fails, archive it under a backup key and start a new Egg.
10.3 THE SYSTEM SHALL provide `repogotchi.exportPet` to save pet state and graveyard as JSON.

### Requirement 11: Demo mode

**User Story:** As a hackathon entrant, I want to compress time on camera, so that the video can show the full life cycle in minutes.

#### Acceptance Criteria

11.1 THE SYSTEM SHALL provide `repogotchi.timeTravel` which asks for a number of hours and offsets the Clock forward by that amount, then rescans.
11.2 WHILE the Clock is offset THE SYSTEM SHALL show a visible "DEMO TIME +Nh" badge on the LCD.
11.3 THE SYSTEM SHALL provide `repogotchi.resetTime` to remove the offset.
11.4 THE SYSTEM SHALL apply the Clock offset to branch age, stage age, streak days and critical-illness duration consistently.
11.5 THE SYSTEM SHALL provide `repogotchi.summonReaper` (labelled "(Demo)") which plays reaper levels 1, 2 and 3 in sequence without changing pet state, so the reaper can be filmed on demand.

### Requirement 12: Installability

**User Story:** As a judge, I want to install and run the project easily.

#### Acceptance Criteria

12.1 THE SYSTEM SHALL package as a single `.vsix` that installs in Kiro and VS Code without extra setup beyond having `git` on `PATH`.
12.2 THE SYSTEM SHALL activate only on `workspaceContains:.git` or on use of a Repogotchi command or view.
12.3 THE SYSTEM SHALL include a README with install steps, settings, a screenshot or GIF, and a "How Kiro was used" section.

### Requirement 13: Fever from the Problems panel

**User Story:** As a developer, I want errors and warnings in my editor to give the pet a fever, so that I fix problems while they're fresh.

#### Acceptance Criteria

13.1 THE SYSTEM SHALL read diagnostics with `vscode.languages.getDiagnostics()` and SHALL count only Error and Warning severities for files inside the workspace repo that do not match `repogotchi.excludeGlobs`.
13.2 WHEN diagnostics change THE SYSTEM SHALL wait until they have been unchanged for `repogotchi.feverSettleSeconds` (default 10) before updating Calm, so that half-typed code doesn't spike a fever.
13.3 THE SYSTEM SHALL compute fever load as `errors + repogotchi.warningWeight × warnings` (`warningWeight` default 0.25) and Calm as `round(100 × e^(−load / repogotchi.feverTolerance))` (`feverTolerance` default 5).
13.4 THE SYSTEM SHALL display Calm as a body temperature: `36.6 + (100 − Calm) / 100 × 4.4` °C, shown to one decimal, or in °F WHERE `repogotchi.temperatureUnit` is `F`.
13.5 THE SYSTEM SHALL update Calm and Health through the fast path without running a full scan, and SHALL persist the result at most once per minute.
13.6 THE SYSTEM SHALL provide a Treat Fever action that opens the Problems panel, filtered to the workspace repo where the API allows.

### Requirement 14: Poison from committed secrets

**User Story:** As a developer, I want a committed secret to make the pet sick at once, so that I learn never to commit credentials.

#### Acceptance Criteria

14.1 WHEN a scan runs THE SYSTEM SHALL check every tracked or staged text file against the secret rules, during the same file pass as the TODO scan.
14.2 THE SYSTEM SHALL include built-in rules for at least: AWS access key IDs, GitHub tokens, GitLab tokens, Slack tokens, Stripe live secret keys, Google API keys, OpenAI and Anthropic API keys, private key blocks (`-----BEGIN … PRIVATE KEY-----`), and generic `key/secret/token/password = "<value>"` assignments whose value has Shannon entropy ≥ 3.5 bits per character and length ≥ 16.
14.3 THE SYSTEM SHALL treat any tracked file whose name matches `.env` or `.env.*` as a secret finding, except names ending in `.example`, `.sample` or `.template`.
14.4 THE SYSTEM SHALL ignore findings on lines containing `repogotchi:allow-secret` and in paths matching `repogotchi.secretAllowGlobs`.
14.5 WHILE at least one finding exists THE SYSTEM SHALL mark the pet Poisoned, cap Health at 19, set mood to Dying, show the poison visuals, and start or continue the critical-illness timer.
14.6 THE SYSTEM SHALL NOT store, log, send or display a full secret value. It SHALL keep only the file path, line number, rule ID and a masked preview (first 4 characters followed by `••••`).
14.7 WHEN the pet becomes Poisoned THE SYSTEM SHALL show one warning notification naming the rule and file, with the actions "Show me" (opens the line) and "How to fix".
14.8 WHEN all findings are gone THE SYSTEM SHALL clear Poisoned and show an "Antidote" message reminding the developer that the secret is still in git history and must be rotated or revoked.
14.9 THE SYSTEM SHALL provide an Antidote action listing all findings, with masked previews, that opens the selected location.

### Requirement 15: Exercise from commit habits

**User Story:** As a developer, I want small, regular, well-described commits to keep the pet fit, so that I build good version-control habits.

#### Acceptance Criteria

15.1 WHEN a scan runs THE SYSTEM SHALL read my commits from the last `repogotchi.exerciseWindowDays` (default 14) using `git log --all --no-merges --numstat`, counting the lines added plus deleted per commit and excluding paths that match `excludeGlobs` or `repogotchi.lockfileGlobs`.
15.2 THE SYSTEM SHALL compute regularity as `min(1, activeDays / repogotchi.exerciseTargetDays)` (`exerciseTargetDays` default 8).
15.3 THE SYSTEM SHALL compute smallness as the share of my commits whose changed lines are at most `repogotchi.smallCommitLines` (default 200).
15.4 THE SYSTEM SHALL compute message quality as the share of my commits whose subject is 10–72 characters long, is not on the low-effort list (`wip`, `fix`, `fixes`, `update`, `updates`, `changes`, `stuff`, `misc`, `temp`, `test`, `asdf`, `.`, case-insensitive, ignoring trailing punctuation), and contains at least two words.
15.5 THE SYSTEM SHALL compute Fitness as `round(100 × (0.4 × regularity + 0.3 × smallness + 0.3 × quality))`. IF there are none of my commits in the window THEN smallness and quality SHALL each be 0.5.
15.6 THE SYSTEM SHALL show the three parts in the Fitness stat page, for example "Active 5/8 days · 82% small · 64% clear messages".
15.7 THE SYSTEM SHALL provide an Exercise action that shows the fitness parts, my five most recent unclear commit subjects and the clear-message rule, without rewriting history.

### Requirement 16: Streaks and rewards

**User Story:** As a developer, I want the pet to flourish when I keep up good habits, so that doing the right thing is rewarded and not only slacking punished.

#### Acceptance Criteria

16.1 THE SYSTEM SHALL track three streaks from git history and pet history:
  - **Tested:** each active day has at least one of my commits that changes both a test file and a non-test source file.
  - **Clear messages:** every one of my commits on each active day meets the quality rule in 15.4.
  - **Healthy:** each local day on which a scan ran had no scan with Health below 80 and no Poisoned scan.
16.2 THE SYSTEM SHALL allow up to `repogotchi.streakGraceDays` (default 2) consecutive idle days inside a Tested or Clear-messages streak without breaking it. Idle days SHALL NOT add to the count.
16.3 THE SYSTEM SHALL add a streak bonus to Health of +3 for each streak at 7 days or more, up to +6 in total.
16.4 WHEN a streak first reaches 3, 7, 14 or 30 days in a pet's life THE SYSTEM SHALL award a cosmetic trophy (Candle, Lantern Glow, Flame Crown, Bat Familiar), play a celebration, and show it on the pet from then on.
16.5 WHEN a pet dies THE SYSTEM SHALL record its trophies and longest streaks in its graveyard entry.
16.6 THE SYSTEM SHALL count Tested and Clear-message streak days at or above 7 as "great care" for evolution, raising the time-weighted average health used in 5.4 by 5 points, capped at 100.
16.7 WHILE the pet is Poisoned THE SYSTEM SHALL pause streak bonuses and the Flourishing mood without resetting streak counts.

### Requirement 17: The grim reaper

**User Story:** As a developer, I want a grim reaper to start lurking when I'm slipping, and my pet to look visibly scared, so that I notice decline early and feel motivated to turn it around.

#### Acceptance Criteria

17.1 THE SYSTEM SHALL compute the reaper level as the highest of: 1 if mood is Grumpy, 2 if mood is Sick, 3 if mood is Dying or the pet is Poisoned, and 1 if Health has dropped by `repogotchi.reaperDropPoints` (default 10) or more from its highest value in the last 24 hours. Otherwise the level SHALL be 0. The reaper SHALL NOT appear for an Egg or a Dead pet.
17.2 WHILE the reaper level is 1 THE SYSTEM SHALL make the reaper peek in from the side edge of the pet view for about 6 seconds, roughly every 4 minutes (randomised ±30%).
17.3 WHILE the reaper level is 2 THE SYSTEM SHALL make the reaper lurk half-visible for about 12 seconds, roughly every 90 seconds.
17.4 WHILE the reaper level is 3 THE SYSTEM SHALL keep the reaper in view holding an hourglass whose sand level shows the critical-illness time remaining before death.
17.5 WHEN the reaper appears THE SYSTEM SHALL play the reaction sequence: the pet's eyes turn toward the reaper, hold for about 1 second, turn back to face the developer, then a sweat drop and a blue tint across the forehead fade in. At level 2 the pet also trembles. At level 3 the pet hides, leaving only its eyes visible.
17.6 WHEN the reaper leaves at levels 1 or 2 THE SYSTEM SHALL fade the blue tint over about 3 seconds. WHEN the reaper level drops to 0 THE SYSTEM SHALL play a retreat (the reaper shrugs and backs away) and a relief reaction (the pet wipes its brow).
17.7 WHEN the developer clicks or activates the reaper THE SYSTEM SHALL show a "Why is the reaper here?" panel listing the top three causes, ordered by how many health points each costs, each with a button for its care action.
17.8 THE SYSTEM SHALL let `repogotchi.reaperSide` (`auto` | `left` | `right`, default `auto`) choose the reaper's entry edge. With `auto`, the reaper enters from the edge facing the editor: the right edge when `workbench.sideBar.location` is `left`, and the left edge when it is `right`. (Extensions can't detect whether a view was dragged into the secondary sidebar, so users who do that can set the side manually.)
17.9 WHERE `repogotchi.reaperInEditor` is `slipping` or `dying` (default `dying`) THE SYSTEM SHALL also show a small reaper gutter icon in the active editor at reaper levels 2–3 or level 3 respectively. The icon goes on the line of the first Error in that file, or the first visible line if there is none, and is removed when the level falls.
17.10 WHERE reduced motion or spookiness `off` applies THE SYSTEM SHALL show the reaper as a static frame at the edge without sliding, trembling or hiding motion, and SHALL still show the sweat drop and tint.
17.11 THE SYSTEM SHALL announce reaper changes through the aria-live region, for example "The reaper is watching. Main cause: 6 errors in the Problems panel."
