# ccurve.c — Curved Track Creation Commands

## Overview

`ccurve.c` implements the interactive command system for creating curved track segments in XTrkCAD. It provides four distinct methods for defining an arc:

1. **From End-Point** (`crvCmdFromEP1`) — Click two endpoints; the curve is tangent to an existing track at one end.
2. **From Tangent** (`crvCmdFromTangent`) — Click endpoint, then center point (tangent from endpoint).
3. **From Center** (`crvCmdFromCenter`) — Click center of arc, then an endpoint on the chord.
4. **From Chord** (`crvCmdFromChord`) — Click two endpoints defining a chord; the arc is tangent to both at those points.

Additionally, it implements:
- `CmdBezCurve` / `CmdCornu` — Bezier and Cornu spiral curve creation (delegates to other modules)
- **Helix track** (`CmdHelix`) — Creates helical tracks for tunnels or elevated sections with configurable vertical separation per turn
- **Fixed-radius circle** (`CmdCircle`) — Creates a full circular arc of fixed radius

---

## Core Data Structures

### `createState_e` — Command State Machine

```c
typedef enum createState_e {
    NOCURVE,        // No curve currently being created (waiting for first click)
    FIRSTEND_DEF,   // First endpoint has been defined; waiting for second point
    SECONDEND_DEF,  // Second endpoint defined; intermediate geometry computed
    CENTER_DEF      // Center point has been locked in
} createState_e;
```

This finite-state machine tracks the multi-step interactive drawing process. Each state corresponds to a specific "drag mode" shown in the info message (e.g., *"Drag from endpoint in direction of curve"*).

---

### `Da` — Curve Drawing State Structure

```c
static struct {
    createState_e create_state;   // Current step in the interactive process
    coOrd pos0;                   // First control point (endpoint or center)
    coOrd pos1;                   // Second control point (other endpoint, etc.)
    curveData_t curveData;       // Computed arc data: center, radius, a0/a1 angles
    track_p trk;                 // Optional: existing track being joined to
    EPINX_T ep;                  // Endpoint index on `trk` for joining
    BOOL_T down;                 // True after first mouse click (C_DOWN action)
    BOOL_T lock0;                // Whether the first point was snapped to an endpoint
    coOrd middle;                // Midpoint of chord (for chord-mode display)
    coOrd end0, end1;            // Computed endpoints for visualization
} Da;
```

This is a global/static structure that persists across mouse events (`C_START`, `C_DOWN`, `C_MOVE`, `C_UP`) until the command completes or cancels.

---

### `tempSegs_da` — Temporary Segments Display Buffer

A dynamic array of `trkSeg_t` structures used to render intermediate geometry (red arrows, anchor arcs) while the user is dragging. These are NOT part of the final track; they're only shown on screen until `C_UP` commits or cancels the creation.

---

### `anchors_da` — Anchor Markers for Curved Track Creation

```c
static dynArr_t anchors_da;  // Array of trkSeg_t used to draw "anchor" markers
```

Each anchor is a small arc (`SEG_CRVLIN`) drawn in blue around an endpoint. These indicate where the curve will connect to existing geometry or show possible join points. They are rendered during the drag but are not part of the final track definition.

---

### `desired_radius` — User-Configured Default Radius

```c
static DIST_T desired_radius = 0.0;
```

A preference-storable value that acts as a default radius when creating curved tracks via the "curve from tangent" or similar modes. This allows power users to set a preferred curve radius and apply it repeatedly without re-entering the value each time.

---

## Core Functions

### `CreateCurve(action, pos, track, color, width, mode, anchor_array, message)` — Main Entry Point

The central function dispatched from the command system. It handles all mouse actions (`C_START`, `C_DOWN`, `C_MOVE`, `C_UP`) and delegates to sub-modes via a `switch` on `action`.

**Parameters:**
- `track`: TRUE if drawing onto an existing track (joining), FALSE for free-form creation
- `color`: Draw color for temporary segments (usually black)
- `mode`: One of the four curve creation modes (`crvCmdFromEP1`, etc.)
- `anchor_array`: Output pointer where anchor markers are accumulated
- `message`: A callback function used to display info messages (e.g., `InfoMessage`)

**Returns:** `C_CONTINUE` or `C_TERMINATE` (and optionally `C_ERROR`).

---

### `DrawArrowHeads(sp, pos, angle, bidirectional, color)` — Draw Direction Indicators

Renders a small arrowhead centered at `pos`, pointing along `angle`. Used to show the "direction of curve" when dragging from an endpoint. The arrow is composed of 5 line segments forming a triangle/chevron shape.

---

### `DrawArrowHeadsArray(anchor_array, pos, angle, bidirectional, color)` — Draw Multiple Arrows

A convenience wrapper that appends new arrows to the `anchor_array` dynamic array and returns the count added (always 2 for bidirectional arrows). The actual drawing is delegated back to `DrawArrowHeads`.

---

### `CreateEndAnchor(pos, anchor_array, lock)` — Create an Anchor Marker

Creates a small circular arc centered at `pos` with radius ≈ 7.5 units (15% of scale factor), used as a visual marker indicating where a curve endpoint is locked or could be placed. The anchor uses a full 360° arc (`a0=0`, `a1=360`) and is colored blue unless the point is locked (then red).

---

### `CmdCurve(action, pos)` — Curve Command Dispatcher

The public command function that users invoke via menu/hotbar. It sets up internal state variables and delegates to `CreateCurve` with a default color (`wDrawColorBlack`) and width (`0`).

**Modes registered in the menu:**
- `"cmdCurveEndPt"` → joins existing track at endpoint 1
- `"cmdCurveTangent"` → tangent-from-endpoint mode
- `"cmdCurveCenter"` → center-point mode
- `"cmdCurveChord"` → chord-between-two-points mode

---

### `CmdCircleCommon(action, pos, helix)` — Shared Circle/Helix Logic

Handles both fixed-radius circles (`CmdCircle`) and helix tracks (`CmdHelix`). The only difference is the `helix` flag:
- If FALSE (circle): uses a simple radius value and creates a full 360° arc or partial arc
- If TRUE (helix): uses a dialog-controlled radius, number of turns, angular separation, and grade

**Circle modes:**
- `"cmdCircleFixedRadius"` — user specifies a fixed radius in a preference field; draws an arc tangent to a selected endpoint
- `"cmdCircleFromTangent"` — click on the edge (tangent point), then drag inward to define center
- `"cmdCircleFromCenter"` — click on the center, then drag outward to define radius

**Helix parameters (via `helixPG` dialog):**
| Field | Type | Description |
|-------|------|-------------|
| `elev` | float | Elevation difference from start to end of helix |
| `radius` | float | Radius of the horizontal circle (≥ 1) |
| `turns` | integer | Number of full turns around the helix axis |
| `angSep` | float | Angular separation between consecutive turns (degrees, 0–360) |
| `grade` | float | Grade percentage; if non-zero, radius is derived from elevation/turns instead |
| `vertSep` | float | Vertical spacing per turn (computed automatically unless grade is set) |

The "Total Length" message field updates dynamically as parameters change.

---

### `CmdCircle(action, pos)` — Circle Track Creation

A thin wrapper around `CmdCircleCommon` with `helix=FALSE`. It uses either a fixed radius from preferences or computes the radius from two clicked points (center + tangent point).

**Validation checks in `C_UP`:**
- Radius must be > 0
- Radius must not exceed map dimensions (`mapD.size.x`, `mapD.size.y`)
- Radius must not exceed 10,000 units (prevents absurdly large circles)

---

### `CmdHelix(action, pos)` — Helix Track Creation

Identical to `CmdCircle` but with `helix=TRUE`. In addition to the usual radius checks, helix mode also requires:
- Radius > 0
- Number of turns ≥ 1
- Total horizontal length (turns × circumference) must fit within map bounds

The vertical geometry is computed from the elevation difference divided by total number of turns.

---

### `ComputeHelix(pg, inx, data)` — Helix Parameter Update Handler

Called every time a helix parameter control changes (elevation, radius, turns, angular separation, grade). It:
1. Loads current values from all controls into `helixDataCur`
2. Computes derived fields: total turns = integer turns + fractional turn from angular separation; vertical separation per turn; grade percentage if elevation is fixed
3. Updates the "Total Length" message field
4. Validates that the resulting helix fits on the map

The function uses a clock-based ordering system (`h_orders`) to detect which fields need recomputation and propagate changes correctly (e.g., changing turns recalculates vertical separation; changing grade recalculates radius).

---

### `InitCmdCurve(menu)` — Menu Initialization

Registers all curve-related commands with the given menu. It creates button groups labeled "Curve Tracks" and "Circle Track", each containing one or more buttons that invoke the respective command functions. Icons are loaded from bitmaps (`.image3` includes) for:
- Curved-end, curved-tangent, curved-middle, curved-chord
- Bezier curve
- Cornu spiral
- Fixed-radius circle

It also registers the `circleRadiusPG` param group so that its floating-point radius control is persisted to preferences.

---

### `InitCmdHelix(menu)` — Helix Command Registration

Adds a single pulldown menu item `"cmdHelix"` labeled "Helix" under which a sub-menu or dialog would appear (the code shows it uses a param dialog rather than icons). The helix dialog is created on first use (`C_START`).

---

### `CmdBezCurve` / `CmdCornu` — Delegates to Other Modules

These are stub command functions that delegate to external modules:
- **Bezier** → `cbezier.c` (Bézier curve segments)
- **Cornu** → `ccornu.c` (Euler spiral transition curves)

They appear in the menu for consistency but their actual implementation lives elsewhere.

---

## State Machine Flow — "Curve from End-Point" Mode

This is the most interactive of all modes; here's how it progresses:

1. **First click (`C_DOWN`) on an endpoint:**
   - Snap to the nearest unconnected track endpoint if snapped mode is enabled and no key modifier is held
   - If the snap succeeds, that point is "locked" (`lock0 = TRUE`); otherwise the user can still drag freely
   - Show info message: *"End locked: Drag out curve start"* (if locked) or *"Drag along curve start"*
   - Draw a blue arrow pointing in the tangent direction of the existing track

2. **Dragging (`C_MOVE`) after first point is set:**
   - If shift key held, the endpoint position is fixed; otherwise it follows the cursor
   - Compute the chord between the two points and display an arc preview
   - Show info message with current angle or radius depending on mode variant

3. **Second click (`C_UP`):**
   - Validate that both endpoints are separated by at least `minLength` (user preference, typically a few units)
   - Create the curved track segment using `NewCurvedTrack()`
   - If not shifted and an existing track was specified, connect the two tracks via `ConnectTracks()`
   - Draw the new track with its endpoint graphics

4. **Cancel:** Press ESC or click outside → reset state variables, clear temporary segments

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `DrawArrowHeads(sp, pos, angle, bidirectional, color)` | Draw a single arrowhead at a given position and orientation | segment pointer, coordinate, heading angle |
| `DrawArrowHeadsArray(anchor_array, pos, angle, bidirectional, color)` | Append two arrowheads to the anchor array for display | dynamic array handle, position, angle |
| `CreateEndAnchor(pos, anchor_array, lock)` | Create a small circular marker at a point; blue if free, red if locked | coordinate, output array, boolean flag |
| `CreateCurve(action, pos, track, color, width, mode, anchor_array, message)` | Main curve-creation dispatcher for all four modes and Bezier/Cornu | action code, cursor position, boolean, color, width, mode enum, dynamic array, callback |
| `CmdCurve(action, pos)` | Public command entry point; registers with menu system | action, coordinate |
| `CmdCircleCommon(action, pos, helix)` | Shared logic for both fixed-radius circles and helices | action, coordinate, boolean flag |
| `CmdCircle(action, pos)` | Create a full or partial circle track of fixed or computed radius | action, coordinate |
| `CmdHelix(action, pos)` | Create a helical (spiral) track with configurable vertical separation | action, coordinate |
| `ComputeHelix(pg, inx, data)` | Update helix geometry when any parameter changes; validate fit on map | param group pointer, control index, junk pointer |
| `InitCmdCurve(menu)` | Register all curve-creation commands with a menu | menu handle |
| `InitCmdHelix(menu)` | Register the helix command (uses a dialog instead of icons) | menu handle |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Provide an interactive, mouse-driven interface for creating curved track segments using four different geometric definitions (endpoint, tangent, center, chord), plus helix and circle tracks. |
| **Domain** | Interactive drawing: the code implements a state machine that translates a sequence of mouse clicks and drags into geometric primitives (arcs). It also handles validation (minimum length, map bounds) and integration with existing track segments via endpoint connection. |
| **Key concept** | The `createState_e` enum drives a multi-step interactive process where each user action advances the state machine from "waiting for first point" → "first point locked" → "dragging second point" → "commit or cancel". Temporary geometry (arrows, anchor arcs) is rendered in a separate dynamic array (`tempSegs_da`) and never becomes part of the track database. |
| **Main entry points** | `CmdCurve()` — invoked from menus/hotbar for creating curved tracks; `InitCmdCurve(menu)` — registers commands with the menu system |
