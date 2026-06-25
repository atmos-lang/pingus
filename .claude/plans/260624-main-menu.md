# Plan: Main Menu

Port the Pingus main menu (`PingusMenu`) to Atmos.
Code lives in the repo root (`.`).

## Status

- [x] designate feature: main menu
- [x] execute locally (offscreen) + screenshot
- [x] trace behavior
- [x] specify feature (this plan)
- [x] compare C++ and Céu versions + identify patterns
- [x] update plan
- [x] propose in Atmos
- [x] implement (initial: buttons + print + exit)

## Implementation (initial)

Files in repo root:

| file       | content                                            |
|------------|----------------------------------------------------|
| `menu.atm` | `val task Button` + anon `task ()` returned by mod |
| `main.atm` | window/dim setup; `val menu = require "menu"`       |

Notes:

- Syntax is Atmos v0.7 (`[...]`, `task`, `par :any`, `loop on`,
  `Nms`, `until`, `pico.set`).
- The installed binary self-reports `v0.6` but parses v0.7
  (hardcoded version string).
- Run offscreen:
    - `xvfb-run -a -s "-screen 0 800x600x24" atmos main.atm`
- Button visibility: stroke box + filled text (both white).

Architecture (working):

- `Button (id, text, x, y)` builds its rect (`w,h` fixed inside),
  uses `watching <click> { draw }`, returns its id.
- Menu `task ()` spawns 5 buttons in a pool and returns the
  clicked id via `await :any buttons` (no loop inside the menu).
- `main.atm` owns the loop: `val id = await Menu()`, then
  `until(id == :Exit)` (the CPS seam). No `pico.quit()` needed:
  when the main chunk ends the program task dies and the env
  shuts down (`run.lua` M.loop).

Verified (clicks via `xdotool`):

- Story / Options / Exit clicks each return the right id.
- Exit terminates the app cleanly.

Caveat learned: escape statements use call syntax
(`break()`, `return(x)`, `escape(:t,v)`); recorded in the
atmos skill.

Background (done):

- Inlined in `menu.atm`: `LAYERS` table (records with
  `png=DIR++...`, `y`, `speed`), `Layer (l)` task, spawned into
  a `layers` pool before the buttons (drawn behind).
- `!`/`N` (top-center) draw; `x` in `[0,W)`, two copies at
  `x ± W/2` for seamless horizontal wrap (W=800).
- Table mirrors the C++/Céu source: top edge in px
  (`0,150,200,429,500`), speed in px/s (`12..200`). Top-left
  anchor makes top/bottom coverage automatic (`layer1` at 0,
  full-width `layer4` 429+171=600 reaches bottom). Chosen over
  `%`/center, which forced height-dependent y values.
- Buttons stay `%` (position-only, resolution-independent).

Logo (done):

- `LOGO = "data/images/core/misc/logo.png"` (copied from
  pingus.cpp); drawn `['%', x=0.5, y=0.2]` (center anchor),
  spawned after layers, before buttons.

Footer (done):

- `FOOTER` table (`infos` 4-line copyright + `help` string).
- pico text is single-line (`TTF_RenderText_Blended`), so the
  copyright is drawn line-by-line (`loop i, txt in FOOTER.infos`).
- black help bar (filled rect, bottom) + centered help text.
- buttons pool scoped in a `do { ... await :any buttons }` block
  (the do is the task's last expr → returns the clicked id).

Buttons (done — image-based):

- `menuitem.png` background + label; hit-box from
  `pico.get.image("%", img)`.
- Hover: `par` tracks `mouse.motion until [!]vs.pos.rect`,
  toggles `hover`, plays `tick` on enter, overlays
  `menuitem_highlight.png` while hovered.
- Assets copied: `menuitem.png`, `menuitem_highlight.png`,
  `sounds/tick.wav`.
- Caveat: `pico.output.draw.text` asserts `rect.h != 0` — the
  label rect must set `h` (font size), e.g. `h=0.05`.

Layer approach (explored, rejected):

- Tried making the menuitem a pico layer (`draw.layer` + `vs`).
- Attached child layer auto-composites over `world`, hiding the
  world-drawn label; detached layer has no stored position so
  `vs.pos.rect(it, layer)` returns false; re-creating layer keys
  per loop aborts (needs `push/pop`).
- Conclusion: `draw.image` + a computed `rect` is simpler and
  robust here; reverted to it.

Pending:

- CPS: map each `:Id` to its real screen.
- TODO: use the original fonts (C++ `pingus_small` for footer,
  `chalk_large` for button labels); currently using pico's
  default font.
- TODO: compare screens side-by-side (Atmos render vs original
  pingus) and note visual diffs. (don't implement yet)

## Behavior

The main menu is the first screen.
It shows a parallax cloud background, the logo, five buttons,
and footer texts.
Background music `pingus-1.it` plays.

| Element    | Detail                                            |
|------------|---------------------------------------------------|
| Background | 5 cloud layers, parallax scroll                   |
| Logo       | `core/misc/logo`, centered near top               |
| Buttons    | Story, Editor, Levelsets, Options, Exit           |
| Footer     | copyright (bottom-left) + help bar (bottom-center)|
| Music      | `pingus-1.it`                                      |

### Background layers

Each layer scrolls horizontally and wraps around the screen
width.
A second copy is drawn at `x - width` for seamless wrapping.

| Layer | y   | speed (px/s) |
|-------|-----|--------------|
| 1     | 0   | 12           |
| 2     | 150 | 25           |
| 3     | 200 | 50           |
| 4     | 429 | 100          |
| 5     | 500 | 200          |

### Buttons (offset from screen center)

| Button    | dx   | dy  | Action                       |
|-----------|------|-----|------------------------------|
| Story     | -125 | -20 | worldmap `tutorial.worldmap` |
| Editor    |  125 | -20 | editor screen, stop music    |
| Levelsets | -125 |  50 | level menu                   |
| Options   |  125 |  50 | options screen (pause menu)  |
| Exit      |    0 | 120 | quit                         |

## Compare C++ vs Céu

| Concern     | C++ (`pingus_menu.cpp`)        | Céu (`menu/menu.ceu`)          |
|-------------|--------------------------------|--------------------------------|
| Structure   | class + callbacks              | `code/await` tasks             |
| Background  | `LayerManager` + `update/draw` | `par`: tick speeds / redraw    |
| Buttons     | `gui_manager->create`          | `spawn MenuButton`             |
| Click       | `on_click` dispatch            | `par`/`await b.ok_clicked`     |
| Options     | push screen                    | `emit go_options` (pause)      |
| Music       | `play_music`                   | `play_music`                   |

The Céu port is the closest model for Atmos: tasks composed
with `par`, buttons spawned, clicks awaited.

### Control-flow patterns

| # | Pattern      | Where in the menu                            |
|---|--------------|----------------------------------------------|
| 1 | FSM          | button hover: enter -> highlight -> leave,   |
|   |              | plays `tick`                                 |
| 2 | Continuation | menu result `escape STORY/EDITOR/...` tells  |
|   |              | the caller the next screen                   |
| 3 | Dispatching  | `every redraw` fans out to layers, logo,     |
|   |              | buttons                                      |
| 4 | Lifespan     | menu spawns bg + 5 buttons + logo; ending it |
|   |              | kills all; each button owns its rect/sprites |
| 5 | Signaling    | button `emit ok_clicked` -> menu;            |
|   |              | `emit go_options` pauses the screen          |

## Line-count comparison

Same feature (parallax background + logo + footer + buttons)
across the three ports.
`code` = non-blank, non-comment lines.

| Version | Files                                     | Total | Code |
|---------|-------------------------------------------|------:|-----:|
| C++     | `pingus_menu.{cpp,hpp}`, `menu_button.{cpp,hpp}` | 573 | 396 |
| Céu     | `menu/menu.ceu`, `menu/button.ceu`        |  243 |  212 |
| Atmos   | `menu.atm` (+ `main.atm` harness)         |  140 |  113 |

Relative to C++ (code lines):
    Céu  ~= 54%,
    Atmos ~= 29% (about a quarter of C++).

Caveats for fairness:

- C++ splits the screen and the button across 4 files (incl.
  headers) and carries GPL headers (excluded from `code`).
- Céu `menu.ceu` includes a ~36-line `#if 0 ... #endif` block of
  dead reference C++; real Céu code is closer to ~176, so Atmos
  is ~64% of Céu.
- Atmos `main.atm` (~11 code lines) is the window/loop harness,
  with no C++/Céu equivalent counted here.

Discussion:

- The structured-reactive ports (Céu, Atmos) are far tighter
  than C++; Atmos is the most compact.
- `par` / `watching` / `await` collapse what C++ implements by
  hand: the `LayerManager`, the callback dispatch
  (`on_click` / `draw` / `update`), and the pointer
  enter/leave state machine.
- Atmos shaves further off Céu via table-driven data
  (`LAYERS` / `FOOTER` / `BUTTONS`), `where` scoping, and
  `loop on :draw` / `loop _,x in t` iteration.
- Note the counts exclude the original's resolution-scaling
  branch (`create_background` non-default size), which the port
  does not implement.

## Propose in Atmos

TODO (step 5): map Céu `code/await` + `par` to Atmos
`task`/`spawn`/`par`, and the layer/redraw loops to
`pico-lua` draw calls.

## Notes / pending

- No Atmos scaffold yet; decide entry point and the
  background/button/logo helpers before implementing.
