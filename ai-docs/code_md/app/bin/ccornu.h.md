# ccornu.h — Cornu Easement Curve Header

## Overview

`ccornu.h` declares the public API for the Cornu easement curve command system. It provides function prototypes and type definitions used by `ccornu.c` to create, modify, and convert between Cornu easements and other track types (straights, circular arcs).

## File Location

```
app/bin/ccornu.h  (37 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `common.h` | Common XTrkCad types (`STATUS_T`, `wAction_t`, `coOrd`, etc.) |

## Type Definitions

### `cornuMessageProc` — Message callback type

A function pointer type used to pass a message procedure (typically `InfoMessage`) into command handlers so they can report status messages back to the user.

```c
typedef void (*cornuMessageProc)( const char *, ... );
```

## Enums & Constants

```c
#define cornuCmdNone         (0)        // No active Cornu command
#define cornuJoinTrack       (1)        // Command: join two tracks with a Cornu
#define cornuCmdCreateTrack  (2)        // Command: create a new Cornu track
#define cornuCmdHotBar       (3)        // Hotbar command; invoked from UI hotbar
```

These constants are used internally by `ccornu.c` to distinguish between different invocation modes of the Cornu creation command.

## Function Declarations

### `STATUS_T CmdCornu(wAction_t action, coOrd pos)`

Main entry point for all user interactions with a Cornu easement. Handles:
- Creating a new Cornu track by clicking two endpoints
- Modifying an existing Cornu via handle dragging
- Converting tracks to/from Cornu representation

**Parameters:**
- `action` — One of the standard mouse event codes (`C_START`, `wActionMove`, `C_DOWN`, `C_MOVE`, `C_UP`, `C_LCLICK`, etc.)
- `pos` — Mouse cursor position in screen coordinates

**Returns:** `STATUS_T` — one of the standard command return codes (`C_CONTINUE`, `C_TERMINATE`, `C_ERROR`)

### `BOOL_T CallCornu0(coOrd pos[2], coOrd center[2], ANGLE_T angle[2], DIST_T radius[2], dynArr_t *array_p, BOOL_T spots)`

Constructs the polynomial spiral spline and converts it to a chain of circular arcs. This is the core solver that interfaces with Raph Levien's spiro library. It builds the knot array, calls `TaggedSpiroCPsToBezier`, then closes the context.

**Parameters:**
- `pos[2]` — The endpoint positions (start and end)
- `center[2]` — Centers of the curve at each end; `(0,0)` means "straight" tangent
- `angle[2]` — Tangent angles at endpoints
- `radius[2]` — Radius at each end; `-1.0` means no endpoint (open), `0.0` means straight tangent
- `array_p` — Pointer to a `dynArr_t` that will receive the resulting track segments
- `spots` — Reserved flag (currently unused)

**Returns:** `TRUE` if successful, `FALSE` otherwise.

### `DIST_T CornuMinRadius(coOrd pos[4], dynArr_t segs)`

Scans a sequence of Cornu spiral segments and returns the minimum radius (i.e., maximum curvature). Used for validation against a minimum-radius constraint.

**Parameters:**
- `pos[4]` — Four positions: start, end point 1, end point 2, end point 3 (for nested Bezier chains)
- `segs` — The array of track segments representing the spiral

**Returns:** Minimum radius along the curve; returns infinity if no curved segment is found.

### `DIST_T CornuMaxRateofChangeofCurvature(coOrd pos[4], dynArr_t segs, DIST_T *last_c)`

Computes $\displaystyle \max_{i} \frac{|\kappa'(s_i)|}{2\,\ell_i}$ where $\kappa'$ is the rate of change of curvature and $\ell_i$ is the length of segment $i$. This measures how rapidly curvature changes, which relates to passenger comfort on real railways.

**Parameters:**
- `pos[4]` — Same as above (for recursive descent into nested segments)
- `segs` — The array of track segments
- `last_c` — Output: the curvature value at the previous segment boundary (passed by pointer so it can be carried through recursion)

**Returns:** Maximum rate-of-change-of-curvature along the entire curve.

### `DIST_T CornuLength(coOrd pos[4], dynArr_t segs)`

Computes total length of a Cornu spiral by summing arc lengths of all constituent circular arcs and straight segments.

### `DIST_T CornuOffsetLength(dynArr_t segs, double offset)`

Computes the length of an offset curve (parallel curve) at a given signed distance from the original. Used for generating offset tracks.

### `DIST_T CornuTotalWindingArc(coOrd pos[4], dynArr_t segs)`

Returns the total accumulated turning angle (winding number × $2\pi$) along the spiral, measured in radians. Equivalent to $\int \kappa(s)\,ds$.

### `STATUS_T CmdCornuModify(track_p trk, wAction_t action, coOrd pos, DIST_T trackG)`

Command handler for modifying an existing Cornu easement track (e.g., dragging endpoint handles). Not all parameters are fully documented in the header — see `ccornu.c` for full behavior.

### `void InitCmdCornu(wMenu_p menu)`

Registers the Cornu command with the hotbar and context menus. Creates:
- "Convert To Cornu" button
- "Convert From Cornu" button
- Hotbar entry point

**Parameters:**
- `menu` — The main application menu (`wMenu_p`)

### `void AddHotBarCornu(void)`

Adds a hotbar entry that invokes the "create new Cornu track" command. This is typically bound to a key or toolbar icon in the UI.

## Notes

- The header file is minimal and serves primarily as an API surface; most of the implementation logic lives in `ccornu.c`.
- The spiro solver integration is opaque from this layer — it appears as a black box that takes endpoint conditions and returns Bezier segments.
- The "convert" commands allow users to switch between representations: explicit Cornu vs. explicit chain of arcs/straights, which matters for editing granularity.
