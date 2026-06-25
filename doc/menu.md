# Main Menu: C++ vs Céu vs Atmos

Comparison of the same main-menu screen (parallax background,
logo, footer + help bar, five hover-highlighting buttons, and the
click-to-screen navigation) across the three ports.

## Functionality

What each port actually implements.
Legend: ✓ done · ✗ absent · ~ partial / different.

| Functionality                  | C++ | Céu | Atmos |
|--------------------------------|:---:|:---:|:-----:|
| Parallax cloud background      |  ✓  |  ✓  |   ✓   |
| Logo                           |  ✓  |  ✓  |   ✓   |
| Footer copyright text          |  ✓  |  ✓  |   ✓   |
| Help bar (bottom)              |  ✓  |  ✓  |   ✓   |
| Five buttons                   |  ✓  |  ✓  |   ✓   |
| Hover highlight                |  ✓  |  ✓  |   ✓   |
| Hover "tick" sound             |  ✓  |  ✓  |   ✓   |
| Original bitmap fonts          |  ✓  |  ✓  |   ✗   |
| Background music (pingus-1.it) |  ✓  |  ✓  |   ✗   |
| Click -> dispatch              |  ✓  |  ✓  |   ✓   |
| Real target screens            |  ✓  |  ✓  |   ✗   |
| Pause / resume prev screen     |  ✓  |  ✓  |   ✓   |
| Fade-over transition           |  ✓  |  ✗  |   ✗   |
| "letsgo" start sound           |  ✓  |  ✗  |   ✗   |
| Resolution scaling (bg)        |  ✓  |  ✗  |   ✗   |
| Window-resize re-layout        |  ✓  |  ~  |   ~   |
| Story-seen / tutorial intro    |  ✓  |  ✗  |   ✗   |
| Credits screen                 |  ✓  |  ✗  |   ✗   |
| Per-button hint / desc         |  ~  |  ✗  |   ✗   |

Notes:

- Original bitmap fonts: `chalk_large` (labels) + `pingus_small`
  (footer); Atmos still uses pico's default font
  (see `260624-font-bitmap.md`).
- Real target screens: Atmos opens `Blank` placeholders instead of
  worldmap / editor / level menu / options.
- Pause / resume: C++ and Céu via the `ScreenManager` stack
  (top-only update); Céu also signals `emit go_options`; Atmos via
  `toggle menu(false/true)`.
- Fade-over: C++ `ScreenManager::fade_over` (centered box grows
  0->full); the Céu/Atmos ports don't reproduce the visual fade.
- Window-resize: Céu/Atmos lay out with relative anchors (`%`), so
  most re-layout is automatic; Atmos background top-edges are still
  px, so it is only partial.
- Per-button hint: C++ stores `desc` and has `set_hint`, but the
  on-screen display is gated by `if (0)` (dead).

## Line counts

"Useful code" excludes, uniformly across all three: `.hpp`
headers, comments, blank lines, pure-brace / punctuation-only
lines (`{`, `}`, `);`, ...), C++ boilerplate (`#include` / `using`
/ `namespace`), and Céu's dead `#if 0 ... #endif` block.
What remains is executable statements.

| Version | Files (implementation)                       | Useful |
|---------|----------------------------------------------|-------:|
| C++     | `pingus_menu.cpp` 156 + `menu_button.cpp` 65 |    221 |
| Céu     | `menu.ceu` 125 + `button.ceu` 37             |    162 |
| Atmos   | `menu.atm` 95 (+ `main.atm` 7 harness)       | 95/102 |

For reference, `cloc` on the two C++ `.cpp` files reports 312
code lines; the drop to 221 removes brace/boilerplate-only lines.

`menu.atm` also carries the in-file orchestration that the C++
keeps in the engine: the click dispatch (`match but@1 -> await
Blank`), the `toggle` pause/resume, and the `Blank` placeholder.
The C++ `on_click` switch is the rough dispatch counterpart; its
stack pause lives in `ScreenManager` (not counted in the 221).

Ratios (useful, `menu.atm` vs impl):

- Atmos ~= 43% of C++,
- Atmos ~= 59% of Céu.

## Like-for-like adjustment

The C++ count includes behavior the Atmos port does not
implement:

| C++-only chunk                                   | Useful |
|--------------------------------------------------|-------:|
| `resize()` (re-layout on window resize)          |     14 |
| `create_background` resolution-scaling branch    |     12 |
| `do_start` story-seen / tutorial logic           |      6 |
| `show_credits` / `set_hint` / `on_escape_press`  |     ~5 |

~37 lines of C++ have no port counterpart, so the comparable C++
is ~184 and Atmos `menu.atm` (95) is ~52% of it.

## Where the savings are (by concern)

| Concern          | C++ | Atmos | What collapses                      |
|------------------|----:|------:|-------------------------------------|
| Button component |  65 |   ~24 | hover state-machine + per-state     |
|                  |     |       | draw + hit-test -> one 3-branch     |
|                  |     |       | `par` (hover ‖ draw ‖ click) + `vs` |
| Background       | ~18 |   ~15 | roughly even; but C++ also needs    |
|                  |     |       | `update`/`draw` plumbing elsewhere  |
| Draw/update      | ~26 |  ~0\* | `draw_background`+`update` callbacks |
|                  |     |       | -> `loop on :draw` inline per task  |
| Click / nav      | ~15 |   ~14 | `on_click` switch + virtual wiring  |
|                  |     |       | -> `emit :Button` + `match but@1`   |
|                  |     |       | -> `await Blank`                    |
| Pause / stack    | eng |    ~6 | `ScreenManager` push/pop/freeze     |
|                  |     |       | -> `toggle menu(false/true)`        |
| Lifecycle        | ~40 |    ~5 | ctor builds / dtor / manager owns   |
|                  |     |       | -> `spawn` + scope-based abort      |

\* folded into each task's `loop on :draw`.

## Takeaway

The headline difference is not the background (~even); it is the
button component plus the callback / dispatch / lifecycle
plumbing.

C++ spends ~65 lines on a `MenuButton` class (state, draw per
state, hit-test, setters) that Atmos expresses in ~24 via a
3-branch `par` (hover ‖ draw ‖ click) + `vs` hit-tests + an
`emit @2 :Button [id]` signal.

The `on_click` switch, the `update` / `draw` overrides, the
ctor/dtor ownership, and the `ScreenManager` push/pause all
disappear into `loop on :draw`, `emit`/`await`/`match`,
`toggle`, and scope-based task abortion.

Atmos shaves further off Céu via table-driven data
(`LAYERS` / `FOOTER` / `BUTTONS`), `where` scoping, and
`loop on :draw` / `loop _,x in t` iteration.

## Caveats

- C++ splits the screen and button across four files (incl.
  headers) and carries GPL headers.
- Céu `menu.ceu` embeds a ~36-line `#if 0` block of dead
  reference C++ (excluded above).
- `main.atm` is now just the window setup + `await Menu()`; the
  menu task, dispatch, pause (`toggle`) and `Blank` placeholders
  all live in `menu.atm`.
- The port targets a fixed 800x600 window, so it omits the
  original's resolution-scaling path.

# TODO

- `Blank` moved from `menu.atm` to `main.atm`. Recalculate the
  `menu.atm` / `main.atm` useful counts, the headline ratios
  (`95/102`, 43% / 59%, ~52%), and the prose that still places
  `Blank` in `menu.atm` (Line-counts intro + Caveats).
- Transition effect (Atmos-natural, harder in C++): pass `menu`
  into `Fade` and keep it *live* during the reveal, so both
  screens run and animate at once — no snapshot backdrop needed.
  C++ `fade_over` is draw-only over a *frozen* screen.
  Caveat: pico's current-layer is global and `clip` is per-layer,
  so the two live screens must sit on different layers (incoming
  on its own clipped layer over the live menu). Worth documenting
  as a showcase difference once the live `Fade` exists.
