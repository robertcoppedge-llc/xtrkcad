# Source Code Documentation: `app/bin/` Directory

This directory contains the core implementation modules for XTrkCad's track geometry, drawing, and interactive operations. The following files have been documented:

| File | Description |
|------|-------------|
| [`cstruct.c`](cstruct.c.md) | Core data structures (track, segment union types, bounding boxes, undo/redo transaction tracking) |
| [`compound.c`](compound.c.md) | Compound track utilities: path management for turnouts, title formatting, description handling |
| [`drawgeom.c`](drawgeom.c.md) | Interactive drawing primitives using a state machine (lines, arcs, circles, polygons, benchwork) |
| [`ccornu.c`](../cornu/all.md) | Cornu easement curve generation using Raph Levien's Bezier approximation approach |
| [`cbezier.c`](../cornu/spiroentrypoints.c.md) | Cubic Bezier curve creation and modification with arc approximation for rendering |
| [`cdraw.c`](../lib/cdraw.c.md) | Drawing primitives: lines, polylines, text, filled regions; line styles (solid, dashed, phantom) |
| [`cswitchmotor.c`](cswitchmotor.c.md) | Switch motor data structure and management commands for layout control system |
| [`custom.c`](custom.c.md) | Layout control custom object manager: registration, deletion, enumeration, highlighting |
| [`cturnout.c`](cturnout.c.md) | Turnout (point machine) geometry: database-driven switching logic with multiple routes per turnout |
| [`cstruct.c`](cstruct.c.md#structure-objects) | Structure object placement via hotbar dialog with bounding-box computation |
| [`cundo.c`](cundo.c.md) | Ring-buffer undo/redo with full extra-data support and nested transaction handling |
| [`command.c`](command.c.md) | Command infrastructure: command menu, track type registration, undo integration |

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

## Summary Statistics

- **Total files in `app/bin/`**: 80+ source files (`.c`, `.h`)
- **Documented so far**: **12** core implementation files
- **Remaining major categories to document**: layout control (`wlib/`), geometry math (`lib/` / `cornu/`), utilities (`misc.c`, `fileio.c`, etc.)

---

## How to Use These Documents

These markdown documents are intended for:
1. **New contributors** — understand the architecture and design patterns of XTrkCad's track system.
2. **Existing developers** — find a quick reference for what each module does without reading all 80+ source files.
3. **Users / testers** — understand which features are implemented where and how they work under the hood.

---

## Quick Reference: What Each Module Does

| File | One-sentence summary |
|------|---------------------|
| `cstruct.c` | Defines the fundamental `track_t`, `trkSeg_t`, and `extraDataBase_t` structures that every track type builds upon. |
| `compound.c` | Provides generic compound track handling: path strings for turnouts, title parsing, description formatting, bounding box computation, file I/O. |
| `drawgeom.c` | Implements a state-machine-based interactive drawing system (lines, arcs, circles, polygons) that can be reused by any command needing to draw under the cursor. |
| `ccornu.c` / `spiro.c` | Generates smooth Cornu spiral transitions between two endpoints using Bezier curve approximation. |
| `cbezier.c` | Creates and modifies cubic Bezier curves, converting them into sequences of straight/arc segments for rendering. |
| `cdraw.c` | Low-level drawing primitives (line, polyline, text, filled shape) with support for multiple line styles and colors. |
| `cturnout.c` | Implements turnout-specific logic: database lookup by manufacturer/model, route selection, flip detection, adjustable switches. |
| `cswitchmotor.c` | Manages switch motors — external devices that control turnouts (e.g., Lenz JMRF, Digitrax DCS400). |
| `custom.c` | Registers custom track objects with the layout control system; handles edit/delete commands and enumeration popups. |
| `cundo.c` | Provides a ring-buffer undo/redo system that can record and replay arbitrary changes to any object's extra data. |
| `command.c` | Sets up command menus, hotbars, and tracks their undo integrations via a registration table (`cmdTable`). |

---

## Next Steps

The following directories remain to be documented:

- **`app/wlib/`** — Layout control logic (block detection, signal interlocking, turnout switching)
- **`app/lib/`** — Geometry utilities (angle normalization, distance computation, path parsing)
- **`app/cornu/spiro.*`** — Raph Levien's spiral approximation library
