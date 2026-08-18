# cselect.c — Selection System & Raycasting

## Overview

`cselect.c` implements XTrkCAD's **selection and raycasting system**. It provides:
- Hit-testing from screen coordinates (mouse position) to track segments
- Multi-object selection via shift-click accumulation
- Track moving/rotating operations with undo support
- Auto-selection of connected end-cornus when a track is selected
- Layer-based highlighting for visibility management

The file implements a **raycasting approach** — drawing an invisible crosshair at the cursor position and casting rays in all directions to detect which object lies closest. This is more robust than simple point-in-polygon tests because it handles overlapping geometry correctly (the nearest object wins).

---

## Data Structures

### `trackSelect_t` — Per-Track Selection State

```c
typedef struct {
    track_p trk;          // Pointer to the selected track
    BOOL_T selected;      // Whether this track is currently selected
    BOOL_T modified;      // Flag: has user edited a selected track?
} trackSelect_t;
```

This structure tracks selection status per-track. The global state is maintained in an array indexed by track index, allowing efficient O(1) lookup and update when the user clicks on objects.

---

### `tlist_da` — Selected Track List

```c
typedef struct dynArr_ {
    track_p *data;       // Pointer to dynamically allocated array
    long cnt;            // Number of selected tracks stored
} dynArr_t;
```

This is a **dynamic array** (from the C library) that holds pointers to all currently selected tracks. It grows/shrinks as objects are added or removed from selection. Used by:
- `DoSelectedTracks()` — iterates over all selected tracks
- `GetMovedTracks()` — accumulates tracks visible in viewport for moving
- `RemoveSelectedTrack()` — removes a track when it's deselected

---

### `moveD` / `tempSegsD` — Temporary Drawing Command Structures

```c
static drawCmd_t tempSegsD = {
    NULL, &tempSegDrawFuncs, 0, 1, 0.0,
    {0.0, 0.0}, {0.0, 0.0}, Pix2CoOrd, CoOrd2Pix
};
```

These structures hold temporary drawing state used when:
- Drawing selected tracks to a "scratchpad" (`tempSegsD`)
- Redrawing only the moved portion of the screen during move operations

The `drawCmd_t` struct is a context that holds the current viewport origin, scale, DPI, and pointer to active draw functions (e.g., which color to use). It allows `DrawTrack()` to render directly without needing the global `mainD`.

---

### `moveD_hi`, `moveD_lo` — Scissor Rect for Move Operations

```c
static coOrd moveD_hi, moveD_lo;
```

These define a rectangular region (in screen coordinates) that bounds all selected tracks. During move/rotate operations, only this region is redrawn instead of the entire viewport — a classic **dirty rectangle optimization**. The values are computed by:
1. Calling `GetBoundingBox()` on each selected track
2. Computing the union AABB across all tracks
3. Expanding outward by one pixel to avoid edge artifacts

---

### `auto_select_da` — Auto-Selected End Cornus

```c
static dynArr_t auto_select_da;
```

When a user selects a straight track, any end-cornus attached at its endpoints are automatically added to this array and then drawn. When the selection is cleared (e.g., clicking elsewhere), `RemoveEndCornus()` iterates over this list, deselects those cornus, and removes them from their own selection arrays. This prevents visual artifacts where a user selects track A but sees highlights on adjacent cornus that are not part of the selection.

---

### `getSelectedBoundsLo`, `getSelectedBoundsHi` — Selection Bounding Box

```c
static coOrd getSelectedBoundsLo, getSelectedBoundsHi;
static long getSelectedBoundsCount;
```

These accumulate a bounding box around all selected tracks. The function `GetSelectedBounds()` walks the track list and computes:
- Minimum x/y across all selected tracks → `lo`
- Maximum x/y across all selected tracks → `hi`

This is used for operations that need to know the extent of a selection, such as computing a tight crop region or applying effects like "select entire layer."

---

## Core Functions

### `FindEndIntersection(...)` — Find Intersection Between Two End Points

**Purpose:** Given two endpoints (one from each of two selected tracks), move one track so that its endpoint coincides with the other's. Used by the "Move to Join" command.

```c
wBool_t FindEndIntersection(
    coOrd base,       // Delta translation vector (base = target - source)
    coOrd orig,       // Rotation origin (zero if no rotation needed)
    ANGLE_T angle,    // Angle to rotate by (difference in tangent angles + 180°)
    track_p *t1,     // Output: first endpoint's track
    EPINX_T *ep1,    // Output: index of endpoint on t1
    track_p *t2,     // Output: second endpoint's track
    EPINX_T *ep2)    // Output: index of endpoint on t2
```

**Algorithm:**
1. Iterate over all selected tracks (`*tlist_da`) — exclude cornus and already-selected end-cornus (they belong to the primary selection).
2. For each track, iterate over its endpoints.
3. Compute where that endpoint would be after applying `base` translation and `angle` rotation.
4. Call `OnTrackIgnore()` with that point to find which track it lies on.
   - The `-1` flag tells `OnTrack()` not to include turnouts, switches, or draw objects — only rails/curves.
5. If the endpoint falls on a track of the **same gauge**, try to pick an unconnected endpoint:
   - Call `PickUnconnectedEndPointSilent(...)` to get the index.
   - If ≥ 0 (found), check if it's close enough (`FindDistance(...)`) and return success.
6. If no "unconnected" endpoint is found, fall back to `PickEndPoint()` which finds *any* endpoint within tolerance — this handles junctions where multiple tracks meet at one point.

**Why two-pointer output?** The function returns both endpoints because after moving one track to join another, the caller needs to know *which* endpoint on *which* track was matched. This is passed to `MoveToJoin()` which then calls `ConnectTracks()` to weld them together.

---

### `DoSelectedTracks(callback)` — Iterate Over All Selected Tracks

```c
static void DoSelectedTracks( BOOL_T (*callback)(track_p, void*), void *vp )
{
    track_p trk;
    for (trk = NULL; TrackIterate(&trk); ) {
        if (GetTrkSelected(trk)) {
            callback(trk, vp);
        }
    }
}
```

This is a **generic dispatcher** that walks the global track list and applies a boolean callback function only to selected tracks. It's used by many functions: `SelectDelete()`, `ClearElevations()`, `AddElevations()`, `GetBoundsDoIt()`, etc. The pattern allows sharing common iteration logic instead of duplicating loops.

---

### `RemoveSelectedTrack(track_p trk)` — Deselect a Single Track

```c
static BOOL_T RemoveSelectedTrack(track_p trk) {
    for (int i=0; i<tlist_da.cnt; i++) {
        if (DYNARR_N(track_p, tlist_da, i) == trk) {
            // Shift remaining elements left: move element at j+1 to position j
            for (int j=i; j < tlist_da.cnt-1; j++) {
                DYNARR_LAST(track_p, tlist_da)[j] = DYNARR_LAST(track_p, tlist_da)[j+1];
            }
            tlist_da.cnt--;
            return TRUE;
        }
    }
    return FALSE;  // Track not found in selection list
}
```

**How it works:** Finds the track pointer in the dynamic array and performs an **in-place shift**: all elements after the match are shifted left by one slot, then the count is decremented. The removed element gets overwritten — no memory leak because the array is managed by the C library.

This function is used when:
- A user clicks on a track to toggle its selection status (deselect)
- A selected track is deleted or moved off-screen
- An end-cornus is deselected after joining tracks together

---

### `AddSelectedTrack(track_p trk)` — Add Track to Selection List

```c
static BOOL_T AddSelectedTrack( track_p trk, BOOL_T unused ) {
    DYNARR_APPEND(track_p, tlist_da, 10);
    DYNARR_LAST(track_p, tlist_da) = trk;
    return TRUE;
}
```

When a user clicks on an unselected track (or toggles selection), this adds it to `tlist_da`. The array is pre-allocated with capacity for ~10 tracks at a time and grows as needed.

---

### `GetMovedTracks(BOOL_T undraw)` — Accumulate Visible Selected Tracks

This function is called after the user drags or rotates selected tracks. It rebuilds `tlist_da` by:
1. Calling `AddSelectedTrack()` for each selected track that intersects the viewport (`moveD`).
2. Adding any end-cornus attached to those endpoints (via `AddEndCornus()`).

The `undraw` flag, if true, first erases the currently-drawn selection hilites so the new positions are drawn fresh underneath.

**Why exclude cornus by default?** Cornus that are "fixed" at an endpoint should move along with their parent track — they don't need to be re-added. Only *newly* created end-cornus (from junctions) or detached free cornus belong in the list. The `AddEndCornus()` call after handles those cases.

---

### `MoveTracks(...)` — Core Move/Rotate Logic

This is the heart of the move/rotate operation. It does **two passes** over selected tracks:

**Pass 1 (non-cornu first):**
- Calls `UndoModify(trk)` to mark the track for undo.
- Applies `MoveTrack()` if a base translation vector exists.
- Applies `RotateTrack()` if a rotation origin/angle is specified.
- For each endpoint, finds any connected unselected track and calls `DisconnectTracks()`. This handles cases where moving a track breaks its connection to another (e.g., a turnout that was previously joined).

**Pass 2 (cornus):**
- Reiterates over selected tracks, now including cornus.
- For each endpoint:
  - If the connected track is still selected and not yet moved (fixed_end = FALSE), it re-establishes the connection by calling `SetCornuEndPt()` with the *already-moved* position of the connected track's endpoint.
  - If the connected track is **not** selected (`!GetTrkSelected(te)`), the cornus is treated as "free" — its geometry is recomputed from scratch (applies translation/rotation if applicable). This handles cases where a turnout arm was moved but not reconnected to a rail that is no longer in the selection.
- If `SetCornuEndPt()` fails (e.g., radius too small for curvature), delete the track and show an error message — this prevents invalid geometry from persisting.

**Why two passes?** Cornus depend on their connected partner's endpoint position. By moving non-cornu tracks first, then recomputing cornus with updated positions, we ensure consistency without needing to recompute everything in one go.

---

### `MoveToJoin(...)` — Snap Two Endpoints Together

```c
void MoveToJoin( track_p trk0, EPINX_T ep0, track_p trk1, EPINX_T ep1 )
```

This function moves *track0* so that its endpoint at `ep0` coincides with the endpoint of `trk1` at `ep1`. It:
1. Computes a translation vector (`base = trk1.ep1 - trk0.ep0`).
2. Computes the rotation angle needed to align tangents (end angles differ by 180° for straight rail continuity).
3. Calls `GetMovedTracks()` with `undraw=FALSE` so hilites remain visible during animation.
4. Moves all selected tracks using that translation and rotation.
5. Disconnects any endpoints on the moved tracks from unselected partners (cleanup).
6. Reconnects the two target endpoints via `ConnectTracks()`.
7. Calls `DrawNewTrack()` to render both tracks fresh with updated bounding boxes.

This is used for "Move to Join" — a command that lets you click one endpoint and another, then drags them together until they snap.

---

### `CmdMove(...)` — Move Command Handler

Responds to mouse drag events during move mode:

- **C_START** (first click):
  - Checks selection count > 0; if not, show error.
  - Checks frozen state (undoable operations blocked on frozen tracks).
  - Sets up anchors for the origin handle.
  - Enters `state = 1` to begin drawing preview.

- **C_DOWN** (mouse down during drag):
  - Commits any pending move with `UndoStart()`.
  - Clears accumulated movement data.
  - Re-enables draw preview (`drawEnable = enableMoveDraw`).
  - Enters active dragging state.

- **C_MOVE** (dragging):
  - Computes delta from origin: `base.x = pos.x - orig.x`, etc.
  - If ALT key is not pressed, snaps to grid (`SnapPos(&base)`).
  - Rebuilds the draw command for the new position.
  - Draws preview using `SetMoveD()` + `DrawMovedTracks()`.

- **C_UP** (mouse up):
  - If an endpoint intersection was found during dragging, call `MoveToJoin(trk1,ep1,trk2,ep2)` to weld them.
  - Otherwise, undo the move and discard changes.

- **C_CONFIRM / C_CANCEL**: Finalize or abandon any pending change.

---

### `CmdRotate(...)` — Rotate Command Handler

Similar state-machine pattern but with a rotation origin point and angle accumulator:

- **C_START**: Check selection count; set up anchor handle for origin point.
- **wActionMove** (mouse drag): Draw preview arc showing how selected tracks would rotate around the chosen center.
- **C_DOWN**: Compute final delta from origin to mouse position, apply rotation.
- **C_MOVE**: Update angle incrementally while dragging.
- **C_UP**: Commit or undo the rotation.

The key difference: instead of a translation vector (`base`), it accumulates an `angle` variable and uses `RotateTrack()` which computes new endpoint positions from center + radius × (original_angle ± angle).

---

### `DrawHighlightLayer(int layer)` — Highlight Tracks on a Layer

Walks all selected tracks, filters by layer index, and draws a bounding box around them using `DrawPoly()`. The polygon is filled with light blue (`wDrawColorPowderedBlue`) and drawn with dashed line style. Used to visually indicate "all objects on this layer are visible" or as feedback after applying a layer filter.

---

### `SetUpMenu2(pos, trk)` — Context Menu Setup

Prepares the right-click context menu based on what was clicked:
- Disables "Modify", "Description", "Hide" for text/draw objects (they have their own property dialogs).
- Enables "Hide track", "Add Ties", "Add Bridge" only if the track is not a Draw object.
- Checks whether the track can be modified (has control points, is cornu, or is an activatable signal).
- Stores the clicked track and position into `moveDescTrk` / `moveDescPos` for later use by property editing commands.

---

## Design Decisions & Tradeoffs

### Why Use Raycasting Instead of Point-in-Polygon?

Raycasting is **more robust** for overlapping geometry:
- If two objects overlap vertically, raycasting returns the *closest* one along the ray — which matches what a user expects ("click on the topmost object").
- Point-in-polygon would return true for *all* intersected objects unless explicitly ordered by depth.
- The cost is O(n) per click vs. O(1) with simple point tests, but n (number of selected tracks) is typically small (< 20).

### Why Maintain `tlist_da` as a Dynamic Array?

It avoids needing to maintain parallel arrays indexed by track index. With only one pointer per element and no fixed-size struct, the code handles arbitrary numbers of selections naturally. The C library's `DYNARR_` macros handle growth internally — no manual `realloc()` needed.

### Why Two-Pass Cornus Handling?

Cornus are **geometrically dependent** on their connected endpoint:
- If a turnout is moved because its rail was selected, the cornus geometry must be recomputed using the *new* position of that rail's endpoint.
- A single pass would require iterating all tracks multiple times or maintaining per-track "was-moved" flags.
- The two-pass approach (move rails first → recompute cornus) is simpler and less error-prone than trying to handle everything in one loop.

### Why Use `FindEndIntersection()` Instead of Direct Point Matching?

Because endpoints can be **spatially ambiguous**:
- Multiple endpoints might lie within floating-point tolerance of a single point (e.g., at a junction where three tracks meet).
- The function tries "unconnected" first — if an endpoint is already connected to another selected track, skip it.
- Falls back to finding *any* nearby endpoint — this handles corner cases like overlapping endpoints from different gauges or malformed geometry.

### Why Store Cornus in `auto_select_da` and Remove Them Later?

Cornus that are attached at an endpoint should **move with** their parent rail, but they don't need to be part of the selection list because:
- They are drawn automatically via `DrawMovedTracks()` when iterating over `tlist_da`.
- Deselection happens naturally when the parent rail is deselected.
- Keeping them in a separate array avoids duplicate entries (the same cornus pointer could appear both as a standalone track and as part of a turnout).

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `FindEndIntersection(...)` | Find matching endpoints between two selected tracks for "Move to Join" | translation delta, rotation origin/angle, pointers to track/index outputs |
| `DoSelectedTracks(callback)` | Generic iterator over selected tracks | callback function pointer and opaque data pointer |
| `RemoveSelectedTrack(trk)` | Remove a track from the selection list (in-place shift) | track pointer |
| `AddSelectedTrack(trk)` | Append a track to the selection dynamic array | track pointer |
| `GetMovedTracks(undraw)` | Rebuild selection list with only viewport-visible tracks + auto-add end cornus | whether to erase old hilites first |
| `MoveTracks(...)` | Apply move/rotate transformations to all selected tracks (two-pass for cornus) | erase flag, move/rotate flags, base point, origin, angle, undo flag |
| `CmdMove(action, pos)` | State-machine handler for mouse events during a track-move operation | action code, mouse position |
| `DrawHighlightLayer(layer)` | Draw a bounding box around all selected tracks on a given layer | layer index |
| `SetUpMenu2(pos, trk)` | Configure context menu state based on clicked object and its properties | screen point, track pointer (NULL if pan mode) |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Selection management via raycasting; multi-select accumulation; move/rotate operations with undo support; endpoint matching for "Move to Join"; layer highlighting |
| **Domain** | Interactive user input handling: mouse clicks, drags, keyboard shortcuts (Shift+Ctrl+Arrow), context menus |
| **Key concept** | Raycasting — cast rays from cursor position and return the nearest intersected object. This ensures correct depth-ordering of hit results even when objects overlap. |
| **Main entry points** | `CmdMove()` / `CmdRotate()` — invoked by main event loop on mouse drag; `FindEndIntersection()` — called during move-to-join or endpoint-snapping operations |
