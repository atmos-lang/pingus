# Plan: Bitmap Fonts

Render the original Pingus bitmap fonts in Atmos/pico, since
pico only renders TTF and the pingus fonts are bitmap
(`.png` sheet + `.font` metadata).

Used by the main menu (see `260624-main-menu.md`):

- `chalk_large`  = `fonts/chalk-40px`        (button labels)
- `pingus_small` = `fonts/pingus-small-20px` (footer + help)

## Status

- [ ] inspect `.font` format
- [ ] transform `.font` s-expr -> Atmos table (per font)
- [ ] copy sheet assets
- [ ] font renderer module
- [ ] integrate into the menu (replace `draw.text`)
- [ ] compare with the original

## Source format

The `.font` file is an s-expression:

```
(pingus-font
 (size 40)
 (glyph-count 588)
 (images
  (image
   (filename "images/fonts/chalk-40px.png")
   (glyphs
    (glyph (unicode 65) (offset 0 -38) (advance 22)
           (rect 393 0 415 45))
    ...))))
```

Per glyph:

| field   | meaning                                  |
|---------|------------------------------------------|
| unicode | codepoint                                |
| offset  | `ox oy` pen-relative draw offset (px)    |
| advance | pen step after the glyph (px)            |
| rect    | `x1 y1 x2 y2` crop in the sheet (px)     |

## Transform: s-expr -> Atmos table

One-time conversion (script) emits a generated `.atm` data file
per font.
Only the ASCII subset (codepoints 32..126) is needed by the
menu; emit just those to keep the table small.

Target shape (`fonts/chalk.atm`, `fonts/small.atm`):

```
;; generated from data/images/fonts/chalk-40px.font
val FONT = [
    sheet = "data/images/fonts/chalk-40px.png",
    size  = 40,
    glyphs = [
        ;; [codepoint] = [ off=[ox,oy], adv, rect=[x,y,w,h] ]
        [32] = [ off=[0,0],   adv=11, rect=[0,0,0,0]     ],
        [65] = [ off=[0,-38], adv=22, rect=[393,0,22,45] ],
        ...
    ],
]
FONT
```

Notes:

- store `rect` as `[x,y,w,h]` (`w=x2-x1`, `h=y2-y1`) for pico
  crops.
- `glyphs` keyed by codepoint (integer keys).
- module returns `FONT` (so `val chalk = require "fonts/chalk"`).

## Renderer module (`font.atm`)

Build glyph sub-layers from the sheet, then draw a string by
stepping the pen.

- init: `pico.layer.images(nil, key, FONT.sheet, ['!', ...])`
  with one crop per needed glyph (names = codepoints), OR
  `pico.layer.sub` per glyph (lazy).
- `draw(FONT, text, x, y, size)`:
    - `scale = size / FONT.size`
    - `pen = x`
    - for each byte `c` in `text`:
        - `g = FONT.glyphs[c]`
        - draw glyph sub-layer at
          `(pen + (g.off[0]*scale), y + (g.off[1]*scale))`
          sized `g.rect.w*scale by g.rect.h*scale`
        - `pen = pen + (g.adv * scale)`
- color: chalk glyphs are white on transparent; tint via
  `pico.set.effect [ color=... ]` if a color is needed.
- a `width(FONT, text, size)` helper enables center/left
  alignment (footer is left, labels centered).

## Integrate into the menu

Replace the `pico.output.draw.text(...)` calls in `menu.atm`:

| place         | font          | align  |
|---------------|---------------|--------|
| button label  | `chalk`       | center |
| footer infos  | `small`       | left   |
| footer help   | `small`       | center |

## Open questions

- Coordinates: render in pixels (glyph rects are px) and convert
  the menu's `%` positions via `W`/`H`, or add a `%` path.
- Sub-layer lifecycle: build glyph layers once at module load
  (detached, shared) to avoid re-creation churn.
- Whether `pico.layer.images` explicit-rect form or per-glyph
  `pico.layer.sub` is cleaner for slicing the sheet.
