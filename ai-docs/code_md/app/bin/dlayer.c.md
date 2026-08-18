# dlayer.c — Layer Management System

## Overview

`dlayer.c` provides a comprehensive **layer management system** for organizing tracks and layout objects. It supports up to 100 layers with features including:

- Per-layer properties (name, color, visibility, frozen state, module flag)
- Linking layers so they show/hide together
- Associating settings files with specific layers
- Tooltips/balloon help for each layer button
- Undo support for layer creation/deletion
- File I/O (`ReadLayers`/`WriteLayers`) for persistence

---

## Core Concepts

### Layer Architecture

XTrkCAD uses a **layer-based** organization model:

| Concept | Description |
|---------|-------------|
| **Main Layer (index 0)** | Always exists; used as the default/current layer |
| **User Layers (1–99)** | User-defined layers that can be toggled on/off |
| **Frozen layers** | Cannot be selected or shown/hidden via toolbar buttons |
| **Module layers** | "All-or-nothing" — either all objects are on the layer or none are |
| **Linked layers** | When one is shown, others show/hide automatically (useful for grouping related track sections) |

---

### Layer vs. Track Layer Assignment

A critical distinction: **layers ≠ tracks**. A single track object can span multiple layers. This is different from many CAD systems where a layer *is* a drawing entity. Here, the layer system provides organizational metadata that influences rendering order and visibility but does not replace the track data structure itself.

---

## Data Structures

### `layer_t` — The Layer Structure

```c
typedef struct {
    char name[STR_SHORT_SIZE];          // Display name (e.g., "Main", "Curve 1")
    wDrawColor color;                    // Color for rendering objects on this layer
    BOOL_T useColor;                     // Whether to use the stored color or inherit
    BOOL_T frozen;                       // Cannot be selected/shown via UI
    BOOL_T visible;                      // Currently visible state
    BOOL_T onMap;                        // Shown in map view (if applicable)
    BOOL_T module;                       // "Module" mode — all-or-nothing visibility
    BOOL_T button_off;                   // Hide the toolbar button for this layer
    BOOL_T inherit;                      // Inherit defaults from layout/scale settings
    SCALEINX_T scaleInx;                // Scale override for objects on this layer
    SCALEDESCINX_T scaleDescInx;        // Human-readable scale name
    GAUGEINX_T gaugeInx;                // Gauge type (standard, narrow, etc.)
    DIST_T minTrackRadius;              // Minimum radius allowed on this layer
    ANGLE_T maxTrackGrade;              // Maximum grade percentage
    tieData_t tieData;                  // Tie spacing/width/length settings
    long objCount;                       // Number of track objects on this layer
    dynArr_t layerLinkList;             // 1-based indices of linked layers
    char settingsName[STR_SHORT_SIZE];  // Settings file name (.xset) for this layer
} layer_t, *layer_p;

#define NUM_LAYERS 100   // Maximum number of defined layers
```

The `layers` array is declared as:
```c
static layer_t layers[NUM_LAYERS];
```

---

### Layer Flags (`LAYERPREF_*`)

Bit flags stored in an integer preference field:

| Flag | Value | Meaning |
|------|-------|---------|
| `LAYERPREF_FROZEN` | 1 | Cannot be selected via toolbar |
| `LAYERPREF_ONMAP` | 2 | Visible in map view |
| `LAYERPREF_VISIBLE` | 4 | Currently visible |
| `LAYERPREF_MODULE` | 8 | Module mode (all-or-nothing) |
| `LAYERPREF_NOBUTTON` | 16 | No toolbar button shown |
| `LAYERPREF_DEFAULT` | 32 | Inherits all defaults from layout/scale |

---

### Layer Preferences (`LAYERPREF_*`)

Preference keys used in the preferences file:

- `name.<n>` — Name of layer N
- `color.<n>` — RGB color value for layer N
- `flags.<n>` — Bitmask of flags
- `scaleInx.<n>` — Scale index override
- `sclDescInx.<n>` — Scale description index
- `gaugeInx.<n>` — Gauge type index
- `minRadius.<n>` — Minimum track radius
- `maxGrade.<n>` — Maximum grade percentage
- `tieLength.<n>`, `tieWidth.<n>`, `tieSpacing.<n>` — Tie data
- `list.<n>` — Semicolon-separated linked layer indices (1-based)
- `settings.<n>` — Settings file name for this layer

---

## Core Functions

### `IsLayerValid(layer)` / `GetLayerVisible()` / `GetLayerFrozen()` / ...

A series of wrapper functions that check validity (`layer <= NUM_LAYERS && layer != -1`) before accessing the array. This guards against out-of-bounds access when callers use invalid indices (e.g., `-1` for "not set").

---

### `FormatLayerName(inx)` — Generate Display Label

Formats a human-readable label combining:
- The numeric index + 1 (layers are 0-indexed internally but displayed as 1–N)
- A prefix character based on flags (`*` = frozen, `m` = module, `+` = has objects, `-` = empty)
- The layer name

Example output: `"3 * Curve1"` or `"8 m Turnout"` — where the prefix gives immediate visual feedback about the layer's state.

---

### `FlipLayer(layerVP)` — Toggle Visibility

The primary function for toggling a layer on/off. It:
1. Validates the index and returns early if frozen (showing "Cannot freeze" message)
2. Calls `RedrawLayer()` to refresh the display before changing state
3. Flips the `visible` flag
4. Updates the toolbar button's icon/color via `wButtonSetBusy()` and `wIconSetColor()`
5. Propagates visibility changes to all linked layers (looping through `layerLinkList`)
6. Calls `RedrawLayer()` again after state change

---

### `SetCurrLayer(inx, name, op, listContext, arg)` — Switch Current Layer

Sets the "current" layer for new track objects:
- If the target is frozen, shows an error and reverts to the first non-frozen layer
- Handles settings file loading (if the current layer has a `.xset` file associated)
- Propagates visibility state to linked layers
- Writes `SETCURRLAYER <n>` to a record/playback stream if recording

---

### `SetLayerColor()` / `FormatLayerName()` — Update UI Feedback

These update the toolbar button's appearance:
- Change the icon color via `wIconSetColor()`
- Update the balloon help text with the layer name (or "Show/Hide Layer" default)

The `oldColorMap[]` array handles backward compatibility for older files that used a different 10-color palette.

---

### `LayerSystemDefault(inx)` / `IsLayerDefault(inx)` / `LayerAllDefaults()` — Reset to Defaults

- **`LayerSystemDefault`**: Resets all properties of one layer to system defaults (empty name, default color, no flags, inherits from layout/scale)
- **`IsLayerDefault`**: Checks whether a layer has been customized in any way — returns TRUE only if the layer differs from its default state. This is used when deciding what to write to file output.
- **`LayerAllDefaults`**: Resets ALL layers (calls `LayerSystemDefault()` for each).

---

### `LayerAdd()` / `LayerDelete()` / `LayerDefault()` — Modify Layer List

These functions operate on the global layer list:

**`LayerAdd()`** inserts a new layer after `layerSelected`:
- Shifts all subsequent layers down by one index (like an array insertion)
- Calls `TrackInsertLayer()` to register with the track system
- Initializes with default values and name "New Layer"

**`LayerDelete()`**: Removes a layer — but only if it has **zero objects** on it. This prevents accidental deletion of layers that contain active tracks. After removal, all subsequent layers shift up; `maxLayer` is decremented.

**`LayerDefault()`**: Resets the current layer's scale/tie/grade/radius settings to either layout defaults (if `inherit` flag set) or the currently selected values.

---

### `LoadLayerLists()` — Refresh All Lists

After loading a file or changing parameters, this rebuilds:
- The toolbar layer selector dropdown (`setLayerL`)
- The dialog's layer listbox (`layerL`)
- The settings catalog (used for the "Settings when Current" dropdown)

---

### `DoLayerOp()` — Dialog Button Handler

Dispatches actions from the "Manage Layers" dialog buttons:
- **Clear** → resets all layers to defaults, then calls `InitializeLayers(LayerAllDefaults, -1)`
- **Save** → writes current layer settings to preferences file
- **Reload** → reads layer settings back from preferences
- **Add Layer** → inserts a new layer into the array and updates UI
- **Delete Layer** → removes the selected layer (only if empty)
- **Default Values** → resets scale/tie data to defaults

The function also toggles enablement of the "Delete" button based on whether a non-default layer is selected.

---

### `InitializeLayers(initFunc, newCurrLayer)` — Setup/Reset

A factory-style initializer that:
1. Calls `initFunc()` (could be `LayerAllDefaults`, `LayerPrefLoad`, or a custom function)
2. Counts objects on each layer via `LayerSetCounts()`
3. If a specific current layer index was requested, finds the first non-frozen layer starting from there; if none exist, shows an error and sets layer 0

---

### `LayerChange(changes)` — Change Notification Handler

Registered as a change notification listener. When parameter changes occur (e.g., layout loaded), it reloads the dialog controls if the layer dialog is visible.

---

### `GetLayerLinkString()` / `PutLayerListArray()` — Manage Layer Links

- **`GetLayerLinkString`**: Converts the internal 1-based array of linked layer indices into a semicolon-separated string for file I/O
- **`PutLayerListArray`**: Parses a string like `"3;5;8"` and populates the `layerLinkList` dynarray. It excludes self-reference (a layer isn't linked to itself).

Linked layers allow users to create groups — e.g., "Main curve" + "Turnout A" + "Turnout B" so that toggling one shows/hides all of them together.

---

### `IncrementLayerObjects()` / `DecrementLayerObjects()` — Update Object Counters

These adjust the `objCount` field when objects are moved onto/off a layer. The count is used in the UI to show whether a layer contains any track segments (`+` prefix vs `-`).

**`LayerSetCounts()`** walks all tracks and increments the counter for each track's assigned layer.

---

### `FindUnusedLayer(start)` — Find an Empty, Non-Frozen Layer

Used when adding a new layer or finding a slot to place objects on. Returns the index of the first layer with zero objects that isn't frozen; returns `-1` if none found (triggers error message).

---

## File I/O

### `ReadLayers(line)` — Parse Layer Definitions

Called for each line in the layout file starting with `LAYERS`. Handles multiple formats:
- **Legacy format** (< paramVersion 7): no layer support; returns TRUE silently
- **Simple format**: `"LAYERS <n> <visible> <frozen> <onMap> <color> <module> <useColor> <colorFlags> <buttonOff> \"name\""` — older files without tie data
- **Tie-data format** (paramVersion >= 9): includes min radius, max grade, and tie dimensions
- **Link lines**: `LINK <n> "<indices>"` to define linked layers
- **Settings lines**: `SET <n> "<settingsFile>"`

Backward compatibility is handled by checking the param version number and mapping old color indices through `oldColorMap[]`.

---

### `WriteLayers(f)` — Write Layer Definitions

For each layer that is "configured" (i.e., differs from default or has objects on it), writes a line:
```text
LAYERS <n> <visible> <frozen> <onMap> <colorRGB> <module> <useColorFlag> <colorFlags> <buttonOff> "<name>" <inherit> <scaleIdx> <minRad> <maxGrd> <tieLen> <tieWid> <tieSpc>
```

Followed by `LAYERS CURRENT <n>` and optional LINK/SET lines. Only configured layers are written — this keeps the file size down when using default settings.

---

### `IsLayerConfigured(n)` — Is This Layer Custom?

Returns TRUE if ANY of these differ from the "main" layer's defaults:
- Has a non-empty name (other than "Main")
- Visibility differs from default
- Frozen, onMap, module, or button_off flags are set
- Color differs from the calculated default color
- Inherits flag is FALSE
- Scale/gauge/min radius/max grade/tie data differ
- Has linked layers
- Has a settings file attached

---

## Dialog & Toolbar Integration

### `DoLayer()` — Open the "Manage Layers" Dialog

Creates (or shows) the param dialog. It:
- Builds a catalog of available `.xset` settings files from the working directory
- Populates the layer listbox and settings dropdown
- Shows the current layer's properties in the control fields
- Displays toolbar buttons for each layer

The toolbar button labels are dynamically generated by `FormatLayerName()` to include prefixes like `*`, `m`, `+`.

---

### Toolbar Button Creation (`InitLayers`)

At startup, toolbar buttons are created:
- **Button 0**: "Show/Hide Background" — toggles the background image overlay
- **Buttons 1–N**: Each layer has a button that calls `FlipLayer()` when clicked. The icon bitmap is drawn from bit arrays (stored in `show_layer_bmps[]`), which are generated on-the-fly by converting character-art bitmaps into raw byte arrays.

The number of buttons is read from preferences (`layer-button-count`) and defaults to 10 or fewer, up to a configurable maximum (`NUM_BUTTONS`).

---

## Design Decisions & Tradeoffs

### Why Layers Are Not "Tracks"

A layer can contain zero or many track objects. A track object has one `layer` field that points to *one* of the layers. This means:
- You can have a single-track layout with all segments on "Layer 1" and still use layers for organization (e.g., separating mainline from sidings)
- You can split a complex turnout across multiple layers if desired
- The layer system is purely organizational — it does not affect the track's geometry or behavior

### Why Two-Way Linking?

Layers are linked in both directions. If A is linked to B, then B is also implicitly linked to A (via the same link list). This is maintained by `PutLayerListArray()`, which uses a 1-based indexing scheme internally but stores values as-is (caller must account for the +1 offset when comparing against zero-based indices).

### Why Delete on Zero Objects?

Layers are never deleted if they contain objects. This prevents users from accidentally breaking their layout by deleting a layer that contains active track segments. The error message is clear: "Layer must not have any objects in it."

### Why Separate `IsLayerDefault` and `LayerSystemDefault`?

- **`LayerAllDefaults()`** resets the entire global array to defaults (used when starting fresh or after loading preferences)
- **`IsLayerDefault()`** checks a single layer against its default state — this is used when deciding whether to write that layer's definition to file. The distinction matters because a user might want to keep some layers "as-is" and not have them written out (saving space in the layout file).

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `FlipLayer(layerVP)` | Toggle visibility of one layer; updates toolbar button and linked layers | pointer to layer index |
| `SetCurrLayer(inx, name, op, ctx, arg)` | Set the current/active layer for new track objects | target layer index |
| `FormatLayerName(inx)` | Generate display label with prefix character | layer index |
| `IsLayerDefault(inx)` | Check whether a layer has been customized from defaults | layer index |
| `LayerAdd()` / `LayerDelete()` / `LayerDefault()` | Modify the set of defined layers | none (or current selection) |
| `ReadLayers(line)` / `WriteLayers(f)` | File I/O for layer definitions | line string or FILE* stream |
| `GetLayerLinkString()` / `PutLayerListArray()` | Serialize/deserialize linked layer list | layer index, buffer pointer |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Organize tracks and layout objects into named, colored, configurable layers with visibility controls, linking groups, and settings file associations |
| **Domain** | Layout organization: providing a hierarchical grouping mechanism similar to drawing layers in other CAD tools |
| **Key concept** | Layers are **metadata containers**, not physical track entities. A layer can hold zero or many objects; the same track object cannot belong to two layers simultaneously. The system supports "module" mode (all-or-nothing visibility) and linked groups for coordinated show/hide behavior. |
| **Main entry points** | `FlipLayer()` — toggle a layer's visibility from toolbar; `DoLayer()` / button callback — open the management dialog; `ReadLayers`/`WriteLayers` — file I/O |
