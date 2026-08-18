# ccornu.c — Clothoid (Cornu Spiral) Track Creation and Modification

## Overview

`ccornu.c` implements **clothoid transition curves** for XTrkCAD. A clothoid (also called a Cornu spiral or Euler spiral) is defined by the property that its curvature varies linearly with arc length:

```
κ(s) = s / A²   where κ is curvature and A is the spiral parameter
R(s) = A² / s   radius of curvature at arc position s
```

This means a train traveling at constant speed experiences a **linear increase in lateral acceleration**, which is the primary comfort criterion for passengers. Clothoids are also mathematically elegant — they can be joined end-to-end with G2 (curvature-continuous) continuity, making them ideal for smooth track transitions.

The implementation uses **Raph Levien's Bezier approximation library** to represent clothoids as sequences of Bézier curve segments, enabling them to be drawn, edited, and manipulated within XTrkCAD's track system.

---

## Key Concepts

### Why Clothoids for Railways?

Traditional circular arcs have a sudden jump in curvature at their start/end — this causes an instantaneous change in lateral acceleration (a "jerk" impulse) that passengers feel as uncomfortable. A clothoid eliminates this by smoothly ramping the radius from infinite (straight track) to finite (circular arc), or between two different radii.

The clothoid's defining differential equation is:

```
dκ/ds = constant   ⇒   κ(s) = s / A²
```

where `A` is a design parameter with units of length × angle. The spiral parameter `A` determines how "tight" or "loose" the transition is — smaller `A` means curvature increases more rapidly (sharper transition).

### G2 Continuity

At each junction between two clothoid segments, both the position and tangent are continuous (G1), and the **curvature** is also continuous (G2). This means there's no sudden change in centripetal acceleration — the ride quality is smooth.

---

## Data Structures

### `cornuParm_t` — Input Parameters for Clothoid Computation

```c
typedef struct {
    coOrd pos[2];          // Endpoints of the transition curve
    ANGLE_T angle[2];      // Tangent direction at each endpoint
    DIST_T radius[2];      // Radius at each end (0.0 = straight track)
    coOrd center[2];       // Center point (for circular ends; zero for straight)
} cornuParm_t;
```

### `cornuCmdType_e` — Command Mode Enum

| Constant | Value | Purpose |
|----------|-------|---------|
| `cornuJoinTrack` | 1 | Join two existing tracks with a clothoid transition |
| `cornuCmdCreateTrack` | 2 | Create a new clothoid track from scratch (free-form placement) |
| `cornuCmdHotBar` | 3 | Hotbar palette button for quick access |

### `Da` — Global Command State Structure

```c
static struct {
    enum Cornu_States state;              // NONE, POS_1, LOC_2, PICK_POINT, etc.
    coOrd pos[4];                         // Endpoints and intermediate points
    int number_of_points;                 // Number of interior constraint points
    cornuCmdType_e commandType;           // Create vs. join mode
    track_p trk[2];                       // Connected tracks on each side
    EPINX_T ep[2];                        // Which endpoint index connects to the other track? (-1 if none)
    DIST_T radius[2];                     // Radius at each end (0 = straight)
    ANGLE_T angle[2];                     // Tangent direction at endpoints
    coOrd center[2];                      // Center of circular end arcs
    curveType_e trackType[2];            // Type of connected track

    BOOL_T extend[2];                     // Are we extending the endpoint with a straight/circular segment?
    trkSeg_t extendSeg[2];                // Temporary storage for extension preview geometry

    dynArr_t crvSegs_da;                  // Output: array of track segments forming the clothoid
    int crvSegs_da_cnt;                   // Number of segments in the output array

    // Midpoint constraint points (G2 continuity anchors)
    dynArr_t midPoints;                   // User-added interior G2 anchor points
    dynArr_t tracks;                      // Stack of all connected track objects

    endHandle endHandle[2];              // Handle for radius/angle adjustment handles at endpoints

    bezctx *bezc;                         // Pointer to Raph Levien's Bezier context (allocated on demand)

    DIST_T minRadius;                    // Computed: minimum radius along the curve
} Da;
```

The `Da` structure is global because clothoid creation is a **multi-step wizard** that spans many mouse events. It remembers which points have been placed, whether endpoints are locked to tracks, and maintains intermediate computation results between events.

---

### `endHandle_t` — Endpoint Adjustment Handle

Each endpoint of a free-form clothoid has an attached handle allowing the user to interactively adjust:
- **Radius** (by dragging perpendicular to the chord)
- **Angle / position** (by dragging along the track direction)

```c
typedef struct {
    coOrd end_center;                  // Center point for radius adjustment
    coOrd end_curve;                   // Curve point used for angle/radius display
    DIST_T mid_disp;                   // Longitudinal offset from chord midpoint
    BOOL_T end_valid;                  // Handle is active (not yet dismissed)
    BOOL_T angle_selected;             // Is the angular handle selected?
    BOOL_T radius_selected;            // Is the radial handle selected?
    BOOL_T last_selected;              // Was this handle most recently clicked?
    ANGLE_T arc_angle;                 // Arc angle of the end segment
} endHandle;
```

---

## Core Functions

### `CallCornuM(...)` — Compute Clothoid via Bezier Approximation

This is the **heart** of the clothoid implementation. It calls into Raph Levien's Bezier approximation library to generate a sequence of Bézier curves that approximate a clothoid between two endpoints with specified tangent directions and curvatures (radii).

**Signature:**
```c
BOOL_T CallCornuM(
    dynArr_t extra_points,              // Optional interior G2 anchor points
    BOOL_T end[2],                      // Are endpoints "open" or fixed-radius?
    coOrd pos[2],                       // Start and end positions
    cornuParm_t * cp,                   // Target (center, angle, radius) at each end
    dynArr_t * array_p,                 // Output: allocated array of trkSeg_t
    BOOL_T spots                        // TRUE = use approximation; FALSE = exact computation
);
```

**How it works:**

1. **Set up "knots"** — A sequence of control points that define the clothoid's geometry:
   - Two knots at each endpoint, offset perpendicularly from the tangent by a small distance (determines curvature rate)
   - One knot exactly at each endpoint
   - Additional knots for interior G2 anchor points (user-added midpoints along the curve)

2. **Call `new_bezctx_xtrkcad`** — Initialize Raph Levien's Bezier context with the number of internal knots and a scale factor.

3. **Set knot types:**
   - `SPIRO_OPEN_CONTOUR` — Start/end of an open spiral (free endpoint)
   - `SPIRO_G2` — G2 (curvature-continuous) interior point
   - `SPIRO_RIGHT` / `SPIRO_LEFT` — Endpoints with fixed radius/angle

4. **Compute Bézier approximation** via `TaggedSpiroCPsToBezier`, which decomposes the clothoid into a sequence of Bézier segments.

5. **Close the context** and return the resulting segment array.

The function also handles the special case where one or both ends are "open" (radius = 0, meaning straight track), in which case the endpoint knot is positioned along the tangent direction at a small offset to define the curvature rate.

---

### `CallCornu0(...)` — Direct Computation Without Extra Points

Same as `CallCornuM` but for the simpler two-endpoint case without interior G2 anchors:

```c
BOOL_T CallCornu0(
    coOrd pos[2],              // Endpoints
    coOrd center[2],           // Centers of circular ends (or zero for straight)
    ANGLE_T angle[2],          // Tangent direction at each end
    DIST_T radius[2],          // Radius (0 = straight)
    dynArr_t * array_p,        // Output segment array
    BOOL_T spots               // Whether endpoints are "open" or fixed-radius
);
```

The `spots` flag indicates whether the endpoints are treated as open spirals (`TRUE`) or have a defined radius (`FALSE`). Open endpoints correspond to connections with straight track; fixed-radius endpoints connect to circular arcs.

---

### `CreateCornuFromPoints(...)` — Build Track Object from Computed Segments

```c
track_p CreateCornuFromPoints(
    coOrd pos[2],              // Endpoints of the clothoid
    BOOL_T end_point[2]       // TRUE = this side is open (straight); FALSE = fixed radius
)
```

This function converts a computed clothoid into an actual `track` object:

1. **Determine endpoint parameters** by examining the last Bézier segment on each side:
   - Extract center, angle, and radius from the Bézier control points
   - Handle the "open" case where the endpoint is at infinite curvature (straight track)

2. **Create a new `track` object** with type `T_CORNU` and attach extra data of type `extraDataCornu_t`.

3. **Connect to existing tracks** if provided — calls `ConnectTracks` to join the new clothoid to its neighbors at G2 continuity.

The function also handles error reporting via `InfoMessage` if the computation fails (e.g., degenerate geometry).

---

### `DrawCornuCurve(...)` — Render a Clothoid Track

```c
EXPORT void DrawCornuCurve(
    trkSeg_p first_trk,          // Left-hand connecting track
    trkSeg_p point1,             // Endpoint marker at left end
    int ep1Segs_cnt,
    trkSeg_p curveSegs,          // The clothoid itself (array of segments)
    int crvSegs_cnt,
    trkSeg_p point2,             // Endpoint marker at right end
    int ep2Segs_cnt,
    trkSeg_p second_trk,         // Right-hand connecting track
    trkSeg_p extend1_trk,        // Extension segment on left (if any)
    trkSeg_p extend2_trk,        // Extension segment on right (if any)
    trkSeg_p mids,               // Interior G2 anchor points
    int midSegs_cnt,
    wDrawColor color             // Color: black for normal, exceptionColor if radius < min
);
```

This draws the entire clothoid track assembly, including:
- The connecting tracks on either side (black)
- Red/blue endpoint markers showing which handle is selected
- Blue circular handles at each end (small inner circle + larger outer ring)
- Arrowheads indicating valid drag directions for radius/angle adjustment
- Interior G2 anchor points (blue circles that can be dragged to add interior knots)

The `color` parameter highlights the track in red if its minimum radius falls below the layout's configured minimum.

---

### `AdjustCornuCurve(...)` — Interactive Modification State Machine

This is the main **event handler** for clothoid modification mode (the "FLEX" track palette button). It implements a state machine that handles:
- Clicking on an endpoint to select its adjustment handle
- Dragging to adjust radius or angle
- Adding/removing interior G2 anchor points
- Extending endpoints with straight or circular segments

**States:**

| State | Description |
|-------|-------------|
| `NONE` | Not in clothoid mode; ignore events |
| `PICK_POINT` | Waiting for user to click on a handle (endpoint, midpoint, radius/angle ring) |
| `POINT_PICKED` | A handle is selected and dragging modifies its parameter |
| `TRACK_SELECTED` | In "convert" mode: selecting which tracks to convert |

**Event handling:**

- **C_START / C_DOWN** — Initialize state machine; display message about what the user should click on.
- **C_MOVE** — If in `PICK_POINT` or `POINT_PICKED`, draw preview geometry (red/blue circles, arrowheads) and update parameters. Checks for valid drag directions.
- **C_UP / C_OK** — Confirm or cancel parameter change; recompute the clothoid with new constraints.
- **C_CANCEL** — Reset state machine; discard pending changes.
- **C_TEXT (Backspace/Delete)** — Remove the most recently added interior G2 anchor point.

The function also handles **extending endpoints**: if the user holds Shift while clicking an endpoint that has no track connected, a temporary preview segment is drawn in blue, and dragging extends it along the tangent direction. This allows the user to effectively "grow" a clothoid outward from its connection points.

---

### `CmdCornu(...)` — Top-Level Create Command Handler

Entry point for placing a new clothoid track via the palette or hotbar:

```c
STATUS_T CmdCornu(wAction_t action, coOrd pos)
```

**Sequence of interaction:**

1. **C_START / C_DOWN** — "Left click — Place FlexTrack" message appears.
2. The first click places an endpoint handle (blue circle). The user drags it to position the endpoint and sets the tangent direction.
3. After confirming (C_UP), a second endpoint is placed similarly.
4. At this point both ends are fixed-radius, so a clothoid can be computed between them. If the geometry allows, the command returns `C_TERMINATE` with a new track created.
5. **Optional interior points** — The user may add G2 anchor points by clicking along the preview curve (between the two endpoints). These are stored in `midPoints`.
6. Final confirmation (Enter/Space) computes and creates the clothoid.

If the geometry is degenerate (endpoints too far apart, incompatible angles, etc.), an error message is shown and no track is created.

---

### `CmdConvertTo(...)` — Convert Existing Tracks to Clothoids

Allows the user to select one or more existing circular-arc tracks and convert them into clothoid transitions:

```c
STATUS_T CmdConvertTo(wAction_t action, coOrd pos)
```

**Workflow:**
1. User selects one or more track objects (currently limited to single-track layouts).
2. A dialog asks "Convert all selected tracks to Cornu tracks?"
3. For each selected track:
   - The track is decomposed into its Bézier/circular segments (`GetTracksFromCornuTrack`).
   - Each segment becomes a new clothoid or circular arc.
   - New `track` objects are created and connected via G2 continuity.
   - Old tracks are deleted (with undo support).
4. A summary message reports how many were converted, how many could not be converted (e.g., turnouts, fixed-radius tracks), etc.

---

### `CmdConvertFrom(...)` — Convert Clothoids Back to Fixed Tracks

The inverse operation: decompose a clothoid track back into its constituent circular arcs and straight segments:

```c
STATUS_T CmdConvertFrom(wAction_t action, coOrd pos)
```

**Workflow:**
1. User selects one or more clothoid (or Bézier) tracks.
2. Dialog asks for confirmation.
3. For each selected track, `GetTracksFromCornuTrack` decomposes it into basic segments:
   - Straight segments (`SEG_STRTRK`) — zero curvature
   - Circular arcs (`SEG_CRVTRK`) — constant non-zero curvature
4. A new "fixed" track object is created containing this sequence of segments.
5. Old clothoid objects are deleted.

This operation is useful when you want to lock in a design that was prototyped as a clothoid but ultimately uses standard circular arcs (e.g., for manufacturing or export).

---

## Utility Functions

### `CornuLength(coOrd pos[4], dynArr_t segs)` — Compute Total Length

```c
DIST_T CornuLength(coOrd pos[4], dynArr_t segs)
```

Traverses the segment array and sums arc lengths:
- For straight segments: Euclidean distance between endpoints.
- For circular arcs: `|radius| × Δangle` (absolute value handles both directions).
- For Bézier segments: delegates to recursive call on the underlying Bézier context.

---

### `CornuMinRadius(coOrd pos[4], dynArr_t segs)` — Find Tightest Curve

```c
DIST_T CornuMinRadius(coOrd pos[4], dynArr_t segs)
```

Returns the smallest absolute radius encountered along the curve. Used for validation: if the minimum radius is below a threshold (e.g., 20 meters), the track may be flagged as unsafe at design speed.

---

### `CornuMaxRateofChangeofCurvature(...)` — Compute Maximum dκ/ds

```c
DIST_T CornuMaxRateofChangeofCurvature(coOrd pos[4], dynArr_t segs, DIST_T *last_c)
```

Computes the maximum magnitude of **dκ/ds** (rate of change of curvature per unit arc length). This value is directly proportional to lateral jerk, so it's the primary comfort metric. The function also computes `*last_c`, which is passed recursively between segments to handle the transition at segment boundaries correctly.

A constant value along a pure clothoid segment equals `1/A²` (where `A` is the spiral parameter). Any deviation from this ideal indicates either:
- A circular arc section (`dκ/ds = 0`)
- A junction with discontinuous curvature (not possible in G2-continuous clothoids, but useful for validation)

---

### `CornuTotalWindingArc(...)` — Total Angular Change

```c
DIST_T CornuTotalWindingArc(coOrd pos[4], dynArr_t segs)
```

Sums the angular change across all segments. For a full 360° loop, this should equal (necessarily an integer multiple of) 2π radians. This is useful for validating closed-loop tracks.

---

### `CornuOffsetLength(...)` — Offset Along the Curve

```c
DIST_T CornuOffsetLength(dynArr_t segs, double offset)
```

Given a longitudinal offset along the curve, returns the corresponding arc length from the start. This is used when you want to specify "place a midpoint at 5 meters from the left endpoint."

---

## Design Decisions and Tradeoffs

### Bezier Approximation vs. Pure Parametric Representation

Raph Levien's library approximates the clothoid with Bézier curves of varying degrees. The choice of `spots` (approximate vs. exact) affects:
- **Performance** — approximation is faster than solving transcendental equations exactly at each step.
- **Accuracy** — for most railway applications (curvatures < 1/20 m⁻¹, lengths < a few hundred meters), the approximation error is negligible compared to track gauge tolerances.

### Why Not Use Pure Clothoid Parametric Equations?

The true clothoid has parametric equations involving Fresnel integrals:
```
x(s) = A × ∫₀ˢ cos(t²/2) dt   (scaled by A)
y(s) = A × ∫₀ˢ sin(t²/2) dt
```

These require numerical integration or special functions. For a CAD system that must support undo, serialization, and interactive editing, the Bézier approximation is far more practical:
- Segments can be stored as simple arrays of points/control polygons.
- Each segment knows its own radius, angle span, etc., making collision detection and pathfinding straightforward.
- The "open" endpoint case (straight track) maps naturally to a zero-radius arc, avoiding singularities in the Fresnel formulation.

### Interior G2 Anchor Points

The ability to add interior points (`midPoints`) allows users to constrain the clothoid at arbitrary positions along its length. This is useful for:
- Matching the curve to intermediate waypoints (e.g., aligning with a bridge centerline)
- Creating multi-segment transitions between more than two tracks
- Designing complex track geometries that don't fit a simple two-point definition

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CallCornuM` | Compute clothoid via Bezier approximation | endpoints, target radii/angles, interior points |
| `CallCornu0` | Same without interior G2 anchors | endpoints, centers, angles, radii |
| `CreateCornuFromPoints` | Build track object from computed segments | endpoints, open/fixed flags |
| `DrawCornuCurve` | Render the complete track assembly | segment arrays, endpoint markers |
| `AdjustCornuCurve` | Interactive modification state machine | action code, mouse position |
| `CmdCornu` | Top-level create command handler | — |
| `GetTracksFromCornuTrack` | Decompose clothoid into basic segments | input track object |
| `CornuLength` / `CornuMinRadius` / etc. | Geometric queries on computed curves | segment arrays, endpoints |

---

## Relation to Other Modules

- **`ccornu.h`** — declares the command constants (`cornuCmdNone`, etc.) and function prototypes.
- **`ccurve.c`** — provides `NewCurvedTrack`, `NewStraightTrack`, `ConnectTracks`, and drawing utilities.
- **`tcornu.h/c`** (in the cornu subdirectory) — contains Raph Levien's original Bezier approximation library for clothoids.
- **`spiroentrypoints.h`** — interfaces to the general-purpose spiral/Clothoid computation library.

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Create, edit, and decompose clothoid transition curves between railway tracks |
| **Domain** | Railway track design, ride comfort optimization (linear jerk), G2 continuity |
| **Key concept** | Curvature varies linearly with arc length: κ(s) = s/A² |
| **Main entry point** | `CmdCornu()` — palette button for free-form placement |
| **Computation engine** | Raph Levien's Bezier approximation of the Cornu spiral |
| **Modification mode** | Interactive handles at endpoints and interior G2 anchor points |
