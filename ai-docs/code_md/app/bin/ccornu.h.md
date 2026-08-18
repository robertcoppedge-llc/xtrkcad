# ccornu.h — Cornu Spiral Command Interface

## Overview

`ccornu.h` is a header file that declares the API for **Cornu spiral** (Euler spiral) commands in XTrkCAD. The Cornu spiral is used as a transition curve between straight track and circular arcs, providing a smooth change in curvature (from 0 on a straight line to a constant value on a circle).

---

## Data Types & Constants

### `cornuMessageProc` — Message Callback Type

```c
typedef void (*cornuMessageProc)( const char *, ... );
```

A function pointer type for message callbacks. Used by the Cornu command system to report progress or status messages to the user.

---

### Command Mode Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `cornuCmdNone` | 0 | No command mode (default) |
| `cornuJoinTrack` | 1 | Join an existing track with a Cornu spiral transition |
| `cornuCmdCreateTrack` | 2 | Create a new track using Cornu spirals for transitions between curves and straights |
| `cornuCmdHotBar` | 3 | Enable the hot bar tool (one-click insertion) |

These are used to determine which mode of operation the Cornu command is in. For example, `cornuJoinTrack` means the user has clicked on an existing track segment and wants to add a Cornu spiral at its endpoint; `cornuCmdCreateTrack` means the user is drawing from scratch using the Cornu tool.

---

## Function Declarations

### `STATUS_T CmdCornu(wAction_t action, coOrd pos)`

The main entry point for the Cornu command system. It handles single-click interactions:

- **Parameters:**
  - `action`: The action performed (e.g., mouse click) — see `wAction_t` type in the widget library
  - `pos`: A coordinate pair (`coOrd`) representing the position of the click or cursor
- **Returns:** A status code indicating whether the action was accepted

This function is invoked by the hot bar tool when the user clicks to place a Cornu spiral segment.

---

### `BOOL_T CallCornu0(coOrd pos[2], coOrd center[2], ANGLE_T angle[2], DIST_T radius[2], dynArr_t *array_p, BOOL_T spots)`

Calls a 4-point Cornu spiral computation given the positions of two endpoints and two control points (or centers/radii for arc transitions). It populates a `dynArr_t` array with intermediate point data along the spiral.

- **Parameters:**
  - `pos[2]`: Two endpoint coordinates defining the chord between the straight and the curve
  - `center[2]`: Centers of the two circular arcs (or one center and one offset for transition)
  - `angle[2]`: Deflection angles at each end (0° on the straight, full deflection on the circle)
  - `radius[2]`: Radii of the two connecting curves (infinity or zero-radius for a straight line)
  - `array_p`: Pointer to a dynamic array where intermediate points will be stored
  - `spots`: If true, also compute and store specific "spot" locations along the spiral

- **Returns:** TRUE if successful, FALSE otherwise

---

### `DIST_T CornuMinRadius(coOrd pos[4], dynArr_t segs)`

Computes the minimum radius of curvature encountered by a Cornu spiral defined by four control points.

- **Parameters:**
  - `pos[4]`: Four coordinate pairs defining the endpoints and intermediate control points
  - `segs`: Dynamic array of segment data (likely containing intermediate computed values)
- **Returns:** The minimum radius value found along the spiral path

---

### `DIST_T CornuMaxRateofChangeofCurvature(coOrd pos[4], dynArr_t segs, DIST_T *last_c)`

Computes the maximum rate of change of curvature (i.e., the maximum third derivative of position with respect to arc length) along a Cornu spiral. This is a quality metric: lower values indicate smoother transitions.

- **Parameters:**
  - `pos[4]`: Four control point coordinates
  - `segs`: Dynamic array of segment data
  - `last_c`: Pointer to store the curvature at the last point (the circular arc radius)
- **Returns:** The maximum rate-of-change value

---

### `DIST_T CornuLength(coOrd pos[4], dynArr_t segs)`

Computes the total arc length of a Cornu spiral defined by four control points.

- **Parameters:**
  - `pos[4]`: Four coordinate pairs
  - `segs`: Dynamic array of segment data
- **Returns:** The computed arc length

---

### `DIST_T CornuOffsetLength(dynArr_t segs, double offset)`

Computes the arc length along a Cornu spiral at a given fractional offset (0.0 = start, 1.0 = end). This is used for sampling points along the curve or for interpolation.

- **Parameters:**
  - `segs`: Dynamic array of segment data
  - `offset`: Fractional distance along the curve (0–1)
- **Returns:** The arc length corresponding to that offset

---

### `DIST_T CornuTotalWindingArc(coOrd pos[4], dynArr_t segs)`

Computes the total winding angle (total change in heading direction) along a Cornu spiral. For a true transition from straight to circle, this equals the deflection angle of the circular arc.

- **Parameters:**
  - `pos[4]`: Four control point coordinates
  - `segs`: Dynamic array of segment data
- **Returns:** Total winding angle in radians (or degrees depending on coordinate system)

---

### `STATUS_T CmdCornuModify(track_p trk, wAction_t action, coOrd pos, DIST_T trackG)`

Modifies an existing track to include Cornu spiral transitions at specified endpoints.

- **Parameters:**
  - `trk`: Pointer to the track structure being modified
  - `action`: The type of modification (insert, replace, etc.)
  - `pos`: Position where the modification is made
  - `trackG`: Gauge width of the track
- **Returns:** Status code

---

### `void InitCmdCornu(wMenu_p menu)`

Initializes the Cornu command system and registers it with a given menu. This sets up internal data structures, message handlers, etc.

- **Parameters:**
  - `menu`: Pointer to a widget menu (GTK or equivalent) where the Cornu tool will be placed

---

### `void AddHotBarCornu(void)`

Adds a hot bar button that provides one-click access to the Cornu spiral creation/joining mode. This is a convenience feature for frequent users who want to quickly insert transition curves without navigating through menus.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CmdCornu(action, pos)` | Main entry point; handles mouse clicks during Cornu drawing mode | action code, cursor position |
| `CallCornu0()` | Computes a 4-point Cornu spiral and fills an array with points | endpoints, centers/angles/radii, output array |
| `CornuMinRadius()` | Finds the minimum radius along a computed spiral | four control points, segment array |
| `CornuMaxRateofChangeOfCurvature()` | Quality metric — maximum curvature rate (smoothness check) | four control points, segment array |
| `CornuLength()` | Computes total arc length of the spiral | four control points, segment array |
| `CornuOffsetLength()` | Samples a point at a given fractional offset along the curve | segment array, 0–1 offset |
| `CmdCornuModify(trk, action, pos, trackG)` | Inserts Cornu transitions into an existing track | track pointer, action, position, gauge |
| `InitCmdCornu(menu)` | Registers the Cornu command with a menu system | widget menu handle |
| `AddHotBarCornu()` | Adds a toolbar/hot-bar button for quick access | none |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide an interface and utility functions for computing, drawing, and modifying track geometry using Cornu (Euler) spirals as transition curves between straights and circular arcs. |
| **Domain** | Geometric computation: the Cornu spiral is a special curve whose curvature changes linearly with arc length. It is ideal for transitioning from zero curvature (straight) to constant non-zero curvature (circle). The functions here handle coordinate geometry, numerical integration of curvature along the path, and interaction with the track data structure. |
| **Key concept** | A Cornu spiral is defined parametrically by `x(s) = ∫₀ˢ Ci(t) dt`, `y(s) = ∫₀ˢ Si(t) dt` where `Ci` and `Si` are cosine and sine integrals of the accumulated curvature. The "4-point" interface abstracts away the parameterization and instead takes four points (start, end, and two intermediate control points that define the chord and deflection angles). |
| **Main entry points** | `CmdCornu()` — called from the hot bar tool; `AddHotBarCornu()` — registers the toolbar button |
