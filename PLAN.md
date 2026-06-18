# **GEAAR Flight Path \- Pass 5 Implementation Plan**

Purpose: turn the Pass 5 build brief into an ordered, independently testable plan, grounded in the current working mock-up. The brief governs this pass: where the build and the brief diverge, the brief wins, and the conflicting build behaviours are dropped (recorded in Section 7).

---

## **0\. Scope and authority**

- Source of truth this pass: the Pass 5 build brief. The working HTML mock-up is the behavioural reference, but the brief takes precedence over it. The Unity engineering handover is frozen.  
- The architect sets direction; product owner feedback is input; the architect's call is final.  
- Deliverable: an updated single-file `index.html` that reflects every Pass 5 decision.

---

## **1\. Build baseline (read first)**

The working build is the uploaded `index.html` (the deployed `ephemeral-pudding-c7dcab` drop, 15 June 2026). It still carries the `v0.2` label in the brand strip but is well ahead of the repo snapshot. It is selection-driven, has a bottom timeline, a speed colour ramp, multi-select and the new control scheme. It does not yet have the onboarding wizard or a path-level camera type, and it took a different route on look-at handling that the brief now overrules.

So Pass 5 is mostly a focused gap-fill plus a few reversals. The two big remaining pieces are the onboarding wizard (A2) and the Fixed or Targeted camera type (A1). The rest are small changes, verifications, or removals.

---

## **2\. Already implemented (verify and keep)**

These brief items are present in the build. Treat them as verify-and-align, not build. Code references are to the uploaded file.

| Brief | Status in build | Note |
| :---- | :---- | :---- |
| B1 selection drives preview and editor | Present | `selection` set, `selectNode`, `clearSelection`; click selects in map, layer list and timeline dots; prompt and editor follow selection. |
| D1 control scheme | Present | Click add or select; right-press-drag aims look-at (`startLookatDrag`); Esc deselects; middle-drag pans; scroll zooms; I focuses. |
| D2 multi-select | Present | Shift-click toggles, shift-drag box-select. The build also adds cascade editing, which the brief defers; cascade is removed this pass (Step 10). |
| C1 speed colour ramp | Present and matches | `SPEED_COLOURS` \= purple, blue, green, yellow, orange for levels 1 to 5, applied to the path line per segment, the chevrons, and the timeline. |
| C2 timeline (base) | Present | `#timeline` with play, speed-coloured segments, selectable dots, draggable playhead scrub, total time at right; clicking the total opens the retime modal. |
| C2 retime (even stretch) | Present | `openTimeModal` sets `timeScale = target / base`, preserving relative speeds. |
| E1 simplify layer panel | Present (verify) | Rows carry speed, height, look-at toggle, grip, trash and click-to-select. Confirm no coordinate read-outs remain. |
| Terminology in copy | Present (verify) | Prompts and instructions say "waypoint". Internal identifiers (`selectedNode`, `nodes`) may still say node; cosmetic only. |

---

## **3\. Terminology and data model deltas still required**

- Terminology: copy already uses "waypoint". The button renames (Edit Waypoints, Exit Editing) come with the button-stack restoration in Step 9\.  
- Speed x4 (C4): `SPEED_VELS` is still `[0, 12, 24, 40, 58, 80]`. Scale to `[0, 48, 96, 160, 232, 320]`, keeping the five levels proportional, level 2 normal. Check `PREVIEW_SPD` for consistency.  
- Target elevation (B3): `target.y` is stored but the look-at is placed and dragged on the ground. Make it raiseable. See Step 5\.

---

## **4\. Remaining work, ordered**

Foundations and quick wins first, the headline wizard last because it teaches functions that must already work. Each step is independently testable.

### **Step 1\. Speed x4 (C4)**

- Now: `SPEED_VELS = [0, 12, 24, 40, 58, 80]`.  
- Change: scale by four to `[0, 48, 96, 160, 232, 320]`; check `PREVIEW_SPD`.  
- Done when: default cruise reads roughly four times faster with levels still proportional and level 2 normal.

### **Step 2\. Total Time helper text and timeline grab-to-scale (C2 remainder)**

- Now: `openTimeModal` retimes via the modal; the timeline total opens that modal; there is no grab-to-scale on the timeline itself.  
- Change: add the helper text to the Total Time dialogue: "We recommend 90 to 120 seconds for long shots, under 20 seconds for something short." Add direct scaling on the timeline by grabbing the end marker and dragging left to compress or right to stretch, numbers adjusting live, with a release confirm popup, for example "Scale timeline from 3 min 14 sec to 1 min 20 sec?". Apply the same even stretch as Total Time, preserving relative speeds.  
- Done when: the end marker scales the whole flight with a confirm popup, relative speeds hold, and the helper text shows in the dialogue.

### **Step 3\. Faster-moving arrows (C1 remainder)**

- Now: `updateArrows` colours arrows by speed and sets chevron count to the speed level, but advances `arrowPhase` at a uniform rate, so arrows do not move faster in faster sections.  
- Change: tie the per-segment scroll rate to the segment speed so motion speed reads alongside colour and chevron count.  
- Done when: arrows visibly scroll faster on higher-speed legs.

### **Step 4\. Preview viewport full labels (C3)**

- Now: the preview chips use short labels.  
- Change: read them in full with the waypoint name: "Speed to Next Waypoint (name)" and "Height of Next Waypoint (name)". They act on the upcoming waypoint while previewing; per-waypoint editing remains.  
- Done when: both labels show the upcoming waypoint name and edit it live.

### **Step 5\. Targets raiseable in 2D and 3D (B3)**

- Now: the fixed look-at target is dragged on the ground; `target.y` is not user-editable.  
- Change: let a target be raised and moved in both the top-down and the 3D preview views, like a waypoint, with `target.y` editable. Keep a sensible default seating, but allow lift.  
- Done when: a target can be lifted off the ground in both views and the preview aim follows the raised target.

### **Step 6\. Clean preview by default and reveal toggle (B5)**

- Now: the darkened overlay is visible whenever a path has waypoints, and the preview shows the authoring scene.  
- Change: in the preview camera, hide gizmos and the flight path and overlays by default so the preview reads like the real camera output. Holding Ctrl or pressing a reveal toggle shows the authoring overlays in the preview.  
- Done when: the default preview is clean and Ctrl or the toggle restores the authoring overlays.

### **Step 7\. Drone icon (B4)**

- Now: `makeDroneMarker` draws a sphere, stem and ground disc.  
- Change: make the drone icon clearly the camera position (a drone sitting in the sky), distinct from a generic path dot; keep the target icon as the look-at and colour-coordinate the link line.  
- Done when: drone, target and link line are visually distinct at a glance.

### **Step 8\. Camera type Fixed or Targeted at creation (A1, A3) \- brief governs**

- Now: there is no path-level camera type. Look-at is per-waypoint: each waypoint is either "forward along path" (`n.target` null) or a draggable fixed target. This per-waypoint model is dropped (Section 7).  
- Change: add the creation-time choice as a property of the path. Targeted is the place then look-at flow; the look-at starts ahead of the camera along the direction of travel with a comfortable arc, and a suggested ring around the camera shows the ideal horizon position to drop the target (A3). Fixed travels the line with a fixed downward framing, nadir by default, and no per-waypoint target. For both types, Ctrl brings up the full per-waypoint gizmo during placement. Mixing types within one path is out of scope.  
- Done when: a Fixed path previews with nadir framing and never needs a target; a Targeted path proposes a forward look-at with the suggestion ring; the type is chosen at creation and fixed for the path.

### **Step 9\. Button stack and terminology (brief's seven-button stack) \- brief governs**

- Now: four buttons (Preview flight, Total time, Recentre, Reset all).  
- Change: restore the brief's final stack with renames: Preview flight, Total time, Edit Waypoints (Ctrl), Add waypoints, Recentre (I), Exit Editing (Esc), Reset all, plus a wizard-hat replay button. Selection coexists as an additional way in. Exit Editing keeps the autosave and reopen reassurance. Reconcile Esc: deselect when a selection exists, otherwise Exit Editing.  
- Done when: the stack matches the brief, the renames are applied, and the wizard-hat button is present.

### **Step 10\. Remove cascade editing (D2) \- brief governs**

- Now: editing one waypoint cascades to neighbouring waypoints that share its value, with shift-select as the escape hatch (`selMode`, cascade-run logic).  
- Change: remove the cascade. A single-waypoint edit affects only that waypoint. Multi-select (shift-click, shift-drag) remains the only way to edit several at once. The brief defers cascading.  
- Done when: editing one selected waypoint changes only it, and a multi-selection edit applies to exactly the selected set.

### **Step 11\. Onboarding wizard and wizard-hat replay (A2) \- headline**

- Now: a static seven-step instructions modal (`showInstructions`).  
- Change: replace it with a guided wizard that teaches by doing, in natural language, with animated wireframe graphics, advancing one action per click or button press until three waypoints are placed and all functions are covered. Use the brief's twelve-step Targeted script verbatim; do not paraphrase the voice. Provide a Fixed variant in the same voice minus the look-at steps. The wizard-hat button replays it any time.  
- Done when: a first run walks a new user to three placed waypoints touching every function, the Fixed variant omits look-at steps, and the hat replays it.

### **Step 12\. Regression pass (E2)**

- Verify the top-down and the 3D preview gizmos work, now including target raise and move from Step 5\.  
- Confirm the earlier raycaster and layer changes, and the overlap-hide of top-down gizmos over the preview frame, did not regress 2D dragging.  
- Confirm the placement click versus drag ambiguity stays resolved by the D1 scheme.  
- Done when: all three checks pass on a clean build.

---

## **5\. Speed colour ramp reference**

The build already matches the brief ramp. Recorded here for completeness; keep it identical across the path line, timeline and chevrons.

| Level | Colour | Hex in build |
| :---- | :---- | :---- |
| 1 | purple | `0xc77dff` |
| 2 | blue | `0x3da9ff` |
| 3 | green | `0x3ddc84` |
| 4 | yellow | `0xffd23d` |
| 5 | orange | `0xff8a3d` |

---

## **6\. Resolution: the brief governs**

The brief is authoritative this pass. The three earlier divergences resolve in the brief's favour:

1. Camera type over per-waypoint look-at. Build the path-level Fixed or Targeted choice at creation (Step 8). Fixed frames nadir with no targets; Targeted uses the place then look-at flow with the forward-arc default and suggestion ring. The per-waypoint look-at toggle is removed.  
2. Seven-button stack over four. Restore Edit Waypoints (Ctrl), Add waypoints and Exit Editing (Esc) with the renames, selection coexisting (Step 9).  
3. No cascade editing. Remove the cascade behaviour; multi-select is the only bulk-edit path (Step 10).

---

## **7\. Left out (dropped from the build)**

Because the brief governs, these build behaviours are not carried forward:

1. Per-waypoint look-at toggle. The build let each waypoint be "forward along path" or a fixed point, toggled by the look-at pill or right-click. Under the brief, look-at is determined by the path's camera type (Targeted aims at a target, Fixed holds nadir), not toggled per waypoint. Right-click to aim a Targeted waypoint's look-at (D1) is kept.  
2. The "forward along path" framing. The brief's two types are Targeted (aims at a target) and Fixed (nadir downward). A camera that looks forward along the path tangent is neither, so that framing is dropped.  
3. Cascade editing. The build cascaded an edit to neighbouring waypoints sharing a value, with shift-select as the escape hatch. The brief defers cascading, so bulk edits come only from multi-select.  
4. The four-button selection-only stack. The build's minimal action stack is replaced by the brief's seven-button stack with renames, with selection coexisting as an additional way in rather than the only way.

---

## **8\. Conventions**

- Australian spelling, dates and units throughout in-app copy and docs. No em or en dashes in generated content. Note that the current build's instructions copy contains an em dash and an arrow glyph; clean these when that copy is touched.  
- The wizard script is reproduced verbatim from the brief for the Targeted walk, with a Fixed variant in the same voice minus look-at steps.

