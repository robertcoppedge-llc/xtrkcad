# spiroentrypoints.c — XTrkCad Spiro Wrapper Layer

## Overview

`spiroentrypoints.h` and `spiroentrypoints.c` provide a thin **adaptation layer** that wraps the original Raph Levien `ppedit` library (`spiro.c`) to fit XTrkCad's geometry model. The original `ppedit` library operates on an array of "tagged control points" — each point has coordinates and a type character encoding how the spline behaves at that location (open, closed, corner, cusp).

XTrkCad needed to adapt this for:
- Track segments represented as `[from_segment][to_segment]` pairs rather than arbitrary 2D waypoints.
- Integration with XTrkCad's Bezier rendering context (`bezctx`).
- Handling of the "closed loop" case (e.g., a turnout connecting back to itself).

---

## Public API

```c
/* TaggedSpiroCPsToBezier: Convert an array of spiro_cp structs into a
 * BezContext path that can be rendered or serialized. */
extern void TaggedSpiroCPsToBezier(spiro_cp *spiros, int ncp, bezctx *bc);

/* SpiroCPsToBezier: Variant that takes an explicit "isclosed" flag.
 * This is the primary entry point used by XTrkCad when constructing
 * easement curves between two track segments. */
extern void SpiroCPsToBezier(spiro_cp *spiros, int ncp, int isclosed,
                              bezctx *bc);
```

---

## Type Definitions

```c
#define SPIRO_CORNER   'v'    /* A vertex — symmetric join (mirrored spirals) */
#define SPIRO_G4       'o'    /* Open endpoint: no curvature constraint at this end */
#define SPIRO_G2       'c'    /* Closed-loop endpoint */
#define SPIRO_LEFT     '['    /* Left-side tangent constrained horizontal */
#define SPIRO_RIGHT    ']'    /* Right-side tangent constrained vertical */

/* For open contours, the first cp must have ty='{' and the last must have ty='}' */
#define SPIRO_OPEN_CONTOUR       '{'
#define SPIRO_END_OPEN_CONTOUR   '}'

/* For a closed contour, append an extra cp with ty='z' (its x/y are ignored) */
#define SPIRO_END                'z'
```

---

## How It Works

The wrapper functions:
1. Allocate a `spiro_seg` array using the internal spiro solver (`run_spiro`)
2. Call `spiro_to_bpath()` to convert each segment into a chain of Bezier curves
3. Write the resulting Beziers into a `bezctx *bc` buffer

The original `ppedit` code stores its result as an array of `spiro_seg` structs (each representing one interval between control points). The wrapper then iterates over that array and calls `spiro_seg_to_bpath()` for each element.

---

## Differences from Original ppedit

- **No GUI:** The original `ppedit_gtk1.c` contains a GTK2 dialog for manually drawing spline contours by clicking points on screen. This is omitted from XTrkCad since easement curves are generated algorithmically (not interactively).
- **Closed-loop detection:** When used between two track segments that form a closed loop, an extra control point with `ty='z'` is appended to signal the solver that the path should be treated as cyclic.

---

## See Also

- [`spiro.c`](./spiro.c.md) — The underlying polynomial spiral spline algorithm
- [`bezctx.h/bezctx.c`](../lib/tbezier/) — Bezier path context used for rendering and serialization
