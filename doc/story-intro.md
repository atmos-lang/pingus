# Story Intro: C++ vs Céu vs Atmos

Comparison of the story-intro screen — a blackboard backdrop with
a title, a sequence of pages (each a drawing + a block of text),
and a `>>>` next button that advances them — across the three
ports. Entered from the menu's `Story` button through a `Fade`.

Like `menu.md`, this adds a per-concern **code comparison**. The
Atmos port reuses the same `gui.atm` primitives as the menu —
notably the shared `Button` and `Image` / `Text` — so the screen
file (`story/intro.atm`) is almost entirely page data + a paging
loop.

## Functionality

Legend: ✓ done · ✗ absent · ~ partial / different.

| Functionality                    | C++ | Céu | Atmos |
|----------------------------------|:---:|:---:|:-----:|
| Blackboard backdrop + title      |  ✓  |  ✓  |   ✓   |
| Paged story (image + text)       |  ✓  |  ✓  |   ✓   |
| `>>>` next button (hover)        |  ✓  |  ✓  |   ✓   |
| Advance on click                 |  ✓  |  ✓  |   ✓   |
| Escape to leave                  |  ✓  |  ✓  |   ✓   |
| Finish on last page              |  ✓  |  ✓  |   ✓   |
| Fade-in from the menu            |  ~  |  ✓  |   ✓   |
| `.story` file parsing            |  ✓  |  ✓  |   ✗   |
| Word-wrap to width               |  ✓  |  ✓  |   ✗   |
| Char-by-char text reveal         |  ✓  |  ✓  |   ✗   |
| Original bitmap fonts            |  ✓  |  ✓  |   ✗   |
| Credits hand-off (tutorial end)  |  ✓  |  ✗  |   ✗   |

Notes:

- Atmos hand-wraps the page text in `PAGES` and shows it whole;
  `.story` parsing, width word-wrap, and the timed reveal are
  deferred (see the comment in `story/intro.atm`).
- The `>>>` button is the shared `gui.atm` `Button`, configured
  with no label (`text = nil`).

## Building blocks (Atmos)

`story/intro.atm` reuses the `gui.atm` primitives introduced in
`menu.md`:

| primitive  | use here                                  |
|------------|-------------------------------------------|
| `Image`    | backdrop, each page drawing               |
| `Text`     | title, each wrapped text line             |
| `Button`   | the `>>>` next button (label-less)        |
| `Fade`     | entry transition from the menu            |

The screen-specific part is the `PAGES` table and one paging loop.

## Code comparison

### Paging — current page + advance

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

Atmos (`story/intro.atm`) — a loop that awaits the button:

```
loop _, page in PAGES {
    spawn Image(page.img, ['%', x=0.5, y=0.39])
    ... spawn the text lines ...
    await :Button
}
```

The C++ page state (`current_page`, `page_surface`, flags) is
replaced by the loop variable `page`; advancing is just the next
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

Both C++ and Céu specialize a button class for this screen; Atmos
reuses the menu's `Button` task, differing only in the arguments
(`text = nil`, a different `style`).

### Page content — backdrop, title, body

C++ (`story_screen.cpp`) — direct `draw` calls each frame:

```cpp
gc.draw(blackboard, center);
gc.print_center(chalk_large, title_pos, story->get_title());
gc.draw(page_surface, image_pos);
gc.print_left(chalk_normal, text_pos, display_text);
```

Atmos (`story/intro.atm`) — one task per visual element:

```
spawn Image(BOARD, ['%', x=0.5, y=0.5, w=1, h=1])   ;; backdrop
spawn Text(TITLE, ['%', x=0.5, y=0.13, h=0.06])     ;; title
spawn Image(page.img, ['%', x=0.5, y=0.39])         ;; page image
pin texts = tasks(#page.text)                       ;; text lines
loop j, line in page.text {
    spawn @texts Text(line, r) where { ... }
}
```

The single `draw` override becomes one persistent `Image` / `Text`
task each; the body lines are a pooled set of `Text` tasks.

### Word-wrap / `.story` parsing

C++ (`story_screen.cpp`) — a timed UTF-8 reveal of pre-wrapped
text:

```cpp
len = (size_type)(20.0f * time_passed);
display_text = utf8::substr(current_page.text, 0, min(text_len, len));
```

Atmos — no parser or reveal: the text is hand-wrapped in `PAGES`
and drawn whole. This is the main behavioral gap (see the
`story/intro.atm` header comment); the original loads `.story`
files and `break_line`s to ~570px.

### Lifespan / cleanup between pages

C++ — implicit: reassigning `page_surface` drops the old sprite;
flags are reset by hand in `next_text`.

Céu — loop-scope: the per-page `RRect` / `Sprite` spawns are
killed at the iteration boundary.

Atmos — the same loop-scope idea, made explicit by `await`:

```
loop _, page in PAGES {
    spawn Image(page.img, ...)        ;; this page's image
    pin texts = tasks(#page.text)     ;; this page's lines
    ... spawn @texts Text(...) ...
    await :Button                     ;; hold the page open
}                                     ;; iteration ends -> page freed
```

The page's `Image` and the `texts` pool live exactly as long as
the iteration; the next `:Button` ends it (freeing them) and the
loop respawns for the next page. The backdrop / title / button,
spawned once *before* the loop, persist across all pages.

### Entry / exit

C++ (`story_screen.cpp`) — Escape pops, last page pops or hands
to Credits:

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
        await :Button
    }
}                          ;; Escape or last page -> task returns
```

`watching` gives Escape-to-leave for free; running off the end of
`PAGES` finishes the screen, returning to the menu's `Fade`, which
thaws it.

## Line counts

Same "useful" definition as `menu.md`.

| port  | screen file                  | useful |
|-------|------------------------------|-------:|
| C++   | `story_screen.cpp`           |    162 |
| Céu   | `story.ceu`                  |   ~115 |
| Atmos | `story/intro.atm`            |     52 |

The C++ count omits the shared `SurfaceButton` (~62 useful) and
the `.story` loader; the Atmos count omits the shared `gui.atm`
(reused from the menu) — its `Button` / `Image` / `Text` / `Fade`
are not re-spent here. Of `story/intro.atm`'s 52 useful lines,
most are the `PAGES` data table; the paging logic itself is ~12.

## Control-flow patterns

| # | pattern   | where                                            |
|---|-----------|--------------------------------------------------|
| 3 | Dispatch  | `await :Button` advances the page loop           |
| 4 | Lifespan  | per-page `Image` + `texts` pool die per iteration |
| 5 | Signaling | `>>>` `Button` emits `:Button` -> the page loop  |
|   | Watching  | `watching :key.dn [Escape]` aborts the screen    |

## Takeaway

The story screen is where the shared-primitive payoff shows: with
`Button`, `Image`, `Text`, and `Fade` already defined for the
menu, the whole screen is a `PAGES` table plus a ~12-line paging
loop. C++/Céu re-derive a screen class, a specialized button, a
draw override, and explicit page-state management.

The structural wins mirror the menu's: per-page setup/teardown is
loop-scope (`spawn` + `await` + iteration end) instead of manual
`next_text` bookkeeping, and Escape-to-leave is a `watching`
wrapper instead of an `on_escape_press` override.

## Caveats

- Behavioral gap: no `.story` parsing, width word-wrap, or timed
  char-by-char reveal — text is hand-wrapped in `PAGES` and shown
  whole.
- No tutorial-end Credits hand-off.
- Uses pico's default font, not the original `chalk_*` bitmaps.
