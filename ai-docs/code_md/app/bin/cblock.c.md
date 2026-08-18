# cblock.c — Block Objects (Layout Control: Track Blocks)

## Overview

`cblock.c` implements **Block** objects in XTrkCad. A *block* is a logical grouping of track segments that defines a "section" of the layout for purposes such as:
- **Turnout switching control** — determining which turnout positions are valid given the current route selection.
- **Signal placement and logic** — signals are placed at block boundaries and controlled by interlocking rules.
- **Route locking** — ensuring only one path is allowed between two points (for safety).

Blocks are defined geometrically as a connected region of track bounded by turnouts or switches, with endpoints at the turnout diverging points. The code handles:
- Block creation from selected turnouts/switches
- Block detection and numbering
- Block management commands (create, delete, edit)
- Integration with the container manager system for interactive editing

---

## Key Data Structures

### `blockData_t` — Block Extra Data

```c
typedef struct blockData_s {
    extraDataBase_t base;        // Generic header (index, layer)
    trkSeg_p ep0;               // Segment at endpoint 0
    trkSeg_p ep1;               // Segment at endpoint 1
    track_p turnout[2];         // Turnout(s) defining this block
    BOOL_T deleted;             // Deletion flag
} blockData_t, *blockData_p;
```

- **`ep0`, `ep1`** — The two endpoints of the block (turnout diverging points). These are defined by the track segments immediately after each turnout.
- **`turnout[2]`** — Points to the turnouts that define this block. A single turnout can have multiple blocks attached if it has more than one switch motor (multi-path turnout).

---

### `blockInfo_t` — Block Definition Record

```c
typedef struct {
    char  *title;              // Human-readable name ("Block A", "Switching Yards")
    BOOL_T deleted;            // Deletion flag
    int    segCnt;            // Number of segments in the block outline
    trkSeg_t *segs;          // Array of segment descriptors
    coOrd  orig;              // Bounding box origin
    coOrd  size;              // Bounding box dimensions
    BOOL_T flipped;           // Horizontal flip flag (for mirrored blocks)
    wIndex_t scaleInx;        // Layout scale index for this block definition
} blockInfo_t, *blockInfo_p;
```

This is analogous to `turnoutInfo_t` — it stores reusable block definitions loaded from parameter files or user-defined library entries. Each entry has:
- A title (e.g., "Switching Yards Block")
- Scale compatibility (`scaleInx`)
- Segment count and pointer to the segment array defining the block's outline

---

### `extraDataBlock_t` — Generic Compound Extra Data for Blocks

Blocks inherit from the compound track type, so they also contain:

```c
typedef struct extraDataCompound_s {
    extraDataBase_t base;
    coOrd orig;                    // Global origin for transform
    ANGLE_T angle;                 // Rotation around origin
    BOOL_T flipped;                // Mirrored horizontally?
    BOOL_T ungrouped;             // Segments individually selectable?
    int split;                     // Split segment index (multi-part blocks)
    char *descriptionOrig;        // Original description offset
    coOrd descriptionOff;         // Offset to label position
} extraDataCompound_t, ...;
```

---

## Core Functions

### `BlockCreate(track_p trk)` — Create a New Block Track Object

Creates a new track of type `T_BLOCK`. It:
1. Calls `NewTrack()` with type `T_BLOCK` and an appropriate size for the extra-data block.
2. Initializes fields to zero.
3. Marks it as deleted (`deleted = TRUE`) so it won't be drawn until populated.

---

### `BlockFind(track_p t)` — Find Associated Blocks for a Turnout

Given a turnout track, finds all blocks attached to it by scanning the doubly-linked list of blocks whose `turnout` pointer references this turnout. Returns a pointer to the matching extra-data block or NULL if none found. Used when detecting route changes to update interlocking state.

---

### `BlockDelete(track_p trk)` — Free a Block Object

Frees dynamically allocated memory (name string, segment array) and removes the block from its doubly-linked list (`first_block` ↔ `last_block`). Updates head/tail pointers. Called when a turnout is deleted or the block definition is removed from memory.

---

### `WriteBlock(track_p t, FILE *f)` — Serialize Block to File Format

Writes a line such as:
```text
BLOCK 42 7 "Switching Yards" (0.1) "Block A"
```

Format: `BLOCK <index> <turnout_index> "<name>" (<scale>) "<title>"`

The scale is stored as a float multiplier relative to the base layout scale (e.g., `(0.1)` means 1/10th scale). If no special block definition exists, a generic "Block" title is used with scale `*`.

---

### `ReadBlock(char *line)` — Deserialize Block from File Format

Parses a line starting with `"BLOCK"`. Extracts:
- Turnout index (to locate the associated turnout)
- Name string
- Scale multiplier (as a float, or "*" for generic)
- Title (human-readable label)

Creates a new track object of type `T_BLOCK` and stores it in a global array (`block_da`). The block is marked as deleted until its endpoints are resolved.

---

### `ResolveBlock(track_p trk)` — Resolve Turnout References After Load

When blocks are loaded from file, the turnouts they reference may not yet exist (because turnout records are loaded later). This function:
1. Checks if the associated turnout exists via `FindTrack()`.
2. If found, resolves any flip/angle transforms and re-computes the bounding box.
3. If not found, leaves the block in a deleted state until its turnout is created.

---

### `DeleteBlocks(track_p t)` — Delete All Blocks Attached to a Turnout

When a turnout is deleted (or its associated switch motors are removed), this function iterates over all blocks whose `turnout` pointer references that turnout, deletes each block, and finally calls `FreeTrack()` on the track record. Used during cleanup to avoid memory leaks.

---

### `DrawBlock(track_p t, drawCmd_p d, wDrawColor color)` — Render a Block Track

Renders the block outline using its segment array. Since blocks are defined as closed polylines (or open chains depending on layout), each segment is drawn at its appropriate position relative to the block's origin and angle. The fill style depends on whether the block is "active" or in a conflicting state (e.g., two trains occupying the same block simultaneously).

---

### `BlockMgmLoad(void)` — Register Blocks with Container Manager

Called during initialization, this iterates over all loaded blocks and registers each with the container manager system:
- Creates an icon handle (`wIconCreatePixMap()`) from a bitmap resource.
- Calls `ContMgmLoad()` to wire up mouse-click, drag, and edit commands for each block track object.

This enables users to click on a block in the drawing to open its properties dialog.

---

### `InitCmdBlock(wMenu_p menu)` — Register Block Menu Command

Adds a "Block" button to the command menu (with a building/padding icon). Also registers the parameter group (`blockPG`) for any user-defined parameters associated with blocks.

---

### `InitTrkBlock(void)` — Initialize Block Track Type

Finalizes initialization:
- Calls `InitObject()` to register `T_BLOCK` as a valid track type in the command system.
- Finds or creates a log category named "block".
- Initializes a dynamic array for per-track block information (`blockTrk_da`).
- Sets up global pointers (`first_block`, `last_block`) for doubly-linked list management.

---

### `BlockMgmProc(int cmd, void *data)` — Block Management Command Dispatcher

The central event handler registered with the container manager. Handles:
- **`CONTMGM_CAN_EDIT`** → always returns TRUE (blocks can be edited)
- **`CONTMGM_DO_EDIT`** → opens a properties dialog for the selected block
- **`CONTMGM_CAN_DELETE`** → always returns TRUE (blocks can be deleted)
- **`CONTMGM_DO_DELETE`** → calls `DeleteTrack()` on the track record
- **`CONTMGM_GET_TITLE`** → formats a title like `"Block A"` for display in lists

---

## Usage Flow

1. **During initialization**, `InitTrkBlock()` is called, which registers `T_BLOCK` as a valid track type and initializes bookkeeping structures.
2. When the user selects turnouts that define a block region, `BlockCreate()` allocates a new track record.
3. The block's segment endpoints are computed (based on the turnout diverging points) and stored in the extra-data block.
4. During rendering, `DrawBlock()` draws the polyline outline.
5. When editing or deleting is requested via mouse interaction, `BlockMgmProc()` dispatches to the appropriate handler.
6. On file load, blocks are deserialized via `ReadBlock()` and later resolved against existing turnouts via `ResolveBlock()`.

---

## Summary Table

| Function | Purpose |
|----------|---------|
| `BlockCreate()` | Allocate a new block track record |
| `BlockFind()` | Find all blocks attached to a given turnout |
| `BlockDelete()` | Free memory and unlink from doubly-linked list |
| `WriteBlock()` | Serialize block definition to file |
| `ReadBlock()` | Deserialize block from file format |
| `ResolveBlock()` | Resolve turnout references after loading |
| `DeleteBlocks()` | Delete all blocks attached to a deleted turnout |
| `DrawBlock()` | Render block outline on canvas |
| `BlockMgmLoad()` | Register blocks with container manager system |
| `InitCmdBlock()` | Register menu button and parameter group |
| `InitTrkBlock()` | Final initialization, register track type |
| `BlockMgmProc()` | Management command dispatcher (edit/delete) |

---

## Notes

- Blocks are a higher-level concept than simple turnouts: they represent *regions* of track bounded by turnout diverging points. A single turnout can have multiple blocks attached if it has two or more switch motors (multiple paths).
- The `deleted` flag is used extensively — newly created blocks start as deleted and remain so until their endpoints are resolved (which happens when the associated turnouts exist in the layout).
- Blocks support a flip transform, allowing mirrored block definitions to be reused across different sides of a layout.
