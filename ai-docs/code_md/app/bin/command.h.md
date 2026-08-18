# command.h — Command System Constants & Declarations

## Overview

`command.h` defines the **action constants**, **command flags**, and **external function declarations** that form the foundation of XTrkCAD's event-driven command system. Every command handler (in `*.c` files) uses these action codes as their switch-case discriminants, making this header a shared vocabulary for all interactive code.

---

## Command Action Constants

### Mouse/Button Actions (`C_...`)

| Constant | Underlying Value | Description |
|----------|-----------------|-------------|
| `C_DOWN` | `wActionLDown` | Left mouse button pressed |
| `C_MOVE` | `wActionLDrag` | Left mouse button dragging |
| `C_UP` | `wActionLUp` | Left mouse button released |
| `C_RDOWN` | `wActionRDown` | Right mouse button pressed (context menu) |
| `C_RMOVE` | `wActionRDrag` | Right mouse button dragging |
| `C_RUP` | `wActionRUp` | Right mouse button released |
| `C_TEXT` | `wActionText` | Text input / special key event |
| `C_WUP` | `wActionWheelUp` | Mouse wheel up |
| `C_WDOWN` | `wActionWheelDown` | Mouse wheel down |
| `C_LDOUBLE` | `wActionLDownDouble` | Left-click double-tap |
| `C_MODKEY` | `wActionModKey` | Modifier key press (Shift, Ctrl, Alt) |
| `C_SCROLLUP` | `wActionScrollUp` | Scroll wheel up event |
| `C_SCROLLDOWN` | `wActionScrollDown` | Scroll wheel down event |
| `C_SCROLLLEFT` | `wActionScrollLeft` | Horizontal scroll left |
| `C_SCROLLRIGHT` | `wActionScrollRight` | Horizontal scroll right |
| `C_MDOWN` | `wActionMDown` | Middle mouse button pressed |
| `C_MMOVE` | `wActionMDrag` | Middle mouse dragging |
| `C_MUP` | `wActionMUp` | Middle mouse released |

### Special Actions (≥ 100)

These are internal actions used by command handlers for state transitions:

| Constant | Value | Description |
|----------|-------|-------------|
| `C_INIT` | 100+ | Command has been initialized |
| `C_START` | 101+ | User pressed the button / initiated a command |
| `C_REDRAW` | 102+ | Request to redraw without changing state |
| `C_CANCEL` | 103+ | Cancel (e.g., ESC key) — discard changes |
| `C_OK` | 104+ | Confirm / finish the current command |
| `C_CONFIRM` | 105+ | Show a confirmation dialog before committing |
| `C_LCLICK` | 106+ | Left-click (special handling for some commands) |
| `C_RCLICK` | 107+ | Right-click (context menu trigger) |
| `C_CMDMENU` | 108+ | Context menu action |
| `C_FINISH` | 109+ | Command completed successfully |
| `C_UPDATE` | 110+ | Refresh the command's internal state |

### Return Values from Command Handlers

Command handlers must return one of:

- **`C_CONTINUE (100)`** — The command is still active; process more events.
- **`C_TERMINATE (101)`** — The command has completed successfully; reset state.
- **`C_ERROR (102)`** — An error occurred; cancel the command and restore previous state.

---

## Command Button Flags (`IC_...`)

These flags are passed to `AddMenuButton()` when registering a command palette button:

| Flag | Hex Value | Description |
|------|-----------|-------------|
| `IC_STICKY` | `1<<0` | The button stays highlighted after activation (e.g., Select, Modify modes) |
| `IC_INITNOTSTICKY` | `1<<1` | Do not set the "sticky" flag automatically on init |
| `IC_CANCEL` | `1<<2` | Button acts as a cancel/escape handler |
| `IC_MENU` | `1<<3` | This button opens a submenu (popup menu) |
| `IC_NORESTART` | `1<<4` | Do not restart the command on repeated clicks |
| `IC_SELECTED` | `1<<5` | Button is pre-selected (shown in bold) |
| `IC_POPUP` | `1<<6` | Show a popup menu when clicked |
| `IC_LCLICK` | `1<<7` | Left-click activates the command |
| `IC_RCLICK` | `1<<8` | Right-click activates the command |
| `IC_CMDMENU` | `1<<9` | Button is part of a context (command) menu |
| `IC_POPUP2` | `1<<10` | Secondary popup behavior |
| `IC_ABUT` | `1<<11` | Button abuts another button in the palette |
| `IC_ACCLKEY` | `1<<12` | Has an accelerator key (e.g., F3) associated with it |
| `IC_MODETRAIN_TOO` | `1<<13` | Too many train modes already active |
| `IC_MODETRAIN_ONLY` | `1<<14` | Only one train mode allowed at a time |
| `IC_WANT_MOVE` | `1<<15` | The command wants to track mouse movement events |
| `IC_PLAYBACK_PUSH` | `1<<16` | For playback/recording systems |
| `IC_WANT_MODKEYS` | `1<<17` | Command responds to modifier keys (Shift, Ctrl, Alt) |
| `IC_POPUP3` | `1<<18` | Third-level popup behavior |

---

## Level Definitions (Obsolete)

```c
#define LEVEL0        0   /* Top-level commands */
#define LEVEL0_50     1   /* Sub-commands under a menu button */
#define LEVEL1        2   /* Deep sub-menu items */
#define LEVEL2        3   /* Deepest nesting level */
```

These are marked as **obsolete** in the header — the current codebase likely uses numeric constants directly or the `IC_...` flags instead.

---

## Global Variables (extern declarations)

| Variable | Type | Purpose |
|----------|------|---------|
| `buttonCnt` | `int` | Number of registered command buttons in the palette |
| `commandCnt` | `int` | Total number of active commands |
| `preSelect` | `long` | Tracks the previously selected button index (for sticky highlighting) |
| `rightClickMode` | `long` | Current right-click mode state |
| `commandContext` | `void*` | Per-command context pointer stored per button |
| `cmdMenuPos` | `coOrd` | Position of a popup/context menu when shown |

---

## Function Declarations (extern)

| Function | Signature | Description |
|----------|-----------|-------------|
| `GetCurCommandName()` | `const char * GetCurCommandName(void)` | Returns the name string of the currently active command (useful for logging/UI). |
| `IsCommandEnabled()` | `EXPORT bool IsCommandEnabled(long mode, long options)` | Checks whether a given command is enabled/disabled in its current state. |
| `EnableCommands()` | `void EnableCommands(void)` | Enables all commands after a reset or initialization sequence. |
| `GetCurrentCommand()` | `wIndex_t GetCurrentCommand(void)` | Returns the index of the currently active command (or -1 if none). |
| `Reset()` | `void Reset(void)` | Resets the global command state — clears sticky flags, resets contexts, etc. Called on ESC or when no command is active. |
| `DoCurCommand()` | `wBool_t DoCurCommand(wAction_t action, coOrd pos)` | Dispatches an event to the currently active command handler. This is the main entry point for all mouse events during a command session. |
| `ConfirmReset(BOOL_T canceling)` | `int ConfirmReset(BOOL_T canceling)` | Prompts the user if there are pending changes before resetting (used with sticky commands). Returns 0 to cancel, non-zero to proceed. |
| `DoCommandB(void *arg)` | `void DoCommandB(void *arg)` | A generic command dispatcher that looks up a button by its context pointer and calls its handler. Used for palette buttons and context menus. |
| `CommandEnabled(wIndex_t inx)` | `BOOL_T CommandEnabled(wIndex_t inx)` | Checks if the command at index `inx` is currently enabled (not disabled by state, mode locks, etc.). |
| `NUM_CMDMENUS` | `#define 4` | Number of top-level context menus supported. |
| `IsCurCommandSticky()` | `BOOL_T IsCurCommandSticky(void)` | Returns true if the current command is sticky (stays active until explicitly canceled). |
| `ResetIfNotSticky()` | `void ResetIfNotSticky(void)` | Resets the global command state only if no sticky command is currently active. |
| `CommandInit()` | `void CommandInit(void)` | Initializes the command system — registers all buttons, sets up event handlers, etc. Called early in startup. |

---

## Key Design Notes

- **Action codes are shared across all command handlers.** Every file that implements a command (e.g., `cmodify.c`, `cturnout.c`) includes or references this header and uses the same `C_*` constants as their switch-case discriminants.
  
- **Sticky commands** (`IC_STICKY`) remain active until the user explicitly cancels them (ESC, Cancel button) or completes them. Non-sticky commands auto-reset on CANCEL/ENTER.

- The **command context pointer** stored per-button allows each command handler to maintain its own data structure without global pollution. For example, `cmodify.c` uses a static `Dex` struct as part of the context passed through the command system.
