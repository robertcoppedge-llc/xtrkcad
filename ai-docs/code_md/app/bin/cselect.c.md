# cselect.c — Track Selection, Movement, Rotation, and Joining Operations

## Overview

`cselect.c` implements the core selection and transformation system for XTrackCAD. It handles:

- **Select/deselect tracks** with automatic connected-track propagation
- **Move and rotate selected tracks** interactively or via commands
- **Join two endpoints together** (creates a connection)
- **Layer management** — moving tracks between layers, selecting all on current layer
- **Elevation editing** — adding/removing elevations from track ends
- **Orphan detection** — auto-selecting unconnected single-track segments
- **Hotbar actions** for common operations (delete, invert select, clear elevations, etc.)

The file is organized into several logical sections: anchor drawing utilities, selection state management, movement/rotation commands, join operations, layer handling, elevation editing, and hotbar/command dispatch.

---

## Key Data Structures

### `tlist_da` — Dynamic Array of Selected Tracks

A dynamic array (`dynArr_t`) that accumulates all currently selected tracks during a multi-select operation or connected-track walk. Used internally by selection functions.

```c
static dynArr_t tlist_da;      // Accumulated selected tracks (during command)
#define Tlist(N) DYNARR_N(track_p, tlist_da, N)
#define TListAppend(T) \
    { DYNARR_APPEND(track_p,tlist_da,10); Tlist(tlist_da.cnt-1)=T; }

static BOOL_T TListSearch(track_p T)  // Returns TRUE if track is already in list
{
    for (int i=0; i<tlist_da.cnt-1; i++) {
        if (Tlist(i) == T) return TRUE;
    }
    return FALSE;
}
```

### `drawCmd_t tempSegsD` — Temporary Drawing Context

A drawing context used to render selected tracks before they are moved/rotated. Stores the original origin, size, and DPI so that temporary drawings can be transformed relative to them.

### Static State Variables

| Variable | Purpose |
|----------|---------|
| `selectedTrackCount` | Number of currently selected track objects |
| `selectMode` | Current selection mode flag |
| `moveMode` | Move operation state (MAXMOVEMODE=3 values) |
| `doMoveDraw`, `enableMoveDraw` | Controls whether move preview is drawn |
| `moving`, `rotateAlignState` | Command state flags |
| `panCenter` | Center point for panning operations |
| `microCount` | Counter for micro-step moves (debounce) |
| `flipHiddenDoSelectRecount` | Flag to trigger recount after hiding tracks |

---

## Anchor Drawing Functions

These functions create temporary graphical anchors that appear under the cursor during interactive commands. They draw small shapes using a global anchor array (`anchors_da`) so they don't interfere with normal drawing.

### `CreateArrowAnchor(coOrd pos, ANGLE_T a, DIST_T len)` — Draw an arrow pointing in direction `a`

Draws two blue line segments originating from `pos`, each at angle `a±135°` and of length `len`. Used to indicate "move" direction when the user drags. The cursor is set to invisible (`wCursorNone`) so that arrows don't obscure clicks.

### `CreateRotateAnchor(coOrd pos)` — Draw a rotate handle circle with directional arrows

Draws:
- A cyan arc from 180° to 360° (half-circle) centered at `pos` with radius ≈ 7.5% of the current scale unit.
- Three small blue arrow segments pointing outward at 90°, 210°, and 330°.

Indicates that a track can be rotated around this point. The cursor becomes invisible after drawing so clicks land on the anchor rather than being intercepted by a standard pointer image.

### `CreateModifyAnchor(coOrd pos)` — Draw a modify (description) handle circle

Draws a small cyan-filled circle plus a vertical blue line to indicate that this track can be opened into its description dialog for field editing. The cursor becomes invisible after drawing.

### `CreateDescribeAnchor(coOrd pos)` — Draw description-edit handles (two circles + bracket lines)

Used when a track's description fields need to be edited. Draws two small circles and connecting vertical segments near the track center, plus sets the cursor to invisible.

### `CreateActivateAnchor(coOrd pos)` — Draw an activable-object indicator (small circle + diagonal line)

Draws a small cyan arc (70°–320° sweep, radius 75% of scale unit) and a short blue diagonal line extending from its center. Used to indicate that double-clicking this object will trigger some action. Cursor is set invisible after drawing.

### `CreateEndAnchor(coOrd p, wBool_t lock)` — Draw an endpoint anchor for join/move operations

Draws either:
- A small cyan-filled circle (if `lock` = TRUE) — indicates the point is locked to a track endpoint during a move/join operation.
- An open cyan arc (180° sweep, radius half of above) — indicates an unlinked/adjustable endpoint.

### `CreateModifyAnchor(coOrd pos)` — Draw modify anchor with two concentric arcs (already covered above)

---

## Selection State Management

### `SetAllTrackSelect(BOOL_T select)` — Select or deselect all visible non-module tracks

Iterates over every track in the layout using `TrackIterate()`. For each:
- If `select` is TRUE, sets the `TB_SELECTED` bit and increments `selectedTrackCount`.
- If FALSE (deselect), checks whether it's already selected; if so, clears the flag and decrements the count.

Tracks on frozen layers or module layers are skipped. When going from unselected to selected, a redraw is triggered. When deselecting, only the boundary markers between selected/unselected tracks need redrawing (`RedrawSelectedTracksBoundary()`).

### `InvertTrackSelect(void)` — Toggle selection state of all visible non-module tracks

Walks all tracks and flips their selected bit. Used when pressing Ctrl+A to select/deselect everything. Triggers a recount of selected tracks and redraws boundary markers between adjacent selected/unselected groups.

### `OrphanedTrackSelect(void)` — Select any single-track segments that are not connected to anything

For each track, counts how many endpoints connect to another track. If none do (and the track is visible and not frozen), it selects the track. Useful for picking up stray pieces of track or debris that got left behind after editing. Triggers a recount and redraw.

### `SelectOneTrack(track_p trk, wBool_t selected)` — Select or deselect a single track object

Sets/clears the `TB_SELECTED` bit on a given track record. Updates `selectedTrackCount`. Marks the track with `TB_SELREDRAW` so its boundary markers are refreshed on the next redraw pass. If selection state hasn't changed, does nothing and returns early.

---

## Drawing Selected Tracks

### `DrawTrackAndEndPts(track_p trk, wDrawColor color)` — Draw a single track (including connected endpoints)

Calls `DrawTrack()` to render the main polyline/arc geometry, then for each endpoint that connects to another track (`GetTrkEndTrk()`), draws an end-point marker at that junction. The color is black by default; if the track itself was drawn in selectedColor, the endpoint gets selectedColor too (making the join visually obvious).

### `RedrawSelectedTracksBoundary(void)` — Redraw "X" markers between selected and unselected tracks

Walks all tracks with `TB_SELREDRAW` set. For each such track, examines every connected neighbor:
- If a neighbor is also marked for redraw (`TB_SELREDRAW`), skip it (both changed together).
- Otherwise, draw two short lines across the junction gap between the two endpoints. These lines cross at the join point and are colored white if one side is selected and the other isn't (indicating a "boundary" between selection groups). If both sides share the same selection state, the marker is removed (undrawn in white over black).

After finishing all tracks in this group, clears `TB_SELREDRAW` on the original track.

### `DrawSelectedTracksD(drawCmd_p d, wDrawColor color)` — Draw selected tracks with optional culling and undrawing

Iterates over `tlist_da`, which contains all currently selected tracks. For each:
- If not drawing to the map display (`&mapD`), checks whether the track's bounding box intersects the viewport; if not, skips it (culling).
- Draws the track in either white or black depending on a culling/undo pass, setting/clearing `TB_UNDRAWN`.

Used during drag-and-drop previews and undo operations.

---

## Selection Counting and Reconciliation

### `SelectedTrackCountChange(void)` — Ensure `selectedTrackCount` matches reality

Compares the current count with `oldCount`; if they differ, re-enables or disables command menu items (e.g., "Delete", "Rotate") accordingly. Only used when transitioning into/out of a selected state to avoid spurious redraws.

### `SelectRecount(void)` — Recompute selection count without changing any bits

Walks all tracks and recalculates `selectedTrackCount`. Called after operations that might have changed the set (e.g., hiding some tracks with "Hide Tracks").

---

## Layer-Based Selection

### `SelectCurrentLayer(void)` — Select all visible, non-module tracks on the current layer

Iterates over all tracks; selects those whose layer matches `curLayer` and are not frozen. Used to quickly select everything on one layer for bulk editing or moving.

### `DeselectLayer(unsigned int layer)` — Deselect all tracks on a given layer (but leave them selected if already deselected globally)

Walks all visible, non-module tracks whose layer equals the argument; clears their selection bits and removes them from the global select list.

---

## Track Properties: Width, Line Style, Color, Layer

### `SelectTrackWidth(void *width)` — Change width of all selected tracks (undoable)

Increments `selectedTrackCount`, then for each selected track:
- Undoes any prior modification (`UndoModify()`).
- Calls `SetTrkWidth()` with the new value.
- Redraws the track in white, then black again to show the change.

### `SelectLineType(void *widthVP)` — Change line style (solid/dashed/dotted) of all selected tracks

Similar pattern: undo old state, call `SetBezierLineType()`, `SetLineType()`, or `SetCompoundLineType()` depending on track type, then redraw. Handles Bezier, Draw, and Compound tracks differently.

### `SelectDelete(void)` — Delete all selected tracks (if not in modify mode)

Checks that a delete command is active. If so, iterates over selections; if any are train cars, skips the undo step since nothing will be deleted. Otherwise calls `UndoStart("delete")`, walks selections, and for each:
- Checks whether it's frozen → returns early with error.
- Calls `DeleteTrack()` to free memory and remove from internal lists.
Then undoes end if started.

### `TrySelectDelete(void)` — Handle keyboard Delete key press

Calls `SelectDelete()`. If that returns 1 (meaning the track is in Modify mode, not Select mode), sends a "Delete" text command via hotbar to trigger deletion through the description dialog instead.

---

## Visibility Toggles: Tunnel, Bridge, Roadbed, Ties

### `FlipHidden(track_p trk, BOOL_T unused)` — Toggle tunnel visibility (hide/show)

Undoes any prior change on this track. If currently visible, clears `TB_VISIBLE` and `TB_SELECTED|TB_SELREDRAW`; otherwise sets `TB_VISIBLE`. Also toggles `drawTunnel` global flag for consistent redraw behavior. Then redraws the track in black. Calls `SelectRecount()` at the end if a recount is needed.

### `FlipBridge(track_p trk, BOOL_T unused)` — Toggle bridge visibility

Sets/clears `TB_BRIDGE` and `TB_VISIBLE`. Bridges are drawn over other layers; when toggled off they become invisible but can be shown again later.

### `FlipRoadbed(track_p trk, BOOL_T unused)` — Toggle roadbed visibility

Toggles `TB_ROADBED` and `TB_VISIBLE`. Roadbed is a solid fill background; when hidden it leaves only rails visible.

### `FlipTies(track_p trk, BOOL_T unused)` — Toggle ties (sleepers) visibility

Toggles `TB_NOTIES` (when set = no ties drawn). Also toggles `TB_VISIBLE` to keep track of whether the object is shown at all.

---

## Elevation Editing Functions

### `ClearElevations(void *unused)` — Remove elevation from all selected tracks' endpoints

For each endpoint on every selected track:
- Calls `SetTrkEndElev(..., ELEV_NONE, 0.0, NULL)` which sets the Z-coordinate to zero and marks it as "no elevation" mode.
- Redraws the end-point marker in black afterward.

### `AddElevations(DIST_T delta)` — Add/subtract a constant vertical offset from all selected endpoints

Uses a global variable `elevDelta` (set by caller) as the amount to add. For each endpoint with an elevation already defined:
- Undraws and redraws it in white, then black.
- Calls `SetTrkEndElev(..., mode, elev+elevDelta, NULL)` which updates the Z coordinate while preserving its original edit mode (`DESC_RO`, `DESC_EDITABLE`, etc.).

---

## Writing Selected Tracks to Temporary Geometry

### `WriteSelectedTracksToTempSegs(void)` — Export selected track geometry into a flat segment array

Called before operations that need to manipulate tracks as raw segments (e.g., during move/rotate previews). For each selected track:
- Undoes any prior modification (`UndoModify()`), then draws the track into `tempSegs_da` via `DrawTrack()`.
- Redraws it in black again and re-applies selection bits.

This "flattens" compound objects (cornus, turnouts) into their constituent segments so they can be moved/rotated as a group.

---

## Move Operation

### `MoveTracks(BOOL_T eraseFirst, BOOL_T move, BOOL_T rotate, coOrd base, coOrd orig, ANGLE_T angle, BOOL_T undo)` — Core movement function

Called from the move command after the user drags to a new position. Parameters:
- `eraseFirst` = whether to first undraw all selected tracks (for visual feedback).
- `move` = translate by `base`.
- `rotate` = rotate around `orig` by `angle`.
- `undo` = wrap in an undo block (`UndoEnd()` at the end).

Algorithm:
1. Iterate over `tlist_da` (selected tracks). For each non-Cornu track, call `MoveTrack()` and/or `RotateTrack()`, then for each endpoint that connects to a neighbor, disconnect it so it doesn't move with this group.
2. Repeat the loop but now only for Cornu tracks:
   - If an endpoint is attached to a **selected** neighbor track, reconnect using a new Cornu curve (`SetCornuEndPt()` + `ConnectTracks()`). If that fails (curve too tight), delete the track and report an error.
   - If no neighbor exists, update the open endpoint by translating/rotating its stored center and angle.
3. After all moves are done:
   - Remove any auto-selected Cornu endpoints (`RemoveEndCornus()`).
   - Clear `TB_UNDRAWN` bits on all tracks.
   - Call `DoRedraw()`.
   - End the undo block if requested.

### `AccumulateTracks(void)` — Rebuild `tlist_da` from current selection

Called at the beginning of a move operation to repopulate the selected list (since some tracks may have been deleted or deselected). Skips Cornu objects and any non-track entities.

---

## Rotate Operation

### `CmdRotate(wAction_t action, coOrd pos)` — Command dispatcher for rotate command

Handles:
- **C_START**: checks selection count; sets initial state variables (`base`, `orig`, `angle`).
- **wActionMove**: draws a rotate handle (cyan arc + arrows).
- **C_DOWN / C_UP / C_MOVE**: processes dragging of the rotate anchor. Updates `angle` relative to the original base position and redraws.
- **C_OK**: applies rotation via `MoveTracks(TRUE, FALSE, TRUE, ...)` then undoes end.
- **C_CANCEL**: abandons the operation without committing changes.

---

## Join Operation

### `FindEndIntersection(coOrd base, coOrd orig, ANGLE_T angle, track_p *t1, EPINX_T *ep1, track_p *t2, EPINX_T *ep2)` — Find two endpoints that meet after a move/rotate

Given a hypothetical displacement (`base`, `orig`, `angle`), this function:
- Iterates over all selected tracks.
- For each endpoint of each track (that isn't already connected to another selected track or already a Cornu itself):
  - Computes where it would land after the transform.
  - Calls `OnTrackIgnore()` on that transformed point to see if it lies on another track's polyline.
  - If so, uses `PickUnconnectedEndPointSilent()` to find which endpoint of that other track is closest.
  - If both points are within a small tolerance (defined by `IsClose`), returns TRUE and fills in the two tracks and their endpoint indices.

Used during move-to-join preview to tell the user whether dragging will connect two endpoints.

### `MoveToJoin(track_p trk0, EPINX_T ep0, track_p trk1, EPINX_T ep1)` — Connect two endpoints after a drag-and-drop

Called when the user releases the mouse over a valid join target:
- Computes the translation vector (`base`) and rotation angle needed to make endpoint 0 coincide with endpoint 1.
- Calls `GetMovedTracks(FALSE)` to rebuild the selection list.
- Calls `MoveTracks(TRUE, TRUE, TRUE, base, orig, angle, TRUE)` which performs the actual move/rotate and reconnects endpoints.
- Finally calls `ConnectAllEndPts()` on all selected tracks so that any dangling endpoints (created by the join) get linked up automatically.

---

## Layer Operations

### `SetLayer(track_p trk, BOOL_T unused)` — Move a track to the current layer

Undoes any prior change and calls `SetTrkLayer(trk, curLayer)`. Used when "Move To Current Layer" is invoked from a popup menu.

### `DrawHighlightLayer(int layer)` — Draw a blue outline around all visible tracks on a given layer

Walks all selected tracks whose layer matches the argument; computes their combined bounding box and draws a polygon in powder-blue with dashed stroke. Used to show which tracks belong to a particular layer when hovering over it.

---

## Utility Functions

### `FreeTempStrings(void)` — Free any text strings stored in temporary segments

Iterates over `tempSegs_da`; if a segment is of type SEG_TEXT, frees its string pointer and resets the pointer to NULL. Prevents memory leaks after undo/redo operations.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `CreateArrowAnchor()` / `CreateRotateAnchor()` / `CreateModifyAnchor()` / `CreateDescribeAnchor()` / `CreateActivateAnchor()` / `CreateEndAnchor()` | Draw visual anchors under the cursor during interactive commands |
| `SetAllTrackSelect(select)` | Select/deselect all visible non-module tracks |
| `InvertTrackSelect()` | Toggle selection of all visible non-module tracks |
| `OrphanedTrackSelect()` | Auto-select unconnected single-track segments |
| `SelectOneTrack(trk, selected)` | Select/deselect a specific track object |
| `DrawTrackAndEndPts()` / `RedrawSelectedTracksBoundary()` | Draw tracks and highlight selection boundaries |
| `ClearElevations()` | Remove elevation from all endpoints of selected tracks |
| `AddElevations(delta)` | Add/subtract a constant offset to all endpoint elevations |
| `WriteSelectedTracksToTempSegs()` | Flatten selected tracks into a flat segment list for manipulation |
| `MoveTracks(...)` / `AccumulateTracks()` / `GetMovedTracks()` | Move and/or rotate selected tracks, handling Cornu reconnection |
| `FindEndIntersection(...)` | Determine whether two endpoints meet after a proposed move/rotate |
| `MoveToJoin(trk0,ep0,trk1,ep1)` | Connect two track endpoints together after moving them into alignment |
| `SelectTrackWidth()` / `SelectLineType()` | Change width or line style of selected tracks |
| `SelectDelete()` / `TrySelectDelete()` | Delete selected tracks (via keyboard or menu) |
| `FlipHidden()` / `FlipBridge()` / `FlipRoadbed()` / `FlipTies()` | Toggle visibility attributes on a track |
| `SelectCurrentLayer()` / `DeselectLayer(layer)` | Select/deselect by layer membership |
| `DrawHighlightLayer(layer)` | Draw a bounding box around all tracks on a given layer |

---

## Notes

- All selection operations use the undo system (`UndoStart`/`UndoEnd`) except for simple toggle operations that affect only display. This allows users to revert moves, joins, or visibility changes via Ctrl+Z.
- Cornu curves are special-cased because they consist of multiple segments (arcs + lines) and must be reconnected after a move; the other track types (fixed arcs/straights) don't need this extra step.
- The `tlist_da` dynamic array is used as a scratch buffer during command execution; it's reset at the start and cleared again before returning to normal display mode.
