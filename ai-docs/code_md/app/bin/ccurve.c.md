# ccurve.c — Curve Track Creation (Circular Arcs, Helices)

## Overview

`ccurve.c` implements interactive curve track creation tools that allow users to draw circular arc segments and helical spiral tracks. It provides three distinct "from-endpoint" modes for drawing curved tracks from a known endpoint:
- **From End-Pt** — drag along the tangent direction at an existing endpoint
- **From Tangent** — specify center location relative to the tangent at an endpoint
- **From Center** — drag directly to define the radius (center is fixed)

It also supports "from-chord" mode where a full circular arc is defined by two endpoints and the chord connecting them. Additionally, it implements circle creation from a fixed radius or via tangent/center clicks, and helix tracks for vertical spiral transitions between different elevations.

## File Location

```
app/bin/ccurve.c  (1089 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `ccurve.h` | Local declarations (`curveData_t`, curve modes, etc.) |
| `cjoin.h` / `cstraigh.h` | Track joining utilities and straight segment handling |
| `cundo.h` | Undo/redo transaction support |
| `custom.h` / `fileio.h` | Custom types, dynamic arrays, file I/O |
| `cselect.h` | Selection management (`SetAllTrackSelect`, `SnapPos`) |
| `param.h` | Parameter dialog system |
| `track.h` | Track type definitions and creation routines |
| `cbezier.h` / `ccornu.h` | Bezier and Cornu spiral utilities |

## Key Concepts

### Curve Creation Modes

```text
+------------------+------------------------------------------+
| Mode             | Description                                 |
+------------------+------------------------------------------+
| From EP1         | Drag along tangent at endpoint to define  |
|                   | arc direction; center is computed from     |
|                   | chord perpendicular bisector.              |
+------------------+------------------------------------------+
| From Tangent     | Click on desired center location           |
|                   | relative to the endpoint's tangent         |
|                   | direction.                                 |
+------------------+------------------------------------------+
| From Center      | Click anywhere; distance from click        |
|                   | becomes radius. Drag defines sweep angle.  |
+------------------+------------------------------------------+
| From Chord       | Define two endpoints (chord) — the full   |
|                   | circle is generated between them.          |
+------------------+------------------------------------------+
```

### Helix Tracks

A helix track connects an endpoint at a lower elevation to an endpoint at a higher elevation via a circular arc in 3D space:
- **Turns** — number of full vertical revolutions (integer)
- **Angular Separation** — angle between start and end endpoints (degrees, 0–360°)
- **Elevation Difference** — total height change (meters)
- **Grade** — derived from the above as `elevation / (total arc length)`

The helix is rendered as a circular arc with constant radius in model space.

### State Machine States

```c
typedef enum createState_e {
  NOCURVE,         // Not currently creating any curve
  FIRSTEND_DEF,    // First endpoint defined; dragging second point
  SECONDEND_DEF,   // Both endpoints defined; dragging control point for fit
  CENTER_DEF       // Center and radius defined; awaiting final click
} createState_e;
```

## Data Structures

### `struct { ... } Da` — Command State Machine

| Field | Type | Purpose |
|-------|------|---------|
| `state` | `STATE_T` | General command state (-1=off, 0=first end, 1=finalized) |
| `create_state` | `createState_e` | Which step of curve creation are we in? |
| `pos0` / `pos1` | `coOrd` | First and second endpoint coordinates |
| `curveData_t curveData` | — | Computed arc geometry (center, radius, angles) |
| `track_p trk` | pointer | The track object currently being created (if any) |
| `EPINX_T ep` | int | Endpoint index on the neighbor track |
| `down` | BOOL_T | Have we received a valid first click yet? |
| `lock0` | BOOL_T | Is the start endpoint locked to an existing track endpoint? |
| `middle` | `coOrd` | Midpoint of chord (for "from-chord" mode) |

### `struct helixData_s`

Parameters for a helical track:

```c
struct helixData_s {
  long   turns;          // Number of full revolutions (e.g., 5)
  ANGLE_T angSep;        // Angular separation between endpoints in degrees
  DIST_T elev;           // Total elevation change
  DIST_T radius;         // Radius of the circular projection
  DIST_T grade;          // Derived: elev / (turns*2πR + chord_length) * 100%
  DIST_T vertSep;       // Vertical separation per turn
};
```

### `dynArr_t tempSegs_da` & `anchors_da`

Temporary segment arrays used during interactive creation to hold preview geometry and anchor circles.

## Core Functions

### `DrawArrowHeads(trkSeg_p sp, coOrd pos, ANGLE_T angle, BOOL_T bidirectional, wDrawColor color)`

Draws small arrowheads at a given point along a track segment's tangent direction. Used to provide visual guidance during curve creation — red arrows show the valid direction of drag for the current mode. The function writes 1–5 `trkSeg_t` entries into an array depending on whether the arrow should be bidirectional (two-headed) or unidirectional.

### `CreateEndAnchor(coOrd p, dynArr_t * anchor_array, wBool_t lock)`

Creates a small blue circle at point `p` that marks either:
- A locked endpoint (filled black) — indicating that this point is constrained to an existing track's endpoint and cannot be moved freely.
- An unlocked control handle (empty circle with outline) — the user can drag it to adjust the curve radius or direction.

### `CreateCurve(wAction_t action, coOrd pos, BOOL_T track, ...)`

The main state machine for drawing a curved track segment. It handles:
- **START** — reset internal arrays, show mode-appropriate message
- **DOWN** — store first click; if snapping to track endpoint is enabled and ALT not held, snap to that endpoint and lock it via `CreateEndAnchor`. Set initial state.
- **MOVE** — as the mouse moves: constrain motion along tangent/chord direction if locked, update preview segment geometry, show messages with current angle/radius/length values. Draw arrows pointing in valid drag directions.
- **UP/CANCEL** — validate minimum length; compute final arc geometry; create track via `NewCurvedTrack()` or discard.

### `CmdCurve(wAction_t action, coOrd pos)` — Top-level command dispatcher

Dispatches based on `curveMode` which is set by the menu choice (e.g., "Curve from End-Pt", "Curve from Tangent", etc.). Calls `CreateCurve()` internally and handles undo start/end around track creation.

### `PlotCurve(...)` / `ConvertToArcs(...)`

Internal helper functions that compute a circular arc between two endpoints given the current mode constraints, then convert it into a sequence of small straight/arc segments suitable for rendering by the general draw engine. The "from-chord" variant places the midpoint at the chord's center rather than on the track itself.

### `CreateCurveFromChord(...)` / `CreateCurveFromTangent(...)` / `CreateCurveFixedRadius(...)`

These are internal helper functions that compute a circular arc between two endpoints given different constraints (chord length, tangent direction, fixed radius). They are called from `PlotCurve()` and `CmdCircleCommon()`.

### `CmdCircleFixedRadius(wAction_t action, coOrd pos)` / `CmdCircleFromTangent(...)` / `CmdCircleFromCenter(...)`

Three modes for creating a full-circle track:
- **Fixed Radius** — user enters radius in a dialog; clicking anywhere places that circle centered at the click point.
- **From Tangent** — first click sets tangent direction, second click sets center location.
- **From Center** — first click defines center, dragging to edge sets radius; second click finalizes.

All share a common implementation via `CmdCircleCommon()`.

### `ComputeHelix(paramGroup_p pg, int h_inx, void *data)`

Callback invoked each time the helix dialog parameter changes. It recomputes derived values:
- If turns and angular separation change → vertical separation per turn changes.
- If elevation changes → total length or grade adjusts automatically.
- The "Total Length" message field is updated to show the computed track length.

### `CmdHelix(wAction_t action, coOrd pos)`

Similar state machine to curve creation but uses a helical arc that rises in elevation as it advances around its circle. On OK, calls `NewCurvedTrack()` with 0° start angle and a full turn count (e.g., 5 turns) and the derived end angle.

## File Format (.xtp Export)

Circular arcs are serialized using the standard track segment format:

```text
TURNOUT HO "My Curve"
    P "Route1" 1 0
    E -450.732 -1829.654 270.0
    C -6000.0 0.0 0.0 0.0 360.0   ; center at origin, radius=-6000, full circle
    T 0
END_TURNOUT
```

Helical tracks are exported by decomposing them into a sequence of small straight segments (along the helix) and arc segments that approximate the spiral's curvature in projection. Each segment is written as either `SEG_STRLIN` or `SEG_CRVTRK`.

## Notes

- All curve creation modes support undo/redo via `UndoStart()` / `UndoEnd()`.
- The "from-chord" mode is useful when you want to define a circular arc by its chord endpoints rather than an endpoint and tangent direction.
- Helix tracks are mathematically helical (constant slope) but rendered as piecewise arcs for compatibility with the general track drawing engine.
