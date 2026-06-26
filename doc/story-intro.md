# Story Intro: C++ vs Céu vs Atmos

Comparison of the story-intro screen — a blackboard backdrop with
a title, a sequence of pages (each a drawing + a block of text that
types out letter-by-letter), and a `>>>` next button that advances
them — across the three ports. Entered from the menu's `Story`
button through a `Fade`.

Like `menu.md`, this adds a per-concern **code comparison**. The
Atmos port reuses the same `gui.atm` primitives as the menu —
notably the shared `Button` and `Image` / `Text` — so the screen
file (`story/intro.atm`) is mostly page data plus one paging /
reveal loop.

## Line counts

Same "useful" definition as `menu.md`.

| port  | screen file                  | useful |
|-------|------------------------------|-------:|
| C++   | `story_screen.cpp`           |    162 |
| Céu   | `story.ceu`                  |   ~115 |
| Atmos | `story/intro.atm`            |     61 |

The C++ count omits the shared `SurfaceButton` (~62 useful) and
the `.story` loader; the Atmos count omits the shared `gui.atm`
(reused from the menu) — its `Button` / `Image` / `Text` / `Fade`
are not re-spent here. Of `story/intro.atm`'s 61 useful lines,
roughly half are the `PAGES` data table; the rest is the paging +
per-char reveal loop.

## Control-flow patterns

| # | pattern   | where                                              |
|---|-----------|----------------------------------------------------|
| 3 | Dispatch  | `await (:Button \|\| :key.dn[Space])` pages on     |
| 4 | Lifespan  | per-page `Image` + `ls` pool, and each per-char    |
|   |           | `Text`, freed at their own iteration boundary      |
| 5 | Signaling | `>>>` `Button` emits `:Button`; Space via `:key.dn` |
|   | Watching  | outer `:key.dn[Escape]` aborts the screen; inner   |
|   |           | `(advance)` snaps the reveal to the full page      |

## Takeaway

The story screen is where the shared-primitive payoff shows: with
`Button`, `Image`, `Text`, and `Fade` already defined for the
menu, the screen is a `PAGES` table plus one paging/reveal loop.
C++/Céu re-derive a screen class, a specialized button, a draw
override, explicit page-state management, and a timed-substring
text reveal.

The structural wins mirror the menu's: per-page (and per-character)
setup/teardown is loop-scope (`spawn` + `await` + iteration end)
instead of manual `next_text` bookkeeping, and Escape-to-leave is a
`watching` wrapper instead of an `on_escape_press` override.

## Caveats

- Behavioral gap: no `.story` parsing and no width word-wrap — the
  text is hand-wrapped in `PAGES`. (The timed letter-by-letter
  reveal *is* implemented.)
- No tutorial-end Credits hand-off.
- Uses pico's default font, not the original `chalk_*` bitmaps.

## Functionality

Legend: ✓ done · ✗ absent · ~ partial / different.

| Functionality                    | C++ | Céu | Atmos |
|----------------------------------|:---:|:---:|:-----:|
| Blackboard backdrop + title      |  ✓  |  ✓  |   ✓   |
| Paged story (image + text)       |  ✓  |  ✓  |   ✓   |
| `>>>` next button (hover)        |  ✓  |  ✓  |   ✓   |
| Advance — click or key           |  ✓  |  ✓  |   ✓   |
| Char-by-char text reveal         |  ✓  |  ✓  |   ✓   |
| Snap reveal on advance           |  ✓  |  ✓  |   ✓   |
| Escape to leave                  |  ✓  |  ✓  |   ✓   |
| Finish on last page              |  ✓  |  ✓  |   ✓   |
| Fade-in from the menu            |  ~  |  ✓  |   ✓   |
| `.story` file parsing            |  ✓  |  ✓  |   ✗   |
| Word-wrap to width               |  ✓  |  ✓  |   ✗   |
| Original bitmap fonts            |  ✓  |  ✓  |   ✗   |
| Credits hand-off (tutorial end)  |  ✓  |  ✗  |   ✗   |

Notes:

- Advance: C++/Céu accept the `>>>` click or the fast-forward key;
  Atmos accepts the click or `Space` (`:Button || :key.dn[Space]`).
- Text reveal: types ~20 chars/s; an advance mid-reveal snaps to the
  full page (the next one pages on) — as in C++/Céu. The lines are
  hand-wrapped in `PAGES`; `.story` parsing and width word-wrap are
  still deferred.
- The `>>>` button is the shared `gui.atm` `Button`, label-less
  (`text = nil`).

## Building blocks (Atmos)

`story/intro.atm` reuses the `gui.atm` primitives introduced in
`menu.md`:

| primitive  | use here                                          |
|------------|---------------------------------------------------|
| `Image`    | backdrop, each page drawing                       |
| `Text`     | title; the line being typed; each finished line   |
| `Button`   | the `>>>` next button (label-less)                |
| `Fade`     | entry transition from the menu                    |

The screen-specific part is the `PAGES` table and one paging /
reveal loop.

## Code comparison

### Paging — page loop + advance

C++ (`story_screen.cpp`) — a page stack, popped to advance:

```cpp
void StoryScreenComponent::next_text() {
  ...
  pages.pop_back();
  if (!pages.empty()) {
    current_page = pages.back();
    page_surface = Sprite(current_page.image);
  }
}
```

Céu (`story.ceu`) — a counted loop over the pages:

```ceu
loop i in [0 <- {pages.size()}[ do
    { static StoryPage page; page = pages.at(@i); };
    ...
end
```

Atmos (`story/intro.atm`) — a loop that reveals then awaits advance:

```
loop _, page in PAGES {
    spawn Image(page.img, ['%', x=0.5, y=0.39])
    pin ls = tasks(#page.lines)
    watching (:Button || :key.dn [key='Space']) {
        ... type the lines, pooling each finished one ...
    }
    ... fill any not-yet-typed lines (on a snap) ...
    await (:Button || :key.dn [key='Space'])
}
```

The C++ page state (`current_page`, `page_surface`, flags) is
replaced by the loop variable `page`; advancing is the next
iteration.

### The `>>>` next button

C++ (`story_screen.cpp`) — a `SurfaceButton` subclass:

```cpp
class StoryScreenContinueButton : public gui::SurfaceButton {
  void on_click() override { story_comp->next_text(); }
};
```

Céu (`story.ceu`) — a `SurfaceButton`, click emits an event:

```ceu
var& SurfaceButton next =
    spawn SurfaceButton(&r1.pub, "core/misc/next", ...);
every next.component.on_click do
    emit next_text;
end
```

Atmos (`story/intro.atm`) — the shared `Button`, no label:

```
spawn Button(:Next, 0, BUTTON, nil, ['%', x=0.85, y=0.85, ...])
```

C++ and Céu specialize a button class for this screen; Atmos reuses
the menu's `Button`, differing only in the args (`text = nil`, the
`BUTTON` style record).

### Page content — backdrop, title, image

C++ (`story_screen.cpp`) — direct `draw` calls each frame:

```cpp
gc.draw(blackboard, center);
gc.print_center(chalk_large, title_pos, story->get_title());
gc.draw(page_surface, image_pos);
gc.print_left(chalk_normal, text_pos, display_text);
```

Atmos (`story/intro.atm`) — one task per static element:

```
spawn Image(BOARD, ['%', x=0.5, y=0.5, w=1, h=1])   ;; backdrop
spawn Text(TITLE, ['%', x=0.5, y=0.13, h=0.06])     ;; title
spawn Image(page.img, ['%', x=0.5, y=0.39])         ;; page image
```

The single `draw` override becomes one persistent `Image` / `Text`
task each; the body text is the reveal loop below.

### Text reveal

C++ (`story_screen.cpp`) — a timed UTF-8 substring of pre-wrapped
text, redrawn each frame:

```cpp
len = (size_type)(20.0f * time_passed);
display_text = utf8::substr(current_page.text, 0, min(text_len, len));
```

Atmos (`story/intro.atm`) — each line typed by re-spawning a one-
char-longer `Text` every 50ms; the finished line is pooled:

```
loop l, line in page.lines {
    val r = ['%', x=0.15, y=0.56+((l-1)*0.04), h=0.025, anchor=:W]
    loop c in #line {
        spawn Text(string.sub(line, 1, c), r)   ;; growing prefix
        await 50ms
    }
    spawn @ls Text(line, r)                      ;; freeze finished line
}
```

C++ recomputes a substring against a clock in its `update`/`draw`;
Atmos lets the loop *be* the clock — each character is one
`Text` that lives 50ms (the loop iteration is its scope). Both
assume pre-wrapped text; Atmos hand-wraps in `PAGES` (no `.story`
loader, no width `break_line`).

### Snap on advance

C++ — a `page_displayed_completly` flag: the first advance completes
the reveal, the next pages on.

Atmos — the reveal runs inside `watching (advance)`; an advance
aborts it, then the not-yet-typed lines are filled in full (the
pool's count `#ls` marks how far typing got), and a second advance
pages on:

```
watching (:Button || :key.dn [key='Space']) {
    ... per-line reveal ...
}
val n = #ls                           ;; lines already shown
loop l in #page.lines-n {             ;; fill the rest
    val i = n+l
    spawn @ls Text(page.lines@i) <- [...]
}
```

### Lifespan / cleanup

C++ — implicit: reassigning `page_surface` drops the old sprite;
flags are reset by hand in `next_text`.

Céu — loop-scope: the per-page `RRect` / `Sprite` spawns are killed
at the iteration boundary.

Atmos — loop-scope at two levels. The page's `Image` and `ls` pool
live for the page iteration; each per-char `Text` lives for one
char iteration (re-spawned a char longer each 50ms). The backdrop /
title / button, spawned once *before* the loop, persist across all
pages.

### Entry / exit

C++ (`story_screen.cpp`) — Escape pops, last page pops or hands to
Credits:

```cpp
void StoryScreen::on_escape_press() { pop_screen(); }
// next_text(): if (pages.empty()) pop_screen();  // or Credits
```

Atmos (`story/intro.atm`) — Escape aborts, exhausting the loop
finishes:

```
watching :key.dn [key='Escape'] {
    loop _, page in PAGES {
        ...
        await (:Button || :key.dn [key='Space'])
    }
}                          ;; Escape or last page -> task returns
```

`watching` gives Escape-to-leave for free; running off the end of
`PAGES` finishes the screen, returning to the menu's `Fade`, which
thaws it.
