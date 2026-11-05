---
inclusion: always
foundationalType: tech
---

# Tech stack and conventions

## Platform

- A VS Code-compatible extension that runs in **Kiro** (a Code OSS fork) and in **VS Code**.
- `engines.vscode`: `^1.90.0`. Use only stable APIs. No proposed APIs.
- Distributed as a `.vsix` built with `@vscode/vsce`, and published to **Open VSX** (the registry Kiro uses) and optionally to the VS Code Marketplace.

## Language and build

- TypeScript in `strict` mode, `"target": "ES2022"`, `"module": "commonjs"` for the extension host.
- Bundle with **esbuild** into `dist/extension.js`. The webview script is bundled separately into `dist/webview.js`.
- Node built-ins are fine: `child_process.execFile`, `fs/promises`, `path`.
- **Runtime dependencies: zero if possible.** Prefer a small hand-written parser over a dependency. Any runtime dependency needs a one-line justification in `design.md`.

## Git access

- Shell out to the `git` CLI with `execFile` (never `exec`, never a shell string). Always pass `cwd` and a timeout (default 10s).
- Do not depend on `simple-git` or the built-in Git extension API. The CLI is more predictable across Kiro and VS Code.
- Handle "not a git repo", "git not installed", detached HEAD and repos with no commits gracefully. The pet shows a friendly empty state and never throws to the user.

## Testing

- **Vitest** for all pure logic in `src/core/` and parsers in `src/collectors/`. Target ≥ 90% line coverage on `src/core/`. The pet should be well fed.
- **fast-check** for property-based tests of the scoring and lifecycle rules (monotonicity, bounds, determinism).
- **@vscode/test-electron** for a small integration smoke suite: activation, command registration and webview render.
- Test fixtures live in `test/fixtures/`. Git fixture repos are created on the fly in a temp directory by a helper, never committed as nested `.git` folders.

## Commands

- `npm run build` bundles with esbuild
- `npm run watch` bundles in watch mode
- `npm run typecheck` runs `tsc --noEmit`
- `npm test` runs Vitest unit and property tests
- `npm run verify` runs `typecheck` then `test` (used by the verify-after-task hook; chaining lives in package.json so the hook works in any Windows shell)
- `npm run test:coverage` runs Vitest with `--coverage`, writing `coverage/lcov.info`. Repogotchi eats its own food.
- `npm run test:integration` runs the VS Code integration smoke tests
- `npm run lint` runs ESLint with `@typescript-eslint`
- `npm run package` runs `vsce package` and produces `repogotchi-<version>.vsix`

## Code style

- Pure functions in `src/core/`. **No `vscode` import anywhere under `src/core/`.** It must be testable in plain Node.
- Time is injected through a `Clock` interface. Never call `Date.now()` directly in core logic, because demo time travel depends on this.
- Prefer discriminated unions over boolean flags for state.
- No `any`. Use `unknown` and narrow it.
- User-facing strings live in one place (`src/ui/strings.ts`) so tone stays consistent.
