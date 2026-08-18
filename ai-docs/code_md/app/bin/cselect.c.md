# cselect.c — Track Selection, Movement, and Alignment

## Overview

`cselect.c` implements the **track selection and manipulation system** for XTrkCad. It handles:
- Selecting/unselecting individual tracks or groups of connected tracks
- Drag-selecting multiple tracks with a bounding box
- Moving selected tracks (with automatic snapping to nearest endpoint)
- Rotating/aligning tracks relative to the viewport origin
- Flipping track orientation

The module also manages **selection anchors** — small graphical overlays drawn on top of selected objects using `trkSeg_t` segments in a dynamic array (`anchors_da`).

---

## Global State Variables

| Variable | Type | Description |
|----------|------|-------------|
| `selectedTrackCount` | `long` | Number of currently selected tracks (0 = none) |
| `selectMode` | `long` | Selection mode: 0=normal, 1=drag-select, etc. |
| `tlist_da` | `dynArr_t<track_p>` | Dynamic array holding selected track pointers |
| `moveMode` | `int` | Current movement sub-mode (MAXMOVEMODE = 3) |
| `enableMoveDraw` | `BOOL_T` | Whether to draw the move cursor/indicators |

---

## Core Data Structures

### `tlist_da` — Selected Tracks List

```c
static dynArr_t tlist_da;   // Dynamic array of track_p pointers
#define TListAppend(T) DYNARR_APPEND(track_p, tlist_da, 10); Tlist(tlist_da.cnt-1) = T;
BOOL_T TListSearch(track_p T);   // O(n) search for a track in the selection list
```

Tracks are appended to this list when selected and removed when deselected. The list maintains insertion order (order of selection).

---

## Selection Anchors System

Selection anchors are drawn using `trkSeg_t` segments stored in a dynamic array (`anchors_da`). Each anchor is itself a small track segment with:
- `type`: SEG_STRLIN (line), SEG_CRVLIN (arc), or SEG_FILCRCL (filled circle)
- `color`: wDrawColorBlue, Aqua, or PowderedBlue to distinguish anchor types
- `lineWidth`: 0 for invisible anchors; >0 for visible indicators

### Anchor Types and Their Meanings:

| Function | Visual | Meaning |
|----------|--------|---------|
| `CreateArrowAnchor()` | → arrow pointing away | Endpoint of a connected track (can be extended) |
| `CreateRotateAnchor()` | ◯ ring + three arrows | Track can be rotated around its center |
| `CreateModifyAnchor()` | ● filled circle | Object has editable properties |
| `CreateDescribeAnchor()` | ── two parallel arcs | Object has a description/description field |
| `CreateActivateAnchor()` | ◐ half-arc + line | Double-click activates an object (e.g., notes) |
| `CreateEndAnchor(coOrd p, wBool_t lock)` | ○ or ● | Endpoint of a curve; filled = locked/unmodifiable |

---

## Key Functions

### `SelectOneTrack(track_p trk, wBool_t selected)`

Selects (`selected == TRUE`) or deselects a single track. It:
1. Adds/removes the track from `tlist_da` using `TListAppend()` / removal logic
2. Calls `SetSelected(trk, selected)` to set the per-track flag
3. Invalidates the bounding box for redraw

**Note:** If adding a track that already exists in the list, it is skipped (prevents duplicates).

---

### `DrawSelectedTracksD(drawCmd_p d, wDrawColor color)`

Renders all currently selected tracks using the provided drawing context and color. Used during move/rotate operations to show what is being manipulated.

---

### `TListSearch(track_p T)`

Linear search through the selection list. Returns `TRUE` if the track is already selected (prevents duplicate entries).

---

### `CreateRotateAnchor(coOrd pos)`

Draws a circular rotation indicator at `pos` with three outward-pointing arrowheads spaced at 120° intervals. The blue color indicates "this object can be rotated." Used when a track is positioned at the viewport origin (the default pivot for rotation operations).

---

### `CreateModifyAnchor(coOrd pos)`

Draws:
- An inner filled circle (radius = d/4) — indicates properties are editable
- An outer ring arc (full 180°→360° sweep) — acts as a selection handle
- Cursor set to "none" (crosshair disabled since this is the drag target)

---

### `CreateDescribeAnchor(coOrd pos)`

Draws two short parallel arcs with vertical tick marks. Used for tracks that have a descriptive label or metadata field available via a description dialog.

---

### `CreateActivateAnchor(coOrd pos)`

Draws:
- A half-circle arc (from 70° to 320°, leaving a ~90° gap)
- A short radial line pointing outward from the midpoint of the gap
Used for objects that respond to double-click or right-click (e.g., notes, switch motors).

---

### `CreateEndAnchor(coOrd p, wBool_t lock)`

Draws either an open arc (`SEG_CRVLIN`) or filled circle (`SEG_FILCRCL`). Used at endpoints of Cornu curves and Bezier segments to indicate:
- **Open** — endpoint is free to be moved or extended
- **Filled** — endpoint is locked (connected to another track, cannot move)

---

### `CreateMoveAnchor(coOrd pos)`

Draws five arrowheads pointing outward from a central point. Indicates that the object can be dragged with Ctrl key held down. Cursor is set to "none."

---

## Selection Modes

| Mode | selectMode Value | Behavior |
|------|-----------------|----------|
| Normal selection | 0 | Click selects/deselects single tracks |
| Drag-select (box) | 1 | Mouse drag creates a rectangle; all tracks within are selected |
| Other modes | 2+ | Various specialized selections (e.g., connected components) |

---

## Movement Sub-modes (`moveMode`)

The move operation has multiple sub-states tracked by `moveMode`:
- Mode 0: Selecting which track to drag among the selection set
- Mode 1+: Actual dragging with different constraint behaviors

`enableMoveDraw` toggles whether the move cursor and anchors are drawn on screen.

---

## Related Files

| File | Purpose |
|------|---------|
| `cselect.h` | Type definitions (`tlist_da`, anchor structures) |
| `draw.h/cdraw.c` | Rendering primitives used by anchor functions |
| `trackx.h` | Track list management (splicing in/out of the main track array) |
| `custom.c/custom.h` | Extra data access for selecting tracks with custom properties |

---

## Summary

The selection system is designed around a **selection list** (`tlist_da`) and a set of **visual anchors**. The anchors provide immediate visual feedback about what operations are available on each selected object (move, rotate, modify, describe). This is consistent with the state-machine interaction model used throughout XTrkCad's drawing commands.
