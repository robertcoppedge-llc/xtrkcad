# cturnout.c — Turnout Placement and Connection Logic

## Overview

`cturnout.c` implements the interactive **turnout placement** system for xtrkcad. A turnout (switch) is a track segment that branches from one mainline track to another. The file handles:
- Selecting a turnout template from a dialog.
- Dragging to position the turnout on the layout canvas.
- Connecting each end-point of the turnout to nearby tracks.
- Splitting existing tracks at connection points (creating new track segments).
- Detecting and resolving conflicts (e.g., two turnouts connecting to the same track segment in incompatible directions).

---

## Key Data Structures

### `turnoutInfo_t` — Turnout Template Record

```c
typedef struct turnoutInfo_s {
    coOrd orig;                      // Origin point for drawing
    ANGLE_T angle;                   // Rotation angle of the template
    dynArr_t segs_da;               // Array of trkSeg_t segments defining the turnout geometry
    wIndex_t endCnt;                // Number of end-points (typically 2)
    track_p *endPt;                 // Pointer to array of endpoint records
    coOrd tempEndPts[4];            // Temporary storage for trial placements
    dynArr_t title_da;              // Human-readable description/title string
    BOOL_T customInfo;              // If TRUE, additional info is stored in extra-data block
    struct extraDataCompound_t *u;  // Union-specific data (e.g., switch motor association)
} turnoutInfo_t, *turnoutInfo_p;
```

Each turnout template consists of a sequence of segments (`segs_da`) that define its curved or straight geometry. The `endCnt` field indicates how many endpoints the turnout has (usually 2).

### `Dto` — Placement Command Context

```c
typedef struct {
    int state;                      // Current phase: 0=unplaced, 1=being placed, 2=rotating
    coOrd pos;                      // Current mouse position / trial placement point
    track_p trk;                   // The adjacent track to which the turnout will connect
    ANGLE_T angle;                 // Angle of the adjacent track at connection point
    coOrd rot0, rot1;             // Rotation anchor points (for rotating turnouts)
} Dto;
```

### `turnoutInfo_t` — Turnout Template Record

Each entry in `turnoutInfo_da` holds a different turnout geometry (e.g., "6#3" means a 6-inch divergent turnout with 3-degree switch angle). The template includes:
- **`segs_da`**: A piecewise-defined curve made of straight and curved segments.
- **`endCnt`**: Number of endpoints (typically 2, one at each end of the turnout).
- **`tempEndPts[4]`**: Temporary coordinates used during trial placement to compute connection angles.

---

## Core Functions

### `DrawTurnout(track_p trk, drawCmd_p d)` — Render a Turnout Track

Draws the turnout track segments stored in `trk->extra.segs`. The function:
1. Retrieves the compound extra-data block (`extraDataCompound_t`).
2. Iterates over each segment and dispatches based on its type:
   - `SEG_CRVLIN` / `SEG_CRVTRK`: draws a circular arc using `DrawCircularArc()`.
   - `SEG_STRLIN`: draws a straight line segment.

---

### `PlaceTurnoutTrial(track_p *trkR, coOrd *posR, ANGLE_T *angle1R, ANGLE_T *angle2R, int *connCntR, DIST_T *maxDR, vector_t *v)` — Trial Placement Evaluation

Given a tentative placement position (`pos`), this function:
1. Determines the track closest to that point (or NULL if none is nearby).
2. Computes the angle of that track at the connection point.
3. Iterates over each endpoint of the turnout template, translating it by the trial offset and computing its relative orientation to the adjacent track.
4. Collects all valid connections into a vector array (`v`).

Returns:
- `trkR`: The adjacent track (or NULL).
- `posR`: Adjusted placement position (snapped if close enough to an existing endpoint).
- `angle1R`, `angle2R`: Angles of the two endpoints relative to their respective tracks.
- `connCntR`: Number of valid connections found.
- `maxDR`: Maximum offset distance among connected endpoints.

---

### `PlaceTurnout(coOrd pos, track_p trk)` — Place Turnout at a Given Position

Called when the user releases the mouse after dragging. It:
1. Stores the trial placement into global state (`Dto.pos`, `Dto.trk`, `Dto.angle`).
2. If more than one connection exists and the shift key is not held, tries to **optimize** the placement by adjusting position slightly to maximize the number of connections while minimizing maximum offset.

This optimization step handles cases where a turnout could plausibly connect to multiple nearby tracks — it picks the configuration that gives the best geometric fit.

---

### `AddTurnout(void)` — Finalize Turnout Placement and Connect Tracks

The main placement handler, called when the user presses Enter/Space or clicks "OK":
1. Validates that all endpoints have been placed (`endCnt` segments exist).
2. Calls `UndoStart(_("Place New Turnout"))`.
3. For each endpoint of the turnout template:
   - Computes its absolute position by adding the trial offset to the adjacent track's endpoint position.
   - If a valid connection exists (i.e., a nearby track lies within `connectDistance` and aligns within `connectAngle`), it calls `SplitTrack()` on that neighbor at the connection point, creating two new segments: one belonging to the turnout and one continuing along the original track.
   - The newly created segment is copied into the turnout's extra-data block, inheriting attributes (visibility, ties, etc.) from the neighbor unless explicitly overridden.
4. For any "leftover" segments (those that could not find a valid connection), the function attempts to reconnect them if possible — for example, by extending an endpoint of the turnout or trimming a leftover segment to match a nearby Cornu spiral.
5. Sets `visible` and `no_ties` flags on the new turnout track based on the connected tracks.
6. Calls `DrawNewTrack()` and `UndoEnd()`.

---

### `ConnectTracks(track_p trk, int epIndex, track_p neighbor, int neighborEp)` — Connect Two Track Segments at a Junction

Given two endpoints (`trk` at `epIndex`, `neighbor` at `neighborEp`) that are nearly coincident:
1. If the neighbor is a Cornu spiral and within 2× gauge distance, it calls `SetCornuEndPt()` to attach the turnout as an "incoming" branch of the Cornu. The turnout becomes a child segment of the Cornu's path tree.
2. Otherwise, if both endpoints are nearly aligned (angle difference ≤ `connectAngle`), it calls `ConnectTracks()` which:
   - Updates endpoint metadata for both tracks.
   - Calls `UndoModify()` on each track to record the connection change in the undo stack.
3. If neither condition is met, the connection is rejected and no modification occurs.

---

### `CreateArrowAnchor(coOrd pos, ANGLE_T a, DIST_T len)` — Draw a Connection Arrow

Creates two small line segments forming an arrowhead pointing along direction `a` from point `pos`. Used to visually indicate the direction of track flow at a turnout junction when dragging or rotating.

---

### `CreateRotateAnchor(coOrd pos)` / `CreateMoveAnchor(coOrd pos)` — Draw Rotation/Movement Guides

- **Rotation anchor**: draws a small arc around `pos` with three arrowheads indicating the allowed rotation range (±45°).
- **Movement anchor**: draws two line segments with arrowheads indicating allowable translation directions.

These are drawn while the user drags to position/rotate a turnout template.

---

### `TurnoutChange(long changes)` — Redraw Turnout Dialog When Parameters Change

Responds to dialog updates (scale change, parameter change) by:
1. Hiding/showing the turnout list control.
2. Rebuilding the list of available turnouts from `turnoutInfo_da`.
3. Calling `TurnoutAdd()` to populate the list box with entries.
4. Selecting the current template and redrawing its preview in the dialog window.

---

### `RedrawTurnout(wDraw_p d, void *context, wWinPix_t x, wWinPix_t y)` — Redraw Turnout Preview

Rescales the turnout preview to fit within the dialog window, then draws the selected turnout template at a scaled size centered in the preview area. Also highlights the currently selected endpoint with a white dot.

---

### `TurnoutOk(void)` — Dialog "OK" Button Handler

Called when the user accepts the selected turnout template:
1. Calls `AddTurnout()` to perform actual placement (reusing global state).
2. Hides and resets the dialog window.
3. Calls `UndoEnd()`.

---

### `TurnoutDlgUpdate(paramGroup_p pg, int inx, void *valueP)` — Dialog Control Callback

Responds to changes in the turnout dialog parameters:
- If a list control (turnout selection) is updated, it calls `AddTurnout()` and sets the current template pointer.
- Other parameter changes are ignored for now.

---

### `TOpickEndPoint(coOrd pos, turnoutInfo_p to)` — Find Which Endpoint Is Closest to Mouse Position

Iterates over all endpoints of a turnout template and returns the index of the closest one. Used to highlight/label the endpoint currently under the cursor while dragging.

---

## Summary

| Aspect | Detail |
|--------|--------|
| Turnout representation | Piecewise curve defined by `segs_da` (mix of straight + circular arc segments) stored in a global pool (`turnoutInfo_da`) |
| Placement model | User selects a template from a dialog, drags to position, optionally rotates with Ctrl-drag; on release the system snaps connections to nearby tracks and splits them at junction points |
| Connection logic | For each endpoint, finds the nearest track within `connectDistance` (default ~2× gauge) whose tangent angle matches within `connectAngle`; if valid, the neighbor is split and a new segment is copied into the turnout's extra-data block |
| Conflict resolution | If an existing track already has two incoming branches from the same Cornu spiral, additional connections are rejected to prevent topology errors |
