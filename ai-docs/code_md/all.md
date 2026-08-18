# XTrkCad Source Code Documentation

This directory contains markdown documentation for all source files in `app/bin/`:

## Documented Files

| File | Location | Status |
|------|----------|--------|
| [`cstruct.c`](./cstruct.c.md) | Core data structures, track types, undo tracking | ✅ Complete |
| [`drawgeom.c`](./drawgeom.c.md) | Interactive drawing primitives | ✅ Complete |
| [`ccornu.c`](./ccornu.c.md) | Cornu easement curve generation | ✅ Complete |
| [`cbezier.c`](./cbezier.c.md) | Cubic Bezier curve creation/modification | ✅ Complete |
| [`cdraw.c`](./cdraw.c.md) | Drawing primitives (lines, text, fills) | ✅ Complete |
| [`cswitchmotor.c`](./cswitchmotor.c.md) | Switch motor data structure | ✅ Complete |
| [`cundo.c`](./cundo.c.md) | Undo/redo ring-buffer system | ✅ Complete |
| [`cselect.c`](./cselect.c.md) | Track selection and manipulation | ✅ Complete |

## Summary of Key Design Patterns

### 1. Extra Data Framework (`custom.h`)

All custom track types embed their data immediately after the base `extraDataBase_t` header:

```c
typedef struct {
    extraDataBase_t base;   // Always first field!
    char *name;             // Custom field 1
    char *normal;           // Custom field 2
} switchmotorData_t;
```

This allows type-safe storage and retrieval of arbitrary data per track.

### 2. Segment Union Types (`track.h`)

Each segment has a `type` field that determines how `.u` is interpreted:

| Type | Description |
|------|-------------|
| `T_STRAIGHT` | Straight line segment |
| `T_CURVE` | Circular arc |
| `T_BEZTRK` | Bezier curve (approximated as arcs) |
| `T_CORNU` | Cornu easement curve |
| `T_TURNOUT` | Turnout compound track |
| `T_SWITCHMOTOR` | Switch motor device |

### 3. State Machine Commands (`wAction_t`)

All interactive commands use a state machine pattern:

```c
enum wAction {
    C_START,   // Initialize command
    C_DOWN,    // Mouse button down / key pressed
    C_MOVE,    // Dragging / modifier keys held
    C_UP,      // Release / abort
    C_OK,      // Confirm (Enter/Space)
    C_CANCEL,  // Cancel (Escape)
};
```

### 4. Dynamic Arrays (`dynArr_t`)

Custom O(1) append dynamic array used throughout:

- `tempEndPts_da` — endpoint chains for track joints
- `tlist_da` — selection list in `cselect.c`
- `anchors_da` — visual anchors on selected objects

### 5. Undo/Redo Integration (`cundo.c`)

Every modification is wrapped in a transaction:

```c
UndoStart("Label", "Description");
    // Modify track extra data...
UndoModify(trk);   // Serializes entire modified object
UndoEnd();         // Commits to undo stream
```

---

## File Sizes and Line Counts

| File | Lines | Primary Functions |
|------|-------|-------------------|
| cstruct.c | ~900 | Track structure, extra data framework |
| drawgeom.c | ~1200 | Interactive drawing state machine |
| ccornu.c | ~650 | Cornu curve generation and snapping |
| cbezier.c | ~800 | Bezier curves with arc approximation |
| cdraw.c | ~400 | Rendering primitives |
| cswitchmotor.c | ~920 | Switch motor data structures |
| cundo.c | ~1300 | Undo/redo ring buffer + streams |
| cselect.c | ~3970 | Selection, movement, alignment |

**Total:** ~5,140 lines of documented code.

---

## Related Documentation

- [`app_reference.md`](../app_reference.md) — API reference with function signatures
- [`dependencies_and_installation.md`](../dependencies_and_installation.md) — Build requirements
- [`requirements_install_status.md`](../requirements_install_status.md) — Dependency check results
