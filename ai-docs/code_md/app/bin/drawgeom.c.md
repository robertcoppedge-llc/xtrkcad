# drawgeom.c — Drawing Operations (Lines, Arcs, Polygons, Anchors)

## Overview

`drawgeom.c` implements **interactive drawing primitives** using an event-driven state machine. It handles mouse-down/move/up events to create lines, arcs, circles, filled shapes, polygons, and benchwork annotations. The module also provides a rich set of visual anchors that give user feedback about what actions are available via double-click or right-click menus.

---

## Core Design Pattern: State Machine Drawing

The drawing system uses a **state machine** (`drawContext_t`) to track multi-step operations. Each operation has discrete states:
- `State == 0` — Idle / waiting for initial click
- `State == 1` — Dragging (update live preview)
- `State == 2` — Command complete, awaiting final confirmation

---

## Structures

### `drawContext_t` — Drawing Operation State Machine

```c
typedef struct {
    enum OpType op;                 // Current operation type (line, circle, poly, etc.)
    int state;                      // State: 0=idle, 1=dragging, 2=complete
    BOOL_T started;                 // TRUE if mouse-down event was seen
    BOOL_T changed;                 // TRUE if user modified geometry since last start
    BOOL_T undo_started;            // TRUE if UndoStart() was called for this op
    coOrd pos[4];                  // Accumulated point history (for polygons)
    DIST_T radius;                  // Current radius value (circles, arcs)
    ANGLE_T angle;                  // Current angle value
    DIST_T length;                  // Current segment length
    curveData_t arcData;           // Curve fitting data (center, radius, a0, a1)
} drawContext_t, *drawContext_p;
```

| Field | Description |
|-------|-------------|
| `op` | Operation type: line, circle, filled circle, polygon, bench, etc. |
| `state` | Current state in the operation lifecycle |
| `started` | Whether this command has received a mouse-down event yet |
| `changed` | Flag indicating geometry was modified (triggers undo recording) |
| `undo_started` | Whether an undo transaction was begun for this operation |
| `pos[]` | History of cursor positions; used to detect polygon closure and compute segment angles |
| `radius` | Live radius value being dragged by the user |
| `angle` / `length` | Live angle/length values for dimension lines, benchwork, etc. |
| `arcData` | Computed curve parameters (center point, radius, start/end angles) used during drag preview and finalization |

---

### `polyState_e` — Polygon Selection State

```c
typedef enum { POLY_NONE, POLY_SELECTED, POLYPOINT_SELECTED } polyState_e;
```

Tracks whether a polygon vertex is currently selected for modification.

---

## Key Functions

### `DrawGeomMouse(wAction_t action, coOrd pos, drawContext_t *context)`

Main entry point invoked from the command dispatcher on every mouse event during drawing mode. Dispatches based on `action` type (`C_START`, `wActionLDown`, `wActionLDrag`, `wActionLUp`, etc.) and the current operation type.

**Returns:** `STATUS_T` — one of:
- `C_CONTINUE` — keep processing events in this command
- `C_TERMINATE` — finish the drawing command, finalize geometry
- `C_ERROR` — error occurred (e.g., polygon too few sides)

---

### State Machine Behavior

#### On `wActionLDown` / `wActionRDown` (Mouse Button Pressed)

1. Reset temporary segment array (`tempSegs_da`) and anchors
2. Initialize state to 0 (idle), set `started = TRUE`
3. For most operations, store the first click as both endpoints (`pos0 = pos1 = pos`)
4. Set up operation-specific initial behavior:

   | Operation | Behavior on first click |
   |-----------|-------------------------|
   | Line, benchwork, table edge | Ready to drag second point; shows length/angle feedback |
   | Circle 1 / FilledCircle 1 | Show "Drag to set radius" cursor hint |
   | Box / FillBox | Create a 4-segment rectangle from the single point (all sides zero-length) |
   | Polygon | Ready for first point; next points extend the polygon chain |

---

#### On `wActionLDrag` / `wActionRDrag` (Mouse Dragged)

This is where live preview happens. The behavior differs by operation:

- **Line / Benchwork** — updates `.pos[1]` to current cursor position; shows length/angle tooltip
- **Circle 2** — sets radius = distance from pos0 to current mouse position
- **Curve1–4** — uses `PlotCurve()` (from `ccurve.c`) to compute the circular arc that fits between pos0 and current position with given radius; draws preview arcs and shows red arrowheads for interactive adjustment
- **Polygon** — appends a new straight segment to the polygon chain; detects closure when mouse returns near starting point

---

#### On `wActionLUp` / `wActionRUp` (Mouse Button Released)

Finalizes the geometry:

1. If operation is in state 2 (complete), calls `DrawGeomOk()` to write segments into a track and record undo
2. For lines/arcs/polygons, computes final values (angle = arctan2(dy,dx))
3. Detects **polygon closure** — if the last segment endpoint is within `eps` of the first point, closes the polygon by linking back to start

---

## Operation Types (`enum OpType`)

| Constant | Meaning | Shape Produced |
|----------|---------|----------------|
| `OP_LINE` | Simple line between two points | Straight track centerline |
| `OP_DIMLINE` | Dimension/annotation line | Same as OP_LINE but with different color and no undo |
| `OP_BENCH` | Benchwork annotation | Straight segment with material data |
| `OP_TBLEDGE` | Table edge marker | Special annotation for tabletop modeling |
| `OP_CURVE1` | Drag from endpoint (curvature varies along arc) | Circular arc with varying curvature |
| `OP_CURVE2` | Drag from tangent angle | Circular arc, radius set by cursor distance |
| `OP_CURVE3` | Drag from center point | Circular arc centered at first click |
| `OP_CURVE4` | Drag from chord midpoint | Circular arc via chord-midpoint method |
| `OP_CIRCLE1` | Fill a circle (closed filled shape) | Filled circular region |
| `OP_CIRCLE2` | Draw an arc (not filled) | Open circular arc |
| `OP_CIRCLE3` | Arc with radius set by drag distance | Same as OP_CIRCLE2 but radius from pos0→pos1 |
| `OP_FILLBOX` | Filled rectangle | Rectangular polygon region |
| `OP_BOX` | Empty (outline-only) rectangle | Polygon without fill |
| `OP_POLY` / `OP_POLYLINE` | Open or closed arbitrary polygon | User draws freeform shape; Enter/Tab closes it |
| `OP_FILLPOLY` | Filled arbitrary polygon | Same as POLY but with filled interior |

---

## Anchor System

Anchors are small graphical overlays drawn in a separate dynamic array (`anchors_da`) on top of the drawing canvas. They appear only during an active drawing session and disappear when the command completes or is cancelled. Each anchor is itself a `trkSeg_t` describing lines/arcs that visually indicate what actions are available via double-click/right-click.

### `CreateEndAnchor(coOrd p, wBool_t lock)`

Draws a small filled circle (if `lock == TRUE`) or open arc at point `p`. Used to mark locked endpoints of arcs/circles — the anchor signals "this endpoint is already connected and cannot be moved freely."

---

### `CreateLineAnchor(coOrd p, coOrd p0)`

Draws a line from cursor position `p` back to the first click (`p0`) with an arrowhead. Used during circle-creation mode to show the radius direction.

---

### `CreateSquareAnchor(coOrd p)`

Draws four outward-pointing arrows arranged as a square around point `p`. Signals that this object can be translated (moved) by dragging.

---

### `CreateCurveAnchors(int index, coOrd pm, coOrd pc, coOrd p0, coOrd p1)`

Draws three small filled circles:
- One at the arc center (`pc`) with an arrow pointing toward the curve
- Two at each endpoint of the arc (`p0`, `p1`), each showing a perpendicular tangent indicator
The red/blue coloring differentiates which anchor is active for interaction.

---

### `CreateBoxAnchors(int index, pts_t pt[4])`

For rectangles, draws:
- A central small circle indicating the center point (selectable to rotate)
- Four corner circles with outward arrows indicating draggable corners
- Diagonal arcs connecting opposite corners showing the rectangle is a single selectable object

---

### `CreatePolyAnchors(int index)`

Draws anchors for each vertex of a polygon:
- A small filled circle centered on each vertex (selectable to move that point)
- If two adjacent points are selected, an arc between them indicates they can be dragged together as a side
The red/blue coloring distinguishes the currently active anchor.

---

### `CreateMovingAnchor(coOrd pos, BOOL_T fill)`

Draws a small circle with (optionally) an inner filled center to indicate the object is in "move" mode — typically used when the user has invoked the move/transform command and is about to drag an object around.

---

## Utility Functions

### `FindTempNear(drawContext_t *context, coOrd *p)`

For operations that have a live preview (curves, lines, etc.), this returns TRUE if the current cursor position lies within a small tolerance (`eps`) of the currently drawn geometry. This is used to determine whether to lock the cursor to the nearest point on the curve/line during dragging.

---

### `CleanSegs(drawContext_t *context)`

Called when starting a new drawing operation: resets the temporary segment array, clears anchor arrays, and prepares for fresh input.

---

## Related Files

| File | Purpose |
|------|---------|
| `drawgeom.h` | Type definitions for draw contexts, operation enums |
| `track.h` / `cstruct.c` | Track/segment data structures (`trkSeg_t`, track list) |
| `ccurve.c/ccurve.h` | Curve-fitting math (used by OP_CURVE* operations) |
| `tbezier.c/tbezier.h` | Bezier approximation utilities for curved tracks |
| `cdraw.c` | Drawing routines that consume the segment data produced here |
