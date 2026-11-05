# Design Document

## Overview

Repogotchi is a VS Code-compatible extension with a strict three-layer split:

1. **Core** (`src/core/`): pure, deterministic functions that turn `Metrics` plus the previous `PetState` plus `now` into the next `PetState`. No I/O, no `vscode`.
2. **Collectors** (`src/collectors/`): read the repo through `git` and the filesystem, and read the Problems panel through a thin shell adapter. They return plain data. The parsers inside are pure and are tested with fixtures.
3. **Shell** (`src/services/`, `src/ui/`): scheduling, persistence, commands, the webview, editor decorations and the status bar.

All time flows through a `Clock`, which makes the life cycle testable and lets demo mode fast-forward.

The pet tracks five vitals and two modifiers:

| Signal | Vital or modifier | Update path |
|---|---|---|
| Test coverage | Fullness | full scan |
| TODO markers | Cleanliness | full scan |
| Stale branches | Spirit | full scan |
| Problems panel errors and warnings | Calm (shown as body temperature) | **fast path**, about 10 s after diagnostics settle |
| Commit habits | Fitness | full scan |
| Committed secrets | **Poison** (caps Health at 19) | full scan |
| Positive streaks | **Streak bonus** (+3 or +6), Flourishing mood, trophies | full scan |

```mermaid
flowchart LR
  subgraph Host["Kiro / VS Code extension host"]
    SCH[ScanScheduler] -->|"scan()"| PS[PetService]
    DW[DiagnosticsWatcher] -->|"applyDiagnostics()"| PS
    PS --> COL[Collectors]
    COL -->|"git execFile"| GIT[(git CLI)]
    COL -->|"fs read"| FS[(workspace files)]
    PS -->|"metrics, prev, now"| CORE[Core: scoring, secrets, streaks, lifecycle, reaper]
    CORE -->|"next PetState + effects"| PS
    PS <--> ST[Storage: workspaceState]
    PS -->|onDidChange| UI[PetViewProvider, StatusBar, ReaperGutter]
    UI <-->|"postMessage protocol"| WV[Webview: LCD spider + reaper scene]
    UI --> ACT[Actions]
    ACT -->|"git branch -d, after confirm"| GIT
  end
  CLK[Clock: System or Offset] --> PS
```

## Data model (`src/core/types.ts`)

```ts
export type Mood = 'flourishing' | 'thriving' | 'content' | 'grumpy' | 'sick' | 'dying';
export type Stage = 'egg' | 'baby' | 'child' | 'teen' | 'adult';
export type AdultForm = 'moonweaver' | 'dustcrawler' | 'gloomfang';
export type VitalKey = 'fullness' | 'cleanliness' | 'spirit' | 'calm' | 'fitness';
export type Cause = VitalKey | 'poison';
export type ReaperLevel = 0 | 1 | 2 | 3;
export type StreakKey = 'tested' | 'clear' | 'healthy';
export type TrophyKey = 'candle' | 'lanternGlow' | 'flameCrown' | 'batFamiliar';

export type CoverageMetric =
  | { kind: 'found'; format: 'lcov' | 'istanbul' | 'cobertura'; path: string; linePct: number; reportMtime: number }
  | { kind: 'missing' }
  | { kind: 'unparseable'; path: string; error: string };

export interface TodoMarker { path: string; line: number; token: string; text: string }
export interface TodoMetric { markers: TodoMarker[]; nonEmptyLines: number; filesScanned: number }

export interface SecretFinding { path: string; line: number; ruleId: string; ruleLabel: string; masked: string } // never the raw value
export interface SecretMetric { findings: SecretFinding[] }

export interface BranchInfo {
  name: string; isRemote: boolean; tipTime: number; author: string; subject: string;
  merged: boolean | 'unknown';
}
export interface BranchMetric { defaultBranch: string | null; current: string | null; stale: BranchInfo[]; total: number }

export interface CommitInfo {
  hash: string; time: number; subject: string;
  linesChanged: number;          // added + deleted, excluding excluded/lockfile paths
  touchesTest: boolean; touchesSource: boolean;
}
export interface CommitMetric { identity: string | null; mine: CommitInfo[] } // last 60 days, newest first

export interface DiagnosticsSummary {
  errors: number; warnings: number;
  byFile: Array<{ path: string; errors: number; warnings: number; firstErrorLine: number | null }>;
}

export interface Metrics {
  coverage: CoverageMetric;
  todos: TodoMetric;
  secrets: SecretMetric;
  branches: BranchMetric;
  commits: CommitMetric;
  diagnostics: DiagnosticsSummary;   // last settled value from the fast path
  lastCommitTime: number | null;
  scannedAt: number;
}

export interface Vitals { fullness: number; cleanliness: number; spirit: number; calm: number; fitness: number }
export interface Health { base: number; bonus: number; capped: boolean; value: number }

export interface StreakState { current: number; longest: number; lastQualifyingDay: string | null } // day = 'YYYY-MM-DD'

export type Life =
  | { status: 'alive'; stage: Stage; form?: AdultForm; criticalSince: number | null }
  | { status: 'dead'; diedAt: number; cause: Cause; epitaph: string };

export interface PetState {
  schemaVersion: 1;
  id: string; name: string; generation: number;
  bornAt: number; hatchedAt: number | null;
  life: Life;
  vitals: Vitals | null;           // null until first scan
  health: Health | null;
  mood: Mood | null;
  poison: { since: number; ruleIds: string[]; count: number } | null;
  streaks: Record<StreakKey, StreakState>;
  trophies: Array<{ key: TrophyKey; streak: StreakKey; at: number }>;
  dayLog: Array<{ day: string; minHealth: number; poisoned: boolean }>; // ≤ 60 days, for the Healthy streak
  history: Array<{ t: number; h: number }>;                             // ≤ 500 samples
  lastScanAt: number | null;
}

export interface GraveEntry {
  name: string; generation: number; form: Stage | AdultForm;
  bornAt: number; diedAt: number; cause: Cause; epitaph: string;
  trophies: TrophyKey[]; longestStreaks: Record<StreakKey, number>;
  history: Array<{ t: number; h: number }>;
}
```

## Scoring (`src/core/scoring.ts`)

| Vital | Formula | Defaults | Examples |
|---|---|---|---|
| Fullness | `clamp(round(linePct))` | missing report → 25 | unparseable counts as missing |
| Cleanliness | `round(100 · e^(−d / τ))`, `d = markers / (nonEmptyLines / 1000)` | τ = 4 | 2 per KLOC → 61, 4 → 37, 8 → 14. 0 lines → 100 |
| Spirit | `max(0, 100 − w · staleCount)` | w = 15 | 7 stale branches → 0 |
| Calm | `round(100 · e^(−load / k))`, `load = errors + 0.25 · warnings` | k = 5 | 0 → 100, 1 error → 82, 5 errors → 37, 8 warnings → 67 |
| Fitness | `round(100 · (0.4·regularity + 0.3·smallness + 0.3·quality))` | target 8 days, small ≤ 200 lines | 5/8 days, 80% small, 60% clear → 67 |

**Temperature** (display only): `36.6 + (100 − calm) / 100 × 4.4` °C. Calm 100 → 36.6 °C, 37 → 39.4 °C, 0 → 41.0 °C. Fahrenheit is `°C × 9/5 + 32`.

**Health** (Req 4.5):
1. `base = round(Σ wᵢ·vᵢ / Σ wᵢ)` with weights 0.25 / 0.20 / 0.15 / 0.20 / 0.20 (fullness, cleanliness, spirit, calm, fitness). Negative weights are rejected at settings load.
2. `bonus = poisoned ? 0 : min(6, 3 × |{s : streaks[s].current ≥ 7}|)`
3. `value = clamp(base + bonus)`, then if poisoned `value = min(value, 19)` and `capped = true`.

**Mood** (`mood.ts`): poisoned → `dying`. Otherwise: `≥80` → `flourishing` if any streak ≥ 7, else `thriving`. Then `60–79 content · 40–59 grumpy · 20–39 sick · <20 dying`.

**Top causes** (`causes.ts`, for Req 17.7): `cost(v) = round(wᵥ / Σw × (100 − vᵥ))` health points per vital. If poisoned, `poison` is first with cost `base − 19`. Return the top three where cost > 0, each with its care action (`feed`, `clean`, `walk`, `treatFever`, `exercise` (opens commit tips), `antidote`).

## Secrets (`src/core/secrets.ts`)

Rules are data: `{ id, label, pattern: RegExp, entropyMin?: number, group?: number }`. They are checked in order and the first match per position wins, so the specific rules come before the generic one.

| id | label | pattern (JS) |
|---|---|---|
| `aws-access-key-id` | AWS access key | `\b(?:AKIA\|ASIA)[0-9A-Z]{16}\b` |
| `github-token` | GitHub token | `\b(?:gh[pousr]_[A-Za-z0-9]{36,}\|github_pat_[A-Za-z0-9_]{50,})\b` |
| `gitlab-token` | GitLab token | `\bglpat-[A-Za-z0-9_-]{20,}\b` |
| `slack-token` | Slack token | `\bxox[abprs]-[A-Za-z0-9-]{10,}\b` |
| `stripe-live` | Stripe live key | `\b(?:sk\|rk)_live_[A-Za-z0-9]{20,}\b` |
| `google-api-key` | Google API key | `\bAIza[0-9A-Za-z_-]{35}\b` |
| `anthropic-key` | Anthropic API key | `\bsk-ant-[A-Za-z0-9_-]{20,}` |
| `openai-key` | OpenAI API key | `\bsk-(?:proj-)?[A-Za-z0-9_-]{20,}` |
| `private-key` | Private key | `-----BEGIN (?:[A-Z]+ )?PRIVATE KEY-----` |
| `generic-secret` | Hard-coded secret | `/\b(api[_-]?key\|secret\|token\|passw(?:or)?d\|client[_-]?secret)\b\s*[:=]\s*['"]([^'"\s]{16,})['"]/i`, group 2, entropy ≥ 3.5 |

- **Placeholders are ignored:** values containing `EXAMPLE`, `example`, `xxxx`, `XXXX`, `<`, `${`, `{{`, `process.env`, `your_`, `changeme` or `dummy`. This also skips AWS's documented `AKIAIOSFODNN7EXAMPLE`.
- **Custom rules:** `repogotchi.customSecretRules` holds `{id, label, pattern}` strings. Invalid regexes are dropped with an output-channel warning.
- **Entropy:** `shannon(s) = −Σ p(c)·log₂ p(c)` over the characters of `s`.
- **Masking happens at the source:** `scanSecrets(path, text, rules)` builds `masked = value.slice(0, 4) + '••••'` and drops the raw match before returning. No other function ever receives the raw value. The output channel logs only `ruleId` and `path:line`.
- **`.env` files:** decided by path alone from `git ls-files`, without reading the file.

## Commits, exercise and streaks

### Collector (`collectors/commits.ts`)
- Identity: `git config user.email`. If it is empty, `identity = null` and all authors count.
- `git log --all --no-merges --since=<now−60d ISO> --format=%x1e%H%x1f%at%x1f%ae%x1f%s --numstat`, with the `--since` date computed from the Clock.
- The pure `parseLogNumstat(stdout)` splits on `\x1e`. In numstat lines, `-\t-` (binary) counts as 0 lines.
- `touchesTest` uses `repogotchi.testGlobs` (default `**/*.test.*`, `**/*.spec.*`, `**/test/**`, `**/tests/**`, `**/__tests__/**`, `**/*_test.go`, `**/test_*.py`).
- `touchesSource` is true for a non-test, non-excluded file with a code extension from the built-in list: `ts tsx js jsx mjs cjs py go rs java kt cs rb php swift c cc cpp h hpp m scala ex exs dart vue svelte`.
- Lockfile and excluded paths don't count toward `linesChanged` (default `lockfileGlobs`: `**/package-lock.json`, `**/yarn.lock`, `**/pnpm-lock.yaml`, `**/Cargo.lock`, `**/poetry.lock`, `**/go.sum`, `**/Gemfile.lock`, `**/composer.lock`).

### Fitness (`core/exercise.ts`)
Uses commits from the last `exerciseWindowDays` (default 14). `activeDays` counts distinct local days, so `tzOffsetMinutes` is part of the config and the shell injects it. Message quality uses `isClearMessage(subject)` from Req 15.4. With no commits in the window, smallness and quality are both 0.5, so Fitness is 30.

### Streaks (`core/streaks.ts`)
`computeStreaks(commits, dayLog, today, cfg): Record<StreakKey, StreakState>`, pure.
- **Days** are built from the local calendar between the oldest commit (within 60 days) and today.
- **Tested and Clear:** walk days backward from today. An idle day (no commits) increases the gap counter. If the gap counter exceeds `streakGraceDays`, stop. An active day that qualifies adds 1 and resets the gap. An active day that doesn't qualify stops the walk.
- **Today's grace:** today does not break a streak until it ends. If today has no qualifying commit yet, the walk starts from yesterday.
- **Healthy:** walk the `dayLog` backward, counting days with `minHealth ≥ 80 && !poisoned`. Days with no scan count as idle, using the same grace rule.
- `longest` is the maximum of the stored and computed values.
- **Trophies** (`core/trophies.ts`): when any streak's `current` first reaches 3, 7, 14 or 30 within this pet's life, emit the `trophy` effect and append `candle`, `lanternGlow`, `flameCrown` or `batFamiliar`, at most once per trophy key per pet.
- **Evolution boost** (Req 16.6): if any Tested or Clear streak reached 7 during the Teen stage, add 5 to the average health before choosing the adult form.

## Life cycle (`src/core/lifecycle.ts`)

A single pure reducer:

```ts
export type LifeEvent =
  | { type: 'scan'; metrics: Metrics; now: number; resumed?: boolean }
  | { type: 'diagnostics'; summary: DiagnosticsSummary; now: number }  // fast path
  | { type: 'resurrect'; now: number; seed: string }
  | { type: 'rename'; name: string };

export function step(prev: PetState, ev: LifeEvent, cfg: LifeConfig): { next: PetState; effects: Effect[] };
```

`Effect` is a list of things the shell should do: `notifyMoodDrop`, `playReaction(kind)`, `died(GraveEntry)`, `hatched`, `evolved(form)`, `lastChance`, `poisoned(findingsSummary)`, `antidote`, `trophy(key)`, `reaperChanged(level)`. The reducer never performs side effects itself.

The `diagnostics` event recomputes only Calm, Health, mood and the critical timer. It reuses the last stored values for the other vitals, poison and streaks.

```mermaid
stateDiagram-v2
  [*] --> Egg
  Egg --> Baby: first successful scan
  Baby --> Child: age ≥ 1d
  Child --> Teen: age ≥ 3d
  Teen --> Adult: age ≥ 7d (form from avg health + streak boost)
  state Alive {
    Baby
    Child
    Teen
    Adult
  }
  Alive --> Critical: health < 20 or poisoned
  Critical --> Alive: health ≥ 20 and not poisoned (timer reset)
  Critical --> Dead: critical ≥ deathGraceHours
  Dead --> Egg: resurrect (health ≥ 60), generation + 1
```

`Critical` is not a separate stage. It is `criticalSince !== null`, so a pet keeps its stage and sprite while sick.

- **Resume rule (Req 6.7):** if `ev.resumed` and `prev.life.criticalSince !== null`, the reducer computes `elapsed = now − criticalSince` and, if `elapsed > grace − lastChance`, rewrites `criticalSince = now − (grace − lastChance)` and emits `lastChance`.
- **Poison transitions:** going from no findings to some emits `poisoned` and sets `poison.since`. Going from some to none emits `antidote` and sets `poison = null`.
- **Evolution:** the time-weighted average of `history` (each sample weighted by the gap to the next sample), plus the streak boost, picks the adult form at the Teen → Adult transition. The form is then fixed for that pet's life.
- **History downsampling:** when the history exceeds 500 entries, merge the oldest adjacent pairs (averaging `h`, keeping the earlier `t`) until it has 400. The `diagnostics` fast path appends a history sample at most every 5 minutes.
- **`dayLog`:** update today's `minHealth` and `poisoned` on every event that changes Health. Keep at most 60 days.

**Epitaph** (`epitaph.ts`): the cause is `poison` if poisoned at death. Otherwise it is the lowest vital, with ties broken in the order fullness → cleanliness → spirit → calm → fitness. Templates are picked deterministically from a seeded hash of the pet id:
- fullness: "Starved at {pct}% coverage." / "Waited for tests that never came."
- cleanliness: "Drowned in {n} TODOs." / "Buried under {n} FIXMEs and good intentions."
- spirit: "Haunted to death by {n} forgotten branches." / "Lost wandering {oldestBranch}."
- calm: "Burned up at {temp}° with {n} errors." / "The fever broke. So did everything else."
- fitness: "Withered on the couch: {active} active days in two weeks." / "Died waiting for a commit message longer than 'fix'."
- poison: "Poisoned by a committed {ruleLabel}." / "Swallowed a {ruleLabel} in {file}. Rotate your keys."

## The grim reaper (`src/core/reaper.ts` plus webview choreography)

### Level (pure)
```ts
reaperLevel(pet: PetState, now: number, cfg): ReaperLevel
```
- It is 0 for an egg or dead pet.
- Otherwise it is the max of: mood grumpy → 1, sick → 2, dying or poisoned → 3, and **drop** → 1, where drop means `max(h in history over the last 24h) − current ≥ reaperDropPoints`.
- The level is recomputed on every event. A change emits `reaperChanged`.

### Scene layout
The webview has two canvases:
1. **Scene layer:** fills the whole view at the same pixel scale as the LCD, **behind** the screen. It holds the reaper (a 24×38 logical-pixel sprite: hood, void face, red eyes, bony hand, scythe) and fog.
2. **LCD canvas:** 32×24 logical pixels at the top of the view, holding the spider, its web and thread. There is no device case around it.

The reaper slides in from the view's **side edge** (Req 17.8) and is clipped by it, so it looks as if it's leaning in from beyond the panel. Because it stands behind the LCD, only its hood, eyes and blade show above the screen's top-right corner until it looms. Its position sets the pet's gaze direction (`'left' | 'right'`).

### Choreography (webview `reaperDirector.ts`)
A small timeline engine: `Timeline = Array<{ at: ms; do: Cue }>`. The scheduler `nextAppearance(level, rng)` is pure, with an injectable RNG for tests.

| t (ms) | Level 1 (peek) | Level 2 (lurk) | Level 3 (loom) |
|---|---|---|---|
| 0 | Reaper slides in about 8 px (eyes and scythe tip), 600 ms ease-out | Slides in about 14 px (half body) | Slides in fully and stays |
| 800 | Pet eyes → reaper side, mouth → flat | Same | Same, mouth → wobble |
| 1800 | Pet eyes → front (at you) | Same | Pet ducks behind the tombstone prop. Only its eyes stay visible. |
| 2000 | Sweat drop appears; forehead tint fades in (500 ms) | Same, plus a 1 px tremble at 4 Hz | Sweat drop and full tint. The reaper raises its hourglass. |
| 6000 / 12000 | Reaper slides out (600 ms) | Same | Stays. Every ~8 s the pet's eyes glance at the reaper and back. |
| +3000 after exit | Tint fades out | Same | n/a |

- **Level change to 0:** the reaper plays a shrug (2 frames) and backs out. The pet plays relief: it wipes its brow, the tint clears, and it gives one bounce.
- **Hourglass:** sand level = `1 − critical elapsed / deathGraceHours`, drawn as 8 discrete steps.
- **Clicking the reaper:** it is a focusable `<button>` laid over its sprite bounds (`aria-label="Grim reaper. Why is it here?"`). Clicking posts `{type:'action', action:'whyReaper'}`.
- **Motion gating (Req 17.10):** under reduced motion or spookiness `off`, cues jump to their end frames and skip the slide, tremble and duck. The sweat and tint still appear.
- **Hidden view:** animations pause when `document.visibilityState === 'hidden'` (Req 8.6).

### Expression layers (webview `expressions.ts`)
Each creature's sprite data includes a **face anchor**: `{ eyeL, eyeR, mouth, forehead: {x, y, w, h}, hideOffsetY }`. Faces are composited at render time, so every creature gets every expression without separate sprite sheets:
- **Eyes:** `front`, `left`, `right`, `wide`, `closed`, `x` (dead).
- **Mouths:** `smile`, `flat`, `frown`, `wobble`, `open` (eat), `grit`.
- **Overlays:**

| Overlay | Look | Colour token |
|---|---|---|
| Fear tint | Vertical gradient over the forehead rect, alpha 0 → 0.65 | `--rg-ink-fear: #5B8CFF` |
| Sweat drop | 3×4 px, placed on the side away from the reaper; it drips once every 2 s | `--rg-ink-fear` |
| Fever | Red cheeks at ≥ 37.8 °C, plus a thermometer in the mouth at ≥ 38.5 °C | `--rg-ink-fever: #FF6B6B` |
| Poison | Bubbles rising, a skull-bottle icon in the LCD corner, purple pulse on the LCD border | `--rg-ink-poison: #C77DFF` |
| Flourishing | Ghost-flowers bloom along the LCD floor, glints around the pet | `--rg-ink-gold: #FFD166` |
| Trophies | Candle, glow, crown and bat drawn at anchor points | `--rg-ink-gold` |

### Editor gutter cameo (`ui/reaperGutter.ts`, Req 17.9)
- One `TextEditorDecorationType` with `gutterIconPath: media/reaper-gutter.svg` (16×16, original art) and `gutterIconSize: 'contain'`.
- It is placed on the line of the first Error in the active editor (from `DiagnosticsSummary.byFile`), or on `visibleRanges[0].start.line`.
- The `DecorationOptions.hoverMessage` reads "The reaper is watching. Health {h}. Main cause: {cause}."
- It refreshes on active editor change, visible range change (throttled to 250 ms) and reaper level change. All decorations are cleared when the level falls below the threshold.

## Collectors

### Git (`git.ts`)
`runGit(args: string[], cwd: string, timeoutMs = 10_000): Promise<Result<string, GitError>>`, where `GitError` is `'not-installed' | 'not-a-repo' | 'timeout' | { code: number; stderr: string }`. It uses `execFile('git', args, { cwd, timeout, maxBuffer: 16 MB })`.

### Branches (`branches.ts`)
- Default branch: `git symbolic-ref --quiet refs/remotes/origin/HEAD`, then `git config init.defaultBranch`, then probe `main` and `master`.
- List: `git for-each-ref --format=%(refname)%00%(committerdate:unix)%00%(authorname)%00%(subject) refs/heads refs/remotes`. Skip `*/HEAD`.
- Merged set: `git branch -a --merged <default> --format=%(refname)`.
- De-duplication: `refs/heads/foo` and `refs/remotes/origin/foo` collapse into one entry using the newer tip time.
- `parseForEachRef` and `classifyStale(branches, merged, now, days)` are pure and fixture-tested.

### Content scan (`contentScan.ts`): TODOs and secrets in one pass
- Files: `git ls-files -z --cached` (tracked plus staged), filtered by `excludeGlobs`, skipping files > 1 MB and binary files (a NUL byte in the first 8 KB).
- `.env`-style paths become secret findings straight from the file list (Req 14.3).
- Read files in batches of 50, with `await setImmediate()` between batches (Req 8.3). For each file, run the pure `scanText` (TODO markers) and `scanSecrets` (secret rules) on the same string, then release the text.
- The secret allowlist applies here: `repogotchi:allow-secret` on the line, or a path matching `secretAllowGlobs`.

### Coverage (`coverage/`)
- `index.ts` receives candidate paths from the shell (found through `vscode.workspace.findFiles` with the configured globs) and picks the newest by `mtime`.
- `lcov.ts`: ΣLH/ΣLF. If ΣLF = 0, coverage is 0.
- `istanbulSummary.ts`: reads `total.lines.pct`, validated as a number.
- `cobertura.ts`: a regex-extract of `line-rate="…"` on the root `<coverage>` element. No XML dependency.

### Diagnostics (`collectors/diagnostics.ts` pure, `services/diagnosticsWatcher.ts` shell)
- The pure `summarizeDiagnostics(entries: Array<{ path; severity: 'error' | 'warning' | 'info' | 'hint'; line }>, repoRoot, excludes)` keeps only paths under the repo root that aren't excluded.
- The watcher subscribes to `languages.onDidChangeDiagnostics`, restarts a settle timer (`feverSettleSeconds`) on every change, then maps `getDiagnostics()` into plain entries and calls `petService.applyDiagnostics(summary)`.
- Language servers usually report diagnostics only for open files, so Calm reflects the code being worked on. This is intended, and the README explains it.

## Services

### PetService
Holds the current `PetState`, the graveyard, the latest `Metrics` and the `Clock`.
- **`scan()`:**
  1. Collect coverage, content (TODOs and secrets), branches and commits in parallel. Each has its own error boundary that produces a degraded metric rather than throwing.
  2. Attach the latest diagnostics summary.
  3. Run `step(prev, {type:'scan', …})`.
  4. Persist, then fire `onDidChange(state, effects, metrics, reaperLevel)`.
- **`applyDiagnostics(summary)`:** runs `step` with a `diagnostics` event and fires `onDidChange`. Persistence is throttled to once a minute (Req 13.5).
- The latest TODO markers and secret findings (masked) are kept in memory for the Clean and Antidote quick picks and are not persisted.

### ScanScheduler
A single-flight guard: `running: Promise | null` plus `pending: boolean` (Req 8.2). Sources: activation, interval, `onDidSaveTextDocument` (30 s debounce), and a `FileSystemWatcher` on `.git/HEAD`, `.git/index` and `.git/refs/**` (5 s debounce). Watching `.git/index` catches staged secrets. It disposes everything on deactivate.

### Storage
Key: `repogotchi.v1.<sha1(repoRoot)>`. It stores `{ pet, graveyard, clockOffsetMs }`. `migrate(raw): PetState | null` is pure. On failure it copies the raw value to `…backup.<ts>` and starts a fresh egg (Req 10.2).

### Clock
```ts
interface Clock { now(): number }
class SystemClock implements Clock { now() { return Date.now(); } }
class OffsetClock implements Clock { constructor(private base: Clock, public offsetMs = 0) {} now() { return this.base.now() + this.offsetMs; } }
```
`OffsetClock` is always used, with offset 0 by default, so demo mode is just a setter.

## UI

### Webview protocol (`src/ui/protocol.ts`)
```ts
type HostToView =
  | { type: 'state'; pet: PetState; mood: Mood | null; health: Health | null; reaper: ReaperLevel;
      reaperSide: 'left' | 'right'; criticalRemaining: number | null;   // 0..1 for the hourglass
      temperature: { value: number; unit: 'C' | 'F' } | null;
      fitnessParts: { activeDays: number; targetDays: number; smallPct: number; clearPct: number } | null;
      counts: { markers: number; stale: number; coveragePct: number | null; staleFood: boolean;
                errors: number; warnings: number; secrets: number };
      causes: Array<{ cause: Cause; cost: number; action: CareAction }>;
      hints: string[]; demoOffsetHours: number;
      settings: { spookiness: 'off' | 'mild' | 'full' } }
  | { type: 'reaction'; kind: 'eat' | 'tidy' | 'cheer' | 'hatch' | 'evolve' | 'die' | 'resurrect'
        | 'poisoned' | 'antidote' | 'trophy' | 'relief' | 'feverUp' | 'feverDown'; detail?: string }
  | { type: 'demoReaper'; sequence: ReaperLevel[] }
  | { type: 'empty'; reason: 'no-repo' | 'no-git' };
type CareAction = 'feed' | 'clean' | 'walk' | 'treatFever' | 'exercise' | 'antidote';
type ViewToHost =
  | { type: 'ready' }
  | { type: 'action'; action: CareAction | 'rename' | 'resurrect' | 'graveyard' | 'scan' | 'whyReaper' };
```
Both sides validate incoming messages with a small type guard. Unknown messages are ignored.

### Rendering
- The LCD is 32×24 logical pixels, drawn at 6 px per pixel and scaled to the view width. The scene layer is full-view at the same scale. Sprites are `string[]` bitmaps (`'.'` off, `'#'` on, `'+'` half-tone) in `webview/sprites.ts`. Accent inks (fear, fever, poison, gold) are used only for overlays.
- An animation loop with `requestAnimationFrame` at a fixed 4 fps sprite tick (90s feel). The reaper's slide uses 30 fps interpolation for smoothness. Effects are gated by spookiness and `matchMedia('(prefers-reduced-motion: reduce)')`.
- **Compact by default (Req 9.9).** Below the LCD there are only two things. The first is a health line, `Health {h} · {comment}`, with health in the mood colour. The second is two dropdown buttons, `Stats ▾` and `Actions ▾`. The panel is about 380 px tall.
- **Pop-ups.** Each dropdown opens a dialog laid over the LCD (`role="dialog"`, focus moves into it, Escape or × closes it and returns focus). Only one is open at a time. The **Stats** pop-up holds a context note (for example "Main cause: 6 errors (costing 13)") and the five bars FOOD, WEB, GHOSTS, TEMP and FIT. The **Actions** pop-up (Req 9.10) is a 2×2 grid: Run tests, TODOs · N, Branches · N, Errors · N, with the top-cost cause's action highlighted, using `causes.ts`. When poisoned, Remove secret replaces Errors. When dead, there's one Resurrect button. A footer says health changes only when the code does. The open pop-up is stored in `workspaceState` under `repogotchi.ui.panel` (`'stats' | 'actions' | null`).
- CSP: `default-src 'none'; img-src ${cspSource} data:; style-src ${cspSource}; script-src 'nonce-${nonce}';`.

### Status bar
The icon is a codicon per state: `$(beaker)` poisoned, `$(sparkle)` flourishing, `$(flame)` thriving or content, `$(warning)` grumpy, `$(pulse)` sick, `$(error)` dying, `$(circle-slash)` dead. The text is `{name} {health}`, with a tooltip listing all five vitals, the temperature and the active streaks.

### Actions
- **Feed:** look up `repogotchi.testCommand`. If set, run it as a `ShellExecution` task. Otherwise use `vscode.tasks.fetchTasks()` and take the task with `group === TaskGroup.Test && isDefault`, or show a picker. On `onDidEndTaskProcess`, rescan.
- **Clean:** `QuickPick` items `{ label: path:line, description: token, detail: text }` with separators per file.
- **Walk:** `QuickPick` of stale branches. Selecting one shows "Checkout", "Delete local branch…" and "Copy name". Delete goes through `showWarningMessage(..., { modal: true }, 'Delete')` and then `git branch -d`.
- **Treat fever:** `workbench.actions.view.problems`, then a quick pick of files sorted by error count that opens the first error.
- **Exercise:** an information panel with the current fitness parts, the five most recent "unclear" commit subjects, and the clear-message rule. Read-only: no rewriting history.
- **Antidote:** a quick pick of secret findings (`ruleLabel · path:line · masked`) that opens the location. The panel footer reads "Removing it from the file is not enough: it's in git history. Revoke or rotate the credential now."
- **Why reaper:** a small webview panel or quick pick with the top three causes and their action buttons.

## Configuration (`package.json` → `contributes.configuration`)

| Setting | Type | Default |
|---|---|---|
| `repogotchi.coverageFiles` | string[] | see Req 1.1 |
| `repogotchi.noCoverageFullness` | number 0–100 | 25 |
| `repogotchi.todoPatterns` | string[] | `["TODO","FIXME","HACK","XXX"]` |
| `repogotchi.todoTolerance` | number > 0 | 4 |
| `repogotchi.excludeGlobs` | string[] | see Req 2.6 |
| `repogotchi.staleBranchDays` | number ≥ 1 | 30 |
| `repogotchi.ghostWeight` | number ≥ 0 | 15 |
| `repogotchi.weights` | `{fullness, cleanliness, spirit, calm, fitness}` | `0.25 / 0.20 / 0.15 / 0.20 / 0.20` |
| `repogotchi.warningWeight` | number ≥ 0 | 0.25 |
| `repogotchi.feverTolerance` | number > 0 | 5 |
| `repogotchi.feverSettleSeconds` | number ≥ 0 | 10 |
| `repogotchi.temperatureUnit` | `C` \| `F` | `C` |
| `repogotchi.secretAllowGlobs` | string[] | `["**/test/fixtures/**"]` |
| `repogotchi.customSecretRules` | `{id,label,pattern}[]` | `[]` |
| `repogotchi.testGlobs` | string[] | see Commits section |
| `repogotchi.lockfileGlobs` | string[] | see Commits section |
| `repogotchi.exerciseWindowDays` | number 7–60 | 14 |
| `repogotchi.exerciseTargetDays` | number ≥ 1 | 8 |
| `repogotchi.smallCommitLines` | number ≥ 1 | 200 |
| `repogotchi.streakGraceDays` | number 0–7 | 2 |
| `repogotchi.reaperDropPoints` | number ≥ 1 | 10 |
| `repogotchi.reaperSide` | `auto` \| `left` \| `right` | `auto` |
| `repogotchi.reaperInEditor` | `off` \| `slipping` \| `dying` | `dying` |
| `repogotchi.deathGraceHours` | number ≥ 1 | 72 |
| `repogotchi.lastChanceHours` | number ≥ 1 | 24 |
| `repogotchi.scanIntervalMinutes` | number ≥ 1 | 15 |
| `repogotchi.testCommand` | string | `""` |
| `repogotchi.spookiness` | `off` \| `mild` \| `full` | `mild` |

**Commands:** `repogotchi.showPet`, `repogotchi.scan`, `.feed`, `.clean`, `.walk`, `.treatFever`, `.exercise`, `.antidote`, `.whyReaper`, `.rename`, `.resurrect`, `.showGraveyard`, `.exportPet`, `.timeTravel`, `.resetTime`, `.summonReaper` (demo).

## Error handling

| Situation | Behaviour |
|---|---|
| git not installed | Empty state "Repogotchi needs git on PATH." No pet is created. |
| Not a repo | Empty state (Req 8.4). |
| A collector throws or times out | That vital keeps its previous value and the UI shows a small "?" with a tooltip. Logged to the output channel. |
| A secret scan error on one file | Skip the file and log `path` only. Never silently clear existing poison because of an error. |
| A custom secret rule is an invalid regex | Drop that rule and warn once. |
| `user.email` is unset | Count all authors and show a one-time hint. |
| Corrupt stored state | Back it up and start a fresh egg (Req 10.2). |
| Webview disposed mid-message | Ignore. State is re-sent on `ready`. |

## Testing strategy

- **Unit (Vitest):** every parser against fixtures; scoring tables (the examples in the scoring tables become test cases); each secret rule with one positive, one placeholder and one near-miss case; `isClearMessage` cases; epitaphs; storage migration.
- **Property-based (fast-check):** the correctness properties below.
- **Git integration (Vitest + temp repos):** `test/helpers/makeRepo.ts` creates repos with backdated commits and branches (`GIT_COMMITTER_DATE`, `GIT_AUTHOR_DATE`) and a configurable `user.email`. Tests assert on branch classification, commit parsing, streak days and staged-secret detection.
- **Webview logic (Vitest + jsdom):** `reaperDirector` timelines with fake timers and a seeded RNG; motion gating collapses cues to end frames; expression compositing puts eyes at the anchors.
- **Extension smoke (@vscode/test-electron):** activates, registers every command, renders the webview without CSP errors, shows the empty state in a non-git folder, and updates Calm after injecting a diagnostic through a `DiagnosticCollection`.
- **Dogfooding:** `npm run test:coverage` writes `coverage/lcov.info`, so Repogotchi feeds on its own repo. Secret-rule fixtures sit under `test/fixtures/`, which is allowlisted, so Repogotchi doesn't poison itself.

### Correctness properties

1. **Bounds:** for any metrics and config, every vital and Health is an integer in [0, 100].
2. **Monotonicity:** raising coverage never lowers Fullness or Health. Adding a marker never raises Cleanliness. Adding a stale branch never raises Spirit. Adding an error never raises Calm.
3. **Determinism:** `step(prev, ev, cfg)` returns deep-equal results for equal inputs.
4. **No resurrection by accident:** a dead pet only returns to `alive` through a `resurrect` event when Health ≥ 60.
5. **Death requires sustained illness:** for any sequence of events, a pet dies only if Health was < 20 (or it was poisoned) at every event in the last `grace` window.
6. **Stage order:** stages never go backwards within one generation.
7. **History bound:** `history.length ≤ 500` and `dayLog.length ≤ 60` after any event.
8. **Clock consistency:** shifting `now` and all timestamps by a whole number of days does not change stage, mood, staleness or streaks.
9. **Poison dominates:** if `secrets.findings` is non-empty, Health ≤ 19, mood is `dying`, bonus is 0 and the reaper level is 3, whatever the other inputs.
10. **Secrets never retained:** for any text containing a value matched by a rule, no `SecretFinding`, `PetState`, effect or log line contains more than the first 4 characters of that value.
11. **Bonus bounds:** the bonus is in {0, 3, 6}, and Health with the bonus is ≤ 100.
12. **Reaper monotonicity:** with the history fixed, lowering current Health never lowers the reaper level. The level is 0 whenever Health ≥ 60, there's no 24h drop, and the pet is not poisoned.
13. **Streak grace:** inserting up to `streakGraceDays` idle days between qualifying days doesn't change the count. Inserting a non-qualifying active day resets it.
14. **Fitness monotonicity:** adding a small commit with a clear message on a new day in the window never lowers Fitness.

## Dependencies

| Package | Kind | Why |
|---|---|---|
| `esbuild` | dev | bundling |
| `typescript`, `@types/vscode`, `@types/node` | dev | types |
| `vitest`, `@vitest/coverage-v8`, `fast-check`, `jsdom` | dev | tests, coverage, properties, webview logic |
| `@vscode/test-electron`, `@vscode/vsce` | dev | integration tests, packaging |
| `eslint`, `@typescript-eslint/*` | dev | linting |
| _(none)_ | runtime | the glob matcher, parsers and secret rules are hand-written |
