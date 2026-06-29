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
- [ ] **NEXT: dots hover/click live** (the only slice-1 gap) -- see
  "Next step" below
- [ ] re-enable `await Intro()` in `story/init.atm` (commented out
  while testing the worldmap; restore intro -> world flow)
- [x] `Fade` runs on `:window` not `:world`: 2 screenshots `src=:window`
  + 3 `:draw` loops bracket `set.layer(:window) .. set.layer(old)`, so
  the wipe clip is screen-space (was distorted once `:world` -> map).
  Verified: menu->world transition clean (mid-wipe not freezable in
  xvfb -- unthrottled clock makes it near-instant).
- [x] leave button: full-window `:hud` made in `Hud`; `loop { set :hud;
  await(true) }` sets `:hud` at init so `get.image` sizes `'%'` vs
  `:hud`; flush SW like C++ (`SurfaceButton(0, h-37)`); "Leave" label
  via `pico.xin.rect`; freed on exit by `menu.atm` `push`/`pop`.
- [ ] slice 2: walking pingu along paths (path graph + traversal)
- [ ] slice 3: camera follow + parallax layers
- [ ] slice 4: real savegame status, outro/credits, sounds
- [ ] continue here from the intro story's last page

## Next step: clickable dots

The dots draw on `:world` (the map), but the mouse event `e` is in
**window pixels** (`input.c:218`, "regardless of current layer"). The
gui `Object` does `pico.vs.pos.rect(e, rect)` with no layer arg, so it
reads `e` in the current layer (`:world` = map) -> the dot hit-test
always misses. (The leave button dodged this only because its `:hud`
is window-sized, so window px == layer px.)

Fix -- switch `Object` from the raw event `e` to `pico.get.mouse`,
which projects the window cursor into the current layer through the
scene (`input.c:41` `pico_cv_pos`):

| where (`gui.atm` `Object`) | from | to |
|----------------------------|------|----|
| motion loop | `pico.vs.pos.rect(e, rect)` | `pico.vs.pos.rect(pico.get.mouse('%'), rect)` |
| button loop | (uses `pub.over`) | unchanged -- `pub.over` now correct |

- `pub` init already uses `get.mouse('%')`, so this just makes the
  motion loop consistent -- no new layer juggling, no `:window` hint.
- dot `target` is already `2` (the pool adds a level; see the pool
  test), so once the hit-test lands, the click dispatches to `World`.
- caveat: `get.mouse` reads the live cursor, which on a `:mouse.motion`
  equals `e`'s position -- so behaviour for window-space buttons
  (menu, leave) is unchanged; only off-window-layer hit-tests get fixed.
- this is the same `Object` touched by `260629-object.md`; this fix is
  small and orthogonal, so do it first, then the id refactor on top.

Verify offscreen: hover a dot (highlight + tick), click an `:Open`
dot (tick) and an `:invalid` dot (chink).

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
