# **UX Reference Documentation for XTrackCAD**

## 📋 Table of Contents
1. [Main Window Overview](#main-window-overview)
2. [Menu System](#menu-system)
3. [Toolbars and Panels](#toolbars-and-panels)
4. [Drawing Elements](#drawing-elements)
5. [Status Bar Controls](#status-bar-controls)
6. [Dialog Boxes and Modals](#dialog-boxes-and-modals)
7. [Keyboard Shortcuts Reference](#keyboard-shortcuts-reference)
8. [Mouse Interaction Patterns](#mouse-interaction-patterns)

---

## Main Window Overview

### Primary Windows

| Window | Purpose | Associated Code |
|--------|---------|-----------------|
| **Main Canvas** (`mainW`) | Primary drawing area for track layouts | `wlib/gtklib/` - canvas rendering |
| **Hot Bar** (`cmdHotBar`) | Quick access to turnouts, structures, flex-track | `cornu/spiroentrypoints.c` - object instantiation |
| **Map Window** (`cmdMap`) | Alternative viewport for navigation | `lib/` - map view logic |

---

## Menu System

### File Menu (`fileM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Exit | Close application | Application shutdown |
| New | Create new layout | `cmdLayout()` dialog initialization |
| Open | Load existing `.xtc` or `.xtce` archive | File I/O parser |
| Save | Save current layout as `.xtc` | Persistence layer |
| Save As | Save with different name/format | Archive creation (`.xtce`) |
| Revert | Discard unsaved changes | Memory reset to last save point |

### Edit Menu (`editM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Undo | Revert last command | Command history stack management |
| Redo | Redo previous undo operation | Command history replay |
| Cut | Remove selected objects to clipboard | Clipboard API integration |
| Copy | Duplicate selected objects | Object serialization |
| Paste | Insert from clipboard | Deserialization and placement |
| Delete | Remove selected objects | Object deletion with cascade checks |

### View Menu (`viewM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Enable Snap Grid | Toggle grid snapping | `bezctx_intf.h` - snap state toggle |
| Show Snap-Grid | Display/hide grid overlay | Canvas rendering layer |
| Magnetic Snap On/Off | Enable auto-alignment | Distance calculation routines |
| Redraw | Refresh canvas display | OpenGL/Direct2D redraw trigger |
| Pan/Zoom | Adjust viewport | Transform matrix manipulation |

### Change Menu (`changeM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Properties | Enable object property editing | `cmdDescribe()` - read/write fields |
| Select | Toggle object selection state | Selection set management API |
| Move | Translate selected objects | Position delta application |
| Rotate | Spin selected objects around pivot | Rotation matrix computation |
| Flip | Mirror selected objects | Coordinate system reflection |

### Draw Menu (`drawM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Straight Objects | Line, dimension line, benchwork | `cmdDrawStraight()` - primitive geometry |
| Curved Lines | Arc and Bezier curves | Cornu/Bézier curve generation |
| Circle Lines | Empty/filled circular shapes | Circular arc construction |
| Shapes | Box, polygon, polyline creation | Polygon vertex management |

### Options Menu (`optionM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Colors | Set object drawing colors | `cmdRgbcolor()` - color palette manager |
| Display | Configure visual settings | `cmdDisplay()` - renderer options |
| Easement | Adjust transition curve parameters | Cornu library configuration |
| Layout | Define room dimensions and scale | Project metadata storage |

### Help Menu (`helpM`)

| Command | Description | Code Connection |
|---------|-------------|-----------------|
| Help | Open documentation viewer | Built-in help browser |
| Demo Mode | Run recorded macro sequences | Macro playback engine |

---

## Toolbars and Panels

### Toolbar Configuration

The toolbar system is managed through the **Toolbar Options Dialog** (`toolbarOpts`):

| Button Group | Location | Toggleable Items |
|--------------|----------|------------------|
| Track Drawing | Primary section | Straight, Curve, Circle, Shape buttons |
| Object Manipulation | Secondary section | Select, Move, Rotate, Properties |
| Grid Controls | Tertiary section | Snap Grid, Magnetic Snap toggles |
| Layer Buttons | Right-side panel | 1-99 layer visibility controls |

### Hot Bar (`cmdHotBar`)

The hot bar displays a scrollable list of available construction primitives:

- **Flex-track** (diagonal indicator) - For quick curve creation  
- **Turnout library items** - From loaded parameter files  
- **Structure definitions** - Bridges, stations, buildings  
- **Custom objects** - User-defined turnouts and structures  

**Code Connection**: `cornu/spiroentrypoints.c` manages the Hot Bar's object queue and rendering.

### Layer Panel (`cmdLayer`)

The layer system provides 99 customizable layers with these properties:

| Property | Default Value | Code Location |
|----------|---------------|---------------|
| Name | "Default" | `xtrkcad.ini` persistence |
| Color | System theme color | Per-layer override capability |
| Visible | true | Layer state bitmask |
| Frozen | false | Object modification prevention flag |
| Module | false | Group transform enablement |

**Code Connection**: `cmdLayer()` dialog manages the layer data structure with persistent storage in `.xset` files.

---

## Drawing Elements

### Track Objects

#### Fixed Track (Flex-track)
- **Structure**: Series of Straight, Curve, Joint segments  
- **Code**: `cornu/bezctx.c` - Segment concatenation and continuity checks  
- **UI Controls**: Modify radius, adjust angle at endpoints  

#### Cornu Track
- **Structure**: Continuous curve with variable radius  
- **Code**: `cornu/spiro.c` - Mathematical easing functions  
- **UI Features**: Pin management via left-click → drag → delete workflow  

#### Bezier Curve
- **Structure**: Control point manipulation interface  
- **Code**: `wlib/dynstring/` - Quadratic curve interpolation  
- **UI Controls**: Drag control points, add/remove pins  

---

### Draw Objects (Scenery)

| Object Type | Code Module | UI Modification Features |
|-------------|-------------|-------------------------|
| Straight Line | `cmdDrawStraight()` | Length, angle fields with pivot controls |
| Curved Line | `cmdDrawCurveEndPt()` | Radius slider, arc angle input box |
| Circle | `cmdDrawCircleCenter()` | Diameter control, fill toggle |
| Box | `cmdDrawBox()` | Width/height dimensions, filled variant |
| Polygon | `cmdDrawPolygon()` | Vertex edit mode (points/origin modes) |
| Text | `cmdText()` | Font size dropdown, color picker |

### Track Properties Dialog (`cmdDescribe`)

The properties dialog provides these fields based on object type:

#### Common Fields
- **Pivot** - Fix start/middle/end point during edits  
- **Rotation Origin** - Absolute (0,0) vs Relative (blue cross indicator)  
- **Layer/Color** - Selection and color override controls  

#### Track-Specific Fields
- **End Pt 1 / End Pt 2** - Endpoint coordinates with elevation display  
- **Grade** - Computed slope between endpoints  
- **Radius** - Curvature parameter (read-only when connected)  

---

## Status Bar Controls

### Command-Specific Inputs

| Input Field | Activated By | Functionality |
|-------------|--------------|---------------|
| Line Width | Draw command active | Scale-dependent line thickness in pixels or scaled inches |
| Color Picker | Any draw operation | RGB color selection for current object type |
| Dimension Values | Modify/Properties mode | Precise length, angle, radius entry with Tab/Enter confirmation |

### Status Bar Actions

- **Font Size Dropdown** - Controls text rendering scale  
- **Boxed Toggle** - Adds rectangular border around active text objects  
- **Command Name Display** - Shows currently executing command via balloon help  

---

## Dialog Boxes and Modals

### Parameter Files Manager (`cmdPrmfile`)
```
├── File List (color-coded by compatibility)
│   ├── Green star = Perfect fit for current scale/gauge
│   └── Yellow dot = Compatible with minor adjustments needed
├── Load/Unload controls
├── Favorite marking capability
└── Browse system library dialog access
```

### Custom Management (`cmdCustmgm`)
- **Edit Button** - Opens appropriate designer for selected item  
- **Delete/Shelve** - Removes or archives custom definitions  
- **Move To File** - Export to parameter file format  

### Car Inventory (`cmdCarinv`)
| Control | Functionality | Code Connection |
|---------|---------------|-----------------|
| Sort Drop Down | Reorder columns by property | Table model sorting API |
| Find Button | Center on car location | Viewport transform update |
| Add/Edit/Delete | Create/modify inventory entries | Database CRUD operations |

### Car Part Designer (`cmdCarpart`)
- **Prototype Import** - Convert drawn shapes to prototype definition  
- **Flip** - Horizontal mirror for locomotive orientation  
- **Customize Mode** - Override road names, numbers, colors  

---

## Keyboard Shortcuts Reference

### Navigation & Selection

| Shortcut | Action | Equivalent UI Element |
|----------|--------|----------------------|
| `Esc` | Deselect all objects | Edit → Deselect All |
| `Shift+Left-Click` | Add connected tracks to selection | Change → Select (with connection) |
| `Ctrl+C/V/X/Z/R` | Clipboard operations | Edit menu commands |

### Drawing Commands

| Shortcut | Action | Context Required |
|----------|--------|------------------|
| `1-9, 0` | Quick zoom levels 1:1 to 1:9.0 | Any mode |
| `'e'` | Fit entire layout in viewport | Selection required |
| `'s'` | Frame selected objects | Selection required |
| `'c'` | Center on cursor position | Any mode |

### Modification Commands

| Shortcut | Action | Mode Required |
|----------|--------|---------------|
| `Ctrl+Left-Drag` | Rotate around origin | Select or Properties mode |
| `Shift+Ctrl+Arrow` | Micro-move (1-pixel increments) | Move command active |
| `Space/Enter` | Confirm dimension changes | Sticky modification enabled |

---

## Mouse Interaction Patterns

### Click Sequences by Command Type

#### Track Creation Flow
```
Left-Click (start point) → Left-Drag (end point) → Release (complete)
```
**Variations**:
- Hold `Ctrl` to snap angles to 90° increments  
- Use Magnetic Snap (`Alt`) to disable auto-alignment  

#### Object Selection Patterns
| Pattern | Effect | Code Handler |
|---------|--------|--------------|
| Single Left-Click | Select/deselect individual object | Selection toggle API |
| Shift+Left-Click (connected) | Select entire connected component | Flood fill selection algorithm |
| Ctrl+Shift+Right-Click | Context menu with rotation options | Right-click event dispatcher |

#### Dimension Entry Workflow
```
Left-Click (position anchor) → Type values → Tab/Enter (apply changes)
```
**Sticky Mode**: `Shift+Tab` discards current entry for new operation

### Drag-Based Operations

| Drag Pattern | Result | Code Path |
|--------------|--------|-----------|
| Left-Drag on endpoint | Extend/trim track segment | Length delta application |
| Right-Drag over area | Select/deselect all objects in region | Hit-testing within viewport bounds |
| Ctrl+Left-Drag (over track) | Create tangent curve or circle | Geometric construction algorithms |

---

## Code Connection Mapping

### Menu → Command → Module Chain

```
Edit → Undo      ──► cmdUndo()        ──► bezctx_undo_stack.c
View → Snap Grid  ──► EnableSnapGrid() ──► bezctx_intf.h snap flag
Change → Move     ──► cmdMove()       ──► wlib/geometry/move.c
Draw → Straight   ──► cmdDrawStraight() ──► cornu/draw_line.c
Options→ Easement  ──► cmdEasement()   ──► cornu/easement_params.c
```

### Toolbar Button → Function Mapping

| Button Icon | Command | Module Entry Point |
|-------------|---------|-------------------|
| Straight line icon | `cmdDrawStraight()` | `draw_straight_ui.h` |
| Curve icon with radius arrow | `cmdDrawCurveEndPt()` | `cornu/curve_creation.c` |
| Circle with center point | `cmdDrawCircleCenter()` | `wlib/circle_geometry.c` |
| Box outline | `cmdDrawBox()` | `draw_shapes_ui.h` |

---

## Summary of Key UX Components

### Primary Interaction Surface
- **Main Canvas** - Full-screen drawing area with transform controls  
- **Hot Bar** - Scrollable object library for track and structure placement  
- **Layer Strip** - 99-layer visibility toggles (maximized via toolbar options)

### Core Drawing Workflow
1. Select command from toolbar or menu  
2. Left-click to anchor first point  
3. Drag to define second point/shape  
4. Release mouse button → object commits to layout  

### Data Flow Architecture
```
User Input (mouse/keyboard)
    ↓
Command Dispatcher (view_menu.c)
    ↓
UI Widget Handler (wlib/gtklib/)
    ↓
Geometry Calculator (cornu/* modules)
    ↓
Canvas Renderer (OpenGL/Direct2D backend)
```

### Persistence Points
- **Undo Stack** - Limited to last 10 commands per session  
- **Checkpoints** - Configurable frequency in AutoSave preferences  
- **Archives (.xtce)** - Package layout with background image  

---

This UX reference documentation covers the complete user interface of XTrackCAD, including all menu systems, toolbar configurations, drawing elements, and their underlying code modules. For detailed command behavior descriptions, refer to the individual `.but` documentation files in `app/doc/`.