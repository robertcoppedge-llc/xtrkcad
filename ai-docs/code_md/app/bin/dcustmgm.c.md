# dcustmgm.c — Custom Parts Management Framework

## Overview

`dcustmgm.c` provides a **generic management framework** for user-defined (custom) parts and prototypes. It allows users to:

- Create new custom car parts or car prototypes
- Edit existing custom definitions (rename, modify descriptions)
- Delete unused custom entries from memory
- Export selected custom items to a parameter file (`*.xtp`)

The module is **generic** — it doesn't define what "custom" means. Instead, it provides a callback-based framework that other modules plug into:

| Custom Type | Example Module | What It Manages |
|---|---|---|
| Car Part | `car.c` / `dcar.c` | Individual car components (body, cab, wheels, etc.) |
| Car Prototype | `dcar.c` | Complete locomotive or rolling stock definitions |

Each registered type provides callbacks for: edit, delete, copy-to-file, get title.

---

## Core Concepts

### Custom Parts vs. Prototypes

- **Car Part**: A sub-component of a vehicle (e.g., "Boiler", "Cab", "Body") — these are pieces that can be combined into a full prototype
- **Car Prototype**: A complete vehicle definition (locomotive, passenger car, freight car) with dimensions, colors, and geometry

Both can be marked as "custom" (`PARAM_CUSTOM`) meaning they were created by the user rather than loaded from a parameter file.

---

## Data Structures

### `custMgmContext_t` — Registration Context

```c
typedef struct {
    custMgmCallBack_p proc;  // Function pointer: handles edit/delete/copyto/get_title/etc.
    void * data;             // Opaque pointer to the element's own structure (e.g., carPart_t*)
    wIcon_p icon;            // Widget icon for display in the list
} custMgmContext_t, *custMgmContext_p;
```

Each entry in the global management list (`customSelL`) is a `custMgmContext_t`. The `proc` function pointer is what allows different custom types (car parts vs. car prototypes) to coexist — each calls back into its own specialized handler.

---

## Core Functions

### `CustomEdit(action)` — Open Edit Dialog for Selected Item

1. Checks that exactly one item is selected; returns early otherwise
2. Gets the context via `wListGetItemContext()` from the list widget
3. Calls the registered callback's `CUSTMGM_DO_EDIT` handler (which opens a dialog specific to that type)
4. On old code path (disabled), would call `GET_TITLE` and update the list entry

---

### `CustomNewCar(action)` — Create a New Custom Item

Opens a dialog to create a new custom part or prototype:

```c
switch(selectedType) {
    case 1:   // Car Prototype
        CarDlgAddProto();   // from dcar.c
        break;
    case 0:   // Car Part
        CarDlgAddDesc();    // from car.c (or similar)
        break;
}
```

The `selectedType` is set via a dropdown in the dialog to let the user choose what kind of item they want to create.

---

### `CustomDelete(action)` — Delete Selected Custom Items

1. Gets the count of selected items
2. Shows confirmation: "Are you sure you want to delete the N custom part(s)?"
3. If confirmed, starts an undo transaction with label "delete"
4. Iterates through all selected entries in the list
5. For each one: calls its `CUSTMGM_DO_DELETE` callback (sets segment count to 0), frees memory, removes from widget list
6. Ends undo transaction
7. Triggers a change notification

---

### `CustomExport(action)` — Export Selected Items to Parameter File

Opens a file selector (`wFilSel`) and then calls `CustomDoExport()`. That function:

1. Checks if the target file already exists; if not, prompts for a contents label
2. If the file exists but isn't writable, shows an error message
3. Opens the file in append mode (`"a"`)
4. Writes a header line with optional UTF-8 conversion (if compiled with `UTFCONVERT`)
5. Iterates through selected items and calls their `CUSTMGM_DO_COPYTO` callback:
   - For turnout/structure custom parts, this writes the definition to the file
   - For car prototypes/parts, similar write logic exists in `dcustmgm.c` or `dcar.c`
6. Calls each item's `DO_DELETE` handler (removes from memory)
7. Closes the file and reloads the parameter file so changes take effect

---

### `CustomDone(action)` — Save All Custom Items to Default File

Called when the user clicks "OK" on the management dialog without selecting specific items:

```c
FILE * f = OpenCustom("w");   // opens custom.xtp in write mode
CompoundCustomSave(f);        // writes all compound (turnout/structure) definitions
CarCustomSave(f);             // writes all car prototype/part definitions
fclose(f);
```

This is essentially a "save all" operation that persists the user's custom creations to disk.

---

### `CustMgmLoad(icon, proc, data)` — Register a Custom Type

Called by sub-modules (e.g., `CompoundCustMgmLoad()` from `dcmpnd.c`) when they want to add themselves to the management system:

```c
context = MyMalloc(sizeof *context);
context->proc = proc;    // pointer to callback function with CUSTMGM_* operations
context->data = data;     // opaque pointer (e.g., pointer into a sorted array)
context->icon = icon;     // widget icon for UI display
context->proc(CUSTMGM_GET_TITLE, context->data);  // get label text
wListAddValue(customSelL, message, icon, context); // add to the global list
```

The `data` pointer is typically a pointer into a sorted array (e.g., an entry in a `dynArr_t`) so that lookups can be done by binary search.

---

### `LoadCustomMgmList()` — Refresh the Management List

Called after loading a layout file or when parameters change:

1. Saves the current selection index
2. Iterates through all entries and frees them (memory is reclaimed)
3. Clears the widget list
4. Calls sub-loaders in order: `CompoundCustMgmLoad()` then `CarCustMgmLoad()`
5. Restores the selection index

---

### `CustMgmChange(changes)` — Change Notification Handler

Called when layout parameters are modified (e.g., a new file is loaded). It refreshes the management list if parameter changes occurred and the dialog isn't currently visible.

---

### `DoCustomMgr(junk)` — Entry Point for Dialog Creation

Creates or re-shows the "Manage custom designed parts" dialog:
- If the param group doesn't exist yet, creates it with a dropdown of types to create
- Loads all registered custom entries into the list widget
- Shows the window

---

### `CustomMgrInit()` — Module Initialization

Registers the param group, registers change notifications, and returns the button callback function. This is called once at application startup (or when registering menu items).

---

## Design Decisions & Tradeoffs

### Why a Callback Framework?

The core management functions (`Edit`, `Delete`, `CopyTo`) are completely generic — they don't know anything about car parts or turnout definitions. They simply:
- Retrieve the context from the widget list (which holds a pointer to the element's own structure)
- Call the registered callback with a command code and data pointer

This allows new custom types to be added without modifying `dcustmgm.c` itself — just implement a matching `custMgmCallBack_p` function signature and call `CustMgmLoad()` from that module.

### Why Store Context in the Widget List?

The widget list (`customSelL`) stores pointers into the element arrays (e.g., entries in `turnoutInfo_da`). This avoids needing to maintain a separate parallel index table. The context struct holds:
- A pointer back to the original data structure (for editing/deletion)
- An icon for UI display
- A function pointer that knows how to operate on that specific entry

### Why Delete Immediately After Copying?

In `CustomExport()`, each item is copied to a file and then immediately deleted from memory. This keeps the in-memory database clean — custom items are "ephemeral" until explicitly saved to a `.xtp` file, at which point they're persisted externally and removed from XTrkCAD's RAM-resident database.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CustomEdit(action)` | Open edit dialog for selected custom item; calls its DO_EDIT callback | unused action pointer (widget button) |
| `CustomNewCar(action)` | Create a new custom part or prototype based on the currently selected type in dropdown | unused action pointer |
| `CustomDelete(action)` | Delete all selected items from memory and from the widget list | unused action pointer |
| `CustomExport(action)` | Export selected items to a parameter file; removes them from memory after writing | unused action pointer |
| `CustMgmLoad(icon, proc, data)` | Register a custom type with the framework; add entry to management list | icon widget handle, callback function pointer, opaque data pointer |
| `CustomDone(action)` | Save ALL custom items (compound parts and car prototypes) to default file | unused action pointer |
| `LoadCustomMgmList()` | Refresh the management list by clearing and reloading entries from sub-modules | none |
| `CustMgmChange(changes)` | React to parameter changes; refresh the list if needed | bitmask of changed parameters |
| `DoCustomMgr(junk)` | Create/show the "Manage custom parts" dialog window | unused junk pointer |
| `CustomMgrInit()` | Module initialization; registers param group and change handler | none |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide a generic framework for managing user-created (custom) parts and prototypes. Allows edit, delete, export operations without modifying the core manager code. |
| **Domain** | Custom database: user-defined car parts, locomotive definitions, turnout/structure variants that aren't in any parameter file but exist only in memory until exported to a `.xtp` file |
| **Key concept** | A **callback-based dispatch model**: each custom type registers its own handler function. The generic manager doesn't need to know the difference between a car part and a turnout — it just calls `context->proc(cmd, context->data)` and lets that function handle it. |
| **Main entry points** | `CustMgmLoad()` — called by each sub-module when registering itself; `CustomMgrInit()` — module initialization; button callbacks (`Edit`, `Delete`, `Export`) — user-facing operations |
