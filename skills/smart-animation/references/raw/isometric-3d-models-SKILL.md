---
name: "isometric-3d-models"
description: "Create demo-ready animated isometric (2.5D \"3D\") scene components from a prompt — any setting (warehouse, pharmacy, hospital, factory, airport, store, office, data center) — styled from a company's brand/media pages when a link is given, with deterministic clock-driven animation, scripted and API-fired triggers that change the scene, optional Gantt/status panels, and a standalone HTML (or React) deliverable."
---

# Isometric demo scenes

Build a self-contained, animated isometric flat-illustration scene for live demos: a branded environment (props, people, vehicles, machines) whose state is a pure function of a sim clock, plus triggers that visibly change the scene (alarms, arrivals, failures, bookings), plus an optional data panel (status cards, charts, Gantt). No external libraries — hand-built SVG + vanilla JS beats Three.js here (crisper, lighter, fully controllable, no CDN/CSP issues).

## Workflow
1. **Research** (before any code)
   - **Domain layout**: search how the real place is organised (zones, fixtures, workflows, regulations/thresholds). Cite sources; say what was inferred vs. confirmed.
   - **Brand** (if a company or URL is given) → see *Brand styling*.
2. **Scene spec** (write it down briefly; ask only for what can't be inferred): zones, static props, actors (≤ 2–6 per type), the storyline of triggers, what the panel shows, deliverable.
3. **Build** the single file (sections below).
4. **Verify** in Node (syntax + DOM-stub run + NaN scan + assert the storyline states occur at the intended times).
5. **Deliver** with `present_files`; summarise the storyline, controls and API in a few lines; say visuals were not browser-checked if true and what to look at.

## Brand styling (from a link or company name)
- Try the given URL, then common paths: `/brand`, `/brand-guidelines`, `/press`, `/newsroom`, `/media`, `/media-kit`, `/about/brand`; then search `"<company> brand guidelines" colors hex`, `"<company> media kit"`, `"<company> press kit logo usage"`. If fetching is blocked, rely on search results and say so.
- Extract: primary/secondary hex colors, neutrals, typography feel, imagery motifs (e.g. rounded vs. angular, flat vs. textured), tone.
- Map to scene tokens: `brand` (accents: vehicle cabs, stripes, counter bands, slab edge, alarm-safe accents), `brandDark` (shaded faces), `neutral` support colors (walls, floors, goods). Label non-official neutrals as such.
- Use brand **colors and plain-text names only** (e.g. "PHARMACY" in brand color). Never redraw logos, mascots or trademarked marks.
- Keep semantic status colors distinct from brand where possible (ok = green #1D9E75, warn = amber #E8961C, danger = red). If the brand is red, red may double as danger—use it consistently.

## Architecture
```
KPI cards → isometric SVG scene → control bar (▶/❚❚, clock, scrubber, rate, scenario/trigger buttons, reset)
→ panel (status cards with sparklines | Gantt | event log) → shared hover tooltip
```
- One IIFE. Seed data (`CFG0`, `ITEMS0`) is deep-cloned for Reset. Runtime objects are derived on `init()`.
- **Everything is a function of sim time `T`** (minutes of day). Loop: `T += dt*rate` (dt clamped to 0.05 s), wrap at end, `render(T)`. Precompute per-minute series (temps, loads) at init so charts/min/max/out-of-range are lookups.
- Defaults for demos: start at a moment where every interesting state is visible at once; gentle rate (0.5–1 sim-min/s) with faster options; Reset button.

## Isometric kit
```js
const C=.866,SC=.5; let s,ox,oy;
const P=(x,y,z=0)=>[ox+(x-y)*C*s, oy+(x+y)*SC*s - z*s];
const pts=a=>a.map(p=>P(p[0],p[1],p[2]).map(v=>v.toFixed(1)).join(',')).join(' ');
const poly=(a,f,g,st,sw)=>el('polygon',{points:pts(a),fill:f,stroke:st||f,'stroke-width':sw||.6,'stroke-linejoin':'round'},g);
function box(x,y,z,w,d,h,[top,left,right],g){ /* +x face, +y face, then top */ }
const fy=(x1,x2,y,z1,z2,f,g)=>/* quad on a +y-facing plane (fronts, screens, doors, bands) */;
const fx=(x,y1,y2,z1,z2,f,g)=>/* quad on a +x-facing plane (side walls) */;
function ftext(x,y,z,str,size,fill,g){ /* text skewed onto a +y plane: transform=matrix(C,SC,0,1,X,Y) */ }
```
- 1 unit ≈ 1 m. Fit: `s=min(22, 640/((W+D)*C))`, `ox=(680-(W+D)*C*s)/2+D*C*s`, `oy` = tallest height·s + margin (include floating labels), viewBox height from the front corner.
- Materials as `[top,left,right]` triples (top lightest). Stroke = fill to hide seams. Wheels/discs: 12–16 sampled points in the face plane.
- Look: warm light background, white slab with a thick brand-colored edge on the two visible sides, flat faces, dark structural lines, cream/beige goods, brand accents. UI chrome uses CSS variables (light/dark), 0.5px borders, 12px cards, sentence case, weights 400/500, no emoji, no icon fonts (use ❚❚ ▶ ▾ ▸).

## Perspective & draw order (lessons that matter)
- Painter's order by `x+y` (far → near). A height-h object hides the floor shifted (−h,−h).
- **Cutaway**: only the two far walls (back y≈0, left x≈0). Put entrances/docks/vehicles behind the back wall so they never cover the interior. Fixtures that face the viewer go against the back wall (their +y face shows).
- Static order: slab → floors/markings → things behind back wall → walls → wall fixtures (shelving, fridges, sorted by x) → mid islands → near-mid racks/bins → **actor layer** → front counter/near fixtures → foreground.
- Composite objects: draw the farther part first (e.g. tractor cab before the trailer that sits nearer the viewer).
- Actors among tall rows: create one actor layer before each tall row plus a top layer; per frame `layer = rows the actor is fully in front of` (top layer in open cross-aisles); sort by layer then `x+y`; re-append only when the order signature changes. Flat moving items (conveyor boxes) go under tall rows so they pass behind.
- Leave ≥ object-height of clear floor between rows/aisles so layering is unambiguous.
- Doors/panels that swing: hinge on one edge, rotate the free edge `(hx+w·cosθ, y+w·sinθ)` toward the viewer; draw after the body.

## Actors
- Part lists `[x,y,z,w,d,h,palette]` around the origin; build variants (heading × state) into `<defs>` with `ox=oy=0`, place with `<use href>` and only update `transform`.
- **Always** `translate(ox+(x-y)*C*s, oy+(x+y)*SC*s)` — omitting `ox,oy` floats actors off the floor.
- 90° rotation of parts: rot1 `(x,y,w,d)→(-(y+d),x,d,w)`, rot2 `→(-(x+w),-(y+d),w,d)`, rot3 `→(y,-(x+w),d,w)`; re-sort parts.
- People: legs, torso (uniform/coat color), small brand accent, head (vary skin), hair; 1–2px bob while walking; soft shadow ellipse (`rx=r·s·1.2247, ry=r·s·.7071`) and a selection ring.
- **Simplicity**: few actors (people ≤ 2–8), no decorative roaming, purposeful motion only. Exec demos prefer calm over busy.

## Behavior from time
- Itineraries: cyclic stops `[[station,dwell],…]` plus specials `{t,station,dwell,tag}` that interrupt a dwell; route on orthogonal corridors (`route(a,b)` via a known corridor x/y); walk speed in sim m/min tuned to the playback rate.
- Path loops for machines: `loopPos(path,tau,speed,dwellA,dwellB)` → position, heading, carrying, state. Robots should dwell at handoff points (conveyor, station).
- Couple behavior to data: doors open only while a vehicle is present; staff work only while on shift and their resource is occupied; door-ajar raises temperature; access spikes are small and recover.
- Signals: physical-ish curves (`base + small sinusoid noise + drift + Σ event responses`, rise `amp·(1−e^{−t/τ})`, recovery `e^{−t/τ_r}`), classified into ok / near-limit (within configurable margin) / out-of-range.

## Triggers (make the scene change during a demo)
Define triggers as data so new ones need no new plumbing:
```js
const TRIGGERS0=[
 {id:'door-ajar', at:783, target:'FR3', effect:{type:'door', state:'ajar'}, log:'Door left ajar after restock'},
 {id:'respond',   at:858, target:'FR3', effect:{type:'door', state:'closed'}, actor:'RPH2', log:'Door closed'},
 {id:'compressor',at:600, target:'FR2', effect:{type:'drift', rate:.0105, cap:2.7}},
 {id:'arrival',   when:s=>s.queue>3, effect:{type:'spawn', kind:'truck'}}]   // condition-based
```
- Kinds: **scheduled** (`at`), **conditional** (`when(state)`), **manual** (scenario buttons in the control bar, e.g. "Cause excursion", "Restore"), **external** (`SCENE.trigger(id|spec)` / `postMessage({cmd:'trigger',…})`).
- Effects are deterministic: a fired trigger inserts an event into the timeline at its time and the model is rebuilt, so scrubbing stays consistent. Optional actor response (walk to target, perform action) is generated from the trigger.
- Every trigger writes to the event log and emits `scene:trigger`; state changes emit `scene:status` (prev → next) so an external agent can react (e.g. place a call when something crosses a limit).

## Labels & status in-scene
- Use labels only when they aid the demo; otherwise none.
- Status tag = dark pill (brand-neutral #1F1E1D) with status dot, small ID, divider, right-aligned value; danger state turns the pill brand/danger color with a pulsing dot. Size pills to their longest text (e.g. "−20.4°C").
- Anchor each tag with a thin leader line + dot to the object. **Stagger leader heights** so neighbouring tags are ≥ 2× pill height apart vertically (iso rows cascade diagonally—raise the far-back ones more). Reserve headroom in `oy` for the tallest tag.
- Reinforce status redundantly: light bar, on-device readout (skewed text), tinted glass/surface, floor halo (pulse for warn/danger), and the panel card chip.

## Panels
- **Status cards**: name, status chip, big value, range, min/max, minutes out of range, contents/notes, sparkline with safe band, near-limit bands, dashed limits, now line.
- **Gantt (reui.io-like)**: rows per resource (include empty ones), 30-min ticks, scrub header, now chip; bars scheduled/active(progress)/complete; drag move (5-min snap, vertical = change resource), edge resize, conflict validation (invalid → red, not applied); expandable child rows (e.g. staff shifts) with draggable bars.
- **Event log**: last ~6 events with time and status dot, derived from precomputed series + triggers.
- **Hover tooltip** for every object/actor/bar: title, status badge, key rows, then every `meta` key (extensible data).

## Integration API
```js
window.SCENE={ setTime,getTime,play,pause,setRate, select(id), state(),
  setData(items), upsert(item), remove(id), updateItem(id,patch),
  setReading(id,value|null), trigger(idOrSpec), configure(cfg), timeline() }
```
- Mirror as `postMessage({cmd,…})`. Emit `scene:tick` (~4 Hz), `scene:status`, `scene:trigger`, `scene:select`, `scene:change` as CustomEvents, and to `parent` **only when embedded** (`parent!==window`).

## Deliverables
- Default: standalone `.html` in the outputs folder (full document, inline CSS/JS, `:root` vars + dark media query), presented with `present_files`. Inline widget or React (`useEffect` + refs + rAF cleanup) on request.
- Escape all injected strings. Deep-clone seed data; never JSON-clone objects holding DOM refs.

## Verify
- Extract the `<script>`; `node --check`.
- Run against a tiny DOM stub (element class with attrs/children/style/classList/dataset, regex innerHTML parser, querySelector for `tag`, `.c`, `#id`, `[data-x="v"]`; stub `requestAnimationFrame`, `performance`, `CustomEvent`, `addEventListener`). Pre-create the ids the page uses. Step ~2000–4000 frames, `setTime` to storyline checkpoints and assert expected statuses, call every API method, scan all attributes/text for `NaN|undefined|Infinity`.
- Tune signal parameters until noise never flickers across thresholds unintentionally.

## Reference scenes already proven with this recipe
- Distribution center: dock wall at back with trucks behind, shutters open only while docked, shelving rows interleaved with actor layers, L-conveyor, forklifts/dock staff tied to docked trucks and shifts, AMRs dwelling at conveyor, door Gantt + shift sub-rows, unstaffed-door KPI.
- Pharmacy cold chain: CVS-style counter (drop off / pick up / consultation), will-call bins, fill + verification stations, stock shelving, drive-thru window, 3 fridges + freezer on the back wall (CDC 2–8 °C / −50 to −15 °C), two pharmacists with itineraries, door-ajar excursion and compressor-drift triggers, staggered status tags, logger cards and event log, `setReading` for live sensors.

