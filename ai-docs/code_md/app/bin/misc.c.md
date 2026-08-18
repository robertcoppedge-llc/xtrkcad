# misc.c — Utility Functions and Application Main Entry Point

## Overview

`misc.c` contains a diverse collection of utility functions used throughout the application, plus the main entry point `wMain()` which initializes everything on startup. The file also implements:

- **Memory allocation with guards** — custom `malloc`/`realloc`/`free` that insert guard bytes to detect overruns and underflows
- **Change notification system** — a callback registration mechanism allowing subsystems (draw routines, menus, dialogs) to be notified of state changes
- **Message handling** — centralized error/info notice functions with optional confirmation dialogs
- **Accelerator key configuration** — reading keyboard shortcuts from user preferences
- **Checkpoint recovery** — offering to resume work after a crash

---

## Global Variables

| Variable | Type | Description |
|----------|------|-------------|
| `iconSize` | `int` | Size of toolbar icons in pixels |
| `displayWidth`, `displayHeight` | `wWinPix_t` | Physical display resolution |
| `mainW` | `wWin_p` | Pointer to the main application window |
| `message` / `message2` | `char[]` | Buffers for formatted messages displayed in the info bar |
| `paramVersion` | `long` | Version number of the loaded parameter file (used for schema compatibility checks) |
| `zero` | `coOrd` | The point `(0, 0)` used as a zero vector constant |
| `extraButtons` | `BOOL_T` | TRUE if environment variable is set; enables extra toolbar buttons |
| `onStartup` | `long` | Controls behavior on startup: whether to auto-load last layout |
| `verbose` | `int` | Verbose logging flag (incremented via `-v` CLI option) |
| `inMainW` | `BOOL_T` | Flag indicating the main window is visible/active |
| `units` | `long` | Measurement units: 0 = English, 1 = metric |
| `labelScale` | `long` | Scale at which labels are rendered (default 8) |
| `labelEnable` | `long` | Bitmask of label types enabled (`LABELENABLE_ENDPT_ELEV`, etc.) |
| `labelWhen` | `long` | Controls when track labels appear: 0 = never, 1 = zoomed out only, 2 = always (default) |
| `dontHideCursor` | `long` | If nonzero, prevents cursor auto-hide behavior |
| `totalMallocs`, `totalMalloced` | `size_t` | Count and total bytes allocated by custom allocator |
| `totalRealloced`, `totalFreeed` | `size_t` | Statistics for realloc/free accounting |

---

## Memory Allocation Utilities

The file provides guarded memory allocation functions that insert "guard" bytes before and after each block to detect overruns and underflows.

### `MyMalloc(size_t size)` — Allocate with guards

Allocates a new block of memory, surrounds it with guard patterns (`0xDEADBEEF` before, `0xAF00BA8A` after), initializes the interior to zero, and returns a pointer to the start of usable space. The allocation size is stored at the beginning of the block so that deallocation can recover the full allocated size even if only a portion was used.

### `MyRealloc(void *old, size_t size)` — Reallocate with guards

If `old` is NULL, calls `MyMalloc()`. Otherwise:
- Verifies both guard bytes are intact (aborts on corruption).
- Reads the stored original size from before the block.
- If the requested size equals the current size, returns the same pointer.
- Otherwise allocates a new guarded block, copies overlapping contents using `memcpy`, frees the old block, and returns the new pointer.

### `MyFree(void *ptr)` — Free with guard verification

Verifies both guard bytes before freeing; if either is corrupted, an error message would be logged (though the function still proceeds to free). Calls system `free()` on the actual allocated region (which includes guards and header). If `ptr` is NULL, does nothing.

### `memdup(void *src, size_t size)` — Duplicate a block

Allocates a new guarded block of the given size and copies exactly `size` bytes from `src`. Returns pointer to start of usable space.

### `MyStrdup(const char *str)` — Duplicate a C string

Allocates a new zero-terminated copy of the input string with guards around it.

---

## Character Conversion Utilities

### `ConvertToEscapedText(const char *text)` — Escape special characters

Scans an input string and returns a newly allocated buffer where:
- Newline `\n` → backslash + 'n'
- Tab `\t` → backslash + 't'
- Backslash `\\` → double backslash `\\\\`
- Double quote `"` → escaped as `\"`

This prepares strings for safe writing into CSV or other text formats.

### `ConvertFromEscapedText(const char *text)` — Unescape special characters

Scans an input string and returns a newly allocated buffer where escape sequences are converted back:
- `\n` → actual newline (0x0A)
- `\t` → actual tab (0x09)
- `\\` → single backslash
- `\"` → double quote

---

## Message Handling Functions

### `AbortMessage(const char *format, ...)` — Format an abort/error message

Stores a formatted string in a static buffer. Used by the `CHECKMSG` macro to capture error messages before aborting or showing an error dialog.

### `AbortProg(const char *cond, const char *fileName, int lineNumber, const char *msg)` — Fatal error with save offer

Displays an error notice (via `wNoticeEx`) offering the user a chance to save their layout before terminating. If the user declines, calls system `abort()`. Used by assertion failures (`CHECK`, `CHECKMSG`).

### `ParseMessage(const char *msgSrc)` — Parse a localized message

The `_()` macro is used throughout the codebase for gettext localization. This function parses the resulting string:
- If it contains embedded tabs, splits them into two columns and appends both to a global message list (for info bar display).
- Returns the first column as the formatted message string.

### `InfoMessage(const char *format, ...)` — Display informational message

Formats arguments using `vsnprintf` into the static buffer, then calls `SetMessage()` to update the info bar at the bottom of the main window. No sound is played; `inError` flag prevents further messages until cleared.

### `ErrorMessage(const char *format, ...)` — Display error message with beep

Same as `InfoMessage`, but also plays a system beep (`wBeep()`) and sets the global `inError` flag to prevent subsequent messages from being displayed.

### `NoticeMessage(const char *format, const char *yes, const char *no, ...)` — Show confirmation dialog

Formats a message into `message2` and displays it in a standard GTK notice dialog with two buttons labeled by `yes` and `no`. Returns the button index (0 = yes/cancel, 1 = no/save) or -1 if the user clicked "OK" without choosing.

### `NoticeMessage2(...)` — Confirmation with playback handling

Similar to `NoticeMessage`, but returns a previously-stored result code from a recording/playback scenario so that scripted actions can continue after an aborted confirmation.

### `Confirm(char *label2, doSaveCallBack_p after)` — Save-before-close confirmation

If the layout has been modified (`changed` flag is set), displays a dialog asking whether to save changes before proceeding (closing/quit). If "Don't Save" is chosen, calls the supplied callback (typically `after`) and returns TRUE (proceed); if "Cancel", returns FALSE (abort operation).

---

## Window Management

### `DoQuitAfter(void)` — Post-quit cleanup

Cleared on exit: resets changed flag, deletes checkpoint files, removes temporary archive files, calls `SaveState()`.

### `DoQuit(void *unused)` — Handle quit request

Calls `Confirm()` to ask whether to save if changes exist. If confirmed, logs closing and exits via `wExit(0)`.

### `DoClearAfter(void)` — Clear layout cleanup

Resets the entire application: calls `Reset()`, `ClearTracks()`, `ResetLayers()`, then performs a redraw with a blank background. Sets read-only flag and enables all commands.

### `DoClear(void *unused)` — Menu command handler

Prompts for confirmation before calling `DoClearAfter()`.

### `MapWindowToggleShow(void *unused)` — Toggle map window visibility

Calls `MapWindowShow()` with the inverse of the current state.

### `MapWindowShow(int state)` — Set map window visibility

Sets a flag, stores it in preferences, toggles a menu item, and shows/hides the window. If shown, calls `DoChangeNotification(CHANGE_MAP)`.

### `wShow(wWin_p win)` / `wHide(wWin_p win)` — Show/hide arbitrary windows

`wShow()` adds the window title to a drop-down list in the main menu if not already present, then calls GTK's show function. It also tracks shown windows in an array during playback mode so they can be hidden again later.

`wHide()` removes from the menu list, resets invalid parameters (via `ParamResetInvalid()`), and optionally unregisters from the demo window tracking array during playback.

### `CloseDemoWindows(void)` — Close all temporary windows

Iterates over the tracked windows array and hides each one; then resets the count to zero.

### `DefaultProc(wWin_p win, winProcEvent e, void *data)` — Window event handler stub

Handles close events by removing from menu list, confirming reset if applicable, and closing the window.

---

## Accelerator Keys

A table defines all keyboard shortcuts used in the application:

| Key | Shortcut | Action |
|-----|----------|--------|
| `PgDn` / Ctrl+Numpad+ | Zoom In | Increase zoom level |
| `PgUp` / Ctrl+Numpad- | Zoom Out | Decrease zoom level |
| `F5` | Redraw | Force a full redraw of the layout |
| `Delete` (Win) | Delete | Attempt to delete selected object |
| `Shift+Back` | Undo | Invoke undo operation |
| `Ctrl+Ins` | Copy | Copy selected objects to clipboard |
| `Shift+Ins` | Paste | Paste from clipboard |
| `Shift+Del` | Cut | Cut selected objects to clipboard |
| `F6` | Next window | Switch to next open window |

The mapping is loaded from user preferences under the `"accelKey"` group. Each entry can specify a base key (e.g., "Del", "Ins", "F1") and optional modifier flags (`WKEY_SHIFT`, `WKEY_CTRL`). The `SetAccelKeys()` function reads each preference, parses modifiers, maps the textual key name to an enum value, and registers the callback with GTK.

---

## Change Notification System

A registration-based notification mechanism that allows subsystems to be notified of state changes without tight coupling:

### `RegisterChangeNotification(changeNotificationCallBack_t action)` — Register a callback

Stores the function pointer in a global array (max 40 callbacks). Called by subsystems like draw routines, menus, or dialogs when they need to refresh themselves after certain events.

### `DoChangeNotification(long changes)` — Invoke all registered callbacks

Iterates over all registered callbacks and invokes each with the current change flags bitmask. Used after operations that may affect multiple views (e.g., after undo/redo, after layer change, etc.).

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `MyMalloc()` / `MyRealloc()` / `MyFree()` | Guarded memory allocation/deallocation with overrun detection |
| `memdup()` | Duplicate a byte block into guarded storage |
| `MyStrdup()` | Duplicate a C string with guards |
| `ConvertToEscapedText()` | Escape special characters for CSV/text output |
| `ConvertFromEscapedText()` | Unescape text back to original form |
| `AbortMessage()` | Format an error/abort message into static buffer |
| `AbortProg()` | Fatal error handler with save offer |
| `ParseMessage()` | Parse localized string; extract columns for info bar |
| `InfoMessage()` / `ErrorMessage()` | Display messages in the info bar (with/without beep) |
| `NoticeMessage()` / `NoticeMessage2()` | Show confirmation dialog with custom buttons |
| `Confirm()` | Save-before-close confirmation handler |
| `DoQuitAfter()` / `DoQuit()` | Quit request handling with cleanup and save offer |
| `DoClearAfter()` / `DoClear()` | Clear layout operation (reset + confirm) |
| `MapWindowToggleShow()` / `MapWindowShow()` | Toggle/show map window |
| `wShow()` / `wHide()` | Show/hide arbitrary windows; maintain menu list |
| `CloseDemoWindows()` | Close all temporary/playback windows |
| `DefaultProc()` | Window event handler stub (close, etc.) |
| `SetAccelKeys()` | Register keyboard shortcuts from user preferences |
| `RegisterChangeNotification()` / `DoChangeNotification()` | Callback registration system for state change notifications |

---

## Notes

- Guard bytes are used to detect memory corruption. The patterns `0xDEADBEEF` (before) and `0xAF00BA8A` (after) make it easy to spot overrun/underflow via debugger inspection or by checking guard values in logs.
- The `-m` flag on the command line toggles between using `MainRedraw()` (full redraw every frame) and `TempRedraw()` (partial redraw only when dirty). This is useful during development for performance debugging.
- The checkpoint system (`OfferCheckpoint()`, `LoadCheckpoint()`) allows resuming work after a crash by loading a previously saved state from disk.
