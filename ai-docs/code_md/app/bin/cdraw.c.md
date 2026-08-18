# cdraw.c — Drawing of Geometric Elements (Track Segments)

## Overview

`cdraw.c` is the core rendering and editing engine for XTrackCAD. It handles:

- **Drawing track segments** (straights, curves, bezier arcs, polygons, text labels, benches, table edges)
- **The description system** — a dialog-based UI that lets users edit any field of any segment (point coordinates, radius, angle, color, width, line style, fill state, layer, pivot point, etc.)
- **Dynamic drawing of previews** during drag-and-drop operations
- **Hit detection and distance queries** for snapping and selection

The file defines a "segment" data structure that can represent any geometric primitive (line, arc, polyline, text, rectangle, bench, table edge) as well as compound objects like filled shapes. All drawing is done via GTK+ primitives (`gtk_draw_polygon`, `gtk_arc`, etc.).

---

## Key Data Structures

### `trkSeg_t` — A Track Segment

A single segment record can represent:
- A straight line (SEG_STRLIN, SEG_DIMLIN)
- An arc (SEG_CRVLIN, SEG_CRVTRK)
- A Bezier curve (SEG_BEZLIN, SEG_BEZTRK)
- A polyline or filled polygon (SEG_POLY, SEG_FILPOLY)
- Text (SEG_TEXT)
- Bench data (SEG_BENCH)
- Table edge (SEG_TBLEDGE)

```c
typedef struct {
    char type;                          // SEG_STRLIN, SEG_CRVLIN, etc.
    wIndex_t subType;                   // 0 = straight line, 1 = dimensioned
    wDrawColor color;                   // Color index
    LWIDTH_T lineWidth;                 // Stroke width in pixels (scaled)
    unsigned int layer;                 // Layer ID for drawing order
    BOOL_T masked;                      // TRUE if not shown on current scale
    struct {                            // Union of actual geometry:
        struct {                        // SEG_STRLIN / SEG_DIMLIN
            coOrd pos[2];              // Start and end points
        } l;
        struct {                        // SEG_CRVLIN / SEG_CRVTRK
            coOrd center;               // Center point
            DIST_T radius;              // Radius (signed: + for CCW, - for CW)
            ANGLE_T a0;                 // Start angle
            ANGLE_T a1;                 // End angle
        } c;
        struct {                        // SEG_BEZLIN / SEG_BEZTRK
            coOrd pos[2];              // First and last control points of the Bezier
            dynArr_t bezSegs;          // Sub-segments (lines or arcs)
        } b;
        struct {                        // SEG_POLY / SEG_FILPOLY
            int cnt;                    // Number of vertices
            pts_t pts[1];              // Array of (x, y, type) tuples
            enum wPolyType polyType;    // RECTANGLE, FREEFORM, etc.
        } p;
        struct {                        // SEG_TEXT
            char *string;               // Text string
            coOrd pos;                  // Origin point
            ANGLE_T angle;              // Rotation
            long fontSize;              // Font size (in points)
        } t;
        struct {                        // SEG_BENCH / SEG_TBLEDGE
            struct benchData_s *bench;  // Pointer to bench definition data
        } u;                           // Union for user-defined segment types
    } u;
} trkSeg_t, *trkSeg_p;
```

### `extraDataDraw_t` — Draw Object Extra Data (used with T_DRAW track type)

A "draw" object is a collection of segments that share a common origin and angle transform. Used for placing multiple segments together as a group.

```c
typedef struct extraDataDraw_t {
    extraDataBase_t base;               // Generic header: index, layer, etc.
    coOrd orig;                         // Transformation origin (x,y)
    ANGLE_T angle;                      // Rotation around origin
    drawLineType_e lineType;            // SOLID, DASHED, DOT, etc.
    wIndex_t segCnt;                    // Number of segments in the group
    trkSeg_t segs[1];                  // Variable-length array of segments
} extraDataDraw_t, *extraDataDraw_p;
```

### `drawDesc_e` — Description Field Enum

Each field that can be edited via the description dialog has a corresponding enum value:

| Value | Name | Purpose |
|-------|------|---------|
| E0 / E1 | End Pt 1/2 X,Y | Position of endpoint 0 or 1 |
| PP | First Point (polyline) | Origin for polygon drawing |
| CE | Center | Arc center point |
| AL | Angular Length | Sweep angle of an arc |
| LA | Line Angle | Orientation angle of a segment |
| A1 / A2 | CCW Angle / CW Angle | Start and end angles of an arc |
| RD | Radius | Curvature radius (positive = CCW) |
| LN | Length | Total length of the segment |
| HT / WT | Height / Width | For rectangular fills |
| PV | Pivot | Pivot point for rotation |
| VC | Point Count | Number of polygon vertices |
| LW | Line Width | Stroke width in pixels |
| LT | Line Type | SOLID, DASHED, DOT, etc. |
| CO | Color | RGB color index |
| FL | Filled | TRUE/FILL_POLY or FALSE/empty |
| OP | Open End | For open vs closed polyline |
| BX | Boxed | Draw a bounding box around the segment |
| BE | Bench Choice | Which bench definition to use |
| OR | Orientation | How the bench is oriented on the track |
| DS | Size | Dimension label size/font |
| TP | Text Origin | Position of text annotation |
| TA | Text Angle | Rotation of text |
| TS | Font Size | Point size for text |
| TX | Text | The actual string to draw |
| LK | Lock To Origin | Whether origin stays fixed during moves |
| OI | Rot Origin | X,Y of the rotation origin |
| RA | Rotate By | Angle offset applied on creation |
| LY | Layer | Which layer this segment belongs to |

---

## The Description System: Editing Fields via Dialogs

The description system allows editing any field of a segment through a dialog UI. When a user edits a value, the `UpdateDraw()` function recomputes all derived quantities and redraws the track.

### Core Concepts

- **`descData_t`** — A record describing one editable field: its label (translated string), the pointer to where it's stored in memory, and how that memory location changes when edited.
- **`drawDesc[]`** — An array of `descData_t` records defining every possible editable field. Each has a mode flag (`DESC_POS`, `DESC_FLOAT`, `DESC_ANGLE`, `DESC_DIM`, `DESC_LIST`, etc.) and a pointer to the target variable.
- **`DrawNewTrack(track_p trk)`** — Called after an edit: undoes the old drawing, calls `UpdateDraw()` with the changed field index, then redraws.
- **`UndrawNewTrack(track_p trk)`** — Removes the old geometry from the screen before a new draw is rendered.

### Editing Workflow

1. User clicks "Properties" on a track → opens a GTK dialog containing labels and controls (spinners, sliders, text boxes) for each field.
2. Dialog callbacks set the corresponding field in `drawData` (a global struct that accumulates pending edits).
3. When the user hits OK, the description system walks through all fields marked as changed (`DESC_CHANGE`), calling `UpdateDraw()` for each one.
4. `UpdateDraw()` undraws the segment and redraws it with the new geometry.

### Special Cases

- **Pivot-based editing** (LA, AL): changes a field that affects the overall shape by rotating or scaling around a pivot point.
- **Origin locking** (LK, OI): when locked, moving an endpoint also moves the origin; when unlocked, the object stays visually static while internal coordinates change.
- **Radius vs length coupling**: for arcs, changing radius automatically updates length and vice versa via shared computed fields.

---

## Drawing Functions

### `DrawTrack(track_p trk)` — Render a Track Object

This is the primary drawing entry point:

1. If the track has no bounding box or is off-screen, skip it (optimization).
2. Retrieve extra data for the track type (`T_CORNU`, `T_BEZIER`, etc.).
3. Iterate over all segments in that object's geometry array.
4. For each segment:
   - If it's a Bezier sub-segment, delegate to `DrawBezierSegment()`.
   - Otherwise, render directly using GTK+ primitives (`gtk_arc` for arcs, `gtk_draw_polygon` for polygons/lines, etc.).

### `DrawTrack(track_p trk, int type)` — Draw with a Specific Style

A variant that accepts an explicit style argument. Used when drawing previews during drag-and-drop or undo operations.

---

## Distance and Hit Detection

The "arm" (selection cursor) uses these distance functions to determine whether the user is hovering over a segment:

### `DistanceDraw(track_p t, coOrd *p)` — Distance from Point to Draw Object

Returns the shortest distance from point `p` to any of the segments in object `t`. If the object is ignored (e.g., table edge or already-selected), returns infinity. Calls down into generic `DistanceSegs()` which computes perpendicular distances for lines/arcs and Euclidean distance for points.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `LoadFontSizeList()` / `GetFontSize()` / `UpdateFontSizeList()` | Manage a dropdown list of font sizes (4–500 pt) with memory-efficient storage |
| `MakeDrawFromSeg1()` / `MakeDrawFromSeg()` | Create a new draw object from a single segment record; handles Bezier decompression |
| `MakePolyLineFromSegs()` | Convert an array of segments into a polyline (used for slicing curves at gauge width) |
| `SliceCuts(angle, radius)` | Determine how many arc slices are needed to keep chord error below 0.05 units |
| `DrawOriginAnchor(track_p trk)` | Draw blue crosshairs indicating the origin of a draw object (for visual reference during editing) |
| `DistanceDraw()` / `OnDraw(track_p t, coOrd *p)` | Hit-testing and distance queries for selection |
| `UpdateDraw(trk, inx, descUpd, final)` | Apply changes to a segment field; handles reorientation of all dependent points |
| `UndrawNewTrack()` / `DrawNewTrack()` | Remove old geometry before redrawing (used during edit operations) |

---

## Notes

- The draw object system is intentionally generic: the same `UpdateDraw()` function can handle editing of any field on any segment type, because each field's effect is expressed as a transformation applied to its stored coordinates.
- Bezier segments are decomposed into line and arc pieces by `FixUpBezierSegs()`, then rendered using basic GTK+ drawing primitives — no custom rasterization is needed.
- The description system decouples "what the user edits" from "where the data lives," enabling dynamic addition of new fields without changing core logic.
