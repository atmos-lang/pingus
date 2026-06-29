# Plan: Fade in window space

Make the `Fade` screen-wipe (`gui.atm`) run in **screen/window space**,
so its growing/shrinking clip rectangle stays screen-aligned no matter
what the incoming/outgoing screen does to `:world`.

## Problem

`Fade` rides the current layer (`:world`).
It looks right while `:world` maps 1:1 to the window (the menu and its
sub-screens).
But the worldmap (`World`) retargets `:world` to the 1920x1200 map
(`scene.dim` = map + a camera `source`).
From then on `Fade`'s `%` clip is measured in **map space**, so the
wipe rectangle is offset/scaled instead of a clean screen rectangle.

Root cause: `pico.set.scene [clip=..]` and `pico.layer.screenshot`
(src omitted) act on the **current layer** -- there is no layer
argument, so the geometry follows whatever `:world` currently is.

## Tried -- did NOT work

Bracketing each `Fade` op with `pico.set.layer(:window)` + restore and
`screenshot(.., :window)`.
This did not produce a correct wipe -- the clip on `:window` did not
compose with the child `:world` render as expected.
A bare layer-switch is insufficient; do not repeat it.

## To investigate

- pico compositing: does a `clip` on a parent layer clip the render of
  its child layers, or only direct draws on the parent?
  When/where is `:world` composited into `:window`, and where does
  `draw.layer(:snap)` on `:window` land relative to that?
- a **dedicated overlay layer**: a window-sized child of `:window`
  (like `:hud`) that holds the snapshot and owns the clip, independent
  of `:world`'s scene -- the whole wipe happens on the overlay and
  `:world` is untouched.
- **two-snapshot crossfade**: snapshot the old screen and the new
  screen, then reveal/blend on a window-space overlay, with no live
  clipping of the running screens.

## Constraints

- `Fade` is shared: must keep working for menu sub-screen fades
  (`:world` == window) AND menu -> world (`:world` == map).
- no per-screen knowledge inside `Fade` (it only gets `cur`, `Next`).

## Status

- [ ] determine pico clip/composite semantics (parent clip vs child
  render)
- [ ] pick an approach (overlay layer vs two-snapshot vs other)
- [ ] implement; confirm the wipe is screen-aligned over the map
  (slow the wipe to inspect -- the xvfb clock is unthrottled, so it is
  otherwise near-instant)
