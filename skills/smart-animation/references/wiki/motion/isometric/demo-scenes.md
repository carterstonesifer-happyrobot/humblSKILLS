---
title: "Isometric Demo Scenes: SVG Scenes Driven by a Sim Clock"
context: motion
category: isometric
concept: demo-scenes
description: "Build a standalone animated isometric (2.5D) scene for live demos in hand-built SVG + vanilla JS: scene state is a pure function of one sim clock, triggers change it, no 3D library."
tags: isometric, svg, demo, simulation, triggers, gantt, brand-styling, standalone-html
sources:
  - "references/raw/isometric-3d-models-SKILL.md"
last_ingested: 2026-11-23
---

## Isometric Demo Scenes

For a demo that needs a living place (warehouse, pharmacy, hospital, airport,
data center) with props, people, and machines, hand-build an isometric SVG
scene whose entire state is a function of one simulation clock `T`. It is
crisper and lighter than Three.js, fully controllable, and has no CDN or CSP
dependency. Output is one standalone `.html` file (inline CSS/JS).

**Incorrect (free-running per-object timers and a 3D library):**

```js
// Each actor keeps its own setInterval; scrubbing, pause, and Reset drift apart.
// Three.js + CDN for a flat-illustration look that SVG draws better.
trucks.forEach(t => setInterval(() => t.x += 1, 50));
```

**Correct (one clock, pure render, triggers as data):**

```js
let T = 0;                                   // sim minutes of day
function frame(dt) { T += Math.min(dt, .05) * rate; render(T); requestAnimationFrame(frame); }
const TRIGGERS0 = [
  { id: 'door-ajar', at: 783, target: 'FR3', effect: { type: 'door', state: 'ajar' }, log: 'Door left ajar' },
  { id: 'arrival',   when: s => s.queue > 3, effect: { type: 'spawn', kind: 'truck' } },
];
```

### Workflow

1. **Research first.** How the real place is organised (zones, fixtures,
   thresholds); cite sources and say what was inferred. If a company or URL is
   given, extract brand colors (try `/brand`, `/press`, `/media-kit`, then search).
2. **Spec briefly:** zones, props, actors (2-8 per type), the trigger
   storyline, what the panel shows. Ask only what cannot be inferred.
3. **Build** the single file, **verify** in Node, **deliver**.

### Rules that matter

- **Brand:** use colors and plain-text names only. Never redraw logos or
  marks. Keep status colors (ok green, warn amber, danger red) distinct from
  the brand color, or use red for danger consistently if the brand is red.
- **Clock:** `T += dt*rate`, `dt` clamped to 0.05 s, wrap at the end.
  Precompute per-minute series at init so charts and min/max are lookups.
  Start where every interesting state is visible; gentle rate (0.5-1
  sim-min/s) plus faster options; always a Reset (deep-clone seed data).
- **Iso kit:** `P(x,y,z)=[ox+(x-y)*C*s, oy+(x+y)*SC*s-z*s]`, `C=.866`, `SC=.5`;
  materials are `[top,left,right]` triples with stroke = fill. 1 unit ≈ 1 m.
- **Draw order:** painter's order by `x+y`, far to near. Use a cutaway (two far
  walls only); put entrances and vehicles behind the back wall. Actors get a
  layer before each tall row plus a top layer, re-appended only when order changes.
- **Actors:** build heading x state variants once in `<defs>`, place with
  `<use>` and update only `transform` (always include `ox,oy`). Few actors,
  purposeful motion; exec demos prefer calm over busy.
- **Triggers:** scheduled (`at`), conditional (`when`), manual (control-bar
  buttons), external (`SCENE.trigger()` / `postMessage`). A fired trigger
  inserts a timeline event and rebuilds the model, so scrubbing stays
  consistent. Log each one and emit `scene:trigger` / `scene:status`.
- **Status in-scene:** dark pill tags with a leader line; stagger leader
  heights; reinforce status redundantly (light, readout, tint, floor halo).
- **Panels:** status cards with sparklines, a Gantt (drag, resize, conflict
  validation), an event log, and one hover tooltip for every object.
- **Integration:** `window.SCENE` API mirrored as `postMessage`; post to
  `parent` only when embedded.
- **Verify:** `node --check` the extracted script; run against a DOM stub for
  2000-4000 frames; assert storyline states at checkpoints; scan all output
  for `NaN|undefined|Infinity`. Say honestly if visuals were not browser-checked.

Motion here must still follow the shared principles: purposeful, compositor-
friendly where it touches the DOM, and a reduced-motion path for the
control-bar playback (pause by default under `prefers-reduced-motion`).

In a Next.js app, port the scene instead of embedding the HTML: read
`nextjs-components.md` for the engine / wrapper / board split and the chart,
Gantt, and stat-row aids.

## Sources

- `references/raw/isometric-3d-models-SKILL.md` - the full recipe (kit code
  sketches, perspective lessons, panels, API, proven reference scenes).
  Read it when building a scene; this concept is the decision layer.
