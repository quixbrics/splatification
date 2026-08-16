# Splatification — Splat Bench

**[Live demo →](https://quixbrics.github.io/splatification/)** (Chrome or Edge for video export)

A bespoke 3D Gaussian Splatting manipulator and animator for the browser.
Arbitrary per-Gaussian programmable manipulation via Spark's `dyno` shader-graph
system, with an eye toward parameter automation driven from outside (audio FFT,
LFOs, OSC).

Single file, no build step, no npm install. Dependencies are CDN imports via an
importmap: [three](https://threejs.org) 0.180,
[Spark](https://sparkjs.dev) 2.1.0, mp4-muxer 5.x.

## Run

ES modules will not load over `file://` — it must be served.

```bash
python3 -m http.server
```

Then open <http://localhost:8000/> and click **Load test asset**.

Chrome or Edge is required for video export (WebCodecs `VideoEncoder`).

Because the whole app is one cached HTML file, use a hard reload (Cmd+Shift+R)
after editing, or serve with no-store headers:

```bash
python3 -c "import http.server; H=http.server.SimpleHTTPRequestHandler; _e=H.end_headers; H.end_headers=lambda s:(s.send_header('Cache-Control','no-store'),_e(s)); http.server.test(HandlerClass=H,port=8000)"
```

## Effects

All effects are a single `dyno.dynoBlock` running in one GPU pass as an
`objectModifier` on the `SplatMesh`:

- **Displace** — `sin()` field over swizzled center × per-splat random direction
- **Reveal** — soft `smoothstep` sweep plane, shrinking splat scales into the edge
- **Trim** — per-splat cull by minimum opacity and maximum scale
- **Tint** — RGB multiply
- **Region** — Spark's native SDF edit: cut, crop or colorize inside a
  sphere / box / ellipsoid / cylinder / capsule / plane

**Audio** drives any parameter from a track's low / mid / high content. The file is
analysed offline into per-band envelopes rather than sampled live, so scrubbing,
playback and export all agree. Click a parameter's name to focus it, then pick a band
in the lane. Audio layers on top of envelopes rather than replacing them.

**Adjust** binds up to two parameters to the drag axes so you can change them on
the viewport itself, with the splat visible — tap chips to assign X and Y, then
drag. Useful on desktop, essential on a phone where the control rail covers the
subject.

Every parameter has a `↺` reset button, which brightens when the value differs
from its default, and a `◆` keyframe button.

## Animation

Each parameter carries its own envelope — a keyframe list over the normalised
0..1 timeline, with smooth or linear interpolation. Click `◆` on a row to key it
at the playhead and focus it in the curve lane; drag keys in the lane, or drag
the slider itself to auto-key. Armed sliders become live readouts of their
curve. Exports honour envelopes.

**Save JSON** writes the whole editable state — parameters, envelopes, toggles,
camera keys — as a small text file. The splat is referenced by name only, not
embedded, so load your capture first and then the preset. Drop a `.json` on the
viewport to load it.

Keep the tab visible while exporting — Chrome throttles background timers to
~1s, which slows the render to roughly a frame per second.

## Notes

[NOTES.md](NOTES.md) carries the architecture, the Spark API surface (verified
vs assumed), known issues, and the roadmap. Read **The uniform invariant**
section before adding an effect — Spark gates its generate pass, and a uniform
write that does not bump the version is silently invisible.

Splat captures are gitignored; the app loads a remote test asset so the repo is
runnable as cloned.

## Mobile

The rail becomes a bottom sheet on narrow or touch screens, with larger touch
targets. A **Mobile profile** toggle (on by default there) caps the pixel ratio
and forces level-of-detail on load; it caps rather than redefines, and the OSD
reports the pixel ratio actually being rendered. Video export needs WebCodecs,
so the Render button is disabled on browsers without it (including iOS Safari).

## Debugging

Load with `?debug=1` to expose `window.__bench` — a synchronous `draw()`, an
awaitable `settled()`, and live access to the mesh, edit and evaluated
parameters. Rendering is rAF-driven and Chrome stops rAF in a hidden tab, so
this is the only reliable way to measure pixels from an automated harness.
