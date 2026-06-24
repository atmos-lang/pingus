# Plan: Main Menu

Port the Pingus main menu (`PingusMenu`) to Atmos.
Code lives in the repo root (`.`).

## Status

- [x] designate feature: main menu
- [x] execute locally (offscreen) + screenshot
- [x] trace behavior
- [x] specify feature (this plan)
- [x] compare C++ and Céu versions
- [ ] propose in Atmos
- [ ] update plan
- [ ] implement

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

## Propose in Atmos

TODO (step 5): map Céu `code/await` + `par` to Atmos
`task`/`spawn`/`par`, and the layer/redraw loops to
`pico-lua` draw calls.

## Notes / pending

- No Atmos scaffold yet; decide entry point and the
  background/button/logo helpers before implementing.
