# Plan: Rect + Image primitives

Factor two reusable widgets out of `menu.atm`'s `Button`:
an input hit-region (`Rect`) and a per-frame image draw (`Image`).
Code lives in the repo root (`.`).

## Status

- [x] designate feature: Rect + Image primitives
- [ ] execute locally (offscreen) + screenshot
- [ ] trace behavior
- [x] specify feature (this plan)
- [x] compare C++ and Céu versions + identify patterns
- [ ] update plan
- [ ] propose in Atmos
- [ ] implement

## Motivation

`Button` (`menu.atm`) inlines a ~30-line block: hover state-machine
(`mouse.motion until vs.pos.rect`), per-state image draw, and
inside-click hit-test. Two orthogonal concerns are tangled there:
*input* (is the pointer over / clicking this rect) and *output*
(draw an image at a rect). Split them.

## Rect (r)

An **invisible** hit-region over rect `r`. Input-only — it never
draws. Tracks the pointer/rect relationship and exposes it two ways.

Interface (`r` carries `x,y,w,h` plus live state):

| member       | kind   | meaning                              |
|--------------|--------|--------------------------------------|
| `x,y,w,h`    | data   | the rect geometry (passed in)        |
| `over`       | state  | pointer is currently over the rect   |
| `click`      | event  | a click landed inside the rect       |

Events emitted (to the direct parent):

- `:rect.over [true|false]` — on each enter/leave transition,
  carries the new value.
- `:rect.click` — on a click inside.

### Decisions

- **over, not hover.** `Rect` is geometric; `over` names the fact
  ("pointer is over the rect"), pairs with `click`, and matches DOM
  `mouseover`. `hover` (the highlight + tick feedback) is the
  consumer's concern. `menu.atm`'s current `hover` is exactly the
  duplicated code being removed, so no consistency cost.
- **over = state, click = event.** `Button` *polls* `over` in its
  draw loop (to overlay the highlight) and *reacts* to `click` (a
  one-shot action). Expose each in the shape its consumer uses.
- **emit scope.** Emit to the direct parent (`@1`); the parent knows
  which rect by context (it spawned exactly one), so no `id` payload
  is needed. `Button` re-emits `:Button [id]` upward (`@2`) as today.
- **click = down-then-up inside (open).** Current code fires on
  `:mouse.button.dn` inside. A true click (press *and* release
  inside) avoids drag-in/drag-out misfires. Behavior change — confirm
  before adopting.

## Image (r, path)

Dumb per-frame draw. Keep `r` first to match `Rect (r)`.

```
val task Image (r, path) {
    loop on :draw {
        pico.output.draw.image(path, r)
    }
}
```

Always draws while alive; gating (e.g. highlight only while `over`)
is the caller's job, not `Image`'s.

## Button, refactored

`Button` collapses to a composition of the two primitives:

```
val task Button (id, text, r) {
    pin rect = spawn Rect(r)
    par {
        spawn Image(r, BUTTONS.image)              ;; base, always
    } with {
        loop on :draw {                            ;; highlight while over
            if rect.over {
                pico.output.draw.image(BUTTONS.hover, r)
            }
            pico.set.pencil [ color="white" ]
            pico.output.draw.text(text, r)
        }
    } with {
        loop {                                     ;; tick on enter
            await :rect.over until it
            pico.output.sound(BUTTONS.sound)
        }
    } with {
        loop on :rect.click {                      ;; signal the menu
            emit @2 :Button [id]
        }
    }
}
```

The hover state-machine + per-state draw + hit-test become a couple
of `await`s and a `rect.over` read.

## Control-flow patterns

| # | Pattern      | Where                                        |
|---|--------------|----------------------------------------------|
| 1 | FSM          | `Rect` over: enter -> `over=true` -> leave   |
| 3 | Dispatching  | `Button`'s `loop on :draw` fans the frame    |
| 4 | Lifespan     | `Button` spawns `Rect` + `Image`; ending it  |
|   |              | kills both                                   |
| 5 | Signaling    | `Rect` `emit :rect.click/over` -> `Button`   |
|   |              | -> `emit :Button [id]` -> menu               |

## Open questions

- `Rect` invisible (input-only) vs. owning a debug stroke? Plan: keep
  it invisible; `Image`/label do all drawing — keeps the two
  orthogonal.
- `click` on down-only (current) vs. down+up-inside (stricter).
- Does `menu.atm`'s `Button (id, text, x, y)` change signature to
  take `r` directly, or keep building its rect from `x,y` + image
  dims internally? Leaning: keep building `r` inside, pass it to both
  primitives.
