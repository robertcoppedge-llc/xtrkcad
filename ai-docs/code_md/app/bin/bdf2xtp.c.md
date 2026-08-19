# bdf2xtp.c - BDF to XTP File Converter Utility

## Overview

`bdf2xtp` is a utility tool that translates `.bdf` files (source files for WinRail track libraries) into `.xtp` files (XTrkCad parameter files). It parses BDF-format turnout/structure definitions and outputs them in XTrkCad's orthogonal segment-based format.

## File Location

```
app/bin/bdf2xtp.c
```

## Command Line Usage

```
Usage: bdf2xtp OPTIONS SOURCE.BDF TARGET.XTP

OPTIONS:
  -c CONTENTS   description of contents (e.g., "Faller HO Structures")
  -k COLOR      color of non-track segments in hex format (RRGGBB)
  -s SCALE      scale factor for turnouts (HO, HOn3, N, O, S, etc.)
  -v            verbose mode: include .bdf source as comments in .xtp file

Example:
  bdf2xtp -c "Faller HO Structures" -k ff0000 -s HO fallerh0.bdf fallerh0.xtp
```

## Data Structures

### `coOrd`
A coordinate structure used throughout for geometric calculations.

```c
typedef struct {
    double x;
    double y;
} coOrd;
```

### `line_t`
Represents a curve segment (arc).

```c
typedef struct {
    char type;              // 'L' = line, 'A' = arc/curve
    coOrd pos[2];          // start and end points
    double radius;         // radius of curvature
    double a0, a1;        // sweep angles in degrees
    coOrd center;          // center point for arcs
} line_t;
```

### `endPoint_t`
Represents an endpoint (switch frog location) with position and angle.

```c
typedef struct {
    int busy;              // flag: has this endpoint been matched?
    coOrd pos;            // x, y coordinates
    double a;             // tangent angle in degrees
} endPoint_t;
```

### `segs_t`
Represents a track segment (straight or curved).

```c
typedef struct {
    double radius;        // 0.0 = straight, otherwise radius of curve
    coOrd pos[2];        // start and end points
    int mark;            // flag: has this segment been used in pathfinding?
    endPoint_t * ep[2];  // pointers to connected endpoints at each end
} segs_t;
```

### `paths_t`
Represents a route/path through segments connecting two endpoints.

```c
typedef struct {
    int index;           // path identifier (1-based)
    int count;           // number of segments in this path
    int segs[MAXSEG];   // segment indices forming the path
} paths_t;
```

### `tokenDesc_t`
Describes a BDF input line token (keywords with their expected arguments).

```c
typedef struct {
    char * name;        // first word on the line (e.g., "Straight", "Curve")
    class_e class;      // where do we expect this: START, END, or BODY
    action_e action;   // what type of action it triggers
    char *args;         // argument format string (e.g., "SSNN" for 2 strings, 2 numbers)
} tokenDesc_t;
```

### `arg_t` (union)
Holds parsed arguments from a BDF line.

```c
typedef union {
    char * string;
    double number;
    long integer;
} arg_t;
```

## Enums

### `class_e` - Line Classification

| Value | Name   | Description                          |
|-------|---------|--------------------------------------|
| 0     | CLS_NULL| unused                                |
| 1     | CLS_START  | start of a turnout/structure block   |
| 2     | CLS_END    | end of a turnout/structure block      |
| 3     | CLS_BODY   | body (intermediate) lines            |

### `action_e` - Action Types

Defines what each BDF line triggers:

| Value | Name                    | Description                                          |
|-------|-------------------------|------------------------------------------------------|
| 0     | ACT_UNKNOWN             | not yet assigned                                     |
| 1     | ACT_DONE                | END token — finish current object                     |
| 2     | ACT_STRAIGHT            | straight track segment                               |
| 3     | ACT_CURVE               | curved track segment                                 |
| 4     | ACT_TURNOUT_LEFT        | turnout (left throw)                                |
| 5     | ACT_TURNOUT_RIGHT       | turnout (right throw)                               |
| 6     | ACT_CURVEDTURNOUT_LEFT  | curved turnout, left side                           |
| 7     | ACT_CURVEDTURNOUT_RIGHT | curved turnout, right side                          |
| 8     | ACT_THREEWAYTURNOUT     | three-way turnout                                   |
| 9     | ACT_CROSSING_LEFT       | crossing switch (left)                              |
| 10    | ACT_CROSSING_RIGHT      | crossing switch (right)                             |
| 11    | ACT_DOUBLESLIP_LEFT     | double slip (left)                                 |
| 12    | ACT_DOUBLESLIP_RIGHT    | double slip (right)                                |
| 13    | ACT_CROSSING_SYMMETRIC  | symmetric crossing                                  |
| 14    | ACT_DOUBLESLIP_SYMMETRIC| symmetric double slip                              |
| 15    | ACT_ENDCROSSING         | END token for crossing types                         |
| 16    | ACT_TURNTABLE           | turntable                                          |
| 17    | ACT_ENDTURNTABLE        | END token for turntable                             |
| 18    | ACT_TRANSFERTABLE       | travelling platform (transfer table)                |
| 19    | ACT_ENDTRANSFERTABLE    | END token for transfer table                        |
| 20    | ACT_TRACK               | track definition                                    |
| 21    | ACT_STRUCTURE           | structure (building, etc.)                          |
| 22    | ACT_ENDSTRUCTURE       | END token for structures                            |
| 23    | ACT_FILL_POINT          | fill point marker                                   |
| 24    | ACT_LINE                | straight line segment                               |
| 25    | ACT_CURVEDLINE          | curved (arc) line                                   |
| 26    | ACT_CIRCLE              | full circle                                          |
| 27    | ACT_DESCRIPTIONPOS      | description position marker                          |
| 28    | ACT_ARTICLENOPOS        | article number position marker                       |
| 29    | ACT_CONNECTINGPOINT     | connecting point between segments                    |

## Global Variables

```c
FILE * fin;        // input file handle
FILE * fout;       // output file handle
int inch;          // 1 = inches, 0 = metric (mm)
char * scale;      // e.g., "HO", "N" — optional user override
int verbose;       // if set, emit source lines as comments in output
char line[1024];   // input buffer for getLine()
long color = 0x00FF00FF;  // default non-track segment color (green)
```

## Core Functions

### `normalizeAngle(double angle)`
Normalizes an angle to the range [0, 360). Handles negative angles by adding 360°, and angles ≥ 360° by subtracting multiples of 360°.

```c
double normalizeAngle( double angle )
{
    while (angle<0) { angle += 360.0; }
    while (angle>=360) { angle -= 360.0; }
    return angle;
}
```

### `D2R(double angle)`
Converts degrees to radians for trigonometric functions.

### `R2D(double R)`
Converts radians back to degrees, normalizing the result with `normalizeAngle()`.

### `findDistance(coOrd p0, coOrd p1)`
Computes Euclidean distance between two points.

```c
double findDistance( coOrd p0, coOrd p1 )
{
    double dx = p1.x-p0.x, dy = p1.y-p0.y;
    return sqrt( dx*dx + dy*dy );
}
```

### `small(double v)`
Checks whether a value is effectively zero (for near-zero comparisons).

```c
int small(double v )
{
    return (fabs(v) < 0.000000000001);
}
```

### `findAngle(coOrd p0, coOrd p1)`
Computes the angle between two points in degrees (counter-clockwise from positive y-axis toward positive x). Returns 0° when degenerate.

```c
double findAngle( coOrd p0, coOrd p1 )
{
    double dx = p1.x-p0.x, dy = p1.y-p0.y;
    if (small(dx) && small(dy)) {
        if (dy >= 0.0) { return 0.0; }
        else { return 180.0; }
    }
    return R2D(atan2( dx,dy ));
}
```

### `translate(coOrd *res, coOrd orig, double a, double d)`
Applies an offset from a center point at angle `a` by distance `d`. Used for arc segment positioning.

### `computeCurve(...)`
Converts between two representations of curves:
- **Endpoint form**: start/end points and radius
- **Polar form**: center point, radius, and sweep angles (a0, a1)

This is used internally to convert straight segments and arcs into the XTrkCad polar format.

```c
static void computeCurve( coOrd pos0, coOrd pos1, double radius,
                          coOrd * center, double * a0, double * a1 )
{
    // ... computes chord midpoint angle, half-chord length,
    // then derives center position and sweep angles.
}
```

### `X(double v)`
A small utility that returns zero if the value is near-zero (within ±0.000001), otherwise returns the value unchanged. Used to suppress floating-point noise in output.

### `getDim(double value)`
Converts a dimension value from the BDF file format into real-world units:
- If metric mode (`inch == 0`): divides by 25.4 (mm → inches)
- Otherwise: divides by 10 (tenths of an inch)

### `getLine()`
Retrieves the next meaningful line from input. Skips blank lines and comments (`;`). Removes CR/LF terminators. Returns a pointer to the trimmed content.

### `flushInput()`
Error recovery: consumes remaining input until it sees an `END` token, then resets parsing state.

### `process(tokenDesc_t * tp, arg_t *args)`
The main dispatch function. It handles each token/action case:

- **ACT_DONE**: Calls `generateTurnout()` to output the complete turnout/structure in XTP format.
- **ACT_STRAIGHT / ACT_CURVE / ... (all action cases)**: Advances segment and endpoint arrays, computes geometry for that BDF element, and stores it for later output.

### `searchSegs(segs_t * sp, int ep)`
Recursive path-finding algorithm. Starting from a given segment and endpoint, it searches forward through the segment graph to find all possible routes between endpoints. Segments are marked (`mark = 1`) as they're traversed to avoid revisiting them in the same search.

### `computePaths()`
Generates path descriptions (the `P` lines in output). It:
1. Connects each segment's endpoints with pointers.
2. Runs a depth-first traversal from every segment's endpoints, discovering all routes.
3. Outputs one or more paths per object (some objects like crossings have multiple routes; these are separated by a blank line in the output).

### `generateTurnout()`
Outputs the final XTP-formatted turnout/structure:
- Header line with scale and part number
- Path lines (`P "Normal" 1 2 3 ...`) listing segment indices for each route
- Endpoint lines (`E x y angle`) for each endpoint
- Segment lines (`S`, `L`, or `A` depending on type)

### `reset(...)`
Initializes arrays and state variables when a new turnout or structure is encountered. Stores the part number, name, parameters (scale, curve angles, etc.), and resets pointers into the segment/endpoint buffers.

## BDF Token Table

The `tokens[]` array maps keyword strings to their classifications and argument formats:

- German keywords: *Gerade*, *Bogen*, *Weiche_links/right*, *Kreuzung_...*, *Drehscheibe*, etc.
- English keywords: *Straight*, *Curve*, *Turnout_Left/Right*, *Crossing*, *Turntable*, *Track*, *Structure*

Each entry specifies:
- The keyword name (case-insensitive match)
- Whether it's a START, END, or BODY token
- The action code it triggers
- An argument format string like `"SSNN"` meaning two strings followed by two numbers.

## Output Format

The output `.xtp` file uses an orthogonal description:

```
TURNOUT HO "PartName"
P "Normal" 1 2 3 0 6 7 8
E x.xxxxxx y.yyyyyyy aaaa.a
S 0 0 x.xxxxxx y.yyyyyyy ...
L 0 color x y z w   ; straight segment
A 0 color r cx cy a0 a1  ; arc segment
...
END_SEGS
```

- `P` — path description (list of segment indices, blank line separates alternative routes)
- `E` — endpoint coordinates and tangent angle
- `S` — straight track segment
- `L` — straight line segment
- `A` — arc/curved line segment with radius, center, and sweep angles

## Notes

- BDF files are a German-origin format used by WinRail. The token table supports both English and German keywords.
- XTP files use 1-based indexing for segments in path lists (the `-1` convention is a quirk of the array being zero-indexed but paths being 1-indexed in output).
- Paths are not currently fully utilized in XTrkCad but will be used when train routing on layouts becomes supported.
