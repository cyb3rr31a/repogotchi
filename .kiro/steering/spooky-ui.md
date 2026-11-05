---
inclusion: fileMatch
fileMatchPattern: "{webview/**,src/ui/**,media/**}"
---

# Haunted UI direction

The pet is a **retro pixel-art spider** living on a small green **LCD screen**, docked at the **bottom of the left sidebar**, under the file tree. There is **no toy case**: no egg-shaped shell and no physical A/B/C buttons, just the screen, the stats and real buttons. Polish matters: this UI is what judges remember.

A visual reference exists in the "Repogotchi Spider Mockup" design canvas (editor view, every mood, life cycle). Match it.

## Placement

- Contribute the view to the **Explorer** container (`contributes.views.explorer`, id `repogotchi.pet`, name "Repogotchi", type `webview`). It then stacks in the left sidebar **below the file tree**, like Outline and Timeline.
- VS Code and Kiro let users drag views. Don't fight it. Provide `Repogotchi: Show Pet` to reveal and focus the view.
- The screen sits at the top of the view, with stats and actions below it. The view's **right edge faces the editor**, so that's where the reaper leans in from (`repogotchi.reaperSide: auto`).

## Look

- **LCD:** a 32×24 logical pixel grid at 6 px per pixel (192×144), scaled up to fill the view width (about 268 px wide) with `image-rendering: pixelated`. Two-tone ghoulish green-on-black (`#9BFF9B` on `#0B140B`) with a dim tone `#3F6B3F` for webs, threads and ghosts, a faint scanline overlay, and a soft phosphor glow (`filter: drop-shadow`). A 1 px dark-green border and a soft drop shadow are the only frame.
- **Status inks:** the only other colours allowed on the LCD, and only for overlays: fear blue `#5B8CFF`, fever red `#FF6B6B`, poison purple `#C77DFF`, reward gold `#FFD166`. Define them as CSS custom properties (`--rg-ink-*`).
- **Scene:** a full-view layer behind the LCD holds fog and the grim reaper. The reaper stands behind the screen, so its hood, eyes and scythe show above the screen's top-right corner and at the view's right edge.
- **Home:** a corner web (top-left), a silk thread the spider hangs from, and a dotted floor line. When things are good, a caught bug sits in the web.
- **Theme awareness:** the panel sits on `var(--vscode-sideBar-background)`. The LCD keeps its own palette in every theme. All text outside the LCD uses VS Code theme tokens so it's legible in light, dark and high-contrast themes.
- **Fonts:** a pixel font drawn from bitmaps in `sprites.ts` for the LCD. Outside the LCD use the editor's UI font. Don't load web fonts; the webview CSP blocks remote resources.

## The spider (original designs; do not imitate any existing toy's characters)

| Stage | Creature |
|---|---|
| Egg | **Egg sac**: a silk ball with a keyhole slot, hanging from a thread |
| Baby (< 1 day) | **Speck**, a tiny spiderling with four stubby legs |
| Child (1–3 days) | **Skitter**, a small round spider with three eyes |
| Teen (3–7 days) | **Weaver**: round body, eight bent legs, two big eyes, a short fang row |
| Adult, great care (avg ≥ 75) | **Moonweaver**: Weaver with a gold crown of candle flames and a gold crescent moon on its abdomen |
| Adult, average care (45–74) | **Dustcrawler**: dusty speckles, half-lidded eyes |
| Adult, poor care (< 45) | **Gloomfang**: one red eye (the other sealed), torn legs, a dark grin with two fangs |
| Dead | **Ghost**: the spider's body without legs, drawn in the dim tone, a wavy tail, X eyes, an "O" mouth, and a small gravestone beside it |

Each creature needs at least two idle frames (a gentle bob on its thread) plus frames for **eat** (a bug drops into the web), **sick** and **sleep**. Each also needs a **face anchor**: eye sockets, mouth, forehead rows, and how far it sinks when hiding. Faces are composited from shared eye and mouth sprites, so every creature can glance sideways, look straight at you, sweat and go blue without separate sprite sheets.

### What the vitals look like for a spider

| Vital | Spider meaning | On screen |
|---|---|---|
| Fullness (coverage) | **Food.** Tests catch bugs. | Bugs caught in the web. An empty web when there's no coverage report. |
| Cleanliness (TODOs) | **Web holes.** Unfinished work tears the web. | Web strands go missing as TODO density rises. |
| Spirit (stale branches) | **Ghosts** | Tiny ghosts at the LCD edge, one per stale branch, max 5 shown, "+N" beyond that |
| Calm (errors) | **Fever** | Temperature page, red cheeks, thermometer |
| Fitness (commits) | **Fit** | A stat page with 14 day-ticks and the small-commit and clear-message percentages |

The stat labels under the screen read **FOOD · WEB · GHOSTS · TEMP · FIT**.

## The grim reaper

The reaper is the star of the show. It's an **original** pixel figure: a tall hooded silhouette with a void face, two pinprick red eyes, one bony hand, a wooden scythe handle, and a pale blade. It's about 24×38 logical pixels, with a dark cloak and a soft green rim light so it reads on a dark sidebar. It's a generic folklore figure; do not copy any specific game or film design.

| Reaper level | When | What happens |
|---|---|---|
| 1. Peek | Grumpy, or Health fell 10+ points in 24h | The blade, the handle and one red eye ease in from the right edge for ~6 s, about every 4 minutes. |
| 2. Lurk | Sick | More of the hood and both eyes lean in for ~12 s, about every 90 s. It drums its bony fingers. |
| 3. Loom | Dying or Poisoned | The whole hood rises behind the screen and stays, with an hourglass held up beside it. The sand level is the real time left before death. |

**The spider's reaction is the emotional beat. Get the timing right:**
1. The reaper slides in.
2. About 0.8 s later, the spider's pupils snap toward it and its mouth goes flat.
3. It holds for a beat (~1 s).
4. The pupils come back to **face the developer**. This is the "are you seeing this?" moment.
5. A sweat drop appears on the side away from the reaper, and a **blue tint creeps down the forehead**.
6. At level 2 the spider trembles on its thread. At level 3 it ducks behind a tombstone with only its eyes and leg tips peeking over, glancing at the reaper every few seconds.
7. When things improve, the reaper shrugs and backs away, and the spider wipes its brow with a leg and bounces once on its thread.

The reaper is clickable and opens "Why is the reaper here?" with the top causes and their fix buttons. At levels 2–3 a tiny reaper can also peek from the **editor gutter** beside the first error line (setting `repogotchi.reaperInEditor`).

## Fever, poison and flourishing

- **Fever:** the TEMP stat shows the real temperature (36.6 °C is healthy). Red cheeks appear from 37.8 °C, and from 38.5 °C a thermometer sits in the spider's mouth and red heat lines rise above its head.
- **Poison:** the spider's belly is mottled purple, its eyes go to queasy slits, bubbles rise around it, a skull-bottle icon sits in the LCD's top-right corner, and the LCD border glows purple. When the poison clears, a little vial pops in ("Antidote!") followed by a reminder to rotate the key.
- **Flourishing:** ghost-flowers bloom along the LCD floor, gold sparkles drift around, and the spider does a happy spin on its thread every so often.
- **Trophies** stay on the spider for life: a **Candle** (3-day streak), a **Lantern Glow** lantern hanging on its own thread (7), a **Flame Crown** (14) and a **Bat Familiar** (30) that flutters across the screen.

## Mood presentation

| Mood | Visual treatment |
|---|---|
| Flourishing | Everything in Thriving, plus ghost-flowers, gold sparkles, the lantern and the occasional happy spin |
| Thriving | Gentle bob, smile, full web with a caught bug |
| Content | Normal idle, smile, web intact |
| Grumpy | Flat mouth, slower bob, a few web strands missing |
| Sick | Wobbly mouth, screen flickers occasionally |
| Dying | Hiding behind the tombstone, cracks across the LCD glass, fog at the base, heartbeat sound off by default |
| Dead | Static noise burst, then the ghost spider drifts beside a gravestone. Fog rolls along the floor. |

## Keep the panel small

Under the screen there are only two things:

1. **The health line**, which always reads health first and then the comment: `Health 73 · Webster is spinning happily.` The word "Health" and the number use the mood colour. The comment says what's going on in plain words ("Webster has a 39.4° fever.", "Poisoned by an AWS key in src/config.ts.").
2. **Two dropdown buttons, side by side: `Stats ▾` and `Actions ▾`.** Each opens a pop-up laid over the screen. Only one is open at a time, and × or Escape closes it. Clicking the screen itself also opens Stats. Remember which one was open per workspace.

- **Stats pop-up:** a context note ("Main cause: 6 errors (costing 13)") and the five bars FOOD · WEB · GHOSTS · TEMP · FIT with values.
- **Actions pop-up:** a 2×2 grid of **plain-label** buttons, with live counts: **Run tests**, **TODOs · 21**, **Branches · 4**, **Errors · 6**. The action that would help most right now is highlighted in green, and the Actions toggle itself turns green to hint there's a recommended fix. When poisoned, the first button becomes **Remove secret**, in purple, and the Actions toggle is purple. When dead, there's a single dimmed **Resurrect (needs health 60)** button. A footer line reads: "These open the work. Webster's health only changes when your code does."

**The buttons don't change health. Only the code does.** Each button opens the real work: Run tests runs the test task, TODOs lists the markers, Branches lists stale branches, Errors opens the Problems panel, Remove secret opens the leaked line. Health moves only when the next scan sees the repo actually change. Don't use pet verbs like "Feed" or "Clean" on buttons, because people read them as "this makes it healthier". Pet language belongs in the comment line and the epitaphs.

## Accessibility rules (non-negotiable)

- Respect `prefers-reduced-motion` **and** the `repogotchi.spookiness` setting (`off` | `mild` | `full`, default `mild`). `off` removes flicker, noise and fog. Under either, the reaper appears as a still frame at the edge with no sliding, trembling or hiding motion, but the sweat drop and blue tint still show, because they carry information.
- Fear, fever and poison are never shown by colour alone. Each also has a shape (sweat drop, thermometer, skull bottle) and text in the aria-live region.
- No flashing faster than 3 per second.
- Every vital also shows as text (`FOOD 72%`) in the stats pop-up, and the Health number is always visible. The LCD has `role="img"` and a live `aria-label` describing the spider's state.
- Action buttons are at least 44 px tall.
- Sound is off by default. If added, it is opt-in via setting.

## Copy tone

Darkly playful, never mean to the developer. The spider is dramatic; you are its caretaker.
Examples: "Webster is starving. 12% coverage won't catch a single bug.", "Three restless branches whisper in the dark.", "Here lies Webster. Drowned in 47 TODOs.", "39.4° fever. 6 errors in the Problems panel.", "Something tall is standing just out of view…", "POISONED: AWS key in src/config.ts:12", "7-day tested streak. Webster is glowing."
