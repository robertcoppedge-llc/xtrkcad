# ccornu.c — Cornu Easement Curve Command Handling

## Overview

`ccornu.c` implements the **XTrkCad Cornu easement curve** command system. It bridges Raph Levien's polynomial spiral spline library (`app/cornu/`) with XTrkCad's track database, allowing users to create smooth transition curves between track segments using a drag-and-drop interactive workflow.

The module handles:
- Creating new cornu easements (connect two tracks)
- Modifying existing cornu tracks
- Converting other track types to/from Cornu representation
- Interactive point placement via hotbar or direct click

## File Location

```
app/bin/ccornu.c  (3290 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `track.h` | Track data structures, type definitions |
| `spiro.h` | Polynomial spiral spline math from Raph Levien's library |
| `spiroentrypoints.h` | XTrkCad-specific spiro integration layer |
| `bezctx_xtrkcad.h` | Bezier context manager for approximating spirals |
| `draw.h` | Drawing utilities (`DrawSegs`, `DrawNewTrack`) |
| `ccurve.c` | Base curve command framework |
| `tcornu.h` / `tbezier.h` | Track Cornu and Bezier type definitions |
| `cstraigh.h` | Straight track utilities |
| `drawgeom.h` | Geometric math (angles, distances) |
| `cjoin.h` | Track connection logic |
| `common.h` | Common XTrkCad types and macros |
| `param.h` | Parameter dialog framework |
| `layout.h` | Scale/gauge utilities |
| `cundo.h` | Undo/redo stack management |
| `cselect.h` | Track selection & hit testing |
| `fileio.h` | File I/O operations |
| `common-ui.h` | UI widget types (`wWin_p`, etc.) |

## Key Data Structures

### Global State: `Da` (CmdCornu state object)

```c
typedef struct {
    enum Cornu_States state;          // NONE, POS_1, LOC_2, PICK_POINT, ...
    coOrd pos[4];                     // Endpoints and mid-points
    int number_of_points;             // Count of intermediate points
    int selectEndPoint;               // Which endpoint is selected (0 or 1)
    int selectMidPoint;              // Which midpoint is selected
    int selectEndHandle;             // Which end handle (radius/angle)
    int prevSelected;                // Previous selection for drag

    LWIDTH_T lineWidth;              // Display width of preview curve

    track_p trk[2];                  // Connected tracks at each end
    EPINX_T ep[2];                   // Endpoint index on connected track (-1 = none/turntable)
    DIST_T radius[2];                // Radius at each end (0.0 = straight, -1.0 = open)
    ANGLE_T angle[2];               // Angle at each end
    coOrd center[2];                 // Center of curve at each end

    curveType_e trackType[2];       // Track type at each end (straight/curve/cornu/bezier)
    BOOL_T extend[2];               // Whether to extend with an arc segment
    trkSeg_t extendSeg[2];          // The extending segment
    DIST_T minRadius;               // Minimum radius along the curve

    dynArr_t crvSegs_da;            // Bezier segments approximating the spiral
    int crvSegs_da_cnt;             // Number of bezier segments

    dynArr_t midPoints_da;          // Intermediate control points (for G2 continuity)
    dynArr_t tracks;                // All tracks in the chain

    endHandle endHandles[2];        // Per-end handle state for radius/angle editing

    bezctx * bezc;                  // Bezier context from spiro library
} cornuCmdState_t, *cornuCmdState_p;
```

### `endHandle` — Per-end editor handle

Controls the interactive "arm" that lets users drag to adjust the end point of a Cornu:

```c
typedef struct {
    coOrd end_center;               // The center of the circular arm
    coOrd end_curve;                // The cursor-positioned endpoint on the curve
    DIST_T mid_disp;                // Offset from center (for straight extension)
    BOOL_T end_valid;               // Whether this handle is active
    BOOL_T angle_selected;          // Which sub-handle: radius or angle
    BOOL_T radius_selected;         // Radius vs angle mode
    BOOL_T last_selected;           // Was this the last selected handle?
    ANGLE_T arc_angle;              // Angle of the visible arc segment
} endHandle;
```

### `cornuParm_t` — Parameters passed to spiro solver

```c
typedef struct {
    coOrd pos[2];                   // Endpoint positions
    coOrd center[2];                // Centers (zero if radius is 0)
    ANGLE_T angle[2];              // Angles at endpoints
    DIST_T radius[2];              // Radii (0.0 = straight tangent, -1.0 = open end)
} cornuParm_t;
```

## Enums & Constants

```c
enum Cornu_States { NONE, POS_1, LOC_2, POS_2, PICK_POINT, POINT_PICKED, TRACK_SELECTED };

enum cornuCmdType_e { CORNU_MODIFY, CORNU_CREATE };
```

## Core Functions

### `SetKnots(spiro_cp knots[], coOrd posk[], char type[], int count)`

Populates the knot array that Raph Levien's spiro solver expects. Each knot is a `spiropoint_t` containing position and a character flag:

| Flag | Meaning |
|------|----------|
| `SPIRO_OPEN_CONTOUR` | Start of spiral segment |
| `SPIRO_RIGHT` / `SPIRO_LEFT` | Endpoint with specified tangency direction |
| `SPIRO_G2` | G² (position) continuity constraint — curvature must be zero |
| `SPIRO_G4` | G⁴ constraint — used for inserting intermediate points |

The knots define a piecewise polynomial spiral that is $C^2$-continuous across junctions.

### `CallCornuM(dynArr_t extra_points, BOOL_T end[], coOrd pos[2], cornuParm_t *cp, dynArr_t *array_p, BOOL_T spots)`

Main entry point for generating a Cornu spiral as an array of Bezier segments. It calls into the spiro library via `new_bezctx_xtrkcad` and `TaggedSpiroCPsToBezier`, then closes the context with `bezctx_xtrkcad_close`. If `CallCornuM` fails, the returned count is 0.

### `CreateBothEnds(int selectEndPoint, int selectMidPoint, int selectEndHandle, int lastSelected)`

Draws the two circular "handles" at each end of a cornu track:
- A small inner circle (always shown)
- An outer ring if the endpoint is selectable
- If an `endHandle` is passed and valid, draws additional circles for radius/angle adjustment

This function is called on every draw refresh to show what parts of a Cornu are currently editable.

### `DistanceSegs(coOrd pos0, coOrd pos1, int nsegs, trkSeg_p segs[], coOrd *pos, wIndex_t *inx)`

Finds the closest point along a sequence of segments (Bezier or circular arcs) to a query position, returning both the distance and which segment is nearest. Used extensively for hit-testing during interactive editing.

### `GetAngleSegs(int nsegs, trkSeg_p segs[], coOrd *pos0, wIndex_t *inx, BOOL_T *back, wIndex_t *subinx, BOOL_T *neg)`

Walks along a Bezier track from one end to find the angle of the tangent at a given point. Returns the segment index and whether we approached from "behind." Used when reading back endpoint parameters after a user drags an end point off-track.

### `CornuMinRadius(coOrd pos[4], dynArr_t segs)`

Returns the minimum radius (maximum curvature) along the entire Cornu spiral by recursing into nested Bezier segments. Used for validation against a minimum-radius constraint.

### `CornuMaxRateofChangeofCurvature(coOrd pos[4], dynArr_t segs, DIST_T *last_c)`

Computes $\max |\kappa'(s)| / (2 \cdot \text{segment length})$ along the entire spiral. This metric ensures that the rate at which curvature changes doesn't exceed a comfort threshold — relevant for real-world railway design where sudden jerk would be uncomfortable to passengers.

### `CmdCornu(wAction_t action, coOrd pos)`

Main command handler invoked from hotbar or track menu. States:

- **NONE** → waiting for first endpoint
- **POS_1** → first endpoint placed; awaiting second
- **POS_2** → both endpoints placed; showing preview spiral
- **PICK_POINT** → interactive mode; user drags handles or inserts midpoints
- **POINT_PICKED** → handle being dragged, shows result in real-time

### `AdjustCornuCurve(wAction_t action, coOrd pos, cornuMessageProc message)`

Shared state-machine logic used by both `CmdCornu` and the modify workflow. Handles:
- **C_START**: reset state to NONE; show initial message
- **wActionMove**: drag cursor follows mouse; shows hilite boxes for editable regions
- **C_DOWN**: left-click on a handle or midpoint → sets selection mode
- **C_MOVE**: drags selected point/handle, updates preview
- **C_UP**: release mouse button; either accept (enter) or cancel (esc)
- **C_OK / C_CANCEL**: confirm or abort the operation

### `CmdCornuCreate(wAction_t action, coOrd pos)`

High-level entry when invoked from hotbar. Wraps state transitions and redraws the preview curve after each user action.

### `CmdConvertTo(wAction_t action, coOrd pos)`

Iterates over all selected tracks (excluding turnouts/turntables) and converts them to Cornu easements by:
1. Collecting endpoint parameters from connected neighbors
2. Calling the spiro solver for each gap
3. Deleting old track segments and inserting new Bezier-based Cornu segments
4. Reconnecting adjacent tracks

### `CmdConvertFrom(wAction_t action, coOrd pos)`

Performs the inverse: converts a Cornu easement back into its constituent straight and circular arc segments using `GetTracksFromCornuTrack`, then reconnects neighbors if appropriate.

## Algorithmic Notes

### Piecewise Polynomial Spiral Approximation

The spiro library approximates a polynomial spiral $\kappa(s) = c_0 + \dots + c_4 s^4$ by a chain of circular arcs that together $C^2$-approximate the ideal curve. The number of segments is determined by solving for Bezier control points that minimize approximation error.

### Endpoint Handles

Each end of a Cornu easement has two sub-handles:
- **Radius handle** (red circle) — dragging changes the radius at that end
- **Angle handle** (green circle) — dragging rotates the tangent direction

These are implemented as circular arcs drawn around the endpoint, with an arrow indicating which parameter is active. The `endHandle` structure tracks which sub-handle is currently selected and whether a track extension has been applied.

### Extend Mode

When a user drags an end handle outward beyond the connected track, XTrkCad inserts a small straight or circular arc segment to bridge the gap. This allows the Cornu to grow smoothly into its neighbor rather than snapping abruptly.

## Serialization (File I/O)

Cornu easements are serialized using the same format as other curve types:

```text
CORNU <track-index> <type> "<name>" "<description>"
  POS <x> <y> <angle> <radius> <center-x> <center-y>
  BEZIER [ ...bezier segments... ]
```

The spiro-generated Bezier approximation is stored as a nested `dynArr_t` of track segments, each containing either a circular arc or straight line.

## Notes

- Cornu easements are **not** true clothoids — they use higher-order polynomials for $C^2$ continuity where clothoids only give $C^1$.
- The spiro library is Raph Levien's academic work on polynomial splines; XTrkCad adapts it by wrapping the Bezier context in a track-based data structure.
- Conversion back and forth between Cornu and other representations preserves the original geometry but changes how it is stored internally (Cornu = single object with embedded Bezier array vs. explicit chain of arcs/straights).
