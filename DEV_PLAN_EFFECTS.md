# Dev plan — camera rework and the effect family

Successor to `DEV_PLAN.md` (complete bar item 8) and `DEV_PLAN_GESTURE.md`.
Same rules: read `NOTES.md` first, honour the uniform invariant, read `V` never
`P`, measure rather than eyeball, record numbers in the commit message.

Phase 1 is blocking — it prevents authoring camera moves at all, and several
later phases cannot be evaluated without a movable camera.

---

## Two findings before any work starts

### Audio shaping already exists — this is a discoverability bug

Per-route `attack` / `release` / `lag` / `gain` / `floor` / `curve` are
implemented, per-parameter, cached offline, and measured working (see the Audio
shaping section of NOTES, including the identity-at-neutral proof and the ±0.25s
lag verification).

Inspected in the running build: `#audShapePanel` carries class `hide`
(`display:none`) and is opened by `#audShapeToggle`, labelled **`Shape ▾`**,
65×19px, sitting in the lane header alongside `Off` / `Smooth` / `Clear`. The
lane header itself only appears once a parameter is focused.

**Do not rebuild this.** Phase 2 makes it findable.

### The camera bug is the `Go A` / `Go B` bug in a new form

Adding one camera keyframe locks the camera. `driveCamera()` writes
`camera.position` / `controls.target` **every frame** while in Keyframes mode,
so manual orbit input is overwritten on the next frame. With exactly one key the
curve is constant at all `t`, so the camera is pinned to that pose permanently —
and there is no way to move it in order to author key two.

The mode select fixed "two drivers write the camera in one frame" by making one
driver always win. That is correct for playback and wrong for authoring: the
same guarantee that prevents the conflict also prevents the user from ever
being in control.

---

## Phase 1 — Camera architecture: evaluate on playhead change, not per frame

### The change

Split evaluation from ownership.

- **`evaluateCamera(t)`** — a pure function returning `{pos, tgt, fov}` for the
  current mode. No side effects, writes nothing. Keyframes samples the
  Catmull-Rom curves; Orbit computes its position from `t`; Manual returns null.
- **The live camera is owned by manual interaction by default.** `frame()` no
  longer writes the camera unconditionally.
- The camera is written from `evaluateCamera()` **only on these events**:
  playhead changed since the last frame (playback or scrub), a camera key tick
  was clicked, camera mode changed, a preset loaded, `Reframe`, or the export
  loop stepping a frame.

Paused and idle, manual control works in every mode. Move the camera, `Add key`,
scrub, move again, `Add key`. That is the whole fix.

This is how every keyframe animation tool behaves, and it does **not**
reintroduce two drivers: there is still exactly one writer, it just fires on an
event rather than on every frame. State that explicitly in NOTES so the next
person does not "fix" it back.

### Dirty state

When paused, if the live camera differs from `evaluateCamera(playhead)` beyond a
small epsilon, the camera is *modified relative to the curve*. Surface it in
`--edit` violet, consistent with every other automation affordance. `Add key`
commits it; scrubbing discards it. Do not prompt, do not auto-commit.

### Mode-specific notes

- **Orbit** keeps its existing design — reads `controls.target` live, never
  writes it. Under the new rule it writes `camera.position` on playhead change,
  so manual orbiting while paused sticks, and playback takes it back.
- **Mode change force-writes.** Switching to Keyframes while paused should snap
  to the curve, otherwise the mode select appears to do nothing until you scrub.
- **Export force-writes every frame** rather than relying on change detection,
  so frame 0 is not skipped by a playhead that has not moved yet.

### Rejected alternative

A separate manual-offset transform that keyframes compose with. More
architecture, and it makes `Add key` ambiguous — does it store the pose or the
offset? Recorded here so it is not re-proposed.

### Acceptance

- One camera key present, paused: orbit drag moves the camera and it **stays**
  moved across at least 60 frames. This is the bug; assert it directly.
- Author a 3-key move end to end without touching the mode select: add, orbit,
  add, orbit, add. All three keys distinct.
- Scrub with 3 keys: camera follows the curve, no drift.
- **Regressions, all previously measured — re-run and match:** 5-key smoothness
  max/mean step ≤ 3× at `ease` 0 / 0.8 / 1 and both `camSmooth` states;
  `obTurns = 1` returns to start at distance 0.000000; `obTurns = -1` pointwise
  identical to `+1` reversed; orbit export wrap difference ~0.001; no frozen
  frame pairs in a 48-frame export in either driven mode.
- Correct mode still force-sets Manual on entry and restores on exit.
- Exactly one code path writes `camera.position` outside `OrbitControls`.
  Assert it.

---

## Phase 2 — Quick wins

Small, independent, no shared prerequisites. Land them in one pass.

### 2a — Audio shaping discoverability

- Mark `Shape ▾` dirty in `--edit` when any of the six fields is off default,
  reusing the existing `.dirty` convention so it reads as the same language.
- Summarise non-default values on the collapsed control, e.g.
  `Shape ▾ · lag +250 · rel 600`. The label alone does not tell anyone that lag
  and smoothing live behind it, which is exactly why they were not found.
- Verify the toggle is reachable at 390px — the lane header scrolls
  horizontally on mobile and `Shape ▾` is the rightmost control, so it may be
  off-screen with no scroll affordance. Measure, do not assume.

### 2b — Background colour

A `SELECTS` entry, not a slider: a small fixed palette (void / black / mid grey
/ white / chroma green) rather than a colour picker. Persisted in presets
(additive, it is a look property). Applied via `renderer.setClearColor`.

**The trap, and it is a real one:** every coverage figure in NOTES — the cull
table, the six-shape SDF reproduction, the Reveal migration curve, the masking
verification — was measured against `#08090b`. A non-default background
invalidates all of them silently, because coverage is a pixel count against a
known backdrop.

`window.__bench` must force the default background for the duration of any
probe and restore it afterwards. Do this in the same commit as the feature, not
later, or the next regression sweep produces confident nonsense.

Transparent background for compositing is **out of scope**: AVC has no alpha
channel, so it would need a separate export path. Note it against roadmap item 5
(depth pass) where it belongs.

### 2c — Splat brightness

A single `SPEC` slider in group `colour`, multiplying rgb. Fixed chain order,
documented in NOTES:

```
raw rgb → brightness → saturation → tint → alpha ops
```

Brightness before saturation so saturation operates on the corrected image, not
the other way round.

Call it **brightness, not exposure**. NOTES already establishes that the colour
in the graph is display-referred, so a multiply here is a gamma-space multiply.
A true exposure control would need linearise → scale → re-encode, which is a
different job and probably not wanted. Say so in NOTES so the distinction is not
lost.

Range `0..2`, default `1`. Do not clamp the output, for the same reason
saturation does not.

### 2d — Camera keys draggable on the timeline

Camera key ticks on `#camRow` are plain positioned divs; parameter envelope keys
are canvas-drawn and already draggable in both axes. Camera keys need horizontal
drag only — there is no Y value to edit.

- Drag retimes the key, re-sorts `CAMKEYS`, rebuilds `camPosCurve` /
  `camTgtCurve` on drop (not on every pointermove — measure if it matters, but
  rebuilding two `CatmullRomCurve3` per frame during a drag is the obvious
  regression).
- **Clamp to neighbours** rather than allowing reorder. Letting a key cross its
  neighbour changes which key is which mid-drag, which is confusing and makes
  the undo story worse.
- Respect `KEY_EPS` on drop: landing within epsilon of a neighbour should snap
  clear, not silently replace.
- Alt-click delete must not fire at the end of a drag. Gate on movement
  distance, not on pointerup alone.

### Acceptance

- `Shape ▾` shows dirty when any shaping field is off default; visible and
  tappable at 390×844.
- Background changes are visible, persist through a preset round-trip, and
  appear in an export.
- `__bench` probes return the **same** coverage figures with a white background
  set as with the default. This is the point of 2b's guard — verify it.
- Brightness 0.5 / 1 / 2 measured on luma; 1 matches the pre-change baseline
  within 1%.
- Dragging a camera key retimes it; the path through the remaining keys stays
  smooth by the existing max/mean ≤ 3× test.

---

## Phase 3 — Effects that need no new primitive

All pure `objectModifiers`, per-splat, using only what the graph already has.
Cheap, and they are the bulk of the visual payoff.

### 3a — Quantisation

`center = mix(center, floor(center/cell)*cell + cell*0.5, amount)`.

Build it as a family, not one control:

- Per-axis cell size, so slabs and columns are reachable, not only cubes.
- Snap in a **rotated frame**: rotate into a grid basis, snap, rotate back, so
  the grid is not welded to world axes.
- Optional snapping of `scales` and `quaternion` for true axis-aligned voxels
  rather than snapped positions of unchanged ellipsoids. Two distinct looks;
  both worth having, on separate toggles.
- **Single-axis quantisation** with a per-band offset from `hash(bandIndex)` —
  stratification and scanline displacement. Cheapest route to the datamosh
  register and probably the most-used setting here.
- Colour posterisation on the same panel.

`amount` on an envelope gives a dissolve into voxels.

**Known hazard:** many splats collapse onto identical centres, giving degenerate
sort order (tied depths) and heavy overdraw. Expect flicker as ties shuffle.
Measure it before deciding whether it needs mitigating — scaling splats up so
cells read solid may be enough, and deduplication would need per-splat
precomputed data this plan does not build.

### 3b — Curl noise replacing the `sin` field

Volume-preserving, so the cloud swirls instead of inflating. Strictly better
looking than the current field at comparable cost. Keep the existing `sin` field
as a selectable mode rather than deleting it — presets depend on it, and it is a
different, harder-edged look worth keeping.

### 3c — Region as attractor

Pull splats toward a region slot's SDF surface instead of removing them, amount
on a slider, signed so negative repels. Reuses the entire region system: four
slots, six shapes, existing masks, existing envelopes. Morphs a capture into a
sphere or box.

Highest ratio of look to work in this plan.

### 3d — Displace by luminance or hue

Offset along an axis (or along the splat's own short covariance axis, which
approximates the surface normal) by luminance or hue. Turns the capture's own
texture into relief.

### 3e — Index-phase wave

Per-splat time offset from `hash(index)` or from position along a chosen axis,
so effects sweep through the cloud rather than applying uniformly. Applies to
displace, quantise amount, and the attractor.

Pairs directly with the audio envelopes — this is the one that makes audio
reactivity read as motion through the object rather than as the whole thing
pulsing at once.

### Acceptance

Each effect measured in isolation on the butterfly, warmed properly (stable
**and** non-zero, per the warm-up hazard):

- Quantise `amount` 0 reproduces baseline coverage within 1%; `amount` 1 shows a
  measurably different bbox and coverage; intermediate values monotonic.
- Quantise flicker: 60 frames at a fixed camera and fixed `t`, coverage standard
  deviation recorded. This is a number to know, not necessarily a number to fix.
- Curl noise at matched amplitude changes bbox **less** than the `sin` field
  does — that is what volume-preserving means, and it is the check that it is
  actually curl and not just different noise.
- Attractor at full amount against a sphere region: bbox converges toward the
  region's own extent.
- Every new parameter swept before its slider range is settled, per the
  normalisation rule — a correctly normalised control can still put its whole
  useful band in 0.1% of travel.
- Frame time at 177k splats with all of 3a–3e active recorded against the
  existing baseline.

---

## Phase 4 — Spike: object-space camera uniform, and depth fade

**A spike, deliberately.** Depth fade is the cheapest possible consumer of this
primitive, so it exists to prove the primitive before DoF depends on it.

- Push `mesh.worldToLocal(camera.position.clone())` as a `vec3` uniform.
- **This must update in the render loop, not only on parameter change.** That is
  a departure from the current `pushUniforms()` discipline, which fires on
  edits. Decide explicitly whether the camera uniform piggybacks on
  `pushUniforms()` called per frame, or gets its own narrow path, and record the
  reasoning. Note the version bump is already measured free.
- **Ordering matters:** the uniform must be written *after* whatever set the
  camera this frame — after `driveCamera()` in the live loop, and after the
  camera is positioned in the export loop. Getting this backwards gives a
  one-frame-stale focus that will look like a Spark sort issue and waste a day.
- Then: depth fade / aerial perspective. Fade opacity or tint toward a colour
  with distance. Offer both planar (dot with view direction) and radial
  (distance) depth, since DoF will want the same choice.

### Acceptance

- Object-space camera position matches a CPU-computed reference across several
  model transforms, including a committed Correct-mode rotation.
- Depth fade responds to camera movement with **no** one-frame lag: assert via
  `__bench.settled()` with a camera move and a single settled probe.
- A 48-frame export with depth fade active shows the fade tracking the camera
  path, not frozen at frame 0's value.

---

## Phase 5 — Depth of field

The reason screen-space DoF fails on splats: there is no meaningful depth
buffer, because the cloud is alpha-blended and semi-transparent, so "the depth
at this pixel" is not well defined.

Per-splat defocus sidesteps it entirely, because **defocus of a Gaussian is
exactly another Gaussian**: convolving a splat's projected 2D Gaussian with a
circle-of-confusion kernel gives a Gaussian whose covariance is the *sum* of the
two. Defocus is addition of covariance, not a blur pass. It composites correctly
through the existing sort with no special handling, because nothing moves.

### `covSplats` first

Spark 2.0's `SparkRenderer({ covSplats: true })` stores the 3D covariance matrix
instead of scale+rotation, and covariance transforms can be applied directly by
splat modifiers. That is the correct primitive here: add σ²(d)·I to the
covariance and the result is exact.

Without it you are modifying scale+rotation, so isotropic blur applied to a thin
anisotropic splat over-blurs along its short axis — softer than correct on flat
surfaces, which is exactly where it shows.

Spark documents existing dyno pipelines as backward-compatible and
auto-converted. **Verify that against your own coverage tables before building
on it** — that is a vendor claim about a file with 36 region parameters, six
SDFs and a measured migration curve riding on it. Re-run the regression sweep
with `covSplats: true` and nothing else changed; if any figure moves, that is
the finding, and DoF waits.

### Then DoF

- CoC from `|depth − focus|`, with `focus`, `aperture` and a falloff curve as
  `SPEC` parameters — so rack focus and audio-driven focus pulls come free.
- **Peak alpha must drop by the determinant ratio** of old to new covariance, or
  defocused splats gain brightness and it reads as glow rather than blur. This
  is the single most likely thing to get wrong.
- Focus-pull to a region slot's centre as an option — reuses existing machinery
  and is more useful than a raw distance for authoring.
- **Honest limitation, record it in NOTES:** Gaussian bokeh only. No aperture
  shape, no cat's-eye, no shaped highlight discs. Those need custom splat
  shaders, and Spark 2.0 deprecated 0.1's arbitrary splat profiles, so it is the
  new shader path or nothing. Out of scope here.

### Acceptance

- `aperture = 0` reproduces baseline coverage and luma within 1%.
- A splat at the focus distance is unchanged; covariance determinant ratio 1.0.
- Total luma is conserved within 2% across an aperture sweep — this is the
  energy-conservation check, and it is the one that catches the alpha bug.
- Focus sweep across the capture's depth range moves the sharp region
  monotonically front to back.
- Frame time at 177k splats with DoF active, recorded. Camera movement already
  triggers regenerate via `viewChanged`, so the expectation is little or no
  additional cost — verify rather than assume.
- 48-frame export with an animated focus pull: no frozen frames, focus tracking
  the envelope.

---

## Phase 6 — Anisotropic smear

Depends on Phase 5's `covSplats` decision; do not attempt without it.

Stretch each splat along its own displacement direction, amount proportional to
displacement magnitude. Motion blur without motion — makes displace read as flow
rather than jitter, and combines with the index-phase wave for directional
sweeps through the cloud.

### Acceptance

- Zero displacement leaves covariance unchanged.
- Smear direction measurably aligns with displacement direction: sample a set of
  splats, compare the long covariance axis against the displacement vector.
- Total luma conserved within 2% across a smear sweep — same energy check as
  DoF, same failure mode.

---

## Risks

| risk | phase | mitigation |
|---|---|---|
| Camera rework regresses measured orbit/keyframe figures | 1 | Every prior figure is an explicit re-run line, not a spot check |
| "Fix" the event-driven write back to per-frame | 1 | Record the reasoning in NOTES as an invariant, not a detail |
| Background colour silently invalidates every coverage baseline | 2b | `__bench` forces default during probes, same commit |
| Quantise sort-tie flicker | 3a | Measure the variance; decide with a number |
| Camera uniform written before the camera moves | 4 | Explicit ordering requirement; symptom mimics a Spark sort bug |
| `covSplats` shifts existing measured behaviour | 5 | Full regression sweep on the flag alone before any DoF work |
| DoF energy not conserved, reads as glow | 5 | Luma conservation is an acceptance line |
| Effect count pushes `SPEC` past what the accordion handles | 3 | Already grouped; re-check rail height and the gesture-assignment header |

---

## Definition of done, per phase

1. Acceptance numbers in the commit message, not just observed.
2. `NOTES.md` updated in the same commit — new invariants, new measured tables,
   anything moved from assumed to verified.
3. Preset round-trip fingerprint still matches.
4. `?debug=1` regression sweep passes, warmed properly.
5. Loaded once from the Pages origin, not only localhost.
