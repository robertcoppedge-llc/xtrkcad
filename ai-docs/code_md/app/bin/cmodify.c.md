# cmodify.c — Track Modification Operations

## Overview

`cmodify.c` implements the **Modify** command, which provides a unified interface for several track editing operations:
- **Trimming/shortening** a track from either end (the "basic" modify operation)
- **Extending** a track by appending straight or curved segments with optional easement transitions
- **Modifying Bézier control points** (delegates to `cbezier.c`)
- **Modifying clothoid endpoints** (delegates to `ccornu.c`)
- **Modifying Draw objects** (basic shape editing)

The command uses a state machine driven by mouse events (`C_START`, `C_DOWN`, `C_MOVE`, `C_UP`, etc.) and keyboard modifiers (`Shift`, `Ctrl`, `Alt`). Undo support is integrated via `UndoStart()`/`UndoModify()`/`UndoEnd()`.

---

## Data Structures

### `Dex` — Global Modification State

```c
static struct {
    track_p Trk;                       // The currently selected track being modified
    trackParams_t params;              // Extended parameters for extend mode (curve type, radius, length)
    coOrd pos00, pos00x, pos01;        // Points: original endpoint, preview position, drag start
    ANGLE_T angle;                     // Current tangent angle at the modified end
    curveData_t curveData;             // Computed data for curves being extended
    easementData_t jointD;             // Easement computation results (for curved extensions)
    DIST_T r1;                         // Radius of a circular arc extension candidate
    BOOL_T valid;                      // TRUE if the current preview is geometrically valid
    BOOL_T first;                      // Flag for "first move after initial click" in extend mode
} Dex;
```

`Dex.Trk` holds a pointer to the track currently being modified. All modification state (preview points, computed curves, easement data) is accumulated here across multiple mouse events until the user confirms or cancels with `C_UP`.

### `tempSegs_da` — Temporary Segment Array

A dynamically allocated array of `trkSeg_t` used to hold preview segments for the "Extend" operation. When the user drags in extend mode, preview segments are appended here and drawn (in black) as a visual guide before confirmation.

### `anchors_da` — Anchor Points

```c
static dynArr_t anchors_da;  // Array of trkSeg_t used to draw small circular handles at endpoints
#define anchors(N) DYNARR_N(trkSeg_t,anchors_da,N)
```

Each element is a tiny circular arc segment (radius ≈ 0.15 × scale) drawn in blue. Anchors are placed around an endpoint to allow the user to drag them and adjust:
- **Radius** — by dragging along the radial direction
- **Angle / position** — by dragging tangentially

Anchors appear for both Bézier control points (`SEG_FILCRCL` with a radius handle) and clothoid endpoints.

### `log_modify` — Logging Channel

```c
static int log_modify;  // Global log index used by LOG(...) macros
```

The `LOG(log_modify, level, "...")` macro emits messages to the application's logging system (e.g., for debugging or audit trails).

---

## Core Functions

### `CreateEndAnchor(coOrd p, wBool_t lock)` — Create a Circular Handle at an Endpoint

Creates a small circular anchor segment centered at `p`. Used around clothoid endpoints that can be resized. The handle is drawn as a tiny blue circle (radius ≈ 0.15 × scale) with zero line width so it only appears when selected or hovered.

### `CreateCornuAnchor(coOrd p, wBool_t lock)` — Create Two Anchors for a Clothoid Endpoint

Creates **two** small circular anchors around a clothoid endpoint:
- **Inner anchor:** smaller circle used to adjust the radius of the end segment (straight vs. curved transition)
- **Outer anchor:** larger ring used to adjust the tangent angle at the end

The two rings are drawn concentrically; dragging along one or the other changes different parameters. The `lock` flag controls whether the handle is "locked" (fixed while dragging another point) or free.

### `CreateRadiusAnchor(coOrd p, ANGLE_T a, BOOL_T bi)` — Create an Arrow Handle for Radius Adjustment

Creates five polyline segments forming an arrow-shaped handle around an endpoint's radial adjustment ring:
- The center circle indicates the current radius
- An outer ring allows changing from straight (`r=0`) to curved
- The arrow shape visually encodes "pull outward = increase radius" and "push inward = decrease radius"

The `bi` flag controls whether the inner or outer ring of the handle is highlighted.

---

### `ModifyBezier(wAction_t action, coOrd pos)` — Delegate Bézier Editing to cbezier.c

This function acts as a thin wrapper around `CmdBezModify()` from `cbezier.c`. It:
1. Sets `trackGauge` based on the selected track (or zero if not a valid track)
2. Passes the current mouse action and position to the Bézier modifier state machine
3. On termination (`C_TERMINATE`), resets global flags and clears `Dex.Trk`

Bézier modification allows moving control points of a spline-defined track segment, effectively reshaping that portion of the layout interactively.

---

### `ModifyCornu(wAction_t action, coOrd pos)` — Delegate Clothoid Editing to ccornu.c

Similar wrapper around `CmdCornuModify()` from `ccornu.c`. The clothoid modifier state machine handles:
- Selecting and dragging endpoint adjustment anchors (radius/angle rings)
- Adding/removing interior G2 anchor points along the curve
- Validating that the resulting geometry is geometrically feasible

---

### `ModifyDraw(wAction_t action, coOrd pos)` — Modify a Draw Object

A generic modifier for objects of type `DRAW`. These are free-form polygonal shapes (not tracks) that can be used as obstacles or decorative elements. The function handles:
- **C_START/C_DOWN** — select the object and show bounding box hilite
- **C_MOVE** — drag vertices (with preview geometry shown in black)
- **C_UP / C_OK** — commit changes; call `ModifyTrack()` which updates the track's segment array
- **C_CANCEL** — discard pending edits

The function also handles keyboard shortcuts: typing `'0'` or `'o'` enters a "select mode" (likely for multi-object selection), `'c'` centers the view, `'s'` zooms to extents.

---

### `ModifyTrack(track_p trk, wAction_t action, coOrd pos)` — The Generic Track Modifier

This is the central dispatcher for basic track modification operations (trimming). It handles:
- **C_START / C_DOWN** — select a track and show message "Select a track to modify..."
- **wActionMove** — snap position to nearest point on a selected track, draw preview hilite rectangles showing where trimming would occur
- **C_MOVE** — continue dragging the preview; the preview rectangle moves along the track
- **C_UP / C_OK** — commit: compute how much to trim, adjust `Dex.Trk`'s endpoint coordinates, update undo history via `UndoModify()`, call `UndoEnd()`
- **C_TERMINATE** — discard pending changes

The trimming logic (not fully shown in the excerpt) computes a new endpoint coordinate and updates the track object accordingly.

---

### `CmdModify(wAction_t action, coOrd pos)` — Main Entry Point

This is the top-level state machine that routes to sub-handlers depending on:
- Whether the user has selected a track (`Dex.Trk != NULL`)
- Which modifier mode is active (Bezier, Cornu, Draw, Extend, Ruler, Protractor)
- The current mouse action (`C_START`, `C_DOWN`, `C_MOVE`, etc.)

**Key modes:**

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Extend track** (`Ctrl+RightClick`) | Right-click on a track with Ctrl held | Enters extend mode: user drags to define an endpoint, chooses straight or curved extension, optionally adds easement transition. Preview segments are drawn in black. On release, the new segment is inserted and connected via `ConnectTracks()` or `JoinTracks()` (if easement). |
| **Radius handle** (`Shift+LeftClick` on a track with open end) | Left-click while Shift held | Creates radius/angle adjustment anchors around an endpoint. Dragging adjusts the curve's radius at that end. |
| **Turnout extend** (`Ctrl+RightClick` on turnout) | Right-click on a turnout with Ctrl held | Extends one of the turnout's open ends, optionally inserting a curved transition arc. |
| **Bezier control point edit** | Left-click on Bézier track | Delegates to `ModifyBezier()`. Control points are moved interactively. |
| **Clothoid endpoint adjustment** | Left-click on clothoid track | Delegates to `ModifyCornu()`. Endpoint handles allow radius/angle tweaking. |

**Extend mode algorithm (detailed):**

1. User right-clicks a track with Ctrl held → enter extend mode (`Dex.first = TRUE`).
2. The endpoint coordinate is shown as a hilite rectangle. Dragging moves it along the tangent line at ±90° offsets from the current tangent direction.
3. If the drag reaches within `minLength` of the original endpoint, abort (can't extend to zero length).
4. On release (`C_UP`): compute the new track segment:
   - **Straight extension:** create a straight segment between old and new endpoint.
   - **Curved extension:** compute center and radius from two endpoints and tangent direction; generate a circular arc via `PlotCurve()` (calls into `cbezier.c`).
5. Compute easement data (`jointD`) if needed — this determines where to insert a transition curve between the old track and the new segment.
6. If there is no existing easement at that joint, simply connect with `ConnectTracks()`.
7. If an easement exists, use `JoinTracks()` which inserts the transition geometry.

**Easement computation:** When extending a curved track where an easement (clothoid) already exists at that end, the system computes how much of the easement must be "consumed" to connect smoothly to the new segment. The easement length is computed from the change in curvature (`dκ/ds`) and integrated along the arc.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CreateEndAnchor` | Create a small circular handle for endpoint radius adjustment | position, lock flag |
| `CreateCornuAnchor` | Create two concentric handles (radius + angle) at a clothoid endpoint | position, lock flag |
| `CreateRadiusAnchor` | Create arrow-shaped radial handle with inner/outer rings | position, angle, inner flag |
| `ModifyBezier` | Delegate to Bézier editor (`cbezier.c`) | track pointer, action code, mouse pos |
| `ModifyCornu` | Delegate to clothoid editor (`ccornu.c`) | track pointer, action code, mouse pos |
| `ModifyDraw` | Edit Draw object vertices | action code, position |
| `ModifyTrack` | Generic trim/modify dispatcher | track pointer, action code, position |
| `CmdModify` | Main state machine for all modification operations | action code, mouse position |

---

## Design Decisions & Tradeoffs

### Why a Global State Structure (`Dex`) Instead of Passing Context?

The modify command is invoked repeatedly (every mouse move event) while the user drags. Passing a large struct by value on every call would be inefficient and error-prone. Instead, `Dex` is a globally scoped static structure that persists across invocations. This is typical in GUI applications where event-driven code cannot easily maintain per-operation state on the stack (due to nested calls, interrupts from the windowing library, etc.).

### Why Separate Preview from Commit?

The preview geometry (`tempSegs_da`) is drawn but **not** committed until `C_UP` or `C_OK`. This allows the user to experiment with different extension lengths and types without immediately altering the layout. Undo support wraps the entire commit transaction, so if the user backs out before confirming, no changes are made.

### Why Use Small Circular Anchors Instead of Larger Handles?

Small anchors (radius ≈ 0.15 × scale) reduce visual clutter. They only become visible when the cursor hovers over them or is clicked on, keeping the workspace clean. The arrow-shaped radius handle uses a different shape because it needs to convey directionality ("pull outward") more clearly than a plain circle does.

### Why Delegate to Submodules?

The modify command delegates Bézier and clothoid editing to dedicated modules (`cbezier.c`, `ccornu.c`). This avoids duplicating complex state machines in one monolithic file. The delegate functions (`ModifyBezier`, `ModifyCornu`) simply set up their local sub-state (e.g., which control point is selected) and then pass the action through a switch statement back to the submodule's handler.

### Why Compute Easement Data Inline?

The easement computation is done inline in `CmdModify` because it depends on global state (`trackGauge`, `minLength`) and must integrate with the undo/redo system. The easement data structure (`easementData_t`) stores intermediate results of a geometric solver that determines how long a clothoid transition must be to connect two arcs smoothly. This is computed only when needed (i.e., when extending a curved track at an existing easement) to avoid unnecessary computation.

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide interactive modification of tracks: trim endpoints, extend with straight/curved segments, adjust Bézier control points, adjust clothoid parameters |
| **Domain** | Interactive CAD editing; track geometry manipulation; undo-able state transitions |
| **Key concept** | A global state structure (`Dex`) holds all pending modifications until the user confirms or cancels; preview geometry is drawn but not committed until `C_UP`/`C_OK` |
| **Main entry point** | `CmdModify(wAction_t, coOrd)` — a large switch statement that dispatches based on active mode and mouse action |
