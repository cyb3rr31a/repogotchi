---
inclusion: always
foundationalType: product
---

# Product: Repogotchi

Repogotchi is a virtual pet that lives inside the editor (Kiro or VS Code) and **enforces good engineering habits**. If you slack off, it sickens and dies. If you do the right thing, it flourishes. It is a resurrection of the 1996 handheld virtual-pet craze, rebuilt for code instead of pixels, with a haunted, after-dark look. The pet is a **retro pixel-art spider** on a bare green LCD screen (no toy case), docked at the bottom of the left sidebar under the file tree.

## The one-line pitch

Virtual pets died out in the 90s. Your neglected codebase is dying right now. Repogotchi makes that visible, sends the grim reaper when you start slipping, and gives you a reason to care.

## What feeds the pet

| Signal | Pet vital | Meaning |
|---|---|---|
| Test coverage % | **Fullness** ("Food") | Tests catch bugs, and bugs are the spider's food. No coverage report means an empty web. |
| TODO / FIXME / HACK / XXX density | **Cleanliness** ("Web") | Open TODOs are holes torn in the spider's web. |
| Stale unmerged branches | **Spirit** ("Ghosts") | Abandoned branches are restless ghosts that scare the spider. |
| Errors and warnings in the Problems panel | **Calm**, shown as a **fever** temperature | Broken code makes the pet feverish within seconds. |
| Small, regular commits with clear messages | **Fitness** | Good version-control habits are the pet's exercise. |

Two modifiers sit on top:

- **Poison:** a committed secret (API key, token, private key, tracked `.env`) poisons the pet instantly. Health is capped at 19 whatever the other vitals say, and the death clock starts until the secret is gone. The lesson: never commit credentials, and rotate any you did.
- **Streaks:** seven days of commits that include tests, seven days of clear commit messages, or seven healthy days earn a health bonus, a **Flourishing** mood and permanent cosmetic trophies. Doing the right thing is rewarded, not just slacking punished.

The vitals combine into **Health**, and Health sets the pet's mood. When you're slipping, the **grim reaper** peeks in from the side of the pet's panel. The pet glances at it, looks back at you, sweats, goes blue at the forehead, and at worst hides. A pet left critically ill for too long dies and becomes a ghost. You can bring it back with a resurrection ritual once the repo is healthy again.

## Hackathon category

Primary: **Resurrection** (bringing back a dead technology, the 90s virtual pet, to solve a modern problem: neglected code health and bad habits). The haunted UI and reaper should also be strong enough to look like a Costume Contest entry.

## Product principles

1. **Real signals only.** Every number the pet shows comes from the actual repo or editor. Never fake or randomize a vital. Demo commands are clearly labelled "(Demo)".
2. **Every care action is a real dev action.** Feeding runs tests. Cleaning takes you to a TODO. Walking reviews stale branches. Treating fever opens the Problems panel. The antidote takes you to the leaked secret. The pet is a UI for doing good work, not a toy beside it.
3. **Never destructive without consent.** The extension never deletes branches, edits code, rewrites history or commits on its own. Destructive actions need an explicit modal confirmation and only ever affect local branches.
4. **Secrets are radioactive.** A detected secret is never stored, logged or displayed beyond a 4-character masked preview.
5. **Works on any repo, any language.** No language-specific tooling is required beyond git and whatever language servers the editor already runs. Coverage is read from standard report formats.
6. **Hard to game, easy to tune.** Prefer trends and streaks over single numbers, cap each signal's influence, and expose every threshold and weight as a setting.
7. **Fast and quiet.** A scan must not freeze the editor. No network calls. No telemetry.
8. **Spooky but accessible.** Effects respect reduced-motion settings and a "spookiness" setting. Information is never carried by colour or animation alone.

## Out of scope (for the hackathon)

- Multi-repo or team-shared pets
- Cloud sync, accounts or leaderboards
- Remote branch deletion
- Code review metrics (they need a network API)
- Scanning git history for old secrets (only the current tracked and staged files count)
- Computing coverage ourselves (we only read reports)
- The original toy's name or characters. All creatures and the reaper are original designs (see spooky-ui steering).
