# cnvdsgn.c — Designer Line Format Converter Utility

## Overview

`cnvdsgn` is a standalone utility tool that converts designer-coordinate track descriptions into C-array format suitable for use with XTrkCad's drawing engine. It reads lines in a simple text format (`ARROW`, `LINE`, `STRAIGHT`, `CURVE`) and outputs arrays of integer coordinate points representing the track geometry.

## File Location

```
app/bin/cnvdsgn.c
```

## Command Line Usage

```bash
./cnvdsgn < input.txt > output.c
```

The tool reads from standard input (stdin) and writes to standard output (stdout). The output is a C array of `{ 1, x, y }` or `{ 3, x, y }` triplets representing points.

## Input Format

Each line in the input file must begin with one of the following keywords:

### `ARROW <x0> <y0> <x1> <y1>`
Draws an arrowhead pointing from `(p0)` to `(p1)`. This produces three points: the two endpoints and a third point offset 10 units along the centerline at a ±45° angle.

### `LINE <x0> <y0> <x1> <y1>`
Draws a straight line between two points. Outputs one point triplet.

### `STRAIGHT <x0> <y0> <x1> <y1>`
Draws a double-track straight segment centered on the centerline, offset by half the track separation (default: 20 units) to each side. Produces two parallel line segments with integer-rounded endpoints.

### `CURVE <x0> <y0> <x1> <y1> <radius>`
Draws a curved track segment between `(p0)` and `(p1)` with the given radius (signed: positive = left-turn, negative = right-turn). The curve is split into multiple small straight segments; each produces two parallel rails (offset by `trackSeparation/2` from centerline). The number of sub-segments depends on arc length divided by 20.

## Output Format

The output is a C array literal with comma-separated initializer elements:

```c
{ {1, x0, y0},   // arrowhead point or line endpoint
  {1, x1, y1},
  {3, x_outer, y_outer},  // rail on one side (type 3 = track segment)
  {3, x_inner, y_inner} } // rail on other side
```

- Point type `1` = generic point (arrowhead tip or line endpoint).
- Point type `3` = track/rail segment.
- Coordinates are rounded to the nearest integer (`x+0.5`, truncated via cast to `long`).

## Global Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `trackSeparation` | int | 20 | Half-track-gauge offset from centerline (default 20 units) |
| `arrowHeadLength` | int | 10 | Distance along the track direction to arrowhead tip |

## Algorithm Notes

### Arrowheads
The arrowhead is computed by translating from `(p0)` toward `(p1)` by `arrowHeadLength` at an angle of `angle + 135°` and `angle - 135°`. This creates a V-shaped tip pointing along the direction of travel.

### Straight Segments
A double-track straight produces two parallel segments, each offset from the centerline by `trackSeparation/2 = 10` units (since the full gauge is 40). The outward normal angle is `angle + 90°`, inward is `angle - 90°`.

### Curved Segments
The curve arc length is computed as `r * π * θ / 180`. This is divided into sub-segments of approximately 20 units each, and for each sub-segment two parallel track segments are generated. The number of divisions is `ceil(arc_length / 20)`, ensuring each piece stays under ~20 units long.

## Includes

```c
#include "utility.h"  // Provides Translate(), FindAngle(), FindDistance(), NormalizeAngle(), R2D()
```

The utility module supplies geometric primitives used throughout the converter.
