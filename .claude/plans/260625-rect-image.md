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
- [ ] propose in Atmos (Rect/Image/Button drafted in chat)
- [ ] implement in `gui.atm` + `menu.atm`

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

| field      | kind  | value                                          |
|------------|-------|------------------------------------------------|
| `x,y,w,h`  | data  | the rect geometry (passed in)                  |
| `over`     | bool  | `true` while pointer is over the rect          |
| `click`    | enum  | `:dn` / `:up` while over; `nil` when not over  |

`r.click == nil` iff `r.over == false`.

### Events (payload `[rect=r]`)

- `:rect.over` on each enter/leave; consumer reads `it.rect.over`.
- `:rect.click.dn` / `:rect.click.up` on press/release while over.

### Decisions

- **Signature** `Rect (r, target)`.
  `target` is a task ref; omitted (`nil`) falls back to `:parent`
  via `tgt = target || :parent`.
- **Init from current input.**
  Query `pico.get.mouse('%')` so `over`/`click` are valid at `t=0`;
  `click` may start `:dn` if the button is already held over `r`.
- **Two par branches.**
  `:mouse.motion` recomputes `over` (emit on change);
  `:mouse.button` drives `click` gated by `r.over`.
- **Drag guards.**
  Enter forces `click=:up` (held-from-outside is not a click-in);
  a leave mid-press nils `click`, so the `:up` is skipped.
- **emit scope.**
  `Rect` emits to `target||:parent`; `Button` re-emits `:Button`
  upward (`@2`) as today.

## Image (r, path)

Dumb per-frame draw: `loop on :draw { draw.image(path, r) }`.
Always draws while alive; gating is the caller's job.

- Arg order `(r, path)` mirrors `Rect (r, ...)`.
- `r` carries `w,h`; `draw.image` scales to it, but since `Button`
  builds `r` from `get.image` dims the scale is identity (no pixel
  change vs. today).

## Button, refactored

Collapses to a composition:

- build `r` from `x,y` + image dims (signature unchanged).
- `watching :rect.click.dn { par { ... } }` ends on inside-click,
  returns `id` (replaces `await :mouse.button.dn until inside`).
- par branches: `spawn Rect(r)`; `spawn Image(r, base)`;
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

- [x] `click` down-only vs. down+up: emit both `.dn`/`.up`, push the
      policy to the consumer.
- [x] `Rect` invisible: yes; all drawing is `Image`/label.
- [x] pointer query + emit-target: verified (see API check).
- [ ] confirm `||` operator and `match it { :mouse.button.dn => ... }`
      sub-tag pattern in real Atmos.
- [x] global-task form: bare `task Rect (...)` (named, no `val`/`set`
      = global, since `T = task ()` is written `task T ()`).
- [ ] spawn-while-held edge: `click` may stay `:dn` past release.

Plan: 260625-rect-image.md
