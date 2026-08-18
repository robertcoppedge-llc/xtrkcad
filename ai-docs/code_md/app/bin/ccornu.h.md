# ccornu.h — Cornu Spiral (Clothoid) Utilities Header

## Overview

`ccornu.h` provides the **public interface** for clothoid (Cornu spiral) computations used in XTrkCAD. Clothoids are essential in railway engineering because they provide a **linear rate of change of curvature**, which ensures a smooth and predictable transition from straight track to curved track.

The header declares:
- Command constants and type definitions
- Function prototypes for clothoid parameter calculations
- Utility functions for join-track operations
- Hotbar integration support

---

## Key Concept: The Clothoid (Cornu Spiral)

A **clothoid** (also called an Euler spiral or Cornu spiral) is a parametric curve whose curvature varies linearly with arc length:

```
κ(s) = s / A²
```

where `A` is the *spiral parameter* and `s` is the arc length from the start of the spiral. This means that at any point along the clothoid, the radius of curvature is inversely proportional to the distance traveled since the curve began:

```
R(s) = A² / s
```

This property makes the clothoid ideal for track transitions because a train traveling at constant speed experiences a **linear increase in lateral acceleration**, which is what human passengers perceive as comfortable and predictable.

---

## Command Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `cornuCmdNone` | 0 | No command / default state |
| `cornuJoinTrack` | 1 | Join existing track with a clothoid transition |
| `cornuCmdCreateTrack` | 2 | Create a new track using clothoids |
| `cornuCmdHotBar` | 3 | Hotbar UI entry (likely for a palette button) |

These constants are used to control which mode the Cornu command operates in. The hotbar integration allows users to access this tool from the interface without navigating through menus.

---

## Function Prototypes

### `STATUS_T CmdCornu(wAction_t action, coOrd pos)`

The main entry point for the Cornu command handler. It processes mouse actions (down, up, move) and displays interactive preview geometry as the user positions a transition curve along an existing track segment.

**Parameters:**
- `action` — The current event (`C_DOWN`, `C_MOVE`, `C_UP`, etc.) from the command system
- `pos` — Mouse position in layout coordinates

**Returns:**
A status code indicating whether to continue or terminate the command session. Returns `C_CONTINUE` while previewing, `C_TERMINATE` when the user confirms placement.

---

### `STATUS_T CmdCornuModify(track_p trk, wAction_t action, coOrd pos, DIST_T trackG)`

Handles modifications to an existing track object that is being converted or adjusted using clothoid transitions. The `trackG` parameter likely specifies the target gauge (track width) for sizing computations.

---

### `void InitCmdCornu(wMenu_p menu)`

Registers the Cornu command with a menu system. Called during initialization to add an entry point into the application's menu hierarchy.

---

### `void AddHotBarCornu(void)`

Adds an icon or button to the hotbar (toolbar) for quick access to the clothoid tools. This is typical of XTrkCAD's design pattern where complex geometric operations are exposed as single-click palette entries.

---

## Utility Functions

These functions operate on arrays of coordinate points (`coOrd pos[4]`) and a dynamic array of track segments (`dynArr_t`). They compute geometric properties needed for clothoid placement:

### `BOOL_T CallCornu0(coOrd pos[2], coOrd center[2], ANGLE_T angle[2], DIST_T radius[2], dynArr_t *array_p, BOOL_T spots)`

**Purpose:** Calls a lower-level Cornu computation routine.

**Parameters:**
- `pos[2]` — Two endpoint positions (likely start and end of the transition segment)
- `center[2]` — Centers or reference points for each end
- `angle[2]` — Orientation angles at each end
- `radius[2]` — Radius values (possibly infinite/straight track at one end, finite curve radius at the other)
- `array_p` — Output: dynamic array to store computed segment data
- `spots` — A boolean flag whose exact meaning is unclear from this declaration alone; possibly controls whether only endpoint "spot" calculations are performed vs. full discretization

**Returns:** `BOOL_T` indicating success or failure of the computation.

---

### `DIST_T CornuMinRadius(coOrd pos[4], dynArr_t segs)`

Computes the **minimum radius of curvature** encountered along a sequence of segments that includes clothoid transitions. This is useful for validating that no transition exceeds a safe limit (e.g., a spiral segment doesn't curve faster than 10 m/s² centripetal acceleration at design speed).

**Parameters:**
- `pos[4]` — Four points: likely the two endpoints of a track segment plus additional reference points defining the geometry
- `segs` — A dynamic array of track segments (including the clothoid portion)

**Returns:** The smallest radius found along the entire path. Returns infinity or some sentinel value if no curvature is present (i.e., the entire segment is straight).

---

### `DIST_T CornuMaxRateofChangeofCurvature(coOrd pos[4], dynArr_t segs, DIST_T *last_c)`

Computes the **maximum rate of change of curvature** (dκ/ds) along a sequence of segments. This metric directly corresponds to lateral jerk — the third derivative of position with respect to time — which is the primary comfort criterion for passengers on a train. A lower value means a smoother ride.

**Parameters:**
- `pos[4]` — Four defining points (likely start, end, and two intermediate reference points)
- `segs` — Array of track segments forming the transition path
- `last_c` — Output pointer: the curvature at the final point is stored here for continuity checking

**Returns:** The maximum value of |dκ/ds| encountered. This should ideally be a constant value along the clothoid portion (by definition), so any deviation indicates an error in segment construction.

---

### `DIST_T CornuLength(coOrd pos[4], dynArr_t segs)`

Computes the total arc length of all segments provided, including both straight sections and clothoid portions. This is useful for:
- Validating that a transition fits within available track space
- Computing travel time through a transition at constant speed
- Ensuring continuity with adjacent track segments

**Returns:** Total path length in consistent distance units (millimeters by default).

---

### `DIST_T CornuOffsetLength(dynArr_t segs, double offset)`

Computes the **longitudinal offset** of a clothoid curve from its chord. When a clothoid is used as a transition between two straight tracks, it sags away from the chord connecting the endpoints. This function computes that sag distance given an offset parameter (possibly a target minimum radius or some other design constraint).

**Parameters:**
- `segs` — Array of segments forming the transition curve
- `offset` — A scalar offset value; possibly represents how far the curve should deviate from the chord, or it could be a target curvature rate

**Returns:** The longitudinal distance along the track corresponding to that offset.

---

### `DIST_T CornuTotalWindingArc(coOrd pos[4], dynArr_t segs)`

Computes the total **winding angle** (total change in heading) accumulated along all segments. For a clothoid transition, this equals the difference between the angles at its two endpoints. This is useful for:
- Verifying that the sum of all winding arcs around a loop matches 360° (or an integer multiple thereof)
- Computing total direction change through a series of transitions and curves

**Returns:** The net angular change in radians or degrees depending on unit conventions used internally.

---

## Design Notes

The functions are written to operate on:
1. **A small set of reference points** (`pos[4]`) — these define the geometry of a transition segment (endpoints and possibly intermediate control points)
2. **A dynamic array of segments** (`dynArr_t segs`) — this holds the precomputed geometric data for each discrete piece of track, including polygonal approximations or parametric descriptions

This design suggests that:
- Clothoids are not stored as pure mathematical functions but rather as a sequence of linear or quadratic segments that approximate the curve. The dynamic array likely contains pointers to `trkSeg` structures with precomputed vertex coordinates.
- The "four points" (`pos[4]`) may represent: start point, end point, and two additional points defining curvature direction or intermediate control geometry for validation.

The use of `STATUS_T` (a typedef for a status code) suggests that some functions can fail under certain conditions — perhaps when the input geometry is degenerate (e.g., all four points collinear) or when computed values exceed safe limits.

---

## Relation to Other Modules

This header lives in `app/bin/` and includes `"common.h"`, indicating it depends on:
- **Command system** (`command.h`) — for the command handler interface
- **Common utilities** (coordinate types, drawing colors, etc.)

It likely interacts with:
- **Track data structures** — via `track_p` handles passed to `CmdCornuModify`
- **Hotbar UI framework** — through `AddHotBarCornu()` which integrates into the command palette
- **Draw system** — `CmdCornu` will call drawing routines to display preview geometry

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide interface for clothoid transition curve computations |
| **Domain** | Railway track design, geometric transitions, ride comfort optimization |
| **Key concept** | Linear rate of change of curvature (κ = s/A²) ensures passenger comfort |
| **Main entry point** | `CmdCornu()` — handles interactive placement via the command system |
| **Geometric utilities** | Radius computation, max jerk calculation, arc length, winding angle |
