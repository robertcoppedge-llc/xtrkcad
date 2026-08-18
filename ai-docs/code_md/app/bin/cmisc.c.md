# cmisc.c — Track "Describe" Dialog (Properties Editor)

## Overview

`cmisc.c` implements the **Describe** command (`CmdDescribe`) — a dialog-based properties editor for track elements. When activated (by right-clicking or pressing Shift+F4), it presents a dynamically generated dialog showing and allowing modification of all editable parameters of the selected track element.

The dialog is built using the XTrkCAD parameter system, with pre-defined control slots that are bound to specific `descData` fields depending on the element type (straight, curve, turnout, etc.).

---

## Key Data Structures

### The `describePLs[]` Parameter Slot Table

This is a **polymorphic** table of parameter slot definitions. Each slot has a fixed type (`PD_FLOAT`, `PD_LONG`, `PD_STRING`, `PD_COLORLIST`, `PD_DROPLIST`, `PD_TEXT`, `PD_RADIO`, `PD_TOGGLE`) and an associated option flag (e.g., `PDO_DIM`, `PDO_ANGLE`).

The table is organized into groups:

| Group | Range of Indices | Purpose |
|-------|-----------------|---------|
| Float slots (`F1`..`F40`) | 0–39 | Arbitrary float values (position, radius, angle, length, etc.) |
| Integer slots (`I1`..`I5`) | 40–44 | Arbitrary integers (e.g., turnout number) |
| String slots (`S1`..`S4`) | 45–48 | Free-text strings (text notes, names) |
| Layer slot (`Y1`) | 49 | Layer selection dropdown (excludes frozen layers) |
| Color slot (`C1`) | 50 | Color picker |
| List slots (`L1`..`L4`) | 51–54 | Generic dropdowns (e.g., turnout type, motor state) |
| Editable list slot (`LE1`) | 55 | Editable text field |
| Text slot (`T1`) | 56–57 | Multi-line text area |
| Pivot radio (`P1`) | 58 | Radio buttons for "First/Middle/End" pivot points |
| Toggles (`boxed1`..`boxed4`) | 59–62 | Checkbox controls (e.g., "Boxed") |

The table is indexed by `type` via `descTypeMap[]`, which maps each param type (`DESC_POS`, `DESC_FLOAT`, etc.) to the corresponding range of slots.

### The `descData[]` Array

An array of `descData_t` structures, one per track element parameter:

```c
struct descData_s {
    int      type;             // DESC_* type (POS, FLOAT, STRING, PIVOT, etc.)
    char    *label;           // Display label for the control
    void   *valueP;          // Pointer to the actual value stored in track element
    long    mode;             // Mode flags: DESC_RO, DESC_IGNORE, DESC_CHANGE, ...
    int     posx, posy;      // Y-position of the dialog control (for layout)
    wControl_p control0, control1;  // The widget(s) bound to this param
};
```

The `descData` array is type-agnostic — it simply stores pointers to fields in various track element structures (`trackStraight`, `trackCurve`, `turnout_t`, etc.). At dialog creation time, the code walks through `descData[]` and binds each entry to an appropriate slot from `describePLs[]`.

### The `descTypeMap[]` Mapping Table

Maps a param type (e.g., `DESC_FLOAT`) to a range of indices in `describePLs[]`:

```c
static struct {
    parameterType pd_type;   // e.g. PD_FLOAT, PD_LONG, etc.
    long          option;     // e.g. PDO_DIM for dimension values
    int           first;      // First slot index for this type
    int           last;       // Last slot index (exclusive)
} descTypeMap[] = {
    {0, 0, 0, 0},                   /* NULL — unused */
    {PD_FLOAT, PDO_DIM,   I_FLOAT_0, I_FLOAT_N },   /* DESC_POS */
    {PD_FLOAT, 0,         I_FLOAT_0, I_FLOAT_N },   /* DESC_FLOAT */
    {PD_FLOAT, PDO_ANGLE, I_FLOAT_0, I_FLOAT_N },   /* DESC_ANGLE */
    {PD_LONG,  0,         I_LONG_0, I_LONG_N },     /* DESC_LONG */
    ...
};
```

---

## The `CmdDescribe` Command Handler

This is a state machine that runs while the Describe dialog is active. It responds to mouse events and determines what action to take.

### States / Actions:

| Action | Purpose |
|--------|---------|
| `C_START` | User enters describe mode — show cursor as question mark, wait for click |
| `wActionMove` | Cursor moved over canvas — search for a track element under the pointer |
| `C_DOWN` | Mouse button pressed on an element — open the dialog (or close if already open on another) |
| `C_REDRAW` | Redraw loop while dialog is visible — highlight the selected element with a blue rectangle, optionally draw origin anchor |
| `C_CANCEL` | User cancels (ESC or Alt+F4) — close dialog and restore cursor |
| `C_CMDMENU` | Context menu requested |
| `C_FINISH` | Command finished (cleanup) |

### Key Logic in `C_DOWN`:

1. If a different track is already being described, finish updating it first (`DescribeDone()`).
2. Reject frozen layers unless Shift key is held down.
3. Highlight the element with a blue rectangle (via `descNeedDrawHilite`).
4. Build and show the dialog — call `DoDescribe()`.
5. If the element is a note track, don't open the full describe dialog (notes have their own handler).

---

## The `DescribeUpdate()` Callback

Every time a control in the dialog changes value, `DescribeUpdate()` is called via the parameter system's change notification mechanism. It:

- Skips read-only (`DESC_RO`) and ignored parameters.
- If an undo group isn't already active, starts one with `UndoStart()`.
- Calls `descUpdateFunc()` (a function pointer stored when opening the dialog) to write the new value back into the track element structure.
- Re-applies mode flags: read-only controls are deactivated; others are activated again.
- For `DESC_POS` type parameters, toggles a `DESC_CHANGE2` flag between calls — this ensures that both X and Y of a position field get written to in separate updates (since the two float slots share one label).

---

## The `DescribeLayout()` Layout Callback

The parameter system needs a layout function to place each control at the correct (x, y) position. `DescribeLayout()` is called for every parameter data entry as controls are laid out:

- If the param has no context (`pd->context == NULL`), skip it (already handled elsewhere).
- For `DESC_POS` parameters with two separate float slots (`posX`, `posY`), increment x by the width of the first control plus 3 pixels.
- For text fields, set a fixed size via `wTextSetSize()`.

---

## The `DoDescribe()` Function — Dialog Creation

This function builds the entire dialog:

1. Initializes static variables (`descTrk`, `describeW_posy`, etc.).
2. Calls `CreateEditableLayersList()` to build a list of non-frozen layers for the layer selector dropdown.
3. If this is the first call, creates (or recreates) the dialog window via `ParamCreateDialog()`.
4. Hides all controls from previous use and resets their option flags to `PDO_DLGIGNORE` — essentially clearing the table.
5. Walks through the `descData[]` array:
   - For each parameter, assigns it a Y position (`describeW_posy`).
   - Calls `AssignParamToDescribeDialog()` which finds an unused slot in `describePLs[]` matching the param's type and initializes the control (setting labels, connecting to value pointers, etc.).
   - Special handling for layer selectors: rebuilds the list of values each time.
6. Lays out all controls via `ParamLayoutDialog()`.
7. Loads all control initial values with `ParamLoadControls()`.
8. Sets the window title (`"<element description> (Tn)"`).
9. Shows the dialog.

---

## The `DescChange()` Notification Handler

Registered with `RegisterChangeNotification(DescChange)`, this is called whenever a system-wide change occurs (e.g., gauge changes, units changed). If the display units have changed and the dialog is visible, it reloads all controls so that displayed values are in the correct units.

---

## Summary of Key Concepts

| Concept | Description |
|---------|-------------|
| **Polymorphic control slots** | `describePLs[]` defines a pool of pre-built parameter slots; each is reused across multiple track elements by clearing its flags between uses. |
| **Type mapping** | `descTypeMap[]` maps logical param types (FLOAT, STRING, COLOR, etc.) to ranges of slot indices. |
| **Dynamic dialog** | The same dialog window is repurposed for every element type; only the relevant controls are shown/hidden and wired up. |
| **Undo integration** | `descUpdateFunc()` writes changes back into the track structure inside an undo group started by `DescribeUpdate()`. |
| **Freezing check** | Elements on frozen layers require Shift+click to open Describe (preventing accidental edits of locked content). |

---

## File Structure Summary

```c
// Global variables for describe command state
EXPORT wIndex_t        describeCmdInx;  // Menu button index
EXPORT BOOL_T          inDescribeCmd;    // Are we currently in the describe flow?
static track_p        descTrk;           // Currently selected track element
static descData_p     descData;          // Array of parameter descriptors
static descUpdate_t   descUpdateFunc;    // Callback to write changes back
static coOrd          descOrig, descSize;// Bounding box for highlight rectangle
static wDrawColor     descColor;         // Color used for hilite
EXPORT BOOL_T         descUndoStarted;  // Have we started an undo group?
static wMenu_p        descPopupM;       // Context menu for describe

// The polymorphic slot table — defines all possible control types and their options
static paramData_t    describePLs[] = { ... };

// Mapping from logical type to a range of slots in describePLs[]
static struct { parameterType pd_type, option, first, last; } descTypeMap[] = {...};

// Main command handler — state machine for mouse events
EXPORT STATUS_T CmdDescribe(wAction_t action, coOrd pos)
    case C_START:   // show question cursor
    case wActionMove: // find element under cursor
    case C_DOWN:    // open dialog on click
    case C_REDRAW:  // draw hilite rectangle if any element selected
    case C_CANCEL:  // close dialog, restore cursor
    ...

// Called by param system whenever a control changes value
static void DescribeUpdate(paramGroup_p pg, int inx, void *data)
    // writes new value back into track structure via descUpdateFunc()
    // re-enables controls that are no longer read-only

// Layout callback for placing controls in the dialog window
static void DescribeLayout(paramData_t *pd, int inx, ...)

// Builds and shows the dialog window
void DoDescribe(char *title, track_p trk, descData_p data, descUpdate_t update)

// Notification handler called on system changes (unit change, etc.)
static void DescChange(long changes)
```
