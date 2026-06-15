# GEAAR Flight Path Authoring - Unity Engineering Handover

Source of truth: `geaar-flightpath-mockup.html` (three.js r128 reference mock-up).
Audience: a Unity engineer rebuilding this feature, working with Claude Code.

This document captures the data model, coordinate and unit conventions, every feature's logic, the exact algorithms and constants, and a suggested build order. Treat the HTML as the behavioural reference; treat this document as the spec. Work through Section 9 (the checklist) one item at a time.

A note on philosophy: the mock-up is a top-down authoring tool for drone-style camera flight paths. The user places waypoints (drone positions at a height above ground) and targets (the ground point each camera looks at), and the tool solves a smooth, speed-aware, eased flight path and visualises what the camera sees.

---

## 1. Coordinate system and units

The mock-up is three.js (right-handed, Y up). Map this into Unity's left-handed, Y-up space once, centrally, and keep all gameplay code in Unity space.

| Quantity | Mock-up (three.js) | Meaning | Unity mapping |
|---|---|---|---|
| East/West | +X | Easting | +X |
| Up | +Y | Height / elevation | +Y |
| North/South | -Z is North | Northing (readout shows N = -z) | +Z is North, so `unity.z = -three.z` |

- All distances are in **metres**, 1 unit = 1 m.
- Angles in degrees in this doc; convert as needed.
- The cursor readout prints Northing as the negated Z. Keep one conversion helper and do not scatter sign flips.

---

## 2. Core data model

```
FlightPath
  id        : string/guid
  name      : string            // user-editable
  colour    : colour            // swatch + spline colour
  collapsed : bool              // layer-panel UI state
  nodes     : Node[]

Node
  dronePos  : Vector3           // camera position; y = terrainHeight(x,z) + agl
  target    : Vector3           // look-at point on ground; y = terrainHeight(x,z)
  speed     : int 1..5          // speed of the leg ARRIVING at this node (node 0 unused)
  agl       : float metres      // height above ground for this waypoint
```

Global state:
- `paths : FlightPath[]` with one **active** path.
- `timeScale : float` - global total-time stretch multiplier (default 1).
- Authoring state machine: `PLACE_FIRST_DRONE -> PLACE_TARGET -> PLACE_DRONE -> PLACE_TARGET -> ... -> IDLE`.

Invariants (enforce on every mutation):
- `dronePos.y == terrainHeight(dronePos.x, dronePos.z) + agl`
- `target.y == terrainHeight(target.x, target.z)`
- `speed` clamped 1..5; `agl` clamped and quantised (see constants).

---

## 3. Constants (lift these verbatim)

```
WORLD        = 9600      // world is WORLD x WORLD metres
TILE         = 2400      // 4 x 4 grid of tiles
SEG          = 192       // terrain mesh resolution per side

DEFAULT_AGL  = 120       // m, default waypoint height
HEIGHT_STEP  = 20        // m, height stepper increment / cube size
HEIGHT_MIN   = 20        // m
HEIGHT_MAX   = 300       // m

CONE_HALF    = 24        // deg, camera viewing-cone half angle
SWEPT_LIGHT  = 0.5       // brightness of swept area vs 1.0 keyframe cones
DARKNESS     = 0.62      // overlay darkening strength

SPEED_VELS   = [0, 12, 24, 40, 58, 80]   // m/s by speed level 1..5 (index 0 unused); level 2 = normal
EASE_FRAC    = 0.22      // fraction of total flight spent ramping at each end
EASE_FLOOR   = 0.12      // minimum speed multiplier at the very start/end

// camera (map view)
camHeight default 1500   // m, nadir camera height
zoom range  250 .. 5200  // m

// spline sampling
SPLINE_SAMPLES = 400
// motion profile
MOTION_SAMPLES = 600

// lightmap render target
LIGHTMAP_RT = 2048       // px, orthographic top-down
```

Palette: a small Crayola-style set; each path is assigned the next colour. Brand red is `#ff0037` (used for the aiming/target accent).

---

## 4. World and terrain

### 4.1 Terrain heightfield (single source of truth)
`terrainHeight(x, z)` is deterministic and is used for three things: displacing the terrain mesh, seating every prop/landmark, and re-draping waypoints/targets. Do not duplicate the formula; expose one function.

Composition:
- A sum of low-frequency sines producing big rolling hills (amplitudes roughly 150, 70, 28, 40 m on different wavelengths/axes), plus
- Six fixed **Gaussian peaks** (`PEAKS`), each a centre, height and radius, added on top so some summits exceed waypoint height.
- Resulting range approximately **-194 m to +483 m**.

Unity: bake this into a heightmap or a `Terrain`, but keep `terrainHeight(x,z)` callable at runtime for re-draping and seating. A C# port of the same sine+gaussian sum is the cleanest way to guarantee the editor maths matches the mesh.

### 4.2 Tiles, grid, props
- 4 x 4 tiles of 2400 m. Draw a 100 m survey grid; brighten the tile-seam lines.
- Props seated on terrain: clustered low buildings, scattered structures, comms masts, trees.
- **Landmarks**: 1 to 2 tall unique structures per tile, chosen from a set (lattice mast, water tower, chimney, obelisk, wind turbine, domed observatory, pyramid). Deterministic per tile for repeatability.
- Gradient sky dome + distance fog for the perspective views.

---

## 5. Cameras and views

- **Map camera**: perspective camera looking straight down (nadir), behaving like an ortho survey view. Pan (middle-drag / two-finger), zoom (scroll / pinch, clamp 250..5200), focus-on-cursor (`I`). Height default 1500 m.
- **Preview camera (PiP)**: the drone camera. During preview it flies the path; during authoring it shows the live aim; during editing it becomes an orbit/tumble edit camera that frames the active path.
- **Edit camera**: same window as the preview; orbits the path while Move-nodes/Ctrl is active. Drag empty space to tumble.
- **Layers**: in the mock-up, 3D gizmo arrows are on render layer 2 so only the edit camera draws them and the raycaster must opt in to that layer. In Unity this is just a layer mask on the edit camera and the gizmo collider raycast.

---

## 6. Feature logic (rebuild target by target)

### 6.1 Placement
- Raycast the pointer against the terrain to get the ground point `g`.
- Drone placement: `dronePos = (g.x, terrainHeight + agl, g.z)`, `agl = DEFAULT_AGL`.
- Target placement (aiming mode): screen darkens; `target = (g.x, terrainHeight, g.z)`; the cone from the current drone to the live cursor lights up; commit on click.
- State machine alternates drone/target; a new drone inherits the previous target until re-aimed.

### 6.2 Flight path spline (drone positions)
- `CatmullRom(dronePositions, closed=false, type=centripetal)`.
- Node `i` sits at curve parameter `i/(N-1)`. This matters for syncing targets, speed and motion to nodes.
- Sample `SPLINE_SAMPLES` points for the line and the scrolling direction arrows.
- **Centripetal** was chosen deliberately: it passes through every waypoint with no overshoot. (Open decision: an optional per-path "smoothing" that switches to a corner-cutting/B-spline for broader arcs that round past interior waypoints. Not implemented; leave a hook.)
- Direction arrows: instanced chevrons along the sampled curve, scrolling by a time offset in the travel direction.

### 6.3 Target path
- The look-at is a **per-segment linear interpolation** between consecutive node targets, parameterised the same way as the spline so drone-sample i and target-sample i correspond.

### 6.4 Viewing cones + darkening (the "what the camera sees" overlay)
This is a lightmap technique. Reproduce the look; the implementation can differ in Unity.

- Offscreen **orthographic top-down render target** (`LIGHTMAP_RT`) covering the world.
- Each committed camera writes a **cone footprint** into the RT: a triangle-fan sector with half-angle `CONE_HALF`, apex at the drone's ground projection, oriented toward the target, length scaled by drone-to-target distance. Keyframe cones write brightness **1.0**.
- Blend mode is **MAX** so overlapping cones never over-brighten.
- The **swept area** (see 6.5) is stamped at **SWEPT_LIGHT (0.5)**.
- An overlay shader darkens the scene by sampling the RT by world XZ: `alpha = DARKNESS * (1 - lightmapSample)`. Lit cones are clear; everything else is dimmed.
- Unity options: render cone meshes to a RenderTexture with a max blend, then sample it in a full-screen or terrain-draped decal/overlay shader. Or use projector/decal cones. Keep the MAX-blend and the 1.0 vs 0.5 distinction.

### 6.5 Swept area
- Walk the drone spline and the target lerp **in lockstep** across the whole path.
- At each step stamp a dim cone (brightness 0.5) for that instantaneous camera. The union is the area the camera sweeps as it flies.

### 6.6 Height cubes
- Under each waypoint, stack translucent `HEIGHT_STEP` (20 m) cubes from the ground up to the drone: `count = round(agl / HEIGHT_STEP)`. Purely a height read-out (default 120 m = 6 cubes).

### 6.7 Speed model
- `speed` is an int 1..5 on the leg **arriving** at a node; node 0 has none.
- Cruise velocity for the leg into node `k` is `SPEED_VELS[node[k].speed]` m/s. Level 2 is "normal".
- Editable in three places, all cycling 1->2->3->4->5->1: top-down gizmo chip, layer-panel row chip, and the live "SPEED -> WP n" chip during preview.

### 6.8 Motion profile and easing (`buildMotion`)
Produces a time-parameterised playback that is not constant speed.

1. Sample the spline at `MOTION_SAMPLES` points; compute arc-length deltas `ds` between samples.
2. Per sample, cruise speed = `SPEED_VELS[ node[seg+1].speed ]` for the segment that sample falls in.
3. Smooth the per-sample speed with a small moving average so leg-to-leg speed changes are not instant.
4. Apply an **ease envelope** across the whole flight: within the first and last `EASE_FRAC` (0.22) of normalised progress, ramp the speed multiplier from `EASE_FLOOR` (0.12) up to 1 via smoothstep, so it eases in at launch and out at the finish.
5. Integrate `dt = ds / v` and accumulate to a cumulative time array. Total time = last value.
6. `motionSample(t)` maps a wall-clock time to a curve parameter (binary search / lerp into the cumulative array).
7. Multiply total time and the time axis by `timeScale`.

### 6.9 Total time control
- Base total time comes from 6.8 at `timeScale = 1` (so more/longer legs = more time).
- The Total-time modal sets `timeScale = targetTime / baseTime`, stretching every leg evenly and preserving the relative per-waypoint speeds. Vertical slider: up = longer, down = shorter; show live m:ss.

### 6.10 Gizmos and editing
Two parallel gizmo systems share the same node data:

- **Top-down (screen-space/DOM in the mock-up; in Unity, world-space handles seen by the map camera):** per node, red arrow = East-only, green arrow = North-only, centre square = free move; a `- agl m +` stepper (20 m steps, clamp 20..300); a trash button (confirm modal); speed chevrons. Targets get the same move arrows.
- **3D transform gizmo (preview/edit window, layer 2):** red (East), green (North), blue (Up) arrows on each waypoint, with rollover highlight + pointer cursor. Blue drag changes height and snaps to 20 m; red/green drag moves across ground. Drag empty space = orbit/tumble. The window's bottom bar repeats the speed + height controls for the last-grabbed node.

Editing rules (critical, these were bug sources):
- Edit mode is active when **Move-nodes toggle is on OR Ctrl is held** (not toggle only).
- **Every XY move re-drapes**: after changing x/z, set `dronePos.y = terrainHeight + agl` (and `target.y = terrainHeight`) so nodes stay planted on the ground.
- The 3D-gizmo raycaster must include the gizmo layer (layer 2) or hover/drag silently fails.
- Top-down gizmos that overlap the preview window must be hidden, so their controls do not bleed into the 3D view (the 3D view owns its node controls there).

### 6.11 Layer panel
- List of paths: swatch, editable name, waypoint count, collapse caret, delete (confirm modal), "+" to add a path. Click a row to set active.
- Active path: cones lit, editable, flyable. Inactive paths: faint coloured lines only.
- Per-waypoint rows: index, drone+target coords, speed chevrons, `- agl m +` stepper, delete (x). Drag a row by its grip to reorder; show a drop indicator; re-solve on release.

### 6.12 Preview, maximise, navigation, reset
- Preview flies the camera with `motionSample` at the eased, speed-aware pace; button toggles Stop.
- Maximise expands the preview to ~70% screen with the layer panel/controls staying above it.
- Navigation as in Section 5. `Recentre` / `I` focuses the cursor.
- Reset clears all paths, starts one empty path, reopens instructions.

### 6.13 UI furniture
- Solid title bar with brand + a phase chip.
- Prompt pill (top centre, mode-coloured, width-capped to the viewport).
- Cursor readout (top left): Easting, Northing, ground elev, AGL, drone Y, waypoint count, path length, total time.
- Modals: Finish path, Remove waypoint, Delete path, Total time, Instructions.
- Button enable/disable rules per the feature list's button-state summary.

---

## 7. Things to preserve exactly

- Node-to-curve-parameter mapping `i/(N-1)` (targets, speed and motion all rely on it).
- Centripetal Catmull-Rom (no overshoot, passes through waypoints).
- MAX-blend cones with 1.0 keyframe vs 0.5 swept brightness, and `alpha = DARKNESS*(1 - sample)`.
- The ease envelope (`EASE_FRAC`, `EASE_FLOOR`) and the smoothed per-segment cruise speeds.
- `timeScale = target/base` stretch that preserves relative speeds.
- Re-drape on every XY edit.
- The five-level speed chevron semantics (owned by the arriving leg; node 0 has none; level 2 = normal).

## 8. Open / deferred decisions

- Optional per-path smoothing that switches the spline to corner-cutting/B-spline for broader sweeping arcs (would round past interior waypoints). Hook left in 6.2.
- Edit camera currently auto-frames the whole path; could instead focus the selected waypoint and remember orbit angle.

---

## 9. Suggested implementation order (pick up one at a time)

Each step is independently testable. Do not start the next until the current one is verified.

1. **World + terrain.** Port `terrainHeight(x,z)` (sines + 6 gaussian peaks), build the 4x4 / 9600 m terrain, 100 m grid, tile seams, sky, fog. Verify range approx -194..+483 m.
2. **Props + landmarks.** Seat buildings/masts/trees and 1-2 unique landmarks per tile on the terrain deterministically.
3. **Map camera + navigation.** Nadir camera, pan, zoom clamp 250..5200, focus-on-cursor.
4. **Data model + state machine.** `FlightPath`/`Node`, active path, authoring states, invariants.
5. **Placement + raycast.** Drone at terrain+120, aiming-mode darken, target on ground, commit; cursor readout.
6. **Spline.** Centripetal Catmull-Rom, node at i/(N-1), 400 samples, scrolling direction arrows.
7. **Target path.** Per-segment lerp synced to the spline.
8. **Viewing cones + darkening overlay.** Ortho RT, cone footprints, MAX blend, overlay shader (1.0 keyframe).
9. **Swept area.** Lockstep drone+target walk stamping 0.5 cones.
10. **Height + cubes.** Per-node AGL (20 m steps, 20..300), translucent cube stack.
11. **Speed model.** 1..5 chevrons on the arriving leg, SPEED_VELS.
12. **Motion profile.** `buildMotion` with smoothed cruise + ease envelope; `motionSample(t)`.
13. **Preview flight.** Fly with motionSample; Stop toggle.
14. **Total-time modal.** `timeScale = target/base`, vertical slider, live m:ss.
15. **Layer panel.** Path list, add/rename/delete/active, per-waypoint rows, drag-reorder.
16. **Top-down gizmos.** Move arrows + free square (drone and target), height stepper, trash, speed chevrons; re-drape on XY.
17. **3D transform gizmos.** Edit camera + orbit, red/green/blue arrows, rollover highlight, blue 20 m snap, raycaster includes gizmo layer, re-drape on XY, hide top-down gizmos overlapping the preview window, bottom-bar controls bind to last-grabbed node.
18. **Preview maximise + UI furniture + modals + button states + instructions/reset.**

Acceptance for each: matches the mock-up behaviour for that feature in isolation.
