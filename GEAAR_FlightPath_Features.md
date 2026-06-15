# GEAAR Flight Path Authoring - Feature List (User Perspective)

Reference mock-up: `geaar-flightpath-mockup.html`
Purpose: an interactive reference for the camera flight path authoring feature in GEAAR. Every behaviour below is demonstrated in the mock-up and is intended to be rebuilt in Unity.

Conventions used in this document:
- Map view = the locked top-down survey view (the main window).
- Preview window = the bottom-left "Camera preview" panel (also called the 3D view).
- Layer panel = the "Flight paths" panel, top-right.
- Waypoint = a drone camera position (sits at a height above ground). Target = the ground point that waypoint's camera looks at.

---

## A. Getting started

1. **Instructions modal**
   - Location: centre of screen on first load and again after every Reset.
   - Interaction: a numbered six-step walkthrough plus a control cheat-sheet line; one "Start authoring" button dismisses it.
   - Notes: desktop wording only.

2. **On-screen prompt pill**
   - Location: top centre, just under the title bar.
   - Interaction: read-only. It always tells the user the next action ("Click to place the starting drone position", "Aim the camera, then click to set its target", etc.) and changes accent colour by mode (blue = place, red = aim target, green = finished).

3. **Cursor readout**
   - Location: top left.
   - Interaction: read-only live values - cursor Easting/Northing, ground elevation under cursor, default AGL (120 m), resulting drone Y, waypoint count, total path length, and total flight time.

---

## B. Creating a flight path

4. **Place the starting drone position**
   - Location: anywhere on the map.
   - Interaction: left-click (desktop) or tap (tablet). The waypoint is placed 120 m above the ground at that exact point (the height is sampled from the terrain).

5. **Aim and set the camera target (darkened aiming mode)**
   - Location: full screen darkens; the cursor controls the look-at target.
   - Interaction: move the mouse/finger to aim; the cone between the drone and the cursor lights up (this is what the camera sees). Left-click/tap to commit the target.
   - Feedback: the preview window shows the live perspective from that camera as you aim.

6. **Add further waypoints**
   - Location: map.
   - Interaction: keep clicking. Each click alternates drone position then target. The new camera inherits the previous target until you re-aim.
   - Feedback: a smooth curved flight path appears between waypoints with animated blue arrows scrolling in the direction of travel. All committed viewing cones stay lit. The combined "swept" area that the camera sweeps as it travels is lit at about half brightness, dimmer than the keyframe cones.

7. **Finish the path**
   - Location: "Finish path" button (bottom right), Esc key, or right-click.
   - Interaction: raises a modal - "Finish this path? Your flight path saves automatically. Reopen it any time to add waypoints or make changes." with [Keep editing] / [Finish].
   - Notes: everything autosaves; finishing just stops the active edit and returns the path to the list.

---

## C. The flight path layer system

8. **Flight path list**
   - Location: top-right "Flight paths" panel.
   - Interaction: each path has a colour swatch, an editable name (click to rename), a waypoint count, an expand/collapse caret, and a delete (trash) button. Click a path row to make it the active path.
   - Feedback: the active path's cones light up and it becomes editable and flyable; other paths draw as faint coloured lines on the map.

9. **Add a new flight path**
   - Location: the "+" button at the top of the layer panel.
   - Interaction: click to create a new named path and immediately start placing its waypoints.

10. **Delete a flight path**
    - Location: trash icon on each path row.
    - Interaction: click; raises a confirm modal ("Delete flight path?").

11. **Per-waypoint rows**
    - Location: under each expanded path.
    - Interaction: each waypoint shows its index, drone and target coordinates, a speed control, a height stepper, and a delete (x). 

12. **Reorder waypoints by dragging**
    - Location: the waypoint rows of the active path.
    - Interaction: drag a row by its grip handle to a new position; a drop indicator shows where it will land; the flight path re-solves on release.

13. **Add waypoints to an existing path**
    - Location: "Add waypoints" button (bottom right).
    - Interaction: re-enters placement mode appending new waypoints to the end of the active path.

---

## D. Editing nodes (Move nodes mode)

14. **Enter edit mode**
    - Location: "Move nodes" button (bottom right) toggles it on; or hold the Ctrl key for a transient version.
    - Feedback: gizmos appear on every waypoint in the top-down map AND the preview window switches to an interactive 3D editing view.

15. **Move a waypoint or target on the ground (top-down)**
    - Location: gizmo on each waypoint/target in the map view.
    - Interaction: drag the red arrow to move East/West only, the green arrow to move North/South only, or the centre square to move freely. Targets have the same arrows.
    - Notes: moving on XY always re-samples the terrain so the waypoint stays planted on the ground (drone re-drapes to terrain + its height; target re-drapes to terrain).

16. **Change waypoint height (top-down stepper)**
    - Location: the "- 120 m +" stepper attached to each waypoint gizmo.
    - Interaction: click minus/plus to lower/raise the waypoint in 20 m steps (range 20 m to 300 m). The stack of 20 m cubes under the waypoint adds/removes a cube each step.

17. **Delete a waypoint (top-down)**
    - Location: trash icon on each waypoint gizmo.
    - Interaction: click; raises a confirm modal ("Remove this waypoint? Any others in the flight path will remain.").

18. **3D transform gizmo (preview window)**
    - Location: the preview window while Move nodes / Ctrl is active.
    - Interaction: each waypoint shows red (East), green (North) and blue (Up) transform arrows. Roll over an arrow and it highlights with a pointer cursor. Drag an arrow to transform along that axis - the blue Up arrow changes height and snaps to 20 m steps; red/green move across the ground and re-drape. Drag empty space to tumble (orbit) the camera.
    - Notes: the same speed and height controls are repeated at the bottom of this window for the waypoint you last grabbed. Works in both the small and maximised window.

19. **Height cubes**
    - Location: under every waypoint in the 3D view.
    - Interaction: read-only. A translucent stack of 20 m cubes from the ground up to the drone visualises the AGL (default 120 m = six cubes).

---

## E. Speed

20. **Per-waypoint speed (five levels)**
    - Concept: each leg has a speed shown as up to five fast-forward chevrons - one chevron is slowest, two is normal (default), five is fastest. Speed belongs to the leg arriving at that waypoint, so the first waypoint has no speed leg.
    - Locations and interactions, all three cycle 1 -> 2 -> 3 -> 4 -> 5 -> 1 on click:
      - a) On the top-down gizmo (Ctrl/Move on): click the chevrons next to the waypoint.
      - b) In the layer panel: click the chevrons on the waypoint row.
      - c) Live during preview: the "SPEED -> WP n" chip at the bottom of the preview window updates to the waypoint being flown toward; click to change it live.

21. **Eased motion**
    - Concept: playback is not constant speed. Each leg cruises at its speed, transitions between legs are smoothed, and there is a large ease-in at launch and ease-out at the finish, so it reads like a real drone shot.

---

## F. Total time

22. **Total flight time readout**
    - Location: "Time m:ss" line in the cursor readout (top left).
    - Interaction: read-only; updates as waypoints, speeds, distances and the global stretch change.

23. **Adjust total time (modal)**
    - Location: "Total time" button (bottom right).
    - Interaction: opens a modal with a vertical slider - drag up for a longer/slower flight, down for shorter/faster. The new total time is shown live in minutes:seconds. Confirm applies, Cancel discards.
    - Notes: this stretches every leg evenly, so the relative per-waypoint speeds you set are preserved. Base time still comes from distance at a comfortable speed, so more waypoints means more total time.

---

## G. Preview and navigation

24. **Preview flight**
    - Location: "Preview flight" button (bottom right).
    - Interaction: flies the camera along the path at the eased, speed-aware pace in the preview window; toggles to "Stop preview".

25. **Maximise preview**
    - Location: the maximise button in the preview window title bar.
    - Interaction: expands the preview window to about 70% of the screen; the layer panel and controls stay on top. Toggle to restore.

26. **Map navigation**
    - Desktop: left-click place, middle-mouse drag to pan, scroll to zoom, I to focus on the cursor, Ctrl to show gizmos, R-click or Esc to end.
    - Tablet: one-finger drag to aim, tap to place, two-finger pan, pinch zoom, on-screen buttons for the rest.

27. **Reset all**
    - Location: "Reset all" button (bottom right).
    - Interaction: clears all paths, starts a fresh empty path, and reopens the instructions.

---

## H. World context (for scale and realism)

28. **Terrain** - a hilly displaced heightfield roughly -190 m to +480 m, arranged as a 4 x 4 grid of 2400 m tiles (9600 m square), with a 100 m survey grid and brighter tile-seam lines.
29. **Landmarks** - one to two tall unique structures per tile (lattice mast, water tower, chimney, obelisk, wind turbine, domed observatory, pyramid) plus scattered buildings, comms masts and trees, all seated on the terrain.
30. **Sky and depth** - a gradient sky dome and distance fog for orientation in the perspective views.

---

## Button state summary (bottom-right stack)

All buttons are uniform width/height and grey out when not usable:
- Preview flight - needs 2+ waypoints.
- Total time - needs 2+ waypoints.
- Move nodes - needs 1+ waypoint.
- Add waypoints - needs an active path.
- Recentre - always available.
- Finish path - available while creating or editing.
- Reset all - always available.
