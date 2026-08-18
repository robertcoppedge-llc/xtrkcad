# cswitchmotor.c — Switch Motor (Turnout Control) Objects

## Overview

`cswitchmotor.c` implements **Switch Motors** (`T_SWITCHMOTOR`), a compound track object that associates a turnout (switch/motorized point machine) with a specific switch motor device. This allows the program to know which turnouts are controlled by which motors, what their normal/reverse positions are called, and how they sense their current state ("point sense").

Switch motors are used in layout control systems where turnout position is monitored or automatically operated via external controllers (e.g., Lenz JMRF, Digitrax DCS400, etc.). The switch motor object links the CAD geometry to a real-world device.

---

## Key Data Structures

### `switchmotorData_t` — Extra Data for a Switch Motor Track

Attached as extra data on a track of type `T_SWITCHMOTOR`:

```c
typedef struct switchmotorData_t {
    extraDataBase_t base;        // Generic header: index, layer, visible, etc.
    char *name;                  // Human-readable name (e.g., "S8")
    char *normal;                // Name of the "normal" route (default path)
    char *reverse;               // Name of the alternate route
    char *pointsense;            // Description of how the motor senses position
                                 //   ("input 7 low", "sensor on input 3")
    BOOL_T IsHilite;             // TRUE if currently highlighted (for debugging)
    TRKINX_T turnindx;           // Index of the turnout track it controls (-1 if none yet)
    track_p turnout;             // Pointer to the associated turnout track
    track_p next_motor;          // Linked-list pointer: next switch motor object
} switchmotorData_t, *switchmotorData_p;
```

- `next_motor` links all switch motors into a global doubly-linked list (via `first_motor`, `last_motor`) so they can be iterated over in undo/redo or file I/O.
- `turnout` is lazily resolved: initially NULL until the associated turnout track exists, then filled in by `ResolveSwitchmotorTurnout()`.

### Global Linked List State

```c
static track_p first_motor;   // Head of the switch motor linked list
static track_p last_motor;    // Tail of the switch motor linked list
```

Used for efficient iteration and deletion.

---

## Drawing and Bounding Box Computation

### `ComputeSwitchMotorBoundingBox(track_p t)` — Compute bounding box by projecting a polygon

The switch motor is drawn as a small icon (a stylized "motor" symbol) defined by 14 vertices (`switchmotorPoly_Pix[]`). The function:
1. Reads the turnout's origin and angle from its compound data.
2. Converts screen-to-world coordinates using the turnout's scale factor.
3. Transforms each polygon vertex around the turnout's center, accounting for rotation.
4. Computes min/max x/y from all transformed vertices to produce a bounding box.

The result is stored in `t->bbox` so that off-screen culling can skip redrawing this object.

### `DrawSwitchMotor(track_p t, drawCmd_p d, wDrawColor color)` — Draw the switch motor icon

Projects the same 14-vertex polygon (`switchmotorPoly_Pix[]`) into world space using the turnout's transform (origin/angle/scale). Draws it filled with the given color. The shape is a generic "motor" symbol resembling a small house or gear-like blob.

---

## Description Dialog

### `UpdateSwitchMotor(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)` — Edit callback for the description dialog

Handles field edits from the property dialog:
- **Name**: string field (non-blank required).
- **Normal**: name of the default route.
- **Reverse**: name of the alternate route.
- **Point Sense**: description of how position is sensed.

The function compares each new value against the old one; if different, it frees and reassigns the string pointer (preventing leaks). If no field changed, returns early. On OK, an undo block is started (`needUndoStart`).

---

## Distance Query and Description Output

### `DistanceSwitchMotor(track_p t, coOrd *p)` — Compute distance from point to switch motor center

Returns Euclidean distance from the given point to the midpoint of the bounding box. Used for hit testing. The nearest point is stored back in `*p`.

---

## Undo/Redo Support

### `DeleteSwitchMotor(track_p trk)` — Remove a switch motor object from the linked list and free its data

1. Frees the four string fields (`name`, `normal`, `reverse`, `pointsense`).
2. Unlinks the track from the doubly-linked global list:
   - If it was `last_motor` (tail), updates `last_motor` to point to `xx->next_motor`.
   - Walks the list with `first_motor` to find the node whose `next_motor` pointer points to this track, then splices it out.

---

## File I/O: Writing and Reading Switch Motors

### `WriteSwitchMotor(track_p t, FILE *f)` — Write a switch motor to an XTC file

Skips motors that have no associated turnout (they are considered orphaned or incomplete). Writes:

```
SWITCHMOTOR <index> <turnout_index> "<name>" "<normal>" "<reverse>" "<pointsense>"
```

The turnout index is the track index of the associated turnout. If no turnout exists yet, it's recorded as 0 and will be resolved later when the turnout object is read.

### `ReadSwitchMotor(char *line)` — Read a switch motor from an XTC file line

Parses a line beginning with `SWITCHMOTOR`. Extracts:
- Track index (first integer)
- Turnout track index (second integer, 0 means not yet linked to a turnout)
- Four string fields

Creates a new `T_SWITCHMOTOR` track record and inserts it into the global linked list. The associated turnout pointer is left NULL until later resolution.

---

## Resolving Turnout Association

### `ResolveSwitchmotorTurnout(track_p trk)` — Link a switch motor to its turnout by index

Called after loading from file or creating a new turnout object. Looks up the track at `xx->turnindx` and assigns it as `xx->turnout`. Then computes the bounding box (since the turnout may have been created since the last time). If the turnout doesn't exist, shows an error message but does not abort.

---

## Edit Dialog

### `EditSwitchMotor(track_p trk)` — Open the description dialog for a switch motor

Populates the edit dialog fields with the current values from the track's extra data. The user can modify name, routes, and point-sense descriptions. On OK, `SwitchMotorEditOk()` is called to apply changes under an undo block.

---

## Highlighting / Debugging

### `DrawSWMotorTrackHilite(void)` — Draw a gray highlight box around a switch motor

Used during debugging or when the "Highlight" command is active on a switch motor. The highlight rectangle surrounds the turnout geometry (the associated track object) rather than the abstract switch motor icon itself.

---

## Management Commands

### `SwitchmotorMgmProc(int cmd, void *data)` — Command dispatcher for management system integration

Handles messages from the layout control system manager:
- **CONTMGM_CAN_EDIT** → always allowed (return TRUE).
- **CONTMGM_DO_EDIT** → opens the edit dialog via `EditSwitchMotor()`.
- **CONTMGM_CAN_DELETE** → always allowed.
- **CONTMGM_DO_DELETE** → calls `DeleteTrack()` on the switch motor track, which in turn deletes its associated turnout if there is one (via `CheckDeleteSwitchmotor()`).
- **CONTMGM_GET_TITLE** → formats a title string including name and turnout index.

---

## Icon Loading

### `SwitchmotorMgmLoad(void)` — Register switch motors with the management system icon menu

Creates an icon from the embedded bitmap (`switch_motor_image3`) and registers each switch motor track as a controllable object. This integrates switch motors into the "Manage Layout Control Objects" dialog where users can edit or delete them.

---

## Initialization and Command Registration

### `InitTrkSwitchMotor(void)` — Initialize the T_SWITCHMOTOR track type

Calls generic `InitObject()` to register:
- `DrawSwitchMotor` as the drawing function
- `DistanceSwitchMotor` for hit testing
- `DescribeSwitchMotor` for description output
- `DeleteSwitchMotor`, `WriteSwitchMotor`, `ReadSwitchMotor` for lifecycle and persistence
- Empty stubs for move/rotate/rescale (switch motors don't support these operations directly)
- NULL for most advanced track-object operations

### `InitCmdSwitchMotor(wMenu_p menu)` — Register hotbar/menu entry

Creates a sticky popup menu button with the switch motor icon. When clicked, invokes `CmdSwitchMotorCreate()`, which prompts the user to select a turnout and then opens the creation dialog.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `ComputeSwitchMotorBoundingBox()` / `DrawSwitchMotor()` | Compute bounding box and draw the switch motor icon (projected from world space) |
| `UpdateSwitchMotor()` | Edit callback for the property dialog; updates strings under undo |
| `DistanceSwitchMotor()` | Distance query for hit testing |
| `DescribeSwitchMotor()` | Generate description text for the status bar or command line |
| `DeleteSwitchMotor()` | Remove from the global linked list and free memory |
| `WriteSwitchMotor()` / `ReadSwitchMotor()` | Serialize to/from XTC files (lazy linkage to turnout) |
| `ResolveSwitchmotorTurnout()` | Resolve NULL turnout pointer by index lookup |
| `EditSwitchMotor()` / `SwitchMotorEditOk()` | Open and handle the edit dialog |
| `DrawSWMotorTrackHilite()` | Draw highlight box during debugging or management mode |
| `SwitchmotorMgmProc()` | Dispatch layout control manager commands (edit/delete) |
| `SwitchmotorMgmLoad()` | Register switch motors with the icon-based management system |
| `InitTrkSwitchMotor()` / `InitCmdSwitchMotor()` | Initialize track type and hotbar entry |
| `CheckDeleteSwitchmotor()` | Called when deleting a turnout; warns about attached motors |

---

## Notes

- A single turnout can have multiple switch motors (multiple devices controlling the same turnout). The linked list (`first_motor` / `last_motor`) handles this.
- Deleting a turnout automatically removes all its associated switch motors and deletes those objects too.
- Switch motors are drawn as simple geometric icons; no raster image is used.
- The "point sense" field is intended to describe how the external controller reports position (e.g., which digital input line indicates "left" vs. "right").
