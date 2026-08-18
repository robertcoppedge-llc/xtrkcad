# dcustmgm.c — Custom Management Support (Compound/Car)

## Overview

`dcustmgm.c` implements a **custom management framework** for handling user-editable custom data in XTrkCAD. It provides a generic UI dialog that allows users to edit free-form text fields associated with various objects — such as compound track turnouts, car descriptions, and other parameter-file-based entities.

The file is a lightweight utility that wraps around the underlying `ccustmgm.c` (in the `app/cornu/` directory) for the actual editing logic, but provides XTrkCAD-specific integration: undo support, change notifications, icon handling, and a unified dialog interface.

---

## Key Data Structure

```c
typedef struct {
    char *name;          // Human-readable name (e.g., "Turnout", "Car Part")
    FILE *f;             // File handle for writing custom data to parameter file
    contMgmCallBack_p proc;  // Callback dispatcher
    void *data;         // Pointer to the object being managed
    wIcon_p icon;       // Icon displayed in management list
} custMgmContext_t, *custMgmContext_p;
```

The `name` field identifies what type of custom data is being edited (e.g., "Turnout", "Car Part"). The `proc` callback dispatches operations like edit, delete, get title. The `f` file handle is used when writing the custom data back to a parameter file.

---

## Key Functions

### Custom Management Initialization

```c
EXPORT addButtonCallBack_t CustMgmInit(void)
{
    ParamRegister(&custMgmPG);
    RegisterChangeNotification(CustMgmChange);
    return &DoCustMgm;
}
```

Registers the parameter group and change notification handler, returning a callback for attaching to tool buttons.

---

### Element Registration

```c
EXPORT void CustMgmLoad(
    wIcon_p icon,
    custMgmCallBack_p proc,
    void *data
)
{
    custMgmContext_p context;
    context = MyMalloc(sizeof *context);
    context->proc   = proc;
    context->data   = data;
    context->icon   = icon;
    context->name   = "Custom";  // Default name (set by caller)

    if (data != NULL) {
        context->proc(CUSTMGM_GET_TITLE, context->data);
    } else {
        context->name = MyStrdup(_("Unknown"));
    }

    wListAddValue(custSelL, message, icon, context);
}
```

Registers a custom-manageable object. The callback (`proc`) and data pointer are stored for later dispatch of edit/delete operations. If no valid object is registered (e.g., no turnouts/structures defined), the name becomes "Unknown".

---

### Management Dialog Update

```c
static void CustMgmChange(long changes)
{
    if ((changes & CHANGE_PARAMS) == 0 ||
        custMgmPG.win == NULL || !wWinIsVisible(custMgmPG.win)) {
        return;
    }

    LoadCustMgmList();  // Rebuild the list
}
```

Rebuilds the management list whenever parameter changes are detected (e.g., a custom turnout was deleted). The `changes` mask is checked for `CHANGE_PARAMS`.

---

### Management Dialog Open

```c
static void DoCustMgm(void *junk)
{
    if (custMgmPG.win == NULL) {
        ParamCreateDialog(&custMgmPG,
                          MakeWindowTitle(_("Manage Custom Objects")),
                          _("Done"), CustMgmDone,
                          ParamCancel_Current, TRUE, NULL,
                          F_RESIZE|F_RECALLSIZE|F_BLOCK);
    }

    LoadCustMgmList();  // Build list from all registered custom objects
    wShow(custMgmPG.win);
}
```

Opens the management dialog. The `LoadCustMgmList()` function iterates over all registered custom objects (turnouts, car parts, etc.) and populates a drop-down list. Each entry stores its callback and data pointer in the item context for later dispatch.

---

### Editing an Object

```c
static void CustMgmEdit(void *action)
{
    custMgmContext_p context = NULL;
    wIndex_t selcnt, cnt, inx;

    if ((selcnt = wListGetSelectedCount(custSelL)) != 1) return;

    for (inx=0; inx<cnt && !wListGetItemSelected(custSelL, inx); inx++);
    if (inx >= cnt) return;

    context = (custMgmContext_p)wListGetItemContext(custSelL, inx);
    if (context == NULL) return;

    // Dispatch to type-specific edit handler
    context->proc(CUSTMGM_DO_EDIT, context->data);

    // Refresh the list entry with new title
    context->proc(CUSTMGM_GET_TITLE, context->data);
    wListSetValues(custSelL, inx, message, context->icon, context);
}
```

Dispatches to the stored callback for editing. The callback (e.g., `TurnoutEditProc`) opens a type-specific dialog where the user edits parameters and optionally adds custom data.

---

### Deleting an Object

```c
static void CustMgmDelete(void *action)
{
    wIndex_t selcnt, cnt, inx;
    custMgmContext_p context = NULL;

    if ((selcnt = wListGetSelectedCount(custSelL)) <= 0) return;

    if (!NoticeMessage2(1, _("Delete %d custom object(s)?"), _("Yes"), _("No"), selcnt))
        return;

    cnt = wListGetCount(custSelL);
    UndoStart(_("Custom Objects"), "delete");

    for (inx=0; inx<cnt; inx++) {
        if (!wListGetItemSelected(custSelL, inx)) continue;

        context = (custMgmContext_p)wListGetItemContext(custSelL, inx);
        context->proc(CUSTMGM_DO_DELETE, context->data);  // Type-specific delete handler
        MyFree(context);
        wListDelete(custSelL, inx);
        inx--; cnt--;
    }

    UndoEnd();
    DoChangeNotification(CHANGE_PARAMS);
}
```

Calls each object's delete callback (e.g., `TurnoutDelete` sets the turnout's segment count to zero). Deletions are wrapped in an undo transaction.

---

### Done / Close Handler

```c
static void CustMgmDone(void *junk)
{
    custMgmContext_p context = NULL;
    wIndex_t linx, lcnt;

    for (linx=0; linx<lcnt; linx++) {
        context = (custMgmContext_p)wListGetItemContext(custSelL, linx);
        if (context != NULL && context->name[0] == '\0') {
            MyFree(context);
            wListDelete(custSelL, linx);
            linx--; lcnt--;
        }
    }

    wHide(custMgmPG.win);
}
```

Removes entries from the list that have an empty `name` field (indicating they were orphaned or invalidated). Then hides the dialog window.

---

## How It Works with Other Modules

This file is a **thin wrapper** around type-specific implementations in other files:

- **Turnouts/Structures**: Uses callbacks from `dcmpnd.c` (`TurnoutEditProc`, `TurnoutDelete`, etc.)
- **Car Parts/Items**: Uses callbacks from `dcar.c` (e.g., `CarPartEditProc`)

Each object type registers itself via `CustMgmLoad()` with a pointer to its own callback set. The framework code in `dcustmgm.c` never needs to know the concrete type — it just invokes `context->proc(CUSTMGM_DO_EDIT, context->data)`.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `CustMgmInit()` | Initialize framework, register change notification |
| `CustMgmLoad(icon, proc, data)` | Register a custom-manageable object with its callbacks |
| `DoCustMgm(junk)` | Open the management dialog and build the list |
| `CustMgmChange(changes)` | Rebuild list on parameter changes |
| `CustMgmEdit(action)` | Dispatch edit operation to type-specific handler |
| `CustMgmDelete(action)` | Delete selected objects with undo support |
| `CustMgmDone(junk)` | Close dialog, clean up orphaned entries |

---

## Domain & Design Notes

- **Callback-driven dispatch**: Like `dcontmgm.c`, this uses a callback pattern. Each object type provides its own set of callbacks (`DO_EDIT`, `DO_DELETE`, `GET_TITLE`) that the framework invokes polymorphically through the stored function pointer.
- **Change notification integration**: The `CustMgmChange` handler is wired into XTrkCAD's global change notification system, so the list automatically refreshes when objects are added/removed from parameter files.
- **Undo support**: Deletions are wrapped in undo transactions (`UndoStart` / `UndoEnd`), allowing users to recover accidentally deleted custom objects.
