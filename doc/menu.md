# Main Menu: C++ vs Céu vs Atmos

Comparison of the same main-menu screen — parallax cloud
background, logo, footer + help bar, five hover-highlighting
buttons, and the click-to-screen navigation — across the three
ports.

The Atmos port differs structurally: the menu is a *composition*
of small reusable tasks in `gui.atm` (`Object`, `Rect`, `Image`,
`Text`, `Button`, `Fade`), several of which the story screen
reuses verbatim (see `story-intro.md`). So this doc adds, on top
of the usual tables, a per-concern **code comparison** of the
three ports.

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
| Real target screens            |  ✓  |  ✓  |   ~   |
| Pause / resume prev screen     |  ✓  |  ✓  |   ✓   |
| Fade-over transition           |  ✓  |  ✗  |   ✓   |
| Resolution scaling (bg)        |  ✓  |  ✗  |   ✗   |
| Window-resize re-layout        |  ✓  |  ~  |   ~   |
| Credits screen                 |  ✓  |  ✗  |   ✗   |

Notes:

- Bitmap fonts: C++/Céu use `chalk_large` (labels) +
  `pingus_small` (footer); Atmos uses pico's default font
  (see `260624-font-bitmap.md`).
- Real target screens: Atmos opens the story intro for `Story`
  and `Blank` placeholders for the rest.
- Pause / resume: C++ and Céu via the `ScreenManager` stack
  (top-only update); Atmos via `toggle cur(false/true)` inside
  `Fade`.
- Window-resize: Céu/Atmos lay out with relative anchors (`%`);
  Atmos background top-edges are still px, so it is partial.

## Building blocks (Atmos)

The screen is assembled from `gui.atm` primitives, each a tiny
task with one job:

| primitive                                   | role                                                  |
|---------------------------------------------|-------------------------------------------------------|
| `Image (path, rect)`                        | draw an image every frame                             |
| `Text (text, rect)`                         | draw a label every frame                              |
| `Rect (rect, color)`                        | draw a filled rectangle every frame                   |
| `Object (rect)`                             | invisible hit-region; emits `:o.over.*` / `:o.click.*`, exposes `pub` |
| `Button (id, target, style, text, rect)`    | `Object` + `Image`×2 + `Text`; tick + highlight; emits `:Button [id]` |
| `Fade (cur, Next, ...)`                     | freeze `cur`, reveal `Next` through a growing clip    |

`menu.atm` is then mostly data tables (`LAYERS`, `FOOTER`,
`BUTTONS`) plus a `Menu` task that `spawn`s these primitives.

## Code comparison

Each concern shows the shortest representative slice of the three
ports.

### Button — hover state + tick

C++ (`menu_button.cpp`) — set a flag, polled later in `draw`:

```cpp
void MenuButton::on_pointer_enter() {
  mouse_over = true;
  PingusSound::play_sound("tick");
}
void MenuButton::on_pointer_leave() { mouse_over = false; }
```

Céu (`button.ceu`) — await enter, spawn highlight, await leave:

```ceu
loop do
    await component.on_pointer_enter;
    {play_sound("tick");};
    do
        spawn Sprite("menuitem_highlight");
        await component.on_pointer_leave;
    end
end
```

Atmos (`gui.atm` `Button`) — await events, toggle the highlight:

```
loop {
    await :o.over.on
    pico.output.sound(style.sound)
    toggle hl(true)
    await :o.over.off
    toggle hl(false)
}
```

Céu and Atmos express enter→leave as a linear `await` pair; Atmos
freezes a pre-spawned highlight task with `toggle` rather than
spawning/destroying it.

### Button — draw per state

C++ — conditional draw each frame:

```cpp
gc.draw(surface_p, pos);
if (mouse_over) gc.draw(highlight, pos);
gc.print_center(font_large, pos, text);
```

Atmos — three persistent draw tasks; z-order = spawn order:

```
spawn Image(style.image, rect)           ;; base
pin hl = spawn Image(style.hover, rect)  ;; highlight (toggled)
spawn Text(text, ...)                    ;; label, on top
```

The per-frame `if (mouse_over)` becomes a frozen task toggled on
the hover transition; no draw branch polls a flag.

### Button — hit-test

C++ (`menu_button.cpp`) — manual AABB:

```cpp
bool MenuButton::is_at(int x, int y) {
  return x > x_pos - w/2 && x < x_pos + w/2
      && y > y_pos - h/2 && y < y_pos + h/2;
}
```

Atmos (`gui.atm` `Object`) — `pico.vs.pos.rect` against the rect:

```
loop e on :mouse.motion {
    val o = pico.vs.pos.rect(e, rect)
    if o != pub.over { set pub.over = o; ... }
}
```

C++ and Céu hand-roll a rectangle test per button; Atmos delegates
to `pico.vs.pos.rect` inside the one shared `Object`.

### Button → menu signal

C++ — direct callback up a pointer:

```cpp
void MenuButton::on_click() { menu->on_click(this); }
```

Céu — emit a local event:

```ceu
every component.on_click do
    emit ok_clicked;
end
```

Atmos — emit a tagged event to a caller-relative level:

```
loop on :o.click.dn {
    emit @(target + 1) :Button [id=id]
}
```

### Parallax background

Céu (`menu.ceu`) — an update branch ‖ a draw branch:

```ceu
every v in dt do
    x1 = mod(x1 + 12*dt, W);  ...
with
    every redraw do
        draw(s1, x1, y1);  draw(s1, x1-W, y1);  ...
    end
```

Atmos (`menu.atm` `Layer`) — one task per layer, two `Image`s:

```
spawn Image(l.png, r1)
spawn Image(l.png, r2)
loop us on :clock {
    set x = x + ((l.speed * us) / 1000000)
    if x >= W { set x = x - W }
    set r1.x = x + (W/2)
    set r2.x = x - (W/2)
}
```

C++ bakes the velocity into a `LayerManager` descriptor; Céu and
Atmos scroll-and-wrap inline. Atmos draws through two `Image`
tasks over mutable rects, so the layer keeps no draw loop of its
own — the `par` of earlier versions is gone.

### Click dispatch / navigation

C++ (`pingus_menu.cpp`) — a pointer-compare switch:

```cpp
void PingusMenu::on_click(MenuButton* b) {
  if (b == start_button)        do_start("tutorial.worldmap");
  else if (b == options_button) push_screen(OptionMenu());
  ...
}
```

Atmos (`menu.atm`) — `match` on the awaited id:

```
loop {
    val but = await :Button
    match but.id {
        :Story   => await Fade(menu, Story.Intro)
        :Options => await Fade(menu, Blank, "Options")
        :Exit    => break()
    }
}
```

### Pause / resume

C++ — a `ScreenManager` stack: `push_screen` covers the menu,
`pop_screen` resumes it (engine-owned, not in the screen file).

Atmos (`gui.atm` `Fade`) — the caller passes the live `menu`
handle; `Fade` freezes and later thaws it:

```
toggle cur(false)        ;; freeze the menu
pin nxt = spawn Next(...)
await(nxt)               ;; run the next screen to completion
toggle cur(true)         ;; resume the menu
```

The engine-level stack becomes one `toggle` pair around a
`spawn`/`await`.

### Lifecycle

C++ — `gui_manager->create<MenuButton>(...)` in the ctor; the
manager owns the buttons; `~PingusMenu` is empty.

Atmos — `spawn` inside `Menu`; children live in `Menu`'s scope and
die when it ends — no manager, no dtor:

```
spawn Button(:Story, 1, BUTTONS, "Story") <- ['%', x=0.35, ...]
...
await(false)             ;; hold the menu (and its children) alive
```

## Line counts

"Useful" excludes blank, comment-only, brace/punctuation-only,
and (C++) `#include`/`using`/`namespace` lines; Céu's 36-line
`#if 0` reference block is excluded too.

| port  | screen-specific files                        | useful |
|-------|----------------------------------------------|-------:|
| C++   | `pingus_menu.cpp` 156 + `menu_button.cpp` 65 |    221 |
| Céu   | `menu.ceu` ~104 + `button.ceu` 35            |   ~139 |
| Atmos | `menu.atm` 63 (+ `main.atm` 13 harness)      |     63 |

The Atmos figure excludes `gui.atm` (90 useful) because those
primitives are *shared* — the story screen reuses `Object`,
`Image`, `Text`, `Button`, `Fade`. Charging the menu its full
share of `gui.atm` puts it near ~150, still under Céu; but the
marginal cost of the menu *as a screen* is the 63 lines of
`menu.atm`.

None of the three count their substrate: C++ leans on
`ScreenManager` / `gui_manager` / `LayerManager`, Céu on its
`RRect` / `RectComponent` bindings, Atmos on `pico` + `gui.atm`.

## Control-flow patterns

| # | pattern   | where                                               |
|---|-----------|-----------------------------------------------------|
| 1 | FSM       | `Object` over-state; `Button` hover enter/leave     |
| 3 | Dispatch  | `match but.id -> await Fade / Blank`                |
| 4 | Lifespan  | `spawn` children in `Menu`; ending it frees them    |
| 5 | Signaling | `Object` `:o.click.dn` -> `:Button [id]` -> menu    |

## Takeaway

The headline is not the background (~even work in all three); it
is the button component and the callback / dispatch / lifecycle
plumbing.

C++ spends ~65 lines on a `MenuButton` class (flag, draw-per-
state, AABB hit-test, setters) plus a pointer-compare dispatch and
a `ScreenManager` stack. Atmos expresses the button as a `style`
record + one `spawn Button(...)`, the dispatch as `match but.id`,
and the pause as a `toggle` pair — because the button *itself* is
a reusable `gui.atm` task shared with the story screen.

The `on_click` switch, the `update` / `draw` overrides, the
ctor/dtor ownership, and the `ScreenManager` push/pause all
dissolve into `loop on :draw`, `emit` / `await` / `match`,
`toggle`, and scope-based task abortion.

## Caveats

- C++ splits the screen and button across four files (incl.
  headers) and carries GPL headers.
- Céu `menu.ceu` embeds a 36-line `#if 0` block of dead reference
  C++ (excluded above).
- `main.atm` is just window setup + `await Menu()`; the menu task,
  dispatch, pause (`toggle`) and `Blank` placeholder live in
  `menu.atm` / `main.atm`.
- The port targets a fixed 800x600 window, so it omits the
  original's resolution-scaling path.

## TODO

- Transition effect (Atmos-natural, harder in C++): pass `menu`
  into `Fade` and keep it *live* during the reveal, so both
  screens animate at once — no snapshot backdrop. C++ `fade_over`
  is draw-only over a *frozen* screen. Caveat: pico's
  current-layer is global and `clip` is per-layer, so the two
  live screens must sit on different layers.
