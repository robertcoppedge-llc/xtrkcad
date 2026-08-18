# dcontmgm.c — Layout Control Element Management

## Overview

`dcontmgm.c` provides **layout control element management** functionality. It manages a global list of "control elements" that are associated with layout objects (tracks, turnout motors, signals, sensors). These control elements serve as the bridge between physical layout components and external control software (DCC decoders, LCC event systems, etc.).

The module itself is generic — it doesn't define specific element types. Instead, it provides a **callback-based framework** that other modules (e.g., `cswitchmotor.c`, `csignal.c`) plug into to register their own elements.

---

## Core Concepts

### Control Elements

A control element represents an association between:
- A **layout object** (a track segment, turnout motor, sensor location, etc.)
- **Control data** — a text string or other payload that external software uses for I/O addressing, event routing, etc.

Examples of what a control element might represent:
| Element Type | Layout Association | Control Data Example |
|---|---|---|
| Switchmotor | A turnout motor track segment | `"DCC addr=23"` or `"LCC event=SWITCH_04"` |
| Block (occupancy detector) | A block section | `"BLOCK_A1"` for occupancy detection software |
| Signal sensor | A signal location | `"SIG_RED_A1"` for a red-light indication |

The control data is essentially free-form text — XTrkCAD doesn't enforce any particular syntax. The external software (e.g., DCC controller, LCC network) parses it as needed.

---

### The Generic Callback Framework

Each element type registers itself with the generic manager by calling `ContMgmLoad(icon, proc, data)` where:
- `icon`: A widget icon (used in the dialog UI for visual identification)
- `proc`: A function pointer that handles all operations (edit, delete, highlight, get title, etc.)
- `data`: An opaque pointer passed through to each operation

The manager then presents a unified list view with "Edit" and "Delete" buttons.

---

## Data Structures

### `contMgmContext_t` — Element Registration Context

```c
typedef struct {
    contMgmCallBack_p proc;  // Function pointer: CONTMGM_DO_EDIT, DO_DELETE, GET_TITLE, etc.
    void * data;             // Opaque pointer to the element's own structure
    wIcon_p icon;            // Widget icon for display in the list
} contMgmContext_t, *contMgmContext_p;
```

Each entry in the global management list (`controlSelL`) is a `contMgmContext_t` struct. The `proc` field points to a function that handles all operations on this particular element type. This allows multiple different element types (switch motors, sensors, etc.) to coexist in the same list while being handled by their own specialized code paths.

---

## Core Functions

### `ContMgmLoad(icon, proc, data)` — Register an Element Type

Called when a layout file is loaded or when new elements are added. Registers a new element type with the management system:
- Allocates a new context struct
- Stores the callback function and associated data
- Calls `GET_TITLE` to retrieve a label string (for display in the list)
- Adds an entry to the global widget list

---

### `ControlEdit(action)` — Open Edit Dialog for Selected Element

When the user clicks "Edit" on a selected element:
1. Checks that exactly one item is selected; if not, returns early
2. Retrieves the context via `wListGetItemContext()`
3. Calls the registered callback's `CONTMGM_DO_EDIT` handler
4. Calls `CONTMGM_GET_TITLE` to fetch the current value for pre-filling the dialog
5. Sets the list entry with the new message and icon

---

### `ControlDelete(action)` — Delete Selected Elements

1. Gets the count of selected items
2. If zero are selected, returns early
3. Shows a confirmation dialog: "Are you sure you want to delete the N control element(s)?"
4. On affirmative, starts an undo transaction with message "Control Elements" (action="delete")
5. Iterates through all selected entries in the list, calling each context's `CONTMGM_DO_DELETE` handler
6. Frees the context struct and removes it from the widget list
7. Ends the undo transaction
8. Triggers a change notification (`CHANGE_PARAMS`) so dependent views refresh

---

### `ControlDone(action)` — Close the Dialog

Called when the user clicks "OK" without making changes or when closing the dialog:
- If any element is currently highlighted (via a HILIGHT callback), it calls each context's unhighlight handler to clear visual feedback
- Hides the dialog window

---

### `ControlDlgUpdate(pg, inx, valueP)` — Update Dialog State

Called whenever the selection in the list changes. It:
1. Returns early if the changed control isn't the element-list widget
2. Loops through all items currently selected in the list
3. For each one, toggles its HILIGHT/UN_HILIGHT callback on or off
4. Enables/disables the "Edit" and "Delete" buttons based on whether any item is selected

---

### `LoadControlMgmList()` — Refresh the Management List

Called after a layout file load (or when parameters change). It:
1. Saves the current selection index
2. Iterates through all entries and frees them
3. Clears the widget list
4. Calls sub-loaders for each element type: `BlockMgmLoad()`, `SwitchmotorMgmLoad()`, `SignalMgmLoad()`, `SensorMgmLoad()`
5. Restores the selection index

---

### `ContMgmChange(changes)` — React to Parameter Changes

A change notification handler that refreshes the list when parameters are modified (e.g., a layout file is loaded or reloaded). If changes include parameter updates and the dialog isn't visible, it calls `LoadControlMgmList()`.

---

### `DoControlMgr(junk)` — Entry Point for Dialog Creation

Called from the button handler to create/show the management dialog. It creates the param dialog (if not already created), clears the list, then loads all registered element types via their respective load functions.

---

### `ControlMgrInit()` — Module Initialization

The module's initialization function that:
- Registers the param group with its controls
- Registers a change notification handler so the list refreshes when needed
- Returns the button callback to be registered in a toolbar/menu

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `ContMgmLoad(icon, proc, data)` | Register an element type with the management system; add entry to global list | icon widget handle, callback function pointer, opaque data pointer |
| `ControlEdit(action)` | Open edit dialog for selected element; calls its GET_TITLE handler and sets pre-filled values | unused action pointer (widget button) |
| `ControlDelete(action)` | Delete selected elements one-by-one via their DO_DELETE callbacks | unused action pointer |
| `ControlDone(action)` | Clean up highlights and hide the management dialog window | unused action pointer |
| `ControlDlgUpdate(pg, inx, valueP)` | Refresh button states and highlight state based on current selection | param group pointer, control index |
| `LoadControlMgmList()` | Load all registered element types into the global list; called after file load | none |
| `ContMgmChange(changes)` | React to parameter changes by reloading the list if needed | bitmask of changed parameters |
| `DoControlMgr(junk)` | Create and show the management dialog window | unused junk pointer |
| `ControlMgrInit()` | Module initialization; registers param group, change handler, returns button callback | none |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide a generic framework for managing control elements (switch motors, block sensors, signal indicators) that associate external control data with layout objects |
| **Domain** | Layout control system integration: bridging physical track elements to digital control signals and I/O addresses used by external software |
| **Key concept** | A **generic callback-based framework**: element types register themselves by providing a function pointer that handles all operations (edit, delete, get title). The manager doesn't need to know the specific type — it just calls through the registered callbacks. This allows adding new element types without modifying the core management code. |
| **Main entry points** | `ContMgmLoad()` — register an element type; `DoControlMgr()` / button callback — open the dialog; each element type must provide a function matching the `contMgmCallBack_p` signature to handle edit/delete/get title/highlight operations |
