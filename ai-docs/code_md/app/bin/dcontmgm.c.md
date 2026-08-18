# dcontmgm.c — Control Element Management Framework

## Overview

`dcontmgm.c` provides a **generic management framework** for "layout control elements" in XTrkCAD. These are logical objects that bridge the CAD layout with external control software (LCC, DCC decoders, etc.). The file implements:

- A unified dialog-based management UI
- A callback-driven architecture allowing different element types (blocks, switchmotors, sensors) to share the same management code
- Undo support for deletions
- Change notification integration with other modules

---

## What Are Control Elements?

Control elements are **logical objects** associated with layout features but not physical track pieces. They include:

| Element Type | Purpose | Example Use Case |
|--------------|---------|------------------|
| Block | Occupancy detection zone | Detect train presence on a segment of track |
| Switchmotor | Actuator command for turnout | Send LCC "throw" command to move a switch |
| Signal | Signal aspect control | Display red/green/yellow aspect on DCC decoder |
| Sensor | Input from external device | Read temperature, pressure, or other layout sensors |

These elements contain **"scripts"** — free-form text that describes how the element interfaces with external software. XTrkCAD does **not** impose any syntax; it simply stores and associates these scripts with their corresponding layout features.

---

## Core Architecture: Callback-Driven Context

The framework uses a **context pointer pattern** to route operations to type-specific implementations:

```c
typedef struct {
    contMgmCallBack_p proc;  // Type-specific callback function
    void *data;              // Pointer to type-specific data structure
    wIcon_p icon;            // Icon for the UI list entry
} contMgmContext_t, *contMgmContext_p;
```

The `contMgmContext_p` is stored as item context in a GTK list widget. Each element type (blocks, switchmotors, etc.) registers its own elements via:

```c
ContMgmLoad(wIcon_p icon, contMgmCallBack_p proc, void *data);
```

This single entry point handles registration for **all** control element types — the callback (`proc`) and data pointer are used to dispatch operations like `CONTMGM_DO_EDIT`, `CONTMGM_GET_TITLE`, etc.

---

## Key Data Structures

### Control Management Context

| Field | Type | Description |
|-------|------|-------------|
| `proc` | `contMgmCallBack_p` | Callback function pointer (e.g., `BlockEditProc`) |
| `data` | `void*` | Pointer to element-specific data structure |
| `icon` | `wIcon_p` | Icon displayed in the management list |

### Control List Data Structure

```c
static paramListData_t controlListData = { 10, 400, 3, controlListWidths, controlListTitles };
```

Where:
- `10` — initial capacity of the list
- `400` — column widths array size (columns: "Name", "Tracks")
- `3` — number of columns
- `controlListWidths[]` = `{ 18, 100, 150 }` — per-column widths

---

## Key Functions

### Control Management Initialization

```c
EXPORT addButtonCallBack_t ControlMgrInit(void)
{
    ParamRegister(&controlPG);
    RegisterChangeNotification(ContMgmChange);
    return &DoControlMgr;
}
```

Registers the parameter group, change notification handler, and returns a callback function that can be attached to a tool button (e.g., "Manage Controls" menu item).

---

### Element Loading

```c
EXPORT void ContMgmLoad(
    wIcon_p icon,       // Icon for this element type
    contMgmCallBack_p proc,  // Callback dispatcher
    void *data         // Pointer to the first/representative data struct
)
```

Registers a control element. The callback (`proc`) and data pointer are stored in the context. When an element is selected, these are used to dispatch operations like editing or deletion.

---

### Management Dialog Update

```c
static void ControlDlgUpdate(paramGroup_p pg, int inx, void *valueP)
{
    contMgmContext_p context = NULL;
    wIndex_t selcnt = wListGetSelectedCount(controlSelL);
    wIndex_t linx, lcnt;

    if (inx != I_CONTROLLIST) { return; }  // Only react to list selection changes

    // Toggle highlight on selected items, un-highlight others
    for (linx=0; linx < lcnt; linx++) {
        if (wListGetItemSelected(controlSelL, linx)) {
            context = (contMgmContext_p)wListGetItemContext(controlSelL, linx);
            context->proc(CONTMGM_DO_HILIGHT, context->data);
            AnyHILIGHT = TRUE;
        } else {
            context = (contMgmContext_p)wListGetItemContext(controlSelL, linx);
            context->proc(CONTMGM_UN_HILIGHT, context->data);
        }
    }

    // Enable/disable Edit/Delete buttons based on selection count
    ParamControlActive(&controlPG, I_CONTROLEDIT,  selcnt > 0);
    ParamControlActive(&controlPG, I_CONTROLDEL,   selcnt > 0);
}
```

This is the glue function that:
- Highlights selected elements in the list (via callbacks)
- Enables/disables Edit/Delete buttons based on whether any element is selected

---

### Editing an Element

```c
static void ControlEdit(void *action)
{
    contMgmContext_p context = NULL;
    wIndex_t selcnt = wListGetSelectedCount(controlSelL);
    wIndex_t inx, cnt;

    if (selcnt != 1) { return; }  // Must have exactly one selection

    cnt = wListGetCount(controlSelL);
    for (inx=0; inx<cnt && !wListGetItemSelected(controlSelL, inx); inx++);
    if (inx >= cnt) { return; }

    context = (contMgmContext_p)wListGetItemContext(controlSelL, inx);

    // Dispatch to type-specific edit handler
    context->proc(CONTMGM_DO_EDIT, context->data);

    // Update the list entry with the new title (e.g., edited name or aspect string)
    context->proc(CONTMGM_GET_TITLE, context->data);
    wListSetValues(controlSelL, inx, message, context->icon, context);
}
```

The framework **does not** perform editing itself. Instead, it calls the registered element's callback with `CONTMGM_DO_EDIT`, which then opens a type-specific dialog (e.g., "Edit Block" for blocks, "Edit Switchmotor" for switchmotors). After editing completes, `CONTMGM_GET_TITLE` is called to refresh the list display.

---

### Deleting Elements

```c
static void ControlDelete(void *action)
{
    wIndex_t selcnt = wListGetSelectedCount(controlSelL);
    wIndex_t inx, cnt;
    contMgmContext_p context = NULL;

    if (selcnt <= 0) { return; }

    // Confirm deletion with user
    if (!NoticeMessage2(1, _("Are you sure..."), _("Yes"), _("No"), selcnt)) {
        return;
    }

    cnt = wListGetCount(controlSelL);
    UndoStart(_("Control Elements"), "delete");

    for (inx=0; inx<cnt; inx++) {
        if (!wListGetItemSelected(controlSelL, inx)) continue;

        context = (contMgmContext_p)wListGetItemContext(controlSelL, inx);
        context->proc(CONTMGM_DO_DELETE, context->data);  // Type-specific delete handler
        MyFree(context);
        wListDelete(controlSelL, inx);
        inx--; cnt--;
    }

    UndoEnd();
    DoChangeNotification(CHANGE_PARAMS);
}
```

Deletion is wrapped in an undo transaction. Each element type handles its own cleanup (e.g., freeing allocated memory associated with the element).

---

### Done / Close Handler

```c
static void ControlDone(void *action)
{
    contMgmContext_p context = NULL;
    wIndex_t linx, lcnt;

    // Un-highlight all elements before closing dialog
    if (AnyHILIGHT) {
        for (linx=0; linx<lcnt; linx++) {
            context = (contMgmContext_p)wListGetItemContext(controlSelL, linx);
            context->proc(CONTMGM_UN_HILIGHT, context->data);
        }
    }

    wHide(controlPG.win);  // Hide the dialog window
}
```

---

### Change Notification Handler

```c
static void ContMgmChange(long changes)
{
    if (changes) {
        if (changed) {
            changed = checkPtMark = 1;
        }
    }

    // If no parameter change or dialog not visible, bail out
    if ((changes & CHANGE_PARAMS) == 0 ||
        controlPG.win == NULL || !wWinIsVisible(controlPG.win)) {
        return;
    }

    LoadControlMgmList();  // Rebuild the list from current state
}
```

This handler is registered via `RegisterChangeNotification()`. It rebuilds the management list whenever parameter changes are detected (e.g., a new block was added, or an element's script was modified). The `changes` bitmask indicates what kind of change occurred.

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `ControlMgrInit()` | Initialize the management framework | Registers param group and change handler; returns callback for tool button |
| `ContMgmLoad(icon, proc, data)` | Register a control element type or instance | Callback-driven registration pattern |
| `ControlDlgUpdate(pg, inx, valueP)` | Handle list selection changes (highlight, enable buttons) | Iterates all items and calls their highlight callbacks |
| `ControlEdit(action)` | Open edit dialog for selected element | Uses the stored callback to dispatch to type-specific handler |
| `ControlDelete(action)` | Delete selected elements with undo support | Calls each element's delete callback; frees memory |
| `ControlDone(action)` | Close dialog, un-highlight, hide window | Ensures clean state before closing |
| `ContMgmChange(changes)` | Rebuild list on parameter changes | Called from change notification system |

---

## Domain & Design Notes

- **Callback-driven architecture**: This is the key design pattern. Every element type (blocks, switchmotors, sensors) registers its own set of callbacks (`CONTMGM_GET_TITLE`, `CONTMGM_DO_EDIT`, `CONTMGM_GET_TITLE`, `CONTMGM_UN_HILIGHT`, etc.). The framework code doesn't need to know about any specific element type — it simply invokes the stored callback pointer.

- **Change notification integration**: The `ContMgmChange` handler is wired into XTrkCAD's global change notification system (via `RegisterChangeNotification`). This means that whenever *any* parameter file changes are detected, the control management list is automatically refreshed without requiring explicit refresh calls from elsewhere in the codebase.

- **Highlighting**: The `AnyHILIGHT` flag and `CONTMGM_DO_HILIGHT` / `CONTMGM_UN_HILIGHT` callbacks provide a generic way for element types to highlight themselves on the layout canvas when selected in the dialog (e.g., dimming tracks associated with a block, highlighting a turnout controlled by a switchmotor).

- **Undo support**: Deletions are wrapped in an undo transaction (`UndoStart` / `UndoEnd`). This allows users to undo accidental deletions via XTrkCAD's undo system.
