# cswitchmotor.c — Switch Motor Management for Turnout Control

## Overview

`cswitchmotor.c` implements **Switch Motors** — a layout control mechanism that associates a track-mounted device (e.g., a motorized turnout switch) with a specific turnout track segment. When the switch motor is triggered, it changes the position of its associated turnout, enabling automatic routing based on physical sensor input.

This module provides:
- Creation and editing of switch motors via dialog windows
- Deletion of unused switch motors
- Drawing of the switch motor symbol (a small arrow pointing to the turnout)
- Hit testing against the switch motor icon for selection
- Context menu integration for "Edit" and "Delete" actions
- Undo/redo support

## File Location

```
app/bin/cswitchmotor.c  (917 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `compound.h` | Turnout data structures (`extraDataCompound_t`) |
| `cselect.h` | Selection utilities, track layer checks |
| `cundo.h` | Undo stack management |
| `custom.h` | Custom widget types (`wWin_p`, `wString_p`) |
| `fileio.h` | File I/O for serialization |
| `param.h` | Parameter dialog framework |
| `track.h` | Track data structures, bounding box utilities |

## External Variables

```c
EXPORT TRKTYP_T T_SWITCHMOTOR = -1;   /* track type ID for switch motors */
```

> **Note:** Switch motor tracks are *not* actual geometric tracks. They are auxiliary objects that live in a linked list (`first_motor` ↔ `last_motor`) and are associated with turnouts via the `turnout` pointer in their `extraDataCompound_t`.

## Data Structures

### `switchmotorData_t` — Per-switch-motor data stored as extra data on its host track

```c
typedef struct switchmotorData_t {
    extraDataBase_t base;           /* header for all extra data */
    char * name;                     /* human-readable label, e.g. "Switch 5" */
    char * normal;                   /* turnout position when motor is in "normal" (up) state */
    char * reverse;                  /* turnout position when motor is in "reverse" (down) state */
    char * pointsense;               /* description of what the switch detects, e.g. "12VDC sensor on #3 rail" */
    BOOL_T IsHilite;                 /* flag: has a hilite box been drawn around this? */
    TRKINX_T turnindx;              /* index of the turnout track it controls (0-based) */
    track_p turnout;                /* pointer to the turnout track itself (NULL if deleted) */
    switchmotorData_p next_motor;   /* doubly-linked list: next motor in the chain */
} switchmotorData_t, *switchmotorData_p;
```

### Global Linked List of All Switch Motors

A simple doubly-linked list tracks all active switch motors for efficient iteration and deletion.

```c
static track_p last_motor;         /* last node in the linked list (tail) */
static track_p first_motor;        /* first node (head, initialized to NULL) */
```

## Core Functions

### `ComputeSwitchMotorBoundingBox(track_p t)`

Computes the bounding box of the switch motor symbol on its host turnout track. The symbol is a 10×24 pixel polygon (defined in `switchmotorPoly_Pix[]`) scaled by `3.0` and transformed to match the turnout's scale and rotation angle.

**Steps:**
1. Retrieve the associated turnout from the extra data via `GET_EXTRA_DATA(trk, T_SWITCHMOTOR, switchmotorData_t)`.
2. Extract the turnout's origin (`orig`) and angle (`angle`).
3. Get the turnout's scale index and convert to a pixel/scale ratio.
4. Rotate each polygon vertex by `(90 - angle)` around the turnout center (which is at `orig + offset` from the turnout endpoint).

### `DrawSwitchMotor(track_p t, drawCmd_p d, wDrawColor color)`

Draws the switch motor symbol on a track's bounding box. The polygon consists of:
- A solid rectangle body (filled)
- An arrowhead at one end pointing toward the turnout center

The entire shape is clipped to the track's bounding box implicitly via normal draw clipping.

### `DescribeSwitchMotor(track_p trk, char *str, CSIZE_T len)`

Fills a string buffer with a human-readable description of the switch motor, including its name and associated turnout index. This is used by the "Describe" command and other UI text displays.

### `UpdateSwitchMotor(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)`

Updates the internal strings (`name`, `normal`, `reverse`, `pointsense`) when the user modifies them in a dialog. It:
- Truncates new string values if they exceed their buffer sizes and warns the user
- Frees old pointers and replaces with newly allocated ones
- Wraps the entire update in an Undo transaction via `UndoStart`/`UndoEnd`

### `DistanceSwitchMotor(track_p t, coOrd *p)`

Returns the Euclidean distance from a given point to the center of the switch motor's bounding box. Used for hit testing during mouse-down events.

### `DeleteSwitchMotor(track_p trk)`

Removes a switch motor from both its associated list and memory:
- Frees all allocated string fields (`name`, `normal`, `reverse`, `pointsense`)
- Unlinks the node from the doubly-linked list (updates `next_motor` pointer of predecessor)
- Updates `first_motor` / `last_motor` pointers

### `SwitchMotorOk(void *junk)` / `SwitchMotorEditOk(void *junk)`

Dialog OK handlers for creation and editing. They:
1. Validate that a non-empty name was entered
2. Wrap the object creation in an Undo transaction
3. Allocate new strings from dialog fields
4. Create a fresh `switchmotorData_t` record attached to a newly created turnout track (index 0)
5. Insert into the linked list and draw the symbol

### `NewSwitchMotorDialog(track_p trk)` / `EditSwitchMotor(track_p trk)`

Opens parameter dialogs that let users enter/edit:
- Name
- Normal position (e.g. "straight")
- Reverse position (e.g. "left")
- Point sense description
- Turnout number (read-only, derived from the associated turnout index)

### `SwitchmotorMgmLoad(void)`

Iterates over all tracks and registers each switch motor as a context menu item with:
- An icon (`switch-motor.image3`)
- A custom callback `SwitchmotorMgmProc` that handles:
  - `CAN_EDIT` / `DO_EDIT` → opens the edit dialog
  - `CAN_DELETE` / `DO_DELETE` → deletes the switch motor (with confirmation)
  - `DO_HILIGHT` / `UN_HILIGHT` → draws/removes a hilite box around the track

### `CheckDeleteSwitchmotor(track_p t)`

Called during turnout deletion. It walks the linked list looking for any switch motors whose `turnout == t` and deletes them, printing an info message for each one removed.

## Enums & Constants

```c
enum { NM, NOR, REV, PS, TO }   /* index into switchmotorDesc[] */
/*
  NM    — Name field
  NOR   — Normal position string
  REV   — Reverse position string
  PS    — Point sense description
  TO    — Turnout number (read-only)
*/

enum { DESC_STRING, DESC_LONG, DESC_RO }  /* descriptor types for UI */
```

## Serialization (File I/O)

### `WriteSwitchMotor(track_p t, FILE *f)`

Writes a switch motor to the XTrkCad file format:

```text
SWITCHMOTOR <track-index> <turnout-index> "<name>" "<normal>" "<reverse>" "<pointsense>"
```

If the associated turnout is missing, the function returns `FALSE` (skip writing).

### `ReadSwitchMotor(char *line)`

Parses a line of the format file and creates a new switch motor track:
- Creates a fresh turnout track at index 0 (switch motors are always "type -1" / auxiliary)
- Allocates strings from the parsed fields
- Inserts into the doubly-linked list (prepended to `first_motor`)

## Notes

- Switch motors have **no geometric geometry** — they draw only on top of their host turnout's bounding box. Their own bounding box is computed dynamically by projecting a fixed polygon onto the turnout's coordinate frame.
- They are stored as separate track objects with type `T_SWITCHMOTOR` so that the undo system can record deletions, but they don't appear in normal track lists or be selected directly (selection goes through context menu).
- The linked list (`first_motor` ↔ `last_motor`) allows O(1) deletion without a full scan of all tracks.
