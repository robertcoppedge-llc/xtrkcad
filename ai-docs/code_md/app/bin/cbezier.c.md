# cbezier.c — Bezier Curve Creation and Modification Commands

## Overview

`cbezier.c` implements the command for creating and modifying **Bezier curves** in XTrkCad. A cubic Bézier curve is defined by four control points: two endpoints and two internal control points that determine the curve's shape via Bernstein polynomial blending.

The file provides both a creation wizard (`CmdBezCurve`) and an edit mode modifier (`CmdBezModify`). It also converts Bézier curves into sequences of circular arcs (for efficient rendering) and analyzes their topology to detect invalid forms like loops or cusps.

## File Location

```
app/bin/cbezier.c  (1289 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `common.h` | Core types (`coOrd`, `track_p`, `wAction_t`, etc.) |
| `draw.h` | Drawing primitives (`DrawSegs`) |
| `ccurve.c` / `ccurve.h` | Circular arc curve data and functions |
| `tbezier.h` | Bézier track/line data structures (from `app/tools/tcurve`) |
| `cstraigh.h` | Straight segment handling |
| `drawgeom.h` | Geometric utilities (`FindDistance`, `FindAngle`) |
| `cjoin.h` | Track joining utilities |
| `track.h` | Track types and definitions |
| `wcolors.h` | Widget colors (`wDrawColorBlack/White/Red/Blue`) |
| `param.h` | Parameter types (`LWIDTH_T`, `DIST_T`) |
| `fileio.h` / `layout.h` / `cundo.h` / `compound.h` | File I/O, layout context, undo support |

## Key Concepts

### Cubic Bézier Curve Definition

A cubic Bézier curve is defined by four points:

```text
P(t) = (1-t)³·P0 + 3(1-t)²t·P1 + 3(1-t)t²·P2 + t³·P3   for t ∈ [0,1]
```

- **P0** — Start endpoint
- **P1** — First control point (pulls the curve toward it near P0)
- **P2** — Second control point (pulls the curve toward it near P3)
- **P3** — End endpoint

### Why Convert to Arcs?

XTrkCad renders tracks as a sequence of circular arcs and straight lines. Bézier curves are converted because:
1. Many operations (parallel offsetting, track joining, etc.) work naturally on arc/straight segments.
2. Exporting to DXF or other formats requires segment-by-segment representation.

### Bézier Types Allowed

| Type | Description |
|------|-------------|
| `PLAIN` | Standard S-shaped curve |
| `LINE` | All four points are collinear (degenerate) |
| `INFLECTION` | Has an inflection point but no loop/cusp |
| `DOUBLEINFLECTION` | Two inflections |
| `LOOP` | Self-intersecting — **rejected** as invalid for tracks |
| `CUSP` | Sharp cusp / sharp turn — **rejected** as invalid |
| `ENDS` | Identical endpoints (zero-length) — rejected |
| `COINCIDENT` | Three or more points coincide — rejected |

Only `PLAIN`, `LINE`, `INFLECTION`, and `DOUBLEINFLECTION` are valid. Loops, cusps, and degenerate forms are rejected with an error message.

## Data Structures

### `enum Bezier_States`

```c
enum { NONE,
       POS_1,              // First endpoint placed; dragging first control arm
       CONTROL_ARM_1,      // First control arm dragged; awaiting second endpoint
       POS_2,              // Second endpoint placed; dragging second control arm
       PICK_POINT,         // Picking a point on the curve (modify mode)
       POINT_PICKED,      // A point is picked and being dragged
       TRACK_SELECTED     // In modify mode, original track shown
     };
```

### `bCurveData_t`

Holds data for one arc segment of an approximated Bézier:

```c
typedef struct {
  curveData_t curveData;   // center, radius, start/end angles
  double start;            // param t where this arc begins (0.0–1.0)
  double end;              // param t where this arc ends (start ≤ end ≤ 1.0)
  coOrd pos0, pos1;        // endpoints of the arc segment in model coords
} bCurveData_t;
```

### `static struct { ... } Da` — The command's private state

| Field | Type | Description |
|-------|------|-------------|
| `state` | `enum Bezier_States` | Current wizard step or mode |
| `pos[4]` | `coOrd`[] | The four Bézier control points |
| `selectPoint` | `int` | Which point is selected (-1 = none) |
| `trk[2]` | `track_p`[] | Neighbor tracks attached to endpoints (in track mode) |
| `ep[2]` | `EPINX_T`[] | Endpoint index on the neighbor track |
| `crvSegs_da` | `dynArr_t trkSeg_t` | Dynamic array of arc segments approximating the Bézier |
| `cp1Segs_da`, `cp2Segs_da` | arrays of `trkSeg_t` | Control arms (visual handles) |
| `unlocked` | `BOOL_T` | Whether control points are locked to direction |
| `selectTrack` | `track_p` | The track being modified (in modify mode) |
| `track` | `BOOL_T` | Are we creating a Track or a Line? |
| `minRadius` | `DIST_T` | Minimum radius of curvature for the Bézier |
| `trackGauge` | `DIST_T` | Gauge of this track (for validation) |

### `enum BezierType AnalyseCurve(...)` return values

```c
enum { PLAIN, LOOP, CUSP, INFLECTION, DOUBLEINFLECTION, LINE, ENDS, COINCIDENT } bType;
```

## Core Functions

### `getPoint(coOrd pos[4], double s)`

Evaluates the Bézier curve at parameter `s ∈ [0,1]` using de Casteljau's algorithm:

```c
double mt = 1-s;
double a=mt*mt*mt, b=mt*mt*s*3, c=mt*s*s*3, d=s*s*s;
return (a*pos[0]+b*pos[1]+c*pos[2]+d*pos[3]);
```

### `BezError(...)` / `BezErrorLine(...)`

Compute the maximum perpendicular distance between a trial circular arc and the Bézier curve. The arc is considered "good" if the error ≤ 0.5 pixels (at maximum zoom). This binary-search-driven approximation builds the segment list in `Da.crvSegs_da`.

### `AnalyseCurve(coOrd pos[4], double *Rfx, double *Rfy, double *cusp)`

Determines which of the eight Bézier types the curve falls into by:
1. Checking for coincident endpoints → `ENDS`
2. Checking collinearity (all four points lie on one line) → `LINE`
3. Translating so P0 is at the origin and computing coefficients from the implicit form of a conic fitted to the control points.
4. Comparing against the cusp condition curve: `cusp = -(fx²-2fx+3)/4`.

Returns one of the eight types. Loops, cusps, and degenerate curves trigger error messages.

### `ConvertToArcs(...)` — Bézier → Circular Arc Segments

This is the core approximation function. It works by:
1. Binary search to find a trial arc (`t_s` … `t_e`) that satisfies the error threshold (≤ 0.5px).
2. If good, record it in `Da.crvSegs_da`, then advance `t_s = prev_e` and repeat with a wider span.
3. If bad, shrink the trial interval by moving `t_e` inward.

The function returns `FALSE` if no valid approximation can be found (e.g., extreme curvature that even tiny arcs cannot represent within tolerance).

### `createControlArm(...)`

Draws one of the two Bézier control arms. Each arm is a visual handle showing:
- A line from the endpoint to its associated control point.
- Two small circles at each end — filled red if it's the currently selected point, empty black otherwise. Filled black means "locked" (direction fixed by a neighboring track).

Returns the number of segments drawn.

### `DrawTempBezier(...)` / `CreateMoveAnchor(...)`

When in edit mode, these functions redraw the temporary Bézier with its control arms and the underlying arc-segment representation. Anchors are blue circles that appear under each control point so the user knows where they can click to grab them.

## Command: `CmdBezCurve` — Creating a Bézier Track or Line

**State machine:** `START → POS_1 → CONTROL_ARM_1 → POS_2 → PICK_POINT → POINT_PICKED → OK/CANCEL`

| Step | User Action | What happens |
|------|-------------|---------------|
| START | Click to place first endpoint | Shows "Drag end of first control arm" |
| MOVE (first control point) | Drag the handle at the start-endpoint | The line between P0 and P1 follows; releasing moves it freely. If a track is attached, dragging while holding ALT locks that control point to stay aligned with that track's tangent. |
| UP | Click elsewhere (no drag) | State advances: `CONTROL_ARM_1 → POS_2` showing "Select other end of Bezier" |
| MOVE (second endpoint/control arm) | Drag the handle at the far-endpoint | The second control arm is drawn; releasing it completes placement. If a track is attached, ALT locks that side. |
| UP/OK | Click elsewhere or press ENTER | Validates and finalizes the Bézier, converting to arcs and creating the `track_p` object. |
| CANCEL | ESC | Discards the in-progress Bézier; resets state to `NONE`. |

**Track vs Line:** If `cmd == bezCmdCreateTrack`, the curve is a track (gauge-checked against layout gauge). Otherwise it's an ordinary line drawing.

## Command: `CmdBezModify` — Modifying an Existing Bézier

When called from the Modify command, it loads the existing Bézier's four points into `Da.pos[]` and lets the user drag them just like in create mode. The original track is hidden; once the user confirms (ENTER), the old track is deleted and replaced with a new one built from the modified control points.

## Notes

- Bezier curves are internally always represented as circular arcs for rendering and DXF export.
- A Bézier that approximates a straight line will have all its arc segments be straights (`SEG_STRLIN`).
- The `-v` (verbose) option in `bdf2xtp.c` is unrelated; this module has no command-line interface — it's purely an interactive drawing tool invoked by the GUI.
