# Dev plan — gesture axis assignment

Successor to `DEV_PLAN.md`, which is complete except roadmap item 8 (multiple
`SplatMesh` instances), deferred deliberately. Same rules apply: read `NOTES.md`
first, honour the uniform invariant, read `V` and never `P`, measure rather than
eyeball, and record numbers in the commit message.

One phase. It can land independently of multi-instance.

---

## The problem

`Adjust` binds up to two parameters to the drag axes via a chip strip that
enumerates `SPEC`. When that was built, `SPEC` had 18 entries. Region alone
added 36, and with saturation and orbit it now sits around 55–60. Cycling
**unbound → X → Y → unbound** on a single tap is a good interaction; scrolling
55 chips to find the one you want on a 390px screen is not.

The strip is not the wrong widget, it is doing two jobs. It is a reasonable
*switcher* and a bad *picker*, and only the picking half breaks.

---

## Verify this before designing around it

If the strip enumerates `SPEC`, it contains chips for all 36 region parameters
regardless of which slot tab is currently visible — all four `.rgpanel`s are
permanently in the DOM but only one is shown. That means there are chips that
bind and edit rows the user cannot see, and the strip and the panel disagree
about what exists.

Check it first. If it is real, it is an argument for removing the strip rather
than reorganising it, and the fix falls out of the design below for free rather
than needing its own work.

---

## Design

### Assignment moves to the lane header

The lane header is already where per-parameter routing decisions live: the audio
band/depth widget and the Phase 3 `Mask` control both sit there, shown
contextually for the focused parameter. Axis binding is the same kind of
decision and belongs beside them.

- Two toggles, `X` and `Y`, acting on the currently focused parameter.
- Keep the existing displacement rule: binding an axis already held by another
  parameter displaces the previous holder rather than refusing.
- Focusing still must not mutate. Tapping a parameter name binds nothing.

This costs no new navigation — you find the parameter through the accordion
exactly as you already do for envelopes and audio routing — and it scales to any
`SPEC` size because there is no parallel list to scroll.

**Not row-level buttons.** The row grid is already carrying label, output, `↺`,
`◆` and slider, with `.key.ghost` placeholders in the transform and region rows
specifically to keep those columns aligned across generated entries. Two more
per-row controls means touching that grid for ~57 rows to serve a control most
of them will never use.

### The strip is replaced, not reorganised

Delete the `SPEC`-enumerating chip strip. In its place, pinned to the viewport
and visible only while `Adjust` is on:

- Two chips showing the bound parameter names and their live `V` values. This is
  the half of the strip actually worth keeping — you need to see what you are
  dragging without opening the sheet, which is the whole point of gesture mode.
- Tapping a chip clears that axis. Nothing else; do not overload it.
- `pointer-events: none` on the container with the chips themselves as the only
  hit targets, so the readout never eats an orbit drag.

### Bindings stay visible in the sheet

The bound axis letter renders on the parameter's own row in `--edit` violet,
alongside the existing automation affordances. `--edit` already means "armed
envelope, changed-from-default, gesture binding" per the colour table, so this
is consistent rather than a fourth meaning. A distinct glyph, not a recoloured
existing one — the row must not become ambiguous about whether it is armed or
bound.

### Per-axis sub-range

Sweeping a parameter's full range across 390px of travel is too coarse to play.

```
GEST = { x: {id, lo, hi, invert}, y: {id, lo, hi, invert} }
```

- `lo`/`hi` are fractions of the parameter's own slider travel, defaulting to
  `0` and `1`, so today's behaviour is the default and nothing changes for
  anyone who does not touch it.
- Plain JS routing state, **not `SPEC`** — same treatment as `AUD`'s band/depth
  and region `shape`/`mode`/`invert`. It routes; it is not itself a value.
- Two small numeric fields per axis in the lane header, behind the same
  disclosure discipline as audio shaping: the axis toggle is the thing anyone
  touches, the range is refinement.
- `invert` per axis, which costs nothing and saves rebinding to get a direction.

### The write path does not change

Drags continue to write through the slider's own `input` event, clamped to
`[lo, hi]`. That is the single path already handling auto-key on armed
envelopes, dirty marking, `dpr`/`dur` side effects and `pushUniforms`. Do not
add a second one — this decision is why gesture-vs-envelope precedence is
already a solved problem rather than an open question.

### Preset

Add a `gesture` block: two axis bindings with their ranges and inverts.
Additive, absent means unbound, no version bump — still v5. Same
clamp-field-by-field treatment on load as every other preset value.

---

## Work order

1. Confirm or rule out the hidden-slot chip inconsistency. Record the finding
   either way.
2. Lane-header `X`/`Y` toggles wired to the existing binding logic, strip still
   present. Both paths working at once, briefly.
3. Viewport readout chips.
4. Delete the strip and its `SPEC` enumeration.
5. Row axis-letter indicator.
6. Sub-range and invert.
7. Preset block, mobile pass, measurements.

Steps 2–4 in that order specifically: build the replacement, prove it, then
remove the old path. Not the reverse.

---

## Acceptance

All measured, on the butterfly, warmed properly — stable **and** non-zero, per
the warm-up hazard in NOTES.

- No element enumerating `SPEC` for gesture binding remains in the DOM. Assert
  it, do not inspect it.
- Focus `dAmt`, bind X, full-stage drag: `V.dAmt` traverses the full slider
  range, endpoint to endpoint, within one step.
- Binding X to a second parameter unbinds the first. Exactly one parameter per
  axis at all times — assert in the debug harness.
- **Regression:** dragging a bound parameter whose envelope is armed still adds
  a keyframe, identically to dragging its slider. This is existing verified
  behaviour and the thing most likely to break.
- `lo = 0.4, hi = 0.6`: a full-stage drag spans exactly that sub-range, within
  one step at each end.
- `invert`: a full drag traverses the same values in reverse, pointwise.
- No parameter in a non-visible region slot can be focused, and therefore none
  can be bound. This is the hidden-slot inconsistency closing itself; confirm it
  rather than assuming it.
- At 390×844 the lane header still scrolls horizontally with no stage overflow,
  in the worst case: a maskable parameter with an armed envelope, an audio route
  and both axis toggles present.
- With `Adjust` off, the readout chips do not intercept an orbit drag — verify
  by dragging through where a chip sits and confirming the camera moves.
- Preset round-trip byte-exact with both axes bound, sub-ranged and inverted.

---

## Risks

| risk | mitigation |
|---|---|
| Lane header crowds at 390px with mask + audio + gesture all contextual | Measure the worst case explicitly, per acceptance; sub-range behind a disclosure |
| Removing the strip hurts `Adjust` discoverability | Viewport chips persist while the mode is on; empty state names the mechanism rather than showing nothing |
| Row axis letter reads as ambiguous against armed/dirty, all `--edit` | Distinct glyph, not a recoloured existing one; check all three states on one row |
| Second write path creeps in for sub-range clamping | Clamp inside the existing `input` handler, not before it |
| Auto-key regression goes unnoticed | It is an explicit acceptance line, not a spot check |

---

## Interaction with multi-instance

Roadmap item 8 will make `SPEC` per-mesh or introduce mesh selection. `GEST`
holds parameter ids, so it will need the same scoping decision as `AUD` and the
region slots when that lands — bindings either follow the selected mesh or are
global. **Do not pre-solve it.** Note it in NOTES when this phase lands so the
multi-instance work has a list of things holding parameter ids rather than
discovering them one at a time.

---

## Deferred

**Saved XY pair slots.** Assignment happens rarely, switching happens
constantly, and a few named pairs would be the real performance feature. It is
not this work — get the assignment mechanism right first, and reuse the preset
machinery for pairs rather than inventing a second store.
