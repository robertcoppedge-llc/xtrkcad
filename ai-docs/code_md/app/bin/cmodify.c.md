# cmodify.c — Track Modify Command (Extend/Alter Tracks, Rulers, Bezier/Cornu Modification)

## Overview

`cmodify.c` implements the **Modify** command in XTrkCad. It allows users to:
- Extend a track with straight or curved segments (with optional easement curves).
- Alter an existing track by changing its length.
- Modify rulers, Bezier tracks, Cornu spiral tracks, and DRAW objects.
- Add flexible ("flextrack") extensions using Ctrl+Right-click on a turnout end.

The core function is `CmdModify(action, pos)` which dispatches on mouse actions (`C_START`, `C_DOWN`, `C_MOVE`, `C_UP`, `C_OK`, etc.) to perform the appropriate modification operation.

## File Location

```
app/bin/cmodify.c
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `cjoin.h` | Track joining utilities (`ConnectTracks`, `JoinTracks`) |
| `ccurve.h` | Curve drawing helpers (`NewCurvedTrack`, `PlotCurve`) |
| `cbezier.c` | Bezier curve track modification |
| `ccornu.h` | Cornu spiral track handling |
| `cstraigh.h` | Straight track creation/adjustment |
| `cundo.h` | Undo stack management (`UndoStart`, `UndoEnd`, `UndoModify`) |
| `fileio.h` | I/O utilities |
| `param.h` | Parameter dialog framework (used for easement length input) |
| `track.h` | Track data structures and accessor functions |
| `drawgeom.h` | Geometric drawing primitives (`DrawEndPt`, `DrawArrowHeads`) |
| `common.h` | Common utilities (`MyMalloc`, `InfoMessage`, etc.) |
| `layout.h` | Layout state (pan center, zoom extents) |
| `cselect.h` | Object selection helpers |
| `common-ui.h` | UI widget types (`wMenu_p`, `wAction_t`) |
| `draw.h` | Drawing context (`mainD.d`, drawing functions) |

## Main External Variable

```c
EXPORT wIndex_t modifyCmdInx;   /* index of "Modify" in the main command menu */
```

## Enums Used

### `curveType_e`
Represents the type of curve being extended:

| Value | Name | Description |
|-------|------|-------------|
| `curveTypeStraight` | Straight | A straight segment extension |
| `curveTypeNone` | None / Back | No valid curve selected (cancel) |
| `curveTypeCurve` | Curve | A circular arc extension |

## Structs & Data Members

### `Dex` — Global modification state struct

```c
static struct {
    track_p Trk;                /* currently modified track */
    trackParams_t params;       /* extended track parameters (type, arcR, len) */
    coOrd pos00;                /* original end point before drag started */
    coOrd pos00x;               /* temporary scratch coordinate */
    coOrd pos01;                /* previous position during drag (for animation) */
    ANGLE_T angle;              /* current tangent angle at the extend point */
    curveData_t curveData;      /* computed curve data from endpoint + radius/angle */
    easementData_t jointD;      /* easement joint geometry (d0, d1, x, flip, etc.) */
    DIST_T r1;                  /* transition arc radius for composite curves */
    BOOL_T valid;               /* is the current extended segment valid? */
    BOOL_T first;               /* flag: are we at the start of a drag operation? */
} Dex;
```

### `anchors_da` (dynArr)
A dynamic array used to draw temporary anchor arcs when dragging endpoints or radius anchors. Each element represents a small arc centered at an endpoint with radius equal to half the track gauge, drawn in blue (`wDrawColorBlue`). Used for visual feedback during interactive modification.

## Core Functions

### `CmdModify(wAction_t action, coOrd pos)`
The main dispatch function handling all modify operations. Dispatches on mouse events:

| Action | Behavior |
|--------|----------|
| `C_START` | Initialize the command — show message "Select a track to modify..." |
| `C_DOWN / C_LDOUBLE` | Left-click or double-click on a track endpoint (or ruler/protractor) |
| `C_RDOWN` / `C_MOVE` (with Ctrl held) | Extend track with straight or curved extension; drag the end point |
| `C_RUP` | Accept the extended segment and commit to undo stack |
| `C_MOVE` (no Ctrl, on a DRAW object) | Drag an endpoint of a DRAW object |
| `C_UP` | Commit modification to Undo stack, redraw track |
| `C_TEXT` / `C_OK` | Confirm/apply changes for Bezier/Cornu/DRAW modifications |
| `C_TERMINATE` | Terminate the modify session (e.g., after Ctrl+RDOWN or "Enter") |

### Internal Sub-functions

#### `ModifyBezier(wAction_t action, coOrd pos)`
Delegates to `cbezier.c`'s `CmdBezModify`. Handles control point editing for Bezier track segments. Sets global flags (`modifyBezierMode`) and handles mouse events including drag, confirm (Enter), cancel, and termination.

#### `ModifyCornu(wAction_t action, coOrd pos)`
Delegates to `ccornu.c`'s `CmdCornuModify`. Handles editing Cornu spiral endpoints. Supports dragging endpoints and terminating with Enter or clicking outside.

#### `ModifyDraw(wAction_t action, coOrd pos)`
Handles point-based modification of DRAW objects (free-form drawn lines). Allows adding/removing points by left-clicking to place new points. Terminates on Enter/Shift+Enter.

## Modify Command Modes

The command supports several interaction modes distinguished by flags and actions:

| Mode | Flag / Key | Trigger |
|------|------------|----------|
| **Track modify** (trim) | Left-click on track endpoint | Extends or truncates the selected track |
| **Flextrack extend** | Ctrl + Right-click on turnout end | Adds a straight extension with an easement curve |
| **Ruler mode** | Shift + Left-click on ruler | Modifies the ruler's scale/tick marks |
| **Protractor mode** | Left-click on protractor arc | Adjusts angle markings |
| **Bezier modify** | Query returns `Q_CAN_MODIFY_CONTROL_POINTS` | Edit Bezier control points |
| **Cornu modify** | Query returns `Q_IS_CORNU` | Edit Cornu spiral parameters |

## Track Queries Used (from `track.h`)

```c
BOOL_T  QueryTrack( track_p t, int q );   // check track property
TRACK_P OnTrack(coOrd *pos, BOOL_T canExtend, BOOL_T canModify);
TRACK_P OnTrackSilent(...);                // no InfoMessage if fails

BOOL_T CheckTrackLayer(track_p trk);       // must not be frozen or module-only
BOOL_T CheckTrackLayerSilent(...);          // silent version

#define Q_CAN_MODIFY_CONTROL_POINTS  ...   // is a Bezier segment?
#define Q_IS_CORNU                   ...    // is a Cornu spiral?
#define Q_IS_DRAW                    ...    // is a DRAW object?
#define Q_CAN_EXTEND                ...    // can this end be extended?
```

## Geometry: Computing an Extended Segment

When extending with a curve, the following steps occur (simplified):

1. **Compute joint geometry** — find where the easement arc meets the existing track and the new circular arc. This uses `ComputeJoint(arcR, r1, &jointD)` which solves for the tangent point between two circles of radii `arcR` and `r1`.
2. **Clip to minimum length** — if the extended segment is shorter than `minLength`, an error message appears.
3. **Build track segments**:
   - If straight: create a `SEG_STRTRK` with endpoints at the original end and the new computed point.
   - If curved: create a `SEG_CRVTRK` with center, radius, start angle (`a0`), and sweep angle (`a1`).
4. **Connect** the old track to the new segment via `ConnectTracks(oldTrk, epIdx, newTrk, inx)`.

## Includes (utility.h is implicitly included via other headers)

```c
#include "cjoin.h"
#include "ccurve.h"
#include "cbezier.h"
#include "ccornu.h"
#include "cstraigh.h"
#include "cundo.h"
#include "fileio.h"
#include "param.h"
#include "track.h"
#include "drawgeom.h"
#include "common.h"
#include "layout.h"
#include "cselect.h"
#include "common-ui.h"
#include "draw.h"
```

## Notes

- The `Dex` struct is a single global instance shared across all threads; this is safe because the command is interactive and state changes are reflected immediately on the display.
- `anchors_da` is used only during drag operations to show blue arcs around endpoints, giving visual feedback that an endpoint is being dragged.
- The undo system (`UndoStart`, `UndoEnd`) wraps each successful modify operation so it can be undone via Ctrl+Z later.
