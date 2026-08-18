# doption.c — Preferences & Display Options Dialogs

## Overview

`doption.c` implements three user-configurable dialog windows:

1. **Display Options** (`displayPG`) — Rendering preferences, label toggles, tunnel drawing mode, etc.
2. **Command Options** (`cmdoptPG`) — Selection behavior, right-click menu customization.
3. **Preferences** (`prefPG`) — Units, angle system, measurement format, autosave/checkpoint settings, general application options.
4. **Color Dialog** (`colorPG`) — Per-category color assignments (track object colors, layer coloring, grid/marker/border colors).

The module is entirely UI/dialog-focused: it creates param dialogs with control elements and wires them up to global application state variables. Changes are propagated via the `DoChangeNotification()` mechanism (see `cundo.c`).

---

## Core Concepts

### Dialog Groups (`paramGroup_t`)

Each dialog is a `paramGroup` containing one or more `paramData` entries:
- **Radio buttons** for mutually exclusive choices (e.g., "English" vs. "Metric", tunnel drawing modes)
- **Dropdowns** (droplist controls) for format selection, car label mode
- **Toggles** for boolean flags (balloon help on/off, audio enabled/disabled)
- **Color selectors** (`PD_COLORLIST`) — each category gets its own color picker

The `PGO_PREFMISC` flag marks the group as a "miscellaneous preferences" group (persisted to user's config).

---

## Dialog 1: Display Options (`displayPG`)

### Controls

| Control | Type | Variable | Purpose |
|---------|------|----------|---------|
| Color Track / Color Draw | Radio button group | `colorTrack` / `colorDraw` | Choose whether tracks and drawn segments inherit color from the layer or from a global object draw order (track, bridge, roadbed) |
| Draw Tunnel | Radio | `drawTunnel` | Hide, dash pattern, or solid fill for tunnels |
| Draw EndPts | Radio | `drawEndPtV` | None / Turnouts only / All endpoints |
| Draw Unconnected EndPts | Radio | `drawUnconnectedEndPt` | Normal size / thick / exception (exceeded min length) |
| Draw Ties | Radio | `tieDrawMode` | None / outline only / solid fill |
| Draw Centers | Radio | `centerDrawMode` | Off / On — draws centerline circles on curved track segments |
| Two Rail Scale | Long integer | `twoRailScale` | Scaling factor for two-rail drawing mode |
| Map Scale | Float | `mapD.scale` | Scale used when drawing a map view of the layout |
| Don't Hide Cursor | Toggle | `dontHideCursor` | Prevents the system cursor from being hidden during interactive operations |
| Constrain Drawing Area | Toggle | `constrainMain` | Restricts panning to within room boundaries (if defined) |
| Live Map | Toggle | `liveMap` | Enable live map updates |
| Auto Pan | Toggle | `autoPan` | Automatically pan when dragging beyond visible area |
| Label Enable | Toggle group | `labelEnable` | Which label categories are enabled: Track Descriptions, Lengths, EndPt Elevations, Track Elevations, Cars |
| Label Scale | Long integer | `labelScale` | Scaling factor for text labels (affects font size) |
| Description Font Size | Long integer | `descriptionFontSize` | Points-size of label fonts |
| Car Labels | Dropdown | `carHotbarModeInx` | How car hot-bar items are labeled: Proto only / Proto+Manuf / Proto+PartNo / Proto+Manuf+PartNo+Item / Manuf/Proto combinations, etc. |
| Train Update Delay | Long integer | `trainPause` | Milliseconds between train position updates (for reducing CPU load) |
| Hide Trains in Tunnels | Toggle | `hideTrainsInTunnels` | Hides trains when their track segment is inside a tunnel |

---

## Dialog 2: Command Options (`cmdoptPG`)

### Controls

| Control | Type | Variable | Purpose |
|---------|------|----------|---------|
| Default Command | Radio | `preSelect` | Whether to show the Properties dialog or Select dialog on right-click (when no track is selected) |
| Hide Selection Window | Toggle | `hideSelectionWindow` *(conditional)* | Hides the selection popup window after a selection is made |
| Right Click | Radio | `rightClickMode` | Which action triggers which mode: Normal right-click = Command List OR Command Options dialog |
| Select Mode | Radio | `selectMode` | How multi-selection works: single item selected, or add to existing selection with Ctrl pressed |
| Deselect All on Empty Selection | Toggle | `selectZero` | Whether clicking in empty space deselects all previously selected items |

---

## Dialog 3: Preferences (`prefPG`)

### Controls

| Control | Type | Variable | Purpose |
|---------|------|----------|---------|
| Icon Size | Radio | `iconSize` | Small (16px), medium (24px), large (32px) — affects toolbar and icon sizes throughout the app |
| Angles | Radio | `angleSystem` | Polar coordinates or Cartesian (x/y) coordinates for angle input |
| Units | Radio | `units` | English or Metric system |
| Length Format | Dropdown | `distanceFormatInx` | Selects a distance number format from the unit-specific list: plain decimal, fractions with 8ths/16ths/32nds/64ths, feet+inches, meters, centimeters, etc. |
| Min Track Length | Float | `minLength` | Shortest segment that can be created (prevents creating infinitesimally short segments) |
| Connection Distance | Float | `connectDistance` | Tolerance for snapping two track endpoints together when they are within this distance of each other |
| Connection Angle | Float | `connectAngle` | Maximum angular difference allowed between two connected tracks before a joint is created (in degrees) |
| Turntable Angle | Float | `turntableAngle` | The angle at which a turntable turnout operates |
| Max Coupling Speed | Long integer | `maxCouplingSpeed` | Maximum speed (mph?) for automatic coupling of cars; prevents unrealistic high-speed couplings |
| Balloon Help | Toggle | `enableBalloonHelp` | Enables/disables the balloon help system on toolbars and menus |
| Enable Audio | Toggle | `enableAudio` | Plays sound effects when events occur (e.g., a train passing a signal) |
| Show FlexTrack in HotBar | Toggle | `showFlexTrack` | Shows the FlexTrack plugin button in the hot bar |
| Drag Distance | Long integer | `dragPixels` | Number of pixels you must drag to "lock" an object in place after moving it |
| Drag Timeout | Long integer | `dragTimeout` | Milliseconds before a dragged object is considered dropped and committed |
| Min Grid Spacing | Long integer | `minGridSpacing` | Smallest grid cell size allowed (prevents infinitely fine grids) |
| Checkpoint Frequency | Long integer | `checkPtInterval` | How often (in seconds?) the application writes an automatic checkpoint |
| Autosave Checkpoint Frequency | Long integer | `autosaveChkPoints` | Similar to checkpoint but specifically for autosaving; if 0, disabled |
| On Program Startup | Radio | `onStartup` | Load last layout file on startup OR start with a new blank layout |

---

## Dialog 4: Color (`colorPG`)

### Controls (Color Pickers)

Each of these is a color picker that assigns a color to a specific category:
- **Snap Grid** — color of the underlying snap grid overlay
- **Marker** — color of temporary markers / construction geometry
- **Border** — background border around the drawing canvas
- **Primary Axis** — major axis (X-axis) grid lines
- **Secondary Axis** — minor axis (Y-axis) grid lines
- **Normal Track** — standard track segments
- **Selected Track** — tracks that are currently selected by the user
- **Profile Path** — path shown when drawing a profile view of the layout
- **Exception Track** — track segments that violate constraints or rules
- **Track Ties** — the ties (sleepers) underneath the track
- **Bridge Base** — bridge structures
- **Roadbed** — the roadbed under the track

---

## Core Functions

### `GetChanges(pg)` — Compute a Bitmask of Changed Controls

Iterates over all controls in a param group and builds an integer bitmask where each bit represents whether that control's value changed since it was loaded. This is used to determine which actions need to be taken (e.g., redraw the canvas, update labels, write preferences).

```c
long GetChanges(paramGroup_p pg)
{
    long changes = 0;
    long changed;
    int inx;
    for (changed = ParamUpdate(pg), inx = 0, changes = 0; changed; changed >>= 1, inx++) {
        if (changed & 1) {
            changes |= VP2L(pg->paramPtr[inx].context);   // encode the control index into the bitmask
        }
    }
    return changes;
}
```

---

### `OptionDlgUpdate(pg, inx, valueP)` — Dialog Control Update Handler

Called whenever a control inside any of the three main dialogs is modified. The switch statement on `inx` handles each control individually:

- **Balloon help toggle** → calls `wEnableBalloonHelp()`
- **Label enable group** → toggles which label categories are drawn, reloads the "Car Labels" dropdown if needed
- **Units change** → calls `UpdatePrefD()`, which switches between English and metric format lists in the length-format dropdown
- **Distance format change** → calls `UpdateMeasureFmt()` to refresh all dimension-related controls (so their units display correctly)
- **Show FlexTrack toggle** → triggers a change notification so toolbar buttons can be updated
- **Checkpoint frequency** → toggles balloon tooltip text explaining what's being enabled/disabled

---

### `DisplayOk(junk)` — Display Dialog OK Handler

Collects the bitmask of changed controls (via `GetChanges`), hides the dialog window, and calls `DoChangeNotification(changes)` to trigger all necessary redraws and state updates.

---

### `LoadDstFmtList()` / `UpdatePrefD()` / `UpdateMeasureFmt()` — Unit System Handling

When the user switches between English and Metric units:
1. The listbox of available number formats is cleared and repopulated with entries appropriate to the selected unit system (English: feet/inches/fractions; Metric: mm, cm, m)
2. All dimension-related controls (length, connection distance, min length, etc.) are reloaded so they show values in the correct units

---

### `PrefOk(junk)` — Preferences Dialog OK Handler

Collects changes and hides the dialog. It also performs **value validation**: if any numeric field is below its minimum or above its maximum, it resets to a valid default and shows an error message balloon explaining why (e.g., "Connection angle too small" or "Autosave frequency cannot be zero"). It saves the audio enable/disable preference to the system preferences file.

---

### `DoPref(junk)` — Preferences Dialog Entry Point

Creates the dialog window if it hasn't been created yet, loads all controls with their current values from global variables (or saved preferences), and shows the window.

---

## Data Structures

### `dstFmts_t` — Distance Format Definition

```c
typedef struct {
    char * name;       // Human-readable label shown in the dropdown (translatable via _)
    long  fmt;         // Bitmask combining DISTFMT_FMT_* flags that controls how numbers are formatted
} dstFmts_t;
```

Two arrays exist: `englishDstFmts[]` and `metricDstFmts[]`, each containing format strings like `"999.99"`, `"999' 11.99""`, etc., along with a bitmask that tells the number formatter how to interpret the value.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `GetChanges(pg)` | Compute a change bitmask for a param group | param group pointer |
| `DisplayOk(junk)` | Hide display dialog and trigger notifications | unused junk pointer |
| `OptionDlgUpdate(pg, inx, valueP)` | Handle an individual control change within any option dialog | param group, control index, new value pointer |
| `LoadDstFmtList()` | Repopulate the distance format dropdown with unit-appropriate entries | none |
| `UpdatePrefD()` | Called when units radio button changes; switches between English and metric formats | none |
| `UpdateMeasureFmt()` | Reloads all dimension-related controls after a format change | none |
| `PrefOk(junk)` | Hide preferences dialog; validate numeric ranges; save audio preference | unused junk pointer |
| `DoPref(junk)` | Create and show the Preferences dialog window | unused junk pointer |
| `ColorOk(junk)` | Hide color dialog and trigger notifications | unused junk pointer |
| `DoColor(junk)` | Create and show the Color dialog | unused junk pointer |

---

## Design Decisions & Tradeoffs

### Why a Single "Distance Format" Dropdown?

Instead of having separate dropdowns for each dimension field (length, connection distance, min length), there is one global dropdown. Changing it updates all dimension fields simultaneously. This prevents inconsistency: you can't have feet on the connection distance and meters on the min length.

### Why Separate Display and Preferences Dialogs?

The **Display** dialog contains settings that affect visual rendering (colors, labels, drawing styles). The **Preferences** dialog contains general application behavior (units, units of measure, audio, autosave). Separating them reduces cognitive load: users who only care about how things look can ignore the preferences dialog entirely.

### Why `GetChanges` Returns a Bitmask?

The bitmask encodes which controls changed as powers of two. A value like `0b101010` means controls at indices 0, 2, and 4 (from the least-significant bit) all changed. This allows multiple independent handlers to respond to their specific control without needing a shared global flag.

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide user-configurable dialogs for display rendering preferences, command behavior, general application settings (units, measurement format), and color scheme selection |
| **Domain** | User interface / preferences: storing per-user configuration in a persistent way and applying it to the running application state |
| **Key concept** | The `paramGroup` abstraction encapsulates an entire dialog window as a collection of named controls. Each control has a type (radio, toggle, dropdown, color picker), a variable pointer, a label string, and options flags. The param library handles serialization to disk and persistence across restarts. |
| **Main entry points** | `DisplayInit()` — register display dialog; `CmdoptInit()` — register command options; `PrefInit()` — register preferences; `ColorInit()` — register color picker |
