# dxfformat.c — DXF Export Formatting Utilities

## Overview

`dxfformat.c` provides a suite of helper functions for generating well-formatted **DXF (Drawing Exchange Format)** files. It handles:

- **Unit conversion** between English and metric systems
- **Layer name construction** with automatic numbering
- **Line styles** (continuous, dashed, dotted)
- **Integer, position, and text formatting** following DXF group code conventions
- **Complete file prologue** including header section, tables (line types, layers), and entity definitions
- **File epilogue** to close sections

The code uses a **streaming output pattern**: all functions append formatted DXF content into a `DynString` buffer that is progressively grown. This avoids pre-allocating a large block upfront — instead, memory expands on demand as commands are written out.

---

## Core Macros & Globals

```c
#include <dynstring.h>
#include "dxfformat.h"

extern char *sProdNameUpper;   /* Product name (e.g., "XTrkCAD") used in layer names */
extern long units;             /* 0 = English, 1 = metric */
```

---

## DXF Group Code Basics

DXF files use **group code / value** pairs. Each pair is a separate line:

| Group | Meaning | Example Value |
|-------|---------|---------------|
| `9`   | Entity name (e.g., "LINE", "TEXT") |
| `8`   | Layer name |
| `6`   | Line type (STYLE) |
| `0` / `2` | End of entity list |
| `10`/`20` | X/Y coordinates |
| `40` | Float value (radius, angle, text height) |
| `3` | Text style name |
| `70` | Flags (e.g., 65 = text not mirrored) |
| `420` | True color index |

The macro `DXF_INDENT` is used to prefix every line with spaces for readability. This keeps the DXF human-readable and editable in a text editor.

---

## Unit Conversion

```c
void DxfFormatPosition(DynString *result, int type, double value)
{
    if (units == 1) {        /* Metric mode */
        if( type < 50 || type > 58 ) {
            value *= 25.4;   /* Convert inches → mm for group codes 10–29, etc. */
        }
    }

    DynStringPrintf(result, DXF_INDENT "%d\n%0.6f\n", type, value);
}
```

Group codes `10` and `20` (X/Y) are the most commonly converted. Codes in the range 50–59 represent angles — these are **not** converted because DXF angles use radians regardless of units.

---

## Layer Names

Layers are built from a base name plus an incrementing index:

```c
void DxfLayerName(DynString *result, char *name, int layer)
{
    DynStringPrintf(result, DXF_INDENT "8\n%s%d\n", name, layer);
}
```

If `sProdNameUpper` is `"XTRKCAD"`, the first layer becomes `XTRKCAD1`, then `XTRKCAD2`, etc. The group code `8` tells the DXF reader to interpret the following string as a **layer name**.

---

## Line Styles

```c
void DxfLineStyle(DynString *result, int style)
{
    char* s = "CONTINUOUS";
    switch ( style ) {
        case 1: s = "DASHEDTINY"; break;
        case 2: s = "DOTTINY"; break;
    }
    DynStringPrintf(result, DXF_INDENT "6\n%s\n", s);
}
```

Group code `6` is the **LTYPE** name. The strings `"CONTINUOUS"`, `"DASHEDTINY"`, and `"DOTTINY"` are standard AutoCAD line types. Group code `70 = 0` indicates a simple (non-pattern) linestyle.

---

## Helper: Append Functions

The main formatting functions (`DxfLineCommand`, etc.) delegate to small append helpers that allocate into a temporary `DynString`, cat the result, then free it:

```c
static void DxfAppendPosition(DynString *output, int type, double value)
{
    DynString formatted = NaS;
    DynStringMalloc(&formatted, 0);
    DxfFormatPosition(&formatted, type, value);
    DynStringCatStr(output, &formatted);
    DynStringFree(&formatted);
}

static void DxfAppendInteger(DynString *output, int type, int value)
{
    DynString formatted = NaS;
    DynStringMalloc(&formatted, 0);
    DxfFormatInteger(&formatted, type, value);
    DynStringCatStr(output, &formatted);
    DynStringFree(&formatted);
}

static void DxfAppendLineStyle(DynString *output, int style)
{
    /* same pattern */
}
```

This "append helper" pattern avoids polluting the global namespace and keeps each `Dxf*Command` function's signature clean — it only takes a `DynString *result` parameter.

---

## Entity Commands

### LINE

```c
void DxfLineCommand(DynString *result, int layer, double x0, double y0,
                     double x1, double y1, int style, int color)
{
    DynStringCatCStr(result, DXF_INDENT "0\nLINE\n");
    DxfAppendLayerName(result, layer);
    DxfAppendPosition(result, 10, x0);
    DxfAppendPosition(result, 20, y0);
    DxfAppendPosition(result, 11, x1);
    DxfAppendPosition(result, 21, y1);
    DxfAppendLineStyle(result, style);
    DxfAppendInteger(result, 420, color);
}
```

The group code order matches the DXF reference format: entity name first (`0\nLINE`), then layer (`8`), start point (`10/20`), end point (`11/21`), linetype (`6`), truecolor (`420`).

---

### CIRCLE

```c
void DxfCircleCommand(DynString *result, int layer, double x, double y,
                       double r, int style, int color)
{
    DynStringCatCStr(result, DXF_INDENT "0\nCIRCLE\n");
    DxfAppendPosition(result, 10, x);   /* center X */
    DxfAppendPosition(result, 20, y);   /* center Y */
    DxfAppendPosition(result, 40, r);   /* radius in group 40 */
    DxfAppendLayerName(result, layer);
    DxfAppendLineStyle(result, style);
    DxfAppendInteger(result, 420, color);
}
```

Note that circle **center** is at `(10, 20)` and **radius** at `40` — a common DXF convention.

---

### ARC

```c
void DxfArcCommand(DynString *result, int layer, double x, double y,
                    double r, double a0, double a1, int style, int color)
{
    DynStringCatCStr(result, DXF_INDENT "0\nARC\n");
    DxfAppendPosition(result, 10, x);
    DxfAppendPosition(result, 20, y);
    DxfAppendPosition(result, 40, r);      /* radius */
    DxfAppendPosition(result, 50, a0);     /* start angle (radians) */
    DxfAppendPosition(result, 51, a0 + a1);/* sweep angle */
    DxfAppendLayerName(result, layer);
    DxfAppendLineStyle(result, style);
    DxfAppendInteger(result, 420, color);
}
```

Group code `50` is the **start angle** and `51` is the **sweep (end minus start)** — both in radians. The second arc endpoint is derived by adding the sweep to the start; DXF readers can reconstruct it if needed.

---

### TEXT

```c
void DxfTextCommand(DynString *result, int layer, double x, double y,
                     double size, char *text, int color)
{
    DynStringCatCStr(result, DXF_INDENT "0\nTEXT\n");
    DynStringCatCStrs(result, DXF_INDENT "1\n", text, "\n", NULL); /* group 1 = string */
    DxfAppendPosition(result, 10, x);
    DxfAppendPosition(result, 20, y);
    DxfAppendPosition(result, 40, size / 72.0);   /* inches → DXF units */
    DxfAppendLayerName(result, layer);
    DxfAppendInteger(result, 420, color);
}
```

DXF text is stored in **inches** internally regardless of user units (group code `1` holds the string, group `40` holds height in inches). The division by 72.0 converts the user's millimeter size into DXF inches. Group code `7 = "STANDARD"` would be a text style name if used.

---

## Prologue: Header + Tables

The prologue builds a complete DXF header section, defines line type and layer tables, then closes them before entering the entities section:

```c
void DxfPrologue(DynString *result, int layerCount, double x0, double y0,
                  double x1, double y1)
{
    int i;
    DynString layer = NaS;
    DynStringMalloc(&layer, 0);
    DynStringCatCStr(result, "\
      0\nSECTION\n\
      2\nHEADER\n\
      9\n$ACADVER\n 1\nAC1009\n");

    DxfUnits(result);                          /* $MEASUREMENT + $INSUNITS */
    DxfDimensionSize(result, DXF_DIMTEXTSIZE); /* group 70 = text height (default) */
    DxfDimensionSize(result, DXF_DIMARROWSIZE);/* group 420 = arrowhead size */

    DynStringCatCStr(result, " 9\n$EXTMIN\n");
    DxfAppendPosition(result, 10, x0);   /* world minimum corner */
    DxfAppendPosition(result, 20, y0);
    DynStringCatCStr(result, " 9\n$EXTMAX\n");
    DxfAppendPosition(result, 10, x1);   /* world maximum corner */
    DxfAppendPosition(result, 20, y1);

    DynStringCatCStr(result, "\
      9\n$TEXTSTYLE\n 7\nSTANDARD\n\
      0\nENDSEC\n\
      0\nSECTION\n\
      2\nTABLES\n\
      0\nTABLE\n\
      2\nLTYPE\n");

    /* Define three line types: continuous, dashed (tiny dash), dotted */
    DynStringCatCStr(result, "\
        0\nLTYPE\n  2\nCONTINUOUS\n  70\n0\n\
        3\nSolid line\n\
        72\n65\n  73\n0\n  40\n0\n\
        0\nLTYPE\n  2\nDASHED\n  70\n0\n\
        3\n__ __ __ __ __ __ __ __ __ __ __ __\n\
        72\n65\n  73\n2\n  40\n0.15\n  49\n0.1\n  49\n-0.05\n\
        0\nLTYPE\n  2\nDOT\n  70\n0\n\
        3\n...................................\n\
        72\n65\n  73\n2\n  40\n0.1\n  49\n0\n  49\n-0.05\n\
      ");

    /* Layer table */
    DynStringCatCStr(result, "\
        0\nTABLE\n 2\nLAYER\n 70\n0\n");

    for (i = 0; i < layerCount; i++) {
        DynStringPrintf(&layer,
                        DXF_INDENT"0\nLAYER\n  2\n%s%d\n  6\nCONTINUOUS\n  62\n7\n  70\n0\n",
                        sProdNameUpper, i + 1);
        DynStringCatStr(result, &layer);
    }

    DynStringCatCStr(result, "\
      0\nENDTAB\n\
      0\nENDSEC\n\
      0\nSECTION\n\
      2\nENTITIES\n");
}
```

Structure:

1. **HEADER section** — includes version (`$ACADVER`), units (`$MEASUREMENT`, `$INSUNITS`), extents, text style.
2. **TABLES section** — contains the LTYPE table (three entries) and the LAYER table (one entry per layer).
3. The loop over `layerCount` builds each layer block dynamically; group codes `6 = "CONTINUOUS"`, `62 = 7` (true color), `70 = 0` (on by default).
4. **ENTITIES section** — where actual geometry goes.

---

## Epilogue

```c
void DxfEpilogue(DynString *result)
{
    DynStringCatCStr(result, DXF_INDENT "0\nENDSEC\n" DXF_INDENT "0\nEOF\n");
}
```

The `ENDSEC` closes the ENTITIES section; `EOF` marks end of file. Group code `0` is a terminator.

---

## Dimension Defaults

A static lookup table provides default dimension values:

```c
static char *dxfDimensionDefaults[][3] = {
    { "1.0",  "25.0", "$DIMTXT" },   /* text height (group 70) */
    { "0.8",  "20.0", "$DIMASZ" }    /* arrowhead size (group 420) */
};

void DxfDimensionSize(DynString *result, enum DXF_DIMENSIONS dimension )
{
    DynString formatted;
    DynStringMalloc(&formatted, 0);

    DynStringPrintf(&formatted,
                    DXF_INDENT "9\n%s\n  40\n%s\n",
                    dxfDimensionDefaults[dimension][2],   /* group name */
                    dxfDimensionDefaults[dimension][units]);/* value in user units */

    DynStringCatStr(result, &formatted);
    DynStringFree(&formatted);
}
```

The enum `DXF_DIMENSIONS` maps to indices 0 and 1. The third column of the table is the DXF group name (`$DIMTXT`, `$DIMASZ`). Group code `40` holds a float value — the default text height is **1 inch** (25 mm) by default, which matches typical CAD defaults for engineering drawings.

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `DxfLayerName()` | Build layer name (`XTRKCAD1`, etc.) | Uses global `sProdNameUpper` as base |
| `DxfFormatInteger()` | Write a group code / integer pair | Used for flags like `70=0` (layer on) |
| `DxfFormatPosition()` | Write a coordinate, unit-aware | Converts inches→mm when metric is active |
| `DxfLineStyle()` | Write linetype name | Supports continuous/dashed/dotted |
| `DxfAppend*()` helpers | Append to an existing buffer | Allocation + cat + free pattern |
| `DxfLineCommand()` | Full LINE entity group codes 1–5 | Entity name first, then layer, coords, style, color |
| `DxfCircleCommand()` | CIRCLE entity | Center at (10,20), radius at 40 |
| `DxfArcCommand()` | ARC entity | Start angle + sweep angle in radians |
| `DxfTextCommand()` | TEXT entity | Height converted to inches; group 1 holds string |
| `DxfUnits()` | Write measurement system | `$MEASUREMENT` (70) and `$INSUNITS` (70) |
| `DxfDimensionSize()` | Set dimension defaults | Text height, arrow size via lookup table |
| `DxfPrologue()` | Build complete header+tables | Header → LTYPE table → LAYER table → ENTITIES header |
| `DxfEpilogue()` | Close file | `ENDSEC` + `EOF` markers |

---

## Design Notes

- **Streaming output** via `DynString`: Every function appends to a shared buffer. The caller owns the final string; no large static arrays are needed upfront.
- **Unit conversion is localized**: Only group codes representing coordinates (10, 20, 30, 40, etc.) are converted when metric mode is active. Angles (50–59) are never touched — DXF stores angles in radians regardless of units.
- **Layer numbering** relies on `sProdNameUpper` and a loop counter. If layers must be named arbitrarily, the caller can pass a custom name string instead of relying solely on index-based naming.
