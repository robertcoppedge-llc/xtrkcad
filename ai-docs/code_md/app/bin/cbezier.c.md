# cbezier.c — Bezier Curve Creation and Editing

## Overview

`cbezier.c` provides infrastructure for **Bezier curves** in xtrkcad. Bezier curves are defined by four control points: start, control point 1, control point 2, and end. The curve is rendered as a series of circular arcs that approximate the true Bezier form within a tolerance of 0.5 pixels at maximum zoom.

This approximation approach allows xtrkcad to use standard track operations (splitting, connecting, parallel offsetting) which are difficult or impossible with pure Bézier mathematics.

---

## Key Data Structures

### `struct bCurveData` — Bezier Curve Editing State

```c
typedef struct {
    curveData_t curveData;   // Arc approximation data from ccurve.c
    double start;            // Start parameter (0.0) of the arc segment
    double end;              // End parameter (1.0) of the arc segment
    coOrd pos0;              // First endpoint of the Bezier
} bCurveData_t, *bCurveData_p;
```

### `Da` — Global Bezier Editing Context

```c
static struct {
    enum Bezier_States state;       // Current editing phase
    coOrd pos[4];                   // The four control points of the Bezier
    int selectPoint;                // Index of currently selected point (-1 if none)
    track_p trk[2];                 // Tracks connected to each end (for turnsouts)
    EPINX_T ep[2];                  // Endpoint index on connected tracks
    dynArr_t crvSegs_da;            // Array of arc segments approximating the Bezier
    int crvSegs_da_cnt;             // Number of arc segments in `crvSegs_da`
    trkSeg_t cp1Segs_da[4];         // Control arm 1 (visual handle)
    trkSeg_t cp2Segs_da[4];         // Control arm 2 (visual handle)
    int cp1Segs_da_cnt;             // Number of segments in control arm 1
    int cp2Segs_da_cnt;             // Number of segments in control arm 2
    BOOL_T unlocked;                // Whether the Bezier is locked to existing tracks
    track_p selectTrack;            // The track currently being modified (if editing)
    BOOL_T track;                   // TRUE = creating a track, FALSE = line
    DIST_T minRadius;              // Minimum radius of curvature along the curve
} Da;
```

---

## Core Functions

### `AnalyseCurve(coOrd inpos[4], double *Rfx, double *Rfy, double *cusp)` — Characterize a Bezier Curve

This function determines what kind of geometry a given set of four control points produces. It returns one of:

| Return value | Description |
|---|---|
| `ENDS` | Start and end points are coincident (no valid curve) |
| `LINE` | A perfectly straight line (all points collinear) |
| `PLAIN` | A standard, well-behaved Bezier curve |
| `CUSP` | The curve has a cusp (sharp point where tangent is undefined) — invalid for tracks |
| `LOOP` | The curve crosses itself forming a loop — invalid for tracks |
| `INFLECTION` | The curve changes direction such that the radius of curvature becomes infinite — potentially problematic |
| `DOUBLEINFLECTION` | Two inflection points exist — rare but possible |

The function uses analytic geometry derived from work by Maureen C. Stone (Xerox PARC) and Tony deRose (U Washington). It translates the control points so the start point is at origin, then computes parameters that determine curve type.

---

### `ConvertToArcs(coOrd pos[4], dynArr_t *segs, BOOL_T track, wDrawColor color, LWIDTH_T lineWidth)` — Approximate a Bezier with Circular Arcs

This is the central conversion routine. It:
1. Uses **binary search** to find the transition points between adjacent circular arcs along the curve.
2. For each arc segment, it finds the circle that best fits the Bézier over an interval `[t_s, t_e]` such that the maximum deviation error is ≤ 0.5 pixels (the tolerance threshold).
3. Returns `TRUE` on success, `FALSE` if conversion failed (e.g., due to inflection points or cusps).

The binary search works by:
- Starting with the largest possible arc (`t_e = 1.0`).
- Halving the interval until the error falls below threshold or the next larger arc would exceed tolerance.
- If the full curve can be represented by one arc, that's used directly.

---

### `CreateControlArm(trkSeg_t sp[], coOrd pos0, coOrd pos1, BOOL_T track, BOOL_T selectable, BOOL_T cp_direction_locked, int point_selected, wDrawColor color)` — Build Visual Control Handle

Creates a visual handle (control arm) for the interactive editor. It draws:
- A straight line segment from `pos0` to `pos1`.
- Two small circles at each end if the point is selectable and unlocked. The selected point's circle is drawn red; others are black. If the control point is locked (i.e., constrained to a connected track), its circle is filled.

---

### `DrawTempBezier(BOOL_T track)` — Draw Preview of Current Bezier Edit

Renders the current state of the interactive editor:
- The approximated arc segments in normal or exception color (if radius < minimum allowed).
- Two control arms showing which points are currently draggable.

---

### `AdjustBezCurve(wAction_t action, coOrd pos, BOOL_T track, wDrawColor color, LWIDTH_T lineWidth, bezMessageProc message)` — Handle Mouse Input for Bezier Editing

This is the main event dispatcher for interactive Bezier editing. It handles:
- **`C_START`** — Reset state to `NONE`, show info message "Select End-Point".
- **`wActionMove`** (while in `PICK_POINT`) — Show anchor circles around each selectable endpoint; display move guides near the cursor.
- **`C_DOWN`** — Select the nearest valid endpoint and enter `POINT_PICKED` state; show info message "Drag point N to new location".
- **`C_MOVE`** — Drag the selected control point, recompute arc approximation, show radius/length statistics or error messages (cusp, loop, etc.).
- **`C_UP`** — Release selection; if a track is connected, snap the endpoint to an unconnected track endpoint if SHIFT is held.
- **`C_OK` / `C_TEXT` (Space)** — Finalize and create the new track/line. If the curve has cusps or loops, display an error and abort.
- **`C_CANCEL`** — Abort editing; discard all temporary data.

---

### `CmdBezModify(track_p trk, wAction_t action, coOrd pos)` — Modify an Existing Bezier Track

This function is called when the user invokes the "Modify" command on a selected Bezier track. It:
1. Reads the current control point values from the extra-data block of the original track.
2. Enters the same interactive editing sequence as `CmdBezCurve`.
3. On confirmation, it undoes the modification to the old track and draws the new one in its place.

---

### `BezierLength(coOrd pos[4], dynArr_t segs)` — Compute Total Length of a Bezier (Approximated)

Walks through all arc segments that approximate the Bézier curve and sums their lengths using `fabs(radius * D2R(angle_range))`.

---

### `BezierMinRadius(coOrd pos[4], dynArr_t segs)` — Find Minimum Radius of Curvature Along a Bezier

Returns the smallest absolute radius among all arc segments. This is used to detect whether any part of the track curves tighter than the minimum allowed radius for the current scale.

---

### `BezierOffsetLength(dynArr_t segs, double offset)` — Compute Offset Length along a Bezier

This function computes the length of an offset curve (parallel curve) by walking through each arc segment and adjusting its radius by `±offset` before computing arc length. This is used for drawing centerlines or parallel tracks alongside a Bézier track.

---

## Summary

| Aspect | Detail |
|--------|--------|
| Representation | Four control points; rendered as approximating circular arcs via binary search |
| Editing model | Interactive "control arm" handles similar to Adobe Illustrator / CorelDraw |
| Track snapping | Endpoints can be locked/unlocked to existing unconnected track endpoints (via SHIFT+drag) |
| Validation | Cusps and loops are rejected; inflection points may produce very large radii that could exceed minimum radius constraints |
