# cselect.c — Track Selection, Highlighting, and Transformation Commands

## Overview

`cselect.c` implements the entire **Select Mode** in XTrkCad. This is a major module responsible for:

- Selecting/deselecting individual track components (tracks, cars, notes, structures)
- Drawing selection boundary markers ("X" crosses at endpoints of selected tracks)
- Highlighting selected objects in preview colors when not currently selected
- Transformations: Move, Rotate, Flip (mirror), Delete, Hide, Bridge, Roadbed, Ties, Layer assignment
- Connection tracking between adjacent track segments

The core entry point is `CmdSelect(action, pos)` which dispatches on mouse events and keyboard shortcuts.

## File Location

```
app/bin/cselect.c  (3971 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `common.h` | Global state (`mainD`, `tempD`, drawing context, cursors) |
| `draw.h` | Drawing primitives (`DrawTrack`, `DrawEndPt`, `DrawLine`) |
| `ccurve.h` | Curve creation (`NewCurvedTrack`) |
| `tcornu.h` | Cornu spiral utilities |
| `tbezier.h` | Bezier curve utilities |
| `track.h` | Track data structures and accessors |
| `compound.h` | Compound/structure track handling |
| `cselect.h` | Selection API declarations |
| `cundo.h` | Undo stack management (`UndoStart`, `UndoEnd`, `UndoModify`) |
| `custom.h` | Custom widget types (`wMenu_p`, `wIndex_t`) |
| `fileio.h` | I/O utilities |
| `layout.h` | Layout state (layers) |
| `param.h` | Parameter dialogs |
| `cjoin.h` | Track joining (`ConnectTracks`, `JoinTracks`) |
| `drawgeom.h` | Geometric drawing helpers |
| `ctrain.h` | Train/car-related functions |

## External Variables

```c
EXPORT wIndex_t selectCmdInx;   /* command menu index for Select Mode */
EXPORT wIndex_t moveCmdInx;     /* "Move" subcommand */
EXPORT wIndex_t rotateCmdInx;   /* "Rotate" subcommand */
EXPORT wIndex_t flipCmdInx;     /* "Flip (Mirror)" subcommand */

EXPORT long selectMode = 0;      /* current selection command index */
EXPORT long selectZero = 1;      /* flag indicating whether to allow deselecting */
```

## Selection State Variables

```c
static wDrawBitMap_p endpt_bm;    /* bitmap for drawing endpoint markers */
static wDrawBitMap_p angle_bm[4]; /* bitmaps at 0°, 45°, 90°, 135° offsets */

export long selectedTrackCount = 0;   /* number of currently selected tracks */

static dynArr_t anchors_da;           /* dynamic array for visual anchors (arcs, arrows) */
#define anchors(N) DYNARR_N(trkSeg_t,anchors_da,N)

// Anchor creation functions:
// CreateEndAnchor()        — endpoint anchor for extend/modify operations
// CreateModifyAnchor()     — concentric arcs indicating "drag to modify"
// CreateRotateAnchor()     — arc with arrowheads indicating rotation direction
// CreateArrowAnchor()      — straight line with V-tip at one end (arrow)
// CreateDescribeAnchor()   — small circle with radial line for description mode
// CreateActivateAnchor()   — clickable object indicator (e.g. notes)
// static CreateMoveAnchor() — X-shaped cross for move target

static track_p moveDescTrk;   /* the track being dragged during a Move operation */
static coOrd moveDescPos;     /* current mouse position during move drag */

int incrementalDrawLimit = 0; // threshold for delayed redraw vs immediate
static int microCount = 0;    /* counter to avoid flicker during large selections */
```

## Core Functions (Partial Listing)

### `CmdSelect(wAction_t action, coOrd pos)`

The main dispatch function for all selection-related commands. Handles:

- **`C_START`**: Initialize Select Mode — show "Select Mode" message, clear previous state.
- **`C_DOWN / C_LCLICK`**: Left-click on a track segment or endpoint → select/deselect that component.
- **`C_RDOWN` (Right-click)**: Right-click toggles between Move and Rotate modes in the selection toolbar.
- **`C_MOVE`**: Dragging an object (Move mode) — updates `moveDescTrk` and `moveDescPos`.
- **`C_UP` / `C_OK`**: Confirm a transformation (Move, Rotate, Flip).
- **`C_TEXT`**: Keyboard shortcuts for Delete ('Delete' key), Bridge/Roadbed/Ties toggles.
- **`C_CMDMENU`**: Context menu popup for transform operations.

### `SelectOneTrack(track_p trk, wBool_t selected)`

Selects or deselects a single track component. Called by other functions to change the selection state atomically. Updates `selectedTrackCount` and marks the track with `TB_SELECTED` / clears it.

### `DrawSelectedTracksBoundary()`

Redraws the "X" markers at endpoints of tracks that have changed their selected state since the last render. Uses a flag `TB_SELREDRAW` to mark tracks whose boundary needs redrawing, then clears it after drawing. This avoids unnecessary redraws when only one track's selection changes.

### `SetAllTrackSelect(BOOL_T select)`

Iterates over all visible (non-frozen) layers and sets or clears the `TB_SELECTED` bit on every track. Used by the "Select All" / "Deselect All" commands. Draws selected tracks in `selectedColor`, unselected in black.

### `InvertTrackSelect(void)`

Inverts the selection state of all visible, non-module, non-frozen objects (selects what was deselected and vice versa).

### `OrphanedTrackSelect(void)`

Automatically selects any track segments that are "orphaned" — i.e., they have no connected neighbor at either endpoint. Useful for finding disconnected pieces after editing.

## Selection Queries & Helpers

```c
EXPORT void HighlightSelectedTracks(track_p trk_ignore, BOOL_T keep, BOOL_T invert)
/* Draws selected tracks in preview colors (not affecting the active selection).
 * Used when you want to show what's selected without changing it. */

EXPORT BOOL_T SelectedTracksAreFrozen(void)
/* Returns TRUE if any of the currently selected tracks is on a frozen layer.
 * If so, most operations are disabled and an error message shown. */

EXPORT void SelectTrackWidth(void* width)
/* Changes the width (gauge) of all selected tracks. Uses UndoStart/UndoEnd. */

EXPORT void SelectLineType(void* widthVP)
/* Changes line style/pattern for Bezier, DRAW, or compound structure tracks. */

EXPORT int SelectDelete(void)
/* Deletes all selected track components. Returns -1 if not in select mode, 0 if success. */

EXPORT BOOL_T TListSearch(track_p T)
/* Searches the dynamic selection list (tlist_da) for a given track pointer. Used to avoid adding duplicates. */
```

## Transform Commands (All use UndoStart/UndoEnd)

### `SelectTunnel(void)` — Hide tracks (toggle visibility off)

Calls `FlipHidden()` which flips the `TB_VISIBLE` bit and redraws.

### `SelectBridge(void)` — Toggle "bridge" layer for selected tracks

### `SelectRoadbed(void)` — Toggle roadbed layer for selected tracks

### `SelectTies(void)` — Toggle whether ties (sleepers) are drawn on top of the track

### `MoveSelectedTracksToCurrentLayer(void)` — Moves all selected tracks to the current layer. Useful when you've created a turnout from another layer and want to move it back.

## Transformation Workflow

All transformations follow this pattern:

1. **`C_DOWN / C_RDOWN`**: Enter transformation mode (show arrows/arcs indicating direction).
2. **`C_MOVE`**: Drag the object with mouse — preview updates in real time.
3. **`C_OK`** or **`C_UP`**: Commit the change, wrapped in an Undo transaction via `UndoStart("Move", ...)` / `UndoEnd()`.

## Context Menu Integration

The select mode popup menu (`selectPopup1M`, `selectPopup2M`, etc.) provides access to:

- Move
- Rotate
- Flip (mirror)
- Delete
- Bridge / Roadbed / Ties toggles
- Tunnel / Show all / Hide selected
- Layer assignments

Each menu entry corresponds to a callback that invokes the appropriate subcommand.

## Notes

- The `tlist_da` dynamic array maintains a list of connected track components during a selection operation (used for compound turnout creation).
- The `TB_SELREDRAW` bit is used as an optimization: instead of redrawing every selected track immediately, only those whose connection state changed are marked and redrawn later.
- Select mode does not affect train simulation — it's purely a construction/editing aid.
