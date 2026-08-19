# acclkeys.h — Accelerator Key Definitions

## Overview

`acclkeys.h` defines the **accelerator key** macros used throughout XTrkCAD's command system. Each macro maps a combination of modifier keys and character keys to a specific command.

The naming convention encodes the required modifiers:
- `WCTL` = Windows/Command key (Windows) or Command/Super key (macOS)
- `WALT` = Alt / Option key
- `WSHIFT` = Shift key
- `WCTRL` = Ctrl key (sometimes aliased to WCTL)

---

## Command Groupings

The keys are organized into logical groups:

| Group | Prefix | Purpose |
|-------|--------|---------|
| Drawing commands | `ACCL_` + command name | Create/edit track geometry, turnouts, etc. |
| File operations | `ACCL_NEW`, `ACCL_OPEN`, `ACCL_SAVE*`, `ACCL_EXPORT*`, `ACCL_IMPORT*` | File management |
| Edit operations | `ACCL_UNDO`, `ACCL_REDO`, `ACCL_COPY`, `ACCL_CUT`, `ACCL_PASTE`, etc. | Undo/redo, clipboard, selection |
| View commands | `ACCL_REDRAW`, `ACCL_ZOOMIN`, `ACCL_SNAP*` | Canvas manipulation |
| Option dialogs | `ACCL_LAYOUTW`, `ACCL_DISPLAYW`, `ACCL_PREFERENCES`, etc. | Open settings dialog for a category |

---

## Full Key Table

### Drawing Commands

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_DESCRIBE` | WCTL+? | Describe/inspect selected objects |
| `ACCL_SELECT` | WCTL+e | Select objects |
| `ACCL_PAN` | WCTL+/ | Pan the canvas |
| `ACCL_STRAIGHT` | WCTL+g | Draw a straight track segment |
| `ACCL_CURVE1` | WCTL+4 | Draw curve #1 (short radius) |
| `ACCL_CURVE2` | WCTL+5 | Draw curve #2 |
| `ACCL_CURVE3` | WCTL+6 | Draw curve #3 |
| `ACCL_CURVE4` | WCTL+7 | Draw curve #4 (long radius) |
| `ACCL_CIRCLE1` | WCTL+8 | Draw circle arc #1 |
| `ACCL_CIRCLE2` | WCTL+9 | Draw circle arc #2 |
| `ACCL_CIRCLE3` | WCTL+0 | Draw circle arc #3 |
| `ACCL_BEZIER` | — (unused) | Bezier curve draw command |
| `ACCL_CORNU` | — (unused) | Cornu curve draw command |
| `ACCL_CONVERTTO` | — (unused) | Convert track to another type |
| `ACCL_CONVERTFR` | — (unused) | Convert French turnout |
| `ACCL_TURNOUT` | WCTL+t | Draw a turnout switch |
| `ACCL_TURNTABLE` | WCTL+Shift+n | Place a turntable |
| `ACCL_PARALLEL` | WCTL+Shift+p | Create parallel track |
| `ACCL_MOVE` | WCTL+Shift+m | Move selected geometry |
| `ACCL_ROTATE` | WCTL+Shift+r | Rotate selected geometry |
| `ACCL_FLIP` | — (unused) | Flip/reflect geometry |
| `ACCL_MOVEDESC` | WCTL+Shift+z | Move with description field |
| `ACCL_MODIFY` | WCTL+m | Modify selected object |
| `ACCL_JOIN` | WCTL+j | Join endpoints |
| `ACCL_CONNECT` | WCTL+Shift+j | Connect two separate tracks |
| `ACCL_HELIX` | WCTL+Shift+h | Draw helix (3D track) |
| `ACCL_SPLIT` | WCTL+Shift+s | Split a track at selection point |
| `ACCL_SPLITDRAW` | — (unused) | Split drawing layer |
| `ACCL_TRIMDRAW` | — (unused) | Trim drawing elements |
| `ACCL_ELEVATION` | WCTL+Shift+e | Elevation editing command |
| `ACCL_PROFILE` | WCTL+Shift+f | Profile view command |
| `ACCL_DELETE` | WCTL+d | Delete selected objects |
| `ACCL_TUNNEL` | WCTL+Shift+t | Draw tunnel segment |
| `ACCL_BRIDGE` | — (unused) | Draw bridge segment |
| `ACCL_ROADBED` | — (unused) | Add roadbed to track |
| `ACCL_TIES` | — (unused) | Add ties/railroad sleepers |

### Drawing Tools (Freehand / Sketching)

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_DRAWLINE` | WCTL+Shift+1 | Draw a line segment |
| `ACCL_DRAWDIMLINE` | WCTL+Shift+d | Draw dimension line |
| `ACCL_DRAWBENCH` | WCTL+b | Draw bench (level ground) segment |
| `ACCL_DRAWTBLEDGE` | WCTL+Shift+3 | Draw tbleading edge geometry |
| `ACCL_DRAWCURVE1` | WCTL+Shift+4 | Draw curve #1 tool |
| `ACCL_DRAWCURVE2` | WCTL+Shift+5 | Draw curve #2 tool |
| `ACCL_DRAWCURVE3` | WCTL+Shift+6 | Draw curve #3 tool |
| `ACCL_DRAWCURVE4` | WCTL+Shift+7 | Draw curve #4 tool |
| `ACCL_DRAWCIRCLE1` | WCTL+Shift+8 | Draw circle arc #1 |
| `ACCL_DRAWCIRCLE2` | WCTL+Shift+9 | Draw circle arc #2 |
| `ACCL_DRAWCIRCLE3` | WCTL+Shift+0 | Draw circle arc #3 |
| `ACCL_DRAWFILLCIRCLE1` | WALT+WCTL+8 | Draw filled circle (color 1) |
| `ACCL_DRAWFILLCIRCLE2` | WALT+WCTL+9 | Draw filled circle (color 2) |
| `ACCL_DRAWFILLCIRCLE3` | WALT+WCTL+0 | Draw filled circle (color 3) |
| `ACCL_DRAWBEZLINE` | — (unused) | Draw Bezier curve line |
| `ACCL_DRAWBOX` | WCTL+Shift+[ | Draw a rectangular box |
| `ACCL_DRAWFILLBOX` | WALT+WCTL+[ | Draw filled rectangle |
| `ACCL_DRAWPOLYLINE` | — (unused) | Draw polyline chain |
| `ACCL_DRAWPOLYGON` | WALT+WCTL+2 | Draw closed polygon |
| `ACCL_DRAWPOLY` | — (unused) | Draw arbitrary polygon |
| `ACCL_DRAWFILLPOLYGON` | WCTL+Shift+2 | Draw filled polygon |

### Note & Structure Commands

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_NOTE` | WALT+WCTL+n | Create a note (text annotation) |
| `ACCL_STRUCTURE` | WCTL+Shift+c | Toggle structure display |
| `ACCL_ABOVE` | WCTL+Shift+b | Show objects above current layer |
| `ACCL_BELOW` | WCTL+Shift+w | Show objects below current layer |

### File Operations

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_NEW` | WCTL+n | Create a new document |
| `ACCL_OPEN` | WCTL+o | Open an existing track plan file |
| `ACCL_SAVE` | WCTL+s | Save current layout to disk |
| `ACCL_SAVEAS` | WCTL+a | Save as (with dialog) |
| `ACCL_REVERT` | — (unused) | Revert to last saved state |
| `ACCL_PARAMFILES` | WALT+WCTL+s | Manage parameter files (.xtc parameters) |
| `ACCL_PRICELIST` | WALT+WCTL+q | Open price list dialog |
| `ACCL_PRINT` | WCTL+p | Print current layout |
| `ACCL_PRINTSETUP` | — (unused) | Print setup dialog |
| `ACCL_PRINTBM` | WCTL+Shift+q | Print bitmap / screenshot |
| `ACCL_PARTSLIST` | WALT+WCTL+l | Open parts list dialog |
| `ACCL_NOTES` | WALT+WCTL+t | Manage notes (text annotations) |
| `ACCL_REGISTER` | — (unused) | Register layout with a server |

### Edit Operations

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_UNDO` | WCTL+z | Undo last action |
| `ACCL_REDO` | WCTL+r | Redo undone action |
| `ACCL_COPY` | WCTL+c | Copy selected objects to clipboard |
| `ACCL_CUT` | WCTL+x | Cut (copy + delete) |
| `ACCL_PASTE` | WCTL+v | Paste from clipboard |
| `ACCL_CLONE` | — (unused) | Clone (copy in place) |
| `ACCL_SELECTALL` | WCTL+Shift+a | Select all objects |
| `ACCL_DESELECTALL` | — (unused) | Deselect all objects |
| `ACCL_THIN` | WCTL+1 | Set stroke to thin |
| `ACCL_MEDIUM` | WCTL+2 | Set stroke to medium |
| `ACCL_THICK` | WCTL+3 | Set stroke to thick |
| `ACCL_EXPORT` | WALT+WCTL+x | Export selected objects |
| `ACCL_IMPORT` | WALT+WCTL+i | Import track plan file |
| `ACCL_IMPORT_MOD` | — (unused) | Import as module layer |
| `ACCL_EXPORTDXF` | — (unused) | Export to DXF format |
| `ACCL_EXPORTSVG` | — (unused) | Export to SVG format |
| `ACCL_LOOSEN` | WCTL+Shift+k | Loosen track constraints |
| `ACCL_GROUP` | WCTL+Shift+g | Group selected objects |
| `ACCL_UNGROUP` | WCTL+Shift+u | Ungroup grouped objects |
| `ACCL_CUSTMGM` | WALT+WCTL+u | Custom management dialog |
| `ACCL_CONTMGM` | WALT+WCTL+c | Container management dialog |
| `ACCL_CARINV` | WALT+WCTL+v | Car inventory dialog |
| `ACCL_LAYERS` | WALT+WCTL+y | Layer manager dialog |
| `ACCL_SETCURLAYER` | — (unused) | Set current layer |
| `ACCL_MOVCURLAYER` | — (unused) | Move current layer index |
| `ACCL_CLRELEV` | — (unused) | Clear elevation data |
| `ACCL_CHGELEV` | — (unused) | Change elevation mode |

### View Commands

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_REDRAW` | WCTL+l | Redraw current view |
| `ACCL_REDRAWALL` | WCTL+Shift+l | Force full redraw of entire canvas |
| `ACCL_ZOOMIN` | WCTL++ | Zoom in on center |
| `ACCL_ZOOMOUT` | WCTL+- | Zoom out from center |
| `ACCL_SNAPSHOW` | WCTL+] | Toggle snap preview display |
| `ACCL_SNAPENABLE` | WCTL+[ | Enable/disable snapping to geometry |
| `ACCL_MAPSHOW` | WCTL+Space | Toggle map overlay display |

### Option Dialogs (Open Settings)

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_LAYOUTW` | WALT+WCTL+a | Layout settings dialog |
| `ACCL_DISPLAYW` | WALT+WCTL+d | Display/rendering settings |
| `ACCL_CMDOPTW` | WALT+WCTL+m | Command options dialog |
| `ACCL_EASEW` | WALT+WCTL+e | Ease/transition settings |
| `ACCL_FONTW` | WALT+WCTL+f | Font settings |
| `ACCL_GRIDW` | WALT+WCTL+g | Grid / snap settings |
| `ACCL_STICKY` | WALT+WCTL+k | Sticky key mode toggle |
| `ACCL_PREFERENCES` | WALT+WCTL+p | General preferences dialog |
| `ACCL_COLORW` | WALT+WCTL+c | Color scheme picker |

### Macros

| Macro | Shortcut | Description |
|-------|----------|-------------|
| `ACCL_RECORD` | WALT+WCTL+r | Start recording a macro |
| `ACCL_PLAYBACK` | WALT+WCTL+b | Play back recorded macro |

### Unused / Deprecated Keys

| Macro | Shortcut | Status |
|-------|----------|--------|
| `ACCL_FLIP` | — | Not assigned |
| `ACCL_SPLITDRAW` | — | Not assigned |
| `ACCL_TRIMDRAW` | — | Not assigned |
| `ACCL_BRIDGE` | 0 | Not assigned |
| `ACCL_TIES` | 0 | Not assigned |
| `ACCL_BLOCK1/2/3` | 0 | Not assigned (reserved for blocks) |
| `ACCL_SWITCHMOTOR1/2/3` | 0 | Not assigned |
| `ACCL_EXPORTDXF` | — | Not assigned |
| `ACCL_EXPORTSVG` | — | Not assigned |
| `ACCL_CLRELEV` | — | Not assigned |
| `ACCL_CHGELEV` | — | Not assigned |
| `ACCL_SETCURLAYER` | — | Not assigned |
| `ACCL_MOVCURLAYER` | — | Not assigned |
| `ACCL_DESELECTALL` | — | Not assigned |

---

## Design Notes

- **Modifier prefixes** (`WCTL`, `WALT`, `WSHIFT`) are defined elsewhere (likely in a windowing or input module). They ensure keys work correctly on different platforms (Windows uses Ctrl+K, macOS uses Cmd+K).
- The header file includes a comment: `use 'sort +2 acclkeys.h' to check usage`. This implies that the keys can be sorted alphabetically by macro name and then reviewed for gaps or reassignments.
- Many keys are marked as `(unused)` — these are likely reserved for future features or legacy compatibility.
