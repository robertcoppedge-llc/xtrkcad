# cdraw.c — Drawing Primitives and Text Rendering

## Overview

`cdraw.c` implements the **drawing subsystem** for XTrkCad. It handles rendering of geometric elements (lines, polylines, filled regions), text labels with font management, and provides a modifier menu system for changing drawing style (solid lines, dashed lines, dots, center marks, phantom lines, etc.). The module also converts Bezier curve approximations into renderable polygon vertex arrays.

---

## Core Data Structure: `extraDataDraw_t`

```c
typedef struct extraDataDraw_t {
    extraDataBase_t base;   // Shared undo/redo chain header + type ID
    coOrd orig;             // Origin point (where the element is placed)
    ANGLE_T angle;          // Rotation angle around origin
    drawLineType_e lineType;// Line style: SOLID, DASHED, DOT, CENTER, PHANTOM, etc.
    wIndex_t segCnt;        // Number of segments in this drawing object
    trkSeg_t segs[1];       // Dynamic array head for segment data
} extraDataDraw_t;
```

This structure is attached to every drawn track (via `extraDataBase_p` pointer) and stores the full geometric description needed for rendering.

---

## Drawing Line Types

| Constant | Value | Appearance | Usage |
|----------|--------|------------|-------|
| `DRAWLINESOLID` | 0 | Continuous line | Standard track geometry, dimension lines |
| `DRAWLINECENTERSOLID` | 1 | Solid with center ticks | Centerlines, centermarks |
| `DRAWLINECENTERDOT` | 2 | Dashed centerline | Alternate centermark style |
| `DRAWLINEDASHED` | 3 | Long-dash pattern | Hidden edges, optional features |
| `DRAWLINEDASHDOT` | 4 | Dash-dot pattern | Construction lines |
| `DRAWLINEDASHDOTDOT` | 5 | Dash-dot-dot pattern | Fine construction lines |
| `DRAWLINEPHANTOM` | 6 | Long dash + short dot | Phantom lines (alternate position) |
| `DRAWLINEDOT` | 7 | Dotted line | Dimension extensions, leader lines |

---

## Key Functions

### `LoadFontSizeList(wList_p list, long curFontSize)`

Populates a GTK combo-box/list with available font sizes. The function:
- Builds a list of common sizes (4pt through 180pt) plus the user's custom "large" size
- Inserts the current font size into the middle of the list so it's easily accessible
- Calls `wSetSelectedFontSize()` to set GTK's internal cursor position

**Parameters:**
- `list` — The GTK combo-box or list widget to populate
- `curFontSize` — Current selected font size (in points)

---

### `GetFontSize(wIndex_t inx)`

Retrieves the font size value corresponding to a given index into `fontSizeList`. Used by the GTK menu callback to convert the selected index back to a point value.

**Returns:** Font size in points, or -1 if index is out of range.

---

### `GetFontSizeIndex(long size)`

Finds the index (0-based) for a given font size within the standard list. Returns -1 if the size is not found in the predefined table.

---

### `UpdateFontSizeList(long *fontSizeR, wList_p list, wIndex_t listInx)`

Called when the user modifies a custom font size via the "large-font-size" preference dialog. It:
- Validates that the new size is positive
- Warns if it exceeds the system-wide "large font size" threshold (pref `misc.large-font-size`, default 500) and asks for confirmation
- Updates both the GTK display value and the stored preference
- Refreshes the list if needed

**Parameters:**
- `fontSizeR` — Pointer to the current font size variable (updated in place)
- `list` — The widget list being updated
- `listInx` — Index of the item being edited, or -1 for a new entry

---

### `MakeDrawFromSeg(coOrd pos, ANGLE_T angle, trkSeg_p sp)`

Converts a single track segment into a drawing object (type T_DRAW). It:
- Allocates a new track with type `T_DRAW`
- Copies the segment data into an `extraDataDraw_t` structure
- If the segment is a Bezier curve, calls `FixUpBezierSegs()` to normalize control points before converting
- Calls `MoveBezier()` and `RotateBezier()` to transform the curve to its final position/angle

**Parameters:**
- `pos` — Placement origin (pixels from viewport origin)
- `angle` — Rotation angle in degrees
- `sp` — Pointer to the source segment (`trkSeg_t`)

**Returns:** The newly created drawing track, or NULL if the source segment is invalid.

---

### `MakePolyLineFromSegs(coOrd pos, ANGLE_T angle, dynArr_t * segsArr)`

Aggregates multiple segments (lines, arcs, Bezier approximations) into a single **polyline** drawing object. This is used when:
- Converting a chain of connected segments into one polyline
- Combining several small straight/arced pieces into a single polygon for filling/rendering optimization

The function walks through the segment array and:
1. Detects where consecutive segments meet (by checking if endpoints are coincident)
2. For each arc, computes intermediate points using `SliceCuts()` to determine how many line/arc segments approximate it well enough
3. Builds a vertex list with `pts_t` entries tagged as `wPolyLineStraight`, `wPolyLineCorner`, or `wPolyLineSmooth` (for arcs)

**Parameters:**
- `pos` — Origin of the resulting polyline
- `angle` — Rotation angle
- `segsArr` — Dynamic array of segments to aggregate

**Returns:** A new track containing a single polygon segment with all vertices.

---

### `MakePolyFromSegs(coOrd pos, ANGLE_T angle, dynArr_t * segsArr)`

Similar to `MakePolyLineFromSegs`, but produces a **filled polygon**. The segments must form a closed loop (first and last endpoints coincide). This is used for:
- Filled regions (e.g., shaded areas)
- Polygonal track shapes that need interior filling

---

### `MakeTextTrack(coOrd pos, ANGLE_T angle, char * textString)`

Creates a track containing only a text label. The text is stored in an embedded string field and drawn using the GTK font rendering system (via `wDrawLabel()` or equivalent). Used for:
- Elevation markers
- Switch names
- Station numbers
- Notes and annotations

---

### `MakeLineFromSeg(coOrd pos, ANGLE_T angle, trkSeg_p sp)`

Creates a drawing object from a segment. If the segment is a Bezier curve, it first expands the Bezier into its underlying line/arc approximations before creating the draw track.

**Returns:** A new T_DRAW track or NULL on error.

---

## SliceCuts() — Arc Discretization

```c
int SliceCuts(ANGLE_T a, DIST_T radius)
{
    double Error = 0.05;
    double Error_angle = acos(1-(Error/fabs(radius)));
    if (Error_angle < 0.0001) { return 0; }
    return (int)(floor(D2R(a)/(2*Error_angle)));
}
```

Determines how many line segments are needed to approximate a circular arc of angle `a` and radius `radius`. The error bound is set at 0.05 pixels — the maximum perpendicular deviation between the true arc and its polyline approximation must not exceed this value. This ensures rendering quality even for high-resolution displays.

---

## BoundingBox Computation

```c
static void ComputeDrawBoundingBox(track_p t)
{
    struct extraDataDraw_t * xx = GET_EXTRA_DATA(t, T_DRAW, extraDataDraw_t);
    coOrd lo, hi;
    GetSegBounds(xx->orig, xx->angle, xx->segCnt, xx->segs, &lo, &hi);
    hi.x += lo.x;
    hi.y += lo.y;
    SetBoundingBox(t, hi, lo);
}
```

Computes the axis-aligned bounding box of a drawing object by:
1. Calling `GetSegBounds()` to get local min/max coordinates relative to origin
2. Shifting back to viewport space (adding origin offset)

The bounding box is stored in the track for culling and acceleration purposes.

---

## Text Font Management

XTrkCad uses GTK's font rendering system. The application maintains a list of available font families and sizes. Custom large fonts are stored as preferences (`misc.large-font-size`). When the user changes the custom size, `UpdateFontSizeList()` is invoked to refresh both the preference file and the GTK widget display.

---

## Related Files

| File | Purpose |
|------|---------|
| `cdraw.h` | Type definitions (`extraDataDraw_t`, drawing line types) |
| `track.h` / `cstruct.c` | Base track structure and segment union types |
| `tbezier.h` / `tbezier.c` | Bezier curve evaluation (used when converting drawn tracks back to segments) |
| `common-ui.h` | GTK widget wrappers (`wList`, `wFontSize_t`) |
| `drawgeom.c` | Interactive drawing commands that produce the segments fed into this module |
