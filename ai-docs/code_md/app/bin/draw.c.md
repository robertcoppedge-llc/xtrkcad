# draw.c — Drawing Engine, Rulers, Zoom/Pan, Mouse Handling

## Overview

`draw.c` is the **core drawing engine** for XTrkCAD. It handles:

- Rendering track segments (lines, arcs, strings, polygons, fills) via a deferred-command system (`tempSegs`)
- Main window and map window rendering with background tiles, snap grids, and rulers
- Zoom/pan operations including mouse-wheel zooming, drag-to-pan, extent-based zoom, and preset zoom levels
- Mouse event dispatch for track drawing (click-drag-to-create-track-segments)
- Ruler/tick marking on the layout canvas

---

## Core Structures

### Drawing Command (`drawCmd_t`)

```c
typedef struct {
    drawFuncs_t * drawFuncs;  // Array of function pointers: DrawLine, DrawArc, DrawString, etc.
    drawOptions_e options;    // DC_TICKS|DC_TEMP|DC_SIMPLE — drawing mode flags
    DIST_T scale;             // Current zoom scale (1.0 = full scale)
    DIST_T dpi;               // DPI of the display device
    coOrd orig;               // Origin offset for the drawing area
    coOrd size;               // Viewport size in world coordinates
    Pix2CoOrd pix2coord;      // Conversion function: pixel → world coords
    CoOrd2Pix coord2pix;      // Inverse conversion
    wDraw_p d;                // Underlying GTK drawing context (wlib window)
} drawCmd_t;
```

Three instances exist:

| Instance | Purpose | Key Settings |
|----------|---------|--------------|
| `mainD`   | Main layout canvas | `DC_TICKS`, scale from preferences, `Pix2CoOrd`/`MainCoOrd2Pix` conversion |
| `tempD`   | Temporary drawing (cursor preview) | Same as mainD but with `DC_TEMP` option |
| `mapD`    | Map window (room-scale view) | `DC_SIMPLE`, smaller default scale, different DPI handling |

---

### Drawing Options (`drawOptions_e`)

```c
typedef enum {
    DC_NONE      = 0,
    DC_TICKS     = 1 << 0,   // Draw axis ticks and ruler labels (bottom/left only)
    DC_TEMP      = 1 << 1,   // Temporary drawing mode (cursor preview, undoable)
    DC_SIMPLE    = 1 << 2    // Simple mode: no ticks, no rulers, faster rendering
} drawOptions_e;
```

- **`DC_TICKS`** — Standard main window rendering with axis rulers.
- **`DC_TEMP`** — Used for cursor preview (ghost track segments) and undo buffers. Does not draw tick marks or ruler labels to reduce overhead during dragging.
- **`DC_SIMPLE`** — Map window uses this mode: no ticks, no rulers, just the room boundary.

---

### Drawing Functions (`drawFuncs_t`)

```c
typedef struct {
    drawFunc (*DrawLine)(  drawCmd_p d, coOrd p0, coOrd p1, wDrawWidth width, wDrawColor color );
    drawFunc (*DrawArc)(   drawCmd_p d, coOrd p, DIST_T r, ANGLE_T angle0, ANGLE_T angle1,
                            BOOL_T drawCenter, wDrawWidth width, wDrawColor color );
    drawFunc (*DrawString)( drawCmd_p d, coOrd p, ANGLE_T a, char * s,
                            wFont_p fp, FONTSIZE_T fontSize, wDrawColor color );
    drawFunc (*DrawBitMap)( drawCmd_p d, coOrd p, wDrawBitMap_p bm, wDrawColor color );
    drawFunc (*DrawPoly)(   drawCmd_p d, int cnt, coOrd * pts, wDrawWidth width, wDrawColor color,
                            drawFill_e fillOpt );
    drawFunc (*DrawFillCircle)( drawCmd_p d, coOrd p, DIST_T r, wDrawColor color );
    drawFunc (*DrawRectangle)( drawCmd_p d, coOrd orig, coOrd size, wDrawColor color, drawFill_e eOpts );
} drawFuncs_t;
```

Each function pointer maps a drawing primitive to its rendering implementation. The `drawFunc` type is a typedef for `void (*)(...)`.

---

### Deferred Segment Array (`tempSegs`)

The deferred-command pattern avoids calling GTK's drawing API directly from the mouse-move loop. Instead, operations are queued into an array and flushed once per frame (or on user-requested redraw).

```c
typedef struct {
    segType_e type;          // SEG_LINE, SEG_ARC, SEG_TEXT, SEG_POLY, SEG_FILPOLY, SEG_FILCRCL
    wDrawColor color;        // Color for this segment
    int lineWidth;           // Stroke width in pixels (0 = no stroke)
    union {
        coOrd center;       // For arcs/circles
        DIST_T radius;
        ANGLE_T a0, a1;     // Arc start/end angles

        char * string;      // Text string for SEG_TEXT
        ANGLE_T angle;
        wFont_p fontP;
        FONTSIZE_T fontSize;

        int cnt;            // Number of vertices in polygon
        polyType_e polyType;  // POLYLINE or FREEFORM (open vs closed)
        coOrd pts[1];       // Vertex array (points to allocated memory)

        DIST_T radius;      // For fill circles
    } u;
} trkSeg_t;

/* Max ~80 segments per frame */
DYNARR( trkSeg_t, tempSegs_da )
```

The `tempSegs` array grows dynamically (max size 80). After each mouse move or drag segment is completed:

1. A new element is appended via `DYNARR_APPEND(tempSegs_da, N)`
2. The appropriate field (`u.c.*`, `u.t.*`, etc.) is populated based on the segment type
3. At rendering time, all queued segments are drawn in order by calling their respective function pointers from the `tempSegDrawFuncs` table

---

### Drawing Function Table (`tempSegDrawFuncs`)

```c
EXPORT drawFuncs_t tempSegDrawFuncs = {
    TempSegLine,
    TempSegArc,
    TempSegString,
    NoDrawBitMap,          // Bitmap not supported for cursor preview
    TempSegPoly,
    TempSegFillCircle,
    TempSegRectangle
};
```

Each function takes a `drawCmd_p` (the target drawing context), and writes directly into the shared `tempSegs_da` array. This avoids allocating new segments per frame — the array is reused across frames via dynamic array growth/shrinking.

---

## Key Functions

### Creating Track Segments

#### Circle Arc Segment

```c
static void TempSegArc(
        drawCmd_p d,
        coOrd p,           // Center of arc
        DIST_T r,          // Radius
        ANGLE_T angle0,    // Start angle (radians)
        ANGLE_T angle1,    // End angle
        BOOL_T drawCenter, // Whether to draw the center point
        wDrawWidth width,  // Line width (negative = use default)
        wDrawColor color )
{
    DYNARR_APPEND( trkSeg_t, tempSegs_da, 10 );
    tempSegs(tempSegs_da.cnt-1).type = SEG_CRVLIN;
    tempSegs(tempSegs_da.cnt-1).color = color;

    if (d->options & DC_SIMPLE) {
        tempSegs(tempSegs_da.cnt-1).lineWidth = 0;
    } else if (width < 0) {
        tempSegs(tempSegs_da.cnt-1).lineWidth = width;
    } else {
        tempSegs(tempSegs_da.cnt-1).lineWidth = width * d->scale / d->dpi;
    }

    tempSegs(tempSegs_da.cnt-1).u.center = p;
    tempSegs(tempSegs_da.cnt-1).u.radius = r;
    tempSegs(tempSegs_da.cnt-1).u.a0   = angle0;
    tempSegs(tempSegs_da.cnt-1).u.a1   = angle1;
}
```

Notes:

- The line width is converted from **user pixels** to **world units** via `width * scale / dpi`. This means a 2-pixel-wide rail at full scale (scale=1.0, dpi=96) has `lineWidth = 2` world units.
- In simple mode (`DC_SIMPLE`), line widths are forced to zero — used for the map window where fine detail isn't needed.

---

#### Text Segment

```c
static void TempSegString(
        drawCmd_p d,
        coOrd p,
        ANGLE_T a,
        char * s,
        wFont_p fp,
        FONTSIZE_T fontSize,
        wDrawColor color )
{
    DYNARR_APPEND( trkSeg_t, tempSegs_da, 10 );
    tempSegs(tempSegs_da.cnt-1).type = SEG_TEXT;
    tempSegs(tempSegs_da.cnt-1).color = color;
    tempSegs(tempSegs_da.cnt-1).u.t.boxed = FALSE;      // Not boxed (no background)
    tempSegs(tempSegs_da.cnt-1).lineWidth = 0;          // No stroke
    tempSegs(tempSegs_da.cnt-1).u.t.pos   = p;
    tempSegs(tempSegs_da.cnt-1).u.t.angle = a;
    tempSegs(tempSegs_da.cnt-1).u.t.fontP  = fp;
    tempSegs(tempSegs_da.cnt-1).u.t.fontSize = fontSize;
    tempSegs(tempSegs_da.cnt-1).u.t.string = MyStrdup(s);   // Owned by caller, freed later?
}
```

The `SEG_TEXT` segment carries a boxed string (`MyStrdup`) — the memory is **not** automatically freed after rendering. This is intentional: text strings are kept in the undo buffer and written back to track pieces during redo.

---

#### Polygon / Polyline Segment

```c
static void TempSegPoly(
        drawCmd_p d,
        int cnt,
        coOrd * pts,
        int * types,       // NULL or array of edge types (straight/curve)
        wDrawColor color,
        wDrawWidth width,
        drawFill_e eFillOpt )  // DRAW_OPEN, DRAW_CLOSED, DRAW_FILL, DRAW_TRANSPARENT
{
    int fill = 0;
    int open = 0;

    switch (eFillOpt) {
        case DRAW_OPEN:     open = 1; break;
        case DRAW_CLOSED:   break;
        case DRAW_FILL:     fill = 1; break;
        case DRAW_TRANSPARENT: fill = 1; break;
        default: CHECK(FALSE);
    }

    DYNARR_APPEND( trkSeg_t, tempSegs_da, 1);

    /* SEG_FILPOLY for filled polygons, SEG_POLY for outlines */
    tempSegs(tempSegs_da.cnt-1).type = fill ? SEG_FILPOLY : SEG_POLY;
    tempSegs(tempSegs_da.cnt-1).color   = color;

    if (d->options & DC_SIMPLE) {
        tempSegs(tempSegs_da.cnt-1).lineWidth = 0;
    } else if (width < 0) {
        tempSegs(tempSegs_da.cnt-1).lineWidth = width;
    } else {
        tempSegs(tempSegs_da.cnt-1).lineWidth = width * d->scale / d->dpi;
    }

    /* POLYLINE: edges are straight or curved; FREEFORM: arbitrary polygon */
    tempSegs(tempSegs_da.cnt-1).u.p.polyType = open ? POLYLINE : FREEFORM;
    tempSegs(tempSegs_da.cnt-1).u.p.cnt     = cnt;
    tempSegs(tempSegs_da.cnt-1).u.p.orig   = zero;  // No rotation offset yet
    tempSegs(tempSegs_da.cnt-1).u.p.angle  = 0.0;

    /* Allocate and copy vertex array */
    tempSegs(tempSegs_da.cnt-1).u.p.pts = (pts_t *)MyMalloc(cnt * sizeof(pts_t));

    for (int i=0; i<=cnt-1; i++) {
        tempSegs(tempSegs_da.cnt-1).u.p.pts[i].pt         = pts[i];
        /* If not in simple mode and types is provided, use the edge type */
        tempSegs(tempSegs_da.cnt-1).u.p.pts[i].pt_type    = ((d->options & DC_SIMPLE) == 0 && (types != 0)) ? types[i] : wPolyLineStraight;
    }
}
```

Notes:

- `polyType` distinguishes between a **POLYLINE** (open path, no automatic closure) and **FREEFORM** (closed polygon). This matters for rendering — closed polygons get filled.
- The vertex array (`u.p.pts`) is allocated with `MyMalloc`. After the deferred draw flushes this segment, the caller must free it. This is handled by the drawing engine via a cleanup callback registered in `drawCmd_t`.

---

#### Filled Circle

```c
static void TempSegFillCircle(
        drawCmd_p d,
        coOrd p,       // Center
        DIST_T r,      // Radius
        wDrawColor color )
{
    DYNARR_APPEND( trkSeg_t, tempSegs_da, 10 );
    tempSegs(tempSegs_da.cnt-1).type = SEG_FILCRCL;  // Filled circle
    tempSegs(tempSegs_da.cnt-1).color = color;
    tempSegs(tempSegs_da.cnt-1).lineWidth = 0;       // No stroke
    tempSegs(tempSegs_da.cnt-1).u.center = p;
    tempSegs(tempSegs_da.cnt-1).u.radius = r;
    tempSegs(tempSegs_da.cnt-1).u.a0   = 0.0;
    tempSegs(tempSegs_da.cnt-1).u.a1   = 360.0;     // Full circle
}
```

---

#### Rectangle (via Poly)

Rectangles are drawn using the polygon machinery, with four vertices and a closed shape:

```c
static void TempSegRectangle(
        drawCmd_p d,
        coOrd orig,    // Bottom-left corner
        coOrd size,    // Width × height in world units
        wDrawColor color,
        drawFill_e eOpts )  // Filled or outline only
{
    coOrd p[4];

    /* Build four corners (p1+p2 at bottom, p0+p3 at top) */
    p[0].x = p[1].x = orig.x;      // left side
    p[2].x = p[3].x = orig.x + size.x;  // right side
    p[0].y = p[3].y = orig.y;       // bottom
    p[1].y = p[2].y = orig.y + size.y;  // top

    TempSegPoly( d, 4, p, NULL, color, 0, eOpts );
}
```

---

### Drawing the Main Window

`MainRedraw()` is the central rendering function for the main layout canvas. It:

1. Clears the drawing context
2. Draws a background rectangle (light gray) inside a darker room boundary
3. Draws the snap grid (`DrawSnapGrid`)
4. Draws all track segments via `DrawTracks()`
5. Draws markers, playback cursor, and rulers

```c
EXPORT void MainRedraw( void )
{
    coOrd orig, size;
    static int cMR = 0;
    unsigned long time0 = wGetTimer();

    if (delayUpdate) {
        wDrawDelayUpdate( mainD.d, TRUE );   // Queue redraw to next frame
    }

    wDrawSetTempMode( mainD.d, FALSE );       // Exit temp-draw mode
    wDrawClear( mainD.d );                    // Clear background

    orig = mainD.orig;
    size = mainD.size;
    orig.x -= LBORDER / mainD.dpi * mainD.scale;   // Clip to canvas borders
    orig.y -= BBORDER / mainD.dpi * mainD.scale;

    DrawRoomWalls( TRUE );  // Background gray rectangle + room outline
    if (GetLayoutBackGroundScreen() < 100.0 && GetLayoutBackGroundVisible()) {
        wWinPix_t bitmapPosX, bitmapPosY, bitmapWidth;
        TranslateBackground(&mainD, orig.x, orig.y, &bitmapPosX, &bitmapPosY, &bitmapWidth);
        wDrawShowBackground(mainD.d, bitmapPosX, bitmapPosY, bitmapWidth, ... );
    }

    DrawSnapGrid( &mainD, mapD.size, TRUE );  // Snap grid (optional)

    /* Redraw tracks */
    orig = mainD.orig;
    size = mainD.size;
    orig.x -= RBORDER / mainD.dpi * mainD.scale;
    orig.y -= BBORDER / mainD.dpi * mainD.scale;
    size.x += (RBORDER+LBORDER)/mainD.dpi*mainD.scale;
    size.y += (BBORDER+TBORDER)/mainD.dpi*mainD.scale;
    DrawTracks( &mainD, mainD.scale, orig, size );

    DrawRoomWalls( FALSE );  // Redraw room walls without background (just rulers)

    currRedraw++;            // Increment redraw counter for undo tracking

    wDrawSetTempMode( tempD.d, TRUE );   // Enter temp-draw mode
    DoCurCommand( C_REDRAW, zero );      // Draw temporary segments (cursor preview)
    DrawMarkers();                      // Marker icons (e.g. turnouts)
    RulerRedraw( FALSE );               // Redraw ruler labels on top of track lines
    RedrawPlaybackCursor();             // If in playback mode

    wDrawSetTempMode( tempD.d, FALSE );  // Exit temp-draw
    wDrawDelayUpdate( mainD.d, FALSE );   // Flush queued drawing to screen
}
```

The `currRedraw` counter is used by the undo system: each redraw increments it so that subsequent track edits are recorded as new "redraw states" in the history.

---

### Map Window Redraw (`MapRedraw`)

The map window is rendered at a different scale and size than the main canvas. It uses `DC_SIMPLE` (no ticks, no rulers) and has its own zoom mechanism based on room dimensions:

```c
static void MapRedraw( wDraw_p bd, void *pContext, wWinPix_t px, wWinPix_t py )
{
    if (inPlaybackQuit) { return; }
    static int cMR = 0;
    LOG( log_redraw, 2, ("MapRedraw: %d\n", cMR++) );

    if (!mapVisible) { return; }
    if (delayUpdate) {
        wDrawDelayUpdate( mapD.d, TRUE );
    }

    /* Find new scale based on window size and room size */
    if ((px <= 0 || py <= 0) && mapD.d) {
        wDrawGetSize( mapD.d, &px, &py );
        px += 2; py += 2;
    }

    if (px > 0 && py > 0) {
        FLOAT_T scaleX = mapD.size.x * mapD.dpi / px;
        FLOAT_T scaleY = mapD.size.y * mapD.dpi / py;
        FLOAT_T scale = max(scaleX, scaleY);

        /* Clamp to min/max */
        if (scale > MAX_MAIN_SCALE) { scale = MAX_MAIN_SCALE; }
        if (scale < MIN_MAIN_MACRO)  { scale = MIN_MAIN_MACRO; }

        scale = ceil(scale);   // Round up to nearest preset
        mapD.scale = scale;
    }

    wDrawClear( mapD.d );
    DrawTracks( &mapD, mapD.scale, mapD.orig, mapD.size );
    DrawMapBoundingBox( TRUE );  // Highlight the main window's view rectangle

    wDrawSetTempMode( mapD.d, FALSE );
}
```

The map scale is computed so that the room (whose size is stored in `mapD.size`) fits within the window. The resulting scale is clamped to a discrete set of values (`MIN_MAIN_MACRO` through `MAX_MAIN_SCALE`). This allows switching between "room view" and "full-scale layout view".

---

### Ruler Drawing (`DrawRuler`)

The ruler function draws tick marks along an axis, with labels at major intervals. It supports both **metric** (mm) and **English** (inches/feet) units and scales the spacing of ticks based on zoom level:

```c
static void DrawRuler(
        drawCmd_p d,
        coOrd pos0,      // Start point of ruler axis
        coOrd pos1,      // End point of ruler axis
        DIST_T offset,   // Perpendicular offset (for bottom vs top rail)
        int number,       // Whether to show all labels or only major ones
        int tickSide,    // 0 = above/left, 1 = below/right
        wDrawColor color )
{
    coOrd orig = pos0;
    wAngle_t a, aa;
    DIST_T start, end;

    /* Rotate so ruler is horizontal for drawing purposes */
    a   = FindAngle( pos0, pos1 );
    Translate(&pos0, pos0, a, offset);
    Translate(&pos1, pos1, a, offset);
    aa  = NormalizeAngle(a + (tickSide==0 ? +90 : -90));

    end = FindDistance(pos0, pos1);
    if (end < 0.1) { return; }

    /* Clip to viewport */
    coOrd d_orig, d_size;
    d_orig.x = d->orig.x - 0.1;
    d_orig.y = d->orig.y - 0.1;
    d_size.x = d->size.x + 0.2;
    d_size.y = d->size.y + 0.2;
    if (!ClipLine(&pos0, &pos1, d_orig, d->angle, d_size)) { return; }

    start = FindDistance(orig, pos0);
    end   = FindDistance(orig, pos1);

    /* Draw the baseline */
    DrawLine( d, pos0, pos1, 3, wDrawColorWhite );
    DrawLine( d, pos0, pos1, 0, color );

    if (units == UNITS_METRIC) {
        /* Metric ruler: mm ticks */
        int mm0 = (int)ceil(start*25.4 - 0.5);
        int mm1 = (int)floor(end*25.4 + 0.5);

        /* Choose tick interval based on scale */
        DIST_T len;   // Distance between ticks in pixels
        int power;    // The power of 10 for the current decade (1, 10, 100, 1000)

        if (d->scale <= 1.0)      { power = 1; }
        else if (d->scale <= 8)   { power = 10; }
        else if (d->scale <= 32)  { power = 100; }
        else                       { power = 1000; }

        /* Tick interval in mm */
        int len_mm;
        if (power == 1)    len_mm = 5;   // every 5mm at full scale
        else if (power==10){ len_mm = 50; }
        else if (power==100){ len_mm = 200; }
        else               { len_mm = 500; }

        /* Label interval for larger scales */
        int skip_mm;
        if (d->scale <= 200) { skip_mm = 2000; }   // label every 2m at 1:200
        else if (d->scale <= 400){ skip_mm = 5000;}
        else                  { skip_mm = 10000;}

        for (int mm=mm0; mm<=mm1; mm+=power) {
            /* Skip some ticks at large scales */
            if (!number || (d->scale > 40 && mm % skip_mm != 0)) continue;

            coOrd p0, p1;
            Translate(&p0, orig, a, mm / 25.4);   // Convert world → pixels

            /* Tick length in pixels */
            if (power == 1) len = 3;   // short tick for minor marks
            else           len = 6;     // longer tick for major marks

            Translate(&p0, p0, aa, len * d->scale / mainD.dpi);
            DrawLine( d, pos0, p0, 0, color );      // Tick line perpendicular to axis
            if (number == FALSE) continue;           // no label on minor ticks

            /* Label at larger intervals */
            if ((power >= 1000) || (d->scale <= 8 && power >= 100) || ...) {
                char buf[32];
                sprintf(buf, "%ld", mm / 10);   // e.g. "2" for 20mm
                Translate(&p0, p0, aa, len * d->scale / mainD.dpi + fontSize/2);
                DrawString(d, p0, buf, font, fontSize, color);
            }
        }
    } else {
        /* English ruler: inches and fractions */
        int incr;  // Smallest fraction to show (16ths at full scale)

        if      (d->scale >= 1.0)   incr = 1;       // 1/16" ticks
        else if (d->scale <= 3)     incr = 2;       // 1/8"
        else if (d->scale <= 5)     incr = 4;       // 1/4"
        else if (d->scale <= 7)     incr = 8;       // 1/2"
        else if (d->scale <= 48)    incr = 32;      // 1/32"
        else                         incr = 16;      // inches only

        /* Determine which inch to start labeling from */
        int lastInch     = (int)floor(end);
        int firstFraction = (((int)((inch - start)*16)) / incr) * incr;

        for (int inch=ceil(start); inch<=lastInch+incr; inch++) {
            /* Special handling for foot marks at 12-inch intervals */
            if (inch % 12 == 0) {
                DrawNumber(inch/12, "'");   // "1'" or "2'", etc.
            } else {
                DrawNumber(inch, '"');       // "3"" or "4"", etc.
            }

            /* Fractional ticks within this inch */
            for (int frac = 0; frac <= 16; frac += incr) {
                coOrd p0, p1;
                Translate(&p0, orig, a, inch + frac/16.0);
                Translate(&p1, p0, aa, length[frac] * d->scale / mainD.dpi);

                DrawLine( d, pos0, p1, 3, wDrawColorWhite );   // white tick for visibility
                DrawLine( d, pos0, p1, 0, color );             // colored overlay

                if (frac == 0) {
                    /* Label the inch mark */
                    char buf[8];
                    sprintf(buf, "%d", inch);
                    Translate(&p0, p0, aa, len * d->scale / mainD.dpi + fontSize/2);
                    DrawString(d, p0, buf, font, fontSize, color);
                }
            }
        }
    }
}
```

Key observations:

- The ruler is drawn in **world coordinates**, not pixels. Tick positions are computed by converting millimeters or inches into pixel offsets using the current scale and DPI.
- Tick lengths (`len`) are chosen based on zoom level so that minor ticks don't clutter the view at large scales but remain visible when zoomed out.
- The `number` flag controls whether **all** tick marks get labels, or only major ones (e.g., every 5mm in metric mode).

---

### Zoom and Pan Controls

#### Zoom Levels (`zoomList`)

```c
static struct {
    DIST_T value;       // Scale factor (1.0 = full scale)
    char * name;        // Label shown in menu
} zoomList[] = {
    { 256.0, "256:1" },
    { 128.0, "128:1" },
    { 64.0,  "64:1" },
    { 32.0,  "32:1" },
    { 16.0,  "16:1" },
    { 8.0,   "8:1" },
    { 4.0,   "4:1" },
    { 2.0,   "2:1" },
    { 1.5,   "1.5:1" },
    { 1.33333, "1.33:1" },
    { 1.25,  "1.25:1" },
    { 1.0,   "1:1" },
    { 0.75,  "0.75:1" },
    { 0.625, "0.625:1 (3/4)" },
    { 0.5,   "1:2" },
    { 0.4,   "1:2.5" },
    { 0.33333, "1:3" },
    { 0.25,  "1:4" },
    { 0.2,   "1:5" },
    { 0.16666, "1:6" },
};

/* Macros to clamp scale */
#define MAX_MAIN_SCALE  256.0
#define MIN_MAIN_MACRO  0.125
```

These predefined levels are used by the zoom buttons and menu items. The `DoZoomUp` / `DoZoomDown` functions find the nearest level and apply it, clamping to min/max bounds.

#### Preset Zoom Levels

Users can store a custom "zoom in" value (e.g., 80:1) for quick access:

```c
wPrefGetInteger("misc", "zoomin", &newScale, 4);   // Default preset = 4x
DoNewScale(newScale);
```

The current zoom scale is persisted to `"draw/zoom"` so that reopening the program restores the last used magnification.

---

#### Pan Popup Menu (`InitCmdPan`)

The pan mode popup menu provides keyboard shortcuts and quick-access actions:

```c
EXPORT void InitCmdPan2( wMenu_p menu )
{
    /* Zoom extents (to selected objects or full room) */
    zoomExtents = wMenuPushCreate(panPopupM, "", "Zoom to extents - 'e'", 0, PanMenuEnter, I2VP('e'));

    /* Preset zoom levels (1–9) */
    zoomLvl1  = wMenuPushCreate(panPopupM, "", "Zoom to 1:1 - '1'");
    zoomLvl2  = wMenuPushCreate(panPopupM, "", "Zoom to 1:2 - '2'");
    // ... up through 1:9

    /* Pan to origin or center */
    panOrig   = wMenuPushCreate(panPopupM, "", "Pan to Origin - 'o'/'0'", 0, PanMenuEnter, I2VP('o'));
    panHere   = wMenuPushCreate(panPopupM, "", "Pan center here - 'c'", 0, PanHere, I2VP(3));

    /* Additional zoom levels */
    zoomLvl9  = wMenuPushCreate(...);

    InitCmdZoom(NULL, NULL, NULL, zoomPanM);   // Register zoom radio buttons in the menu
}
```

Keyboard shortcuts:

| Key | Action |
|-----|--------|
| `e` | Zoom to extents (selected objects or full room) |
| `o` / `0` | Center pan on origin (0, 0) |
| `c` | Pan center to mouse pointer position |
| `1–9` | Jump to preset zoom level n:1 |

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `TempSegLine()` | Queue a line segment for deferred drawing | Writes into shared `tempSegs_da` array |
| `TempSegArc()` | Queue an arc segment | Converts user pixel width → world units via scale/DPI |
| `TempSegString()` | Queue text (label) on the layout | Uses `MyStrdup` — caller must free after undo flush |
| `TempSegPoly()` | Queue a polygon/polyline (open or closed) | Allocates vertex array; used for bridges, tunnels, custom shapes |
| `DrawRoomWalls(TRUE/FALSE)` | Draw background room boundary + optional background bitmap | Called from `MainRedraw` to layer graphics |
| `TranslateBackground()` | Compute position/size of tiled background image | Used when a photo or texture is applied to the layout area |
| `MapRedraw()` | Render the map window at appropriate zoom level | Scale is clamped; uses `DC_SIMPLE` mode |
| `DrawRuler()` | Draw tick marks and labels on an axis | Supports metric (mm) and English (inches/feet); adaptive tick spacing based on scale |
| `DoZoomUp()/Down()` | Adjust zoom by one level up or down | Snaps to nearest preset in `zoomList[]`; clamps to min/max |
| `DoZoomExtents()` | Zoom so selected objects fit within the window | Computes a new scale that fits the bounding box of all selected track pieces |
| `CmdPan()` | Pan/zoom command handler | Supports drag-to-pan, Ctrl+drag-to-zoom, keyboard shortcuts |

---

## Domain & Design Notes

### Deferred Command Pattern

The drawing engine uses a **deferred-command** pattern instead of calling GTK's drawing API directly from the mouse-move event loop. This avoids:

- Excessive `wDraw()` calls that would block the UI thread
- Drawing partially-formed tracks (the user sees only completed segments)
- Undo/redo complexity — each "step" is a complete track piece, not every intermediate pixel movement

Segments are queued into `tempSegs_da` and flushed once per frame via `wDrawDelayUpdate()`. The undo system records the contents of this array after each successful command (e.g., placing a turnout), so that if the user undoes the placement, the segments can be re-rendered from the previous state.

### Temp Draw Mode (`DC_TEMP`)

When `wDrawSetTempMode()` is called with `TRUE`, subsequent drawing calls write directly to the temp-draw context instead of the main canvas. This is used for:

- **Cursor preview** — as the user drags a new track segment, temporary arcs are drawn in the temp context and then composited on top of the main view
- **Undo buffers** — each command's "redo" data is stored by drawing its segments into the temp buffer so they can be re-rendered during redo

The flag is toggled via `wDrawSetTempMode(d, TRUE/FALSE)` before/after flushing.

### Border Clipping

Track drawing uses a border offset:

```c
orig.x -= RBORDER / mainD.dpi * mainD.scale;   // Clip right border
orig.y -= BBORDER / mainD.dpi * mainD.scale;   // Clip bottom border
```

This ensures that track pieces don't extend beyond the visible window, even if their world coordinates place them partially outside. The `RBORDER`/`BBORDER` constants define how far off-canvas is allowed before clipping kicks in.

### Background Bitmap Support

The map and main windows support a background image (e.g., a photograph of the physical room). The bitmap is tiled to fill the view, scaled by the current zoom level, and its position is computed relative to the layout origin (`GetLayoutBackGroundPos()`). This allows users to overlay a real photo onto their CAD model for verification.
