# Splatification — dev plan: regions-as-masks, camera keyframes, orbit, audio shaping, saturation

Read `NOTES.md` first. This plan assumes every invariant in it, in particular:

- **The uniform invariant.** Any code path that writes a uniform routes through
  `pushUniforms()`, which ends in `mesh?.updateVersion()`. Nothing writes
  `U.*.value` directly.
- **`V`, never `P`.** Every downstream consumer reads the evaluated value.
- **Normalisation is necessary but not sufficient.** Sweep a new parameter and
  find where its response actually lives before settling its slider range.
- **Measure, do not eyeball.** Every acceptance criterion below is a number.

Work the phases in order. Phase 0 is a prerequisite for all of them; Phase 3
depends on Phase 2 and Phase 5 depends on Phase 4. Land each phase working and measured before starting the
next — several of these touch `SPEC`, and overlapping them will make a
regression impossible to attribute.

---

## Scope note — settled

"Orbit mode" means **auto-orbit as a camera drive mode**: a generated turntable
move about a chosen axis, driven by the timeline, as an alternative to
keyframing. It does **not** mean changing the up-axis of manual `OrbitControls`.
That was considered and cut.

One consequence to keep in mind: manual orbit therefore stays Y-up regardless of
how the capture is oriented. This is only a problem for a non-Y-up scan, and
`obAxis` already covers the auto-orbit half of it. If manual orbiting on a Z-up
capture turns out to be unworkable in practice, the up-axis select is a small
separate piece of work — do not pre-emptively build it.

---

## Phase 0 — Section accordion and `SPEC` groups

**Why first.** `SPEC` goes from 18 parameters to roughly 60 by the end of this
plan. Everything hangs off `SPEC` — reset buttons, envelope lanes, audio
routing, preset persistence — so the grouping mechanism has to exist before
anything adds to it, or Phases 1–5 each get retrofitted.

### Work

- Add a `group` field to each `SPEC` entry. Groups: `camera`, `trim`, `region`,
  `displace`, `colour`, `audio`, `output`, `viewport`.
- Render each group as a native `<details>` / `<summary>`. Native, not a custom
  widget: keyboard accessible for free, works inside the mobile bottom sheet,
  no state machine to own.
- The summary row shows the group name and a **count of driven parameters in
  that group** (armed envelope or audio route), using `--edit` violet per the
  colour table. A collapsed group must not hide the fact that something inside
  it is animating.
- A group whose contents are all at default gets no marker; any dirty parameter
  inside marks the summary, mirroring the existing per-row `.dirty` behaviour.
- Persist open/closed state in `localStorage`. **Not in presets** — same
  reasoning as `Mobile profile`: it describes your workspace, not the look.
- Default open: `camera`, `output`. Everything else closed.

### Acceptance

- All 18 existing parameters still reset, arm, route and persist. Run the
  existing preset round-trip: the fingerprint must match byte-exactly.
- Collapsing a group with an armed envelope shows the count on the summary.
- Rail height with all groups collapsed fits a 900px viewport without scrolling.

---

## Phase 1 — Saturation

Smallest item, and it exercises the one dyno form the notes still flag as
unverified.

### Work

In `buildModifier()`, after the tint multiply:

```js
const LUMA = dyno.dynoConst("vec3", new THREE.Vector3(0.2126, 0.7152, 0.0722));
const l = dyno.dot(rgb, LUMA);
const grey = dyno.mul(ONE3, l);
const sat = dyno.add(grey, dyno.mul(dyno.sub(rgb, grey), U.sat));
```

**Deliberately not `dyno.mix(grey, rgb, U.sat)`.** That is the vec3-operand form
of `mix` your notes list as still-unverified. Verify it separately as a one-line
experiment; if it works, simplify to `mix` and record it as confirmed in NOTES.
If it does not, the arithmetic above is already correct and you have your
answer. Do not let an unverified call site into the effect chain.

- Range `0..2`, default `1`. Above 1 oversaturates, `0` is monochrome.
- Confirm whether `rgba` at this point is linear or sRGB-encoded, because
  perceptual saturation differs between them. Record the finding in NOTES either
  way — this is the kind of thing that gets rediscovered otherwise.
- Do not clamp. Splat rasterisation blends; clamping per-splat is not the same
  as clamping the frame, and negative channels are a legitimate look.
- `sat` goes in `SPEC` group `colour`, so it inherits reset, envelopes, audio
  routing and presets with no extra work.

### Acceptance

Measured on the butterfly at `?debug=1`, via `window.__bench.settled()`:

- `sat = 0`: per-channel means converge to within 2% of each other.
- `sat = 1`: channel means match the pre-change baseline within 1%.
- `sat = 2`: channel spread (max mean − min mean) at least doubles vs `sat = 1`.
- No measurable frame-time change at 177k splats.

---

## Phase 2 — Model transform (scene correction)

Position, rotation and scale on the loaded capture, so a scan that came out of
reconstruction tilted, off-centre or in the wrong units can be corrected in the
tool rather than in SuperSplat and re-exported.

**Do this before Phase 3**, on workflow grounds: correct the capture once, then
author against it. Note the technical justification is weaker than it first
appears — regions live in object space and therefore travel *with* the model, so
a later correction does not orphan them. The genuine exception is a region meant
to stay world-aligned, such as a horizontal floor cut, which will tilt with the
model. If that turns out to matter, add a per-region `world-aligned` flag rather
than reworking the frame.

### Where the transform lives

Standard three.js `mesh.position` / `mesh.rotation` / `mesh.scale`. No dyno, no
`worldModifiers` — Spark respects the mesh's world matrix, and putting the
correction in the shader would mean object space no longer matches what the
panel says.

This replaces the hardcoded `mesh.quaternion.set(1, 0, 0, 0)` Y-down fix. That
value becomes the **default** of the rotation controls rather than a constant,
so it is now visible and adjustable instead of being an invisible assumption
baked into the loader. Captures that are not Y-down have never been correctable
until now.

### Locked behind an explicit mode

The transform is **not** live-editable from the rail. `Correct` is a mode:
enter it, adjust, commit or cancel. Outside the mode the transform is locked.

This is the same reasoning as `Orbit` vs `Adjust` in NOTES — an explicit mode is
duller and much harder to get wrong than something that can be nudged by
accident. Three specific gains here:

- **The transform cannot be animated**, which removes the frame-to-frame
  normalisation wobble by construction rather than by care.
- **One write path.** Only the commit touches the transform, so the
  invalidation question below has exactly one place to be answered.
- **A gizmo becomes viable.** Use `TransformControls` from `three/addons` with
  the standard `dragging-changed` interlock disabling `OrbitControls` mid-drag.
  Dragging a capture into level beats three Euler sliders. Keep the numeric
  fields alongside it for exact values and for touch.

Interlocks, stated rather than assumed: entering `Correct` disables gesture
`Adjust` and forces camera drive to `Manual`. Leaving it restores the previous
camera drive mode. Exporting while in `Correct` mode is refused with a message.

`Cancel` restores the transform as it was on entry — so the mode is safe to
poke at, which is the point of having it.

### Controls

| param | range | default | notes |
|---|---|---|---|
| `mdRotX` | −180..180° | 180 | the old Y-down correction lives here |
| `mdRotY` | −180..180° | 0 | |
| `mdRotZ` | −180..180° | 0 | |
| `mdPosX/Y/Z` | −2..2 | 0 | multiples of object-space `radius` |
| `mdScale` | 0.1..10 | 1 | log-mapped, uniform only |

- Euler order `XYZ`, stated in the UI. This is a correction tool, not an
  animation rig; gimbal lock at ±90° on the middle axis is acceptable and not
  worth quaternion controls. The gizmo writes through these fields so both
  routes stay in sync and there is still only one source of truth.
- Uniform scale only. Non-uniform scale on a splat cloud distorts the gaussians
  themselves in ways that are rarely what anyone wants, and it would break the
  region radius/scale semantics measured in Phase 3.
- `mdScale` is log-mapped for the same reason `cullS` had to be: a linear
  0.1..10 slider puts everything useful in the first 10% of travel. Sweep it
  and confirm before settling the mapping.
- A `Reset transform` button restoring all seven to default, alongside the
  existing per-row reset buttons.

### The catch — normalisation basis must stop following the transform

`reframe()` currently computes `radius` and `bounds` from the **world-space**
bounding box. Every normalised slider multiplies through those. If the transform
feeds the same box, then rotating the model silently changes the meaning of
`dAmt`, `cullS`, region sizes and everything else — and if the transform is ever
animated, they wobble frame to frame.

Split the two uses of the bounding box, which have always been different jobs:

- **Normalisation basis** — object-space bbox, computed once at load, before any
  transform. `radius` and `bounds` come from here. Slider meanings then never
  move.
- **Camera framing** — world-space bbox, recomputed whenever the transform
  changes, so `reframe()` still frames what is actually on screen. Keep the
  aspect-aware `min(vFov, hFov)` fix from NOTES.

This is a behavioural change to existing code. Verify that every current
parameter still measures the same before and after, with the transform at
default — if any of them shift, the split is wrong.

### The catch the mode does not fix

The lock stops the transform from *changing continuously*, but committing one
still reinterprets any world-derived normalisation at that instant. It turns a
silent drift into a discrete event, which is a real improvement and not a fix.
The object-space basis freeze below is what makes the discrete event harmless
too. Do both — they are cheap and they fail in different directions.

### The other catch — does a transform change alone invalidate?

A mesh transform does not change the baked object-space splat data, so no
regenerate should be needed; but it does change the view relationship, so a
re-sort is. **Do not assume either way.** This is precisely the failure mode
that cost a day already: the change appears not to work, then pops in when
something unrelated moves the camera.

Test it explicitly — set a rotation with the camera perfectly still, via
`?debug=1` and `window.__bench.settled()`, and confirm the render changes. If it
does not, route the transform writes through `pushUniforms()` like everything
else and record the finding in NOTES.

### Not animatable, and therefore not in `SPEC`

A locked transform cannot carry an envelope, so `mdRot*` / `mdPos*` / `mdScale`
stay out of `SPEC`. That means no free reset buttons, no free preset
persistence, no audio routing — roughly 30 lines of hand-wiring for reset and
save/load. Accept the cost; it buys the wobble elimination above.

Spinning the model rather than the camera was considered as a look and cut. It
is achievable in Phase 5 by orbiting the camera, which does not require an
animatable model transform and does not put every normalised parameter
downstream of an animated basis.

If an animated model transform is ever genuinely wanted, it should be a
**separate** animatable pose layered on top of the locked correction, not the
correction itself becoming live. Keep those two concepts apart.

### Preset

Add a `transform` block. It is a property of the capture, so it belongs in the
file — unlike `Mobile profile`. Note that presets reference the asset by name
only, so a transform saved against one capture and loaded against another is
meaningless; it is harmless, and guarding against it is not worth the code.

Additive and load-forgiving: a v3 preset with no `transform` block takes the
defaults, which reproduce the old hardcoded behaviour exactly. No version bump
needed for this phase.

### Deferred

**Auto-level.** Fitting a ground plane to the lowest few percent of splats and
levelling to it would be a genuinely nice affordance, and it is a separate piece
of work with its own failure modes (a scan with no floor, a scan whose lowest
splats are noise). Not in this plan. Manual correction first; automate it later
if the manual controls turn out to be tedious in practice.

### Acceptance

- `mdRotX = 180`, everything else default, reproduces the current render exactly
  — same coverage, same bbox, within noise.
- Rotating 90° about Z rotates the rendered bbox by 90°, measured, not observed.
- `radius` and `bounds` are **identical** before and after any transform change.
  Assert this in the debug harness; it is the whole point of the split.
- With normalisation frozen, `dAmt` at a fixed value produces the same
  displacement magnitude at every model rotation.
- `mdScale = 2` doubles the rendered bbox extent and leaves every normalised
  parameter's response unchanged.
- A transform change with a stationary camera is visible without any further
  interaction.
- `Cancel` restores the entry transform exactly; `Commit` persists it.
- No mode combination allows the gizmo and `OrbitControls` to both act on one
  drag, or `Adjust` to be live inside `Correct`. Assert in the debug harness.
- A v3 preset loads and renders identically to before the phase.

---



The largest phase by a wide margin. Budget accordingly and do not start it in
the same session as anything else.

### The finding that drives the design

`SplatEdit` / `SplatEditSdf` run inside Spark's pipeline. They can change colour
and opacity; they **cannot** hand a per-splat mask to your dyno graph. Masking
`dAmt`, `sat` or `cullA` by a region therefore requires evaluating the SDF
inside `buildModifier()`.

Once the SDF is in dyno, the native edit is redundant — cut, crop and colorize
are all trivially expressible against a dyno mask. **Remove the `SplatEdit`
path entirely rather than running two region systems.** Two systems means two
transform sources of truth and a UI that has to explain the difference.

Keep the measured findings from the native implementation; they are still the
correct semantics:

- `ROUND_BY_RADIUS = {sphere, cylinder, capsule}` — the radius/scale distinction
  is real and per-shape, and getting it wrong fails silently. Reproduce the
  coverage table from NOTES against the new dyno SDFs; the numbers should match
  the native ones within measurement noise. If they do not, the SDF is wrong.
- `cut` = opacity 0 inside, `crop` = the same inverted, `colorize` = multiply.

### Architecture

```
REGIONS = [ {enabled, shape, mode, pos:[x,y,z], size, softness, invert, rgb}, … ]
MAX_REGIONS = 4
```

- **Fixed slot count, unrolled in JS.** Write a JS `for` loop over
  `MAX_REGIONS` inside the `dynoBlock` closure. It unrolls into inline
  expressions at graph-build time, so no GLSL loop is needed and no dynamic
  indexing is involved. All four slots always exist in the graph; a disabled
  slot is a neutral mask of 1.0 driven by an `enabled` uniform. This preserves
  the no-recompile invariant for everything except shape.
- Start at `MAX_REGIONS = 4`. Raising it is a one-line change plus a recompile;
  do not make it dynamic.
- Each slot produces `mask_i` in `0..1`, soft-edged via `smoothstep` over the
  `softness` band, with `invert` flipping it.

### The one sanctioned recompile

Six shapes × four slots evaluated unconditionally is 24 SDF evaluations per
splat per frame. The alternative is rebuilding the graph when a shape changes.

**Take the recompile.** Shape is a discrete click, not a slider drag. Gate it:

- Only `rgShape*` selects trigger `updateGenerator()`. Nothing else may.
- Show it in the status line so it is never silent.
- **Measure compile time before committing to this.** If `updateGenerator()`
  with four slots exceeds ~150 ms on the butterfly, fall back to evaluating all
  six shapes per slot with a `select` chain and eat the arithmetic instead.
  Record whichever you chose, and the number that decided it, in NOTES.

This is a deliberate exception to the invariant in NOTES. Document it there as
an exception, with its reasoning, so it does not read as drift.

### Masking effect parameters

Each maskable effect gets one **mask source** select: `none | R1 | R2 | R3 | R4`,
plus an `invert` toggle. Maskable: `dAmt`, `sat`, tint RGB (as a group),
`cullA`, `cullS`, `scale`.

- The mask multiplies the effect's *amount*, not its output:
  `effectiveAmount = V[id] * mask`. For `sat` and `scale`, whose neutral value
  is 1 rather than 0, mask toward the neutral value:
  `mix(neutral, V[id], mask)`.
- **One mask source per effect, not a set.** Combining masks is the obvious next
  ask and the obvious way to make the UI unnavigable. If it is genuinely needed
  later, add a per-region `combine` mode (`add` / `multiply` / `max`) that
  composites regions into a single mask *before* effects sample it — one mask
  bus rather than per-effect combination logic.

### Removing Reveal

Reveal becomes a plane region in `cut` mode with a soft edge, animated by an
envelope on the region's position. Verify that specific setup reproduces the
old look before deleting anything.

- Remove `revY`, `revS`, `revK` from `SPEC`; remove `revAnim`, `revFlip`.
- Remove the reveal block from `buildModifier()`.
- The old reveal shrank splat scales into the edge as well as fading alpha —
  that was the nicer half of the effect. Preserve it as a per-region
  **`shrink`** parameter (0..1) so region masks can shrink as well as cut.
  Without this the migration is a downgrade.

### Preset migration — this is the breaking change

Bump `PRESET_VERSION` to 4 and **migrate rather than reject**, per NOTES:

- A v3 preset carrying `revY`/`revS`/`revK` envelopes maps onto region slot 1:
  plane shape, `cut` mode, position from `revY`, softness from `revS`, shrink
  from `revK`, `invert` from `revFlip`.
- A v3 envelope on `revY` becomes an envelope on `rg1Pos`.
- A v3 preset with the old single-region `rg*` keys maps onto slot 1.
- Anything not mappable is skipped with a named message, not silently.

### Acceptance

- Coverage table from NOTES reproduced against dyno SDFs, all six shapes, within
  noise of the native numbers.
- `cut` / `crop` / `colorize` measurably distinct, matching the NOTES figures
  (off 9.95, cut 4.39, crop 6.07; colorize shifts R up and B down).
- Four simultaneous regions with different shapes render correctly and
  independently.
- Masking verified numerically: `dAmt` at full with a sphere mask displaces
  splats inside the sphere and leaves the bbox outside it unchanged.
- A v3 preset using reveal loads, migrates, and produces the same coverage curve
  across the timeline as it did pre-change, within 2%.
- Frame time at 177k splats with four regions active stays above 60 fps.

---

## Phase 4 — Camera keyframes

Replaces `key.A` / `key.B` with a list.

### Model

```js
CAMKEYS = [ {t, pos:[x,y,z], tgt:[x,y,z], fov}, … ]   // sorted by t
```

- Store `fov` per key. It costs nothing now and retrofitting it later means
  another preset migration.
- Keys carry `t` in the same normalised 0..1 clip space as envelopes, so
  changing `Duration` rescales the camera move exactly as it rescales
  everything else.

### Interpolation

- **Catmull-Rom on position and target** for three or more keys. Linear lerp
  through a multi-key path gives visible corner-kinks at every key, which is
  the main reason A/B felt acceptable and a key list will not.
- Duplicate the first and last keys as phantom control points so the spline is
  defined at the ends.
- Fall back to linear for exactly two keys — Catmull-Rom with two points is
  linear anyway, but the explicit path avoids the phantom-point edge case.
- Offer a `linear | smooth` toggle mirroring the envelope lane's `smooth`, for
  consistency and for when a hard linear dolly is wanted.
- **`ease` now applies per segment**, not globally. Keep the slider and its
  current meaning; apply it to the normalised position *within* each segment.
  Note the known issue from NOTES: `ease(0.5, k) = 0.5` for all `k`, so probe
  segment easing at 0.25 of a segment, not its midpoint.

### Interaction

No sliders, per the brief.

- **`Add key`** — captures current camera at the current playhead. Adding at a
  `t` that already has a key replaces it.
- **`Remove key`** — removes the key at the playhead if one is within a small
  tolerance; otherwise reports that there is nothing there rather than removing
  the nearest.
- Keys render as ticks on the existing envelope lane, in a **dedicated camera
  row** so they are never confused with parameter keys. Alt-click deletes,
  matching the existing lane convention exactly — do not invent a second gesture
  vocabulary.
- Clicking a camera tick moves the playhead to it and applies that key's pose.
- The `Go A` / `Go B` fix from NOTES generalises: applying a key must also move
  the playhead to that key's `t` when camera drive is on, or `driveCamera()`
  overwrites the pose on the next frame. This was already fixed once for A/B —
  do not regress it.

### Preset

`camera: {A, B}` becomes `camera: {keys: [...], smooth: bool}`. Migrate v3 by
mapping A to `t: 0` and B to `t: 1`. Additive for load-forgiveness: a preset
with no keys leaves the camera manual.

### Acceptance

- Five keys interpolate smoothly; sample camera position at 100 points along the
  timeline and confirm no discontinuity in first difference greater than 3× the
  local mean.
- Removing a middle key leaves a valid path and does not throw.
- Removing all keys returns the camera to manual control.
- Export with five keys produces a monotonic luma curve consistent with the
  path (per the existing ffprobe `signalstats` check in NOTES).
- v3 preset with A/B loads and reproduces its original camera move.

---

## Phase 5 — Auto-orbit as a camera drive mode

Make the camera drive an explicit three-way mode rather than a checkbox:

```
Camera drive:  Manual | Keyframes | Orbit
```

This replaces `camDrive`. The checkbox already caused one bug (`Go A` silently
overwritten) precisely because two drivers could both think they owned the
camera; a single mode select makes that structurally impossible.

Orbit parameters, all in `SPEC` group `camera` so they animate and persist:

- `obAxis` — `X | Y | Z` select, the axis revolved about.
- `obTurns` — revolutions across the clip, `-3..3`, default `1`. Signed, so
  negative reverses.
- `obPhase` — start angle, `0..1`, default `0`.
- `obElev` — elevation above the orbit plane, `-1..1`, default `0`.
- `obDist` — distance as a multiple of scene `radius`, default from `reframe()`.

Orbit centres on `controls.target`, so framing a subject then switching to Orbit
does what you expect.

### Acceptance

- `obAxis` X, Y and Z each produce a distinct path: sample 100 camera positions
  across the clip and confirm the near-constant coordinate is the chosen axis.
- `obTurns = 1` returns the camera to within 0.1% of its start position at
  `t = 1`.
- `obTurns = -1` traverses the same path in reverse.
- Mode select is exclusive — no combination of buttons produces two drivers
  writing the camera in one frame. Assert this in the debug harness.
- An export in Orbit mode produces a smooth luma curve with no discontinuity at
  the wrap point.

---

## Phase 6 — Audio shaping

### The finding that drives the design

Attack/release smoothing is stateful and sequential. It cannot be applied at
lookup time: scrubbing backwards through a one-pole filter would give different
values than scrubbing forwards, and the export would not match the scrub. The
whole reason the analysis is offline is determinism, and a lookup-time filter
would throw that away.

**Therefore: shaping is a pass over the precomputed envelope, producing a
derived array cached per route.** At 200 Hz an eight-minute track is ~96k
floats per band — trivial to recompute, so invalidate aggressively rather than
trying to be clever about partial updates.

### Per-route parameters

Current shape is `{band, depth}`. Extend to:

| param | range | default | meaning |
|---|---|---|---|
| `band` | low/mid/high | — | unchanged |
| `depth` | −1..1 | — | unchanged, signed fraction of parameter range |
| `attack` | 0..500 ms | 0 | rise smoothing time constant |
| `release` | 0..2000 ms | 120 | fall smoothing time constant |
| `lag` | −500..500 ms | 0 | time offset; negative anticipates |
| `gain` | 0..4 | 1 | pre-curve multiply |
| `floor` | 0..1 | 0 | gate; below this the band reads 0 |
| `curve` | 0.25..4 | 1 | exponent applied after gain, 1 = linear |

Order of operations, fixed and documented:

```
raw  → gate(floor) → ×gain → ^curve → clamp 0..1
     → asymmetric one-pole (attack/release)
     → shift by lag
     → ×depth × parameterRange
```

- `attack` / `release` separately is the point. A single "smoothness" control
  cannot give a fast transient with a slow tail, which is the main thing anyone
  wants from audio reactivity.
- `lag` shifts the sampling index. Negative lag is free here and impossible in a
  realtime tap — worth having, because it lets an effect land *on* the beat
  rather than after it.
- Apply the curve **before** smoothing. After smoothing it distorts the envelope
  shape you just carefully designed.

### Caching

- Cache key: `band + all shaping params`. Two routes with identical shaping on
  the same band share one derived array.
- Invalidate on any shaping change and on reanalysis.
- Recompute is synchronous and must stay under 16 ms for a five-minute track.
  Measure it; if it does not, move it to a worker rather than debouncing, since
  a debounce makes slider drags feel broken.

### UI

The routing panel now has eight controls per route, which is where a flat list
stops working.

- One route row per focused parameter, expanded inline under it.
- **`band` and `depth` stay on the collapsed row.** They are the two anyone
  touches; the other six are shaping and belong behind a disclosure.
- Show a small sparkline of the shaped envelope, with a playhead marker. This is
  the only affordance that makes attack/release legible — the numbers alone tell
  you nothing about what the curve is doing.
- Preserve the non-destructive focus rule from NOTES: clicking a parameter name
  focuses without arming anything. This has already bitten once.

### Preset

`audio.routes` entries gain the six new fields. Additive and load-forgiving —
missing fields take defaults, so v3 presets keep working without a bump.

### Acceptance

Against the existing synthetic `bandtest.wav` fixture:

- `attack = 0, release = 0` reproduces the current per-band table exactly
  (low 0.78/0.04/0.00, mid 0.10/1.00/0.09, high 0.00/0.16/1.00).
- `release = 1000` on the low band: value at t=2.5s is measurably above the
  unsmoothed value, and decays monotonically to t=4s.
- `lag = 250` shifts the low-band peak from 1.0s to 1.25s within one sample.
- `lag = -250` shifts it to 0.75s.
- `floor = 0.5` zeroes all off-band bleed (the 0.09–0.16 biquad skirt).
- `gain = 2, curve = 2` compounds correctly: verify one sample by hand against
  the documented order of operations.
- Export determinism: render the same 30 frames twice with shaping active and
  confirm identical per-frame luma.

---

## Mobile

Per the brief, features that do not work on touch may be dropped. Everything
stays in the one `@media (max-width: 900px), (pointer: coarse)` block at the
**end** of the stylesheet, subtracting only.

Drop on mobile:

- The camera key lane row — reading and hitting ticks on a 390px lane is not
  viable. Keep `Add key` / `Remove key` buttons; drop direct tick manipulation.
- The audio sparkline. Keep band and depth; put shaping behind the disclosure
  and accept that it is fiddly.

Keep on mobile:

- Region controls. They are sliders and selects, which the bottom sheet handles.
- Orbit mode. It is the *most* useful mobile feature here — a generated camera
  move avoids needing precise camera positioning on a touchscreen.
- Saturation. It is one slider.

Re-verify the 390×844 layout numbers from NOTES after Phase 0, since the
accordion changes rail height.

---

## Risk register

| risk | phase | mitigation |
|---|---|---|
| Normalisation basis still follows the transform | 2 | Assert `radius`/`bounds` unchanged across transform changes in the debug harness |
| Transform change invisible until the camera moves | 2 | Test with a stationary camera via `settled()`; route through `pushUniforms()` if needed |
| dyno SDFs silently wrong (the radius/scale trap) | 3 | Reproduce the NOTES coverage table shape by shape; treat any mismatch as an SDF bug |
| `updateGenerator()` on shape change is too slow | 3 | Measure before committing; fall back to select-chain over all six shapes |
| Warm-up false readings after a load | 3 | Established hazard — probe until two consecutive reads agree; treat all-zero as not-warm |
| `SPEC` at ~60 entries makes the lane unusable | 0, 3 | Accordion first; lane shows only focused + armed parameters |
| Preset v3→v4 reveal migration loses the look | 3 | Coverage-curve comparison across the timeline, not a visual check |
| Two camera drivers writing in one frame | 4, 5 | Single mode select, asserted in the debug harness |
| Shaping recompute stutters slider drags | 6 | Measure; move to a worker rather than debouncing |
| Chrome caching the old build mid-debug | all | Serve with `Cache-Control: no-store` per the NOTES one-liner |

---

## Definition of done, per phase

1. Acceptance numbers recorded in the commit message, not just observed.
2. `NOTES.md` updated in the same commit — new invariants, new measured tables,
   anything moved from assumed to verified.
3. Preset round-trip fingerprint still matches.
4. The `?debug=1` regression sweep passes, warmed properly.
5. Deployed to Pages and loaded once from the Pages origin, not only localhost.
