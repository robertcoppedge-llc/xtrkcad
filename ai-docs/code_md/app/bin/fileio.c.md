# fileio.c — File Input/Output, Import/Export, and Clipboard Operations

## Overview

`fileio.c` is the **central I/O module** for XTrkCAD. It handles:

- Reading and writing track plan files (`.xtc`)
- ZIP archive support (`.xtce` packages with manifest.json)
- Clipboard copy/cut/paste operations using a disk-based clipboard
- File import/export of track segments between layouts
- Checkpoint/auto-save management for crash recovery
- Parameter file parsing (line-by-line format parser)

The module uses a **disk-backed clipboard** approach — instead of an in-memory clipboard, exports and cut/copies are written to a temporary file (`xtrkcad.clipboard`) on disk. This allows large track segments to be copied without memory pressure.

---

## External Globals

```c
EXPORT const char * workingDir;     /* Temporary working directory (for checkpoints, archives) */
EXPORT const char * libDir;         /* Library directory for example files */
EXPORT char * clipBoardN;           /* Disk-backed clipboard file path */
static coOrd paste_offset;          /* Accumulated offset for pasted geometry */
static coOrd cursor_offset;         /* Cursor-relative offset */

wBool_t bExample = FALSE;           /* When TRUE, loads an example layout (read-only) */
wBool_t bReadOnly = FALSE;          /* File is open but not writable */
```

---

## Core I/O: Load / Save Tracks

### `LoadTracks(int cnt, char **fileName, void *data)`

Loads a track plan from disk into the current session.

**Steps:**
1. Validate filename and check readability (`access(..., R_OK)`)
2. Open file in read mode → `paramFile`
3. Parse line-by-line using the format parser (see *Format Parser* below)
4. On EOF or error: close file, restore locale/user settings
5. Call `RecomputeElevations(NULL)` to compute elevations for all loaded tracks
6. Call `AttachTrains()` to link train definitions

**Key detail:** If the file is a ZIP archive (`.xtce`), it first unpacks into a temporary directory (`zip_in.<pid>`), reads the included `.xtc` file, and writes a `manifest.json` describing included dependencies (e.g., background images).

---

### `SaveTracks(int cnt, char** fileName, void* data)`

Saves the current layout to disk. Supports both classic `.xtc` files and ZIP archives (`.xtce`).

**For `.xtc`:**
- Writes a header with product name/version/date
- Calls internal write functions for layers, notes, tracks
- Sets `bReadOnly = FALSE` since file is now writable

**For `.xtce` (ZIP):**
1. Create temporary directory (`zip_out.<pid>`)
2. Copy the background image (if any) into `includes/` subdirectory
3. Write a `manifest.json` listing all included files
4. Call internal save to write the `.xtc` payload into the ZIP stream
5. Archive everything using `CreateArchive()`

---

### `DoSave(void *doAfterSaveVP)` / `DoSaveAs(...)`

User-facing wrappers that show a file chooser dialog (if no filename exists) and then call `SaveTracks()`. The optional callback (`doAfterSave`) is invoked after saving — typically used to trigger a redraw or notification.

---

### `LoadTracks()` / `DoLoad(void)`

Shorthand functions that create the file-chooser widget lazily on first use, then call the internal save/load function.

---

## Parameter File Format Parser

XTrkCAD uses a custom line-based format (similar to ASCII INI but more flexible). The parser is in `fileio.c` and referenced from `paramfilelist.c`.

### `GetNextLine(void)` — Get next non-empty, non-comment line

Reads one line from the open parameter file, strips trailing CR/LF, and returns it. If EOF is reached, closes the file and returns `NULL`.

---

### `InputError(char *msg, BOOL_T showLine, ...)`

If a parsing error occurs (unknown command, bad value, unexpected EOL), this function:
1. Constructs an error message with line number and filename
2. Displays a dialog asking whether to continue or stop
3. If stopped, closes the file and aborts loading

---

### `GetArgs(char *line, const char *format, ...)` — Format-based argument parsing

A **variadic format string** system for reading parameters from a line:

| Format char | Action |
|---|---|
| `0` | Read an integer and discard it |
| `X` | Set pointed-to int to 0 (skip) |
| `Y` | Set pointed-to float to 0.0 (skip) |
| `Z` | Set pointed-to long to 0L (skip) |
| `L` | Read integer into *int* pointer |
| `d` | Read integer into *int* pointer (duplicate of `L`) |
| `w` | Read width: parse as int, then divide by DPI if metric mode |
| `u` | Read unsigned long |
| `l` | Read signed long |
| `f` | Read double/float |
| `z` | Set value to 0.0 (skip) |
| `p` | Read a coordinate pair into a `coOrd` struct |
| `s` | Read a plain string (space-delimited, no quotes) |
| `q` | Read a quoted string (handles escaped quotes) — returns allocated memory |
| `c` | Return pointer to next non-space character or NULL |

Example:

```c
GetArgs( line, "VERSION %d %s", &version, versionString );
```

This reads an integer followed by a space-separated string into the two variables.

---

### `IsEND(char *sEnd)` — Check for END marker

Strips leading whitespace and checks if the remaining content matches a given keyword (`"END"` or `"END_TRK_FILE"`). Used to delimit multi-line blocks (e.g., notes, car text).

---

### `ReadMultilineText()` — Read until "END"

Used when reading note/car text fields. Reads lines until an `END` marker is found, concatenating them into a single string with newlines preserved.

---

## Clipboard Operations (Disk-Based)

Instead of using the system clipboard (which can be unreliable or have size limits), XTrkCAD writes/exported geometry to a temporary file (`xtrkcad.clipboard`) and reads it back when pasting.

### `EditCopy(void *unused)`

Writes all currently selected tracks to the clipboard file in `.xtc` format, including the version header. Sets `editStatus = FALSE`.

---

### `EditCut(void *unused)`

Calls `EditCopy()` then calls `SelectDelete()` (deletes the selected geometry). If copy fails, nothing is cut.

---

### `EditPaste( void *unused )` / `EditClone( void *unused )`

Reads the clipboard file into the current layout:
- Calls `ImportStart()`, suspends undo stack
- Reads from clipboard file using the same parser as normal load
- Applies an offset (20 units to avoid pasting exactly over existing geometry)
- Calls `ImportEnd(...)` which connects tracks, recomputes elevations if needed
- Restores undo and redraws

**Clone** adds an additional offset (`paste_offset += 20`) each time it's called so that successive clones form a chain of copies.

---

## Checkpoint / Auto-Save System

### `DoCheckPoint(void)`

Creates a backup checkpoint file (`.bkp`) if one doesn't already exist and the change count exceeds the threshold. Calls internal `SaveTracks()` to write the checkpoint to disk. If successful, renames the previous checkpoint to `.01` and `.02`, rotating up to five generations.

---

### `TryCheckPoint(void)`

Called periodically (every N changes or at idle intervals). Triggers a checkpoint save if needed, then optionally triggers an auto-save depending on settings (`autosaveChkPoints`).

---

### `CleanupCheckpointFiles(void)` / `CleanupTempArchive(void)`

Cleans up all temporary files (checkpoints, archive temp dirs) before program exit. This prevents leaving stray `.bkp` files or unpacked archive contents behind.

---

### `ExistsCheckpoint(void)` / `LoadCheckpoint(BOOL_T sameName)`

Called on startup to detect if the application was previously crashed and left a checkpoint file. If found, it is loaded automatically (optionally restoring the previous filename).

---

## Import / Export Functions

### `DoExportTracks(int cnt, char **fileName, void *data)`

Exports selected tracks to a new `.xtc` file on disk. Opens a file chooser (`exportFile_fs`) lazily and writes out the selection in full track plan format (including layers, notes, etc.).

---

### `DoImportObjects( void *unused )` / `DoImportModule( ... )`

Opens a file chooser to select an external `.xtc` file. The imported tracks are inserted into a **new layer** (`Module - <filename>`) so they don't interfere with existing geometry. The import is wrapped in an undo transaction labeled `"Import Tracks"`.

---

## Utility Functions

### `Copyfile(const char *fn1, const char *fn2)`

Simple file copy utility used for Windows compatibility (on Unix it just uses `fopen/fread/fwrite`). Copies a block at a time (`COPYBLOCKSIZE = 1024`) to avoid large memory allocations.

---

## Summary Table

| Function | Purpose | Notes |
|----------|---------|-------|
| `LoadTracks()` | Load track plan from file or ZIP archive | Handles `.xtc` and `.xtce`; unpacks archives on load |
| `SaveTracks()` | Save current layout to disk (`.xtc` or `.xtce`) | Writes manifest.json for ZIP files |
| `DoLoad()` / `DoSave()` / `DoSaveAs()` | User-facing wrappers with file choosers | — |
| `GetNextLine()` | Read next line from param file | Strips CR/LF, returns NULL at EOF |
| `InputError()` | Report parsing error and ask user to continue/stop | Closes file if stopped |
| `GetArgs()` | Parse a line using a format string (variadic) | Similar to scanf but simpler; handles quoted strings |
| `IsEND()` | Check for "END" keyword in current line | Used to delimit multi-line blocks |
| `ReadMultilineText()` | Read text until END marker | Used for notes/car descriptions |
| `EditCopy()` / `EditCut()` / `EditPaste()` | Copy/cut/paste using disk-based clipboard | Writes/reads from a temp file on disk |
| `DoExportTracks()` | Export selected geometry to a new file | Same format as full track plan files |
| `DoImportObjects()` | Import an external .xtc into current layout | Inserts into a new layer named after the source file |
| `DoCheckPoint()` / `TryCheckPoint()` | Save checkpoint backups for crash recovery | Rotates up to 5 generations |
| `CleanupCheckpointFiles()` | Remove all temp files on exit | Prevents stray `.bkp` and archive dirs |

---

## Design Notes

- **Disk-backed clipboard** avoids memory pressure and system clipboard compatibility issues. This is particularly useful when copying large track segments that might exceed the system clipboard size limit.
- **ZIP archives (`.xtce`)** allow packaging a layout with its background image and other dependencies in a single portable package. The `manifest.json` records included files, making it easy to rebuild or validate the archive contents.
- **Checkpoint rotation** keeps at most five generations of backup files. This trades disk space for protection against crashes between saves.
- The **parameter file parser** is intentionally simple and line-oriented — no complex data structures are needed during loading, only a single pass with state (current layer, current track index, etc.).
