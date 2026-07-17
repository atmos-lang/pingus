# Plan: Story worldmap

The worldmap (level-selection map), shown after the intro story
(`260625-story-intro.md`) in the menu's `:Story` flow.
Opened via `Fade` (`260625-fade.md`).

~4-5x the menu, so port in **slices** — a static MVP first, then
the algorithmic parts.

## Status

- [x] copy assets (dots, pingu, arrow, leave button, layer0 bg,
  chink) into the repo
- [x] slice 1 (MVP): `story/world.atm` (`task World`, Story flow:
  intro -> world). Runs (verified offscreen). Scene-based static
  camera: `scene.dim` = the map + `scene.source` = a `CAM` viewport
  (literal `'!'` map coords; hit-test projects through the scene;
  restored on exit). bg + 4 status dots (Buttons) + standing pingu
  + arrow + leave button + dispatch (Leave/locked/open).
  Two pico gotchas fixed: `'!'` rects need `get.image("!",..)` (not
  `"%"`, which gives normalized dims -> ~0px -> `layer.c aux`
  assertion); scene fields are `source`/`target`, not `src`/`dst`.
- [x] Leave -> menu: full-window `:hud` (window px == hud px, so the
  gui `Object` hit-test works unchanged) created in `Hud`; `par`
  enter/leave brackets the leave `Button` (`:hud` / `:world`);
  `await Button` is in a `par` branch -> +1 task level -> `target=3`
  (`Button -> branch -> Hud -> World`, see `bug.atm`). Verified
  offscreen: hover highlights, click returns to the menu (scene
  restored).
- [x] **dots hover/click live**: `gui.atm` `Object` motion loop now
  hit-tests `pico.get.mouse('%')` (cursor projected into the current
  layer) instead of the raw window-pixel event `e` -- dots on the
  retargeted `:world` layer are clickable; window-space buttons
  unchanged (`get.mouse == e` there). See "Next step" below.
- [x] blue **"Watch Intro"** story dot: added the original `introdot`
  (map 429,760, `status=:story`, `story_dot.png`/`_highlight`) as a
  `:story` NODE -> act `:Intro`. Dispatch `break(:intro)`; `Leave`
  now `break(:leave)`. `story/init.atm` loops: `await World()` ->
  `:intro` replays `Intro()` then re-enters the map, `:leave` exits.
  Copied `story_dot{,_highlight}.png` into the repo.
- [x] leave button: full-window `:hud` made in `Hud`; `loop { set :hud;
  await(true) }` sets `:hud` at init so `get.image` sizes `'%'` vs
  `:hud`; flush SW like C++ (`SurfaceButton(0, h-37)`); "Leave" label
  via `pico.xin.rect`; freed on exit by `menu.atm` `push`/`pop`.
- [~] `story/init.atm`: intro -> world loop wired, BUT `await Intro()`
  is currently commented (`;;await Intro()`) for faster world testing.
  RESTORE before final (see Next step 0).
- [ ] **VERIFY offscreen** the clickable-dots + Watch-Intro work (Next
  step 0) -- NOT yet run
- [ ] slice 2: walking pingu along paths (path graph + traversal)
- [ ] slice 3: camera follow + parallax layers
- [ ] slice 4: real savegame status, outro/credits, sounds
- [ ] continue here from the intro story's last page

## Next steps (explicit — resume here on the other machine)

Clickable dots + the blue "Watch Intro" dot are IMPLEMENTED but
UNVERIFIED. Uncommitted edits touch: `gui.atm`, `story/world.atm`,
`story/init.atm`, and two new assets under
`data/images/core/worldmap/`. Work top-down.

### 0. Restore + verify what's already implemented

- `story/init.atm`: un-comment `await Intro()` (currently
  `;;await Intro()`) so the flow is intro -> world again.
- Ask the user to run offscreen (never run the game yourself):
    - `xvfb-run -a -s "-screen 0 800x600x24" pingus` on the port, then
      `import -window root out.png`.
- Confirm:
    - hovering any dot highlights it + plays `tick`;
    - clicking the blue **Watch Intro** dot (map 429,760) replays the
      intro, then returns to the map;
    - clicking an `:invalid` dot plays `chink`; `:green`/`:red` tick;
    - **Leave** still returns to the menu (scene restored).
- What changed (so a fresh session has context):
    - `gui.atm` `Object` motion loop: hit-test
      `pico.vs.pos.rect(pico.get.mouse('%'), rect)` (not raw `e`) so
      dots on the retargeted `:world` layer are clickable; window-space
      buttons unchanged (`get.mouse == e` there).
    - `story/world.atm`: added `introdot` NODE (`status=:story`), a
      `:story` sprite branch (`story_dot.png`/`story_dot_highlight.png`)
      and `act=:Intro`; dispatch `break(:intro)` / `break(:leave)`.
    - `story/init.atm`: loop — `await World()` returns `:intro`
      (replay `Intro()`) or `:leave` (exit to menu).

### 1. Parse `data/worldmaps/tutorial.worldmap` (kill hardcoded NODES)

`NODES` in `story/world.atm` is the only remaining inlined data file
(the intro `.story` is already a `require`d module). Mirror that:
convert the s-expr to an Atmos data module, then `require` it.

- Source: `pingus.cpp/data/worldmaps/tutorial.worldmap` (s-expr:
  `worldmap.objects` = `leveldot`/`storydot` with `dot.name`,
  `dot.position x y`, `levelname` or `story`; `worldmap.graph.edges`
  = `edge` with `source`/`destination`/`positions`).
- Emit `data/worldmaps/tutorial.atm` returning
  `[ dim, layer, nodes, edges ]` where each node is
  `[ name, x, y, kind (:level|:story), ref ]`.
- In `story/world.atm`: replace `val NODES = [...]` with
  `val DATA = require "data/worldmaps/tutorial"` and derive `NODES`
  from `DATA.nodes` (map real status later — keep mock status until
  slice 4 savegame).
- Naming caveat (from the intro port): `require` turns every `.` into
  `/`, so drop the inner `.worldmap` from the module name (use
  `tutorial.atm`, not `tutorial.worldmap.atm`).

### 2. Slice 2 — walking pingu along paths

- Use `DATA.edges` (from step 1) as the path graph.
- Introduce the `Sprite` task (draw + frame-animation + hotspot;
  left/right frames) — see "Sprite / Component" below.
- On a reachable dot click: walk the pingu edge-by-edge
  (Continuation pattern — accumulate position per frame until the
  edge completes, then the next edge), then enter/replay.
- Hover mutual-exclusion caveat: worldmap dots may overlap; our
  per-task hover has none (unlike C++ `mouse_over_comp`). Add a
  shared "hovered id" / z-order check if overlaps misbehave.

### 3. Then slices 3-4

- slice 3: camera follows the pingu + parallax layers (SpriteDrawable).
- slice 4: real savegame status (dot colors), outro/credits, sounds.

## Behavior (from C++/Céu survey)

- background worldmap image (larger than the window; camera follows
  the pingu);
- a graph of **dots**: `LevelDot` (a level) and `StoryDot` (a
  narrative node), 19+ on the tutorial map;
- **paths** connect dots; the **pingu** stands on a dot when idle
  and walks the dotted path when moving;
- click a dot: if reachable, the pingu walks there; if already on
  it, enter the level / show the story; if locked, play `chink`;
- dot status drives the sprite: green = finished, red =
  accessible, gray = locked; hover highlights + plays `tick`;
- a "Leave?" button (bottom-left) returns to the menu.

## Entities -> Atmos decomposition

| entity        | Atmos                                              |
|---------------|----------------------------------------------------|
| Worldmap      | `task Worldmap ()` — hosts the screen, owns the    |
|               | dot/pingu pools, returns on Leave / level pick     |
| Dot           | `task Dot (id, x, y, status)` — draws its sprite   |
|               | (FSM on status), hover (tick + highlight), signals |
|               | on click (`emit @.. :Dot [id]`), like the menu     |
|               | `Button`                                           |
| Pingu         | `task Pingu (node)` — slice 1: draw standing sprite |
|               | at the node; slice 2: walk along paths             |
| Paths / graph | deferred (slice 2): nodes + edges + traversal      |
| SpriteDrawable| deferred (slice 3): parallax/foreground layers     |

Mirrors the menu: a container task `par`-composing a pool of
clickable child tasks that signal a tag; the container `await`s the
signal and dispatches.

## Sprite / Component (decision, from the C++ survey)

C++ splits visuals from interaction:

- `Sprite` = **pure visual**: `render` + **animation** (`update`,
  `set_frame`, `set_play_loop`, `restart`) + `hotspot`. No input.
- `Component` = **interaction**: `is_at` hit-test + `draw`/`update`
  + callbacks (`on_*_button_click`, `on_pointer_enter/leave/move`,
  keys); the parent `GroupComponent` detects hover (tracks one
  `mouse_over_comp` -> **mutually exclusive**) and dispatches.

Our port has **no `Component`** — each task draws (`loop on :draw`)
and `await`s its own events filtered by `pico.vs.pos.rect`; the
`Button` task IS the Component-equivalent (already used by menu +
intro `>>>`). So `Dot` follows that same pattern.

Decisions:

- **Introduce a `Sprite` task here** (draw + frame-animation +
  hotspot) — the worldmap **walking pingu** forces it (frames /
  left-right). Static images (board, dots, leave button) can stay
  plain `draw.image`; only animated ones need `Sprite`.
- Keep input in the tasks (no `Component` base).
- **Hover caveat**: our per-task hover has NO mutual exclusion
  (unlike C++'s parent-tracked `mouse_over_comp`). Fine for
  non-overlapping menu/intro buttons; **worldmap dots may overlap**
  -> may need a shared "hovered id" / z-order check.

## Control-flow patterns (the study's point)

| # | pattern        | in the worldmap                              |
|---|----------------|----------------------------------------------|
| 1 | FSM            | dot status -> sprite (green/red/gray); pingu |
|   |                | node transitions                             |
| 2 | Continuation   | walking a path: accumulate position per      |
|   |                | frame until the edge completes, then next    |
| 3 | Dispatching    | `loop on :draw` fans out to dots, pingu,     |
|   |                | layers (camera offset applied to all)        |
| 4 | Lifespan       | Worldmap ends -> its dot/pingu pools abort   |
| 5 | Signaling      | dot click `emit :Dot [id]` -> Worldmap;      |
|   |                | `emit pingu.go_walking(id)`; Leave -> escape |

## Data & assets (must copy into the repo)

Repo currently has none. Copy from the original
(`pingus.orig/data/`):

- `data/images/core/worldmap/` — `pingus_standing.png`,
  `pingus/left.png`, `pingus/right.png`, `dot_{green,red}{,_hl}.png`,
  `dot_invalid.png`, `story_dot{,_highlight}.png`, `arrow.png`,
  `leave_button_*.png`;
- `data/worldmaps/tutorial.worldmap` (+ `tutorial.xml`) — defines
  name, dims, nodes, paths, layers, intro/outro.

Slice 1 hardcodes a few dots; real parsing of the data file is a
later slice.

## Slice 1 (MVP) — include vs defer

Include:

- `Worldmap` container opened via `Fade(menu, Worldmap)`;
- a single static background image (no parallax, static camera);
- 3-4 `Dot`s at hardcoded positions with mock status
  (finished / accessible / locked) -> correct sprite;
- hover highlight + `tick` (reuse the menu Button pattern);
- a standing `Pingu` sprite on one dot (no walking);
- circle hit-test (`dx^2 + dy^2 < r^2`) on click;
- click an accessible dot -> for now open `Blank` (mock level);
  locked -> `chink`;
- "Leave?" button -> return to the menu (Worldmap escapes).

Defer:

- pathfinding + walking animation (slice 2);
- camera follow + parallax layers + SpriteDrawables (slice 3);
- real savegame status, level loading (slice 4).

~200-300 lines of Atmos for slice 1.

## Proposed Atmos structure (slice 1)

```
;; main.atm dispatch (menu.atm): :Story -> ... -> await Fade(menu, Worldmap)

val task Dot (id, x, y, status) {
    ;; FSM: pick sprite by status (green/red/invalid)
    ;; par { hover-track (tick + highlight) }
    ;;     { loop on :draw { draw dot sprite (+ highlight) } }
    ;;     { loop { await click inside; emit @.. :Dot [id] } }
}

task Worldmap () {
    ;; background (static)
    ;; spawn the standing Pingu sprite
    ;; pin dots = tasks(N); spawn Dot(...) x N
    ;; leave button (signals :Leave)
    ;; loop {
    ;;   match await (:Dot | :Leave) {
    ;;     :Dot[id] => if accessible -> await Blank("level "++id)
    ;;     :Leave   => break()   ;; return to menu
    ;;   }
    ;; }
}
```

## Decisions

- slice-1 dot click (accessible) -> open `Blank("level N")`
  (plain `await`; nested fade deferred), exercising the flow.
- launch the original directly into the worldmap (no menu click):
  `pingus -s -w -u <userdir> --no-cfg-file
  /usr/share/games/pingus/data/worldmaps/tutorial.worldmap`.

## Visual reference (original, tutorial worldmap)

Observed running the original offscreen:

- background = an icy island; the map is wider than the 800 window
  (camera offset; water to the left, island right);
- winding dark **paths** (dotted) connect small **dots**;
- the **pingu** stands on a dot near center with a small red
  **target arrow** above it;
- hovering/current dot shows the level name in green above it
  (e.g. "Learning to dig");
- grey **"Leave?"** button bottom-left.

This matches the slice plan; slice 1 reproduces a static subset.
