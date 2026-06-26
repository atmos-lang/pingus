# Plan: Rect + Image primitives

Factor two reusable GUI widgets out of `menu.atm`'s `Button`:
an input hit-region (`Rect`) and a per-frame image draw (`Image`).
Both live as **global tasks** in a new `gui.atm`.

## Status

- [x] designate feature: Rect + Image primitives
- [ ] execute locally (offscreen) + screenshot
- [ ] trace behavior
- [x] specify feature (this plan)
- [x] compare C++ and Ceu versions + identify patterns
- [x] verify APIs (pointer query, emit target)
- [x] propose in Atmos (Rect/Image drafted in chat)
- [x] implement `Rect` + `Image` in `gui.atm`
- [~] Button refactor: out of scope (future)

## Motivation

`Button` inlines a ~30-line block tangling two orthogonal concerns:
*input* (is the pointer over / clicking this rect) and *output*
(draw an image at a rect).
`Rect` owns input, `Image` owns output.

## Rect (r, target)

Invisible, input-only hit-region over rect `r`.
Never draws.
Tracks the pointer/rect relationship as live state on `r` and emits
transitions.

### State (written onto `r`)

| field                       | kind | value                                |
|-----------------------------|------|--------------------------------------|
| `x,y,w,h`                   | data | the rect geometry (passed in)        |
| `over`                      | bool | `true` while pointer is over         |
| `left` / `right` / `middle` | bool | mirrored mouse buttons; only         |
|                             |      | meaningful while `over` is `true`    |

No `:dn`/`:up` enum: `Rect` mirrors the raw button booleans the
event already carries (`pico.c` `L_set_mouse`), so no leaf
extraction or `match` is needed.

### Events (payload `[rect=r]`)

- `:rect.over` on each enter/leave; consumer reads `it.rect.over`.
- `:rect.click` on any button change while over; consumer reads
  `it.rect.left` etc. (a press inside = `await :rect.click until
  r.left`).

### Decisions

- **Signature** `Rect (r, target)`.
  `target` is a task ref; omitted (`nil`) falls back to `:parent`
  via `tgt = target || :parent`.
- **Init from current input.**
  Query `pico.get.mouse('%')` so `over`/`click` are valid at `t=0`;
  `click` may start `:dn` if the button is already held over `r`.
- **Two par branches.**
  `:mouse.motion` handles `over` only (emit on change);
  `:mouse.button until r.over` mirrors `left/right/middle` while over.
- **Buttons valid only while over.**
  Motion does not touch the button fields, so they hold their last
  in-rect value; the consumer gates on `r.over` (a menu `Button`
  reads `r.over` for its highlight anyway).
- **emit scope.**
  `Rect` emits to `target||:parent`; `Button` re-emits `:Button`
  upward (`@2`) as today.

## Adoption

- `Image`: logo (`menu.atm`) + board backdrop (`intro.atm`).
  `PAGES@i.img` stays inline for now (dynamic; revisited in the
  intro restructure step).

## Image (path, r)

Dumb per-frame draw: `loop on :draw { draw.image(path, r) }`.
Always draws while alive; gating is the caller's job.

- Arg order `(path, r)` mirrors `pico.output.draw.image(path, r)`.
- `r` carries `w,h`; `draw.image` scales to it, but since `Button`
  builds `r` from `get.image` dims the scale is identity (no pixel
  change vs. today).

## Button (out of scope)

NOT part of this plan — `menu.atm` stays as-is.
Kept here only as the intended future consumer / validation sketch.

Would collapse to a composition:

- build `r` from `x,y` + image dims (signature unchanged).
- `watching (await :rect.click until r.left) { par { ... } }` ends
  on a left-press inside, returns `id` (replaces `await
  :mouse.button.dn until inside`).
- par branches: `spawn Rect(r)`; `spawn Image(base, r)`;
  `loop on :draw` highlight-while-`r.over` + label; tick on enter
  (`await :rect.over until r.over`).
- text keeps its own `h=0.05` rect (not `r`).
- call-sites `menu.atm:120-124` untouched.

## API check (verified)

| item              | result                                              |
|-------------------|-----------------------------------------------------|
| pointer query     | `pico.get.mouse('%')` -> `{x,y,left,right,middle}`  |
| emit target       | `emit @(expr) (...)`; expr may be a task-ref var    |
| emit payload      | record form: `(:tag [field=...])`                   |
| hierarchical tags | `await :rect.click` catches `.dn` and `.up`         |
| default params    | not supported -> `target` arg + `||` fallback       |

Sources: `pico-sdl/src/input.c`, `pico-sdl` get-image plan,
atmos manual + `HISTORY.md`.

## Control-flow patterns

| # | Pattern     | Where                                          |
|---|-------------|------------------------------------------------|
| 1 | FSM         | `Rect` over/click state on `r`                 |
| 3 | Dispatching | `Button`'s `loop on :draw` fans the frame      |
| 4 | Lifespan    | `Button` spawns `Rect`+`Image`; click ends par |
| 5 | Signaling   | `Rect` emit -> `Button` -> emit `:Button [id]` |

## Open questions

- [x] click encoding: dropped `:dn`/`:up` + sub-tags; mirror raw
      `left/right/middle` booleans instead (no `match`, no leaf).
- [x] `Rect` invisible: yes; all drawing is `Image`/label.
- [x] pointer query + emit-target: verified (see API check).
- [x] spawn/enter-while-held: buttons are valid only while over and
      mirror live state; no stale-`:dn` edge anymore.
- [x] global-task form: bare `task Rect (...)` (named, no `val`/`set`
      = global, since `T = task ()` is written `task T ()`).
- [ ] confirm `||` operator and `loop on :ev until cond` in real
      Atmos.

Plan: 260625-rect-image.md
