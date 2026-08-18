# drawgeom.c — Drawing Geometry Commands (Lines, Polygons, Arcs, Boxes, Fills)

## Overview

`drawgeom.c` implements the interactive drawing system for creating geometric primitives in XTrkCAD. It handles:

- **Straight lines** (`SEG_STRLIN`)
- **Dimension lines** (`SEG_DIMLIN`)
- **Bench marks** (`SEG_BENCH`)
- **Table edges** (`SEG_TBLEDGE`)
- **Circular arcs** (`SEG_CRVLIN`, `SEG_FILCRCL`) — full and partial circles
- **Polygons** (`SEG_POLY`, `SEG_FILPOLY`) — free-form, open polyline, rectangle
- **Bezier curves** (via delegate)

The file implements a state machine that responds to mouse actions: start, down, move, up, confirm/cancel. It uses temporary segment arrays (`tempSegs_da` and `anchors_da`) for intermediate geometry display during the drawing process.

---

## Core Data Structures

### `drawContext_t` — Drawing Context Structure

The main state container passed to all drawing functions:

| Field | Type | Purpose |
|-------|------|---------|
| `State` | int | Current step in the draw sequence (0=ready, 1=drawing, 2=done) |
| `Op` | long | Operation type (`OP_LINE`, `OP_CIRCLE1`, etc.) — encodes which shape and sub-mode |
| `radius` | double | Current radius of circle/arc being drawn |
| `angle` | float | Angle for line/drag direction |
| `length` | DIST_T | Computed length of current segment |
| `ArcData` | struct | Holds center, radius, a0/a1 angles for arc creation |
| `type`, `subtype`, `filled`, `open` | — | For polygon: shape type and fill state |

---

### `drawModContext_t` — Modification Context

Used when modifying existing objects. Contains the original segment pointers (`segPtr`) and offset/rotation information needed to map user edits back into the track structure.

| Field | Type | Purpose |
|-------|------|---------|
| `orig`, `angle` | coOrd, ANGLE_T | Origin point and rotation angle for the object being modified |
| `segPtr` | trkSeg_p* | Array of pointers to original segments in the track |
| `prev_inx`, `max_inx` | int | Previous index and maximum index within a polygon |

---

### PolyState / PolyInx — Polygon Edit State

```c
typedef enum {POLY_NONE, POLY_SELECTED, POLYPOINT_SELECTED} PolyState_e;
static PolyState_e polyState = POLY_NONE;
static coOrd rotate_origin;
static ANGLE_T rotate_angle;
```

Tracks whether the user is in point-selection mode (moving polygon vertices) or origin/rotation mode.

---

## Command Mode Enumerations

### `OP_*` — Drawing Operations

| Constant | Value | Description |
|----------|-------|-------------|
| `OP_LINE` | 1 | Draw a straight line segment |
| `OP_DIMLINE` | 2 | Dimension line (parallel guide) |
| `OP_BENCH` | 3 | Bench mark symbol |
| `OP_TBLEDGE` | 4 | Table edge indicator |
| `OP_CIRCLE1` / `OP_FILLCIRCLE1` | 5/6 | Full circle, fill color variant |
| `OP_CIRCLE2` / `OP_FILLCIRCLE2` | 7/8 | Radius drag mode (center locked) |
| `OP_CIRCLE3` / `OP_FILLCIRCLE3` | 9/10 | Center click then radius drag mode |
| `OP_CURVE1..4` | 11–14 | Curved track creation modes (delegate to `ccurve.c`) |
| `OP_BOX` / `OP_FILLBOX` | 15/16 | Rectangle/freeform polygon, with or without fill |
| `OP_POLY` / `OP_FILLPOLY` | 17/18 | Free-form polygon creation (click points) |
| `OP_POLYLINE` | 19 | Open polyline (no closing required) |

The first digit of the operation code distinguishes shape type; the last two digits distinguish variants.

---

## Core Functions

### `EndPoly(context, cnt, open)` — Finalize a Polygon

When a polygon draw action ends, this function:
- Validates that at least 3 points exist (otherwise issues an error)
- Creates a dynamic array of point structures (`pts_t`) from the internal buffer
- Constructs a single segment with type `SEG_POLY` or `SEG_FILPOLY` and stores the point array

**Parameters:**
- `context`: The drawing context holding temporary geometry
- `cnt`: Number of vertices in the polygon
- `open`: TRUE for open polyline (`POLYLINE`), FALSE for closed free-form polygon

---

### `DrawGeomOk(started)` — Commit Segments to Track

Called when the user finishes a draw operation (click-up or OK). For each segment currently held in `tempSegs_da`, it calls `MakeDrawFromSeg()` to create a real track segment and draws it. If `started` is FALSE, an undo transaction is started first.

---

### `CreateEndAnchor(pos, lock)` — Create Anchor Marker for Curved Tracks

Creates a small circular arc centered at `pos` (radius ≈ 7.5 units) that marks where a curve endpoint is being defined. Blue if free to move; red if locked (`lock=TRUE`). Used during interactive curved track creation.

---

### `CreateLineAnchor(p, p0)` — Create Line Endpoint Anchor

Creates two small arcs centered at `p` and an additional arc from `p` toward `p0`. Used in line drawing to indicate the current endpoint position relative to the starting point (`p0`).

---

### `CreateSquareAnchor(pos)` — Create Square Marker for Polygon Edges

Creates four arrow-like segments arranged into a square around `pos`, indicating that this edge of a polygon is selected and can be moved. Used during polygon vertex editing.

---

### `FindTempNear(context, p)` — Point-in-Curve/Line Test

A "near" test: determines whether the mouse position lies close to an existing temporary segment (either on a line or circular arc). Returns TRUE if within a small tolerance. This is used for snapping during drag operations.

---

### `DrawGeomMouse(action, pos, context)` — Main Drawing Dispatcher

The central function that dispatches all drawing actions based on the current operation (`context->Op`). It handles mouse up/down/move events and updates the temporary geometry buffer accordingly.

**Key modes handled:**
- Lines: draw a segment from one point to another; show length/angle feedback
- Circles: center-click then radius-drag, or tangent-point method
- Rectangles: click two opposite corners; also supports fill toggle
- Polygons: click successive vertices until closing the shape

---

### `DrawGeomPolyModify(action, pos, context)` — Polygon Vertex Editing Mode

When a polygon is already drawn and the user enters edit mode (double-click), this function handles:
- **Point selection:** clicking near a vertex selects it for movement; new points can be inserted between edges
- **Dragging:** moving a selected point updates neighboring edge lengths and angles, preserving adjacent edge constraints
- **Delete/Backspace:** removes the most recently added point if at least 3 vertices remain
- **Vertex type switching:** 'o'/'s'/'v'/'r' keys switch between smooth, straight, and round corners

---

### `DrawGeomOriginMove(action, pos, context)` — Move Rotation Origin / Translate Object

Used when a user wants to reposition the "pivot point" of an object (e.g., for rotating about a different center). The origin is shown as a crosshair marker; selecting a new origin changes where rotations/transformations are applied.

---

### `DrawGeomModify(action, pos, context)` — Unified Modify Dispatcher

The top-level modify function that dispatches to sub-handlers based on the segment type (`SEG_STRLIN`, `SEG_CRVLIN`, `SEG_POLY`, etc.). It handles:
- **Line endpoints:** drag to move; Ctrl+Shift locks to 90° relative angle from previous line (useful for perpendicular offsets)
- **Circle arcs:** dragging center changes radius; dragging an endpoint changes the swept arc angle while preserving radius; holding Shift preserves radius and only changes start/end angles
- **Polygons:** vertex selection, edge dragging with right-angle constraint (Ctrl key), corner point movement

---

## Geometry Helper Functions

### `CreateCurveAnchors(index, pm, pc, p0, p1)`

Creates anchor markers for a curved arc:
- An arc at the center (`pc`) if the segment is a full circle
- Two arcs at the endpoints (`p0`, `p1`) indicating sweep direction
- An arrow marker at the midpoint (`pm`) indicating the bisector of the arc

The index parameter selects which anchor type to draw (center vs. endpoint).

---

### `CreateCircleAnchor(selected, center, radius, angle)`

Creates a small arc centered on a circle's center point that indicates where the current sweep angle points. Used during circle creation and editing.

---

### `BuildCircleContext(context, segInx)` — Prepare Circle Edit Context

Computes the midpoint of an existing circular segment (`pm`), its center (`pc`), and stores them in the context so they can be used for interactive editing. Computes the start angle (`a0`) and end angle (`a1`) from the stored geometry.

---

### `CreateMovingAnchor(pos, fill)` — Temporary Anchor During Drag

Creates a small circular arc centered at `pos`. If `fill` is TRUE, uses an inner radius (`d/4`) to show it's the currently-selected point being dragged. Used during drag operations when no specific anchor type exists.

---

## State Machine Summary

The drawing system operates as a finite state machine:

| Action | Typical Effect |
|--------|---------------|
| `C_START` | Reset context; set up message or cursor; initialize undo start flag |
| `wActionMove` (drag) | Update preview geometry in `tempSegs_da`; show anchors; update info message with current length/angle/radius |
| `wActionLDown` / `wActionRDown` | Lock first point (`pos0 = pos`); switch to drawing state (`State=1`) |
| `C_MOVE` | Continue dragging: compute next segment, validate nearness to existing geometry, show preview |
| `C_UP` | Finalize the object; create undo transaction if needed; return control |
| `C_CONFIRM` / `C_OK` | Same as C_UP but with explicit user confirmation (e.g., from a dialog) |

The `State` field in the context tracks: 0 = idle, 1 = drawing (preview), 2 = finalized.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `EndPoly(context, cnt, open)` | Finalize a polygon; create point array and track segment | context, vertex count, boolean for polyline vs closed |
| `DrawGeomOk(started)` | Commit all segments in the temp buffer to real tracks | started flag (undo needed?) |
| `CreateEndAnchor(pos, lock)` | Draw a small circular marker at a curve endpoint | coordinate, lock boolean |
| `CreateLineAnchor(p, p0)` | Mark a line's current endpoint relative to its start | current point, starting point |
| `CreateSquareAnchor(pos)` | Create four-segment square marker for polygon edge selection | center coordinate |
| `FindTempNear(context, p)` | Test if mouse is close to any existing temp segment | context, test point; returns boolean |
| `DrawGeomMouse(action, pos, context)` | Main entry; dispatches based on operation type and action | action code, mouse position, context pointer |
| `DrawGeomPolyModify(action, pos, context)` | Handle polygon vertex selection and movement during edit mode | action, position, modify context |
| `DrawGeomOriginMove(action, pos, context)` | Move the rotation/translation origin for an object | action, position, context |
| `DrawGeomModify(action, pos, context)` | Unified dispatcher for all drawing operations (lines, arcs, boxes) | action, position, context |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Implement an interactive drawing system that lets users create lines, circles, polygons, and other geometric primitives by clicking and dragging on the screen. It also supports editing existing objects (moving endpoints, resizing arcs, moving polygon vertices). |
| **Domain** | Interactive computer graphics / CAD geometry modeling: mouse-driven shape creation with immediate visual feedback through temporary buffers (`tempSegs_da`, `anchors_da`). |
| **Key concept** | The system uses a state machine (`context->State`) and a dynamic array of segments (`tempSegs_da`) to show a live preview as the user drags. The preview is never part of the track database until `C_UP` or `C_OK` commits it. Anchors (small arcs/lines) are drawn to indicate which parts of the object are being edited and in what direction. |
| **Main entry points** | `DrawGeomMouse()` — called from the main event loop for drawing operations; `DrawGeomModify()` — called when modifying an existing track segment |
