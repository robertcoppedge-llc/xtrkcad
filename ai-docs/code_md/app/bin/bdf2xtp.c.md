# bdf2xtp.c — WinRail BDF to XTrkCad .xtp Converter Utility

## Overview

`bdf2xtp.c` is a standalone command-line utility that converts track turnout and structure definition files from **WinRail's BDF (Bridge Definition File)** format into XTrkCad's `.xtp` parameter file format. It is used when importing turnouts, crossings, double slips, and structures defined in WinRail into an XTrkCad layout project.

## File Location

```
app/bin/bdf2xtp.c  (1260 lines)
```

## Compilation & Usage

This is a standalone executable tool that must be compiled separately:

```bash
gcc -o bdf2xtp bdf2xtp.c -lm
```

**Usage:**
```text
bdf2xtp OPTIONS SOURCE.BDF TARGET.XTP

OPTIONS:
  -c CONTENTS   description of contents (for .xtp header)
  -k COLOR      color of non-track segments as hex (e.g. ff0000 = red)
  -s SCALE      scale of turnouts (HO, HOn3, N, O, S...)
  -v             verbose mode: include original BDF source as comments in .xtp output

For example:
  bdf2xtp -c "Faller HO Structures" -k ff0000 -s HO fallerh0.bdf fallerh0.xtp
```

## Input Format (BDF — WinRail)

A BDF file is a plain-text, line-oriented format describing track components. Each component (straight, curve, turnout, structure) begins with an `END`-terminated header block followed by geometry definitions composed of:
- **Fill points** (`FillPoint`)
- **Lines** (`Line`) — straight segments  
- **Curved lines** (`CurveLine`, `Circle`) — arc segments
- The body ends at the next component or at `End`

### BDF Tokens (First token on each line):

| Token | Class | Action | Description |
|-------|-------|--------|-------------|
| `Straight` / `Gerade` | START | ACT_STRAIGHT | Begin a straight track segment |
| `EndStraight` / `EndGerade` | END | ACT_DONE | End of straight definition |
| `Curve` / `Bogen` | START | ACT_CURVE | Begin a curved track (arc) |
| `EndCurve` / `EndBogen` | END | ACT_DONE | End of curve |
| `Turnout_Left` / `Weiche_links` | START | ACT_TURNOUT_LEFT | Left-turn turnout header |
| `Turnout_Right` / `Weiche_Rechts` | START | ACT_TURNOUT_RIGHT | Right-turn turnout header |
| `EndTurnout` / `EndWeiche` | END | ACT_DONE | End of turnout definition |
| `CurvedTurnout_Left` / `Bogenweiche_Links` | START | ACT_CURVEDTURNOUT_LEFT | Turnout with curved lead rail |
| `CurvedTurnout_Right` / `Bogenweiche_Rechts` | START | ACT_CURVEDTURNOUT_RIGHT | Right-hand curved turnout |
| `ThreeWayTurnout` / `Dreiwegweiche` | START | ACT_THREEWAYTURNOUT | Three-way switch |
| `Crossing_Left` / `Kreuzung_Links` | START | ACT_CROSSING_LEFT | Left-turn crossing (diamond) |
| `Crossing_Right` / `Kreuzung_Rechts` | START | ACT_CROSSING_RIGHT | Right-turn crossing |
| `Crossing_Symetric` / `Kreuzung_Symmetrisch` | START | ACT_CROSSING_SYMMETRIC | Symmetric diamond crossing |
| `DoubleSlip_Left` / `DKW_Links` | START | ACT_DOUBLESLIP_LEFT | Left-turn double slip switch |
| `DoubleSlip_Right` / `DKW_Rechts` | START | ACT_DOUBLESLIP_RIGHT | Right-hand double slip |
| `DoubleSlip_Symetric` / `DKW_Symmetrisch` | START | ACT_DOUBLESLIP_SYMMETRIC | Symmetric double slip |
| `EndCrossing` | END | ACT_DONE | End of crossing definition |
| `Turntable` / `Drehscheibe` | START | ACT_TURNTABLE | Turntable (rotateable platform) header |
| `EndTurntable` / `EndDrehscheibe` | END | ACT_ENDTURNTABLE | End of turntable |
| `TravellingPlatform` / `Schiebebuehne` | START | ACT_TRANSFERTABLE | Transfer table (sliding platform) |
| `EndTravellingPlatform` / `EndSchiebebuehne` | END | ACT_ENDTRANSFERTABLE | End of transfer table |
| `Track` / `Schiene` | START | ACT_TRACK | Simple track segment |
| `EndTrack` / `EndSchiene` | END | ACT_DONE | End of track definition |
| `Structure` / `Haus` | START | ACT_STRUCTURE | Structure header (e.g., building) |
| `EndStructure` / `EndHaus` | END | ACT_ENDSTRUCTURE | End of structure |
| `FillPoint` / `FuellPunkt` | BODY | ACT_FILL_POINT | Point on a line (fills gap) |
| `Line` / `Linie` | BODY | ACT_LINE | Straight line segment |
| `CurvedLine` / `Bogenlinie` | BODY | ACT_CURVEDLINE | Circular arc line segment |
| `Circle` / `Kreislinie` | BODY | ACT_CIRCLE | Full circle arc (360°) |
| `DescriptionPos` / `BezeichnungsPos` | BODY | ACT_DESCRIPTIONPOS | Position label/comment |
| `ArticleNoPos` / `ArtikelNrPos` | BODY | ACT_DESCRIPTIONPOS | Article number/label position |
| `ConnectingPoint` / `Anschlusspunkt` | BODY | ACT_CONNECTINGPOINT | Connection point marker |
| `Price` / `Preis` | BODY | ACT_PRICE | Price information (ignored in .xtp) |

## Output Format (.xtp)

The output `.xtp` file uses XTrkCad's track parameter format:

```text
TURNOUT <SCALE> "<CONTENTS> <PARTNO>"
  P "<PATH_NAME>" <SEGMENT_INDEX_1> [ SEGMENT_INDEX_2 ] ... 0 <SEG_INDEX_3> ...
  E <X> <Y> <ANGLE_DEGREES>        ; endpoint position and angle
  S 0 0 <X1> <Y1> <X2> <Y2>       ; straight track segment (radius=0)
  C <RADIUS> <CENTER_X> <CENTER_Y> <A0> <A1>   ; circular arc
  L <COLOR> <X1> <Y1> <X2> <Y2>        ; straight line segment
  A <COLOR> <RADIUS> <CENTER_X> <CENTER_Y> <ANGLE_A0> <ANGLE_A1>   ; circular arc line
  T 0                                 ; terminator (EndSegments)
STRUCTURE ...
```

### Field descriptions:

- **`TURNOUT`** — Header line containing scale (`HO`, `N`, etc.), contents description, and part number.
- **`P <name>`** — Defines a path name for a route through the turnout/crossing; followed by segment indices (1-based) forming that route. Multiple paths separated by `0`.
- **`E <x> <y> <angle>`** — Endpoint coordinates and tangent angle at that end of the turnout.
- **`S ...`** — Straight track segment with start/end positions.
- **`C <RADIUS> <Cx> <Cy> <a0> <a1>`** — Circular arc with radius, center, and sweep angles in degrees (`a0` to `a1`).
- **`L <COLOR> ...`** — Line segment (non-track geometry such as platform edges or structure outlines). Color is a 32-bit hex value (AABBGGRR format).
- **`A <COLOR> <RADIUS> <Cx> <Cy> <a0> <a1>`** — Circular arc line.

## Internal Data Structures

### `tokenDesc_t tokens[]` — Token table

A lookup table that maps the first token on each input line to its parsing class and expected arguments:

```c
typedef struct {
    char * name;   // First word on the line (e.g., "Straight", "Turnout_Left")
    class_e class;  // CLS_START, CLS_BODY, or CLS_END
    action_e action;// ACT_STRAIGHT, ACT_CURVE, etc.
    char *args;    // Format string describing expected arguments (e.g., "SSNN" = two strings, two numbers)
} tokenDesc_t;
```

The table includes both English and German BDF keywords. For example:

- `"Straight"` maps to `CLS_START`, action `ACT_STRAIGHT` with format `"SSNN"`
- `"Bogen"` (German for "Curve") maps to the same entry so German files are also accepted.

### `action_e` — Action enumeration

Each recognized token maps to one of these actions:

```c
enum {
    ACT_UNKNOWN,
    ACT_DONE,
    ACT_STRAIGHT,
    ACT_CURVE,
    ACT_TURNOUT_LEFT,
    ACT_TURNOUT_RIGHT,
    ACT_CURVEDTURNOUT_LEFT,
    ACT_CURVEDTURNOUT_RIGHT,
    ACT_THREEWAYTURNOUT,
    ACT_CROSSING_LEFT,
    ACT_CROSSING_RIGHT,
    ACT_DOUBLESLIP_LEFT,
    ACT_DOUBLESLIP_RIGHT,
    ACT_CROSSING_SYMMETRIC,
    ACT_DOUBLESLIP_SYMMETRIC,
    ACT_TURNTABLE,
    ACT_ENDTURNTABLE,
    ACT_TRANSFERTABLE,
    ACT_ENDTRANSFERTABLE,
    ACT_TRACK,
    ACT_STRUCTURE,
    ACT_ENDSTRUCTURE,
    ACT_FILL_POINT,
    ACT_LINE,
    ACT_CURVEDLINE,
    ACT_CIRCLE,
    ACT_DESCRIPTIONPOS,
    ACT_ARTICLENOPOS,
    ACT_CONNECTINGPOINT,
    ACT_STRAIGHTTRACK,
    ACT_CURVEDTRACK,
    ACT_STRAIGHT_BODY,   // body line (geometry)
    ACT_CURVE_BODY,      // body curve
    ACT_PRICE
};
```

### `coOrd` — Coordinate type

A simple 2D point used throughout:

```c
typedef struct { double x; double y; } coOrd;
```

### `line_t` — A line segment (either straight or arc)

```c
typedef struct {
    char type;     // 'L' = straight line, 'A' = circular arc
    coOrd pos[2];  // start and end points in model coordinates
    double radius, a0, a1;   // for arcs: radius (negative if left-handed) and sweep angles
    coOrd center;  // center of the circular arc
} line_t;
```

### `segs_t` — A track segment (rail or structure edge)

```c
typedef struct {
    double radius;      // 0.0 = straight, otherwise circular arc
    coOrd pos[2];       // start and end in model coordinates
    int mark;           // flag used during path searching
    endPoint_t * ep[2];// pointers to the endpoints this segment connects to
} segs_t;
```

### `endPoint_t` — An endpoint of a turnout/structure

```c
typedef struct {
    int busy;
    coOrd pos;
    double a;           // angle of the tangent at this endpoint
} endPoint_t;
```

### `paths_t` — A path (route) through a turnout/crossing

```c
typedef struct {
    int index;   // 1-based index into paths[] array
    int count;   // number of segments in this route
    int segs[MAXSEG];  // segment indices (1-based, negative for right-hand turns?)
} paths_t;
```

## Parsing & Generation Flow

### `getLine()` — Input line reader

Reads a line from the BDF file into a buffer. Strips trailing `\r` and `\n`. If a comment begins with `;`, everything after is discarded. Leading whitespace is stripped. Returns `NULL` at EOF.

### `reset(tokenDesc_t *tp, arg_t *args)` — Initialize object state

Called when the first token of a new component (turnout/structure) is encountered. Saves:
- Part number (`partNo`) and contents description (`name`) from arguments
- Optional parameters like scale angle or crossing length into `params[]`
- Resets pointers to buffers for segments, endpoints, lines

### `process(tokenDesc_t *tp, arg_t *args)` — Parse a single line

Dispatches based on the token's action:

| Action | What it does |
|--------|--------------|
| `ACT_DONE` | Calls `generateTurnout()` to output the complete TURNOUT/STRUCTURE block; resets counters. |
| `ACT_STRAIGHT` / `ACT_CURVE` | Initializes a new segment at position 0 and endpoint at angle 270° (pointing up). For curves, also computes the second endpoint based on the arc radius and sweep angle. |
| `ACT_TURNOUT_LEFT/RIGHT` | Sets the `right` flag to indicate which side of the turnout geometry follows next. |
| `ACT_CURVEDTURNOUT_*` / `ACT_CROSSING_*` / `ACT_DOUBLESLIP_*` / `ACT_TURNTABLE` | Reads additional parameters (curvature radius, crossing angle, platform lengths) from subsequent lines; constructs the geometry by creating endpoint/segment pairs and laying out arcs. |
| `ACT_STRAIGHTBODY` / `ACT_CURVE_BODY` | Adds a body line to a structure or turnout. These are drawn but not part of the track routing (e.g., building outlines). |
| `ACT_TURNTABLE` | Reads an array of "0"/"1" flags that indicate which radial spokes exist, then writes out turntable geometry with radial arms and arc platforms. |

### `generateTurnout()` — Emit the .xtp block

Writes the TURNOUT header line (scale, contents description, part number), then:
1. Calls `computePaths()` to compute routes through the turnout/crossing by searching for non-overlapping paths between endpoints.
2. Iterates over all endpoints and writes each endpoint record (`E ...`).
3. Iterates over all segments: straight ones become `S` lines, curved ones have their center/radius/angles computed (via `computeCurve()`) and written as `C`.
4. Writes the terminator line.

### `computePaths()` — Compute routing through crossings/slips

A depth-first search that finds all distinct routes between endpoints:

1. Connect each segment to its two adjacent endpoints via the `ep[]` pointers.
2. From a starting endpoint, recursively traverse segments (`searchSegs()`) marking them as "used" and building up a route list in `curPath[]`.
3. When no more unvisited neighbors exist at an endpoint, backtrack and try another segment.
4. Once all paths are found (no unvisited endpoints remain), convert the internal path representation into output format by writing `P "<name>" <seg1> <seg2> ... 0` lines.

### `computeCurve()` — Convert two-point + radius to center+angle form

Given two endpoints and a radius, computes:
- The distance between them (`d`)
- The chord angle (`a = atan2(dx, dy)`)
- Half the subtended angle from geometry of an isosceles triangle: `asin(d / (2*radius))`
- Constructs the center point by rotating from `pos0` along the normal direction
- Writes the output angles as `a1 - a0 = 2 * half_angle`

## Notes

- BDF files are case-insensitive; both English and German keywords work.
- The `-v` option is useful for debugging because it prints each input line as an XML-style comment at the top of its corresponding `.xtp` output.
- This utility predates XTrkCad's native `.bdf` import filter, which uses a more structured (possibly binary or semi-binary) format.
