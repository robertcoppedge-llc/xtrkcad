# dcmpnd.c — Compound Refresh & Custom Management

## Overview

`dcmpnd.c` provides **compound object refresh** and **custom management** functionality. It handles:

- **Refreshing turnouts/structures**: When a turnout or structure definition changes (e.g., from a parameter file update), this module detects the change and replaces the in-memory geometry with the new definition
- **Custom management**: Loading custom-defined turnouts/structures for editing, renaming, copying to a permanent custom database, or deleting

The "dcmpnd" prefix indicates it's a data-access/utility layer that works with compound objects (turnouts/structures) — similar to how `dcar.c` handles car prototypes.

---

## Core Concepts

### Refresh vs. Replace

When XTrkCAD loads layouts from parameter files, the database contains *definitions* for turnouts and structures. When a user edits a turnout in the layout (moves it around), those modifications are stored in the track's extra-data structure (`extraDataCompound_t`). Later, if the parameter file changes, `RefreshCompound()` is called to update the track with new geometry from the definition — while preserving or discarding user modifications based on flags.

---

## Data Structures

### `extraDataCompound_t` (referenced here)

The per-track extra-data structure used throughout `compound.c`. Key fields:
- `segs`: Array of segment descriptors defining the track geometry
- `pathOverRide`, `pathNoCombine`: Flags for turnout path handling
- `title`: Human-readable label string
- `segCnt`: Number of segments

---

### `refreshSpecial_t` — Pending Refresh Candidate

```c
typedef struct {
    char * name;      // Turnout/structure title (for display in dialog)
    turnoutInfo_t * to;  // Pointer to the definition that should replace this track
} refreshSpecial_t;
static dynArr_t refreshSpecial_da;  // Global list of pending refreshes
```

Tracks turnouts/structures that have been loaded into memory but not yet successfully refreshed. They are queued here until a matching parameter file is found and applied.

---

### `renameManuf`, `renameDesc`, `renamePartno` — Rename Dialog Buffers

Static buffers holding the manufacturer, description, and part number strings when presenting a rename dialog to the user. The title string is reconstructed from these three parts via `ParseCompoundTitle()`.

---

## Core Functions

### `CheckCompoundEndPoint(trk, trkEp, to, toEp, flip)` — Validate End-Point Alignment

Checks whether endpoint `trkEp` of a track aligns with endpoint `toEp` of a turnout definition (`to`). It:
1. Gets the position of the track's endpoint in world coordinates
2. Rotates by `-angle` and translates by `-orig` to get local coordinates relative to the object origin
3. If `flip` is true, flips Y (mirrors across X-axis) — used when detecting a flipped turnout
4. Computes distance from that point to the definition's endpoint position
5. Checks if distance ≤ `connectDistance` (typically ~0.1 mm)
6. Compares angles: track angle minus object rotation minus definition angle must be within `connectAngle` tolerance

Returns TRUE if aligned, FALSE otherwise — in which case an error message is displayed to the user.

---

### `RefreshCompound1(trk, to)` — Apply a Single Definition to a Track

This is the core refresh routine for one track object:
1. Checks that the endpoint count matches between the track and definition
2. Iterates through all endpoints calling `CheckCompoundEndPoint()`; if any fail, checks flipped alternatives (for 2-endpoint turnouts like single-slip) or swapped alternatives (for 3/4-endpoint ones)
3. If alignment fails for all possibilities, returns FALSE with an error message
4. Calls `UndoModify(trk)` to mark the track as needing a redo entry in the undo stack
5. Frees the old segment array and allocates a new one from the definition (`to->segs`)
6. Sets the path override flags from the definition
7. Calls `SetPaths()` with paths from the parameter file (or NULL if not applicable)
8. If flipped, calls `FlipSegs()` to mirror all segments across an axis
9. Clears the SELECTED bit so the track isn't highlighted after refresh
10. Increments a counter (`refreshCompoundCnt`) — used for progress reporting
11. Calls `CloneFilledDraw()` on the new segments so they appear immediately

---

### `RefreshSpecialOk(junk)` / `RefreshSkip(junk)` / `RefreshDone(junk)` — Dialog Action Callbacks

A param dialog presents a list of available turnouts/structures that could replace the selected object. These callbacks handle the user's response:
- **OK**: The selection is accepted; the track is replaced (in the caller, after this returns).
- **Skip**: Discard the current candidate and show the next one in the list.
- **Done**: Exit without applying any replacement.

---

### `RefreshCompound(trk, junk)` — Main Refresh Dispatcher

This function is called for each selected track:
1. If the track is NULL (batch completion), displays a summary message showing how many tracks were refreshed and cleans up the pending list
2. For turnout/structure tracks: retrieves the extra-data struct; if it's in the refresh special queue, finds a matching definition from that queue or searches globally via `FindCompound()`, then calls `RefreshCompound1()`
3. If no match is found in any loaded parameter file and there are no pending candidates, opens a dialog listing all available turnouts/structures of the appropriate type (turnout vs structure) based on whether the track has endpoints
4. The user picks one from the list; if it matches by title, `RefreshCompound1()` is invoked

---

### `DoRefreshCompound(unused)` — Entry Point for Refresh Command

Called when the user selects a "Refresh" command from a menu or toolbar:
- Checks if frozen tracks (undo-disabled state) and returns early if so
- Starts an undo transaction with message "Refresh Compound"
- Calls `DoSelectedTracks(RefreshCompound)` to process each selected track
- Calls `RefreshCompound(NULL, FALSE)` to handle batch completion messaging
- Ends the undo transaction

---

### `CompoundCustomSave(f)` — Write Custom Turnouts/Structures to File

Iterates over all turnout and structure definitions in memory. For each one marked as `PARAM_CUSTOM` (user-defined) with non-zero segment count:
- Writes a header line: `"TURNOUT <scale> \"title\"" ` or `"STRUCTURE ..."`
- If the definition has custom info (`customInfo`), writes that under an indented "U" key
- Writes all paths, endpoints, and segments via `WriteCompoundPathsEndPtsSegs()`

This is used to export a user's custom turnouts/structures to a parameter file so they persist across sessions.

---

### `RenameOk(junk)` — Rename Dialog OK Handler

Reconstructs the full title from the three component fields (manufacturer, description, part number) and stores it in the track's extra-data structure. Hides the dialog and triggers a change notification so the UI updates.

---

### `CompoundCustMgmProc(cmd, data)` — Custom Management Command Dispatcher

This is the **main entry point** for custom management operations, called from the generic custom management framework (`custom.c`). It handles:
- **DO_COPYTO**: Writes the turnout/structure to the current "custom" file stream (for export)
- **CAN_EDIT**: Returns TRUE if the object can be edited (has endpoints and a `customInfo` field set)
- **DO_EDIT**: 
  - If no custom info exists, presents a rename dialog populating fields from the existing title
  - If there is already a `customInfo`, compares it with neighbors in the sorted array to find duplicates; if found, offers to merge by calling `EditCustomTurnout()` (from `cmodify.c`)
- **CAN_DELETE**: Always returns TRUE — custom objects can be deleted
- **DO_DELETE**: Sets segment count to zero, effectively removing the object from memory

---

### `CompoundCustMgmLoad()` — Load Custom Objects into Management System

Called when the application starts or after loading a parameter file. It iterates over all turnout and structure definitions:
- For each with `paramFileIndex == PARAM_CUSTOM` and non-zero segment count, it registers one entry in the custom management system using `CustMgmLoad()`
- This makes them available for rename/edit/delete operations via the UI

---

### `FindListItemByContext(listP, context)` — Find List Item by Pointer

A utility that scans a widget list for an item whose stored "context" pointer matches the given value. Used to find which row in a dialog corresponds to a particular turnout definition object so it can be selected or modified.

---

## Design Decisions & Tradeoffs

### Why Two-Stage Refresh (Queue + Dialog)?

The refresh system uses a two-stage approach:
1. First, track modifications are queued into `refreshSpecial_da` while the user is browsing parameter files
2. Only after all files have been loaded do we attempt to match them against pending candidates

This prevents race conditions where a file is unloaded before it can be used, and allows the UI to show "no matching turnout found" even when multiple files contain the same definition.

### Why Flip Checks in RefreshCompound1?

Turnouts can be placed in two orientations: normal or flipped (mirrored). The `flip` flag indicates whether we should mirror the geometry before comparing endpoint positions. For a single-slip turnout with only two endpoints, one orientation may not align but the other will — so both are tried.

### Why Search Neighbors When Editing Custom Info?

Custom objects stored in memory may be duplicated (loaded from multiple sources). The `customInfo` field acts as a primary key; if two adjacent entries share the same value, they represent duplicates that should be merged into one entry rather than kept separate. This keeps the database compact and prevents duplicate editing sessions on the same object.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CheckCompoundEndPoint()` | Validate alignment between a track endpoint and definition endpoint; handles flipping | track pointer, track endpoint index, definition pointer, definition endpoint index, flip flag |
| `RefreshCompound1()` | Apply one definition to replace a track's geometry | track pointer, turnoutInfo_t* definition pointer |
| `RefreshCompound(trk)` | Main dispatcher: tries queue first, then global search, then dialog | track pointer (NULL for batch finish) |
| `DoRefreshCompound(unused)` | Entry point from menu; starts undo transaction and processes all selected tracks | unused argument |
| `CompoundCustomSave(f)` | Write custom turnouts/structures to a text file | FILE* stream |
| `RenameOk(junk)` | Handle OK button on rename dialog; reconstruct title string | unused junk pointer |
| `CompoundCustMgmProc(cmd, data)` | Dispatch to edit/rename/delete/copy operations for custom objects | command ID, turnoutInfo_t* object pointer |
| `CompoundCustMgmLoad()` | Scan all loaded definitions and register custom ones with the management system | unused |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Refresh compound track geometry from updated parameter files; manage user-defined turnouts/structures (rename, edit, copy to permanent storage, delete) |
| **Domain** | Compound object lifecycle: initial load → user modification → refresh from source definition; custom objects stored in a special `PARAM_CUSTOM` bucket separate from parameter files |
| **Key concept** | Refresh is a *replacement* operation — the track's geometry is discarded and reloaded from a definition. The system tries multiple orientations (flipped) to find a match before reporting an error. Custom management allows users to edit or delete objects without needing a source parameter file. |
| **Main entry points** | `RefreshCompound()` — called per-track for refresh; `DoRefreshCompound()` — menu command; `CompoundCustMgmProc()` — custom management dispatch handler |
