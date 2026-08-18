# cswitchmotor.c — Switch Motor (Layout Control) Module

## Overview

`cswitchmotor.c` implements the **switch motor** data structure and management for XTrkCad's layout control system. A switch motor is a device that physically moves a turnout (switch/point machine). It links a specific turnout in the track database to a real-world servo or relay controller, storing metadata such as normal position, reverse position, point sense direction, and name labels.

This module is part of the layout control subsystem (alongside `cblock.c` for block detection) that enables model railroads with programmable switching.

---

## Core Data Structure: `switchmotorData_t`

```c
typedef struct switchmotorData_t {
    extraDataBase_t base;       // Undo/redo chain header + type ID
    char * name;                // User-defined name (e.g., "Mainline Switch 1")
    char * normal;              // Name of turnout in NORMAL position
    char * reverse;             // Name of turnout in REVERSE position
    char * pointsense;          // Description of point-sense direction
    BOOL_T IsHilite;            // Flag for highlighting (debug/selection)
    TRKINX_T turnindx;         // Index of the associated turnout track
    track_p turnout;           // Pointer to the turnout track object
    track_p next_motor;        // Next motor in doubly-linked list (for iteration)
} switchmotorData_t, *switchmotorData_p;
```

| Field | Description |
|-------|-------------|
| `base` | Shared header with type ID `T_SWITCHMOTOR`; enables undo/redo integration and type identification |
| `name` | Human-readable identifier (e.g., "Switch A to B") |
| `normal` | Name of the track configuration when motor is in normal position |
| `reverse` | Name of the track configuration when motor is in reverse position |
| `pointsense` | Description of which way points are set (e.g., "Points toward Mainline", "Normal=Main") |
| `turnindx` | The index (`TRKINX_T`) of the turnout track this motor controls |
| `turnout` | Direct pointer to the turnout track object (for quick access) |
| `next_motor` | Pointer to next motor in a linked list — used for iterating over all motors during undo/redo or layout control operations |

---

## Drawing Functions

### `ComputeSwitchMotorBoundingBox(track_p t)`

Computes the axis-aligned bounding box of the switch motor graphic. The graphic is a polygon defined by `switchmotorPoly_Pix[]` (15 vertices). The function:
- Retrieves the turnout's origin and angle from its compound extra data
- Transforms each polygon vertex using the turnout's origin, angle, and scale
- Computes min/max x/y to establish the bounding box

Used for viewport culling — if a motor falls outside the viewport, it doesn't need rendering.

---

### `DrawSwitchMotor(track_p t, drawCmd_p d, wDrawColor color)`

Renders the switch motor symbol onto a drawing command context. The polygon is scaled by `switchmotorPoly_SF` (default 3.0) and transformed into screen coordinates using the turnout's geometry data. Filled with the specified color.

The graphic resembles a stylized "point" or switch arm — an irregular polygon that visually represents the mechanical device.

---

## Update/Delete Functions

### `UpdateSwitchMotor(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)`

Updates a motor's properties (name, normal/reverse position labels, point sense description). Handles:
- String truncation if input exceeds buffer size (`STR_SHORT_SIZE` or `STR_LONG_SIZE`)
- Memory management: frees old string pointers and allocates new ones via `MyStrdup()`
- Calls `UndoStart()`/`UndoModify()` when changes occur to record the operation for undo history

**Parameters:**
- `trk` — The switch motor track
- `inx` — If -1, update ALL motors from current dialog state; otherwise update only this specific motor
- `descUpd` — Pointer to the parametric description array containing new values (populated by GTK widgets)
- `needUndoStart` — If TRUE and changes were made, begin a new undo transaction

---

### `DeleteSwitchMotor(track_p trk)`

Removes a switch motor from its doubly-linked list. It:
1. Frees all string pointers (`name`, `normal`, `reverse`, `pointsense`)
2. Adjusts the linked list by updating `next_motor` pointer of the preceding node
3. Updates head/tail pointers if this was the first or last element

The associated turnout track remains intact — only the motor relationship is removed.

---

## Serialization (File I/O)

### `WriteSwitchMotor(track_p t, FILE *f)`

Writes a motor's data to the layout control file format:

```text
SWITCHMOTOR <trkindex> <turnout_index> "name" "normal" "reverse" "pointsense"\n
```

The turnout index is written as an integer; all string fields are enclosed in double quotes. If the associated turnout does not exist (`turnout == NULL`), returns `FALSE` (no file output).

---

### `ReadSwitchMotor(char *line)`

Parses a line from the layout control file. Expects format:

```text
SWITCHMOTOR <trkindex> <turnout_index> "name" "normal" "reverse" "pointsense"
```

Allocates new track, allocates and copies strings, links into doubly-linked list (`first_motor` ↔ `last_motor`). Calls `switchmotorDebug()` for logging.

---

## Describe Function

### `DescribeSwitchMotor(track_p trk, char *str, CSIZE_T len)`

Populates a description string by copying field values from the motor data structure:
- Trims each label to fit within its buffer size (`STR_SHORT_SIZE` or `STR_LONG_SIZE`)
- Converts turnout index to a human-readable format
- Prepares a parametric description list for display in a dialog widget

Used when displaying motor properties in an edit dialog.

---

## Distance Helper

### `DistanceSwitchMotor(track_p t, coOrd *p)`

Computes the distance from a given point `p` to the center of the switch motor graphic (computed as the midpoint of its bounding box). Returns the distance and stores the center coordinates into `*p`. Used for hit-testing during mouse-down events in layout control mode.

---

## Linked List Maintenance

The motors are maintained in a doubly-linked list via `next_motor` pointers, with global head (`first_motor`) and tail (`last_motor`) pointers. This allows:
- Fast iteration over all motors (e.g., for bulk undo/redo)
- O(1) deletion without searching
- Efficient traversal when building lists of affected objects

---

## Related Files

| File | Purpose |
|------|---------|
| `cswitchmotor.h` | Type definitions (`switchmotorData_t`, type IDs, enums) |
| `compound.c/compound.h` | Compound track types (turnouts, switches) — motors are attached here |
| `custom.c/custom.h` | Custom data storage framework (used by the undo system for switch motors) |
| `cundo.c/cundo.h` | Undo transaction management |
| `fileio.c/fileio.h` | Layout control file I/O utilities |

---

## Usage in Context

The switch motor is a **compound** track type — it does not represent physical track geometry but rather controls a turnout via an external servo/relay. Its primary role is to:
1. Store configuration (which turnout it moves, what positions correspond to normal/reverse)
2. Provide user-facing labels for layout control software
3. Enable undo/redo of motor property changes

The actual physical movement is handled by external hardware and communicated back via GPIO or serial interface from a layout controller (e.g., Arduino, Raspberry Pi, dedicated PLC).
