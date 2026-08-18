# cstruct.c — Track Structure and Data Structures

## Overview

`cstruct.c` defines the core **track segment** data structures that represent physical track geometry in XTrkCad. It provides the fundamental building blocks for all track types (roadway, carriage way, flex track easements) including endpoints, segments, joints, and associated metadata like gauge, width, color, line type, and elevation.

---

## Key Structures

### `trkSeg_t` — Track Segment Descriptor

The fundamental unit of a track is the **segment**. A segment may represent:
- A straight line between two points (or two curved arcs)
- An arc of a circle or helix
- A vertical curve (transition) for flex-track easements
- A module component (switch, turnout, etc.)

```c
typedef struct {
    TRKTYP_T type;                   // Segment type: STRLIN, CRVLIN, BEZLIN, FILCRCL, FILPOLY, POLYLINE, TEXT, BENCH, TBLEDGE, etc.
    wIndex_t subType;               // Sub-type (e.g., POLYLINE vs RECTANGLE for polygons)
    DIST_T width;                   // Width of the track segment centerline in user units
    wDrawColor color;              // Drawing color (can be overridden by layer settings)
    LWIDTH_T lineWidth;            // Line width for rendering
    BOOL_T isDeleted;              // TRUE if this segment has been logically deleted but not yet freed
    union u {                       // Union of type-specific data
        struct {                    // Straight line or dimension line
            coOrd pos[2];          // Endpoint A and B coordinates (pixels from viewport origin)
            ANGLE_T angle;         // Angle of the segment at endpoint 0
            BOOL_T isDimLine;      // TRUE if this is a dimension/annotation line rather than physical track
        } l;

        struct {                    // Circle arc or filled circle arc
            coOrd center;          // Center point in screen coordinates (pixels)
            DIST_T radius;         // Radius of the circular arc. 0 means full circle/helix
            ANGLE_T a0;           // Start angle in degrees (CCW from horizontal right)
            ANGLE_T a1;           // End angle in degrees
        } c;

        struct {                    // Bezier curve segment
            coOrd pos[4];          // Control points: P0, P1, midpoint, P3 (cubic Bezier)
            dynArr_t bezSegs;      // Array of approximating line/arc segments for rendering
        } b;

        struct {                    // Polygon or filled polygon
            wIndex_t polyType;     // POLYLINE = open chain, RECTANGLE = closed box
            pts_t pts[];           // Vertex points. Each pt has a .pt_type flag (wPolyLineStraight, wPolyLineCorner, etc.)
        } p;

        struct {                    // Benchwork annotation line
            DIST_T width;          // Length of bench piece in user units
            ANGLE_T angle;         // Orientation of the bench
            BOOL_T grainDirection;// TRUE if grain direction is opposite to laydown direction
            char * materialName;   // Name of lumber type (e.g., "2x4", "6x10")
        } l_bench;

        struct {                    // Table edge annotation
            coOrd pos[2];          // Two endpoints defining the table edge line
        } te;
    } u;                            // Union containing exactly one of: .l, .c, .b, .p, .l_bench, or .te
} trkSeg_t, *trkSeg_p;
```

---

## Enumerations and Constants

### `TRKTYP_T` — Track Type Identifiers

| Value | Name | Description |
|-------|------|-------------|
| 0 | STRLIN | Straight line segment (physical track) |
| 1 | CRVLIN | Circular arc segment (physical track) |
| 2 | BEZLIN | Bezier curve approximation segment |
| 3 | FILCRCL | Filled circular arc (for filling interior of shapes) |
| 4 | FILPOLY | Filled polygon region |
| 5 | POLYLINE | Open polyline chain of straight segments |
| 6 | TEXT | Text annotation attached to a track segment |

---

### `wPolyLineStraight` / `wPolyLineCorner` — Polygon Vertex Types

Used within the `.p.pts[]` vertex array:

- **`wPolyLineStraight`** (0): The vertex lies along a straight edge of the polygon
- **`wPolyLineCorner`**: The vertex is a corner where two edges meet at an angle

---

### `TRKINX_T` — Track Index Type

A 32-bit signed integer used as a unique identifier for each track segment. Non-zero values indicate valid, live tracks; zero means the structure exists but represents a null/invalid entry.

```c
typedef int TRKINX_T[1];   // typedef'd to an array of length 1 to prevent accidental use as pointer arithmetic
```

---

## Segment Type Details

### Straight Line (`SEG_STRLIN`)

Used for: physical track centerlines, dimension lines, annotation leaders.

Data layout (`.u.l`):
- `.pos[0]` — First endpoint in screen coordinates
- `.pos[1]` — Second endpoint in screen coordinates
- `.angle` — Tangent angle at the first endpoint (degrees from horizontal)

---

### Circular Arc (`SEG_CRVLIN`) / Filled Circle Arc (`SEG_FILCRCL`)

Used for: circular track arcs, filled circles (for shape filling).

Data layout (`.u.c`):
- `.center` — Center point in screen coordinates
- `.radius` — Radius of the arc. **Zero** means a full circle or helix with infinite radius.
- `.a0` — Start angle in degrees (measured CCW from positive X-axis)
- `.a1` — End angle in degrees

For filled circles, `type == FILCRCL`; for arcs only, `type == CRVLIN`.

---

### Bezier Curve (`SEG_BEZLIN`)

Used to approximate smooth curved tracks (flex track easements). The segment stores four control points and a dynamic array of pre-computed line/arc approximation segments.

Data layout (`.u.b`):
- `.pos[0]` — Start point of the cubic Bezier curve
- `.pos[1]`, `.pos[2]` — Inner and outer control handles
- `.pos[3]` — End point
- `.bezSegs` — Dynamic array of small `trkSeg_t` elements that approximate the Bezier with lines/arcs

---

### Polygon (`SEG_POLY`) / Filled Polygon (`SEG_FILPOLY`)

Used for: polygonal track shapes (irregular loops, custom shapes).

Data layout (`.u.p`):
- `.polyType`: `RECTANGLE` — closed box; `POLYLINE` — open chain of segments
- `.pts[]` — Array of vertex points. Each point has:
  - `.pt` — Screen coordinates
  - `.pt_type` — `wPolyLineStraight` or `wPolyLineCorner`

---

### Benchwork (`SEG_BENCH`)

Used to draw lumber/bench annotations alongside track segments.

Data layout (`.u.l_bench`):
- `.width` — Length of the bench piece in user units
- `.angle` — Orientation angle relative to the rail direction
- `.grainDirection` — TRUE if grain runs opposite to laydown direction
- `.materialName` — Name string like "2x4", "6x10", etc.

---

### Table Edge (`SEG_TBLEDGE`)

A special annotation type marking where a table edge meets the track (used for tabletop modeling). Points to two endpoints defining the table edge line.

---

### Text Annotation (`SEG_TEXT`)

Attaches text labels to track segments (e.g., elevation notes, switch names).

Data layout (`.u.t`):
- `.pos` — Position of the text baseline
- `.angle` — Rotation angle of the text
- `.fontSize` — Font size in points
- `.string` — Pointer to dynamically allocated string holding the text content
- `.boxed` — TRUE if a rectangular box should be drawn around the text

---

## Related Structures (from other headers)

### `extraDataBase_t`

Shared header for all track segment types, containing undo/redo chain pointers and deletion flags. Defined in `track.h`.

```c
typedef struct {
    TRKTYP_T type;              // Segment type identifier
    trkSeg_p nextExtra;         // Next segment in the undo/redo chain
    BOOL_T isDeleted;           // TRUE if this segment has been logically deleted
} extraDataBase_t, *extraDataBase_p;
```

### `track_t` — Full Track Segment Structure

Combines the header with a pointer into the main track list:

```c
typedef struct {
    TRKINX_T index;            // Unique track ID (non-zero if valid)
    trkNext_t next;            // Pointer to next track in doubly-linked list
    coOrd endPos[2];          // Endpoint coordinates (pixels from viewport origin)
    ANGLE_T endAngle[2];      // Tangent angle at each endpoint (degrees CCW from horizontal)
    DIST_T gauge;             // Center-to-center rail spacing in user units
    DIST_T width;             // Width of track centerline for drawing
    TRKTYP_T type;            // Track segment type identifier
    BOOL_T isDeleted;         // TRUE if logically deleted but still in memory
    BOOL_T new;               // TRUE if newly created and not yet fully integrated
    wIndex_t layer;          // Layer index (visibility, module status, freeze state)
    extraDataBase_p extraData;  // Pointer to type-specific extra data
    trkSeg_p segs;           // Head of linked list of segments for this track
    int endCnt;              // Number of endpoints / degrees of freedom count
    BOOL_T deleted;          // TRUE if memory has been freed (logical deletion)
    BOOL_T modified;         // TRUE if geometry changed since last undo recording
    BOOL_T selected;         // TRUE if currently selected by user
} track_t, *track_p;
```

---

## Related Files

| File | Purpose |
|------|---------|
| `track.h` | Type definitions for tracks and segments |
| `cstruct.c` | Core data structure declarations (this file) |
| `drawgeom.h/drawgeom.c` | Drawing routines that consume these structures |
| `ccornu.c/ccornu.h` | Cornu easement generation (produces Bezier approximations stored here) |
| `cbezier.c/cbezier.h` / `tbezier.h` | Bezier curve mathematics and approximation |
