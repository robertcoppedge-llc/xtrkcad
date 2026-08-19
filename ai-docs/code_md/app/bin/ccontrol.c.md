# ccontrol.c — Control Objects (Signal Detectors)

## Overview

`ccontrol.c` implements **controls** in XTrkCad. A control is a circular marker placed at a specific location on the layout that represents a signal detector or controller device. Controls are used to:
- Represent physical detectors mounted on track or structures
- Attach scripts (e.g., `detector.pl`, `signal.pl`) that execute when their associated track passes over them
- Group related tracks into logical units (like a switch machine controlling multiple turnouts)

## File Location

```
app/bin/ccontrol.c  (653 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `compound.h` | Compound track iteration macros (`TRK_ITERATE`) |
| `cselect.h` | Selection utilities (`SetAllTrackSelect`, `SnapPos`) |
| `cundo.h` | Undo/redo transaction support (`UndoStart`, `UndoEnd`, `UndoModify`) |
| `custom.h` / `fileio.h` | Custom types, file I/O utilities |
| `layout.h` | Layout context (scale lookup) |
| `param.h` | Parameter dialog system (`ParamCreateDialog`, etc.) |
| `track.h` | Track type definitions and accessors |
| `common-ui.h` | UI dialogs, messages, icons |

## Key Concepts

### Control as a Standalone Track Type

Controls are stored in the track list just like any other object but with `T_CONTROL` as their type code:

```c
EXPORT TRKTYP_T T_CONTROL = -1;  // Defined here (uninitialized)
```

They have no constituent geometry — they are purely logical markers at a point on the layout.

### Control Data Structure

```c
typedef struct controlData_t {
  extraDataBase_t base;   // Base structure linking to track object
  coOrd orig;             // The center position of the control marker (model coords)
  BOOL_T IsHilite;        // Flag: has this control been highlighted?
  char   *name;           // User-friendly name (e.g., "Detector A")
  char   *onscript;       // Script executed when track passes over (ON script)
  char   *offscript;      // Script executed when track leaves OFF zone (OFF script)
} controlData_t, *controlData_p;
```

The `base` field is embedded in the track's extra data area and links back to the main track object so the control can be manipulated through the standard compound track API.

### Control Marker Geometry

A control is drawn as a circle with three spokes radiating from its center (like a propeller or detector symbol). The drawing parameters are:
- `RADIUS = 6` — radius of the filled circle (in model units)
- `LINE = 8` — length of each spoke segment beyond the circle
- `control_SF = 3.0` — scale factor applied to both

The bounding box is computed to include the full circle plus all three spokes, so it can be highlighted as a single rectangle by the compound track manager.

## Command Interface (`controlCmds`)

| Member | Function | Description |
|--------|----------|-------------|
| draw   | `DrawControl` | Draws the circular marker with three spokes |
| distance | `DistanceControl` | Distance from point to control center |
| describe | `DescribeControl` | Populates description dialog (name, position, scripts) |
| delete  | `DeleteControl` | Frees name and script pointers; unlinks from global list |
| write   | `WriteControl` | Writes `"CONTROL <index> <layer> <scale> <visible> <x> <y> "name" "onscript" "offscript"\n"` to file |
| read    | `ReadControl` | Parses a control line from an `.xtp` file and reconstructs the object |
| move    | `MoveControl` | Translates the control's center position (for undo/redo support) |
| rotate  | empty | Not applicable — controls have no rotation semantics |
| rescale | empty | Not applicable |
| flip    | `FlipControl` | Flips the control across a point (rarely used) |

## File Format

Controls appear in `.xtp` files as:

```text
CONTROL <index> <layer_number> <scale_code> <visible> <X> <Y> "name" "ON_script" "OFF_script"
```

Example:

```text
CONTROL 5 2 N 1 -450.732 -1829.654 "SwitchMachine_1" "switch.pl" ""
```

The `ReadControl()` function parses this line using `GetArgs()`:
- Index is extracted to allocate the track object
- Layer and scale are looked up via `GetTrkLayer()` / `LookupScale()`
- Visibility flag determines whether the control is drawn
- The name and two script strings are duplicated into dynamically allocated memory

## Core Functions

### `DrawControl(track_p t, drawCmd_p d, wDrawColor color)`

Draws a single control marker:
1. Fetches its `controlData_p` via `GetcontrolData()`.
2. Draws the filled circle at `orig`.
3. Draws three line segments radiating from the center at 0°, 90°, and 180° relative to the drawing coordinate system (the spokes).

The scale ratio is computed from the current layout's DPI so that the control renders at a consistent physical size regardless of zoom level.

### `DistanceControl(track_p t, coOrd *p)`

Returns the Euclidean distance from point `p` to the control center (`xx->orig`). Used by track collision detection and path-following algorithms. The closest point is written back into `*p`.

### `DescribeControl(track_p trk, char *str, CSIZE_T len)`

Populates a description string used in layout commands (e.g., "TRACK (5): Layer=3 Control: SwitchMachine_1"). It fills the internal buffers with the current values and sets up fields for the dialog.

### `UpdateControlProperties(...)`

The callback invoked when a user edits any field of the control in the description dialog. It compares new values against old ones, triggers undo start if anything changed, frees old strings, allocates new ones, updates the track's extra data, and recomputes the bounding box so highlighting remains correct.

### `DeleteControl(track_p trk)`

Removes the control from memory:
- Frees `name`, `onscript`, and `offscript` (which may point into the track's own data area).
- Does not delete the track itself — it simply strips its extra data so that subsequent operations treat it as a generic, untyped track.

### `WriteControl(track_p t, FILE *f)` & `ReadControl(char *line)`

File I/O pair that serializes/deserializes the control's index, layer, scale, position, visibility flag, name, and both scripts. The `ReadControl()` function also calls `ComputeControlBoundingBox()` to set up the highlight rectangle for future compound management operations.

### `MoveControl(...)` & `RotateControl(...)` / `RescaleControl(...)`

These are mostly empty stubs — controls do not support meaningful rotation or rescaling because they are always drawn at a fixed visual size and position relative to the layout scale. The flip operation (`FlipControl`) is implemented but rarely used in practice.

### `CmdControl(wAction_t action, coOrd pos)`

The interactive creation command:
- **START**: Shows "Place control" message; sets up for click-to-place.
- **DOWN/MOVE**: As the mouse moves near a track or structure, snaps to that point and displays a preview circle (`DDrawControl`) in red. The position is cached in `control_pos`.
- **UP/OK**: Finalizes placement by calling `CreateNewControl(pos)`, which shows the edit dialog where the user enters the name and scripts.
- **CANCEL**: Discards the in-progress creation.

### `EditControl(track_p trk)` & `ControlEditOk(...)` / `CreateNewControl(coOrd orig)`

When a control is selected (from context menu or track list), `EditControl()` populates the internal buffers with its current values and shows the parameter dialog. The user can modify:
- Name (required, non-blank)
- Origin X/Y (model coordinates of the detector center)
- "On Script" — executed when a train passes over the control's trigger zone
- "Off Script" — executed when the train leaves that zone

The `ControlEditOk()` callback applies changes via undo transaction and redraws. If no track is associated (`controlEditTrack == NULL`), it creates a new control at the origin point instead of modifying an existing one.

### `DrawControlTrackHilite(void)` / `ControlMgmProc(...)`

Highlights a selected control by drawing a light gray semi-transparent rectangle around its bounding box. The bounding box is recomputed each time highlighting is toggled because controls can have arbitrary positions and may be flipped or otherwise transformed.

## Global State & Static Variables

| Variable | Purpose |
|----------|---------|
| `controlEditOrig` | Scratch variable for the control's origin while editing |
| `controlEditTrack` | The track object currently being edited (or NULL if creating new) |
| `controlEditName[]`, `OnScript[]`, `OffScript[]` | Buffers holding current values for the edit dialog |

## Notes

- Controls are **standalone objects** — they do not contain constituent tracks or sub-components. Their purpose is solely to act as a trigger point on the layout for executing scripts (typically detector routines).
- The "On/Off Scripts" fields allow attaching custom behavior (e.g., setting a signal aspect, logging sensor readings) that runs when a track crosses the control's location.
- Because controls are drawn at a fixed visual size using `control_SF`, they do not scale with the layout — this is intentional so that their on-screen appearance remains consistent regardless of zoom level.
