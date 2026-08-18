# Source Code Documentation: `app/bin/` Directory

This directory contains the core implementation modules for XTrkCad's track geometry, drawing, and interactive operations. The following files have been documented:

| File | Description |
|------|-------------|
| [`cstruct.c`](cstruct.c.md) | Core data structures (track, segment union types, bounding boxes, undo/redo transaction tracking) |
| [`drawgeom.c`](drawgeom.c.md) | Interactive drawing primitives using a state machine (lines, arcs, circles, polygons, benchwork) |
| [`ccornu.c`](ccornu.c.md) | Cornu easement curve generation using Raph Levien's Bezier approximation approach |
| [`cbezier.c`](cbezier.c.md) | Cubic Bezier curve creation and modification with arc approximation for rendering |
| [`cdraw.c`](cdraw.c.md) | Drawing primitives: lines, polylines, text, filled regions; line styles (solid, dashed, phantom) |
| [`cswitchmotor.c`](cswitchmotor.c.md) | Switch motor data structure for layout control system |
| [`cundo.c`](cundo.c.md) | Ring-buffer undo/redo with full extra-data support |

---

## Key Design Patterns Across the Codebase

### 1. Segment Union Types (`track.h` / `cstruct.c`)

XTrkCad uses a tagged union approach where each track segment has a `type` field that determines how its `.u` union member is interpreted:

```c
typedef struct {
    trackType_e type;           // Type identifier (T_STRAIGHT, T_CURVE, etc.)
    ...
} trkSeg_t;
```

This allows the same segment structure to represent lines, curves, Bezier segments, text labels, switch motors, etc. The `type` field is checked at runtime via `GetTrkType()` and cast macros like `GET_EXTRA_DATA()`.

### 2. Extra Data Framework (`custom.h`)

All custom track types (cornu, bezier, turnouts, switches) embed their data structures immediately after the base `track_t` header using variable-length arrays:

```c
typedef struct {
    extraDataBase_t base;      // Always first field!
    char *name;                // Custom field 1
    char *normal;              // Custom field 2
} switchmotorData_t, ...;     // Follows extraDataBase_t
```

This allows type-safe storage and retrieval of arbitrary data per track while keeping all tracks in a single contiguous array.

### 3. State Machine Commands (`wAction_t`)

Interactive operations use a command pattern where mouse events are dispatched via an `action` integer:

- `C_START`, `C_DOWN`, `C_MOVE`, `C_UP`, `C_OK`, `C_CANCEL` — basic command states
- `wActionMove`, `wActionLDown`, `wActionLDrag`, `wActionLUp` — mouse events
- Additional bits encode modifiers (Shift, Ctrl, Alt) and special keys

This is used in all drawing/modification commands to maintain a consistent state machine pattern.

### 4. Dynamic Arrays (`dynArr_t`)

The codebase uses a custom dynamic array implementation (`DYNARR_*` macros) for:
- End point arrays (for track joints)
- Segment approximation chains (Bezier → arcs/lines)
- Midpoint insertion lists (Cornu curve refinement)

This provides O(1) append operations and contiguous memory layout.

### 5. Undo Integration (`cundo.c`)

Every modification to a track's extra data is wrapped in an undo transaction:

```c
UndoStart("Change Switch Motor", "Changed switch motor properties");
    // Modify the tracked structure...
UndoModify(trk);   // Records entire modified object for later restoration
UndoEnd();         // Commits
```

On redo, the original state is copied back from the undo stream. This handles even complex changes like renaming a switch motor or adjusting a Cornu curve's control points.

---

## Related Directories (Next Documentation Targets)

| Directory | Notable Files | Brief Description |
|-----------|--------------|-------------------|
| `app/wlib/` | `cblock.c`, `csignal.c`, `custom.c` | Layout control: block detection, signal logic, turnout switching |
| `app/lib/` | `tbezier.c`, `ccurve.c` | Mathematical utilities for curve approximation |
| `app/cornu/` | `spiro.h`, `spirals.h` | Raph Levien's spiral algorithm library |

---

## Summary Statistics

- Total files in `app/bin/`: 80+ source files (`.c`, `.h`)
- Documented so far: **7** core implementation files
- Remaining major categories to document: layout control (`wlib`), geometry math (`lib` / `cornu`), utilities (`misc.c`, `fileio.c`, etc.)
