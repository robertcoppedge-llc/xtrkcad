# ccornu.c — Cornu Easement Curve Generation and Editing

## Overview

`ccornu.c` implements **Cornu easement curves** (also known as Euler spirals or clothoid curves) for smooth transitions between track segments of different radii. A Cornu curve has the property that **curvature increases linearly along its length**, which means lateral acceleration changes evenly — a critical comfort and safety requirement in rail design.

The implementation uses Raph Levien's PhD-thesis-based Bezier approximation approach: it generates control points (knots) for a series of cubic Bezier curves that collectively approximate the mathematical Cornu spiral. The final track geometry is then converted into arc/line segments via `ccurve.c`.

---

## Key Concepts

### What is a Cornu Easement?

A Cornu spiral connects two circular arcs (or a straight line and an arc) with **G² continuity** — meaning position, tangent direction, AND curvature are all continuous at the junctions. Unlike simple clothoids which require infinite length to achieve zero-to-infinite-radius transitions, this implementation supports:
- Start or end with a fixed-radius circle (non-zero radius)
- Start or end with zero curvature (straight line)
- Smoothly varying sharpness (the rate of curvature change is derived from endpoint constraints rather than being user-specified)

### Knots and Control Points

The Cornu curve is defined by a sequence of **knot points** along a parameterized "line" in the complex plane. These knots define piecewise cubic Bezier curves whose control polygon approximates the Cornu spiral. The algorithm:
1. Takes two end conditions (position, angle, radius at each end)
2. Interpolates intermediate control points between them
3. Produces a chain of Bezier segments that smoothly connect

### Why Not Use the Mathematical Formula Directly?

The true Cornu integral involves Fresnel integrals which are expensive to evaluate repeatedly during interactive editing. The Bezier approximation approach:
- Is computationally cheap for real-time preview and drag operations
- Can be evaluated using simple arc/line segments (already handled by `ccurve.c`)
- Allows the track to be stored as a series of standard segment types

---

## Core Structures

### `endHandle` — Endpoint Condition Handler

```c
typedef struct {
    coOrd end_center;      // Center point for this endpoint's circular arc condition
    coOrd end_curve;       // Point on the curve at this endpoint (or projected from center)
    DIST_T mid_disp;       // Offset used to position anchor graphics
    BOOL_T end_valid;      // TRUE if this endpoint has a valid, usable condition
    BOOL_T angle_selected; // User is currently dragging the angle control
    BOOL_T radius_selected;// User is currently dragging the radius control
    BOOL_T last_selected;  // TRUE if this is the most recently selected endpoint
    ANGLE_T arc_angle;     // Computed sweep angle of the circular arc segment
} endHandle;
```

This structure holds data for **one side** (start or end) of a Cornu easement. Two instances are maintained simultaneously — one for each track that will be connected.

---

## State Machine

The editing state machine has these states:

| Constant | Value | Meaning |
|----------|-------|---------|
| `NONE` | 0 | Not in a Cornu command; idle |
| `POS_1` | 1 | User is picking the first endpoint location |
| `LOC_2` | 2 | Dragging radius at endpoint 2 |
| `POS_2` | 3 | Dragging position of second endpoint |
| `PICK_POINT` | 4 | Picking a midpoint insertion point along the curve |
| `POINT_PICKED` | 5 | A midpoint has been selected; dragging it changes shape |
| `TRACK_SELECTED` | 6 | Connected track is being modified (e.g., its end condition changed) |

---

## Key Functions

### `CallCornuM(dynArr_t extra_points, BOOL_T end[2], coOrd pos[2], cornuParm_t * cp, dynArr_t * array_p, BOOL_T spots)`

**Main entry point for creating a Cornu curve.** It:
1. Allocates and initializes a new Bezier context (`new_bezctx_xtrkcad`)
2. Constructs an array of 6+ knots (control points) defining the Cornu spiral
3. Calls `TaggedSpiroCPsToBezier()` to close the Bezier context and compute control polygons
4. Writes the resulting segments into the output dynamic array

The knot construction logic:
- If endpoint radius is zero → creates two offset points along the tangent direction (simulating a "zero-curvature" start/end)
- If endpoint radius is non-zero → computes points on the circle at ±5° and ±10° from the connection angle (these become inner/outer control handles of the adjacent Bezier segment)
- Intermediate knots (G₂, G₄ junctions) are placed between the two ends

**Return value:** `TRUE` if a valid Cornu curve was computed; `FALSE` if the algorithm failed (e.g., too much looping detected).

---

### `CallCornu0(coOrd pos[2], coOrd center[2], ANGLE_T angle[2], DIST_T radius[2], dynArr_t * array_p, BOOL_T spots)`

Similar to `CallCornuM` but **without** extra user-inserted midpoint points. Used during initial creation when no midpoints have been added yet.

---

### `createEndPoint(trkSeg_t sp[], coOrd pos0, BOOL_T point_selected, BOOL_T point_selectable, BOOL_T track_modifyable, BOOL_T track_present, ANGLE_T angle, DIST_T radius, coOrd centert, endHandle * endHandle)`

Draws the **endpoint anchor graphics** for a given side of the Cornu curve. It creates:
- A small filled circle at the endpoint (indicates locked/cannot-move) or open arc (free to move)
- If `track_modifyable` is TRUE and no connected track exists, draws extra anchors showing radius-adjustment handles and an angle-hint arrow
- If a connected track exists, draws line segments along the gauge perpendicular to the connection direction

The function also populates `endHandle->arc_angle` with the sweep angle of any circular arc component.

**Return value:** Number of segments drawn (always 1 or 2).

---

### `createMidPoint(dynArr_t * ap, coOrd pos0, BOOL_T point_selected, BOOL_T point_selectable, BOOL_T track_modifyable)`

Creates a midpoint anchor along the Cornu curve. Each midpoint is represented as a small circular arc centered on that location — drag it to insert a new knot (refining the Bezier approximation), or delete it by moving the cursor away.

---

### `DrawTempCornu()`

Draws all the preview graphics for an in-progress Cornu easement:
- The two end anchor circles/lines
- The series of small circles representing midpoints
- The main curve itself (as a black line using `DrawSegs()`)
- Red coloring if the minimum radius along the curve falls below the layout's configured minimum

This is called from the drawing state machine on every mouse event.

---

### `GetConnectedTrackParms(track_p t, const coOrd pos, int end, EPINX_T track_end, wBool_t extend)`

Extracts the endpoint condition parameters (position, angle, radius) from a connected track and stores them into global variables for use by `CallCornuM`. It handles:
- Straight tracks → maps to zero-radius Cornu end
- Circular arcs → extracts center/radius/angle
- Other Cornu easements → reads their stored endpoint parameters
- Turntables → special handling via turntable-specific parameters

**Return value:** `TRUE` if successful.

---

### `CorrectHelixAngles()`

Adjusts the angle values for helical (3D) tracks so that the Cornu computation uses consistent reference directions at each end. If one endpoint belongs to a helix, its angle must be rotated by 180° relative to the other side.

---

### `CheckHelix(track_p trk)`

Validates that a proposed Cornu easement does not create illegal connections: if both connected tracks are helices, their helical directions must be compatible (they point toward each other in space). Returns `FALSE` and displays an error message if incompatible.

---

### `CreateCornuFromPoints(coOrd pos[2], BOOL_T track_end[2])`

Constructs a fully formed Cornu easement track object from two endpoint positions and their associated conditions (angles/radii/centers already stored in global state). Returns a new `track_p` that contains the full segment list. If creation fails, beeps and shows an error with the computed parameters.

---

### `GetAngleSegs(int segCount, trkSeg_t * segs, coOrd * pos, int * segInx, track_p * connectedTrackP, wBool_t * back, int * subinx, wBool_t * neg)`

A utility that walks along a polyline or Bezier-chain (the Cornu curve) and finds the angle at a given point. This is used when midpoints are inserted so that the user can drag them to reshape the curve interactively.

---

## The `cornuParm_t` Structure

```c
typedef struct {
    coOrd pos[2];      // Endpoint positions (pixels from viewport origin)
    ANGLE_T angle[2];  // Tangent angle at each endpoint (degrees CCW from horizontal)
    coOrd center[2];   // Center of circular arc at this end (zero if straight)
    DIST_T radius[2];  // Radius of circular arc; zero means straight line
} cornuParm_t, *cornuParm_p;
```

This is the canonical parameter set used by `CallCornuM` and related functions.

---

## Related Files

| File | Purpose |
|------|---------|
| `ccornu.h` | Type definitions including `endHandle`, `cornuParm_t`, enums |
| `tcornu.c/tcornu.h` | User interface dialogs for Cornu creation/editing (param panel, hotbar) |
| `cbezier.c/cbezier.h` / `tbezier.h` | Bezier curve generation and evaluation |
| `ccurve.c/ccurve.h` | Conversion of Bezier chains to arc/line segments |
| `spiro.h/spiroentrypoints.h` | Raph Levien's spiral/Bézier control point utilities |
