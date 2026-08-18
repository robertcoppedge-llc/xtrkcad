# command.c — Application Command Dispatcher

## Overview

`command.c` is the **central application dispatcher** that maps user input (mouse clicks, keyboard shortcuts, toolbar buttons) to actionable operations. It implements a "sticky command" pattern where certain commands remain active until explicitly cancelled or completed, and manages command enablement based on program mode (Design vs Train), selection state, and other runtime conditions.

---

## Global State

### `commandList[COMMAND_MAX]`

An array of command descriptors, each representing one toolbar button/menu item:

```c
struct {
    procCommand_t cmdProc;          // Pointer to the command's action handler
    char * helpKey;                 // Short name for undo/redo labels and menus
    wIndex_t buttInx;              // Toolbar button index (or -1)
    char * labelStr;               // Full display name of the command
    wIcon_p icon;                  // Icon bitmap for toolbar/menu
    int reqLevel;                  // Required menu level depth
    wBool_t enabled;               // Current enabled/disabled state
    long options;                  // Bitmask of flags (IC_MODETRAIN_ONLY, etc.)
    long stickyMask;              // Triggers when a key is pressed to activate/sticky mode
    wMenuPush_p menu[NUM_CMDMENUS];  // Menu items in each popup level
    void * context;               // Per-command context data passed to cmdProc
} commandList[COMMAND_MAX];
```

| Field | Description |
|-------|-------------|
| `cmdProc` | Function pointer implementing the command's behavior |
| `helpKey` | Short identifier used in undo history (e.g., "DrawLine", "MoveTrack") |
| `buttInx` | Index into toolbar button array; -1 if no toolbar button |
| `icon` | Pixmap or icon for display on toolbar/menu |
| `reqLevel` | Required menu nesting depth before this command appears |
| `options` | Bitmask controlling enablement logic (mode checks, selection requirements) |
| `stickyMask` | Accumulated key flags; when matching `stickySet`, the command becomes sticky |
| `menu[]` | Pointers to sub-menu items for popup menus |
| `context` | Per-command state data passed through all mouse-event callbacks |

---

### `curCommand`

Index into `commandList` indicating which command is currently active. When a new command starts, the previous one's cleanup (`C_CANCEL`) runs first.

---

### `preSelect`, `rightClickMode`

- **`preSelect`** — Default command when no drag/drag operation is in progress (0 = Describe, 1 = Select).
- **`rightClickMode`** — Tracks whether right-click was just triggered; used to suppress spurious menu pops.

---

### `commandContext`

Pointer stored in each `commandList[*].context` field and passed through all mouse event handlers (`C_START`, `C_MOVE`, `C_UP`, etc.) so a command can maintain per-session state (e.g., for multi-point drawing, accumulated measurements).

---

## Core Functions

### `IsCommandEnabled(long mode, long options)`

Determines whether a command should be enabled based on the current program mode (`MODE_DESIGN` or `MODE_TRAIN`) and its option flags:

- **Design-only** commands: enabled only in Design mode
- **Train-only** commands: enabled only in Train mode  
- **Both-mode** commands: always enabled regardless of mode
- **Selection-dependent** commands: disabled when no tracks are selected (`selectedTrackCount == 0`)

The logic uses a CNF-style boolean expression to avoid redundant checks.

---

### `EnableCommands(void)`

Walks all registered commands and updates their `.enabled` flag, then calls `ToolbarButtonEnable()` or `wMenuPushEnable()` as appropriate. Also enables/disables the "Select" command based on whether any tracks are currently selected.

Called after undo/redo operations to reflect changes in selection state.

---

### `GetCurrentCommand(void)` / `GetCurCommandName(void)`

Returns the current active command index and its short name, respectively. Used for logging, undo descriptions, and status bar display.

---

### `Reset(void)`

Cleans up the current sticky/in-progress command:
- Calls the previous command's `C_CANCEL` handler to restore geometry
- If a toolbar button is associated, releases the "busy" flag
- Resets cursor to default or question-mark (depending on `preSelect`)
- Clears the temporary segment array (`tempSegs_da`) — used by multi-point drawing commands
- Calls `TryCheckPoint()` to save state before reset
- Redraws and re-enables all commands

---

### `DoCurCommand(wAction_t action, coOrd pos)`

Main entry point for processing mouse events within an active command. It:
1. Checks whether the event should terminate the current command (e.g., cursor moved out of a dragging zone)
2. Handles right-click menu invocation (popup 1 or popup 2 depending on selection count)
3. Calls `commandList[curCommand].cmdProc(action, pos)` to invoke the specific handler
4. Processes the return code (`C_CONTINUE`, `C_TERMINATE`, `C_ERROR`) and redraws accordingly

Returns one of:
- `C_CONTINUE` — keep processing events in this command
- `C_TERMINATE` — finish this command (usually leads to calling `ConfirmReset()`)
- `C_ERROR` — an error occurred; reset will occur after confirmation

---

### `ConfirmReset(BOOL_T retry)`

Called when the user cancels a command via Escape or another means. Prompts with a warning dialog if there are pending modifications, offering "Yes" (commit), "No" (cancel), or "Cancel". If no changes exist, silently resets without prompting. Returns the result of the confirmation dialog.

---

### `DoCommandB(void *data)`

Invoked by toolbar button callbacks and menu selections. It:
1. Finalizes the previous command (`C_FINISH`, `C_CONFIRM`)
2. Updates the toolbar button to show busy state with the new icon
3. Calls the new command's `C_START` handler
4. Handles return codes similarly to `DoCurCommand`

---

### `PlaybackCommand(const char *line, wIndex_t lineNum)`

Called during playback of recorded sessions (macro replay). Parses a "COMMAND X" token from the recording file and executes that command by setting its toolbar button into playback mode. Inserts small pauses between steps so the user can follow along visually.

---

### `AddCommand(...)` / `AddParam(...)` — Registration Functions

(These are aliases/wrappers used in practice.)

Adds a new entry to `commandList`. Registers:
- The command handler function (`cmdProc`)
- Toolbar button index (if any)
- Menu items under "Edit", "View", etc.
- Icon and help key strings

---

## Command Flags (in `.options` field)

| Flag | Value | Meaning |
|------|-------|---------|
| `IC_MODETRAIN_ONLY` | bit 0 | Only enable in Train mode |
| `IC_MODETRAIN_TOO` | bit 1 | Also enable in Design mode |
| `IC_SELECTED` | bit 2 | Require at least one selected track to be enabled |
| `IC_RCLICK` | bit 3 | Show right-click popup menu when cursor is over the command area |
| `IC_WANT_MOVE` | bit 4 | Cursor moves during drag (e.g., selecting) |
| `IC_WANT_MODKEYS` | bit 5 | Consumes modifier keys (Shift/Ctrl/Alt) |
| `IC_LCLICK` | bit 6 | Treat mouse down as a left-click event (for double-click detection) |
| `IC_STICKY` | bit 7 | Command remains active until cancelled/completed |
| `IC_NORESTART` | bit 8 | Do not restart the command on sticky key trigger; just activate it |

---

## Sticky Commands

Sticky commands (bit 7 set) remain active after their initial activation event. They continue to respond to mouse moves/drag events until:
- The user cancels via Escape or clicks elsewhere
- The command reaches completion (`C_OK`, `C_FINISH`)
- A sticky-triggering key is pressed and the accumulated mask matches `stickySet`

This pattern supports operations like:
- **Drawing** (point-and-drag lines, circles, polygons)
- **Moving/rotating/flipping** selected tracks
- **Measuring** distances/angles while dragging cursor

---

## Related Files

| File | Purpose |
|------|---------|
| `command.h` | Type definitions (`procCommand_t`, flag constants) |
| `cselect.c` | Selection logic, anchor drawing for command hints |
| `cundo.c` | Undo/redo transaction handling |
| `track.h/track.c` | Track data structures |
| `common-ui.h` / `menu.h` | Menu and toolbar widget APIs |
