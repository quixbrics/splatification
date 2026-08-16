# Splatification — Splat Bench

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

Then open <http://localhost:8000/splat-bench.html> and click **Load test asset**.

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

Every parameter has a `↺` reset button, which brightens when the value differs
from its default.

## Notes

[NOTES.md](NOTES.md) carries the architecture, the Spark API surface (verified
vs assumed), known issues, and the roadmap. Read **The uniform invariant**
section before adding an effect — Spark gates its generate pass, and a uniform
write that does not bump the version is silently invisible.

Splat captures are gitignored; the app loads a remote test asset so the repo is
runnable as cloned.
