# Splatification — project notes

Working folder: `/Users/55147209/Documents/Claude_Builds/Splatification`
Main file: `splat-bench.html` (single file, no build step, no npm install)
Serve with `python3 -m http.server` and open `http://localhost:8000/splat-bench.html`.
ES modules will **not** load over `file://` — it must be served.

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

**`br` and `settle` have never been exercised at runtime.** Both are correctly
wired (`P.br * 1000000` into `encoder.configure`, `P.settle` into the offline
loop) but only take effect during an actual export, which has not been run
end-to-end since the `updateVersion()` fix. First real export is the test.

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

**Dev-loop gotcha:** `python3 -m http.server` sends only `Last-Modified`, and
Chrome heuristically caches the page. After the TDZ fix the browser kept running
v1 and the stack traces referenced a `bind()` that no longer existed in the file
— which reads exactly like the bug never got fixed. Hard-reload (Cmd+Shift+R),
or serve with `Cache-Control: no-store`:

```
python3 -c "import http.server; H=http.server.SimpleHTTPRequestHandler; _e=H.end_headers; H.end_headers=lambda s:(s.send_header('Cache-Control','no-store'),_e(s)); http.server.test(HandlerClass=H,port=8000)"
```

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
- `dyno.mix()` with a `vec3` a/b and a scalar `t`. Both current `mix()` call
  sites pass floats, so the vec3 form is still untested.

Splat captures are usually Y-down relative to three.js, hence
`mesh.quaternion.set(1, 0, 0, 0)` after load.

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
   This is what surfaced the missing `updateVersion()`. Still unexercised at
   runtime: `br` and `settle`, which only take effect during an actual export.
3. **Per-parameter envelopes.** Currently one global `t` drives everything and
   the camera has exactly two keyframes. A real animator needs per-parameter
   automation curves. This is the biggest functional gap.
4. **SDF-shaped edits** via Spark's `SplatEdit` / `SplatEditSdf` /
   `SplatEditSdfType` — box and sphere region colorize/clip. This is the
   Irrealix "crop with spherical or box shape" primitive and Spark supports it
   natively; no need to hand-roll it in dyno.
5. Multiple `SplatMesh` instances in one scene with independent transforms.
6. Depth pass output for compositing.
7. Audio-reactive parameter drive (Web Audio FFT → uniforms). This is the reason
   for building this rather than using an AE plugin.
8. Mobile: `lod: true` plus reduced pixel ratio, gated behind a toggle.

## Non-goals

- Reimplementing SuperSplat's cleanup UI. Use SuperSplat for cleanup, load the
  cleaned file here.
- A build step or npm dependency tree. Single file, CDN imports, works from
  GitHub Pages.
- Runtime LLM generation of GLSL. Effects should be authored ahead of time as
  parameterised dyno graphs; anything model-driven should map language onto
  parameters and keyframes of graphs that already compile, not emit shader code
  the user cannot debug.
