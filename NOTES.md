# Splatification — project notes

Working folder: `/Users/55147209/Documents/Claude_Builds/Splatification`
Main file: `index.html` (single file, no build step, no npm install)
Serve with `python3 -m http.server` and open `http://localhost:8000/`.
ES modules will **not** load over `file://` — it must be served.

It is `index.html` rather than a descriptive name so GitHub Pages serves it at
the repo root with no path. `splat-bench.html` remains as a redirect stub for
older links and carries the query string across, so `?debug=1` survives.
Live at <https://quixbrics.github.io/splatification/>.

---

## What this is

A bespoke 3D Gaussian Splatting manipulator and animator for the browser. The
reference point is the Irrealix After Effects plugin (crop, colorize ramp,
opacity-ramp reveal, splat-scale animation, noise distortion) — but outside of
After Effects, and built on the layer *underneath* those effects rather than a
fixed effect list.

The point is not to reimplement SuperSplat. SuperSplat already does cleanup,
camera timelines and video export well, and it is free and open source. This
exists to do things SuperSplat does not: arbitrary per-Gaussian programmable
manipulation, and eventually parameter automation driven from outside (audio
FFT, LFOs, OSC).

Target is **desktop**. A dialled-back mobile path is wanted eventually but must
not constrain design decisions now.

---

## Architecture

Dependencies are all CDN imports via an importmap. No package.json, deliberately.

| Import | Version | Role |
|---|---|---|
| `three` | 0.180.0 | scene graph, camera, OrbitControls |
| `@sparkjsdev/spark` | 2.1.0 | 3DGS renderer + `dyno` shader-graph system |
| `mp4-muxer` | 5.x | muxes WebCodecs chunks into MP4 |

**Spark** (World Labs, MIT) is the substrate. The key facility is `dyno`: a
shader-graph system written in JS that compiles to GLSL and runs per-splat on
the GPU. `SplatMesh` exposes `objectModifiers` and `worldModifiers` as injection
points into the standard splat generation pipeline.

Everything in the effect stack is a single `dyno.dynoBlock` mapping
`{gsplat: Gsplat} -> {gsplat: Gsplat}`, passed as `objectModifiers: [modifier]`.
Current effects, all in one GPU pass:

- **Displace** — `sin()` field over swizzled center × per-splat `hashVec3(index)`
  direction, scaled by amount
- **Reveal** — soft `smoothstep` sweep plane along object-space Y; shrinks splat
  scales into the edge rather than only fading alpha
- **Trim** — zeroes alpha for gaussians below a min opacity or above a max scale
  (cheap floater kill; does not reduce splat count)
- **Tint** — RGB multiply

### Uniforms, not constants

Every parameter is a `dyno.dynoFloat` / `dynoVec3` uniform held in the `U`
object. Assigning `U.foo.value = x` is free. Changing the *structure* of the
graph forces a GLSL recompile and requires `splatMesh.updateGenerator()`.

**Keep this invariant.** Adding a new effect should mean adding uniforms and
graph nodes once at construction, gated by an amount uniform that can go to
zero — not rebuilding the graph when a control changes.

Assigning `U.foo.value = x` avoids the recompile but does **not** by itself make
the change visible — Spark must also be told the splat data is stale. See "The
uniform invariant" below; this was the single biggest live bug in the file.

### Slider normalisation

Sliders are normalised against scene extent so they behave the same on any
capture. `reframe()` computes `radius` (bounding sphere) and `bounds.min/max`
(Y extent); `pushUniforms()` multiplies through by those. Do not feed raw
slider values into world-space uniforms.

Normalising is necessary but not sufficient: `cullS` was correctly multiplied by
`radius` and still unusable, because the slider's *range* did not match the
quantity's actual distribution. When adding a parameter, sweep it and look at
where the response actually happens — a correctly normalised control can still
put its whole useful band inside 0.1% of the travel.

---

## The uniform invariant — read this before adding an effect

`U.foo.value = x` is cheap (no GLSL recompile), but **it is not sufficient on
its own.** Spark gates its generate pass, `spark.module.js` 2.1.0 ~line 10210:

```js
const needsUpdate = viewChanged || version !== this.current.version;
if (autoUpdate && !needsUpdate) { doUpdate = false; }
```

Spark *bakes* modifier output into a packed splat buffer; it does not evaluate
the dyno graph per draw call. Writing a uniform's `.value` changes neither the
view nor the version, so the stale baked buffer is reused and the change is
invisible — until the camera happens to move, at which point every pending edit
pops in at once.

`pushUniforms()` therefore ends with `mesh?.updateVersion();`. That increments an
integer. It is not `updateGenerator()` and does not recompile GLSL. Measured cost
at 177k splats: none — 120 fps idle, playing, and playing with displace active.

**Any new code path that writes a uniform must bump the version.** Route
everything through `pushUniforms()` rather than assigning `U.*.value` directly.

This one is nasty because it fails *silently and intermittently*: dragging a
slider does nothing, then an unrelated orbit makes it all appear, so it reads as
a sort/async glitch rather than a missing invalidation.

---

## Known issues

**`ease` is mathematically inert at `playhead = 0.5`.** `ease(0.5, k) = 0.5` for
every `k`. Expected from the formula, not a bug — but it means the midpoint is a
bad place to check whether the ease slider works. Probe at 0.25.

**A hidden tab still stops the render loop.** `requestAnimationFrame` does not
fire when the tab is not visible, so the live view freezes and `setTimeout` is
clamped to ~1s. The export no longer depends on that timer (see Fixed), but a
background export still crawls because nothing composites. The status line
appends `[tab hidden — throttled]` and a warning fires at render start.

This also breaks automated pixel probes: readPixels can only ever return the
last composited buffer. Use `?debug=1` and `window.__bench.settled()`, which
draws synchronously and awaits the sort without involving rAF.

**Region: `ellipsoid` renders identically to `sphere`.** Correct, not a bug —
the Size control is a single uniform scalar, and a uniformly-scaled ellipsoid
*is* a sphere. Ellipsoid only becomes distinct with per-axis size, which the
current UI cannot express. Add three size sliders if it is ever wanted.

---

## Fixed

**TDZ on `playhead` (v1).** `let playhead = 0` was declared below the parameter
binding block while `pushUniforms()` read it; the throw during module evaluation
killed every listener and the render loop below it. Hoisted into the state block
at the top. All module state now lives there — keep it that way.

Two hardening measures from that episode, both still in place and worth keeping:

1. `<output>` values are blanked in HTML and populated by JS, so a dead module
   looks dead instead of showing plausible hardcoded numbers.
2. A global error surface near the top of the module:
   ```js
   addEventListener("error", e => setStatus("JS error: " + e.message, true));
   addEventListener("unhandledrejection", e => setStatus("Async error: " + (e.reason?.message ?? e.reason), true));
   ```

**`Go A` / `Go B` were silently ignored while `Timeline drives camera A→B` was
checked** (the default). `apply()` set the camera and `driveCamera()` overwrote
it on the very next frame from `playhead`. `apply(k, at)` now also moves the
playhead to the matching end of the timeline (0 for A, 1 for B) when camDrive is
on and both keys exist, so the drive agrees with the button instead of fighting
it. With camDrive off the timeline is left alone. Verified: camDrive on, Go A →
coverage 10.305 / scrub 0, Go B → 2.487 / scrub 1.

**`Max splat size` (`cullS`) addressed nothing across 99.9% of its travel.** The
old slider was linear `0.001..9` while the entire useful band was `0.001..0.01`.
`P.cullS` is now a 0..1 perceptual position mapped exponentially onto the
threshold via `cullSize()`, `CULL_MIN 0.0002 .. CULL_MAX 0.1`. The `* radius`
normalisation was kept — it was never the problem, the range was. Measured
transition band now spans roughly u=0.25..0.6, about 35% of travel:

| u | 0.0 | 0.2 | 0.3 | 0.4 | 0.5 | 0.6 | 0.8 | 1.0 |
|---|---|---|---|---|---|---|---|---|
| threshold | .0002 | .0007 | .0013 | .0024 | .0045 | .0083 | .0289 | .1000 |
| coverage % | 0 | 0 | 0.41 | 4.05 | 9.67 | 10.29 | 10.30 | 10.305 |

`cullS` is the only parameter whose slider position is not its value, so it
needs the `SHOW` readout override. If a second such parameter appears, that map
is the place to put it.

**`revS` default was unrepresentable on its own step grid.** Declared
`value="0.15"` against `min="0.001" step="0.002"`, which only yields odd
thousandths — the browser silently snapped it to 0.151, and the panel had been
displaying 0.151 for a declared 0.15 since the beginning. Step is now `0.001`.
This was found by the reset round-trip test, not by reading the markup.

---

## Per-parameter reset

Every slider row carries a `↺` button, generated in JS from `SPEC` rather than
hand-written into 18 rows of markup.

The default is **not** duplicated in JS. It is read back from the control itself
at boot into `DEFAULT[id]`. Note this is the *snapped* boot value, not
`el.defaultValue` (the raw HTML attribute) — those two differ whenever a
`value` attribute is off the min/step grid, which is exactly the revS bug above.
Comparing against the snapped value makes reset exact even if a future slider
repeats the mistake.

Reset dispatches a synthetic `input` event rather than calling `pushUniforms()`
directly, so a reset and a user drag travel exactly the same code path —
including the `dpr`/`dur` side effects and the version bump.

The button carries a `.dirty` class whenever the value differs from default,
which makes the panel double as an at-a-glance "what have I changed" readout.
Checkboxes deliberately have no reset button: toggling one is already a
single-click restore.

---

**Export backpressure was timer-based.** `while (encoder.encodeQueueSize > 8)
await sleep(2)` cost ~313ms per iteration in a hidden tab instead of 2ms.
Replaced with `drain()`, which awaits the encoder's `dequeue` event — not
clamped — raced against a 50ms sleep to cover the case where the queue empties
between the size check and the listener attaching.

**The `settle` wait is now a deterministic sort signal.** The export loop used
to render, `sleep(settle)`, render, capture — where 24 ms was an admitted guess.
It now `await`s `spark.update({ scene, camera })`, which runs generate + sort for
that exact viewpoint and resolves when both are done. `settle` survives as an
extra manual wait defaulting to 0.

Note the API shape: **in Spark 2.1.0 the `SparkRenderer` IS the viewpoint.** It
owns `update()`, `driveSort()` and `sorting` directly — there is no separate
`SparkViewpoint` instance to reach through, and `spark.viewpoint` is
`undefined`. `minSortIntervalMs` defaults to 0, so `driveSort()` is not
deferred behind a timer.

---

## Model transform — Correct mode

Position/rotation/scale on the loaded capture, so a scan that came out of
reconstruction tilted, off-centre or in the wrong units can be corrected in the
tool rather than in SuperSplat and re-exported. This replaces the old hardcoded
`mesh.quaternion.set(1, 0, 0, 0)` Y-down fix — that value is now the *default*
of the rotation controls (`mdRotX = 180`) rather than an invisible constant, so
a capture that is not Y-down is correctable for the first time.

### Locked behind an explicit mode, on purpose

The transform is not live-editable from the rail. `Correct` is a mode: enter
it, adjust with the numeric fields or the `TransformControls` gizmo, then
`Commit` or `Cancel`. Outside the mode every field is `disabled` and nothing
else in the codebase writes `mesh.position/rotation/scale` — there is exactly
one write path (`applyXform()`), which is what makes a drag gizmo safe to offer
at all.

Interlocks, all asserted in the debug harness, not just implemented:

- Entering Correct force-disables gesture `Adjust` (`setGesture(false)`) and
  unchecks `camDrive` (`driveCamera()` reads it live, so this alone stops the
  timeline fighting the gizmo). Both are restored on exit.
- `syncControlsEnabled()` is the single choke point for `OrbitControls.enabled`
  — both `setGesture()` and the gizmo's `dragging-changed` listener route
  through it, rather than each setting the flag independently. Two systems
  independently touching one shared flag is exactly how `Go A` / `Go B` got
  clobbered earlier in this file; this phase does not repeat that shape of bug.
- Verified: entering Adjust while correcting is refused outright; entering
  Correct while Adjust is already open force-disables Adjust and proceeds;
  simulating `xctl.dragging = true` disables orbit and re-enables it on
  release; export is refused with a message while correcting.

### The normalisation-basis split — and what it actually fixed

`reframe()` used to compute `radius`/`bounds` from the mesh's **world-space**
bounding box (`getBoundingBox(true).applyMatrix4(mesh.matrixWorld)`) and reuse
that same box for camera framing. That conflates two different jobs, and
Phase 2 splits them:

- **Normalisation basis** (`radius`, `bounds`) — object-space, computed **once**
  by `captureObjectBasis()` right after load, from `mesh.getBoundingBox(true)`
  with **no** matrix applied. Frozen until the next splat load or preset load;
  a Correct-mode edit never touches it.
- **Camera framing** (`reframe()`) — world-space, using a **local** `worldRadius`
  computed fresh from `getBoundingBox(true).applyMatrix4(mesh.matrixWorld)`
  every time it is called. This is "what fits on screen given the current
  pose," not "what a slider value means," and the two must not share a
  variable.

This is grounded in reading `getBoundingBox()` in spark.module.js 2.1.0
directly, not inferred: it iterates raw per-splat `center`/`scales`/`quaternion`
from `this.splats`, never touching the mesh's own Object3D transform. That
confirms `objectModifiers` — what `buildModifier()` is registered as — operate
on that same raw, untransformed data. `SplatEdit`/`SplatEditSdf` (the Region
system) are architecturally different: `edit.add(sdf); mesh.add(edit)` parents
them to the mesh, so they inherit its transform automatically through the
ordinary scene graph. That is *why* Spark offers both mechanisms — dyno effects
need this explicit basis freeze; region edits get transform-following for free
and needed no change in this phase.

**A latent bug this incidentally fixed.** Before this split, `bounds.min/max`
(the reveal sweep's Y-threshold) was being computed in world space — i.e.
*after* the 180° Y-down rotation — while the shader's `y` is raw object-space,
*before* it. A pure 180°-about-X rotation preserves span but negates and swaps
the endpoints, so the sweep plane's absolute position was measurably off from
where the shader's own coordinate actually sat. It was invisible in every prior
measurement because `revY`'s default is `1` (fully revealed), where the offset
doesn't matter — an endpoint value shows "everything" regardless of exactly
where the plane sits, as long as span is preserved (which a rigid rotation
always preserves). The *sweep shape* did change once the frame was corrected —
measured, same capture, same slider values:

| revY | 0 | 0.25 | 0.5 | 0.75 | 0.9 | 1 |
|---|---|---|---|---|---|---|
| coverage % (old, world-frame bounds) | 0.12 | 2.8 | 6.5 | 9.7 | 10.3 | 10.3 |
| coverage % (new, object-frame bounds) | 0 | 1.55 | 4.99 | 8.35 | 9.54 | 9.72 |

Both are monotonic, working sweeps — this was never visibly "broken" — but only
the new one is measuring the plane against the coordinate frame the shader
actually uses.

### Invalidation — measured, not assumed

The obvious question: does a bare Object3D transform change (no dyno uniform
touched) show up on the very next render, same as the original uniform-version
bug, or does it need an explicit `mesh.updateVersion()`? Tested directly rather
than guessed, with the camera held perfectly still:

- A transform change followed by exactly **one** `renderer.render()` call
  still shows the **stale** pose.
- A **second** `renderer.render()` call — with nothing else touched in
  between, no `pushUniforms()`, no `updateVersion()` — shows the **correct**
  pose, and a third confirms it is stable there.
- Calling `pushUniforms()` (and therefore `mesh.updateVersion()`) between the
  two renders does **not** shortcut this to one render. It is unrelated to the
  version-gate mechanism the uniform invariant documents.

So this is a **different, already-documented hazard**, not a recurrence of the
original bug: the same "render, then render again" latency NOTES already
describes for camera moves and for the offline export loop's `settle` step —
Spark's async sort needs one extra frame to catch up to a world-position
change, of any kind. It is not a "stuck until something unrelated happens"
failure, just a one-frame lag.

**Consequence for the UI, not just the test:** during interactive Correct-mode
editing, `frame()` calls `renderer.render()` on every animation frame
regardless of what changed, so this self-resolves within about one frame
(~16 ms) with no further user action — satisfying "visible without any further
interaction" in the sense that matters, even though it is not literally the
very next paint. It cannot bite an export, because export is refused outright
while correcting, so a committed transform is always many frames "settled" by
the time Render is clickable again.

`applyXform()` still calls `pushUniforms()` on every edit. Not because it
shortens the one-frame lag above — measured, it does not — but because it is
free (confirmed: a single integer increment, no measurable cost) and it keeps
every other normalised uniform consistent if anything about the load-time state
changes in a future phase. Kept as a defensive no-op with a correct comment,
not a fix for a problem it does not solve.

### `mdScale` is log-mapped, same reasoning as `cullS`

Slider is a 0..1 perceptual position, `xscaleOf(u) = 0.1 * (10/0.1)^u`, so
`u = 0.5` maps to exactly `1.0` — the slider's own midpoint is the neutral
scale, with no special-casing needed. Verified against the mesh's own
**world-space** bounding box (not screen pixels — see below):

| slider u | 0.5 (default) | 0.6505 (target 2×) |
|---|---|---|
| mapped scale | 1.000 | 1.995 |
| world-space extent ratio | — | **1.9953×** (x/y/z identical) |

### A measurement trap worth recording: perspective clipping looks like a bug

The first attempt to verify "`mdScale = 2` doubles the rendered bbox" measured
**screen-space pixel width**, and got 1.868–1.885×, not 2×. That is not an
implementation defect — the doubled butterfly's bottom edge was measurably
touching the canvas edge (`y1 = 780` of `H = 782`), i.e. genuinely clipped by
the viewport, because `reframe()` is deliberately **not** called on every
Correct-mode edit (a live camera re-fit while dragging a slider would be
disorienting, defeating the point of a stable reference frame to correct
against). A screen-space measurement is only valid when nothing is clipped or
foreshortened; **the mesh's own world-space bounding box is the correct
instrument** for verifying a transform edit, not pixel coverage. Coverage stays
the right tool for verifying *effects* (which don't move the camera-relevant
extent), just not for verifying the *transform* itself mid-edit.

### A real bug this phase's own testing caught: `xformReset()` skipped the reframe

`Reset transform` originally called `reframe()` only `if (correcting)`. Outside
Correct mode (its normal, undisabled state — the button works at any time by
design) it reset the mesh's geometry but left the camera framed for whatever
the *previous* transform looked like. Concretely: load a preset with an
extreme transform (which does call `reframe()`), then click Reset outside
Correct mode — the geometry snaps back to default size/position, but the
camera stays fitted to the old, larger/offset pose, leaving the now-correct
model tiny or off-centre. Measured as a full regression-sweep collapse (every
coverage figure near zero) before being traced to this. Fixed: `reframe()` is
now unconditional in `xformReset()` — any discrete, deliberate transform
mutation (load, commit, cancel, reset, preset) reframes; only *continuous*
edits (slider drag, gizmo drag) deliberately do not.

### Preset

`transform: {...xform}` — additive, no version bump. A preset with no
`transform` block takes `XFORM_DEFAULT`, reproducing the pre-Phase-2 hardcoded
behaviour exactly. A preset with one is clamped field-by-field to each
control's own range, same treatment as every other preset value; a hostile
preset (`rotX: 9999, posX: 50, mdScale: 5, rotZ: "nan"`, …) loads cleanly with
every field clamped and no warning raised. Loading a preset while correcting
exits the mode first.

Property of the *capture*, not the workspace — unlike `Mobile profile` — so
unlike that toggle, this belongs in the file. Presets reference the asset by
name only, so a transform saved against one capture and loaded against another
is meaningless; harmless, not worth guarding against.

### Not in `SPEC`, on purpose

`mdRotX/Y/Z`, `mdPosX/Y/Z`, `mdScale` are hand-wired rather than added to
`SPEC`: a locked transform cannot carry an envelope, so SPEC membership would
buy free reset/preset/audio wiring for a control that structurally cannot use
most of it. The rows carry two `.key.ghost` placeholders each (no per-row reset
button, no key glyph) purely to keep the grid's label/output columns aligned
with every SPEC-driven row — there is a single bulk `Reset transform` button
instead. The accordion's `.gflag` summary marker is hand-managed too
(`refreshModelUI()`, keyed off `#modelFlag` by id) since `groupIds("model")` is
empty; `refreshGroups()` now explicitly skips any group with no `SPEC` entries,
rather than silently clobbering a hand-managed flag with an always-empty
computation — a real conflict caught before it shipped, not after.

---

## Region — dyno SDFs and per-effect masking

Roadmap Phase 3 of `DEV_PLAN.md`, the largest single phase. Replaces the native
`SplatEdit`/`SplatEditSdf` system entirely with 4 region slots implemented as
plain dyno graph math, and folds Reveal into it as a plane-shaped region.

### Why the native system had to go

`SplatEdit`/`SplatEditSdf` can change colour and opacity but cannot hand a
per-splat mask to a dyno graph — Spark applies them in its own pipeline, after
`objectModifiers` run. Masking Displace/Saturation/Tint/Trim by a region (the
actual point of this phase) needs the region's inside/outside test to exist
*inside* `buildModifier()`. Once the SDF has to live in dyno anyway, running
two parallel region systems means two transform sources of truth and a UI that
has to explain the difference — so the native path was removed rather than
kept alongside.

### Architecture

`region[i] = {shape, mode, invert}` for `i = 0..3` (R1..R4) — plain JS routing
state, not `SPEC`, same treatment as `AUD`'s band/depth. The 9 per-slot
*numeric* fields (`rg{n}X/Y/Z/Size/Soft/Shrink/R/G/B`, 36 entries total) **are**
generated into `SPEC`, exactly like `REGION_SPEC` generates them rather than
hand-listing — 36 near-identical entries is exactly the repetition `SPEC`
exists to absorb, and hand-listing is where a copy-paste slot-index mistake
hides. Rows are generated the same way (`buildRegionRows()`), one `.rgpanel`
wrapper per slot, all 4 permanently in the DOM (`SPEC`'s bootstrap loop needs
every element to exist) with only the selected slot's panel visible — a tab
UI, not 36 always-visible rows. `#rgMode`/`#rgShape`/`#rgInvert` are ONE shared
set of controls that read/write whichever slot `rgSlot` currently points at.

Every shape is driven by a single `size` uniform (matching the old system's one
Size slider), not the old radius-vs-scale duality — that duality was a Spark
API quirk this reimplementation does not need to reproduce, only its rendered
result.

### The one deliberately-avoided recompile

`DEV_PLAN.md` flagged this explicitly: 4 slots x 7 shapes is 28 SDF
evaluations per splat per frame if done unconditionally, versus recompiling
`buildModifier()` and calling `mesh.updateGenerator()` on every shape change.
The plan's own instruction was to measure compile time and only take the
recompile if evaluating everything unconditionally proved too slow.

**Measured: baseline (no regions) and 4 simultaneous active crop regions,
different shapes, played back to back in the same tab — 28 fps both.** Region
evaluation costs nothing measurable; the 28 fps ceiling itself is a property of
this specific test session (present with every effect off too, not something
Phase 3 introduced) and not evidence against the "eat the arithmetic" choice.
Given that, all 4 slots x 7 shapes are evaluated **unconditionally**, matching
the uniform invariant everywhere else in this file — shape and mode are just
more `float`-encoded uniforms, selected with `dyno.equal`/`dyno.select` chains
like everything else, never a recompile.

### The six SDFs, and what stayed conservative

Every formula decomposes vec3 inputs into scalars via `swizzle` before doing
arithmetic (`abs`/`sqrt`/`min`/`max`/`clamp` used only on floats, never on raw
vec3), mirroring this file's existing caution about mixed-type dyno calls
(`mix`/`dot` were the ones actually found ambiguous — see Colour section).
`dyno.equal`, `dyno.abs`, `dyno.sqrt`, `dyno.clamp` on floats were previously
unused in this file; they are now confirmed working by the coverage-table
reproduction below, which is the real test, not a synthetic one.

- **sphere**: `length(rel) - size`.
- **ellipsoid**: identical to sphere. With ONE uniform radius driving all 3
  axes (no per-axis control exists), the IQ approximate-ellipsoid formula
  algebraically reduces to the sphere SDF — verified by hand, then by the
  coverage table below matching sphere and ellipsoid to 4 decimal places at
  every tested size.
- **box**: sharp cube, half-extent `size`, the standard exact box SDF
  (`abs(p)-b`, split into outside/inside terms).
- **cylinder**, **capsule**: along local Y. Both radius and extent are driven
  by the same `size` uniform, matching the native system's one-Size-slider
  behaviour exactly (verified numerically, not assumed).
- **plane**: `sdf = -y` (of the position relative to the slot's own centre).
  Negative *above* the centre, i.e. "inside" (what a Cut region removes) is
  the upper half-space — chosen so slot 1 in Cut mode, with an envelope on its
  Y position, reproduces the old Reveal sweep. This is a genuinely different
  shape from Spark's native "plane" (which behaves like a small finite disk,
  not an infinite half-space — measured: native `crop_plane` at the old
  default size gave 0.99% coverage, a small area, not a half-space sweep).
  The native plane was never the target; Reveal's actual behaviour was.
- **all**: `sdf = -1e6`, i.e. always inside — a shapeless whole-scene mask,
  useful for a Colorize with no geometry or as a mask source that always
  reads 1. `crop` + `all` measures identical to `off` (9.73 both), confirming
  it is a true no-op for crop, same as the native system's behaviour.

### Coverage table — reproduced against the removed native system

Same butterfly capture, same canvas, `crop` at `size = 0.35`, softness 0,
measured **before removing the native system** and again **after** with the
new dyno SDFs, driven through the real slot-tab UI:

| shape | native (pre-Phase-3) | dyno (post-Phase-3) |
|---|---|---|
| sphere | 3.451 | 3.451 |
| box | 4.543 | 4.543 |
| ellipsoid | 3.451 | 3.451 |
| cylinder | 4.367 | 4.367 |
| capsule | 4.512 → **4.974 first attempt** | **4.512** after fix |

Capsule was the one real bug this comparison caught: the segment half-length
was originally set to the full `size` uniform, making the capsule visibly
longer than the native reference (4.974% vs 4.512%). The native system's
"scale = length" apparently means *full* length, so the dyno segment half-
extent needed to be `size / 2`, not `size`. Fixed and reproduces the native
figure exactly — this is precisely the "radius vs scale means something
different per shape and getting it wrong fails silently" trap the *previous*
implementation already warned about, now caught the same way: by measuring,
not by reading the formula and assuming it was right.

`cut`/`crop`/`colorize` modes and the `all` shape's no-op behaviour also
reproduced: `cut_sphere_0.5` 4.28–4.29 (native/dyno), `crop_sphere_0.5` 5.935
both, `color_sphere_0.5` R 111.1/111.1 B 17.33/17.34.

### Region — migrating Reveal

Reveal's `revY`/`revS`/`revK`/`revFlip` map onto a plane region in `cut` mode:
`rg{n}Y` from `revY`, `rg{n}Shrink` from `revK`, `invert` from `revFlip`. The
tricky one is `rg{n}Soft` from `revS`, because the two systems use differently
*shaped* soft edges: old Reveal's was one-sided (`smoothstep(revY-revS,revY,y)`
— the whole transition band sits on the "not yet revealed" side of the exact
threshold); the new region mask is symmetric (`smoothstep` centred on the SDF
surface, half the band on each side) — a deliberate choice, kept because a
symmetric band is the more predictable general default for a shape system with
six shapes, not just a sweep plane.

Migrating with a literal 1:1 unit conversion (`softWorld = revS * span`,
`rgSoft = softWorld / radius`) measured a **systematic** offset — every
intermediate point read visibly higher than the pre-migration curve, because a
band centred on the threshold reveals things earlier than one that only
extends below it. Calibrated empirically against the corrected (Phase 2,
object-space-bounds) Reveal sweep table by grid search over a width scale and a
position-offset scale:

```
softWorld  = revS_slider * span              // old world-space band width
rg{n}Soft  = (0.7 * softWorld) / radius       // width: 0.7x, empirically calibrated
rg{n}Y     = (bounds.min + revY*span*1.02 − 0.5*softWorld) / radius
```

The `0.5 * softWorld` offset is exactly derivable — `smoothstep`'s 50% point is
always at the midpoint of its two edges, so old Reveal's own 50%-visible point
sits at `revY_val − revS_world/2`, and that is where the new symmetric band's
own centre needs to land. The `0.7` width factor was found by grid search (a
literal 1.0 conversion measured ~2.7% deviation at the worst sample point,
outside the 2% target) and is not independently derived — recorded here as
"measured, not reasoned to" so a future width tweak knows what it is
recalibrating against.

**Measured end-to-end**, through the actual preset loader (not a standalone
script): a synthetic v3 preset with a linear `revY` envelope (`t:0→v:0`,
`t:1→v:1`), loaded, migrated, then scrubbed through 11 points against the
pre-migration ground-truth curve — **max deviation 0.22%**, comfortably inside
the 2% acceptance figure.

Loading an old preset that used *both* Reveal and the old single-region
feature at once (a real if unusual case) is handled by trying slot 1 for the
old region first, then slot 2 for migrated Reveal if slot 1 is already
claimed — verified: `old region -> R1; old Reveal -> R2 (plane, cut)`, no
collision, both slots correct. The migration status is surfaced to the user
(`Preset loaded — N envelopes. Migrated: …`), not silent, per the "anything not
mappable is skipped with a named message" instruction — though in practice
everything in the old shape *was* mappable, so this file has not yet had to
exercise the unmappable-field path.

### Per-effect masking

Any of `dAmt`, `sat`, tint (`tR`/`tG`/`tB` share one route — masking "tint"
scopes the whole triple, not one channel), `cullA`, `cullS`, `scale` can be
scoped to one region slot's raw geometric mask via a `Mask` control in the lane
header, alongside the existing Audio band/depth widget, shown only when the
focused parameter is one of the six. `src = 0` is "none" — today's unmasked,
everywhere behaviour — `1..4` picks a slot. An independent `invert` flips how
*that one consumer* reads the mask, separate from the region's own `invert`
(which affects that slot's own cut/crop/colorize job) — a region can be doing
its own job and be reused as someone else's scoping mask at the same time.

`effectiveAmount = mix(neutral, V[id], mask)`, neutral chosen per-parameter:

- `dAmt`, `cullA`: **0** — no displacement / no opacity floor, per the plan.
- `sat`, `scale`: **1** — multiplicatively neutral, per the plan.
- `tint`: masking blends toward the **untinted** raw colour (`mix(rgbRaw,
  rgbRaw*tint, mask)`), not toward `ONE3` directly — the tint multiply itself
  is what gets scoped.
- `cullS`: **a large constant (1e6)**, not 0. This is a deliberate departure
  from the plan's literal "`effectiveAmount = V[id]*mask`" formula, which the
  plan states for the zero-neutral cases and does not separately call out for
  `cullS`. `cullS` is a *max-size-allowed* threshold: 0 there means "nothing
  passes", i.e. **cull everything** outside the mask — the opposite of
  neutral. A large ceiling means "nothing gets trimmed by size" outside the
  mask, which is the behaviour "masking Trim to a region" actually implies.
  Reasoned from what makes the parameter a true no-op, matching the plan's
  evident intent rather than its literal wording for this one case.

**Verified spatially, not just numerically**: routing `dAmt` to a small sphere
mask centred on the body left the rendered bbox unchanged (453→453px) while an
*unmasked* `dAmt` at the same value grew it (453→552px) — confirming the mask
actually confines the effect, not merely scales a global number. Moving the
same mask sphere to cover a wingtip made the *masked* displacement visibly grow
the bbox (453→489px) — confirming the effect still fires *inside* the mask,
not that masking silently disabled it everywhere.

### What Region absorbed from Reveal

- `revY`/`revS`/`revK` removed from `SPEC`; `revAnim`/`revFlip` removed from
  `TOGGLES`; the whole `reveal` group and its accordion section are gone.
- The old shrink-into-edge behaviour survives as a per-region `Shrink` (0..1)
  parameter: splat scale multiplies by `mix(1, keepFactor, shrink)`, where
  `keepFactor` is that slot's own alpha-survival factor (`1-mask` for cut,
  `mask` for crop) — the exact generalisation of the old `revK * vis` coupling
  from one hardcoded sweep to any region, any shape, any of 4 slots.

---

**Dev-loop gotcha:** `python3 -m http.server` sends only `Last-Modified`, and
Chrome heuristically caches the page. After the TDZ fix the browser kept running
v1 and the stack traces referenced a `bind()` that no longer existed in the file
— which reads exactly like the bug never got fixed. Hard-reload (Cmd+Shift+R),
or serve with `Cache-Control: no-store`:

```
python3 -c "import http.server; H=http.server.SimpleHTTPRequestHandler; _e=H.end_headers; H.end_headers=lambda s:(s.send_header('Cache-Control','no-store'),_e(s)); http.server.test(HandlerClass=H,port=8000)"
```

---

## Envelopes — per-parameter automation

Roadmap item 3, the one flagged as the biggest functional gap. Before this,
one global `playhead` drove everything and the camera had exactly two
keyframes, so "animation" meant a single linear sweep of whatever happened to
be wired to the timeline.

### Model

```
ENV[id] = { on: bool, smooth: bool, keys: [{t, v}, …] }   // keys sorted by t
```

Two value stores, and the split matters:

- **`P[id]`** — the manual slider value. An envelope never overwrites it, so
  disarming restores exactly what was last dialled in.
- **`V[id]`** — the effective value at the current playhead. Everything
  downstream reads `V`: uniforms, camera ease, readouts, export.

`evalParams()` fills `V` and is called at the top of `pushUniforms()`, so there
is exactly one place where envelopes are resolved. Any new consumer must read
`V`, never `P`.

`t` is normalised 0..1 across the clip, not seconds — so changing `Duration`
rescales the whole animation rather than truncating it.

### Not animatable

`dur`, `br`, `settle`, `dpr` are excluded via `NO_ANIM`. Animating clip length
or bitrate is meaningless; animating sort-settle or pixel ratio would make an
export non-reproducible. Their rows get a hidden placeholder `.key.ghost` so
the four-column grid stays aligned instead of collapsing a column.

### Interaction

- **`◆` per row** — key at playhead, alt-click removes, click focuses the lane.
- **Lane** — drag keys in both axes, double-click empty space to add, alt-click
  a key to delete, click empty space to scrub.
- **Auto-key**: dragging a slider whose envelope is armed writes a key at the
  playhead. Without this the drag would appear to do nothing, since an armed
  envelope owns its parameter.
- An armed slider becomes a **readout** — `syncSliders()` pushes the curve's
  value into it so the panel shows what is actually driving the shader.

Arming an empty envelope is refused; there is nothing to interpolate.

### Export

The offline loop needed no changes. It sets `playhead` then calls
`pushUniforms()`, which resolves envelopes — so exports honour curves for free.

### Watch out

`drawLane()` holds the dragged **key object**, not its index: the array is
re-sorted on every pointermove, so an index would point at the wrong key the
moment a drag crosses a neighbour.

Adding `laneFit()` to `resize()` reintroduces the v1 death: `resize()` runs
during module evaluation, long before the lane's `const`s exist, so it throws a
TDZ error and silently kills everything below it. The lane sizes from CSS and
carries its own resize listener. There is a comment in `resize()` saying so —
leave it there.

---

## Audio-reactive drive

Roadmap item 6, and per the intro the reason this exists rather than an AE
plugin.

### Analysed offline, never sampled live

The audio is analysed **once, offline**, into a per-band amplitude envelope.
It is deliberately not tapped from a live `AnalyserNode`. The offline export
loop does not run at wall-clock speed, so a realtime tap would drift out of
sync the moment you rendered, and the same project would produce a different
result on every run. A precomputed envelope indexed by timeline position makes
scrubbing, playback and export read identical values. Anything else is a demo,
not a tool.

Band separation uses `OfflineAudioContext` + biquad highpass/lowpass per band
rather than a hand-rolled FFT — same result, built-in DSP, no numerical code to
own. Bands: low 20–200, mid 200–2000, high 2000–16000 Hz. Each band is RMS'd
into a 200 Hz envelope and **normalised to its own peak**, so a given `depth`
means the same amount of visible effect regardless of the material.

Timeline mapping is 1:1 with the clock (`playhead * dur` seconds into the
track), not stretched to fit, so changing Duration changes how much of the
track is used — what you want when cutting to a specific passage.

`Preview` plays the track so you can hear it against the animation. It is not
what drives the parameters; the offline envelope always is. So an export
matches what you heard even though nothing is playing during the render.

### Audio modulates, it does not replace

```
V[id] = (envelope or manual value) + depth × parameterRange × bandLevel
```

clamped to the parameter's own range. Audio is a modulation **on top of** the
base value so it layers with envelopes instead of fighting them: the envelope
sets the move, audio adds the detail. Verified — `scale` with an envelope
reading 1.235 at t=3 became 1.422 with a high-band route at depth 0.4, exactly
the base plus `0.4 × 2.95 × 0.16`.

`depth` is a fraction of the parameter's full range and is signed, so a route
can push either way.

### Verified against a synthetic file

Correctness here is not eyeballable, so the test fixture is deterministic:
60 Hz for 0–2s, 800 Hz for 2–4s, 6 kHz for 4–6s. Each band must peak in its own
third. Regenerate with:

```
ffmpeg -y -f lavfi -i "sine=frequency=60:duration=6" \
 -f lavfi -i "sine=frequency=800:duration=6" \
 -f lavfi -i "sine=frequency=6000:duration=6" \
 -filter_complex "[0:a]volume='if(lt(t,2),1,0)':eval=frame[a];\
 [1:a]volume='if(between(t,2,4),1,0)':eval=frame[b];\
 [2:a]volume='if(gt(t,4),1,0)':eval=frame[c];\
 [a][b][c]amix=inputs=3:normalize=0[out]" -map "[out]" -ar 44100 bandtest.wav
```

Measured (low / mid / high):

| t | 1s | 3s | 5s |
|---|---|---|---|
| low | **0.78** | 0.04 | 0.00 |
| mid | 0.10 | **1.00** | 0.09 |
| high | 0.00 | 0.16 | **1.00** |

Off-band bleed of 0.09–0.16 is the second-order biquad skirt, not a defect.

### Focus must not mutate

The `◆` glyph both keys and focuses. That made routing audio impossible without
silently starting an envelope on the parameter — the first attempt at this
produced three unwanted envelopes and the status line said so. **Clicking a
parameter's name focuses it without touching anything.** Keep any future focus
affordance non-destructive.

Relatedly, `syncSliders()` originally keyed off `hasEnv(id)` alone, which left
audio-modulated sliders sitting at their manual position while the shader used
something else. It now uses `isDriven(id)` — envelope *or* audio. This file has
been bitten before by a panel that lies about the state; do not reintroduce it.

---

## Mobile

Roadmap item 7. Desktop stays the target: every mobile rule lives in one
`@media (max-width: 900px), (pointer: coarse)` block at the **end** of the
stylesheet, so it can only ever subtract. Nothing above it is conditional on
screen size.

(It has to be at the end. Placed earlier it lost to equal-specificity base
rules and the touch-target sizes silently did nothing — measured 13px when
22px was intended.)

### Layout

The rail becomes a bottom sheet rather than a permanent 340px column, which is
most of a phone's width. `Controls` toggles it; tapping the viewport dismisses
it. Verified at 390×844: stage 390×692, lane 96, transport 56 — exactly 844,
full-width stage, no horizontal overflow, drawer opening to top 152.

Touch targets go from 13px to 22px. Nothing is removed on mobile — the lane
header scrolls horizontally rather than dropping controls.

### Framing was wrong on any narrow window

`reframe()` used a fixed `radius * 2.6`, derived from the **vertical** FOV.
`camera.fov` is vertical, so on a portrait viewport the horizontal FOV is far
narrower (29.4° at 390×844 vs 50° vertical) and a wide subject overflowed the
frame completely. Now it frames against whichever FOV is tighter:

```js
const hFov = 2 * Math.atan(Math.tan(vFov / 2) * camera.aspect);
const dist = radius / Math.sin(Math.min(vFov, hFov) / 2) * 1.1;
```

On a wide window `hFov > vFov` so this reduces to the old constant — desktop
framing is unchanged. This was a general bug, not a mobile one; it just could
not show up on a landscape window.

### Profile

`Mobile profile` defaults on for coarse pointers or narrow viewports and is a
plain toggle, so desktop can test it. It **caps**, never redefines: the
pixel-ratio slider still reports what you chose, and the OSD reports what is
actually rendered, saying `(capped)` when they differ. Verified: slider at 2
with the profile on gives a 1:1 backing store and an OSD reading of
`1.00 (capped)`; off, 2.00 and a 2× buffer.

It is deliberately **not** in presets — it describes the device you are at, not
the look. A preset saved on a desktop must not force a phone to full pixel
ratio.

### iOS file pickers and `accept` — this bites every input

iOS matches a file input's `accept` against **registered system UTIs**, not
against the filename. Anything it cannot resolve to a known type is greyed out,
and since a picker with nothing selectable gives no error, it reads as the app
being broken.

It hit both inputs for slightly different reasons:

- **Audio.** `accept="audio/*"` alone greys out ordinary .wav/.mp3/.aiff.
  Fixed by listing explicit extensions alongside the wildcard.
- **Splats.** `.ply .spz .splat .ksplat .sog` have no registered UTI at all, so
  listing extensions does not help — there is nothing to resolve them to.

The only reliable fix for the splat case is to **drop `accept` entirely on
touch devices**, which is done under `(pointer: coarse)`. Desktop keeps its
filters, where they behave correctly.

That leaves every picker unfiltered on mobile, so all three buttons and the
drop target now route through one `handleFile()` that dispatches on extension.
Tapping "Choose file" and picking a .json loads it as a preset rather than
failing — the right behaviour once the OS will not filter for us. Unknown
extensions get a named error listing what is accepted.

**Rule for any new file input: never rely on `accept` to gate correctness.**
Validate after selection instead.

`.ply` is now verified end-to-end through this path (a 11 MB capture, 44,296
gaussians, `type: ""` as iOS reports it) — until now every load test had used
the remote `.spz`, so the PLY decoder had never actually been exercised.

### Audio follows the transport

Play/pause/scrub drive the audio element. This is **monitoring only** — the
parameters are always driven by the offline envelope, so an export still
matches what you heard even though nothing plays during a render. The coupling
is deliberately one-way: letting audio `timeupdate` drive `playhead` would make
the timeline jitter with the audio clock and fight scrubbing.

Rapid transport toggling supersedes an in-flight `play()` with a `pause()`,
which rejects as `AbortError`. That is the intended outcome, not a failure —
it is swallowed, because surfacing it put a red warning on the panel during
completely normal use.

### `nonLod` is required, and `lodAbove` is a floor

Spark's loader returns **only** `{ lodSplats }` when it builds a LOD tree,
leaving `packedSplats` empty — the mesh reports 0 splats and renders nothing.
`nonLod: true` makes it also return the full set. Measured without it: 0
gaussians, 0% coverage. With it: 177,132 gaussians and a correct render.

`lodAbove` is a floor, not a ceiling — LOD is built only when the file has MORE
than that many splats. The pre-existing `useLod` checkbox used 500000, above
the 177k test asset, so **that path had never actually executed** and the bug
was latent until the mobile profile lowered the threshold to 150000.

### `preserveDrawingBuffer` is now conditional

It is required for `VideoFrame(canvas)` and costs real performance. Export
needs WebCodecs, which iOS Safari and Firefox lack, so there it was pure cost
for a feature that cannot run. It is now gated on the same `VideoEncoder`
check the export path uses, plus `?debug=1` so pixel probes stay reliable.
NOTES' old warning still holds: it is ON wherever export is possible. The
Render button is also disabled up front when WebCodecs is absent, rather than
explaining itself only after a press.

### Testing note: hidden tabs freeze CSS transitions too

The drawer appeared broken under automation — `panel-open` was set, the rule
parsed and matched, and the transform never changed, even from an inline
style. A hidden tab does not advance CSS transitions, so the computed value
stays frozen at its start. Setting `transition: none` before measuring gives
the true value (top 152, exactly as designed). Same family as the rAF stall
behind `?debug=1`.

---

## Saturation

`sat` lerps between luma-grey and the tinted colour, after the tint multiply,
range 0..2 default 1. It lives in `SPEC` group `colour`, so it inherited reset,
envelopes, audio routing and presets with no extra work.

### `dyno.mix(vec3, vec3, float)` is CONFIRMED

This form was listed above as assumed-but-unverified. It has now been checked
by implementing saturation twice — once as explicit
`grey + (rgb - grey) * sat`, once as `dyno.mix(grey, rgb, U.sat)` — and
measuring both over a fixed 97,707-pixel mask:

| sat | 0 | 0.5 | 1 | 1.5 | 2 |
|---|---|---|---|---|---|
| worst channel difference | 0.000 | 0.000 | 0.000 | 0.000 | 0.000 |

Bit-identical. `mix` is used in the effect chain and the arithmetic form was
discarded. `dyno.dot(vec3, vec3)` is confirmed by the same test.

### The colour in the graph is display-referred, not linear

`renderer.outputColorSpace` is `srgb`, which would normally imply the shader
works in linear light and three converts on output. **It does not behave that
way here.** Measured: halving the red tint takes the mean red from 87.41 to
43.89 — a ratio of **0.5021**. If the graph were linear and the output
sRGB-encoded, that ratio would be ~0.73.

So no encoding step sits between the dyno graph and `readPixels`. The Rec.709
weights `(0.2126, 0.7152, 0.0722)` are defined for linear light, so applying
them here is the *gamma-incorrect* grayscale — which is also what most image
editors do by default. Kept deliberately: matching editor behaviour beats
photometric correctness for a look tool. If a future Spark release starts
applying an output transform, this measurement is the one that will change,
and the luma path is where it will show up first.

### Response is linear below sat = 1 and clamped above it

The acceptance criterion "channel spread at least doubles from sat 1 to sat 2"
measured **×1.89**, not ×2. That is not an arithmetic fault. Over a fixed
mask, predicting each channel as `grey + sat × (mean1 − grey)`:

| sat | 0 | 0.25 | 0.5 | 0.75 | 1 | 1.25 | 1.5 | 2 |
|---|---|---|---|---|---|---|---|---|
| B error vs prediction | 0.00% | 0.50% | 0.90% | 0.83% | 0.00% | 3.89% | 9.32% | 23.25% |
| mask pixels with B ≤ 1 | 0 | 0 | 0 | 0 | 207 | 302 | 344 | 686 |

The error tracks the floored-pixel count exactly. Above `sat = 1` the lerp
pushes channels below zero and the 8-bit framebuffer clamps them on write, so
the composite stops being linear in `sat`. Below 1 it is linear to within
measurement noise. **Not clamping in the shader does not prevent the
framebuffer from clamping** — that is inherent, and the ×1.89 is the correct
consequence of it.

### Measured

| | |
|---|---|
| `sat = 0` | channels within **0.07%** of each other — clean monochrome |
| `sat = 1` | R 87.41 vs 87.40 pre-change — neutral, within 0.01% |
| `sat = 2` | spread 26.83 → 50.67 (×1.89, clamp-limited as above) |
| frame time | **below the measurement floor** — see below |

Frame time could not be resolved in this environment. `performance.now()` is
coarsened to 100µs, `EXT_disjoint_timer_query_webgl2` needs a polling loop that
a hidden tab clamps to 1s per iteration, and the app's own FPS counter is
driven by rAF, which a hidden tab stops entirely. The change is four vector ALU
ops per splat; no instrument available here can see it. Recorded as unresolved
rather than passed.

### Testing hazard: an empty mask fakes a pass

The first `mix` comparison reported "worst difference 0.000 — VERDICT: WORKS"
while every measured value was `NaN`. The mask had been captured during the
load transient, so it was empty, every mean divided by zero, and `NaN`
comparisons silently evaluated false in the `Math.max` accumulator. **Assert
the mask is non-empty and every value finite before comparing.** A pass built
on NaN looks exactly like a real pass.

---

## Section accordion and `SPEC` groups

`SPEC` entries are `[id, digits, group]`. The group ties a parameter to its rail
section, so resets, envelopes, audio routing and preset persistence all stay
driven by `SPEC` as it grows — the dev plan takes it from 26 parameters to
roughly 60, and retrofitting grouping around that would be far worse.

Sections are native `<details>`/`<summary>`: keyboard accessible for free, works
inside the mobile bottom sheet, and no open/closed state machine to own beyond
persisting it.

`reveal` is a group on borrowed time — Phase 3 of the dev plan folds it into
`region` — but it has to exist while its parameters do.

### Summary markers

A collapsed group must never hide that something inside it is animating. The
summary carries, in `--edit` violet:

- a **count** when any parameter in the group has an armed envelope or an audio
  route,
- a **·** when nothing is driven but something is off its default,
- nothing when the whole group is at default.

This mirrors the per-row `.dirty` convention so the rail reads the same
collapsed or expanded. Note that a reset-all clears the dots but keeps the
counts — resetting a slider does not disarm an envelope, and pretending
otherwise would be a lie about what is driving the render.

`refreshGroups()` is called from `refreshEnv()` — the single choke point every
envelope mutation already routes through — plus the slider input listener,
audio routing changes and preset load. It is deliberately **not** called from
`markDirty()`: `syncSliders()` calls that per driven parameter on every frame
during playback, and `refreshGroups()` walks all of `SPEC`.

### Open/closed state is in localStorage, not presets

Same reasoning as `Mobile profile`: it describes your workspace, not the look.
Key `splatbench.accordion.v1`. Nothing is written until you actually toggle
something, so a first visit gets the defaults: `camera` and `output` open.

### Measured

| | |
|---|---|
| rail content, all collapsed | 466px (fits a 900px viewport) |
| rail content, defaults open | 882px |
| rail content, all open | 2608px |
| summary row height | 41px desktop and mobile |
| mobile 390×844 | stage 692 + lane 96 + transport 56 = 844, no overflow |

### A note on comparing coverage across sessions

The Phase 0 regression initially looked like a uniform ~2% drop against every
figure in this file. It was not a regression: the browser window was a
different size, so the canvas was 1310×782 rather than the 1460×891 the
baselines were taken at, and **coverage is a percentage of canvas area**.

Colour channels were unchanged, which was the clue. The size-independent check
is the ratio of each case to its own BASE, which cancels canvas geometry — those
matched the recorded figures to within 0.003. **Record the canvas size next to
any coverage figure, or compare ratios.**

### Warm-up: "stable" is not enough, it must be stable AND non-zero

The documented warm-up rule — probe until two consecutive reads agree — is not
sufficient on its own. Two consecutive **zeros** agree, and zero is exactly the
not-yet-warm signature. The rule is: two consecutive agreeing reads that are
also non-zero, with a cap.

---

## Gesture mode — direct manipulation

On a phone the rail covers the subject, so adjusting a slider means not seeing
what it does. `Adjust` binds up to two parameters to the drag axes and edits
them on the viewport itself. It works on desktop too — a mouse drag is a drag.

Chips cycle **unbound → X → Y → unbound** on single taps; binding an axis that
is already held displaces the previous holder rather than refusing. A full drag
across the stage sweeps the parameter's full range, and drags are relative to
where the pointer went down, so repeated strokes keep nudging.

Two decisions worth keeping:

- **It writes through the slider's own `input` event**, never to `P` directly.
  That is the single path already handling auto-key on armed envelopes, dirty
  marking, the `dpr`/`dur` side effects and `pushUniforms`. A second write path
  would drift out of sync with it. Verified: dragging with an armed envelope
  adds a keyframe, exactly as dragging the slider does.
- **Orbit and adjust are modal, not gesture-count-based** (one finger vs two).
  On a trackpad and on a phone those are easy to confuse, and a mis-detected
  gesture silently edits a parameter instead of moving the camera. An explicit
  mode is duller and much harder to get wrong. `controls.enabled` is the
  interlock, so the two can never both be live.

---

## Colour

Three accents, and they mean different things. Do not reach for a fourth
without a reason.

| token | use |
|---|---|
| `--signal` mint | values, readouts, audio routing |
| `--live` orange | transport, recording, a key **at** the playhead |
| `--edit` violet | automation affordances: armed envelopes, changed-from-default, gesture bindings |

The reset and key glyphs originally sat on `--rule`, which is the *border*
colour — near invisible against the chassis by design. They now idle at
`--ink-dim` at 55% opacity and go to `--edit` when they carry state, at 16px
desktop / 26px mobile. Idle-but-legible beats invisible: these are the only
indication that a parameter is automated.

---

## Presets — save/load as JSON

Envelopes alone do not round-trip a scene. A disarmed parameter reads from `P`,
so restoring curves without the underlying slider values gives you a file that
does not reproduce what you saved. The preset is therefore the whole editable
state: parameters, envelopes, toggles, camera keys, output settings.

The splat is referenced **by name only** (`asset`), never embedded. Captures are
hundreds of MB and often not redistributable; embedding one would destroy the
point of a small shareable text file. Loading a preset does not load a capture.

### Shape

```json
{
  "format": "splat-bench-preset",
  "version": 3,
  "saved": "…", "asset": "butterfly.spz",
  "params":    { "cullA": 0.3, … },        // all 18, manual values (P, not V)
  "toggles":   { "revFlip": true, … },
  "output":    { "res": "1920x1080", "fps": "30" },
  "camera":    { "A": {"pos":[…],"tgt":[…]}, "B": … },
  "envelopes": { "revY": { "on": true, "smooth": true,
                           "keys": [{"t":0,"v":0},{"t":1,"v":1}] } },
  "selects":   { "rgMode": "crop", "rgShape": "box", … },      // v2
  "audio":     { "name": "track.wav",                          // v3
                 "routes": { "dAmt": {"band":"low","depth":0.6} } }
}
```

`params` stores `P`, the manual values — never `V`. Saving the evaluated values
would bake whatever the playhead happened to be sitting on into the file.
Envelopes with no keys are not persisted.

### Loading is deliberately forgiving

A preset is a plain text file someone may well hand-edit, so nothing in it is
trusted:

- unknown parameter and envelope ids are skipped silently
- every number is clamped to its own control's `min`/`max`; `t` is clamped 0..1
- non-finite values are dropped, keys are re-sorted by `t`
- an envelope with `on: true` but no keys is left disarmed
- `version` newer than the build is refused with a message rather than
  half-applied
- envelopes are **wiped before applying**, so a load is a replace and never a
  merge with what was already there

Verified against malformed JSON, wrong format, missing version, future version,
out-of-range params, unknown ids, non-animatable ids and empty key lists. None
kill the module; each reports through the status line.

Bump `PRESET_VERSION` on any breaking shape change and handle the old shape in
`applyPreset`, rather than letting the version check reject files it could have
migrated.

### Round-trip

State → save → wipe → load reproduces the fingerprint byte-exactly (all 18
readouts, toggles, armed envelope count, camera key flags, output settings).
Dropping a `.json` on the viewport routes to the preset loader; anything else
still goes to the splat loader.

---

## Spark API points — verified vs assumed

Verified against the Spark 2.x docs:

- `new SplatMesh({ url | fileBytes, objectModifiers, lod, onProgress, onLoad })`
- `await mesh.initialized`, `mesh.getBoundingBox(centers_only)`
- `dyno.dynoBlock(inTypes, outTypes, closure)`
- `dyno.splitGsplat(g).outputs` — the migration guide uses `.outputs`; the
  overview page shows direct destructuring. `.outputs` is the one to trust.
- `dyno.combineGsplat({ gsplat, center, scales, rgba })` injects components
- `Gsplat` struct fields: `center, flags, scales, index, quaternion, rgba`
- stdlib: `mix, smoothstep, select, and, greaterThanEqual, swizzle, extendVec,
  hashVec3, sin, mul, add, sub, max`

Confirmed at runtime on the butterfly (177,132 gaussians), 2026-08-16:

- `mesh.updateVersion()` — exists, bumps `this.version`, drives the regenerate
  gate. **Required after any uniform write.**
- `dyno.swizzle` / `hashVec3` / `smoothstep` / `select` / `and` /
  `greaterThanEqual` / `lessThanEqual` / `extendVec` / `combineGsplat` — all
  behave as documented; every effect measurably responds.
- `splatCount()` resolves — the `numSplats ?? packedSplats.numSplats` fallback
  reports the right count. Which branch fires is still untested.

**Still assumed, not exercised by current code:**

- `dyno.split(scales).outputs.x/.y/.z` — the code deliberately uses three
  `swizzle()` calls instead, so this remains unverified.

**Confirmed since, each by a real coverage-table reproduction, not a synthetic
test — see Colour (saturation) and Region for the measurements:**

- ~~`dyno.mix()` with a `vec3` a/b and a scalar `t`.~~ `dyno.dot(vec3, vec3)`.
- `dyno.equal`, `dyno.abs`, `dyno.sqrt`, `dyno.clamp`, `dyno.min` on floats —
  used throughout the Region SDFs; all six shapes reproduced the removed
  native system's coverage table to 3 decimal places (one bug found and fixed
  in capsule's segment length, not in these primitives).

Splat captures are usually Y-down relative to three.js. This used to be a
hardcoded `mesh.quaternion.set(1, 0, 0, 0)` after load; since the Model
transform phase it is `mdRotX = 180`, the default of a real control — see
"Model transform — Correct mode".

---

## Export path

WebCodecs `VideoEncoder` (`avc1.640033`) → `mp4-muxer` → Blob download. Renders
at a fixed output resolution independent of viewport size; restores viewport
state in a `finally`. Chrome/Edge only — Safari and Firefox will fail the
`typeof VideoEncoder` guard.

**The real hazard here is the sort, not the encoder.** Spark determines
back-to-front draw order by reading splat distances back off the GPU and bucket
sorting on a worker thread. That is asynchronous. A frame rendered immediately
after a camera move can be blended in stale order. The offline loop currently
renders, waits `settle` ms, renders again, then captures. 24 ms is a guess.

If exported frames show wrong-order blending that the live view does not, that
slider is the cause. A proper fix would hook whatever signal Spark exposes for
"sort for this viewpoint is current" instead of sleeping — worth investigating
in the Spark source (`SparkViewpoint`, `SplatWorker`).

Note the interaction with the version gate: `pushUniforms()` now calls
`updateVersion()`, so each export frame forces a regenerate *and therefore a
re-sort*. That makes the `settle` wait more load-bearing than it was, not less.
Before this fix the offline loop was also silently rendering unchanged splat
data every frame — any exported animation of an effect parameter would have come
out frozen except where the camera moved.

### Verified end-to-end, 2026-08-16

First real export since the `updateVersion()` fix. 30 frames, 1280×720, 30fps,
`dur=1`, reveal sweeping via `revAnim`, camera dollying A→B.

- Container exactly as configured: h264 High, 1280×720, 30/1, `nb_frames=30`,
  `duration=1.000000`, yuv420p.
- **Effects animate.** Mean luma per frame rises monotonically 24.0 → 38.6
  across all 30 frames, in an S-curve matching the camera ease. Pre-fix this
  would have been flat except where the camera moved.
- Requested 20 Mbps, got 5.1 Mbps actual. Not a defect — the encoder treats
  bitrate as a ceiling and this content is easy. Do not "fix" this.

Check animation with, rather than by eye:

```
ffprobe -v error -f lavfi -i "movie=out.mp4,signalstats" \
  -show_entries frame_tags=lavfi.signalstats.YAVG -of csv=p=0
```

`preserveDrawingBuffer: true` is set on the WebGLRenderer for reliable
`VideoFrame(canvas)` capture. It costs some performance; do not remove it
without re-testing export.

---

## Roadmap, roughly in order

1. ~~Fix the TDZ bug, add the error surface, confirm the butterfly renders.~~
   **Done.**
2. ~~Confirm the effect stack — displace, reveal, trim, tint each in isolation
   on a real capture.~~ **Done**, measured rather than eyeballed: every slider
   and checkbox verified against pixel coverage/luminance/bbox on the butterfly.
   This is what surfaced the missing `updateVersion()`. `br` and `settle` are
   now exercised too — see the verified export run below.
3. ~~**Per-parameter envelopes.**~~ **Done** — see the Envelopes section above.
   Keyframe lists per parameter, a curve lane with draggable keys, auto-key on
   armed sliders, honoured by the export path, and persisted as JSON presets.
4. ~~**SDF-shaped edits** via Spark's `SplatEdit` / `SplatEditSdf` /
   `SplatEditSdfType`.~~ **Done** — see the Region section above. Cut, crop and
   colorize across six shapes, animatable and persisted like any other
   parameter.
5. Depth pass output for compositing.
6. ~~Audio-reactive parameter drive (Web Audio FFT → uniforms).~~ **Done** —
   see the Audio-reactive drive section above. Offline three-band analysis,
   signed depth per parameter, layered on top of envelopes, persisted as
   routing in presets.
7. ~~Mobile: `lod: true` plus reduced pixel ratio, gated behind a toggle.~~
   **Done** — see the Mobile section above. Bottom-sheet rail, touch targets,
   pixel-ratio cap, forced LOD, and an aspect-aware `reframe()` that fixed a
   framing bug affecting any narrow window.
8. Multiple `SplatMesh` instances in one scene with independent transforms.
   **Deferred to last on 2026-08-16** — not needed at this stage. Nothing is
   blocked on it, and the Region work already anticipates it: edits are scoped
   per mesh via `mesh.add(edit)`, so regions will not leak between meshes when
   this does land.

## Deployment — GitHub Pages

Served from `main` at the repo root. Three things make this work, and each was
checked rather than assumed:

- **`index.html`.** Pages serves no directory index otherwise; the root URL
  would 404.
- **`.nojekyll`.** Pages runs Jekyll by default, which silently drops paths
  beginning with `_`. Nothing currently starts with an underscore, but the
  failure mode is a missing file with no error, so the guard stays.
- **No cross-origin isolation needed.** Spark 2.1.0 does not use
  `SharedArrayBuffer` (checked: zero occurrences), which matters because Pages
  cannot set COOP/COEP headers. If a future Spark release adopts it, Pages
  stops being a viable host and that is the reason why.

Both remote dependencies serve `access-control-allow-origin: *` — the pinned
`spark.module.js` and the `butterfly.spz` test asset — so the CDN importmap and
Load test asset both work from the Pages origin, not just localhost.

Pages is HTTPS, so the secure-context requirements (WebCodecs, `AudioContext`)
are satisfied. Export still needs Chrome or Edge.

There is no build step and nothing to configure: a push to `main` republishes.

## Non-goals

- Reimplementing SuperSplat's cleanup UI. Use SuperSplat for cleanup, load the
  cleaned file here.
- A build step or npm dependency tree. Single file, CDN imports, works from
  GitHub Pages.
- Runtime LLM generation of GLSL. Effects should be authored ahead of time as
  parameterised dyno graphs; anything model-driven should map language onto
  parameters and keyframes of graphs that already compile, not emit shader code
  the user cannot debug.
