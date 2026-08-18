# compound.c — Compound Tracks: Turnouts and Structures

## Overview

`compound.c` handles **compound tracks** — track objects that are composed of multiple segments, typically turnouts (switches), structures, or sectional track pieces. This file provides the core data structures for storing paths, endpoints, and segment geometry, as well as functions for reading/writing to layout files, computing bounding boxes, and managing description fields via a generic dialog system.

---

## Key Data Structures

### `extraDataCompound_t` — Generic Extra Data for Compound Tracks

All compound track types (`T_TURNOUT`, `T_STRUCTURE`) share this extra data structure:

```c
typedef struct extraDataCompound_t {
    extraDataBase_t base;               // Generic header (index, layer, etc.)
    coOrd orig;                         // Transformation origin (x,y)
    ANGLE_T angle;                      // Rotation angle around origin
    trkSeg_p segs[1];                   // Variable-length array of segments
    wIndex_t segCnt;                    // Number of segments in the segs[] array
    PATHPTR_T paths;                    // NULL or array of path strings (turnout routes)
    BOOL_T handlaid;                    // TRUE if track is hand-laid
    BOOL_T flipped;                     // TRUE if turnout title describes right-hand route first
    BOOL_T ungrouped;                   // TRUE if not matched to a database definition
    BOOL_T split;                       // TRUE for "split" turnouts (multiple paths)
    coOrd descriptionOff;               // Offset of the description label from center
    coOrd descriptionOrig;              // Origin point used for drawing labels
    descPivot_t pivot;                  // Pivot point for rotating descriptions
    char * title;                       // Turnout/structure name (e.g., "Bachmann #615")
    turnoutSpecial_e special;           // Special type: TOnormal, TOadjustable, TOpier, etc.
    union {                             // Union of special-case data:
        struct {                        // TOadjustable:
            DIST_T minD;               // Minimum switch throw distance
            DIST_T maxD;               // Maximum switch throw distance
        } adjustable;
        struct {                        // TOpier: pier structure
            char name[STR_SIZE];       // Pier identifier (e.g., "P01")
            FLOAT_T height;            // Pier height in layout units
        } pier;
    } u;

    track_p next_motor;                 // Pointer to associated switch motor object
} extraDataCompound_t, *extraDataCompound_p;
```

- `paths`: A NULL-terminated array of path strings (each string is a 4-character turnout route code like `"AA"`, `"AB"`), used for routing logic in turnouts.
- `title`: The full description parsed from the manufacturer name, model number, etc.
- `special`: Indicates whether this is a regular turnout, an adjustable switch, or a pier structure.

### Global Enumeration Table

```c
static dynArr_t enumCompound_da;       // Dynamic array of (type, title, count) tuples
typedef struct {
    long count;                        // How many tracks share this description
    char * type;                       // Track type code ("TS", "TN", etc.)
    char * name;                       // Title string used for enumeration
    FLOAT_T price;                     // Price from the price list
    DynString indexes;                 // Comma-separated list of track indices using this title
} enumCompound_t;
```

Used by the enumeration system to display a popup menu listing all available turnout/structure definitions when creating new tracks.

---

## Path Handling Functions

### `GetPaths(track_p trk)` — Get the path array for a turnout/structure

Returns the NULL-terminated array of route codes (e.g., `"AA"` → `"AB"`). Structures have no paths, so this returns NULL.

### `SetPaths(track_p trk, PATHPTR_T paths)` — Set or clear the path array

Frees any existing paths and replaces with a new pointer (or sets to NULL if passed as such). The caller must pass in a NULL-terminated array of 4-character strings.

### `GetCurrPath(track_p trk)` — Get the current path being used

Turnouts can have multiple routes stored; this returns the string for the currently selected route, cycling through them via `.currPathIndex`. If no paths exist, it returns NULL.

### `SetCurrPathIndex(track_p trk, long position)` — Set which path is "current"

Used to cycle between multiple routes on the same turnout (e.g., a 4-path switch with A→B and B→A). Setting this value also invalidates the cached `.currPath` pointer.

### `GetCurrPathIndex(track_p trk)` — Return the current path index

---

## File I/O: Writing Compound Tracks to Layout Files

### `WriteCompoundPathsEndPtsSegs(FILE *f, PATHPTR_T paths, wIndex_t segCnt, trkSeg_p segs, EPINX_T endPtCnt, trkEndPt_p endPts)` — Write paths, endpoints, and segments

Writes the path strings (NULL-terminated), then writes each endpoint (`x`, `y`, angle) for attached tracks, then writes all segment definitions. Used by `WriteCompound()` to output a turnout or structure definition.

---

## Title Parsing and Formatting

### `ParseCompoundTitle(char *title, char **manufP, int *manufL, char **nameP, int *nameL, char **partnoP, int *partnoL)` — Parse a compound track title string

Turnout titles are formatted as:
```
<Manufacturer>\t<Name>\t<PartNo>
```

Example: `"Bachmann\t#615\t20918"` is split into manufacturer (`"Bachmann"`), name (`"#615"`), and part number (`"20918"`). The function fills in the pointers to each field along with their lengths. If no tabs exist, it assumes a single unformatted title.

### `FormatCompoundTitle(long format, char *title)` — Format a title for display

Applies formatting flags:
- `LABEL_MANUF` — include manufacturer name
- `LABEL_TABBED` — add tab separators between fields
- `LABEL_PARTNO` — include part number
- `LABEL_DESCR` — include the full description text (name + optional "Flipped", "Ungrouped", or "Split" prefix)
- `LABEL_FLIPPED`, `LABEL_UNGROUPED`, `LABEL_SPLIT` — prefixes shown for special track states

Used when rendering titles in status bars, enumerations, and descriptions.

---

## Bounding Box Computation

### `ComputeCompoundBoundingBox(track_p trk)` — Compute bounding box from segment array

Calls `GetSegBounds()` on the transformed segments (accounting for `.orig` and `.angle`), then doubles the resulting coordinate to account for gauge width on both sides of centerline. Stores result in `trk->bbox`.

---

## Description Dialog System

### `UpdateCompound(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)` — Generic edit callback

Handles all field edits from the property dialog:
- **MN/NM/PN** (manufacturer/name/partno): parse and apply changes under an undo block; updates title string.
- **OR** (origin): moves the track by translating its origin point.
- **A0/A1/A2/A3** (end angle): rotates around an endpoint, then recalculates all other endpoints' positions/radii/centers.
- **E0/E1/E2/E3** (endpoint position): translates a specific endpoint; recomputes bounding box.
- **Z0/Z1/Z2/Z3** (elevation): sets Z-coordinate of an endpoint; then interpolates elevations for intermediate endpoints and computes grade between ends 0 and 1.
- **AN** (global angle): rotates the entire turnout around a pivot point (first, midpoint, or second).
- **LT** (line type): sets stroke style (solid/dashed/dotted) for draw-style tracks.

After each edit, `DrawNewTrack()` is called to redraw with updated geometry.

---

## Distance Query and Description Output

### `DistanceCompound(track_p t, coOrd *p)` — Compute distance from point to turnout/structure

If the track has endpoints (a turnout), returns the minimum Euclidean distance to any endpoint (after checking both open and fixed ends). If it's a structure or has no endpoints, simply computes distance to the bounding box center. Used for hit testing during selection.

### `DescribeCompound(track_p trk, char *str, CSIZE_T len)` — Generate description string

Populates a global buffer with turnout/structure metadata:
- Determines track type from endpoint count (0 = structure, >2 = turnout, 1–2 = sectional)
- Formats title using `FormatCompoundTitle()`
- Builds the full label including layer number and price (if available)
- Checks if any endpoints are fixed to another track (via `GetTrkEndTrk()`)

---

## Generic Compound Operations

### `MoveCompound(track_p trk, coOrd orig)` — Translate by a delta vector

Simply adds `dx`, `dy` to the origin; recomputes bounding box.

### `RotateCompound(track_p trk, coOrd orig, ANGLE_T angle)` — Rotate around a point

Rotates both the origin and all endpoints (by incrementing `.angle`). Recomputes bounding box.

### `RescaleCompound(track_p trk, FLOAT_T ratio)` — Uniformly scale by factor

Scales origin, description offset, and each segment individually via `RescaleSegs()`. The segment array is duplicated before scaling to avoid modifying the original data.

### `FlipCompound(track_p trk, coOrd orig, ANGLE_T angle)` — Mirror horizontally

Flips across a vertical axis through `orig` at angle `angle`:
1. Flips the origin and endpoints about the flip line.
2. Duplicates the segment array and calls `FlipSegs()` to reverse each segment's direction.
3. Adjusts the title: if the turnout matches a database definition with reversed left/right routes, updates the title accordingly; otherwise toggles the `.flipped` flag.

---

## Enumeration System

### `EnumerateCompound(track_p trk)` — Populate or return from an enumeration list

If called with a track argument, it adds that track's description to a global list of unique titles. If called without arguments (with all tracks first selected), it sorts and deduplicates the list, then displays a popup menu listing each unique turnout/structure title along with its count and price. Used for "Select Turnout Type" dialogs when adding new turns.

---

## File I/O: Reading Compound Tracks from Layout Files

### `ReadCompound(char *line, TRKTYP_T trkType)` — Parse a single compound track line from an XTC file

Supports multiple parameter versions of the XTC format:
- **Pre-version 3**: old format without layer/options fields.
- **Version 3–5**: added options and path count.
- **Version 6+**: full modern format with width, line type, etc.

For each version it parses:
- Track index, layer, scale name, visibility bits (roadbed/ties/bridge), origin x/y, angle, title string.
- Options flags (hand-laid, flipped, ungrouped, split, path override).
- Path count and line type.
- Endpoint coordinates and angles via `ReadSegs()` / `TempEndPtsReset()`.

Then it:
1. Allocates a new track with extra data sized for compound use.
2. Sets origin, angle, title, flags, scale, layer.
3. Reads the path array and segment definitions from subsequent file lines (not shown here).
4. Calls `SetEndPts()` to link endpoints from the temporary buffer into the newly created track.

Returns FALSE if parsing fails; otherwise returns TRUE with a valid new track object in `trk`.

### `DeleteCompound(track_p t)` — Clean up extra data before freeing

Frees any segment array and resets pointers to NULL.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `GetPaths()` / `SetPaths()` | Access the route path strings (NULL-terminated) for a turnout |
| `GetCurrPath()` / `SetCurrPathIndex()` | Select which of multiple routes is currently active |
| `WriteCompoundPathsEndPtsSegs()` | Write paths, endpoints, and segments to a layout file |
| `ParseCompoundTitle()` | Split manufacturer/name/partno fields from a title string |
| `FormatCompoundTitle()` | Build display strings with tabs and prefixes (flipped/ungrouped/split) |
| `ComputeCompoundBoundingBox()` | Compute min/max x/y for culling and hit testing |
| `UpdateCompound()` | Edit callback for the property dialog; handles origin, angle, endpoints, elevations, line type |
| `DistanceCompound()` | Distance query for selection/hit testing (returns nearest endpoint or center) |
| `DescribeCompound()` | Build a human-readable description string from title and flags |
| `MoveCompound()` / `RotateCompound()` / `RescaleCompound()` / `FlipCompound()` | Transform the track geometry; all update bounding box internally |
| `EnumerateCompound()` | Populate or display a list of unique turnout/structure titles for selection dialogs |
| `ReadCompound()` | Parse an XTC file line into a new compound track object (supports multiple param versions) |
| `DeleteCompound()` | Free the segment array and reset pointers before deallocation |

---

## Notes

- All transform functions (`Move`, `Rotate`, `Rescale`, `Flip`) update the bounding box via `ComputeCompoundBoundingBox()`, which is essential for off-screen culling.
- The `.flipped` flag indicates that the title string's left/right routes are swapped relative to the database definition; this is tracked separately from geometry because flipping a turnout in CAD doesn't change its physical shape—only how it's named in the database.
- Structures (`T_STRUCTURE`) share the same extra data layout but have no paths, segments, or endpoints — they are purely decorative objects placed at a location (e.g., a building on the right side of a track).
