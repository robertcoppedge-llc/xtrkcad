# compound.c — Compound Objects & Turnout Path Management

## Overview

`compound.c` handles **compound objects** in XTrkCAD: turnouts, pier structures, and other multi-part track assemblies. It provides:

- **Path management** for turnout switches (multiple path definitions stored as NULL-terminated string arrays)
- **Description labeling** — drawing text labels next to turnout/pier objects
- **Distance computation** from arbitrary points to compound geometry (turnout paths, pier surfaces)
- **Description editing** — drag-to-edit position of descriptive labels

The file is named "compound" because it deals with *compound track objects* — turnouts and structures that are composed of multiple segments joined together. Unlike simple tracks (straight rail, simple curves), these have complex geometry requiring special handling.

---

## Data Structures

### `PATHPTR_T` / `extraDataCompound_t.paths` — Turnout Path Definitions

```c
typedef char* PATHPTR_T;  // typedef for pointer to array of paths

struct extraDataCompound_t {
    coOrd orig;           // Origin point for the compound object
    ANGLE_T angle;        // Global rotation angle
    wIndex_t segCnt;      // Number of segments in the geometry
    trkSeg_t *segs;      // Variable-length array of segment descriptors
    PATHPTR_T paths;     // Array of path definitions (for turnouts) — NULL for non-turnouts
    char *currPath;      // Current active path index (used when switching between alternatives)
    wIndex_t currPathIndex;  // Index into the paths array
    coOrd descriptionOrig;   // Origin point for drawing the label text
    coOrd descriptionOff;    // Offset of the label from its origin (for editing position)
    BOOL_T flipped;       // Whether the turnout is "flipped" (reversed geometry)
    BOOL_T ungrouped;     // Whether this object has been "ungrouped" from a compound
    BOOL_T split;         // Whether this is a "split" turnout variant
    trkEndPt_t *endPts;   // End points of the compound object
    char title[STR_SIZE];  // Human-readable title (e.g., "Baldwin DCC-80")
    wDrawColor color;      // Display color
    extraDataFlags_t flags; // Bitfield for special states
};
```

Key concept: `paths` is a **NULL-terminated array of strings**. Each string encodes one possible routing path through the turnout. The format (from the code in other files) looks like:
```
P "path1_start" 30 45 -60
P "path2_start" 20 55 -90
NULL
```

The `currPath` pointer tracks which path is currently active, and `currPathIndex` indexes into the array. This allows a single turnout object to represent multiple possible routings (e.g., a double-slip where the user can choose which rail to switch onto).

---

### `carPart_t / carPartParent_t` — Compound Parts (from dcar.c, referenced here)

These structures support grouping multiple track segments into a single compound entity. A "part" is a sub-component of a larger assembly (e.g., body + cab of a locomotive). The parent holds an array of child parts. This enables:
- Group selection (click the whole assembly once instead of each piece)
- Shared bounding box computation
- Unified transformation (move/rotate applies to all children simultaneously)

---

### `compoundData` — Describe Dialog State

```c
static struct {
    coOrd endPt[4];      // Endpoints for angle editing (four possible endpoints)
    ANGLE_T endAngle[4];
    DIST_T endRadius[4];
    coOrd endCenter[4];  // For curved ends
    FLOAT_T elev[4];     // Elevation at each endpoint (for grade control)
    coOrd orig;          // Current origin point of the object
    ANGLE_T angle;       // Current rotation angle
    descPivot_t pivot;   // Pivot point for angle editing
    char manuf[STR_SIZE];
    char name[STR_SIZE];
    char partno[STR_SIZE];
    long epCnt;          // Number of endpoints currently tracked
    long segCnt;         // Segment count
    long pathCnt;        // Number of paths (for turnouts)
    FLOAT_T grade;       // Computed grade between elevations 0 and 1
    DIST_T length;      // Overall length
    drawLineType_e linetype;
    unsigned int layerNumber;
} compoundData;
```

This is a **global state** structure used during the describe/edit dialog. It accumulates all parameters as the user interacts with them (clicking endpoints, typing values). The `descPivot_t` allows choosing which endpoint serves as the pivot for angle editing — you can rotate around any corner.

---

### `compoundDesc[]` — Field Descriptions for the Describe Dialog

```c
typedef enum { E0, A0, C0, R0, Z0, E1, A1, C1, R1, Z1, E2, A2, C2, R2, Z2, E3, A3, C3, R3, Z3, GR, OR, AN, PV, MN, NM, PN, LT, SC, LY } compoundDesc_e;

static descData_t compoundDesc[] = {
    /*E0*/  { DESC_POS, N_("End Pt 1: X,Y"), &compoundData.endPt[0] },
    /*A0*/  { DESC_ANGLE, N_("Angle"),         &compoundData.endAngle[0] },
    /*C0*/  { DESC_POS, N_("Center X,Y"),      &compoundData.endCenter[0] },
    /*R0*/  { DESC_DIM, N_("Radius"),          &compoundData.endRadius[0] },
    /*Z0*/  { DESC_FLOAT, N_("Z1"),             &compoundData.elev[0] },
    /*E1*/  { DESC_POS, N_("End Pt 2: X,Y"),   &compoundData.endPt[1] },
    ...
    /*MN*/  { DESC_STRING, N_("Manufacturer"), &compoundData.manuf, sizeof(compoundData.manuf) },
    /*NM*/  { DESC_STRING, N_("Name"),        &compoundData.name, sizeof(compoundData.name) },
    /*PN*/  { DESC_STRING, N_("Part No"),     &compoundData.partno, sizeof(compoundData.partno) },
    /*LT*/  { DESC_LIST, N_("Line Type"),     &compoundData.linetype },
    /*SC*/  { DESC_LONG, N_("# Segments"),     &compoundData.segCnt },
    /*LY*/  { DESC_LAYER, N_("Layer"),        &compoundData.layerNumber },
    { DESC_NULL }
};
```

Each field maps to a piece of the `extraDataCompound_t` structure. The enum values (`E0`, `A0`, etc.) correspond to which endpoint or property is being edited. Notable fields:
- **GR** — computed grade between elevations 0 and 1 (read-only, auto-updates)
- **MN/NM/PN** — manufacturer, name, part number strings (used for filtering/labeling)
- **LT** — line type (solid/dashed/dotted)

---

## Core Functions

### `GetPaths(trk)` / `SetPaths(trk, paths)` — Get/Set Turnout Paths

```c
EXPORT PATHPTR_T GetPaths( track_p trk ) {
    struct extraDataCompound_t *xx = GET_EXTRA_DATA( trk, T_NOTRACK, extraDataCompound_t );
    if ( GetTrkType(trk) == T_STRUCTURE && xx->paths != NULL ) { ... }
    if ( GetTrkType(trk) == T_TURNOUT && xx->paths == NULL ) { ... }
    return xx->paths;
}

EXPORT void SetPaths( track_p trk, PATHPTR_T paths ) {
    struct extraDataCompound_t *xx = GET_EXTRA_DATA( trk, T_NOTRACK, extraDataCompound_t );
    if ( xx->paths ) { MyFree( xx->paths ); }
    if ( paths == NULL ) {
        xx->paths = NULL;
    } else {
        wIndex_t pathLen = GetPathsLength( paths );
        xx->paths = memdup( paths, pathLen * sizeof *xx->paths );
    }
    xx->currPath = NULL;
    xx->currPathIndex = 0;
}
```

- `GetPaths()` returns the array of path strings (NULL if not a turnout). The caller must check the track type first.
- `SetPaths()` owns the data: it frees any old paths and allocates a new copy via `memdup()`. This ensures that modifying the array afterward doesn't mutate the stored pointer.

**Why two separate functions?** Because paths are only present on turnout-type tracks (`T_TURNOUT` or `T_STRUCTURE`). The function guards against querying non-turnout objects to avoid returning NULL when something else is expected.

---

### `GetPathsLength(paths)` — Count Number of Paths

Iterates over the path array, skipping the NULL terminators that separate individual paths from each other. Each path itself is a single string (like `"P \"name\" 30 45 -60"`), so the function counts how many non-NULL entries exist in the array.

---

### `GetCurrPath(trk)` — Get Current Path Pointer

Returns a pointer to the currently active path within the turnout's path list. The logic is:
1. If there's already a cached pointer (`xx->currPath`), return it directly.
2. Otherwise, start at index `xx->currPathIndex` and walk backward through the array until either:
   - You hit the beginning (no more previous paths) → reset to first path
   - You find the current entry

This is essentially an **iterator pattern**: the pointer advances as you call it repeatedly, but wraps back to the start when exhausted. The caller can use `GetCurrPath()` in a loop to enumerate all available paths and pick one at a time (e.g., for a UI dropdown menu).

---

### `SetCurrPathIndex(trk, position)` — Set Which Path Is Active

Sets which path index should be returned by subsequent calls to `GetCurrPath()`. Used when the user selects a different routing in a dialog or from a command. The `currPath` pointer is nulled so that `GetCurrPath()` will recompute it on the next call (ensuring consistency after modification).

---

### `WriteCompoundPathsEndPtsSegs(f, paths, segCnt, segs, endPtCnt, endPts)` — Write Turnout to File

Outputs a turnout's geometry in text form:
- Each path as a quoted string
- End points with their position and angle
- Segment descriptors (polylines/arcs)

Format example:
```text
P "Main-to-Fast" 30.5 -45.0
P "Main-to-Slow" 28.0 -12.5
E 0.000 6.234 -90.000
...
<segment data>
```

---

### `ParseCompoundTitle(title, manufP, nameP, partnoP)` — Parse Turnout Title String

Turnout titles use a tab-delimited format: `"Manufacturer<TAB>Name<TAB>Part No"`. This function splits the string into three fields (manufacturer, model name, part number) and stores them as pointers into substrings of the original title. The lengths are returned via `manufL`, `nameL`, `partnoL` so that comparisons can be done without full copies.

Used when loading from parameter files or database records.

---

### `FormatCompoundTitle(format, title)` — Format a Title for Display

Takes a bitmask of flags (`LABEL_MANUF`, `LABEL_DESCR`, `LABEL_PARTNO`, `LABEL_TABBED`, `LABEL_FLIPPED`, etc.) and builds a formatted string into the global `message[]` buffer. Used when rendering labels on screen or writing to files. The flags control whether each field is shown, whether tabs are inserted between fields, and whether special prefixes ("Flipped ", "Ungrouped ") are prepended.

---

### `ComputeCompoundBoundingBox(trk)` — Compute Tight Bounding Box

Walks all segments of the compound object (after applying origin offset and rotation) to compute an axis-aligned bounding box. This is used for viewport culling: if the entire object lies outside the visible window, it doesn't need to be drawn.

---

### `FindCompound(type, scale, title)` — Look Up a Turnout Definition by Name

Searches through global arrays of turnout definitions (`turnoutInfo_da`) and pier structure definitions (`structureInfo_da`). Returns a pointer to the matching definition (or NULL). Used when loading from parameter files or database records.

---

### `CompoundClearDemoDefns()` — Remove Demo-Only Definitions

When running in demo mode, only prototypes that are *not* marked with scale "DEMO" should be kept. This function iterates all turnout and structure definitions and zeroes out the segment count for those loaded from a parameter file tagged as DEMO. Effectively removes them from memory without explicit deletion — the next load will simply skip them.

---

### `SetDescriptionOrig(trk)` / `DrawCompoundDescription(...)` — Compute & Draw Label Position

When a turnout or pier is selected, its title is drawn at an offset position relative to the object's origin (or center). This allows labels to move with the object when it's dragged around. The code computes where the label *should* appear and draws a line showing that position (in white over black) so the user can see exactly where the label will end up.

The `descriptionOff` field stores an offset in world coordinates that is relative to the origin point — this allows the user to "drag" the label position by moving it around before confirming with mouse-up.

---

### `DistanceCompound(track_p t, coOrd *p)` — Raycast Distance for Compound Objects

Computes the distance from point `p` to the nearest feature of a compound object (turnout or pier). The algorithm is:
1. If on track in split mode and endpoints exist, use the standard segment-distance function.
2. If not in train program mode or no endpoints, compute distance via `DistanceSegs()` — which walks each segment's geometry.
3. For curved segments (`SEG_CRVTRK`), transform the point into local coordinates (rotate by `-angle`, translate by `-orig`) and then use path-based distance computation that handles rail-switching paths correctly.

This is needed for **selection raycasting** — when you click on a turnout, XTrkCAD must determine which endpoint or surface is closest to your mouse cursor.

---

### `CompoundDescriptionMove(track_p trk, wAction_t action, coOrd pos)` — Handle Dragging the Label Position

A state-machine that lets the user drag the label position:
- **C_DOWN**: Enter edit mode; draw a white outline of where the label will appear.
- **C_MOVE / C_UP**: Compute delta from start position (`p0`) to current mouse position, store as `descriptionOff`. Redraw with updated offset if in edit mode.

This is essentially an interactive UI control: "click and drag to move the description label." The code draws a preview line while dragging so the user can see exactly where it will land.

---

### `UpdateCompound(trk, inx, descUpd, needUndoStart)` — Apply Changes from Describe Dialog

This is the **main update dispatcher**. It takes an index into the `compoundDesc[]` enum and applies the corresponding change to the track object:
- `-1` / `MN` / `NM` / `PN`: Update title fields (manufacturer name, model name, part number). Checks if anything actually changed; if not, returns early. If changes were made, marks them as dirty so they'll be redrawn on next frame.
- `OR`: Move the entire object by a delta vector (origin change)
- `A0`–`A3`: Rotate around endpoint 0–3
- `AN`: Global rotation with pivot point selection
- `E0`–`E3`: Translate so that a specific endpoint is at a new position
- `Z0`–`Z3`: Set elevation of an endpoint, then recompute grade between elevations 0 and 1
- `LT`: Change line type (dashed/dotted/solid)
- `LY`: Change layer

After applying changes, it calls `DrawNewTrack(trk)` to redraw the object with its new appearance.

---

### `DescribeCompound(trk, str, len)` — Build a Description String for Save/Load

Writes all fields of the compound object into a buffer (e.g., when saving to a parameter file). The format is tab-separated values similar to the title parsing above. This function iterates over each field in `compoundDesc[]` and appends its value to `str`. It handles special cases like flipped/ungrouped/split prefixes being stripped from the name field before writing.

---

## Design Decisions & Tradeoffs

### Why Store Paths as a NULL-Terminated String Array?

Turnout paths are essentially free-form strings (they can contain spaces, quotes, special characters). Using `char*` pointers into a contiguous array is memory-efficient and simple to parse back from text files. The caller owns the data; `SetPaths()` copies it so that freeing the original doesn't break anything later.

### Why Have Separate "End Pt" Fields for Rotation/Translation?

The describe dialog allows editing any endpoint individually. Each endpoint (E0, E1, E2, E3) has its own field entry. This decouples the *data structure* (a single `coOrd pos` array in `compoundData`) from the *UI representation* (four separate fields). The enum-based dispatch (`switch(inx)`) maps UI index → action to apply.

### Why Use a Pivot Point for Rotation?

The pivot field lets the user choose which endpoint serves as the center of rotation. This is useful for "flipping" a turnout: you can click a different endpoint and rotate 180° around it, effectively mirroring the geometry. The code computes a new origin that is offset from the current one by half the bounding box (for mid-pivot) or twice the displacement (for secondary pivot).

### Why Two-Pass Update in `UpdateCompound`?

The function first checks if title fields changed; if not, returns early to avoid unnecessary redraws. Then it applies geometric changes (move/rotate/elevate). This separation avoids redundant computations and keeps the code readable — each case is a self-contained transformation step.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `GetPaths(trk)` | Return pointer to turnout path array (NULL if not a turnout) | track pointer |
| `SetPaths(trk, paths)` | Store a new set of paths into the extra-data structure; frees old one first | track, path string array |
| `GetCurrPath(trk)` | Get pointer to current active path in a multi-path turnout (iterator-style enumeration) | track pointer |
| `SetCurrPathIndex(trk, position)` | Set which path index is currently active (for switching between alternatives) | track, 0-based index |
| `ParseCompoundTitle(title, ...)` | Split a tab-delimited title string into manufacturer/model/partno fields; return substring pointers and lengths | full title string |
| `FormatCompoundTitle(format, title)` | Build a formatted display string from component parts based on flag bitmask | format flags, buffer pointer |
| `ComputeCompoundBoundingBox(trk)` | Compute tight AABB around all segments of the compound object (after transform) | track pointer |
| `FindCompound(type, scale, title)` | Look up a turnout or pier definition in global arrays by name and scale | type mask, scale string, model name |
| `WriteCompoundPathsEndPtsSegs(...)` | Serialize a turnout's paths/endpoints/segments to a text file (for param files) | FILE*, path array, segment count & pointer, endpoint count & array |
| `DistanceCompound(trk, p)` | Raycast from point p to nearest feature of a compound object; used for selection hit-testing | track pointer, cursor position output |
| `UpdateCompound(trk, inx, descUpd, needUndoStart)` | Apply one field's change (from describe dialog) to the track object and redraw it | track, enum index describing which field changed, whether to start undo |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Manage compound track objects (turnouts, pier structures); handle multiple path definitions per turnout; compute raycast distances for selection; support description label editing via drag-to-move |
| **Domain** | Compound object data model: turnouts that can have multiple routing paths, pier structures with multiple columns, grouping of sub-components into a single selectable entity |
| **Key concept** | A "compound" track is one whose geometry is defined by an array of `trkSeg_t` segments stored in its extra-data. Turnout-specific data includes a `paths` string array that encodes alternative routings (e.g., double-slip paths). The description label position is offset from the object's origin and moves with it, allowing attached labels to travel when the object is dragged. |
| **Main entry points** | `GetPaths()` / `SetPaths()` — manage turnout path definitions; `DistanceCompound()` — used by the raycaster for hit-testing compound objects |
