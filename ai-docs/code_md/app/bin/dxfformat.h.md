# dxfformat.h — DXF Export Format Declarations

## Overview

`dxfformat.h` is a small header file that declares the **DXF export formatting** interface. It defines:

- A `DXF_DIMENSIONS` enumeration for dimension-related group codes (text size, arrowhead size)
- Function prototypes for all the DXF formatting utilities used in `dxfoutput.c` and `dxfformat.c`

The header is intentionally minimal — it only exposes what's needed to construct well-formatted DXF files from XTrkCAD geometry.

---

## Enum: `DXF_DIMENSIONS`

```c
enum DXF_DIMENSIONS {
    DXF_DIMTEXTSIZE,   /* Group code $DIMTXT  → controls text height in dimension annotations */
    DXF_DIMARROWSIZE   /* Group code $DIMASZ  → controls arrowhead size for dimensions */
};
```

These correspond to DXF system variables used by AutoCAD for dimension styles:

| Enum | DXF Variable | Default Value | Meaning |
|------|-------------|---------------|---------|
| `DXF_DIMTEXTSIZE` | `$DIMTXT` | `"1.0"` (in) or `"25.0"` (mm) | Height of dimension text |
| `DXF_DIMARROWSIZE` | `$DIMASZ` | `"0.8"` (in) or `"20.0"` (mm) | Arrowhead length |

The corresponding values are written to the DXF header via `DxfDimensionSize()` using a static lookup table in `dxfformat.c`.

---

## Function Prototypes

### Layer Naming

```c
void DxfLayerName(DynString *result, char *name, int layer);
```

Builds a layer name string like `"XTRKCAD1"`, `"XTRKCAD2"`, etc. The `layer` parameter is the zero-based index; DXF expects one-based names (so XTrkCAD adds 1 internally when using it).

---

### Position Formatting

```c
void DxfFormatPosition(DynString *result, int type, double value);
```

Writes a coordinate group code/value pair (`type` is the DXF group number, e.g., 10 for X, 20 for Y). Converts from inches to millimeters if the current units setting is metric.

---

### Line Style

```c
void DxfLineStyle(DynString *result, int style);
```

Writes a linetype name: `CONTINUOUS`, `DASHEDTINY`, or `DOTTINY`. The integer argument maps to an internal enum (0 = continuous, 1 = dashed, 2 = dotted).

---

### Entity Command Builders

| Function | Signature | Purpose |
|----------|-----------|---------|
| `DxfLineCommand` | `(DynString *result, int layer, double x0, double y0, double x1, double y1, int style, int color)` | Builds a full LINE entity record |
| `DxfCircleCommand` | `(DynString *result, int layer, double x, double y, double r, int style, int color)` | Builds a CIRCLE entity (center + radius) |
| `DxfArcCommand` | `(DynString *result, int layer, double x, double y, double r, double a0, double a1, int style, int color)` | Builds an ARC entity (start angle + sweep angle) |
| `DxfTextCommand` | `(DynString *result, int layer, double x, double y, double size, char *text, int color)` | Builds a TEXT entity (height in inches internally) |

All of these return a formatted string ready for `fputs()` or writing into a `DynString`. They are called from the corresponding export functions in `dxfoutput.c`.

---

### Units & Dimension Defaults

```c
void DxfUnits(DynString *result);
```

Writes `$MEASUREMENT` (70 = 1 for metric, 0 for English) and `$INSUNITS` (4 = inches, 1 = millimeters) to the DXF header.

```c
void DxfDimensionSize(DynString *result, enum DXF_DIMENSIONS dimension);
```

Writes a dimension system variable (`$DIMTXT` or `$DIMASZ`) using the appropriate default value for the current unit system.

---

### File-Level Sections

```c
void DxfPrologue(DynString *result, int layerCount, double x0, double y0, double x1, double y1);
void DxfEpilogue(DynString *result);
```

- **`DxfPrologue()`**: Emits the HEADER section (version, units, extents) + TABLES section (LTYPE and LAYER definitions). Called at the start of every DXF file.
- **`DxfEpilogue()`**: Emits `ENDSEC` (end entities) and `EOF` markers to close the file properly.

---

### Macro

```c
#define DXF_INDENT "  "
```

A simple indentation macro used throughout all `DynStringCatCStr(..., DXF_INDENT "...")` calls. It keeps every DXF group-code pair nicely indented for readability in a text editor or when opened in AutoCAD.

---

## Summary

| Symbol | Type | Purpose |
|--------|------|---------|
| `DXF_DIMENSIONS` | enum | Dimension variable identifiers |
| `DxfLayerName()` | fn | Builds `"XTRKCAD1"` style layer names |
| `DxfFormatPosition()` | fn | Writes a group code/value pair, unit-aware |
| `DxfLineStyle()` | fn | Writes linetype name (continuous/dashed/dotted) |
| `DxfLineCommand()` | fn | Full LINE entity record |
| `DxfCircleCommand()` | fn | CIRCLE entity (center + radius) |
| `DxfArcCommand()` | fn | ARC entity (start angle, sweep angle) |
| `DxfTextCommand()` | fn | TEXT entity (height converted to inches internally) |
| `DxfUnits()` | fn | Writes `$MEASUREMENT` and `$INSUNITS` |
| `DxfDimensionSize()` | fn | Writes dimension defaults (`$DIMTXT`, `$DIMASZ`) |
| `DxfPrologue()` | fn | Header + tables section |
| `DxfEpilogue()` | fn | Ends entities section |

---

## Design Notes

- **No inline implementations**: All functions are declared but defined in `dxfformat.c`. This is typical for a header-only interface where the implementation is in a separate `.c` file.
- **`DynString *result`** — every function takes an output buffer and appends its contribution. The caller allocates, then calls each formatter, then frees. This streaming pattern avoids large static buffers.
- **No global state**: Unlike `dxfoutput.c`, the header doesn't depend on any globals — it's a pure API surface that can be included anywhere without side effects.
