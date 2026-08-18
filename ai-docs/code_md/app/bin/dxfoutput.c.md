# dxfoutput.c — DXF Export (Drawing to File)

## Overview

`dxfoutput.c` implements the **DXF export functionality** for XTrkCAD. It takes selected track segments from the current layout and writes them into a properly formatted `.dxf` file, using the drawing engine (`drawCmd_t`) as a generic output stream — essentially repurposing the main canvas's drawing context to write directly to a file instead of to an on-screen window.

---

## Key Concepts

- **Drawing command table** — each primitive (line, arc, text) has its own function that knows how to format DXF group codes and write them to a `FILE *`.
- The main drawing context (`drawCmd_t`) is reused: instead of pointing to the GTK window handle (`wDraw_p`), it points to an open `FILE *` stream. All `DrawLine()`, `DrawArc()`, etc., calls are redirected to this file via function pointers in a table.
- The export process follows the same rendering pipeline as normal drawing, but all output goes into a buffered string that is finally written to disk.

---

## Global State

```c
static struct wFilSel_t *exportDXFFile_fs;  /* File chooser for saving DXF files */
static drawCmd_t dxfD = { ... };             /* Reused draw context, redirected to file */
```

The `dxfD` instance is a **dummy drawing command** whose only purpose is to hold the open `FILE *` stream. Its `drawFuncs` table points at the DXF-specific implementation functions (`DxfLine`, `DxfArc`, etc.).

---

## Per-Primitive Export Functions

Each primitive has an exported counterpart that writes its DXF representation into a buffer, then calls `fputs()` to flush it:

### Line

```c
static void DxfLine(
        drawCmd_p d,
        coOrd p0,
        coOrd p1,
        wDrawWidth width,
        wDrawColor color)
{
    long c = wDrawGetRGB(color);   /* Convert GTK RGB (ARGB32) to DXF truecolor index */
    long s = d->options & DC_DASH ? 1 : (d->options & DC_DOT ? 2 : 0);

    DynString command = NaS;
    DynStringMalloc(&command, 100);
    DxfLineCommand(&command, curTrackLayer + 1, p0.x, p0.y, p1.x, p1.y, s, c);
    fputs(DynStringToCStr(&command), (FILE *)d->d);
    DynStringFree(&command);
}
```

- `curTrackLayer + 1` — the current track layer is used; DXF layers are 1-indexed.
- The line style (`s`) is derived from drawing options:
  - `DC_DASH` → linetype index **1** (DASHED)
  - `DC_DOT`   → linetype index **2** (DOT)
  - neither    → linetype index **0** (CONTINUOUS)

---

### Arc / Circle Detection

```c
static void DxfArc(
        drawCmd_p d,
        coOrd p, DIST_T r, ANGLE_T angle0, ANGLE_T angle1,
        BOOL_T drawCenter, wDrawWidth width, wDrawColor color)
{
    long c = wDrawGetRGB(color);
    long s = d->options & DC_DASH ? 1 : (d->options & DC_DOT ? 2 : 0);

    DynString command = NaS;
    DynStringMalloc(&command, 100);

    /* Normalize: DXF arcs are defined as start-angle + sweep-angle */
    angle0 = NormalizeAngle(90.0 - (angle0 + angle1));

    if (angle1 >= 360.0) {
        DxfCircleCommand(&command, curTrackLayer + 1, p.x, p.y, r, s, c);
    } else {
        DxfArcCommand(&command, curTrackLayer + 1, p.x, p.y, r, angle0, angle1, s, c);
    }

    fputs(DynStringToCStr(&command), (FILE *)d->d);
    DynStringFree(&command);
}
```

Notes:

- Arcs in XTrkCAD are stored as a **center point** + a **start/end angle pair**. DXF expects `angle0` = start, `angle1` = sweep. The normalization adjusts for the different internal representation.
- If the total sweep is ≥ 360°, it's emitted as a **CIRCLE** entity instead — DXF prefers that over an arc with a full 360° sweep.

---

### Text / Label

```c
static void DxfString(
        drawCmd_p d,
        coOrd p, ANGLE_T a, char * s,
        wFont_p fp, FONTSIZE_T fontSize, wDrawColor color)
{
    long c = wDrawGetRGB(color);

    DynString command = NaS;
    DynStringMalloc(&command, 100);
    DxfTextCommand(&command, curTrackLayer + 1, p.x, p.y, fontSize, s, c);
    fputs(DynStringToCStr(&command), (FILE *)d->d);
    DynStringFree(&command);
}
```

The text height is passed directly; no unit conversion is needed because `DxfTextCommand` already converts from millimeters to inches internally.

---

### Polygon / Polyline

```c
static void DxfPoly(
        drawCmd_p d, int cnt, coOrd *pts, int *types,
        wDrawColor color, wDrawWidth width, drawFill_e eOpts)
{
    for (int inx=1; inx<cnt; inx++) {
        DxfLine(d, pts[inx-1], pts[inx], width, color);
    }

    /* Close the polygon if not explicitly open */
    if (eOpts != DRAW_OPEN) {
        DxfLine(d, pts[cnt-1], pts[0], width, color);
    }
}
```

The function emits a chain of LINE entities for each edge. If `eOpts` is anything other than `DRAW_OPEN`, the closing segment from the last vertex back to the first is drawn — effectively turning an open polyline into a closed polygon.

---

### Filled Circle / Rectangle

- **Filled circle** → emitted as a CIRCLE with a full 360° sweep:
  ```c
  static void DxfFillCircle(drawCmd_p d, coOrd center, DIST_T radius, wDrawColor color)
  {
      DxfArc(d, center, radius, 0.0, 360, FALSE, 0, color);
  }
  ```

- **Rectangle** → emitted as a closed POLYGON:
  ```c
  static void DxfRectangle(drawCmd_p d, coOrd orig, coOrd size, wDrawColor color, drawFill_e eOpts)
  {
      coOrd p[4];
      /* Bottom-left (p0), bottom-right (p1), top-right (p3), top-left (p2) */
      p[0].x = p[1].x = orig.x;
      p[2].x = p[3].x = orig.x + size.x;
      p[0].y = p[3].y = orig.y;
      p[1].y = p[2].y = orig.y + size.y;

      DxfPoly(d, 4, p, NULL, color, 0, eOpts);
  }
  ```

---

## Drawing Command Table

```c
static drawFuncs_t dxfDrawFuncs = {
    DxfLine,
    DxfArc,
    DxfString,
    DxfBitMap,   /* empty stub — bitmaps are not written to DXF */
    DxfPoly,
    DxfFillCircle,
    DxfRectangle
};

static drawCmd_t dxfD = {
    NULL,                    /* No real drawFuncs table needed; this one is used */
    &dxfDrawFuncs,           /* Pointer to the DXF-specific function table */
    0,                       /* options flags (none for DXF output) */
    1.0,                     /* scale = 1.0: no zoom scaling for DXF export */
    0.0,                     /* origin offset */
    {0.0, 0.0},             /* size — not used when writing to a file */
    Pix2CoOrd,              /* Conversion function (unused in this context) */
    CoOrd2Pix,              /* Inverse conversion (unused) */
    100.0                   /* DPI — arbitrary; DXF doesn't use it */
};
```

Note that `dxfD.d` is never initialized to a real window handle — instead it's set directly to an open `FILE *`. The drawing engine's generic rendering loop calls these functions via the function-pointer table, and each one writes its output to the file stream.

---

## Export Workflow

```c
static int DoExportDXFTracks(int cnt, char **fileName, void *data)
{
    time_t clock;
    DynString command = NaS;
    FILE *dxfF;

    CHECK(fileName != NULL);
    CHECK(cnt == 1);   /* Only one filename is expected */

    DynStringMalloc(&command, 100);

    SetCurrentPath(DXFPATHKEY, fileName[0]);  /* Resolve to an absolute path */
    dxfF = fopen(fileName[0], "w");           /* Open for writing (text mode) */

    if (dxfF == NULL) {
        NoticeMessage(MSG_OPEN_FAIL, _("Continue"), NULL, "DXF", fileName[0], strerror(errno));
        return FALSE;
    }

    SetCLocale();                              /* Use C locale so numbers are ASCII-digits only */
    wSetCursor(mainD.d, wCursorWait);          /* Show wait cursor */
    time(&clock);                             /* Start timing the export */

    /* --- Write header + tables --- */
    DxfPrologue(&command, 10, 0.0, 0.0, mapD.size.x, mapD.size.y);
    fputs(DynStringToCStr(&command), dxfF);   /* Flush prologue to file */

    /* --- Write entity geometry --- */
    dxfD.d = (wDraw_p)dxfF;                    /* Redirect draw context to this file */

    DrawSelectedTracks(&dxfD, FALSE);          /* Render all selected segments */

    DynStringClear(&command);                  /* Clear buffer for footer */
    DxfEpilogue(&command);                     /* Append ENDSEC + EOF markers */
    fputs(DynStringToCStr(&command), dxfF);   /* Flush epilogue to file */

    fclose(dxfF);                              /* Close and flush the stream */
    SetUserLocale();                           /* Restore user's preferred locale */
    Reset();                                   /* Undo any temporary state changes */
    wSetCursor(mainD.d, defaultCursor);        /* Hide wait cursor */

    return TRUE;
}
```

**Step-by-step:**

1. **Open file** in text mode (`"w"`). The `FILE *` is stored directly in `dxfD.d`.
2. **Write prologue** — header section, unit settings, extents, line type and layer tables. This defines the drawing environment (e.g., which layers exist, what linetypes are available).
3. **Redirect draw context**: `dxfD.d = dxfF;` tells the drawing engine to write its output into this file instead of onto a GTK window.
4. **Render selected tracks** — calls to `DrawLine()`, `DrawArc()`, etc., now call their DXF counterparts via the `dxfDrawFuncs` table, which each write into `dxfD.d`.
5. **Write epilogue** — closes out the ENTITIES section and marks end-of-file.
6. **Close file**, restore locale, reset any side effects.

---

## File Chooser Dialog

```c
void DoExportDXF(void *unused)
{
    /* Optionally check: if (selectedTrackCount <= 0) { ... } */
    CHECK(selectedTrackCount > 0);   /* No tracks selected → error */

    if (exportDXFFile_fs == NULL) {
        exportDXFFile_fs = wFilSelCreate(mainW, FS_SAVE, 0,
                                          _("Export to DXF"), sDXFFilePattern,
                                          DoExportDXFTracks, NULL);
    }

    wFilSelect(exportDXFFile_fs, GetCurrentPath(DXFPATHKEY));
}
```

The file chooser is created lazily on first use. `GetCurrentPath(DXFPATHKEY)` seeds the dialog with a sensible default directory (e.g., `/home/user/.xtrkcad/dxf/`).

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `DxfLine()` | Emit LINE entity | Layer = current track layer + 1; linetype from drawing options (dashed/dotted) |
| `DxfArc()` | Emit ARC or CIRCLE | Converts start/end angles to DXF convention; full circle → CIRCLE entity |
| `DxfString()` | Emit TEXT entity | Height in mm is converted internally by `DxfTextCommand` |
| `DxfBitMap()` | Stub — no bitmap support | Empty function |
| `DxfPoly()` | Emit polyline / polygon chain | If not `DRAW_OPEN`, closes the shape with a final segment |
| `DxfFillCircle()` | Emit full-circle ARC (as CIRCLE) | 360° sweep → `DxfCircleCommand` |
| `DxfRectangle()` | Emit closed POLYGON | Four vertices; closing segment added by caller if needed |
| `dxfDrawFuncs` | Function pointer table | Maps each primitive to its DXF writer |
| `DoExportDXFTracks()` | Main export routine | Opens file → writes prologue → renders entities → writes epilogue → closes |
| `DoExportDXF()` | Shows save dialog | Lazily creates the file chooser widget |

---

## Design Notes

- **Generic output stream**: The same rendering pipeline used for on-screen drawing is reused for DXF export. This ensures that exported geometry matches exactly what you see on screen, and it means no special-casing of coordinate transformations or unit conversions — the same `DrawLine()`, etc., functions can be swapped in/out via function pointers.
- **`curTrackLayer + 1`**: Track layers are internally zero-indexed but DXF requires a one-based layer index for group code 8. The current track layer is used as-is rather than trying to infer which layer each primitive belongs to — this assumes all primitives share the same "current" layer during export, which simplifies things.
- **Linetype mapping**: Drawing options (`DC_DASH`, `DC_DOT`) are translated into DXF linetype indices (1 and 2 respectively). This is a minimal set; full pattern definition for each linestyle would require additional tables.
- **No bitmap support**: `DxfBitMap()` is an empty stub. DXF does not have a native image format, so rasters are typically omitted or written as ODBLOB entities (not implemented here).
