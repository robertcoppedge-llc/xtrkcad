# cbezier.c - Bezier Curve Creation and Modification

## Overview

Bezier curves are defined as a set of four points: the first and fourth are endpoints, while the middle two are control points. The program approximates cubic Bezier curves using circular arcs for most operations (joining, offsetting, etc.) since direct Bezier math is computationally difficult for tasks like creating parallel tracks.

---

## Structures

### `bCurveData_t` - Embedded in Bezier Segment Extra Data

```c
typedef struct {
    curveData_t curveData;   // Arc approximation data (center, radius, a0, a1)
    double start;            // Start parameter t (0.0)
    double end;              // End parameter t (1.0)
    coOrd pos0;              // First endpoint
    coOrd pos1;              // Second endpoint
} bCurveData_t;
```

### `extraDataBase_t` - Base Header for All Extra Data

Embedded at the start of every track segment's extra data union, providing:
- `type`: Segment type identifier (e.g., T_BEZTRK)
- `nextExtra`: Pointer to next segment in chain (for undo/redo traversal)
- `isDeleted`: Flag marking deleted segments

---

## Functions

### `createControlArm(trkSeg_t sp[], coOrd pos0, coOrd pos1, BOOL_T track, BOOL_T selectable, BOOL_T cp_direction_locked, int point_selected, wDrawColor color)`

Creates a control arm graphic consisting of:
- A line segment between two points (the control arm itself)
- Optional filled or unfilled circles at each endpoint (for selection handles)
- Red color if the point is currently selected; black otherwise

**Parameters:**
| Parameter | Description |
|-----------|-------------|
| `sp` | Output array for track segments (up to 3: line + up to 2 circles) |
| `pos0` | End point on the curve |
| `pos1` | Control point at the other end of the arm |
| `track` | Whether this is a Bezier track or a Line |
| `selectable` | Whether selection handles should be drawn |
| `cp_direction_locked` | Whether the control point is locked to a track direction (cannot be dragged) |
| `point_selected` | Index of selected point (0=left end, 1=right end) or -1 for none |
| `color` | Color for drawing (`wDrawColorBlack`, `wDrawColorRed`) |

**Returns:** Number of segments created.

---

### `getPoint(coOrd pos[4], double s)`

Evaluates a cubic Bezier curve at parameter `s` where `s ∈ [0,1]`. Uses the standard Bernstein polynomial form:

```
P(s) = (1-s)^3 * P0 + 3*(1-s)^2*s * P1 + 3*(1-s)*s^2 * P2 + s^3 * P3
```

**Parameters:**
- `pos` - Array of the four defining points
- `s` - Parameter in [0, 1]

---

### `BezError(coOrd pos[4], coOrd center, coOrd start_point, double start, double end)`

Computes the maximum error between a circular arc and the Bezier curve. Uses a binary search approach to find the largest arc that stays within tolerance (error ≤ 0.5 pixels at max zoom).

**Parameters:**
- `pos` - Four points defining the Bezier curve
- `center` - Center of the approximating arc
- `start_point` - Start point of the arc segment
- `start`, `end` - Parameter values for arc start/end (in [0,1])

---

### `DistanceToLineSegment(coOrd p, coOrd l1, coOrd l2)`

Computes distance from a point to an infinite line segment and returns the closest point on that segment. Handles three cases:
- Beyond the far end → return far endpoint
- Beyond the near end → return near endpoint  
- Between endpoints → return perpendicular projection

**Parameters:**
- `p` - Query point
- `l1`, `l2` - Endpoints of the line segment

---

### `BezErrorLine(coOrd pos[4], coOrd start_point, coOrd end_point, double start, double end)`

Computes error between a straight line and a Bezier curve (used for line segments within composite Bezier curves).

**Parameters:** Same as `BezError` but checks distance to the line rather than a circular arc center.

---

### `addSegBezier(dynArr_t * array_p, trkSeg_p seg)`

Copies track segment data into a dynamic array for internal use in Bezier processing. Handles recursive copying of nested Beziers (for composite curves).

**Parameters:**
- `array_p` - Pointer to the destination dynArr
- `seg` - Source track segment to copy

---

### `AnalyseCurve(coOrd inpos[4], double *Rfx, double *Rfy, double *cusp)`

Classifies a Bezier curve type using mathematical analysis based on work by Maureen C. Stone (XeroxPARC) and Tony deRose (U of Washington). Returns one of:
- `ENDS` - Identical endpoints (degenerate)
- `LINE` - Collinear points forming a straight line
- `COINCIDENT` - Three or more coincident points
- `PLAIN` - A simple open curve with no inflection/cusp/loop
- `INFLECTION` - Has an inflection point (cusp on one side of tangent)
- `DOUBLEINFLECTION` - Two inflection points
- `CUSP` - Has a cusp
- `LOOP` - Forms a closed loop

**Parameters:**
- `inpos[4]` - The four defining points
- `Rfx`, `Rfy` - Output: x and y components of the radius vector (for arc approximation)
- `cusp` - Output: distance from cusp to inflection point location

---

### `ConvertToArcs(coOrd pos[4], dynArr_t * segs, BOOL_T track, wDrawColor color, LWIDTH_T lineWidth)`

The core conversion routine. Takes a Bezier curve and decomposes it into circular arcs that approximate the curve within 0.5 pixel tolerance at maximum zoom. Uses binary search to find the widest valid arc for each segment.

**Algorithm:**
1. Start with the full curve (t_s=0, t_e=1)
2. Use binary search to find the largest sub-arc that stays within error threshold
3. Split off the best arc and recurse on the remainder
4. Any remaining segment shorter than a minimum is rendered as a straight line

**Parameters:**
- `pos[4]` - The four defining points of the Bezier curve
- `segs` - Output dynamic array receiving the resulting segments (straights and arcs)
- `track` - Whether this is a track or a line
- `color` - Color for rendering
- `lineWidth` - Line width

**Returns:** TRUE on success, FALSE if conversion failed.

---

### `DrawBezCurve(trkSeg_p control_arm1, int cp1Segs_cnt, trkSeg_p control_arm2, int cp2Segs_cnt, trkSeg_p curveSegs, int crvSegs_cnt, wDrawColor color)`

Renders a Bezier in edit mode: draws the approximating arc segments plus the visible control arms. Control arms are drawn in black; selected points appear red.

**Parameters:**
- `control_arm1` - First control arm (left side) or NULL
- `cp1Segs_cnt` - Number of segments in first control arm
- `control_arm2` - Second control arm (right side) or NULL
- `cp2Segs_cnt` - Number of segments in second control arm
- `curveSegs` - Array of curve approximating segments
- `crvSegs_cnt` - Number of curve segments to draw
- `color` - Color for the curve

---

### `DrawTempBezier(BOOL_T track)`

Renders a temporary (preview) Bezier curve during creation. Shows red if min radius < layout minimum, otherwise normal color. Draws both control arms and the approximating arcs.

**Parameters:**
- `track` - Whether this is a track or line

---

### `CreateBothControlArms(int selectPoint, BOOL_T track)`

Sets up the visible control arm graphics based on which point is currently selected (`selectPoint`). Draws filled circles for locked points and empty circles for unlocked ones. Red color highlights the active selection.

**Parameters:**
- `selectPoint` - Index of selected control point (0=left end, 1=right end) or -1 for none
- `track` - Whether this is a track or line

---

### `CreateMoveAnchor(coOrd pos, BOOL_T fill)`

Creates a blue circular anchor mark at a given position. Used during drag operations to show the current cursor position over the Bezier being edited.

**Parameters:**
- `pos` - Position for the anchor circle
- `fill` - TRUE for filled (dragging), FALSE for outline (hover)

---

### `AdjustBezCurve(wAction_t action, coOrd pos, BOOL_T track, wDrawColor color, LWIDTH_T lineWidth, bezMessageProc message)`

Handles all mouse input events during Bezier creation and modification. Manages the state machine:
- `C_START` - Initialize, show endpoints, prompt to select first endpoint
- `wActionMove` - Show anchor circle under cursor if close to a valid point
- `C_DOWN` - Select an endpoint; transition to POINT_PICKED state
- `C_MOVE` - Drag the selected control point; recompute curve approximation
- `C_UP` - Release point; return to PICK_POINT mode for next selection
- `C_OK` (Enter/Space) - Confirm and create the Bezier track/line
- `C_CANCEL` (Esc) - Abort and discard

**Parameters:**
- `action` - Mouse event code
- `pos` - Current cursor position
- `track` - Whether creating a track or line
- `color` / `lineWidth` - Styling for the preview curve
- `message` - Callback function to show info messages (e.g., InfoMessage)

**Returns:** C_CONTINUE or C_TERMINATE

---

### `CmdBezModify(track_p trk, wAction_t action, coOrd pos, DIST_T trackG)`

Modifies an existing Bezier track by allowing the user to adjust its control points. The original track is hidden and a preview of the modified version is shown. The workflow mirrors creation but operates on an existing curve.

**Parameters:**
- `trk` - Pointer to the existing Bezier track being modified
- `action` - Mouse event code
- `pos` - Current cursor position
- `trackG` - Track gauge for snapping validation

---

### `BezierLength(coOrd pos[4], dynArr_t segs)`

Computes the total length of a Bezier curve by summing lengths of its constituent arc and straight segments. Handles composite curves with nested Beziers recursively.

**Parameters:**
- `pos` - Four defining points (used for validation)
- `segs` - The approximating segment array

---

### `BezierMinRadius(coOrd pos[4], dynArr_t segs)`

Computes the minimum radius of curvature along a Bezier curve by examining all arc segments. Used to check against layout constraints (minimum turn radius).

**Parameters:**
- `pos` - Four defining points (used for validation)
- `segs` - The approximating segment array

**Returns:** Minimum radius, or DIST_INF if no curves present.

---

### `CmdBezCurve(wAction_t action, coOrd pos)`

Main entry point for creating a new Bezier curve (track or line). Coordinates the full workflow:
1. User selects first endpoint (snaps to unconnected track end or free line position)
2. Drags out first control arm
3. Selects second endpoint
4. Drags out second control arm
5. Confirms with Enter/Space

**Parameters:**
- `action` - Mouse event code
- `pos` - Cursor position

---

### `InitCmdBezier(wMenu_p menu)`

Initializes the Bezier command system and registers the "Create Bezier Curve" button in the main menu.

**Parameters:**
- `menu` - Pointer to the main command menu widget

---

## Related Files

| File | Purpose |
|------|---------|
| `ccurve.h/ccurve.c` | Circular arc utilities (PlotCurve, etc.) |
| `cjoin.h/cjoin.c` | Track joining (uses arc approximations) |
| `drawgeom.h/drawgeom.c` | Geometric operations used internally |
| `tbezier.h/tbezier.c` | Pure Bezier math and curve properties |
