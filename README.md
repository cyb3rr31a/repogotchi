# Repogotchi

A retro pixel-art spider that lives in your editor and keeps your coding habits honest. Write tests, clear your TODOs, tidy your branches and fix your errors, and it flourishes. Slack off, and the grim reaper starts peeking in.

Repogotchi is an extension for **Kiro** and **VS Code**, inspired by 90s virtual pets. It is not affiliated with any toy maker.

> **Status:** in development for the Kiro hackathon. Features below describe the planned 1.0.

## What it does

The spider, Webster by default, lives on a small green LCD at the bottom of the Explorer sidebar. Its health comes from your actual repository and editor. Nothing is random.

| Your repo | What the spider shows |
|---|---|
| Test coverage | **Food.** Tests catch bugs, and bugs are its food. |
| TODO / FIXME / HACK / XXX markers | **Holes in its web.** |
| Stale, unmerged branches | **Ghosts** that scare it. |
| Errors and warnings in the Problems panel | **A fever**, shown as a temperature from 36.6 °C up to 41 °C. |
| Small, regular commits with clear messages | **Fitness.** |

- **Poison:** a committed or staged secret (API key, token, private key, tracked `.env`) poisons the spider immediately, whatever its other stats.
- **Streaks:** seven days of tested commits, clear commit messages or good health earn a health bonus, a flourishing mood and trophies it keeps for life.
- **The grim reaper:** when you start slipping, it peeks in from the side of the panel. The spider glances at it, looks back at you, sweats and goes blue. Click the reaper to see exactly why it's there.
- **Death and resurrection:** after 72 hours critically ill, the spider dies and goes to the graveyard with an epitaph. Get the repo healthy again and you can resurrect it as the next generation.

## Using the panel

Under the screen you'll see a health line, such as `Health 73 · Webster is spinning happily.`, and two dropdowns:

- **Stats ▾** shows Food, Web, Ghosts, Temp and Fitness. You can also click the screen to open it.
- **Actions ▾** gives you shortcuts to the work: **Run tests**, **TODOs**, **Branches** and **Errors** (plus **Remove secret** when poisoned). The most useful one right now is highlighted.

The buttons don't change health by themselves. Health only changes when your code does.

## Install

Requirements: Kiro or VS Code 1.90 or later, and `git` on your PATH.

1. Download the latest `repogotchi-<version>.vsix` from the Releases page.
2. In Kiro or VS Code, open the Extensions view, choose the **…** menu, then **Install from VSIX…**, and pick the file.
3. Open any git repository. The spider appears at the bottom of the Explorer sidebar. If you don't see it, run **Repogotchi: Show Pet** from the Command Palette.

## Give it real data

- **Coverage:** Repogotchi reads reports your test tools already produce. It looks for `coverage/lcov.info`, `coverage/coverage-summary.json`, `coverage.xml` and `coverage/cobertura-coverage.xml`. Most test runners can write one of these with a coverage flag.
- **Fitness:** it counts commits by the author in `git config user.email`, so make sure that's set.
- **Fever:** it follows whatever your language extensions report in the Problems panel, usually for the files you have open.

## Settings

Everything is under **Settings → Extensions → Repogotchi**. The ones you're most likely to change:

| Setting | Default | What it does |
|---|---|---|
| `repogotchi.staleBranchDays` | 30 | Age at which an unmerged branch becomes a ghost |
| `repogotchi.todoPatterns` | TODO, FIXME, HACK, XXX | Which markers count as holes in the web |
| `repogotchi.testCommand` | (empty) | Command for Run tests. Empty means your default test task. |
| `repogotchi.spookiness` | `mild` | `off`, `mild` or `full`. `off` removes flicker, fog and motion. |
| `repogotchi.reaperInEditor` | `dying` | Show a tiny reaper in the editor gutter: `off`, `slipping` or `dying` |
| `repogotchi.temperatureUnit` | `C` | `C` or `F` |
| `repogotchi.secretAllowGlobs` | `**/test/fixtures/**` | Paths where fake secrets are allowed |

## Privacy and safety

- Repogotchi makes **no network requests** and collects **no telemetry**.
- Detected secrets are never stored, logged or shown beyond their first four characters.
- Secret detection is a nudge, not a guarantee. Use a dedicated scanner or your Git host's push protection for real security. If you ever commit a key, rotate it.
- Repogotchi never edits your code, rewrites history or deletes remote branches. Deleting a local branch always asks for confirmation first.
- Effects respect your system's reduced-motion setting, and nothing is shown by colour alone.

## Development

```
npm install
npm run build
npm test
```

Press **F5** in Kiro or VS Code to launch a development window with the extension loaded. Run `npm run package` to build a `.vsix`.

## How it was built with Kiro

Repogotchi was built spec-first in [Kiro](https://kiro.dev). Everything Kiro worked from is in [`.kiro/`](.kiro):

```
.kiro/
├── steering/         product, tech stack, folder structure, the haunted UI guide, demo prep
├── specs/repogotchi/ requirements (EARS), design (with correctness properties), tasks
└── hooks/            tests on save, a test-writing buddy, a TODO guard, checks after each task,
                      and a check-up that prints Repogotchi's own health
```

<!-- Add highlights from KIRO_LOG.md here before submitting: what the spec drove, what the hooks caught, where vibe mode helped, and what had to be corrected. -->

## License

See [LICENSE](LICENSE).
