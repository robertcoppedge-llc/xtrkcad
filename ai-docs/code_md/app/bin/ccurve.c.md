# ccurve.c — Curve Track Creation (Arcs, Clothoids, Circles, Helices)

## Overview

`ccurve.c` implements the creation of **curved track segments** for XTrkCAD. It provides:
- Interactive curve drawing from endpoint, tangent, or center-point modes
- Circle and helix (vertical spiral) generation
- Bézier curve support (delegated to `cbezier.c`)
- Clothoid/Cornt spiral integration (via `ccornu.h`)

The file is split between:
1. **Interactive command handlers** (`CmdCurve`, `CmdCircle`, `CmdHelix`) — handle mouse events and preview geometry
2. **Drawing utilities** (`DrawArrowHeads*`) — render temporary anchor markers during interactive placement
3. **State machine logic** for multi-step creation wizards

---

## Data Structures

### `createState_e` — Command State Enumerations

```c
typedef enum {NOCURVE, FIRSTEND_DEF, SECONDEND_DEF, CENTER_DEF} createState_e;
```

This enum tracks the progress of a curve creation operation:
- `NOCURVE` — No curve is being created (initial state)
- `FIRSTEND_DEF` — First endpoint has been selected; waiting for second
- `SECONDEND_DEF` — Both endpoints defined; previewing the curve
- `CENTER_DEF` — Center point has been specified

---

### `Da` — Global Curve Creation State

A static struct that persists state across mouse events:

```c
static struct {
    STATE_T state;              // Generic command-state enum (start, move, up, etc.)
    createState_e create_state; // Which stage of curve creation?
    coOrd pos0;                 // First endpoint position
    coOrd pos1;                 // Second endpoint position
    curveData_t curveData;     // Computed curve geometry (center, radius, angles)
    track_p trk;               // Reference to existing track if connecting
    EPINX_T ep;                // Endpoint index of the referenced track
    BOOL_T down;               // Mouse button currently pressed?
    BOOL_T lock0;              // Is the first endpoint locked in place?
    coOrd middle;              // Midpoint of chord (used for chord-mode curves)
    coOrd end0;                // Current endpoint under construction
    coOrd end1;                // Other endpoint
} Da;
```

The `Da` struct is **global** because curve creation is a multi-step command that spans many mouse events. It remembers which points have been selected and whether the user has locked endpoints to an existing track.

---

### `curveData_t` — Computed Curve Geometry

This structure (defined in `ccurve.h`) holds the computed parameters for any type of curve:
- Center point coordinates
- Radius
- Start/end angles (`a0`, `a1`)
- Type indicator (straight, arc, clothoid)
- For helices: number of turns, vertical separation

---

### `helixData_s` — Helix Configuration

```c
struct helixData_s {
    long turns;                    // Number of full 360° revolutions
    ANGLE_T angSep;               // Additional angular separation (partial turn)
    DIST_T elev;                   // Total vertical rise/fall across the helix
    DIST_T radius;                // Radius of the horizontal circle
    DIST_T grade;                 // Grade percentage (rise/run × 100%)
    DIST_T vertSep;               // Vertical spacing between windings
};
```

A helix is defined by its **radius**, number of turns, and total vertical separation. The structure computes derived quantities such as:
- Total length = `turns × radius × 2π`
- Grade = `(elevation / (turns × circumference)) × 100%`
- Vertical spacing per turn = `elev / turns`

---

### `anchors_da` — Anchor Markers for Interactive Placement

```c
static dynArr_t anchors_da;
#define anchors(N) DYNARR_N(trkSeg_t, anchors_da, N)
#define array_anchor(N) DYNARR_N(trkSeg_t, *anchor_array, N)
```

During interactive curve placement (e.g., "drag from endpoint"), the user sees **red arrows** drawn at anchor points. These are stored as a dynamic array of `trkSeg` structures with type `SEG_CRVLIN` or `SEG_FILCRCL`, colored red (`wDrawColorRed`). The anchors guide the user by showing:
- Where an arrowhead points — the direction of allowable movement
- When an endpoint is "locked" to a track — a blue arc appears instead

---

### `curveMode` — Current Command Mode

```c
static long curveMode;
```

Holds the selected creation mode (0 = endpoint-based, 1 = tangent-based, etc.). This is set when the user picks a command from the menu and used throughout the command handler.

---

## Core Functions

### `DrawArrowHeads(...)` — Draw Red Arrowheads for Interactive Guidance

```c
EXPORT int DrawArrowHeads(
    trkSeg_p sp,           // Pointer to an array of segments (buffer)
    coOrd pos,             // Center point of the arrowheads
    ANGLE_T angle,         // Angle of the curve at that point
    BOOL_T bidirectional,  // Draw arrows on both ends?
    wDrawColor color       // Color: typically wDrawColorRed for active anchors
)
```

**Purpose:** Renders a set of five small line segments forming arrowheads. These are drawn as **temporary preview geometry** to show the user where they can drag their mouse to define the curve.

**Geometry:** The function draws five `trkSeg` entries, each with:
- `type = SEG_STRLIN` (a straight-line segment)
- A width of about 2 pixels (`w = mainD.scale/mainD.dpi * 2`)
- Color set by the caller

The arrowhead geometry is constructed from a center point and angle using trigonometry. The base of each triangle has length `d/2`, where `d` is derived from the scale factor.

**Bidirectional mode:** When `bidirectional == TRUE`, an additional pair of arrows is drawn pointing in the opposite direction (at ±45° offset), giving a "double-headed" appearance. This helps users understand that they can drag in either direction along the tangent.

---

### `DrawArrowHeadsArray(...)` — Batch Arrow Drawing via Dynamic Array

```c
EXPORT int DrawArrowHeadsArray(
    dynArr_t *anchor_array, // Output: array of segments to be drawn
    coOrd pos,              // Center position
    ANGLE_T angle,          // Direction along the curve
    BOOL_T bidirectional,   // Draw both directions?
    wDrawColor color        // Color
)
```

This wraps `DrawArrowHeads` by:
1. Appending 5 new segment entries to the dynamic array
2. Calling `DrawArrowHeads` with a pointer into that array

It is used when multiple anchors need to be drawn at once (e.g., chord mode draws arrows at both endpoints).

---

### `CreateEndAnchor(...)` — Create an Anchor Circle

```c
static void CreateEndAnchor(
    coOrd p,              // Center point for the anchor circle
    dynArr_t *anchor_array,  // Output array
    wBool_t lock          // If TRUE, draw a blue arc instead of red arrows (endpoint locked)
)
```

Draws a small **blue circle** at position `p` to mark an endpoint that is either:
- **Locked** (`lock == TRUE`) — the user cannot move this point anymore; only the other endpoint is free to drag
- **Free** (`lock == FALSE`) — the user can still move this point

The anchor is drawn as a filled circle with radius `d/2` (where `d = tempD.scale * 0.15`). The blue color (`wDrawColorBlue`) contrasts with the red arrowheads for active dragging areas.

---

### `CreateCurve(...)` — Main Interactive Command Handler

```c
EXPORT STATUS_T CreateCurve(
    wAction_t action,   // C_START / C_MOVE / C_UP / C_CANCEL / C_TEXT
    coOrd pos,          // Current mouse position
    BOOL_T track,       // TRUE = we are snapping to an existing track endpoint
    wDrawColor color,   // Color for preview geometry
    LWIDTH_T width,     // Line thickness
    long mode,          // Which creation mode: fromEP1, fromTangent, fromCenter, fromChord
    dynArr_t *anchor_array, // Output array of anchor markers
    curveMessageProc message  // Callback for InfoMessage() calls
)
```

This is the **main entry point** for all curve-creation commands. It implements a state machine over three phases:

#### Phase 1: `create_state == NOCURVE` (Initialization)

The user clicks to start creating a curve. The function sets up initial message text and waits for the first click:
- **From Endpoint (`crvCmdFromEP1`)** — "Drag from endpoint in direction of curve"
- **From Tangent (`crvCmdFromTangent`)** — "Drag from endpoint to center"
- **From Center (`crvCmdFromCenter`)** — "Drag from center to endpoint"
- **From Chord (`crvCmdFromChord`)** — "Drag from one to other end of chord"

If `track == TRUE` and the user is holding Alt (or magnetic snap is on), the cursor snaps to an existing track's unconnected endpoint, and that point becomes locked.

#### Phase 2: `create_state ∈ {FIRSTEND_DEF, CENTER_DEF}` (Dragging)

The second point (`pos1`) follows the mouse, but its motion may be constrained depending on the mode:
- **From Endpoint:** The first endpoint is fixed; dragging moves along a tangent direction from it. If the user holds Shift, the endpoint position itself is locked and only the angle changes.
- **From Tangent:** A circle of radius `d = |pos0 – pos1|` is centered at `pos0`. Dragging defines the tangent direction.
- **From Center:** The center point is fixed; dragging moves the second endpoint around a circle, defining the curve's angular span.
- **From Chord:** Both endpoints are free to move, but they must maintain a constant distance (the chord length). The midpoint of the chord is computed and used as the center for drawing preview arrows.

During this phase, `CreateEndAnchor` is called to draw blue/red marker circles at each endpoint, and `DrawArrowHeadsArray` draws red arrowheads along the tangent direction so the user knows where dragging is allowed.

#### Phase 3: `create_state == SECONDEND_DEF` (Finalizing)

The user releases the mouse button (`C_UP`). The function finalizes the curve geometry by computing the circle center and radius from the two endpoints. If the chord length is shorter than a minimum threshold (`minLength`), an error message is shown and the command fails.

If successful, the state advances to `CENTER_DEF`, where:
- A **parameter dialog** appears (with a "Desired Radius" slider)
- The user can adjust the curve radius before confirming
- When confirmed, `NewCurvedTrack` or `NewStraightTrack` is called (depending on computed type)

#### Phase 4: Cleanup (`C_CANCEL`)

The dynamic arrays are reset and the global state machine returns to `NOCURVE`. No track object was created.

---

### `CmdCurve(wAction_t action, coOrd pos)` — Command Handler Wrapper

```c
EXPORT STATUS_T CmdCurve(wAction_t action, coOrd pos)
```

This is the top-level command handler that wraps `CreateCurve`:

- **C_START / C_DOWN** — Initialize the state machine and call `CreateCurve`. Sets up a parameter dialog for radius selection.
- **C_MOVE** — Call `CreateCurve` with mouse position; preview geometry updates automatically via `DrawSegsDA`.
- **C_UP** — Finalize track creation (or show error if constraints are violated). Calls `NewCurvedTrack` or `NewStraightTrack`, optionally connects to an existing track endpoint, and calls `DrawNewTrack`.
- **C_CANCEL** — Reset state.
- **C_REDRAW** — Draw the preview segments for the current mouse position.

---

### `CmdCircle(...)` — Circle Creation Command Handler

```c
static STATUS_T CmdCircle(wAction_t action, coOrd pos)
```

Handles creation of circular track segments (full 360° circles). It supports three modes:

1. **Fixed Radius** (`circleCmdFixedRadius`) — User types or selects a radius in a parameter dialog; dragging places the circle center on the layout.
2. **From Tangent** (`circleCmdFromTangent`) — Click an existing track endpoint and drag to define where the circle is tangent to that point. The user drags until the preview circle aligns with their desired placement.
3. **From Center** (`circleCmdFromCenter`) — User clicks a center point on the layout; dragging defines the radius by specifying the circumference location.

A red circle is drawn as preview geometry during `C_MOVE`, sized according to the current parameter value.

---

### `CmdHelix(...)` — Helix (Vertical Spiral) Creation Command Handler

```c
static STATUS_T CmdHelix(wAction_t action, coOrd pos)
```

Creates a helix track — a vertical spiral that winds upward or downward while rotating horizontally. The dialog (see below) allows the user to specify:
- Elevation difference between top and bottom
- Radius of the horizontal circle
- Number of full turns plus an optional partial turn (`angSep`)
- Grade (%) as a derived readout

The helix is rendered as a series of **full 360° circular track segments** stitched together. Each full turn corresponds to one `track` object with:
- `center = (x, y)` at the horizontal circle's center
- `radius = helixDataCur.radius`
- Start angle `a0 = 0`, end angle `a1 = 360°`
- A vertical offset is applied to successive circles (not shown in this code but handled by `NewCurvedTrack`)

---

### `ComputeHelix(...)` — Helix Parameter Computation

```c
static void ComputeHelix(paramGroup_p pg, int h_inx, void *data)
```

This function maintains **mutual consistency** between the helix parameters. For example:
- If the user changes "Elevation Difference", the number of turns and vertical spacing are recomputed (unless they were explicitly locked).
- If the radius is changed, the grade (%) is recomputed as `elevation / (turns × 2πr) × 100`.

It uses a **clock system** (`h_clock`, `h_orders`) to track which fields have been modified since the last call. Only those fields are re-evaluated and marked dirty with a bit flag, so the parameter dialog can redraw only what changed.

---

## Command Registration

### `InitCmdCurve(wMenu_p menu)` — Register Curve Commands in Menu

```c
EXPORT void InitCmdCurve(wMenu_p menu)
```

Adds entries to the command palette under "Curve Tracks":
- Curve from Endpoint (icon: curved-end)
- Curve from Tangent (icon: curved-tangent)
- Curve from Center (icon: curved-middle)
- Curve from Chord (icon: curved-chord)
- Bézier curve (delegated to `cbezier.c`)
- Cornu spiral (delegated to `ccornu.h`)

Each entry is registered with `IC_STICKY|IC_POPUP2|IC_WANT_MOVE` flags, meaning the command remains active until canceled and responds to mouse movement events.

---

### `InitCmdCircle(wMenu_p menu)` — Register Circle Commands

Adds three entries:
- Fixed Radius Circle
- Circle from Tangent (click endpoint, drag to define tangent point)
- Circle from Center (click center, drag to define radius)

---

### `InitCmdHelix(wMenu_p menu)` — Register Helix Command

The helix command has no icon in the palette and is only accessible through a pulldown menu. It also registers its parameter dialog (`helixPG`) for unit change notifications (e.g., if the user switches from metric to imperial units, the helix dimensions are re-expressed).

---

## Summary of Creation Modes

| Mode | Description |
|------|-------------|
| **From Endpoint** | Click an existing track endpoint; drag along its tangent direction to define a circular arc. The first point is locked; only the second moves. |
| **From Tangent** | Click an endpoint and drag outward; the cursor traces a circle centered at that point, defining where the curve's center lies. |
| **From Center** | Click a point on the layout as the circle center; dragging defines the radius by specifying a point on the circumference. |
| **From Chord** | Drag from one endpoint to another while maintaining a constant chord length. Both endpoints are free to move along their respective tangent directions. |

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `DrawArrowHeads` | Draw red arrowheads at a point to guide mouse dragging | `pos`, `angle`, `bidirectional`, `color` |
| `DrawArrowHeadsArray` | Append arrows to a dynamic array for batch rendering | `anchor_array`, `pos`, `angle` |
| `CreateEndAnchor` | Draw a blue/red circle at an endpoint (locked or free) | `p`, `anchor_array`, `lock` |
| `CreateCurve` | Main state machine: handles C_START, C_MOVE, C_UP for curve creation | `action`, `pos`, `track`, `mode` |
| `CmdCurve` | Command handler wrapper for curves (arcs) | — |
| `CmdCircle` | Circle creation command with three modes | — |
| `CmdHelix` | Helix (vertical spiral) creation command | — |
| `ComputeHelix` | Keep helix parameters consistent across multiple fields | `pg`, `h_inx` |
| `InitCmdCurve` | Register all curve commands in a menu | `menu` |
| `InitCmdCircle` | Register circle commands | `menu` |
| `InitCmdHelix` | Register helix command and parameter dialog | `menu` |

---

## Design Notes

- **Preview geometry** is rendered using the same track-segment drawing system as real tracks. Temporary segments are stored in dynamic arrays (`tempSegs_da`, `anchors_da`) and drawn on top of the layout during `C_MOVE` events.
  
- **State machine pattern** — The global `Da` struct holds progress through a multi-step wizard. This avoids needing to pass state between event handlers (e.g., C_START, C_DOWN, C_MOVE).

- **Error handling** — If the chord length is shorter than `minLength`, an error message is shown and the command returns `C_TERMINATE`. Similarly, if the helix radius would exceed the layout size, an error is displayed.

- **Undo support** — Each track creation call (`NewCurvedTrack`) is wrapped with a pair of `UndoStart`/`UndoEnd`, so the user can press Ctrl+Z to revert their work.
