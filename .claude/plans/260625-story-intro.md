# Plan: Story intro screen

The menu's `:Story` button opens a two-screen flow: this intro
story (a paged chalkboard) then the worldmap
(`260625-story-map.md`). Opened via `Fade` (`260625-fade.md`).

This is the simpler of the two — port it first.

## Status

- [x] assets: copied `blackboard.png`, `next.png`, `story0.png`
- [~] slice 0 STATIC done (`story/intro.atm`, a `story` package:
  `init.atm` exposes `Intro`): board + title + `story0` drawing
  (at 0.5,0.39, per C++ `page_surface`) + left-aligned pre-wrapped
  text (per `print_left`+`break_line 570`) + `>>>`; `Escape`
  returns. Page-advance (click) still TODO.
- [x] wired `:Story -> await Fade(menu, Story.Intro)`
- [x] page-advance: 3 hardcoded pages (story0..2); `par :any`
  (draw ‖ advance). The `>>>` is now a `Button` task (copied from
  menu.atm's Button, specialized: next image + hover/`tick`, no
  label) that `emit @(:parent) :Next` on click; the advance loop
  `await :Next` (i+1; `break` at `#PAGES`); last page or `Escape`
  finishes -> returns.
- [~] convert `.story` -> generated `.atm`: DONE the data file
  `data/stories/tutorial_intro.story.atm` (mirrors the C++ path +
  `.atm` suffix; module returns `[title, music, pages]`; all 8
  pages, text pre-wrapped ~53 cols; mirrors font tables).
  PENDING: point `intro.atm` at
  `require "data/stories/tutorial_intro.story"` (slash form — dots
  collide with `.story`) (replace its hardcoded `TITLE`/`PAGES`).
  - NOTE: `story3..6.png` + `pingus-4.it` not yet copied from
    `pingus.cpp` — pages 4-8 / music will not load until copied.
- [ ] original chalk font (defer; pico default — `260624-font-bitmap.md`)
- [ ] continue to the worldmap after the last page

## Source format

`stories/tutorial_intro.story` (s-expr, ~7 pages):

```
(pingus-story
 (title "The Journey Begins")
 (music "pingus-4.it")
 (pages
  (page (surface (image "story/story0")) (text "..." "..."))
  ... ~7 pages ...))
```

## Screen

- chalkboard frame `core/menu/blackboard.png` as the backdrop;
- per page: the title, the page image (`story/story0..N`), the
  multi-line text, and a `>>>` advance indicator;
- click advances; after the last page, continue to the worldmap;
- music `pingus-4.it`.

## Visual reference (original)

Ran `pingus <…/stories/tutorial_intro.story>` offscreen:

- wooden-framed green chalkboard;
- title centered at top ("The Journey Begins");
- a chalk illustration (sun, igloos, ground);
- narrative text below;
- `>>>` advance arrow bottom-right.

## Céu model (compared)

`cmp/ALL/screens/ceu/story.ceu` is one big `par`:

- a **page loop**: per page, spawn the page image, run a
  **typewriter** (reveal `substr(text, 0, cur)` over time), then
  `await next_text`; after the last page `escape true`;
- a **next button** branch: `on_click -> emit next_text` (also a
  fast-forward key); `Escape -> escape false` (cancel);
- **draw** branches: title (`chalk_large`, top-center) + text
  (`chalk_normal`) + page sprites; backdrop `Wood` + `blackboard`.

Returns a bool (finished/skipped via click vs cancelled).

## Control-flow patterns

| # | pattern      | here                                          |
|---|--------------|-----------------------------------------------|
| 1 | FSM          | page index; typewriter (typing -> done)       |
| 2 | Continuation | walk pages -> `escape` result -> worldmap;    |
|   |              | typewriter accumulates until full             |
| 3 | Dispatching  | draw branch fans out title/text/img           |
| 4 | Lifespan     | the `par`; `escape` kills buttons/sprites     |
| 5 | Signaling    | `emit :next` decouples input (button / key /  |
|   |              | click) from the page-advance loop — key idiom |

Slice 0 defers the typewriter (show full text) and the
cancel/skip; a click advances.

## Slice 0 (MVP) — include vs defer

Include:

- `StoryIntro` opened via `Fade(menu, StoryIntro)`;
- `blackboard.png` backdrop + 2-3 hardcoded pages
  (title + image + text);
- `>>>` indicator; a click advances;
- after the last page, return (to the menu for now; to the
  worldmap once `260625-story-map.md` lands).

Defer:

- `.story` s-expr parsing (hardcode pages first);
- the chalk bitmap font (pico default for now);
- music.

~80-120 lines of Atmos.

## Proposed Atmos structure (slice 0)

Mirrors the Céu `par` (minus typewriter/cancel): a draw branch, an
advance-signal branch, and a page loop driven by `:next`.

```
;; story.atm — global `StoryIntro` (require before menu, like Blank)

val PAGES = [
    [ img="data/images/story/story0.png", txt="For a long time ..." ],
    ... 2-3 hardcoded pages ...
]
val TITLE = "The Journey Begins"
val FRAME = "data/images/core/menu/blackboard.png"

task StoryIntro () {
    var i = 0
    par :any {                         ;; :any -> ends when a branch ends
        ;; draw: blackboard + title + page image + text + ">>>"
        loop on :draw {
            pico.output.draw.image(FRAME, ['%', x=0.5, y=0.5])
            ;; title (top), PAGES@i.img (center), PAGES@i.txt, ">>>"
        }
    } with {
        ;; advance signal: a click emits :next (decoupled, like Céu)
        loop {
            await :mouse.button.dn
            emit @(:task) :next        ;; -> the page loop below
        }
    } with {
        ;; page loop: walk pages on :next, finish after the last
        loop {
            await :next
            if i >= (#PAGES - 1) {
                break()                ;; last page -> StoryIntro ends
            }
            set i = i + 1
        }
    }
}

;; menu.atm dispatch: :Story => await Fade(menu, StoryIntro)
;; (then -> Worldmap, once 260625-story-map.md lands)
```

## Assets

- `images/core/menu/blackboard.png`
- `images/story/story0..N.png`
- (later) `stories/tutorial_intro.story`
