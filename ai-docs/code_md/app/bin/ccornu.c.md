# ccornu.c — Cornu Easement Track Commands

## Overview

`ccornu.c` implements **Cornu easement tracks** (`T_CORNU`) and the "Convert To/From" commands. A Cornu track is a smooth transition between two arbitrary endpoints (position, angle, and curvature). It uses Raph Levien's Bezier approximation library to approximate a true Cornu spiral with a sequence of arcs and straight segments.

The file handles:
- **Creating** a new Cornu track by placing two endpoints and dragging intermediate points along the curve.
- **Modifying** an existing Cornu track (dragging endpoint circles, adding mid-points, changing radius/angle at ends).
- **"Convert To"** — replaces any selectable tracks with a single Cornu that smoothly joins them.
- **"Convert From"** — breaks a Cornu back into individual track segments.

---

## Key Data Structures

### `endHandle` — Handles for end-point control circles

```c
typedef struct {
    coOrd end_center;     // Center of the outer control circle (radius = 0.25 * scale)
    coOrd end_curve;      // A point on the curve at the end handle
    DIST_T mid_disp;      // Offset from center to the curve-point along the tangent direction
    BOOL_T end_valid;     // TRUE if this endpoint is currently valid (not extended off-track)
    BOOL_T angle_selected;// If TRUE, dragging changes the end angle instead of radius
    BOOL_T radius_selected;// If TRUE, dragging changes the end radius instead of angle
    BOOL_T last_selected; // Which handle was last selected (for highlighting)
    ANGLE_T arc_angle;    // Total turn angle at this end (used for drawing)
} endHandle;
```

### `Da` — Global Command State (static)

All command-state variables are stored in the static struct `Da`:

| Member | Type | Description |
|--------|------|-------------|
| `state` | enum Cornu_States | Current sub-state: NONE, POS_1, LOC_2, PICK_POINT, POINT_PICKED, TRACK_SELECTED |
| `pos[4]` | coOrd[] | The two track endpoints and any mid-point handles |
| `number_of_points` | int | Number of mid-points added between the ends |
| `selectEndPoint` / `selectMidPoint` / `selectEndHandle` | int | Which handle is currently under mouse (−1 = none) |
| `prevSelected` | int | Previous selected handle (used for undo/backspace) |
| `trk[2]` | track_p[] | The two tracks connected to the Cornu ends (NULL if not yet connected) |
| `ep[2]` | EPINX_T[] | Index of the endpoint on each track (−1 = open end) |
| `radius[2]`, `angle[2]`, `center[2]` | DIST_T, ANGLE_T, coOrd[] | The three parameters defining each end's geometry |
| `arcA0[2]`, `arcA1[2]` | ANGLE_T[] | Arc start/end angles for this track end (when radius ≠ 0) |
| `extend[2]`, `extendSeg[2]` | BOOL_T, trkSeg_t[] | Whether an extra straight/curved segment is attached to extend the Cornu outward |
| `mid_points` / `crvSegs_da` | dynArr_t[] | Arrays of mid-point positions and generated curve segments |
| `bezc` | bezctx* | Pointer to a Bezier context object (from spiro library) |
| `cmdType` | cornuCmdType_e | Is this CREATE, MODIFY, or hotbar command? |

### Enum: Cornu Command States

```c
enum Cornu_States { NONE,  // No active operation
                     POS_1,     // First endpoint placed; waiting for second
                     LOC_2,     // Second endpoint placed; waiting for confirmation
                     PICK_POINT,// Waiting to pick a mid-point on the curve
                     POINT_PICKED, // A mid-point is selected and ready to drag
                     TRACK_SELECTED  // A track has been selected in modify mode
                   };
```

---

## Anchor Drawing Functions

### `CreateCornuEndAnchor(coOrd p, wBool_t lock)` — Draw an end-point anchor circle

Draws a small blue arc (half-circle) around the given point. If `lock` is TRUE, it's filled solid; otherwise it's an open arc. Used to show whether the endpoint is locked or adjustable during creation/modification.

### `CreateCornuExtendAnchor(coOrd p, ANGLE_T a, wBool_t selected)` — Draw extend handle arrows

Draws three small blue arrow segments pointing outward from point `p` in directions perpendicular to the track tangent. Used when the user holds Shift and drags an endpoint off its natural end point to "extend" the Cornu with a straight or curved extension segment.

### `CreateCornuAnchor(coOrd p, wBool_t open)` — Draw mid-point handle

Draws either a filled circle (if locked) or an open arc around a mid-point on the curve. Used for interactive editing of point positions during modification mode.

---

## Core Command Handler: `CmdCornu`

This is the main entry point, invoked from the hotbar and menu items. It dispatches based on keyboard/mouse actions (`C_START`, `wActionMove`, `C_DOWN`, `C_MOVE`, `C_UP`, `C_OK`, `C_CANCEL`, etc.) through a large switch statement.

### State Machine Overview

1. **NONE** → User places first endpoint (left-click in empty space or on an open track end). Enters `POS_1`.
2. **POS_1** → User places second endpoint. Enters `LOC_2` or `PICK_POINT` depending on whether the end is connected to a track.
3. **POINT_PICKED / TRACK_SELECTED** → Mid-point handles are available for drag interaction.
4. **C_UP** from PICK_POINT returns to PICK_POINT (allowing multiple points). C_OK confirms and creates the Cornu; C_CANCEL resets state.

### Key Functions Called During Command Flow

- `CreateBothEnds()` — Draws all endpoint anchor circles and mid-point handles.
- `CallCornuM()` / `CallCornu0()` — Computes a Bezier approximation of the Cornu given two endpoints (and optional intermediate points).
- `CorrectHelixAngles()` — Adjusts angles for helices/circles so they are oriented correctly.
- `CheckHelix()` — Ensures a track isn't already connected to both ends of a circle/helix (would create a loop).

---

## Cornu Creation and Construction

### `CallCornu(coOrd pos[2], track_p trk[2], EPINX_T ep[2], dynArr_t * array_p, cornuParm_t * cp)` — Compute the curve segments for a Cornu given end conditions

This is the primary constructor. It:
1. Reads the end parameters from either connected tracks (via `GetTrackParams`) or from user-placed endpoints (`Da.pos`, `Da.radius`, `Da.center`).
2. Calls `CallCornu0()` which uses Raph Levien's Bezier context to produce a list of arcs and straight segments that approximate the Cornu spiral between the two given ends.
3. Returns TRUE on success, FALSE if no valid solution exists (e.g., too much winding).

### `CreateBothEnds(int selectEndPoint, int selectMidPoint, int selectEndHandle, int lastSelected)` — Draw all endpoint and mid-point handles

Populates segment arrays for drawing the interactive handles:
- If an endpoint is connected to a track, draws small red/blue circles around its open end.
- If `track_modifyable` is TRUE (the track's Cornu can be edited), the inner circle is drawn as a selectable ring rather than filled.
- Mid-points that have been added are drawn similarly.
- Extend handles (Shift+drag) appear when the user is extending an endpoint off its natural position.

---

## Modify Command: `AdjustCornuCurve` and `CmdCornuModify`

When the user selects a Cornu track via **Right-click → "Edit"** or similar, this command runs. It sets up `Da.commandType = CORNU_MODIFY` and then uses a similar state machine to allow editing of an existing Cornu's geometry.

### How Modify Works

- The original track is hidden; the editable Cornu (computed from current parameters) is shown.
- User can drag endpoint circles to change radius/angle, or click on the curve to add/remove mid-points.
- **Shift+drag** extends an endpoint off its natural position, inserting a straight or curved extension segment.
- On Enter/OK: the old track segments are deleted; new tracks (straight lines and arcs) are created for any extensions; the Cornu is recreated with updated endpoints/midpoints.

### `UpdateSwitchMotor(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)` — Wait, that's from cswitchmotor.c... 

Actually the modify handler here has a similar edit callback via `cornuModPG` (a param group for end radius/angle fields) and a dialog update function.

---

## Convert Commands

### `CmdConvertTo(wAction_t action, coOrd pos)` — Replace selected tracks with a Cornu track

The user selects one or more connected tracks and invokes "Convert To Cornu":
1. For each selected chain of connected tracks, collect the endpoint positions/angles from all segments.
2. Compute a single Cornu that interpolates through all those endpoints (using mid-points derived from the original segments).
3. Delete all old tracks; insert one new `T_CORNU` track representing the smooth transition.

### `CmdConvertFrom(wAction_t action, coOrd pos)` — Break a Cornu back into individual arcs/lines

The user selects a Cornu and invokes "Convert From":
1. Decompose the Cornu's Bezier approximation into its constituent straight segments (`SEG_STRLIN`) and curved segments (`SEG_CRVTRK`).
2. For each segment, create an actual track object of type `T_CORNU` (or `T_BEZIER` depending on implementation).
3. Reconnect endpoints between consecutive segments so that the result is a chain of ordinary tracks rather than one compound Cornu.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `CmdCornu()` | Main entry point for create and join-with-cornu commands; implements the state machine |
| `AdjustCornuCurve()` | Edit callback for modify mode (dragging handles, adding points) |
| `CmdCornuModify()` | Entry point when user edits an existing Cornu track |
| `CallCornu()` / `CallCornu0()` | Compute the Bezier-approximated Cornu curve from endpoint conditions |
| `CreateBothEnds()` | Draw all interactive handle circles (endpoints and mid-points) |
| `GetAngleSegs()` | Extract angle at a given point along a polyline/curve segment list |
| `CornuLength()` | Compute total length of the Cornu by summing arc lengths and straight segments |
| `CornuOffsetLength(offset)` | Compute length with an offset (used for parallel curves) |
| `CornuMinRadius()` / `CornuMaxRateofChangeofCurvature()` / `CornuTotalWindingArc()` | Quality metrics for validating a proposed Cornu solution |
| `CorrectHelixAngles()` | Adjust angles for helices/circles so they align with the tangent direction |
| `GetTracksFromCornuTrack()` | Decompose a Cornu track into its constituent straight/arc segments |
| `CreateCornuEndAnchor()` / `CreateCornuExtendAnchor()` / `CreateCornuAnchor()` | Draw various handle graphics (endpoint circles, mid-point rings, extend arrows) |

---

## Notes

- **Raph Levien's Cornu Library**: The actual spiral computation is offloaded to a library (`spiro.h`/`spiro.c`) which uses Bezier curves as an approximation. This `ccornu.c` file acts as the interface layer, managing state and user interaction.
- **End handles** are drawn using small arc segments (not filled circles) so that they don't interfere with normal track hit-testing; the cursor is set to invisible (`wCursorNone`) when drawing them so that clicks register on the underlying anchor.
- **Helix/turntable compatibility**: The code handles special cases where an endpoint connects to a turntable or helix (tracks with variable endpoints). These have different rules: a turntable can accept a Cornu at any point along its wall; a helix must not already be connected to both ends (which would form a loop).
- **Conversion**: "Convert To" and "Convert From" provide a way to change the fundamental representation of track geometry — from fixed arcs/straights to a single smooth Cornu, or vice versa. This is useful for simplifying complex layouts into cleaner geometric descriptions.
