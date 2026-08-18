# cmisc.c — Describe Dialog & Parameter Group Utilities

## Overview

`cmisc.c` implements the **Describe** dialog system — a generic property editor that allows users to inspect and modify any track object's properties at runtime. When a user right-clicks on a track (or presses F12), XTrkCAD opens a modal dialog containing input controls bound to fields in the track's `extra_data` structure. Changes are applied incrementally as the user types, with finalization on dialog close via an undo transaction.

The file also provides utility functions for:
- **Parameter group management** — dynamically creating dialogs from typed field definitions
- **Layer filtering** — building a dropdown list that excludes frozen layers
- **Bounding box computation** — computing tight axis-aligned bounding boxes around track geometry
- **Describe mode handling** — a "describe state" where the mouse hovers over a track and displays its properties without committing changes

---

## Data Structures

### `descData_t` — Per-Field Storage Embedded in Track Objects

```c
typedef struct descData_t {
    extraDataBase_t base;       // Links back to the owning track_p
    enum { NM, PS, ON, OF } mode;   // Name, position (x/y), onscript, offscript
    char * name;                // Dynamic string buffer for names
    coOrd pos;                  // Position coordinates
    char * onscript;            // Script when track is activated
    char * offscript;           // Script when deactivated
    int layer;                  // Layer index (0..NUM_LAYERS-1)
    wColor color;               // Display color
    enum { T_NONE, T_CURVED } type;  // Track subtype
    char scale[STR_SHORT_SIZE]; // Scale name ("HO", "N", etc.)
    char * text;                // For notes: the note's body text
} descData_t, *descData_p;
```

This structure is embedded at the end of every `track` object (after the base header). Its layout matches exactly what the Describe dialog expects — when a control changes, its `context` pointer points into this struct. The `mode` field indicates which subfield within the track's extra data is being edited (e.g., position X vs. position Y are separate controls because they each have their own float field).

### Global Parameter Definition Arrays

The file defines several flat arrays of `paramData_t` entries that describe all possible controls:

#### Float Fields (`describePLs[]`)

```c
static paramFloatRange_t rdata = { 0, 0, 100, PDO_NORANGECHECK_HIGH|PDO_NORANGECHECK_LOW };
static paramData_t describePLs[] = {
    // F1–F40: generic float fields for positions, radii, angles, etc.
};
```

Forty entries provide generic float slots usable by any track type that declares a `DESC_FLOAT` field with an index 0..39 in its extra data structure.

#### Integer Fields (`idata`)

Five integer fields (I1–I5) for indices or counts (e.g., number of switches, turnout count).

#### String Fields

Four string fields (S1–S4) — used for:
- Scale names ("HO", "N")
- Track type descriptions
- Custom track identifiers

#### Layer Dropdown (`editableLayerList[]`)

A static array mapping layer index → dropdown list index. Frozen layers are skipped, so the dropdown never offers a frozen layer as an option. The `SearchEditableLayerList()` function maps between user-facing indices and internal layer indices when the dialog is refreshed.

#### Color Picker (`BC_HORZ|BC_NOBORDER`)

A horizontal color palette control for setting track display colors.

#### Additional Dropdowns (L1–L4)

Four generic dropdown list controls whose purpose varies by track type. For example, a turnout might use one to select which switch throw direction is active.

#### Editable List (`LE1` with `BL_EDITABLE`)

A combo box that can be edited inline — e.g., selecting a track type from a drop-down but allowing the user to type a custom name if no predefined entry matches.

#### Multi-line Text Area (`T1`, width=300px, height=150px)

Used for notes or script fields where multi-line text is acceptable. The `BT_HSCROLL` flag enables horizontal scrolling (wide lines).

#### Radio Buttons (`P1`)

A single radio button labeled "Lock" used to lock/unlock a specific parameter within the Describe dialog — useful when a track has multiple endpoint parameters and only one should be editable at a time.

#### Toggle Checkboxes (`boxed1–4`)

Four checkbox controls used for boolean flags (e.g., "Boxed" — whether the track is visually boxed in the drawing). These are generic slots that individual track types can fill with their own meanings.

### `describePG` — The Reused Dialog Window

```c
static paramGroup_t describePG = { "describe", 0, describePLs, COUNT(describePLs) };
```

This is a **single** parameter group window created once and reused across all Describe dialog invocations. Controls are hidden/shown rather than destroyed/recreated to avoid flicker and save allocation time. This is why the `DescribeLayout()` callback is used — it lays out controls in their designated positions by toggling visibility flags (`PDO_DLGIGNORE`) on each entry of `describePLs`.

### `descTypeMap[]` — Type-to-Entry Range Mapping Table

```c
static struct {
    parameterType pd_type;   // Widget class (float, integer, string, dropdown, etc.)
    long option;             // Option flags (e.g., PDO_DIM for dimensioned input)
    int first;               // First index in describePLs[] that has this type
    int last;                // Last such entry (+1 for exclusive upper bound)
} descTypeMap[] = {
    [DESC_NULL]     = { 0, 0 },
    [DESC_POS]      = { PD_FLOAT, PDO_DIM, I_FLOAT_0, I_FLOAT_N },   // X/Y position → float widgets
    [DESC_FLOAT]    = { PD_FLOAT, 0,       I_FLOAT_0, I_FLOAT_N },
    [DESC_ANGLE]    = { PD_FLOAT, PDO_ANGLE, I_FLOAT_0, I_FLOAT_N },
    [DESC_LONG]     = { PD_LONG,   0,      I_LONG_0, I_LONG_N },
    [DESC_COLOR]    = { PD_LONG,   0,      I_COLOR_0, I_COLOR_N },
    [DESC_DIM]      = { PD_FLOAT, PDO_DIM, I_FLOAT_0, I_FLOAT_N },
    [DESC_PIVOT]    = { PD_RADIO, 0,       I_PIVOT_0, I_PIVOT_N },
    [DESC_LAYER]    = { PD_DROPLIST,PDO_LISTINDEX,I_LAYER_0, I_LAYER_N },
    [DESC_STRING]   = { PD_STRING,0,       I_STRING_0, I_STRING_N },
    [DESC_TEXT]     = { PD_TEXT,  PDO_DLGNOLABELALIGN, I_TEXT_0, I_TEXT_N },
    [DESC_LIST]     = { PD_DROPLIST,PDO_LISTINDEX,I_LIST_0, I_LIST_N },
    [DESC_EDITABLELIST]={ PD_DROPLIST,0,I_EDITLIST_0, I_EDITLIST_N },
    [DESC_BOXED]    = { PD_TOGGLE, 0,       I_TOGGLE_0, I_TOGGLE_N },
};
```

This table is the heart of the dynamic dialog system. Instead of hard-coding `switch(ddp->type)` everywhere, each field type has a corresponding range of entries in the global `describePLs[]` array that share common option flags and layout positions. When a new field with type `DESC_POS` is encountered (e.g., position X of a turnout), the system finds an unused entry from the `PD_FLOAT` pool (`I_FLOAT_0..I_FLOAT_N-1`), reuses it, and assigns it to the current field. This avoids duplication of widget definitions in every track's type file.

### `editableLayerList[]` — Filtered Layer Indices

```c
static unsigned int editableLayerList[NUM_LAYERS];
```

Built by iterating over all layers and skipping any with `GetLayerFrozen(layer) == TRUE`. The array maps internal layer index → dropdown list index (e.g., if layers 2 and 3 are frozen, their indices are simply omitted). This allows the Describe dialog to always present a valid set of choices without needing to recompute on every refresh.

---

## Core Functions

### `CreateEditableLayersList()` — Build Filtered Layer Index List

**Purpose:** Populate `editableLayerList[]` with all non-frozen layer indices. Called once when the dialog is first created and again whenever a layer's frozen state changes (via `DescChange`).

```c
void CreateEditableLayersList() {
    int i = 0;
    int j = 0;
    while (i < NUM_LAYERS) {
        if (!GetLayerFrozen(i)) {
            editableLayerList[j++] = i;
        }
        i++;
    }
}
```

**Algorithm:** Simple linear scan O(n) over all layers. Each non-frozen layer is appended to the output array at position `j`. The resulting indices are what appear in dropdown lists — the user sees "Layer 1", "Layer 4", etc., but internally stores the raw index (0, 3, ...).

**Search function:** `SearchEditableLayerList(unsigned int layer)` looks up a given internal layer number and returns its position in the filtered array. If no match is found, it returns −1 (shouldn't happen if the caller ensures the layer isn't frozen at this moment).

---

### `DescribeUpdate(paramGroup_p pg, int inx, void *data)` — Per-Control Change Callback

This is the callback registered with `ParamCreateDialog()` via its fourth argument. It fires **every time** any control in the dialog changes value (user types a number, picks a color from a palette, selects an item from a dropdown).

```c
static void DescribeUpdate(paramGroup_p pg, int inx, void *data) {
    descData_p ddp;

    if (inx < 0) return;                          // Negative index means "all controls changed"

    ddp = (descData_p)pg->paramPtr[inx].context;  // Back-pointer to the actual field inside track extra_data

    if ((ddp->mode & (DESC_RO | DESC_IGNORE)) != 0) {
        return;  // Read-only or ignored — don't apply changes
    }

    if (!descUndoStarted) {
        UndoStart(descTitle, "Change Track");      // Start an undo transaction grouping all edits in this session
        descUndoStarted = TRUE;
    }

    if (!descTrk) return;                          // Dialog was closed (timer fired after OK) — bail out

    UndoModify(descTrk);                            // Mark track as modified so it will be redrawn later
    descUpdateFunc(ddp->context, ...);              // Call the track-specific update function (e.g., recompute geometry for a curve)

    if (OFF_D(mapD.orig, mapD.size, descOrig, descSize)) {
        ErrorMessage(MSG_MOVE_OUT_OF_BOUNDS);       // User dragged endpoint outside viewport bounds
    }

    for (inx = 0; inx < COUNT(describePLs); inx++) {
        ddp = (descData_p)describePLs[inx].context;
        if ((ddp->mode & DESC_IGNORE) != 0) continue;
        if ((ddp->mode & DESC_CHANGE) == 0) {      // Only reload controls that have changed
            if ((ddp->mode & DESC_CHANGE2) == 0) continue;
        }
        if (ddp->type == DESC_POS && ddp->control0 != pg->paramPtr[inx].control) {
            wControlActive(ddp->control1, FALSE);   // Hide the second position control (we're only changing X right now)
        } else {
            wControlActive(ddp->control0, TRUE);    // Show this control's widget
            ParamLoadControl(&describePG, inx);     // Load its current value from data structure back into widget
        }
        if (ddp->type == DESC_POS) {
            if ((ddp->mode & DESC_CHANGE2)) {       // This is the second coordinate change (Y after X was already changed)
                ddp->mode &= ~DESC_CHANGE2;         // Clear flag — next change will be treated as "final" for this field pair
            } else {
                ddp->mode |= DESC_CHANGE2;           // Set flag so that when the user changes the other coordinate, we trigger geometry recomputation
            }
        }

        if (ddp->type == DESC_LAYER) {              // Layer dropdown needs special handling:
            wListClear((wList_p)ddp->control0);     // Rebuild list — exclude frozen layers
            for (i = 0; i < NUM_LAYERS; i++) {
                if (!GetLayerFrozen(i)) {
                    char *fmtName = FormatLayerName(i);
                    wListAddValue((wList_p)ddp->control0, fmtName, NULL, I2VP(editableLayerList[j]));
                    free(fmtName);
                }
            }
        }

        // Set read-only if track's layer is frozen in the UI
        if (GetLayerFrozen(GetTrkLayer(descTrk))) {
            wControlActive(ddp->control0, FALSE);
        } else {
            wControlActive(ddp->control0, TRUE);
        }
    }
}
```

**Key design decisions:**

- **Incremental updates:** The callback is invoked for each individual control change. This avoids the "fire-and-forget" approach where all changes are batched until dialog close — it gives immediate feedback (e.g., if changing a curve radius, the preview geometry updates instantly).
  
- **DESC_CHANGE / DESC_CHANGE2 flag toggle** for position fields: Position X and Y each have their own float field. When the user moves one control, we set a `DESC_CHANGE2` flag on that field's data struct. On the next change (of the other coordinate), we detect this flag and trigger geometry recomputation. This avoids redundant computation when only one coordinate is being adjusted.

- **Read-only enforcement:** If a track's layer is frozen in the UI, all controls are set to read-only (`wControlActive(..., FALSE)`). This allows editing unfrozen tracks even if they share fields with frozen ones — each field is independently enabled/disabled based on its owning track's layer state.

---

### `DescribeDone(void *junk)` — Dialog Close Handler (OK or Cancel)

Called when the dialog closes (either OK or Cancel button pressed, or window closed). It finalizes changes and cleans up:

1. If an undo transaction was started (`descUndoStarted == TRUE`), call `UndoEnd()` to commit it.
2. Call the user's update function with a flag indicating that **all** controls have been processed (`!descUndoStarted`) — this is where track-specific finalization happens (e.g., recomputing geometry after both position fields have changed).
3. If the layer dropdown was used, apply the selected layer: look up the internal index via `SearchEditableLayerList()` and call `SetTrkLayer()`.
4. Hide and destroy the dialog window if it is still visible.

**Important:** The function checks whether the track has been deleted (via `IsTrackDeleted(descTrk)`). If so, no undo transaction was started and no changes need to be finalized — the user simply closed the dialog by clicking on another object or pressing Esc.

---

### `DoDescribe(char *title, track_p trk, descData_p data, descUpdate_t update)` — Show/Edit Properties Dialog

This is the top-level entry point for showing a Describe dialog for a given track. It uses a state machine approach:

```c
void DoDescribe(char * title, track_p trk, descData_p data, descUpdate_t update) {
    int inx;
    descData_p ddp;

    if (!inDescribeCmd) return;  // Not currently in "describe mode" — ignore

    CreateEditableLayersList();   // Always rebuild the filtered layer list (layers may have changed)

    descTrk = trk;                // Store pointer to track being described
    descData = data;              // Pointer into track->extra_data (the first descData_t entry)
    descUpdateFunc = update;      // User-defined callback invoked after each control change

    if (!describePG.win) {        // First call: create the parameter group window once
        ParamCreateDialog(&describePG, _("Description"), NULL, NULL,
                          ParamCancel_Reset, TRUE, DescribeLayout, DescribeUpdate);
    }

    for (inx = 0; inx < COUNT(describePLs); inx++) {
        describePLs[inx].option = PDO_DLGIGNORE;   // Reset all entries — we'll fill them dynamically
        wControlShow(describePLs[inx].control, FALSE);
    }

    ro_mode = (GetLayerFrozen(GetTrkLayer(trk)) ? DESC_RO : 0);

    for (ddp = data; ddp->type != DESC_NULL; ddp++) {
        if (ddp->mode & DESC_IGNORE) continue;
        ddp->posy = describeW_posy;   // Record vertical position for layout computation
        ddp->control0 = AssignParamToDescribeDialog(ddp, ddp->valueP, _(ddp->label), 3);

        if (ddp->type != DESC_LAYER) {
            wControlActive(ddp->control0, !!(ro_mode == 0));
        }
    }

    // Special handling for layer dropdown: populate with non-frozen layers only
    for (ddp = data; ddp->type != DESC_NULL; ddp++) {
        if (ddp->type == DESC_LAYER) {
            wListClear((wList_p)ddp->control0);
            for (i = 0; i < NUM_LAYERS; i++) {
                if (!GetLayerFrozen(i)) {
                    char *fmtName = FormatLayerName(editableLayerList[i]);
                    wListAddValue((wList_p)ddp->control0, fmtName, NULL, I2VP(i));
                    free(fmtName);
                }
            }
        }
    }

    ParamLayoutDialog(&describePG);   // Lays out controls in their designated positions (by toggling PDO_DLGIGNORE visibility flags)
    ParamLoadControls(&describePG);   // Loads current values from data structures into widgets
    sprintf(message, "%s (T%d)", title, GetTrkIndex(trk));  // Build window title
    wWinSetTitle(describePG.win, message);
    wShow(describePG.win);            // Display the dialog
}
```

**Workflow summary:**

1. **Create-once window:** The parameter group window is created only on first call. Controls are hidden/shown rather than destroyed/recreated on each invocation — this saves time and avoids flicker, especially important if a user rapidly selects multiple tracks to edit their properties.

2. **Dynamic field layout:** Each track type declares its fields via a struct (e.g., `struct myTrackData { float x; float y; char name[30]; ... };`). The Describe system iterates over these fields using the extra data pointer (`descData = &trk->extraData.descData`) and looks up the appropriate widget class from `describePLs[]` via the `DESC_TYPE_MAP`. It then instantiates a control of that type (float → slider, string → text box, etc.) at a computed vertical position.

3. **Label injection:** Each track's `DescribeTrack()` function (defined in its own header file) fills in human-readable labels for each field (e.g., "Scale Name", "Position X"). The Describe system takes these labels and assigns them as column headers to the corresponding controls.

4. **Layout positioning:** Controls are laid out by toggling their `PDO_DLGIGNORE` flag — when this is zero, the control participates in layout; otherwise it's hidden. This allows dynamic adjustment of which fields appear without moving other fields around.

---

### `AssignParamToDescribeDialog(descData_p ddp, void *valueP, char *label, wWinPix_t sep)` — Instantiate a Widget for a Field

This is the core dispatch function that maps a field's type to its corresponding widget class and creates/initializes it:

```c
static wControl_p AssignParamToDescribeDialog(descData_p ddp, void *valueP, char *label, wWinPix_t sep) {
    int inx;
    for (inx = descTypeMap[ddp->type].first; inx < descTypeMap[ddp->type].last; inx++) {
        // Pick an entry from describePLs[] that matches the requested type
        if ((describePLs[inx].option & PDO_DLGIGNORE) != 0) {
            describePLs[inx].option = descTypeMap[ddp->type].option;   // Reuse this slot for new field
            describePLs[inx].context = ddp;                             // Attach back-pointer to the data struct
            describePLs[inx].valueP = valueP;                          // Pointer into track->extra_data
            if (label) {
                wControlSetLabel(describePLs[inx].control, label);
            }
        }
    }
    return describePLs[inx].control;  // Return the widget handle
}
```

**How it works:** The `describeTypeMap` table gives a range of indices (e.g., `I_FLOAT_0 .. I_FLOAT_N-1`) for each type. When a new float field is needed, we pick the first entry in that range whose `PDO_DLGIGNORE` flag is set — meaning that control was already created during a previous dialog invocation but is currently hidden. We reuse it by copying over its option flags and data pointers. This avoids reallocating memory or creating new widgets on every dialog open.

---

### `DescribeTrack(track_p trk, char *buf, size_t len)` — Generate Human-Readable Title String

Called once when the dialog opens and again on each redraw to keep the window caption in sync. It walks through all fields of the track's extra data and concatenates a summary string like:

```
Signal A [Layer 3]: at -123.456,-789.012  (scale=HO)
```

Each track type defines its own version of this function (via `DescribeTrack()` macro in the header). The Describe system just calls it and places the result in the title bar.

---

### `GetBoundingBox(track_p trk, coOrd *hi, coOrd *lo)` — Compute Tight Bounding Box

Computes the axis-aligned bounding box of a track's geometry (including any endpoint adjustment handles if they exist). Used to size the Describe dialog so it fits tightly around its target object.

**Algorithm:**
1. Start with `lo = trk->pos` and `hi = lo + (trk->endPos - trk->pos)`.
2. For each segment in the track: update `lo.x = min(lo.x, seg.u.l.pos[0].x)` etc.
3. If an endpoint handle exists (stored as a separate structure), also include its bounding box.

The bounding box is then shrunk by `descBorder` (`mainD.scale * 0.1`) so the dialog has a margin around the track.

---

## The "Describe Mode" State Machine

While the user hovers over a track (before clicking to open the properties dialog), XTrkCAD enters **describe mode**. This is indicated by setting global flags (`inDescribeCmd`, `descOrig`, `descSize`, etc.) and drawing a hilite rectangle around the track's bounding box.

The state machine in `CmdDescribe()` handles:

| State | Trigger | Behavior |
|-------|---------|----------|
| **C_START** | Mouse button down while no object is selected | Show message "Click on object for Properties...", change cursor to question mark |
| **wActionMove** | Mouse drags over a track (but not clicked) | Check if pointer is currently over a track; update `descTrk` variable but don't yet show dialog |
| **C_DOWN** | Mouse button released over a track | Show message, check whether layer is frozen (if so, require Shift key), call `DescribeTrack()` to fill title bar with summary string, create/show Describe dialog. For notes (`T_NOTE` type), only show simple description mode (no full property editor). |
| **C_REDRAW** | Redraw event in describe mode | Draw hilite rectangle around track's bounding box; if endpoint handle exists, draw its rings. Also draws preview geometry for flexible tracks (e.g., Bézier segments). |
| **C_CANCEL** | Esc pressed or mouse clicks elsewhere | Hide dialog and reset all global state variables. |
| **C_CMDMENU** | Right-click popup menu request | Show context menu with options: Select Mode, Modify Mode, Pan Mode — useful for exiting describe mode without closing the dialog. |

The `inDescribeCmd` flag guards against re-entrancy (e.g., clicking again while already in describe mode should be ignored). The message shown during this phase is a temporary popup that disappears once the user clicks on an object or cancels.

---

## Utility Functions

### `DrawDescHilite(BOOL_T selected)` — Draw Bounding Box Hilite

Draws a rectangle around the track's bounding box to indicate which object is currently being described. The color is chosen dynamically: gray if no track has been selected, otherwise blue. The hilite is only drawn when `descNeedDrawHilite` is set (which happens on mouse up over a track).

---

### `SearchEditableLayerList(unsigned int layer)` — Map Dropdown Index ↔ Internal Layer Index

Given a raw layer index (e.g., 5), returns its position in the filtered list. For example, if layers 2 and 3 are frozen, then layer 5 would be at index 4 in this array (since indices 0–1 correspond to unfrozen layers 0 and 1). This is needed because the dropdown control stores an integer that must match the internal layer index passed to `SetTrkLayer()`, but the user sees a filtered list.

---

## Design Decisions & Tradeoffs

### Why Reuse the Same Window Instead of Creating a New One Each Time?

The current approach reuses a single parameter group window (`describePG`) and its control widgets across multiple dialog invocations. Controls are hidden/shown rather than destroyed/recreated. This is significantly faster (especially on slower hardware) and avoids flicker. The tradeoff is that the internal layout must be carefully designed so it can accommodate any combination of fields — hence the flat array (`describePLs`) with type dispatch instead of a struct-per-track design.

If performance were not a concern, one could create a fresh window each time (with its own parameter group) and destroy it on close. This would simplify layout computation per-invocation but incur allocation/deallocation overhead.

### Why Not Bind Controls Directly to Fields at Dialog Creation Time?

One might imagine declaring something like:

```c
typedef struct {
    wControl_p control;
    coOrd *pos;
} pos_field_t;

static pos_field_t pos_field[] = { /* ... */ };
```

But this would require duplicating the field layout in two places (the track type definition and the Describe system). Instead, each track type defines its own `DescribeTrack()` function that fills a global array with pointers to the actual buffers. The Describe system then looks up which widget class applies to each buffer's type (float → slider, string → text box, etc.) and instantiates it accordingly.

This achieves **separation of concerns**:
- Track type definitions declare *what* fields exist.
- `DescribeTrack()` declares *how* those fields are labeled and what units they use.
- The Describe system handles *which widget class* to instantiate for each field type.

### Why Use `DESC_CHANGE2` for Position Fields?

Position fields are stored as two separate float controls (X and Y). When the user moves one control, we don't want to immediately recompute the track's geometry — that would be wasteful if the user is still adjusting the other coordinate. Instead, a flag (`DESC_CHANGE2`) is set on the second field. Only when both have changed do we call `descUpdateFunc`. This is implemented by checking whether the control was modified; if so, toggle the flag and let the next change trigger the update.

This avoids redundant computation (e.g., recomputing a curve's endpoint after only X has moved) while still providing incremental feedback when both coordinates are set.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CreateEditableLayersList()` | Build filtered layer index list (excludes frozen layers) | — |
| `SearchEditableLayerList(unsigned int)` | Map between user-facing dropdown index and internal layer index | target layer number |
| `DescribeUpdate(paramGroup_p pg, int inx, void *data)` | Callback invoked on every control change; applies updates incrementally | param group, field index, current value |
| `DoDescribe(char*, track_p, descData_p, descUpdate_t)` | Show property edit dialog for a track (create window once, reuse thereafter) | title string, track pointer, per-field data array, update callback |
| `DescribeDone(void*)` | Called when dialog closes; finalizes changes and cleans up | — |
| `AssignParamToDescribeDialog(descData_p ddp, void *valueP, char *label, wWinPix_t sep)` | Instantiate a widget for a given field type from the global pool | field data struct, value pointer, label string, separator spacing |
| `DescribeTrack(track_p trk, char* buf, size_t len)` | Generate human-readable title string for the dialog window | track pointer, output buffer, buffer length |
| `GetBoundingBox(track_p trk, coOrd *hi, coOrd *lo)` | Compute tight bounding box around a track's geometry | track pointer, out pointers for high/low corners |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Generic property editor for any track object; allows users to edit fields at runtime with undo support and incremental feedback |
| **Domain** | GUI dialog management, parameter binding patterns, reusable UI components in event-driven applications |
| **Key concept** | A single global `parameterData_t` array defines all possible controls; each field type (float, string, color) maps to a widget class via an enum lookup table (`descTypeMap`) that points into the same pool of entries — entries are reused across dialog invocations by toggling visibility flags rather than recreating widgets |
| **Main entry point** | `DoDescribe(title, trk, data, update)` — invoked from right-click on a track or F12 key press; creates the window once and reuses it thereafter |
| **Update mechanism** | `DescribeUpdate()` callback fires for every control change, applying updates incrementally; finalization happens in `DescribeDone()` when the dialog closes |
| **Design highlight** | The "describe mode" state machine allows users to hover over a track to preview its properties before deciding whether to open the full editor — useful for quickly identifying which object is under the cursor |
