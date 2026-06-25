# Main Menu: C++ vs Céu vs Atmos

Line-count comparison of the same feature (parallax background,
logo, footer + help bar, and five hover-highlighting buttons)
across the three ports.

## Methodology

"Useful code" excludes, uniformly across all three:

- `.hpp` headers (declarations only),
- comments, blank lines,
- pure-brace / punctuation-only lines (`{`, `}`, `);`, ...),
- C++ boilerplate (`#include` / `using` / `namespace`),
- Céu's dead `#if 0 ... #endif` reference block.

What remains is executable statements.

## Headline

| Version | Files (implementation)                      | Useful |
|---------|---------------------------------------------|-------:|
| C++     | `pingus_menu.cpp` 156 + `menu_button.cpp` 65 |    221 |
| Céu     | `menu.ceu` 125 + `button.ceu` 37            |    162 |
| Atmos   | `menu.atm` 81 (+ `main.atm` 10 harness)     |  81/91 |

For reference, `cloc` on the two C++ `.cpp` files reports 312
code lines; the drop to 221 removes brace/boilerplate-only lines.

Ratios (useful, `menu.atm` vs C++ impl):

- Atmos ~= 37% of C++,
- Atmos ~= 50% of Céu.

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
is ~184 and Atmos `menu.atm` (81) is ~44% of it.

## Where the savings are (by concern)

| Concern          | C++ | Atmos | What collapses                      |
|------------------|----:|------:|-------------------------------------|
| Button component |  65 |   ~18 | hover state-machine + per-state     |
|                  |     |       | draw + hit-test -> one `par`        |
|                  |     |       | (draw ‖ motion-track) + `vs`        |
| Background       | ~18 |   ~19 | roughly even; but C++ also needs    |
|                  |     |       | `update`/`draw` plumbing elsewhere  |
| Dispatch         | ~26 |  ~0\* | `draw_background`+`update` callbacks |
|                  |     |       | -> `loop on :draw` inline per task  |
| Click handling   | ~15 |    ~7 | `on_click` switch + virtual wiring  |
|                  |     |       | -> `par :any` / `await` returns id  |
| Lifecycle        | ~40 |    ~5 | ctor builds / dtor / manager owns   |
|                  |     |       | -> `spawn`/pool, auto-abort on      |
|                  |     |       | scope end                           |

\* folded into each task's `loop on :draw`.

## Takeaway

The headline difference is not the background (~even); it is the
button component plus the callback / dispatch / lifecycle
plumbing.

C++ spends ~65 lines on a `MenuButton` class (state, draw per
state, hit-test, setters) that Atmos expresses in ~18 via `par`
+ `watching` / `await`.

The `on_click` switch, the `update` / `draw` overrides, and the
ctor/dtor ownership all disappear into `loop on :draw`,
`par :any`, and scope-based task abortion.

Atmos shaves further off Céu via table-driven data
(`LAYERS` / `FOOTER` / `BUTTONS`), `where` scoping, and
`loop on :draw` / `loop _,x in t` iteration.

## Caveats

- C++ splits the screen and button across four files (incl.
  headers) and carries GPL headers.
- Céu `menu.ceu` embeds a ~36-line `#if 0` block of dead
  reference C++ (excluded above).
- `main.atm` is the window/loop harness with no equivalent in
  the counted C++ files.
- The port targets a fixed 800x600 window, so it omits the
  original's resolution-scaling path.
