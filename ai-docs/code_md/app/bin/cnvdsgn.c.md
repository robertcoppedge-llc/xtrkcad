# cnvdsgn.c — Non-Violating Design (NVDSGN) Utility Tool

## Overview

`cnvdsgn.c` is a **standalone command-line utility tool** that generates drawing commands for model railroad track layouts. It reads a textual description of the track geometry and outputs corresponding C code array entries (polyline/arc definitions) suitable for rendering by XTrkCAD's internal graphics system.

The program is invoked as:

```bash
./cnvdsgn < input.txt > output.c
```

It serves as an intermediate tool in the **NVDSGN** ("Non-Violating Design") workflow — a method of describing track layouts in a human-readable format and generating executable code from it. This separates layout design (done with a simple text editor) from rendering (handled by the CAD system).

---

## Data Structures & Constants

### `trackSeparation` — Default Gauge Separation

```c
static int trackSeparation = 20;
```

The distance in pixels between parallel straight track lines. This corresponds to the gauge (distance between rails) scaled to screen/pixel coordinates. The value `20` is a default placeholder; in practice it would be computed from the chosen scale (e.g., HO scale ≈ 16.5mm real → ~70–80 pixels depending on DPI settings).

### `arrowHeadLength` — Arrowhead Dimension for Direction Indicators

```c
static int arrowHeadLength = 10;
```

The length of the arrowheads drawn at the ends of track segments to indicate direction. The output generates two additional line segments that form a triangular arrow pointing along the track's forward direction.

---

## Core Algorithm: `buildDesignerLines()`

### Input Format

A plain-text input file where each line begins with a command keyword followed by comma-separated numeric arguments:

| Keyword | Arguments | Description |
|---------|-----------|-------------|
| `ARROW` | `x0, y0, x1, y1` | Draw an arrowhead indicating direction from `(x0,y0)` toward `(x1,y1)` |
| `LINE`  | `x0, y0, x1, y1` | Draw a straight segment (single polyline point pair) |
| `STRAIGHT` | `x0, y0, x1, y1` | Draw two parallel line segments (outer rails of a track) offset perpendicular to the direction by half the gauge |
| `CURVE`   | `x0, y0, x1, y1, radius` | Approximate a circular arc with a sequence of short straight segments; generates points on both outer and inner rail edges |

### Output Format

A C array initializer listing all generated points. Each entry is:

```c
{ type, x, y, ... }
```

where `type` is:
- `1` → polyline point (straight segment)
- `3` → arc point (generated from the circle center and angular step)

### Detailed Breakdown by Command

#### 1. `ARROW` — Direction Marker

Input:
```
ARROW, x0, y0, x1, y1
```

Algorithm:
1. Compute angle `a0 = FindAngle(p1, p0)` — the direction from start to end of the track segment.
2. Output two polyline points defining the arrowhead triangle:
   - One point offset 135° counter-clockwise from the forward direction (pointing backward)
   - One point offset −135° (clockwise), also pointing backward

The output is:
```c
{ 1, x0+0.5, y0+0.5, x1+0.5, y1+0.5 },      // from p0 to p1
{ 1, x0+0.5, y0+0.5, x_head.x,   y_head.y }, // first arrow tip
{ 1, x0+0.5, y0+0.5, x_tail.x,    y_tail.y } // second arrow tip
```

Note: `x` and `y` are rounded via `(long)(coord + 0.5)` for pixel alignment (round-half-up).

---

#### 2. `LINE` — Single Straight Segment

Input:
```
LINE, x0, y0, x1, y1
```

Simply outputs a single polyline pair:
```c
{ 1, (long)(x0+0.5), (long)(y0+0.5), (long)(x1+0.5), (long)(y1+0.5) }
```

---

#### 3. `STRAIGHT` — Parallel Track Rails with Offset

Input:
```
STRAIGHT, x0, y0, x1, y1
```

Algorithm:
1. Compute the angle of the segment: `a0 = FindAngle(p1, p0)` (direction from `(x0,y0)` to `(x1,y1)`).
2. **Outer rail:** Translate by `+90°` and distance `trackSeparation/2`.  
   This is a left-hand offset relative to the forward direction — for a track oriented eastward (`a0 = 0°`), this produces a northward offset, which is the "outside" of a right-turning (positive-radius) curve.
3. **Inner rail:** Translate by `−90°` and distance `trackSeparation/2`.

Output: two polyline point pairs, one for each rail. Both are rounded to integers.

---

#### 4. `CURVE` — Circular Arc with Parallel Rails

Input:
```
CURVE, x0, y0, x1, y1, radius
```

Algorithm (see the file's `FindCenter()` function):

1. **Compute chord midpoint:**  
   `d = FindDistance(p0, p1) / 2`

2. **Compute angle subtended by half-chord:**  
   `a1 = asin(d / |radius|)` — this is half the total central angle.

3. **Normalize the half-angle to the range [−180°, 180°]** so that arc direction (clockwise vs counter-clockwise) can be distinguished. If >180°, subtract 360°.

4. **Compute start/end angles relative to center:**  
   `a0 = FindAngle(center, p0)` and then `a0 = NormalizeAngle(a0 + (90° − a1))` — this aligns the angular sweep so it proceeds from the first point toward the second along the circle.

5. **Number of segments:**  
   The arc length is `len = 2π·|radius| × (a1/360)`. Divide by segment length `20` pixels, take absolute value and round up to get the number of intervals `num`.

6. **Step angle per segment:**  
   `a1 /= num` — so each step advances by a small constant angle.

7. **Loop through segments:** for `j = 0..num−1`:
   - Compute point on outer rail: rotate from center by `a0 + j·Δa`, translate out by `|radius| + trackSeparation/2`.
   - Compute point on inner rail: same angle, radius `|radius| − trackSeparation/2` (handles both positive and negative radii by taking absolute value).

The output interleaves the two rails at each angular position:
```c
{ 3, x_outer.x+0.5, y_outer.y+0.5, x_inner.x+0.5, y_inner.y+0.5 }
```

This generates a polyline that "zigzags" between outer and inner rail points — an acceptable approximation for rendering purposes. A true circular arc would require a different polygonization algorithm (e.g., dividing the angle into equal steps and outputting all points in angular order). The current approach is simpler but produces a slightly jagged appearance when zoomed in.

---

## Utility Functions

### `FindCenter(coOrd *pos, coOrd p0, coOrd p1, double radius)`

Computes the center of a circle given two points on its circumference and the radius. It returns the central angle (in degrees) subtended by the chord from `p0` to `p1`.

The formula is derived from right-triangle trigonometry:
- Half-chord length: `d = |p1 − p0| / 2`
- Half-angle from center: `a1 = asin(d / radius)`
- The total central angle is `2·a1`.

**Ambiguity handling:** If the computed half-angle exceeds 180°, subtract 360° to normalize. This correctly distinguishes between a "short way" arc (minor arc) and "long way" arc (major arc).

---

## File Format Specification

The input file is a simple text format, one command per line:

```text
# Comment lines starting with # are ignored
ARROW, x0, y0, x1, y1
LINE,  x0, y0, x1, y1
STRAIGHT, x0, y0, x1, y1
CURVE, x0, y0, x1, y1, radius
```

Whitespace between commas is permitted. Negative coordinates and negative radii (for left-turning curves) are supported.

---

## Design Decisions & Tradeoffs

### Why a Standalone Tool?

The NVDSGN workflow separates concerns:
- **Design phase:** The user edits a plain-text file with a simple editor — no CAD software required, easy to version-control the layout description in Git.
- **Conversion phase:** `cnvdsgn` converts the text into C code that XTrkCAD can directly load and render.
- **Render phase:** XTrkCAD interprets the generated polyline/arc commands and draws them on screen.

This is a form of **domain-specific language (DSL)** for track layout, with `cnvdsgn` as the compiler from DSL → C → graphics primitives.

### Why Round Coordinates?

All output coordinates are rounded via `(long)(coord + 0.5)`. This ensures that:
- The generated points are integer-valued pixel coordinates, matching XTrkCAD's internal drawing representation.
- No floating-point drift accumulates over long tracks — every vertex lies exactly on a grid intersection.

### Why "Zigzag" Arc Approximation?

The curve output alternates between outer and inner rail points at each angular step rather than producing a smooth chain of points along the circle perimeter. This is **not** geometrically optimal (it produces a star-shaped polygon approximating the annular track region), but it:
- Keeps the code very simple (just two parallel radii traversed together)
- Produces visually acceptable results at typical zoom levels
- Avoids needing to compute angular steps for each point along the circle perimeter

A more accurate approach would sort all generated points by their polar angle around the center and output them in that order, producing a true polygonal approximation of the circular arc. The current zigzag is a pragmatic tradeoff favoring simplicity over perfection.

---

## Summary Table

| Feature | Detail |
|---------|--------|
| **Input** | Plain text file with ARROW/LINE/STRAIGHT/CURVE commands |
| **Output** | C array of `{type, x, y, ...}` entries for rendering |
| **Track separation (gauge)** | Configured via `trackSeparation` constant; defaults to 20 pixels |
| **Arc approximation** | Zigzag polyline between outer and inner rails at equal angular steps |
| **Coordinate rounding** | Round-half-up to nearest integer pixel |
| **Usage** | `cnvdsgn < layout.txt > output.c` — pipe directly into XTrkCAD's load function or save as a `.c` source file |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Convert a human-readable track description (text) into C code that renders the layout |
| **Domain** | Domain-specific language for track geometry; pipeline tool in a DSL-to-renderer workflow |
| **Key concept** | A textual DSL (`ARROW`, `STRAIGHT`, `CURVE`) is compiled by `cnvdsgn` into executable drawing commands (polylines and arcs) that XTrkCAD renders directly |
| **Main entry point** | `main()` reads from stdin, parses line-by-line, calls `buildDesignerLines()`, writes C array to stdout |
