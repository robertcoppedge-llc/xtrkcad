# cundo.c — Undo/Redo System with Extra Data Support

## Overview

`cundo.c` implements a **ring-buffer undo/redo system** for XTrkCad that integrates with the extra data storage framework. Each track can hold arbitrary typed data (via `extraDataBase_t`) and all modifications to those fields are automatically recorded in the undo stream, enabling full-featured undo/redo even when changing custom properties like motor names or turnout configurations.

---

## Core Design: Circular Transaction Stack + Expandable Streams

### The Undo Stack (`undoStack[]`)

A circular buffer of 10 `undoStack_t` entries (configurable by `UNDO_STACK_SIZE`). Each entry tracks a transaction boundary:

```c
typedef struct {
    wIndex_t modCnt;      // Number of ModifyOp records in this transaction
    wIndex_t newCnt;      // Number of New tracks added since last UndoStart()
    wIndex_t delCnt;      // Number of DeleteOp records (for freeing deleted objects)
    wIndex_t trackCount;  // Total number of tracks when transaction started
    track_p newTrks;      // Head of list of newly created tracks in this transaction
    uintptr_t undoStart;  // Byte offset where this transaction's records begin in the stream
    uintptr_t undoEnd;    // End marker for this transaction in the stream
    uintptr_t redoStart;  // Where redo records would start (cleared on new UndoStart)
    uintptr_t redoEnd;    // Redo end marker
    BOOL_T needRedo;      // TRUE if there are redoable operations available
    track_p *oldTail;     // Pointer to old tail for splice operation during undo
    track_p *newTail;     // Pointer to new tail of modified tracks
    char *label;          // Human-readable label (e.g., "Change Track Name")
    dynArr_t deferFree_da;// Tracks that should be freed after this transaction is recycled
} undoStack_t, *undoStack_p;
```

### The Streams (`undoStream`, `redoStream`)

Both are expandable ring buffers implemented as a dynamic array of `streamBlocks_t` (8KB blocks each). They store serialized track records:
- Operation type (`ModifyOp` = 1 or `DeleteOp` = 2)
- Pointer to the original track object
- Full copy of the track structure
- Endpoints and extra data at the time of modification

The stream grows by allocating new blocks from a dynamic array, allowing indefinite growth (limited only by available memory). When the undo transaction wraps around its circular buffer, `TrimStream()` purges unreferenced old blocks.

---

## Key Functions

### `UndoStart(char *label, char *undoDesc)`

Begins a new undo transaction. Sets up:
- Transaction label and description (for UI display)
- Points to the current slot in the circular stack (`undoHead`)
- Initializes counters for modifications/new tracks/deletions within this transaction
- Clears redo state (`redoStart`, `needRedo` = FALSE)

**Parameters:**
- `label` — Short human-readable label (e.g., "Change Switch Motor")
- `undoDesc` — Full description shown in the undo dialog

---

### `UndoModify(track_p trk)`

Marks a track modification within the current transaction. Calls `WriteObject()` to serialize:
1. Operation type (`MODIFYOP`)
2. The entire track pointer and its contents (including all extra data)
3. A copy of any custom extra data stored via `StoreTrackData()`

The modified track pointer is written into the undo stream for later restoration.

---

### `UndoDelete(track_p trk)`

Marks a track for deletion. Sets the `.delete` flag on the track structure and records it as a `DELETEOP` in the stream. When the transaction wraps around (is recycled), deleted tracks are freed via `FreeTrack()`.

---

### `UndoEnd(void)`

Finalizes the current transaction:
- Writes final markers to the undo/redo streams
- Marks all modified tracks with their respective tail pointers for splice operations
- Calls `UndoClear()` if needed to reset state

---

### `UndoNew(track_p trk)`

Registers a newly created track. The new track is added to the transaction's linked list (`newTrks`) so that when undoing, it can be spliced out of the main track list (restoring the pre-new state).

---

### `UndoFail(char *cause, uintptr_t val, char *file, int line)`

An assertion-failure helper. When an internal invariant is violated (e.g., overrunning a stream buffer), it:
1. Displays an error dialog to the user with file/line information
2. Appends a detailed log dump to `undoTraceFile` (a text file in the working directory) containing:
   - Current stack state for all 10 circular entries
   - Stream contents as hex dumps
   - Record buffer content
3. Calls `UndoClear()` and returns `FALSE`

---

### `Rprintf(char *format, ...)`

A custom logging function that buffers formatted messages in a rotating ring buffer (8KB). When full, it flushes to stdout or stderr. Used for debugging undo stream contents.

---

### `ReadStream(stream_p s, void *ptr, int size)` / `WriteStream(stream_p s, void *ptr, int size)`

Low-level block-oriented I/O:
- **WriteStream**: Copies data into the current stream buffer. If the last block is full, allocates a new one and appends it to the dynamic array (`stream_da`). Handles partial writes across block boundaries.
- **ReadStream**: Reads from the stream at `curr`, advancing the cursor forward. Uses internal assertions (`UASSERT`) to detect overruns.

Both functions use zero-copy when possible (direct memory copy) but ensure proper alignment and block boundary handling.

---

### `TrimStream(stream_p s, uintptr_t off)` / `TruncateStream(stream_p s, uintptr_t off)`

- **TrimStream**: Removes blocks from the *beginning* of the stream (oldest records). Used when a transaction wraps around — its old records are no longer referenced and can be freed.
- **TruncateStream**: Shortens the stream at a given byte offset. Used to shrink the undo/redo buffers when they grow beyond necessary size.

---

## Extra Data Integration

The undo system integrates with the extra data framework (`custom.c`):

```c
static void StoreTrackData(track_p trk, void **buff, long *len)
{
    // Serialize all extra data fields of this track into a buffer
}
```

When `UndoModify()` is called after changing an extra field (e.g., renaming a switch motor), the entire track including its embedded extra data is copied to the undo stream. On redo, the original state is restored by deserializing from the stored copy.

---

## Circular Stack Mechanics

The 10-entry stack operates as follows:

| Scenario | Behavior |
|----------|----------|
| `undoHead` points to a valid slot with active transaction | Operations recorded in that slot's counters |
| Transaction completes (`UndoEnd()`) | Increment `undoHead`; if it wraps past `UNDO_STACK_SIZE-1`, reset to 0 and call `DeleteInStream()` for the old slot |
| `DeleteInStream(undoStack_p us)` | Iterates over all DeleteOp records in that transaction's stream range, freeing each deleted track. The entire undo block is then freed. |

This design ensures memory is reclaimed when older transactions are no longer needed while maintaining a bounded maximum footprint (10 × sizeof(`undoStack_t`) + stream blocks).

---

## Related Files

| File | Purpose |
|------|---------|
| `custom.c` / `custom.h` | Extra data storage framework (`StoreTrackData`, etc.) |
| `trackx.h` | Track list management (splicing new/deleted tracks in/out) |
| `trkendpt.h` | End point structures used by the undo system |
| `draw.h` | Drawing commands (used when reconstructing geometry on redo) |

---

## Usage Pattern

```c
UndoStart("Change Track Name", "Changed track name");
    trk = FindTrackByName("Mainline");
    if (!trk) { return; }
    
    // Modify a custom field
    switchmotorData_p sm = GET_EXTRA_DATA(trk, T_SWITCHMOTOR, switchmotorData_t);
    MyFree(sm->name);
    sm->name = MyStrdup(newName);

UndoModify(trk);
UndoEnd();
```

This records the modification. On Undo, the original name pointer is restored from the undo stream and `MyFree()`'d. The track is re-linked into its proper position in the list.
