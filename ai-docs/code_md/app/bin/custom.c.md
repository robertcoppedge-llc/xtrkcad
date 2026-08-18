# custom.c — Customization & Initialization Utilities

## Overview

`custom.c` is a small utility file that serves two main purposes:

1. **Global initialization** of all track types (curves, Bezier segments, straight tracks, easements, turnouts, turntables, text labels, drawing primitives, blocks, switch motors, signals, control sensors) and other subsystems (notes, car dialog).
2. **Locale string management** — constructs localized filename patterns for "Save As", "Open File", etc., and manages product version strings.

It is included in every XTrkCad build as the final piece of the `app/bin/` module compilation unit.

---

## Global Variables

| Variable | Type | Description |
|----------|------|-------------|
| `sProdName` | `char*` | Full product name (e.g., "XTrackCAD") |
| `sProdNameLower` | `char*` | Lowercase version ("xtrkcad") |
| `sProdNameUpper` | `char*` | Uppercase version ("XTRKCAD") |
| `sEnvExtra` | `char*` | Environment variable name for extra paths (`"XTRKCADEXTRA"`) |
| `sTurnoutDesignerW` | `char*` | Localized string: "XTrackCAD Turnout Designer" (for menu button) |
| `sAboutProd` | `char*` | About dialog string: `"XTrackCAD Version X.Y.Z"` |
| `sClipboardF` | `char*` | Clipboard file extension (`".clp"`) |
| `sParamQF` | `char*` | Parameter query file extension (`.xtq`) |
| `sUndoF` | `char*` | Undo log file extension (`.und`) |
| `sAuditF` | `char*` | Audit log file extension (`.aud`) |
| `sTipF` | `char*` | Tips file extension (`.tip`) |

Additionally, many pattern strings are defined for dialog file filters:

```c
char * sSourceFilePattern;   // "All XTrackCAD Files (*.xtc,*.xtce)|*.xtc;*.xtce|..."
char * sSaveFilePattern;     // Used in Save As dialogs
char * sImageFilePattern;    // For image files (*.* → any extension)
char * sImportFilePattern;   // Import files (*.xti)
char * sDXFFilePattern;      // DXF exchange format (*.dxf)
char * sSVGFilePattern;      // SVG export (*.svg)
char * sRecordFilePattern;   // Recording session files (*.xtr)
char * sNoteFilePattern;     // Note attachments (*.not)
char * sLogFilePattern;      // Application log files (*.log)
char * sPartsListFilePattern;// Parts list text files (*.txt)
```

---

## Core Functions

### `MakeWindowTitle(char *name)` — Construct Window Title String

A simple wrapper that copies a given string into a static buffer and returns it. Used to create localized titles like "Edit track" or "Delete signal". The static buffer avoids multiple allocations per window creation.

Example:
```c
ParamCreateDialog(&pg, MakeWindowTitle(_("Edit turnout")), ...);
// Resulting title: "Edit turnout" (localized via N_ macro)
```

---

### `InitCmdEasement(void)` — Initialize Easement Command Handler

Calls `EasementInit()` to set up a callback pointer (`easementP`). This is used for redirection so that one command can invoke another (for example, the easement command might delegate to a separate design tool). If `easementP` is non-NULL when called, it forwards the call.

---

### `DoEasementRedir(void *unused)` — Redirect Easement Command

A stub that checks if `easementP` has been set and invokes it with a NULL argument. Used for menu command redirection patterns in the application.

---

### `InitTrkCurve(void)` — Initialize Curve Track Type

Calls `InitObject(&curveCmds)` to register curve track segments as a valid object type, assigns their draw/edit/delete commands, and registers them with the undo system. This is called once during global initialization before any curves can be created.

---

### `InitTrkBezier(void)` — Initialize Bezier Track Type

Registers cubic Bézier track segments (used for smooth transitions between curve sections). The Bezier type stores control points that define the curve shape and is typically used in conjunction with corner rounding or smooth join operations.

---

### `InitTrkStraight(void)` — Initialize Straight Track Type

Registers straight segment objects. These are the simplest track primitives: a single line defined by two endpoints. Used for connecting curves, creating simple lines, or as building blocks for compound tracks.

---

### `InitTrkEase(void)` — Initialize Ease Segment Track Type

Eases (linear interpolation segments) bridge curved sections with different radii. They provide smooth transitions without explicit corner rounding, making the track appear continuous even when adjacent curve segments have different parameters. Registered alongside curves as a complementary segment type.

---

### `InitTrkCornu(void)` — Initialize Cornu Spiral Track Type

Registers Cornu spiral segments (Euler spirals) which provide gradual curvature change from zero to full radius over a specified length. Used for realistic transition zones between straight tracks and curved sections, simulating real-world track design practices.

---

### `InitTrkTurnout(void)` — Initialize Turnout Track Type

Registers turnout objects as valid track types. Turnouts are compound segments that include a switch motor object (stored in the same extra-data block) to control their physical position. They can be drawn, edited, deleted, and managed via container-manager hooks.

---

### `InitTrkTurntable(void)` — Initialize Turntable Track Type

Registers turntable track objects. These are special turnout-like structures that switch between multiple routes using a rotating platform rather than simple diverging rails. The type registration enables menu commands like "Create turntable" and integration into the undo/redo system.

---

### `InitTrkStruct(void)` — Initialize Structure Track Type

Registers structure objects (block groups, track pads, etc.) as valid track types. Structures are higher-level grouping constructs that can contain multiple segments and are managed through container-manager interaction.

---

### `InitTrkText(void)` — Initialize Text Label Track Type

Registers text labels as a track type. These are used for annotating the layout with arbitrary strings (e.g., station names, signals references, notes). They support positioning via bounding box computation, optional rotation, and flipping.

---

### `InitTrkDraw(void)` — Initialize Drawing Primitive Track Type

Registers drawing primitive types (freehand shapes, filled polygons) as track objects. These allow users to sketch custom graphics that overlay the track layout for visualization purposes.

---

### `InitCarDlg(void)` — Initialize Car Dialog System

Sets up dialog infrastructure used by car-related commands and dialogs (car placement, vehicle properties). This includes registering parameter groups and initializing internal state variables needed by the car subsystem.

---

### `InitCmdNote(void)` — Initialize Note Command Handler

Registers the note-taking command with the menu system so that users can insert annotation notes onto the layout. Notes are typically stored as separate files (`.not`) linked to their location via coordinates or bounding box references.

---

### `Initialize(void)` — Global Initialization of All Track Types

This is the central initialization function called once at application startup. It sequentially calls every `InitTrk*` and `InitCarDlg` / `InitCmdNote` function to:
- Register each track type with the undo/redo system (`UndoSystemRegister()`).
- Wire up draw, edit, delete commands for each type.
- Set up container-manager event handlers (edit/delete callbacks).
- Initialize bounding box computation routines.

It also calls `InitCustom()` and clears the global message buffer. Returns TRUE on success.

---

## File I/O Pattern Strings (`InitCustom`)

`InitCustom()` is called once during application startup to construct localized filename filter patterns used in file dialogs:

```c
sSourceFilePattern = _("All XTrackCAD Files (*.xtc,*.xtce)|*.xtc;*.xtce|"
                       "XTrackCAD Trackplan (*.xtc)|*.xtc|"
                       "Extended Trackplan (*.xtce)|*.xtce|*");
```

This pattern is used by GTK file selection dialogs to let users filter files by extension. The `_()` macro applies gettext translation, so the displayed text will appear in the user's locale language.

The function also initializes:
- Product version string for `About` dialog.
- Clipboard filename suffix (`.clp`).
- Parameter query file suffix (`.xtq`) — used to inspect saved parameter groups.
- Undo log suffix (`.und`) — stores undo operation records for replay.
- Audit log suffix (`.aud`) — tracks changes for collaboration or debugging.
- Tips file suffix (`.tip`) — user-friendly guidance documents.

---

## Cleanup (`CleanupCustom`)

Frees all dynamically allocated strings to avoid memory leaks at program termination. This is called during `main()` exit sequence after the undo system has been flushed and window resources have been released.

Note: The static buffers used by `MakeWindowTitle` are not freed — they are reused across multiple calls without needing allocation/deallocation.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `MakeWindowTitle()` | Build a localized title string from a raw name |
| `InitCmdEasement()` | Set up easement command redirect handler |
| `DoEasementRedir()` | Execute the stored easement callback (if any) |
| `InitTrkCurve()` | Register curve segments with undo system |
| `InitTrkBezier()` | Register cubic Bézier segments |
| `InitTrkStraight()` | Register straight segment objects |
| `InitTrkEase()` | Register ease interpolation segments |
| `InitTrkCornu()` | Register Cornu spiral segments |
| `InitTrkTurnout()` | Register turnout track types |
| `InitTrkTurntable()` | Register turntable track types |
| `InitTrkStruct()` | Register structure grouping objects |
| `InitTrkText()` | Register text label track objects |
| `InitTrkDraw()` | Register drawing primitive track objects |
| `InitCarDlg()` | Set up car placement dialog system |
| `InitCmdNote()` | Register note-taking command handler |
| `Initialize()` | Call all initialization routines in order |
| `InitCustom()` | Initialize localized file filter patterns and version strings |
| `CleanupCustom()` | Free dynamically allocated locale strings |

---

## Usage Flow

1. **At application startup**, after the undo system is initialized, `Initialize()` is called from the main program entry point.
2. This calls every track type initializer in a fixed order (curve → Bezier → straight → ease → Cornu → turnout → turntable → structure → text → drawing → block → switch motor → signal → control sensor → car dialog → note).
3. After all types are registered, `InitCustom()` runs to set up localized strings for file dialogs and about screens.
4. Throughout the application lifetime, users can create/edit/delete tracks of any of these types via menu commands or hotbar buttons wired through the undo system.
5. On exit, `CleanupCustom()` is called to free memory before the program terminates.

---

## Notes

- The `InitTrk*` functions are not reentrant — they assume a single-threaded initialization context and do not guard against concurrent calls. This is safe because `Initialize()` is invoked exactly once from `main()`.
- Each track type initializer returns the registered object index (e.g., `T_CURVE`, `T_Bezier`) which is used by `NewTrack()` to identify the extra-data size needed for that type.
- The order of initialization matters: earlier types may depend on later ones being registered first if they reference shared global structures.
