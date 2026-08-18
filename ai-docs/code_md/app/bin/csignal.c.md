# csignal.c — Signal Objects (Layout Control: Track Signals)

## Overview

`csignal.c` implements **Signal** objects in XTrkCad. A *signal* is a layout-control object placed at block boundaries to regulate train movement and enforce interlocking rules between conflicting routes. Each signal can have multiple "heads" (lamps) each representing a different aspect or route permission, along with an associated control script that determines which aspects are displayed under what conditions.

---

## Key Data Structures

### `signalAspect_t` — Signal Aspect Definition

```c
typedef struct signalAspect_s {
    char *aspectName;   // Display name for the aspect (e.g., "STOP", "CAUTION", "PROCEED")
    char *aspectScript; // Control script: condition expression → action
} signalAspect_t, *signalAspect_p;
```

Each aspect pairs a human-readable label with a control script that encodes the logic for when that aspect should be displayed. For example, a script might read `"trackA_occupied OR blockB_blocked"` to mean "show STOP when either condition is true."

---

### `signalData_t` — Signal Extra Data

```c
typedef struct signalData_s {
    extraDataBase_t base;
    coOrd orig;                     // Position of the signal mast base
    ANGLE_T angle;                  // Orientation around the origin
    char *name;                    // Human-readable name ("Signal A", "Exit S2")
    wIndex_t numHeads;             // Number of lamps (1-3) on this signal
    BOOL_T IsHilite;               // Highlight flag (drag/selection state)
    wIndex_t numAspects;           // Number of defined aspects
    signalAspect_t aspectList;     // Dynamic array: named/listed aspects
} signalData_t, *signalData_p;
```

- **`orig`** — The global origin point where the signal mast is planted.
- **`angle`** — Rotation around that origin (allows placing signals at arbitrary orientations).
- **`numHeads`** — Number of lamps shown on the signal face (typically 1–3 for standard railway signaling).
- **`aspectList`** — A dynamic array storing all defined aspects for this signal.

---

### `extraDataCompound_t` — Shared Compound Header

Signals also inherit from the compound track type, providing:
```c
coOrd orig;                     // Bounding box / transform origin
ANGLE_T angle;                  // Rotation around origin
BOOL_T flipped;                 // Horizontally mirrored?
BOOL_T ungrouped;              // Segments individually selectable?
int split;                      // Split segment index (multi-part objects)
char *descriptionOrig;         // Original description offset
coOrd descriptionOff;          // Offset from orig to label position
```

---

## Core Functions

### `GetSignalData(track_p trk)` — Retrieve Extra-Data Pointer

Returns a pointer to the signal's extra-data block via the generic `GET_EXTRA_DATA()` macro. Returns `NULL` if the track type is not `T_SIGNAL`. Used extensively throughout this file as the primary accessor for all signal fields.

---

### `DDrawSignal(drawCmd_p d, coOrd orig, ANGLE_T angle,
                wIndex_t numHeads, DIST_T scaleRatio, wDrawColor color)` — Draw Signal Graphic Primitive

This internal function draws a single signal at arbitrary position and orientation:
1. Computes rotated coordinates for the mast base (a vertical line from ground to top of pole).
2. Draws the mast as a thick vertical line (width=2 pixels in device space).
3. Iterates over each head (`numHeads`) and draws a filled circle at the appropriate height along the mast.

The signal face is drawn with `color` — typically black for normal operation, red if associated track segments are deleted.

---

### `DrawSignal(track_p t, drawCmd_p d, wDrawColor color)` — Public Drawing Entry Point

Retrieves the extra-data block and forwards to `DDrawSignal()`. The scale ratio is obtained from the current layout scale so that signals render at a consistent physical size regardless of overall layout zoom level.

---

### `SignalBoundingBox(coOrd orig, ANGLE_T angle, wIndex_t numHeads,
                       DIST_T scaleRatio, coOrd *hi, coOrd *lo)` — Compute Bounding Box

Given a signal's transform parameters and number of heads, computes the axis-aligned bounding box:
1. Rotates coordinate deltas by `90° - (360° - angle)` to convert from local space to global space.
2. Transforms the base mast origin and each head position.
3. Tracks min/max X and Y across all points to produce `(lo, hi)`.

This is used for hit testing, drag handles, and layering order computation.

---

### `ComputeSignalBoundingBox(track_p t)` — Compute Bounding Box on Track Object

Calls `SignalBoundingBox()` with the track's current origin, angle, head count, and scale ratio, then stores the result in the track's bounding-box field via `SetBoundingBox()`. Called after any transform operation (move/rotate) or creation.

---

### `DistanceSignal(track_p t, coOrd *p)` — Distance to Signal Mast Base

Returns the Euclidean distance from point `*p` to the signal's origin (`orig`). Used for snapping and hit-testing. Updates `*p` to be the closest point on the signal (the mast base). Returns infinity if the track is not a signal type.

---

### `DescribeSignal(track_p trk, char *str, CSIZE_T len)` — Build Description String

Populates the global `message[]` buffer with a description such as:
```text
signal (42 [Exit S2]): Layer=3, 3 heads at 120.5,-87.2 A120.5
```

The format includes: type name, track index, layer number, number of heads, origin coordinates, and orientation angle. Also fills internal static buffers (`signalProperties`) for use by the description dialog controls.

---

### `UpdateSignalProperties(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)` — Update from Dialog Edit

Handles changes to a signal's properties when a user modifies fields in its edit dialog:
- **`NM`** (name field) — compares new name against old; if different, allocates memory and updates `xx->name`.
- **`PS`** (position X/Y) — reads from global static buffers (`signalProperties.pos`) into the track's origin.
- **`OR`** (orientation angle) — reads from `signalProperties.orient` into `xx->angle`, then recomputes bounding box and redrawing flags.
- **`HD`** (number of heads) — reads integer value; no redraw needed since head count is purely cosmetic for rendering.
- Field **-1** acts as a sentinel to compare all fields at once before committing any changes.

If `needUndoStart` is TRUE, calls `UndoStart()` with label "Change Signal" and `UndoModify(trk)` before making modifications.

---

### `DeleteSignal(track_p trk)` — Free a Signal Object

Frees:
- The name string (`xx->name`).
- Every aspect's name and script strings in the dynamic array (`aspectList`), iterating from 0 to `numAspects - 1`.

Then calls `FreeTrack(trk)` to free the entire track record.

---

### `WriteSignal(track_p t, FILE *f)` — Serialize Signal to File Format

Writes a signal and all its aspects as:
```text
SIGNAL <index> <layer> "<scale>" <visible? 0|1> \
    <origX> <origY> <angle> <numHeads> "<name>"
    ASPECT "STOP" "trackA_occupied OR blockB_blocked\n"
    ASPECT "CAUTION" "blockC_blocked\n"
    ASPECT "PROCEED" ""
END_SIGNAL
```

The format includes:
- Index, layer, scale name (e.g., `"100MM"`), visibility flag.
- Origin X/Y and angle in degrees.
- Number of heads.
- Name string.
- Zero or more `ASPECT` lines, each containing a quoted name and a quoted script string.
- Terminated by an `END_SIGNAL` marker line.

---

### `ReadSignal(char *line)` — Deserialize Signal from File Format

Parses a file format signal definition:
1. Uses `GetArgs()` to extract the header fields (index, layer, scale, visibility, origin, angle, head count, name).
2. Converts UTF-8 strings if necessary via `ConvertUTF8ToSystem()`.
3. Scans subsequent lines for `ASPECT` entries, parsing each into a dynamic array (`signalAspect_da`). Stops at an `END_SIGNAL` marker line.
4. Calls `NewTrack()` to allocate a new track record of type `T_SIGNAL`, sized appropriately for the number of aspects.
5. Copies the parsed fields into the extra-data block and calls `ComputeSignalBoundingBox()`.

Returns TRUE on success; FALSE if any field fails to parse.

---

### `MoveSignal(track_p trk, coOrd orig)` — Translate Signal by Offset

Adds offsets to the signal's origin and recomputes the bounding box via `ComputeSignalBoundingBox()`. The angle is unchanged (no rotation applied).

---

### `RotateSignal(track_p trk, coOrd orig, ANGLE_T angle)` — Rotate Around a Point

Rotates the signal's origin around point `orig` by the given angle using `Rotate()`, then normalizes the resulting orientation to `[0°, 360°)`. Recomputes bounding box. Note: this rotates the *entire* signal including its aspect labels, so the visual result may not always be what a human expects if the angle is arbitrary — typically used only for snapping or constraint-based operations.

---

### `RescaleSignal(track_p trk, FLOAT_T ratio)` — Scale Signal (No-op)

Currently empty; reserved for future use when signals need to scale with layout zoom levels independently of track segments.

---

### `FlipSignal(track_p trk, coOrd orig, ANGLE_T angle)` — Mirror Signal Horizontally

Mirrors the signal across a vertical axis through `orig`:
1. Calls `FlipPoint()` to compute the mirrored origin.
2. Adjusts the angle by reflecting it: `angle = 2*reflected_angle - current_angle`. This ensures the signal rotates in the opposite direction when flipped, so the "front" of the signal faces correctly.
3. Recomputes bounding box.

---

### `WriteSignal` / `ReadSignal` — File I/O

See above for format details. The file format is line-based and human-readable: each aspect appears on its own line with a quoted name and script string. Scripts are multi-line if they contain newlines (the newline character inside the quotes is preserved literally).

---

## Editing Dialog (`SignalEditOk`, `SignalEditCancel`)

The signal edit dialog presents three fields:
- **Name** — human-readable identifier for the signal in lists/descriptions.
- **Origin X/Y** — absolute screen/coordinate space position where to place the mast base.
- **Angle** — rotation around that origin (0° = upright).
- **Number of heads** — dropdown with values 1–3 (currently limited by range constraints).
- **Aspect list** — a list widget showing all defined aspects for this signal, each row editable via an "Edit Aspect" button.

When the user clicks OK:
1. Calls `UndoStart()` if creating or modifying.
2. Allocates/reallocates memory in the extra-data block to accommodate any new aspect count (`xx->numAspects`).
3. Copies the name, origin, angle, and head count into the track record.
4. For each aspect in the dynamic array, copies the `aspectName` and `aspectScript` strings (handling reallocation if necessary).
5. Calls `UndoEnd()`, redraws, recomputes bounding box, and hides the dialog.

When Cancel is clicked:
- Frees all currently stored aspect names and scripts.
- Resets the dynamic array count to zero.
- Hides the dialog.

---

## Aspect Edit Dialog (`EditAspectDialog`, `aspectEditOK`)

A secondary dialog for editing individual aspects of a signal. It presents:
- **Name** — short label shown in lists/descriptions.
- **Script** — full control logic expression (can span multiple lines).
- **Index** — read-only field showing the aspect's position in the list (for multi-aspect signals).

When OK is clicked (`aspectEditOK`):
- If no index was given (new aspect), appends a new entry to the dynamic array and adds it to the list widget.
- Otherwise, replaces the existing aspect at that index with the new values.
- Hides the dialog.

The main signal edit dialog lists all aspects in a `wList` widget; selected rows can be edited, added (via "Add Aspect" button), or deleted (via "Delete Aspect" button). The list is updated via `SignalEditDlgUpdate()` whenever selections change.

---

## Management Commands (`CmdSignal`, `SignalMgmProc`)

### `CmdSignal(wAction_t action, coOrd pos)` — Interactive Placement Command

Handles placing a new signal:
- **C_START** — prompts "Place base of signal" and sets a creation flag; hides selection handles.
- **C_DOWN / C_MOVE** — captures mouse position; shows an interactive preview (a single-headed signal) that follows the cursor as it is dragged.
- **C_UP** — finalizes placement: calls `CreateNewSignal()` which opens the edit dialog with pre-filled origin and angle derived from the drag vector.
- **C_CANCEL** — aborts creation.

---

### `SignalMgmProc(int cmd, void *data)` — Management Dispatcher

Registered with the container manager system. Handles:
- **CAN_EDIT / DO_EDIT** → calls `EditSignal()` to open the properties dialog.
- **CAN_DELETE / DO_DELETE** → deletes the signal track.
- **DO_HILIGHT** / **UN_HILIGHT** → draws a semi-transparent gray rectangle around the signal during drag/selection mode (`IsHilite` flag).
- **GET_TITLE** → formats a label for list widgets (e.g., `"Signal A"`).

---

### `InitCmdSignal(wMenu_p menu)` — Menu Registration

Adds a "Signal" button to the command palette with an icon. The button invokes `CmdSignal()` which begins interactive placement mode.

---

### `SignalMgmLoad(void)` — Load Management Contexts

Iterates over all signal tracks in memory and registers each with the container manager system, providing per-object management handlers (edit, delete, highlight).

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `GetSignalData()` | Retrieve extra-data pointer for a track |
| `DDrawSignal()` | Internal drawing primitive for mast + heads |
| `DrawSignal()` | Public entry point for rendering a signal |
| `SignalBoundingBox()` | Compute bounding box from parameters |
| `ComputeSignalBoundingBox()` | Compute and store bounding box on track |
| `DistanceSignal()` | Distance from cursor to signal base (for snapping) |
| `DescribeSignal()` | Build human-readable description string |
| `UpdateSignalProperties()` | Apply dialog field changes with undo support |
| `DeleteSignal()` | Free memory and deallocate aspect strings |
| `WriteSignal()` | Serialize signal + aspects to file format |
| `ReadSignal()` | Deserialize from file (parse header + ASPECT lines) |
| `MoveSignal()` | Translate origin by offset vector |
| `RotateSignal()` | Rotate around a point |
| `RescaleSignal()` | No-op; reserved for future |
| `FlipSignal()` | Mirror across vertical axis through origin |
| `CmdSignal()` | Interactive placement: drag-to-place workflow |
| `SignalMgmProc()` | Container manager event dispatcher |
| `SignalEditOk()` / `SignalEditCancel()` | Dialog "OK" and "Cancel" handlers |
| `EditAspectDialog()` / `aspectEditOK()` | Aspect editor dialog logic |
| `MoveAspectUp()` | Reorder aspects in the dynamic array (up swap) |
| `AspectDelete()` | Delete selected aspect(s) from list + internal array |
| `SignalMgmLoad()` | Register all signals with container manager |
| `InitCmdSignal()` | Add menu button and icon |

---

## Usage Flow

1. **Initialization** — `InitTrkSignal()` registers the track type; `SignalMgmLoad()` is called later to wire up management commands for each signal.
2. **Placement** — User clicks "Signal" in the command palette → `CmdSignal(C_START)` → drags cursor → `C_UP` fires → opens edit dialog with pre-filled position/angle → user sets name, head count, and defines aspects via the list widget.
3. **Editing** — User can reorder or delete aspects; each aspect has its own script that controls interlocking logic (e.g., `"trackA_occupied OR blockB_blocked"`).
4. **Rendering** — `DrawSignal()` renders the mast and lamps using the stored origin, angle, and head count. The lamp colors are determined by evaluating each aspect's script against the current layout state.
5. **File I/O** — Signals are written with all aspects inline; on load, `ReadSignal()` reconstructs the dynamic array of aspects from the file lines.

---

## Notes

- Signal scripts use a custom expression language (defined elsewhere) that evaluates conditions like `"trackX_occupied"` or `"blockY_blocked"` to determine which aspect to display.
- The number of heads (1–3) is currently constrained by a range parameter in the edit dialog, but could be expanded later for multi-aspect signals with more lamps.
- Signals are drawn on a separate "screen" draw context (`&screenDrawFuncs`) so they can overlay track segments without interfering with normal track rendering order.
