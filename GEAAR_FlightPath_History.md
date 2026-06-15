# GEAAR Flight Path Authoring - Build History

A version-by-version log of how the mock-up `geaar-flightpath-mockup.html` was built, in order. Each entry lists what that iteration added or changed. This is the narrative record behind the feature list and the Unity handover.

---

## v0.1 - First authoring mock-up
The Phase 1 user journey end to end:
- Locked top-down map view over a flat aerial plane.
- Click to place the starting drone position (default 120 m up).
- Darkened aiming mode: move to aim the camera, the cone to the cursor lights up, click to set the ground target.
- Live perspective preview window (PiP) of the camera's view.
- Catmull-Rom (centripetal) flight path spline through the waypoints, with animated blue direction arrows scrolling along it.
- Committed viewing cones lit on a dark overlay using a lightmap technique (offscreen orthographic render target, MAX blend so overlaps don't over-brighten); the swept travel area lit at half brightness.
- Preview flight along the path.
- Drone/target markers (sphere + stem + ground disc for the drone, ring + cross for the target).
- Desktop controls: left-click place, scroll zoom, right-click / Esc to end.

## Iteration 2 - Tablet and pointer controls
- Tablet support: one-finger drag to aim, tap to place, two-finger pan, pinch to zoom, on-screen buttons.
- Desktop: middle-mouse drag to pan.
- Recentre added as a button and the `I` key.
- Larger, touch-friendly button sizing.

## Iteration 3 - Real terrain, scale and a bigger world
- Displaced terrain heightfield (`terrainHeight(x,z)`) replacing the flat plane, used as the single source of truth for the mesh, prop seating and raycasting.
- Pointer raycasts onto the terrain; drone height is now terrain + AGL (so waypoints sit a true 120 m above the ground they're over).
- 3D scale scenery: clustered buildings, scattered structures, comms masts, trees.
- Gradient sky dome and distance fog.
- World expanded to a 4 x 4 grid of 2400 m tiles (9600 m square).
- `I` now recentres on the mouse cursor.
- Cursor readout gained ground Elevation and resulting Drone Y.

## Iteration 4 (v0.2) - Multi-path layers and node editing
- Flight path layer system: multiple named paths, each with a colour swatch, waypoint count, collapse caret and delete; a "+" to add a path; click a row to make it active; inactive paths draw as faint coloured lines.
- Per-waypoint rows with coordinates and a delete control.
- Drag-to-reorder waypoints with a drop indicator and path re-solve.
- Node editing gizmos via a Move-nodes toggle or holding Ctrl: red (East) / green (North) arrows + a free-move square; a minus control to delete.
- Confirm modals for ending a path, removing a waypoint and deleting a path.
- Re-open a finished path to keep adding/editing.
- Reviewed spline smoothing; kept centripetal Catmull-Rom (passes through every waypoint, no overshoot) and raised sample density.

## Iteration 5 - Button discipline, reset and legibility
- All action buttons made uniform width/height and greyed out when not usable.
- "Reset all" restored.
- Fixed legibility of the top-right layer panel.
- Instructions modal shown on first load and again after every Reset.

## Iteration 6 - Hillier terrain, landmarks, desktop prompts
- Terrain made much hillier - larger rolling amplitude plus six Gaussian peaks, some summits now exceeding waypoint height.
- 1 to 2 tall unique landmarks per tile (lattice mast, water tower, chimney, obelisk, wind turbine, domed observatory, pyramid).
- Instructions modal restricted to desktop wording; prompts reworded to mention scroll-zoom, `I` focus and Ctrl gizmos.

## Iteration 7-8 - "Finish path" language
- Reworked the end-of-path interaction wording. Settled on a "Finish this path?" modal (with "saves automatically, reopen any time" reassurance and Keep editing / Finish) and renamed the button to "Finish path".

## Iteration 9-10 - Top-left prompt overflow
- Fixed the prompt pill drawing outside its frame: capped its max-width to the viewport, made the title bar a solid strip, and dropped the pill just beneath it.

## Iteration 11 - Prompt wording and end control
- Forced the on-screen prompts to desktop wording and updated the end hint to "R-click / Esc end".

## Iteration 12 - Speed and time controls
- Per-waypoint speed in five levels shown as fast-forward chevrons (1 slow, 2 normal default, 5 fast), owned by the leg arriving at each node (node 0 has none).
- Speed editable in the gizmo chip, the layer-panel row, and a live "SPEED -> WP n" chip during preview.
- Eased, variable-speed motion: each leg cruises at its speed, leg-to-leg changes are smoothed, with a strong ease-in at launch and ease-out at the finish.
- Total-time modal with a vertical slider (up = longer/slower, down = shorter/faster), live m:ss, stretching all legs evenly while preserving relative speeds.

## Iteration 13 - Per-waypoint height
- Per-node AGL editing in 20 m steps (range 20-300 m).
- Translucent 20 m height cubes stacked from the ground to each waypoint as a height read-out.
- Height editable from a blue height gizmo, a layer-panel `- +` stepper, and a live height control during preview.
- Preview window can be maximised to ~70% of the screen.

## Iteration 14 - Split the gizmos top-down vs 3D
- Top-down: the blue height arrow became a `- 120 m +` stepper, and the minus became a trash (delete) control.
- The full blue/transform gizmo moved into the 3D preview/edit view, with an orbiting edit camera and pointer interaction inside the preview window.

## Iteration 15 - Rollover on 3D arrows
- Added rollover highlighting (and a pointer cursor) to the 3D transform arrows.

## Iteration 16 - Edit view on Ctrl or Move
- Fixed the 3D edit view so it engages whenever gizmos are on (Move-nodes toggle OR Ctrl held), not only via the toggle. Restored tumble/orbit.

## Iteration 17 - Raycast, re-drape and the duplicate height control (current)
- Fixed 3D arrow selection: the transform arrows live on a separate render layer, but the shared raycaster only tested the default layer, so hover and drag silently missed. The raycaster now also tests the gizmo layer, so rollover and drag-to-move work.
- Confirmed every XY move re-drapes to the terrain (drone to terrain + height, target to terrain) so nodes stay planted on the ground.
- Removed the duplicate height `- +` that floated at the top of the preview window: top-down gizmos that overlap the preview frame are now hidden, leaving only the single height stepper at the bottom of the preview bar.

---

### Notes carried forward (open / deferred)
- Spline is still centripetal Catmull-Rom (through every waypoint). Deferred: an optional per-path smoothing that switches to corner-cutting/B-spline for broader sweeping arcs.
- Edit camera auto-frames the whole path; deferred: focusing the selected waypoint and remembering the orbit angle.
