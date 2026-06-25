# Plan: Story worldmap

The worldmap (level-selection map), shown after the intro story
(`260625-story-intro.md`) in the menu's `:Story` flow.
Opened via `Fade` (`260625-fade.md`).

~4-5x the menu, so port in **slices** — a static MVP first, then
the algorithmic parts.

## Status

- [ ] copy assets + a worldmap data file into the repo
- [ ] slice 1 (MVP): bg + static dots + standing pingu +
  click-to-select + leave button
- [ ] slice 2: walking pingu along paths (path graph + traversal)
- [ ] slice 3: camera follow + parallax layers
- [ ] slice 4: real savegame status, outro/credits, sounds
- [ ] continue here from the intro story's last page

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
