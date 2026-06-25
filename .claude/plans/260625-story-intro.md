# Plan: Story intro screen

The menu's `:Story` button opens a two-screen flow: this intro
story (a paged chalkboard) then the worldmap
(`260625-story-map.md`). Opened via `Fade` (`260625-fade.md`).

This is the simpler of the two — port it first.

## Status

- [ ] copy assets (blackboard frame + story page images)
- [ ] slice 0: paged chalkboard (title + image + text, click to
  advance), returns after the last page
- [ ] parse the `.story` s-expr (defer; hardcode pages first)
- [ ] original chalk font (defer; pico default for now —
  `260624-font-bitmap.md`)
- [ ] continue to the worldmap after the last page
- [ ] wire `:Story -> StoryIntro` (via `Fade`)

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

## Control-flow patterns

| # | pattern      | here                                       |
|---|--------------|--------------------------------------------|
| 2 | Continuation | walk the pages, then -> worldmap           |
| 1 | FSM          | current page index                         |
| 3 | Dispatching  | `loop on :draw` draws frame/title/img/text |

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

```
;; story.atm — global `StoryIntro` (require before menu, like Blank)

task StoryIntro () {
    val PAGES = [ ... ]               ;; hardcoded { img, text } list
    var i = 0
    par {
        loop on :draw {
            ;; blackboard bg + title + PAGES@i.img + text + ">>>"
        }
    } with {
        loop {
            await :mouse.button.dn    ;; (or a >>> hit-box)
            if i >= (#PAGES - 1) {
                break()               ;; last page -> finish
            }
            set i = i + 1
        }
    }
}

;; menu.atm dispatch: :Story => await Fade(menu, StoryIntro)
```

## Assets

- `images/core/menu/blackboard.png`
- `images/story/story0..N.png`
- (later) `stories/tutorial_intro.story`
