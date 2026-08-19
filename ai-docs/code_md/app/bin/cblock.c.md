# cblock.c — Block (Compound Track) Objects

## Overview

`cblock.c` implements **blocks** in XTrkCad. A block is a compound track object that groups multiple individual tracks into a single logical entity with one controller/detector script. Blocks are used to model:

- Signal blocks (a group of tracks controlled by a single signal)
- Switched turnouts treated as a single unit
- Any collection of tracks that should share common attributes (name, controller, etc.)

## File Location

```
app/bin/cblock.c  (1040 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `common.h` | Core types (`track_p`, `coOrd`, `wAction_t`, etc.) |
| `compound.h` / `cundo.h` / `custom.h` | Undo, compound track iteration |
| `fileio.h` | File I/O utilities (`MyStrdup`) |
| `param.h` | Parameter dialog types (`paramData_t`, `wString_p`) |
| `track.h` | Track type system (`T_BLOCK`) |
| `trkendpt.h` | Endpoint accessors (`GetTrkEndPtCnt`, `SetTrkEndPoint`) |
| `common-ui.h` | UI dialogs, messages |
| `include/utf8convert.h` | UTF-8 ↔ native encoding conversion |

## Key Concepts

### Block Data Structure

```c
typedef struct btrackinfo_t {
  track_p t;       // Pointer to the constituent track
  TRKINX_T i;      // Its index (used for serialization)
} btrackinfo_t, *btrackinfo_p;

typedef struct blockData_t {
  extraDataBase_t base;
  char   *name;    // Block name (e.g., "Signal 4")
  char   *script;  // Controller script (e.g., a detector or signal routine)
  BOOL_T IsHilite; // Highlighting flag
  track_p next_block;  // Doubly-linked list pointer to next block
  wIndex_t numTracks;      // Number of constituent tracks
  btrackinfo_t trackList[]; // Variable-length array of member tracks
} blockData_t, *blockData_p;
```

The `next_block` field forms a doubly-linked list of all blocks in the layout.

### Block Types

Blocks are defined by the type code:

```c
EXPORT TRKTYP_T T_BLOCK = -1;  // Defined but not initialized here (done later)
```

## Command Interface (`blockCmds`)

The `trackCmd_t` table for `T_BLOCK`:

| Member | Function | Description |
|--------|----------|-------------|
| draw   | `DrawBlock` | Dummy — blocks are drawn by their constituents |
| distance | `DistanceBlock` | Finds closest point on any constituent track |
| describe | `DescribeBlock` | Describes name, script, length, endpoints |
| delete  | `DeleteBlock` | Removes the block and its name/script strings; unlinks from list |
| write   | `WriteBlock` | Writes `"BLOCK N "name" "script"\n\tTRK i\n"` lines to file |
| read    | `ReadBlock` | Reads a block from an `.xtp` file (parses until `\tEND_BLOCK`) |
| move    | `MoveBlock` | Empty — blocks are not moved individually |
| rotate  | `RotateBlock` | Empty — same reason |
| rescale | `RescaleBlock` | Empty — same reason |

## File Format

Blocks appear in `.xtp` files as:

```text
BLOCK <index> "name" "script"
	TRK <index1>
	TRK <index2>
	...
	END_BLOCK
```

The `ReadBlock()` function parses this format line by line, accumulating constituent track indices until it encounters the `END_BLOCK` marker. Each read block is linked into the global list via `next_block`.

## Core Functions

### `GetblockData(track_p trk)`

Retrieves the extra data pointer for a given track. Returns a `blockData_p` cast from the `extraDataBase_t` base structure attached to the track's private data.

### `DrawBlock(track_p t, drawCmd_p d, wDrawColor color)`

An empty function — blocks are not drawn directly; their constituent tracks draw themselves.

### `DistanceBlock(track_p t, coOrd *p)`

Finds the minimum distance from point `p` to any track inside the block:
- Iterates over all members of `trackList[]`
- For each non-NULL member, calls `GetTrkDistance()` and tracks the minimum.
- Stores the closest point back into `*p` for offset calculations.

Returns that minimum distance.

### `DescribeBlock(track_p trk, char *str, CSIZE_T len)`

Populates a description string used by layout commands (e.g., in the track list dialog). It:
1. Fetches name and script from the block data.
2. Formats a line like `"TRACK (4): Layer=3 Signal 4"`.
3. Sets up a `descData_t` array with fields for Name, Script, Length, End Pt 1, End Pt 2 — each marked as read-only (`DESC_RO`) because the block's geometry is immutable after creation.

### `DeleteBlock(track_p t)`

Removes the block from memory:
- Frees the name and script strings (which may point into the track's data area).
- Unlinks the block from the doubly-linked list of blocks by updating pointers in neighboring entries.
- Does **not** delete the constituent tracks — those remain as separate `track_p` objects.

### `WriteBlock(track_p t, FILE *f)`

Outputs a block to a file:
1. Duplicates the name and script into local buffers (UTF-8 conversion may be applied).
2. Writes the header line `"BLOCK <index> "name" "script"\n"`.
3. For each constituent track, writes `"	TRK <index>\n"`.
4. Writes a terminator line `"\tEND_BLOCK\n"`.

### `ReadBlock(char *line)`

Parses a block definition from an `.xtp` file:
1. Calls `GetArgs()` to extract the index, name, and script.
2. Advances `cp` past the rest of the header line and reads subsequent lines until `TRK` is encountered or `END_BLOCK` terminates the loop.
3. For each `TRK i` line, appends a new `btrackinfo_t` entry to `blockTrk_da`.
4. Constructs a new track object with type `T_BLOCK`, allocating extra space for the block data structure (size = base struct + `numTracks * sizeof(btrackinfo_t)`).
5. Sets up endpoints from the accumulated endpoint points collected during parsing.
6. Stores the name and script into the block's fields.
7. Links the new block into the global list (`first_block` / `last_block`).
8. The member track indices in `trackList[]` are stored as raw indices; they will be resolved later by `ResolveBlockTrack()`.

### `ResolveBlockTrack(track_p trk)`

Resolves the raw integer track indices in a block's `trackList[]` into actual `track_p` pointers:
- Iterates each entry.
- Calls `FindTrack(index)` to get the corresponding track object.
- Stores the result back into the member's `.t` field.

If any referenced track does not exist, an error message is shown and that entry remains NULL. This function is called during layout load so blocks can be reconstructed even if some constituent tracks were deleted or renamed since export.

### `blockCheckContiguousPath()`

Validates that all tracks inside a block form a single connected graph (i.e., they are not disjoint islands). It:
1. Iterates each member track and collects its unconnected endpoints into a temporary list (`TempEndPts`).
2. For every endpoint, checks whether it is within `connectDistance` / `connectAngle` of any other open endpoint (via `TempEndPtsAppend()` logic).
3. If any track has an endpoint with no neighbor found AND more than one track exists in the block, returns `FALSE` — the block would be "discontiguous".

This check prevents accidental creation of blocks that group unrelated, disconnected track pieces.

### `NewBlockDialog()`

Presents a dialog to collect tracks for a new block:
- Iterates all selected tracks; skips non-tracks and any already in another block.
- Increments `blockElementCount`. If zero tracks selected, shows an error.
- Creates the parameter dialog with two fields: "Name" (required) and "Script" (optional).
- Shows the window; when the user clicks **Ok**, `BlockOk()` is invoked.

### `BlockOk(void *junk)`

Handles the **Ok** button in the create dialog:
1. Collects all selected tracks again into `blockTrk_da`.
2. Checks for too many elements (> 128) — aborts with a message if so.
3. Calls `blockCheckContiguousPath()` — aborts with "Block is discontiguous!" if the graph is broken.
4. Creates a new block track object, copying all members' endpoints into it (so the block shares the same geometry).
5. Stores the name and script strings; builds the member track list.
6. Links the block into the global list (`first_block` → `last_block`).
7. Calls `UndoEnd()` to complete the transaction.

### `EditBlock(track_p trk)`

Opens an edit dialog for a selected block:
- Copies current name/script into buffers.
- Builds a comma-separated string of all member track indices.
- Shows a parameter dialog where the user can rename or change the script.
- On **Ok**, `BlockEditOk()` updates the block's strings and calls `UndoEnd()`.

### `DrawBlockTrackHilite()` / `CONTMGM_DO_HILIGHT` etc.

When the user requests highlighting (e.g., from a context menu "Highlight Block"), these functions:
1. Compute a bounding box that encloses all constituent tracks.
2. Draw a semi-transparent rectangle around the entire block in a light gray.

This visual cue helps the user see which tracks belong to a single logical unit.

### `BlockMgmLoad()` / `BlockMgmProc(int cmd, ...)`

Registers each block with the compound track management system via `ContMgmLoad()`. The command handler responds to:
- `CONTMGM_CAN_EDIT` → always returns TRUE (blocks are editable).
- `CONTMGM_DO_EDIT` → calls `EditBlock(trk)`.
- `CONTMGM_CAN_DELETE` → always returns TRUE.
- `CONTMGM_DO_DELETE` → deletes the block track object (which implicitly leaves its constituents orphaned as separate tracks).
- `CONTMGM_GET_TITLE` → builds a title string for tooltips or menus: `"Signal 4, T0, T3"` etc.

### `InitCmdBlock(wMenu_p menu)`

Registers the command with the menu system and creates the parameter dialog object (lazily on first call via static variables). The "Create Block" button is added to the main toolbar/menu.

## Design Notes

- Blocks are **immutable after creation**: you cannot add or remove constituent tracks once a block exists. You must delete the whole block and create a new one if you need to change membership.
- The `DistanceBlock()` function returns the minimum distance from a query point to any track inside the block, which is useful for collision detection or path-following algorithms that treat the entire block as a single obstacle.
- The discontiguity check ensures blocks represent coherent logical units (e.g., one signal controlling a consistent stretch of track) rather than arbitrary collections of unrelated pieces.
