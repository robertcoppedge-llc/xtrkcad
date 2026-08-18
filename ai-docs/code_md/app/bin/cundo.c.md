# cundo.c — Undo/Redo System

## Overview

`cundo.c` implements XTrkCad's **undo/redo** system, enabling users to revert changes by restoring previously saved snapshots of tracks and other objects. The design is a classic "transactional" approach: each modification operation (create, modify, delete) records its effect in an expandable ring buffer (`undoStream`), which can be replayed when `Undo()` is invoked.

Key architectural decisions:
- **Two streams**: `undoStream` holds the history of changes; `redoStream` holds the inverse operations for redo.
- **Circular buffer with wrap-around handling**: When a new transaction begins, old entries are recycled after their undo records have been processed (deleting freed tracks).
- **Defer-free mechanism**: For compound objects containing embedded pointers (e.g., title strings), the old memory is not immediately freed but held in `deferFree_da` until the undo stack entry is expired. This prevents use-after-free bugs when an object's fields are restored during undo.

---

## Key Data Structures

### `undoStack_t` — Transaction Entry

```c
typedef struct {
    wIndex_t modCnt;       // Number of ModifyOp records written to stream
    wIndex_t newCnt;       // Number of New tracks added in this transaction
    wIndex_t delCnt;       // Number of DeleteOp records
    wIndex_t trackCount;   // Snapshot: total number of tracks at start

    track_p newTrks;      // Linked list of newly created tracks (not yet linked)

    uintptr_t undoStart;  // Pointer into undoStream where this transaction starts
    uintptr_t undoEnd;    // Pointer marking the end of recorded operations

    uintptr_t redoStart;  // Pointer into redoStream for this entry's inverse ops
    uintptr_t redoEnd;    // End marker in redoStream

    BOOL_T needRedo;      // Flag: has this transaction been undone? (has it been replayed into redo?)

    track_p *oldTail;     // Pointer to last track pointer before unlinking new tracks
    track_p *newTail;     // Pointer to the tail of newly created tracks list

    char *label;          // Balloon help text: "Create Track", "Move Track", etc.

    dynArr_t deferFree_da;// List of memory blocks that must not be freed until this entry expires
} undoStack_t, *undoStack_p;
```

### `stream_t` — Expandable Ring Buffer

A dynamically allocated ring buffer built from linked blocks:

```c
#define BSTREAM_SIZE (4096)  // Each block holds 4KB of record data

typedef struct {
    dynArr_t stream_da;   // Array of `streamBlocks_p` pointers
    long startBInx;       // Logical start index into the array
    uintptr_t end;        // Absolute offset (byte count from beginning) marking write boundary
    uintptr_t curr;       // Current write position
} stream_t;
```

Records are written sequentially. When `end` reaches the array limit, new blocks are appended via dynamic allocation (`dynArr_append`). This avoids circular buffer modulo arithmetic and keeps memory layout simple.

---

## Core Functions

### `UndoStart(char * label, char * format)` — Begin a Transaction

This is called by every command that modifies the track database:

```c
void UndoStart(
    char   * label,      // Balloon help text (e.g., "Create Track")
    char  * format,      // Log message with %d/%s placeholders
    ...                  // Arguments for log formatting
);
```

Behavior:
1. Sets `undoActive = TRUE` and resets the current stack entry's counters (`modCnt`, `newCnt`, `delCnt`).
2. If this is a *fresh* transaction (first since last undo), it unlinks all previously "New" tracks from the global track list at `to_first` → `to_last`.
3. Calls `ClearStream(&redoStream)` — redo stream is purged for every new transaction.
4. Sets `undoStack[undoHead].undoStart = undoStream.end`, preparing to write records.
5. Increments `doCount` (number of active transactions).

If the circular buffer wraps (`doCount == UNDO_STACK_SIZE`), it calls `DeleteInStream()` to unlink and free all tracks marked as deleted, then frees them from memory.

---

### `UndoModify(track_p trk)` — Record a Modified Track

Called when a track is edited (attributes changed, segments modified). It:
- Checks that the track has been flagged with `trk->modified = TRUE` by the caller.
- Writes a record to `undoStream` containing the *entire* track object (`track_t`) plus its endpoint array and extra data block. The operation tag is `ModifyOp`.
- Increments the current transaction's `modCnt`.

---

### `UndoDelete(track_p trk)` — Record a Deleted Track

Called when a track is removed from the database:

```c
BOOL_T UndoDelete( track_p trk );
```

Logic:
1. If the track was previously **Modified** in this transaction, it changes that record's op tag from `ModifyOp` to `DeleteOp`. The stored snapshot already contains the pre-deletion state — undoing will restore it as a modified object rather than re-creating it anew.
2. If the track was *not* modified and is also not "New", writes a fresh `DeleteOp` record with the current track contents (for restoration).
3. If the track **was New** (created in this transaction), simply removes it from the linked list at its insertion point (`newTrks`). No stream record is needed — the track was never fully committed to the database.

After recording, sets `trk->deleted = TRUE` so other code branches can skip processing deleted tracks.

---

### `UndoNew(track_p trk)` — Record a Newly Created Track

For tracks created in this transaction:
- Sets `trk->new = TRUE`.
- Links into the head of the current entry's `newTrks` list (via `us->newTrks`).

This allows quick cleanup when wrapping around or aborting. The track remains linked at the tail (`to_last`) until undo is executed.

---

### `UndoEnd(void)` — Commit Transaction

Called by every command after successfully completing its action:
- Calls `AttachTrains()` if car attachments were modified during this transaction.
- Calls `UpdateAllElevations()` to recompute track elevations (e.g., after pier modifications).
- **Note:** It does *not* call the generic undo/redo UI — that is done by separate hotbar buttons or menu commands.

---

### `UndoUndo(void)` — Execute Undo

The main undo operation:
1. Decrement `doCount`, increment `undoCount`.
2. Move to the previous stack entry (`DEC_UNDO_INX(undoHead)`).
3. Re-link all "New" tracks into the global track list at their recorded insertion points.
4. Read each record from `undoStream`:
   - **ModifyOp**: Restore the modified track's state (replaces current with snapshot).
   - **DeleteOp**: Restore a deleted track, relink it into the database.
5. If a track was New and got deleted in this undo transaction, delete the old entry to avoid double-free.
6. Call `UpdateAllElevations()` again if needed.

---

### `UndoRedo(void)` — Redo an Undone Transaction

The inverse of Undo:
1. Move forward in stack (`INC_UNDO_INX(undoHead)`).
2. Read records from `redoStream` (which was populated by Undo) and replay them as modifications or creations.
3. Re-link the "New" tracks that were previously restored during undo.

---

### `DeleteInStream(stream_p stream, uintptr_t start, uintptr_t end)` — Free Deleted Tracks

When a transaction is recycled (stack wrap-around), this function:
- Scans records from `start` to `end`.
- For each record tagged with `op == DeleteOp`, frees the track object and decrements global track count.
- The freed tracks are unlinked from the database and no longer referenced anywhere.

---

### `SetDeleteOpInStream(stream_p stream, uintptr_t start, uintptr_t end, track_p trk)` — Convert Modify → Delete

Used when a track is deleted *after* it was modified in a transaction. Since the modification record already exists in `undoStream`, we must change its op tag from `ModifyOp` to `DeleteOp`. The stored snapshot is still valid — undoing will restore the pre-deletion state of that object.

---

### `UndoDeferFree(void * p)` — Postpone Freeing Embedded Memory

For compound objects (e.g., structures with title strings, turnouts with pier names), the old memory holding the previous string value must not be freed immediately when a modification occurs. Instead:
1. Store the pointer in `deferFree_da` of the current undo stack entry.
2. When that entry is recycled, all pointers are walked and each block is freed exactly once.

This prevents "double free" crashes where two different objects share the same string memory but only one expects it to be freed at a given time.

---

### `UndoResume()` / `UndoSuspend()` — Enable/Disable Recording

These functions allow temporarily disabling undo recording (e.g., during certain initialization operations or batch imports). When suspended, modifications are not recorded and no stream entries are written.

---

## Summary

| Aspect | Detail |
|--------|--------|
| Undo stack size | 10 entries (`UNDO_STACK_SIZE`) |
| Stream block size | 4096 bytes (`BSTREAM_SIZE`), allocated dynamically as needed |
| Operation types | `ModifyOp`, `DeleteOp`, and implicitly `New` (tracked via linked list) |
| Memory safety | Defers freeing of embedded pointers until the entire transaction is expired; uses `IsTrackDeleted()` guards everywhere. |
