# Plan: Object + Image primitives

Factor two reusable GUI widgets out of `menu.atm`'s `Button`:
an input hit-region (`Object`) and a per-frame image draw (`Image`).
Both live as **global tasks** in a new `gui.atm`.

## Status

- [x] designate feature: Object + Image primitives
- [ ] execute locally (offscreen) + screenshot
- [ ] trace behavior
- [x] specify feature (this plan)
- [x] compare C++ and Ceu versions + identify patterns
- [x] verify APIs (pointer query, emit target)
- [x] propose in Atmos (Object/Image drafted in chat)
- [x] implement `Object` + `Image` in `gui.atm`
- [x] Button refactor: Object + Image + Text (do/await scopes)

## Motivation

`Button` inlines a ~30-line block tangling two orthogonal concerns:
*input* (is the pointer over / clicking this rect) and *output*
(draw an image at a rect).
`Object` owns input, `Image` owns output.

## Object (rect, target)

Invisible, input-only hit-region over `rect`.
Never draws.
Exposes its rect + live pointer state via `pub` and emits
transitions. The input `rect` is read-only — state lives on `pub`.

### State (on `pub`)

| field                       | kind | value                                |
|-----------------------------|------|--------------------------------------|
| `rect`                      | data | the geometry (read-only, passed in)  |
| `over`                      | bool | `true` while pointer is over         |
| `left` / `right` / `middle` | bool | mirrored mouse buttons; only         |
|                             |      | meaningful while `over` is `true`    |

`pub` is built once via a record literal + `where` (no outer sets).
No `:dn`/`:up` enum: `Object` mirrors the raw button booleans the
event already carries (`pico.c` `L_set_mouse`), so no leaf
extraction or `match` is needed.

### Events (payload `[obj=pub]`)

- `:o.over.on` / `:o.over.off` on enter / leave.
- `:o.click.dn` / `:o.click.up` on press / release while over.

Sub-tags let a consumer `await` the exact transition with no `until`
payload filter (e.g. `await :o.over.on`, `loop on :o.click.dn`).
`pub` still carries geometry + button state for richer reads.
`:o.click.*` is discriminated by `match it { :mouse.button.dn => …;
:mouse.button.up => … }` on the raw event.

### Decisions

- **Signature** `Object (rect, target)`.
  `target` is a task ref; omitted (`nil`) falls back to `:parent`
  via `tgt = target || :parent`.
- **Init from current input.**
  Query `pico.get.mouse('%')` so `over`/buttons are valid at `t=0`.
- **Two par branches.**
  `:mouse.motion` handles `over` only (emit on change);
  `:mouse.button until pub.over` mirrors `left/right/middle`.
- **Buttons valid only while over.**
  Motion does not touch the button fields, so they hold their last
  in-rect value; the consumer gates on `pub.over` (a menu `Button`
  reads it for its highlight anyway).
- **emit scope.**
  `Object` emits to `target||:parent`; `Button` re-emits `:Button`
  upward (`@2`) as today.

## Adoption

- `Image`: logo (`menu.atm`), board backdrop + per-page image
  (`intro.atm`). The page image is re-spawned each page via a
  `loop _, page in PAGES { ... await :Next }` (per-iteration spawns
  die on advance — no `watching :Next` needed).
- `Text`: TITLE + page lines (`intro.atm`), footer help + footer
  infos (`menu.atm`), Blank (`main.atm`). Page lines use a per-page
  `tasks(#page.text)` pool; footer infos are one `Text` per line in
  a `tasks()` pool (loop-body spawns need a pool to outlive the
  iteration).
- `Rect` (filled-rect draw, the `draw.rect` analog) in `gui.atm`;
  used for Blank's backdrop (`main.atm`) and the footer black bar
  (`menu.atm`).

## Image (path, r)

Dumb per-frame draw: `loop on :draw { draw.image(path, r) }`.
Always draws while alive; gating is the caller's job.

- Arg order `(path, r)` mirrors `pico.output.draw.image(path, r)`.
- `r` carries `w,h`; `draw.image` scales to it, but since `Button`
  builds `r` from `get.image` dims the scale is identity (no pixel
  change vs. today).

## Button (done)

`menu.atm`'s `Button` now composes `Object` + `Image` + `Text`:

- `spawn Object(rect)` is the input; emits `:o.over`/`:o.click` to
  the Button (its `:parent`).
- base / highlight / label spawned once at top in z-order
  (`base < highlight < label`); the highlight is `toggle`d off and
  flipped on/off by `:o.over` transitions (freeze semantics, same as
  `Fade`), so its z-slot is preserved — no rebuild.
- hover: `await :o.over.on` / `:o.over.off` (no filter); click:
  `loop on :o.click.dn { emit @2 :Button [id] }`.
- signature `(id, text, x, y)` unchanged → call-sites untouched.

Pattern: `do { ...spawns; await(x) }` replaces `watching x { ...;
await(false) }` — the block scope finalizes the spawns on `x`.

## API check (verified)

| item              | result                                              |
|-------------------|-----------------------------------------------------|
| pointer query     | `pico.get.mouse('%')` -> `{x,y,left,right,middle}`  |
| emit target       | `emit @(expr) (...)`; expr may be a task-ref var    |
| emit payload      | record form: `(:tag [field=...])`                   |
| hierarchical tags | `await :o.click` catches sub-tags                   |
| default params    | not supported -> `target` arg + `||` fallback       |

Sources: `pico-sdl/src/input.c`, `pico-sdl` get-image plan,
atmos manual + `HISTORY.md`.

## Control-flow patterns

| # | Pattern     | Where                                          |
|---|-------------|------------------------------------------------|
| 1 | FSM         | `Object` over/click state on `r`                 |
| 3 | Dispatching | `Button`'s `loop on :draw` fans the frame      |
| 4 | Lifespan    | `Button` spawns `Object`+`Image`; click ends par |
| 5 | Signaling   | `Object` emit -> `Button` -> emit `:Button [id]` |

## Open questions

- [x] click encoding: dropped `:dn`/`:up` + sub-tags; mirror raw
      `left/right/middle` booleans instead (no `match`, no leaf).
- [x] `Object` invisible: yes; all drawing is `Image`/label.
- [x] pointer query + emit-target: verified (see API check).
- [x] spawn/enter-while-held: buttons are valid only while over and
      mirror live state; no stale-`:dn` edge anymore.
- [x] global-task form: bare `task Object (...)` (named, no `val`/`set`
      = global, since `T = task ()` is written `task T ()`).
- [ ] confirm `||` operator and `loop on :ev until cond` in real
      Atmos.

Plan: 260625-rect-image.md
