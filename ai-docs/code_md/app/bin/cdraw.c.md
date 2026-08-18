# cdraw.c — Drawing of Geometric Elements

## Overview

`cdraw.c` is the **core drawing engine** for XTrkCAD. It provides functions to:
- Create `track_p` objects representing Draw objects (polygons, text, lines, etc.) from segment data
- Compute bounding boxes for Draw objects
- Handle distance/hit testing for Draw objects during selection/raycasting

The file defines the **Draw track type** (`T_DRAW`) and its associated extra data structure that stores all geometric parameters needed to render a free-form polygonal shape.

Unlike tracks, which represent railroad geometry with physical meaning (rails, switches, turnouts), Draw objects are arbitrary 2D shapes used as obstacles, decorations, or graphical elements in the layout. They can be:
- Polyline chains (`SEG_POLY`)
- Filled polygons (`SEG_FILPOLY`)
- Straight lines (`SEG_STRLIN`, `SEG_DIMLIN`)
- Curved arcs (`SEG_CRVLIN`, `SEG_FILCRCL`)
- Text labels (`SEG_TEXT`)
- Bench lumber elements (`SEG_BENCH`)
- Table edges (`SEG_TBLEDGE`)

---

## Data Structures

### `extraDataDraw_t` — Draw Object Extra Data

```c
typedef struct extraDataDraw_t {
    extraDataBase_t base;         // Base header linking back to track_p
    coOrd orig;                    // Origin point (reference for transformations)
    ANGLE_T angle;                 // Rotation angle applied to the object
    drawLineType_e lineType;      // Line style: SOLID, DASHED, DOT, etc.
    wIndex_t segCnt;              // Number of segments in the shape
    trkSeg_t segs[1];             // Variable-length array of segment descriptors
} extraDataDraw_t;
```

Key points:
- `orig` is the reference point used for applying transformations (rotation around origin, translation). It can be locked to keep an object anchored while editing.
- `angle` is a global rotation applied to all segments — useful for rotating entire groups of Draw objects together.
- `lineType` controls how line segments are rendered: solid, dashed, dotted, center-line, phantom, etc.
- `segs[]` is a variable-length array (allocated dynamically per track) describing each segment's type and parameters.

### `drawData` — Global State for Drawing Edit Mode

```c
static struct {
    coOrd endPt[4];               // Points used for endpoints of lines/arcs/text
    coOrd origin;                 // Current origin point
    coOrd oldOrigin;              // Previous origin (for "lock to origin" mode)
    coOrd oldE0, oldE1;           // Old positions of endpoint 0 and 1 (for computing delta)
    FLOAT_T length;               // Length of current segment
    FLOAT_T height;               // Height parameter
    FLOAT_T width;                // Width parameter
    coOrd center;                 // Center for arcs/circles
    DIST_T radius;                // Radius for circles/arcs
    ANGLE_T angle0, angle1;       // Start/end angles for an arc
    ANGLE_T angle;                // Angular length or line angle
    ANGLE_T rotate_angle;         // Angle to apply when origin is locked
    long pointCount;              // Point count for polygons (stored in extra data)
    LWIDTH_T lineWidth;           // Line width / dash gap size
    BOOL_T boxed;                 // "Boxed" checkbox state
    BOOL_T filled;                // Filled polygon flag
    BOOL_T open;                  // Open vs. closed polygon
    BOOL_T lock_origin;           // Whether origin is locked (prevents object from moving)
    wDrawColor color;             // Current line fill color
    wIndex_t benchChoice;         // Bench lumber style index
    wIndex_t benchOrient;         // Lumber orientation (parallel/perpendicular/angled)
    wIndex_t dimenSize;           // Dimension text size
    descPivot_t pivot;            // Pivot point for angle editing (first/mid/end)
    wIndex_t fontSizeInx;        // Font size index
    char text[STR_HUGE_SIZE];    // Text label string
    unsigned int layer;          // Layer index
    wIndex_t lineType;           // Line type selector
} drawData;
```

This is a **global state** structure used while the user is editing a Draw object. It accumulates all parameters entered via the Describe dialog and applies them to the track when the edit is confirmed. The `lock_origin` flag allows the origin point to be fixed in place (relative to viewport) so that other parameters can be adjusted without shifting the whole object around.

---

### `drawDesc[]` — Field Definitions for the Describe Dialog

```c
static descData_t drawDesc[] = {
    /*E0*/  { DESC_POS, N_("End Pt 1: X,Y"),   &drawData.endPt[0] },
    /*E1*/  { DESC_POS, N_("End Pt 2: X,Y"),   &drawData.endPt[1] },
    /*PP*/  { DESC_POS, N_("First Point: X,Y"),&drawData.endPt[0] },
    /*CE*/  { DESC_POS, N_("Center: X,Y"),     &drawData.center },
    /*AL*/  { DESC_FLOAT,N_("Angular Length"), &drawData.angle },
    /*LA*/  { DESC_FLOAT,N_("Line Angle"),     &drawData.angle },
    /*A1*/  { DESC_ANGLE,N_("CCW Angle"),      &drawData.angle0 },
    /*A2*/  { DESC_ANGLE,N_("CW Angle"),       &drawData.angle1 },
    /*RD*/  { DESC_DIM, N_("Radius"),          &drawData.radius },
    /*LN*/  { DESC_DIM, N_("Length"),          &drawData.length },
    /*HT*/  { DESC_DIM, N_("Height"),          &drawData.height },
    /*WT*/  { DESC_DIM, N_("Width"),           &drawData.width },
    /*PV*/  { DESC_PIVOT, N_("Pivot"),         &drawData.pivot },
    /*VC*/  { DESC_LONG, N_("Point Count"),    &drawData.pointCount },
    /*LW*/  { DESC_DIM, N_("Line Width"),      &drawData.lineWidth },
    /*LT*/  { DESC_LIST, N_("Line Type"),      &drawData.lineType },
    /*CO*/  { DESC_COLOR,N_("Color"),          &drawData.color },
    /*FL*/  { DESC_BOXED, N_("Filled"),        &drawData.filled },
    /*OP*/  { DESC_BOXED, N_("Open End"),      &drawData.open },
    /*BX*/  { DESC_BOXED, N_("Boxed"),         &drawData.boxed },
    /*BE*/  { DESC_LIST, N_("Lumber"),         &drawData.benchChoice },
    /*OR*/  { DESC_LIST, N_("Orientation"),    &drawData.benchOrient },
    /*DS*/  { DESC_LIST, N_("Size"),           &drawData.dimenSize },
    /*TP*/  { DESC_POS, N_("Text Origin: X,Y"),&drawData.endPt[0] },
    /*TA*/  { DESC_FLOAT, N_("Text Angle"),    &drawData.angle },
    /*TS*/  { DESC_EDITABLELIST,N_("Font Size"),&drawData.fontSizeInx },
    /*TX*/  { DESC_TEXT, N_("Text"),           &drawData.text },
    /*LK*/  { DESC_BOXED, N_("Lock To Origin"),&drawData.lock_origin},
    /*OI*/  { DESC_POS, N_("Rot Origin: X,Y"), &drawData.origin },
    /*RA*/  { DESC_FLOAT, N_("Rotate By"),     &drawData.rotate_angle },
    /*LY*/  { DESC_LAYER, N_("Layer"),         &drawData.layer },
    { DESC_NULL }
};
```

This array maps each field name shown in the Describe dialog to:
- A `descData_t` struct (pointer into a Draw object's extra data)
- The appropriate type flag (`DESC_POS`, `DESC_FLOAT`, `DESC_LIST`, etc.)
- A human-readable label for the column header

Note that some fields share the same pointer (e.g., both "End Pt 1" and "First Point" point to `endPt[0]`) — this is because they represent the *same* underlying data, just labeled differently depending on whether the user is editing a line's endpoints or a polygon's vertices.

---

## Core Functions

### `MakeDrawFromSeg(trkSeg_p sp)` / `MakeDrawFromSeg1(...)` — Create a Draw Track from a Segment Descriptor

**Purpose:** Convert a single segment descriptor (from a selection hit-test) into a fully-formed `track_p` object of type `T_DRAW`.

**Input:** A pointer to a `trkSeg_t` describing one geometric element (e.g., a polyline vertex pair, a circle center/radius/angle triple). The caller passes the position and angle at which the segment was hit.

**Algorithm:**
1. Check if the segment has no valid type — return `NULL`.
2. If it's a Bézier-like segment (`SEG_BEZLIN`), convert to a track: call `NewBezierLine()` with appropriate parameters, then move/rotate so that the Bézier control points align with the hit position and orientation.
3. Allocate a new track via `NewTrack(0, T_DRAW, 0, sizeof(extraDataDraw_t))`.
4. Copy segment data into the extra data structure: store origin, angle, line type (default SOLID), set `segCnt = 1`.
5. Copy the segment's coordinates/parameters into `xx->segs[0].u.l.pos[]` or `.u.c.center/...` as appropriate.
6. If the segment is a polyline (`SEG_POLY`) or filled polygon (`SEG_FILPOLY`), allocate and copy the array of points — each point carries its own sub-type flag (`wPolyLineStraight`, `wPolyLineSmooth`, etc.).
7. If it's text, duplicate the string into a new buffer.
8. Call `ComputeDrawBoundingBox(trk)` to compute the tight axis-aligned bounding box for selection hilites.

**Why two functions?** `MakeDrawFromSeg1` is the internal implementation; `MakeDrawFromSeg` wraps it with a default index of 0 and delegates to the same code path. This avoids duplication when calling from multiple places in the state machine.

---

### `ComputeDrawBoundingBox(track_p t)` — Compute Tight Bounding Box

```c
static void ComputeDrawBoundingBox( track_p t ) {
    struct extraDataDraw_t * xx = GET_EXTRA_DATA(t, T_DRAW, extraDataDraw_t);
    coOrd lo, hi;
    GetSegBounds( xx->orig, xx->angle, xx->segCnt, xx->segs, &lo, &hi );
    hi.x += lo.x;
    hi.y += lo.y;
    SetBoundingBox( t, hi, lo );
}
```

This is a **simple axis-aligned bounding box** around all segments after applying the object's origin offset and global rotation angle. It uses `GetSegBounds()` — a helper that walks each segment, computes min/max x/y in the *local* coordinate space (relative to origin), then translates by adding the origin offsets at the end.

The function is called every time a Draw object is created or modified so that its bounding box stays consistent with its visual appearance on screen. This is used later during raycasting for hit detection and during selection hilite drawing.

---

### `MakePolyLineFromSegs(...)` — Concatenate Multiple Segments into One Polyline

**Purpose:** Given an array of segment descriptors (`dynArr_t *segsArr`), build a single polyline that stitches them together seamlessly. This is used when the user selects multiple segments (e.g., via shift-click) and wants to create one unified Draw object representing their union.

**Key algorithmic detail — deduplication:**

At each junction between consecutive segments, the code checks:
```c
if (!first && IsClose(FindDistance(spb->u.l.pos[0], last))) {
    cnt++;  // Skip this point (it's a duplicate of the previous one)
} else {
    cnt = cnt + 2;  // Add both endpoints as new points
}
```

This ensures that shared vertices are not duplicated in the output — adjacent segments share an endpoint, so only one copy is needed. The `IsClose(...)` check uses a small tolerance to handle floating-point rounding errors (e.g., due to accumulated roundoff from many Bézier segments).

**Arc handling:** For curved segments (`SEG_CRVLIN` or `SEG_CRVTRK`), the code:
1. Computes the point on each rail at angle `a0`.
2. Calls `SliceCuts(a1, radius)` — which computes how many equal angular slices are needed to approximate the arc within a given error bound (default 0.05 pixels).
3. Inserts intermediate points along the arc using linear interpolation between `a0` and `a0+a1`.
4. The resulting polyline "zigzags" between outer and inner rails — an acceptable approximation for rendering purposes.

**Line segments:** Straight segments (`SEG_STRLIN`) simply add two new points (start/end) unless they are collinear with the previous segment, in which case only one point is needed.

---

### `SliceCuts(angle a, DIST_T radius)` — Determine Number of Arc Slices

```c
static long SliceCuts(ANGLE_T a, DIST_T radius) {
    double Error = 0.05;
    double Error_angle = acos(1-(Error/fabs(radius)));
    if (Error_angle < 0.0001) return 0;
    return (long)(floor(D2R(a)/(2*Error_angle)));
}
```

This computes how many equal angular intervals are needed to approximate an arc of total angle `a` and radius `radius` within a chord-length error tolerance of 0.05 pixels (half a pixel, ensuring sub-pixel accuracy). The formula comes from the chord length formula:
\[ \text{chord} = 2r\sin(\theta/2) \approx r \cdot (\theta/2)^2 / 2 \]
for small angles, leading to an error bound of `Error_angle ≈ sqrt(Error / radius)`.

The function returns the number of *intervals*, so if you have an arc spanning angle `a` total, you need that many steps plus one more point for each interval (since there are N+1 points in N intervals). The caller adds 1 to account for the closing point.

---

### `CreateOriginAnchor(coOrd origin, wBool_t trans_selected)` — Draw Anchor Handles Around a Draw Object's Origin

Creates four small circular handles (two vertical lines forming a cross) around the Draw object's stored origin point. These are drawn in blue and allow the user to adjust the origin position by dragging. The anchors are only shown when the object is selected or being edited.

This is used during editing: if the user clicks on a Draw object, small blue handles appear around its stored origin (the point that defines "0,0" for that object). Dragging them repositions the origin while keeping the shape's geometry consistent relative to it.

---

### `DistanceDraw(track_p t, coOrd *p)` — Raycast Distance Function for Draw Objects

This function is called during raycasting (selection) to determine how far a ray from position `p` travels before intersecting a Draw object. It returns the distance along the ray to the intersection point.

**Short-circuit rules:**
- If `t == ignoredTableEdge` or `t == ignoredDraw`, return infinity — these are special objects that should not be selectable (e.g., layout edges, the "ignore" draw used during selection mode).
- Otherwise, compute distance by checking each segment:
  - For a polyline: use standard line-segment intersection.
  - For filled polygons: also check if the ray enters/exits the polygon interior (filled vs. non-filled distinction handled via `segPtr->u.p.polyType`).

**Why is this needed?** During selection mode, XTrkCAD draws a crosshair at the cursor position and casts a ray in all directions to see which objects are under the mouse. The distance function tells it how far along each ray direction an object lies. The closest intersection across all rays determines what's "under the cursor."

---

## Design Decisions & Tradeoffs

### Why Use `extraDataDraw_t` with a Variable-Length Segment Array?

Draw objects can have arbitrary numbers of segments (e.g., a polygon with 12 vertices, or a polyline with 45 segments). By storing them in a dynamically allocated array within the track object itself, the Draw type avoids needing a fixed-size struct. This is more memory-efficient than allocating a separate `segs` structure each time — the segment data lives right next to the track header on the heap.

### Why Have Both `orig` and `angle` as Separate Fields?

The `orig` field stores an absolute coordinate that serves as a pivot for transformations. The `angle` field is a global rotation applied to all segments simultaneously. Separating them allows:
- **Localized editing:** You can move the origin point independently of rotating the whole shape.
- **Grouping:** Multiple Draw objects could share the same origin and angle (e.g., a group of signs along a street).

### Why Use `lock_origin` Instead of Always Locking?

When creating a new Draw object, its origin is initially `{0,0}` relative to itself. If you immediately set it to an absolute coordinate like `(100, 200)`, then editing endpoint coordinates would move the whole shape away from that point. The `lock_origin` flag tells the update logic: "keep this origin fixed in viewport space while I adjust other parameters." Without locking, moving endpoints would recenter everything around a new origin — which is usually not what the user wants when editing an existing object.

### Why Deduplicate Segment Endpoints When Merging?

When multiple segments are selected and merged into one polyline, shared vertices should appear only once in the vertex list. Otherwise the renderer would draw redundant points (no visual harm), but it wastes memory and could cause issues if later operations assume a clean vertex list. The `IsClose()` tolerance handles floating-point drift from Bézier approximations — without it, tiny gaps between adjacent segments might create spurious extra vertices.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `MakeDrawFromSeg(trkSeg_p)` | Create a Draw track object from a segment descriptor | segment pointer, hit position, hit angle |
| `ComputeDrawBoundingBox(track_p t)` | Compute tight AABB for selection hilites | track pointer |
| `MakePolyLineFromSegs(...)` | Merge multiple segments into one polyline with deduplication | origin, global angle, segment array |
| `SliceCuts(a, radius)` | Compute number of angular slices needed for arc approximation | total angle, radius, error tolerance |
| `CreateOriginAnchor(coOrd p, wBool_t trans_selected)` | Draw blue cross-handle at the object's stored origin | position, whether transformation is currently active |
| `DistanceDraw(track_p t, coOrd *p)` | Raycast distance for selection under cursor | track pointer, ray endpoint |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Create and render Draw objects (arbitrary 2D shapes) from segment descriptors; compute bounding boxes; support editing of endpoints, origin, rotation, fill style, line width, text labels, etc. |
| **Domain** | Free-form polygon rendering, hit-testing for selection, interactive shape editing via property dialogs |
| **Key concept** | A Draw object is a collection of `trkSeg_t` descriptors stored in an extra-data structure; the `orig` field serves as a transformation pivot point that can be locked or unlocked during editing |
| **Main entry points** | `MakeDrawFromSeg()` — creates a new Draw track from a selected segment; `ComputeDrawBoundingBox()` — computes AABB for rendering and selection; `DistanceDraw()` — used by the raycaster during object picking |
