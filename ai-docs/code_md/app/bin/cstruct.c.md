# cstruct.c — Structure Objects (Layout Control: Track Structures)

## Overview

`cstruct.c` implements **Structure** objects in XTrkCad. A *structure* is a compound track segment that represents a bridge, overpass, or other 3D object placed on top of the track layout. Unlike simple curves or straight segments, structures are defined as collections of polylines (segments) drawn at arbitrary orientation and scale. They provide:

- **Visual overlay** — bridges, tunnels, buildings can be placed directly onto the track without interfering with normal track geometry.
- **Pier support objects** — vertical supports for overpasses that connect to real track endpoints via a pier-detection system.
- **Parametric library loading** — structures are loaded from external `.xtc` parameter files and displayed in a searchable list.

---

## Key Data Structures

### `turnoutInfo_t` — Structure Definition Record

```c
typedef struct {
    char *title;              // Human-readable name ("Overpass A", "Tunnel")
    wIndex_t scaleInx;        // Scale index for compatible layout gauge
    BOOL_T deleted;           // Deletion flag
    trkSeg_p segs;           // Pointer to segment array defining the structure outline
    int segCnt;              // Number of segments in the definition
    char *contentsLabel;     // Short label shown on hotbar ("Overpass", "Tunnel")
    turnoutInfo_t **next;    // Doubly-linked list for all loaded structures
} turnoutInfo_t, *turnoutInfo_p;
```

Structures are stored in a global dynamic array `structureInfo_da` and linked via the `**next` pointer into a doubly-linked list (head/tail maintained by `firstStructure` / `lastStructure`).

---

### `extraDataCompound_t` — Compound Extra-Data Header

All compound track types (curves, turnouts, structures) share this header:

```c
typedef struct extraDataCompound_s {
    extraDataBase_t base;   // Generic header: index, layer, visible flag, etc.
    coOrd orig;             // Transform origin for placement
    ANGLE_T angle;          // Rotation around origin
    BOOL_T flipped;         // Horizontally mirrored?
    BOOL_T ungrouped;      // Segments individually selectable?
    int split;              // Split segment index (multi-part objects)
    char *descriptionOrig;  // Original description text offset
    coOrd descriptionOff;   // Offset from orig to label position
    DIST_T descriptionSize; // Label bounding box dimensions
    struct {                // Special-purpose fields:
        BOOL_T special;     // Nonzero if a specialized structure type is used
        union {             // Union of possible special types
            struct {       // TOnormal — ordinary structure placement
                char *name;   // Name of the pier or support element
            } pier;
            struct {       // TOpierInfo — overpass with multiple piers
                int cnt;     // Number of pier definitions
                struct {
                    DIST_T height;   // Pier height above track
                    char *name;      // Human-readable name ("Pier 1", "Support A")
                } info[1];         // Variable-length array (dynamic)
            } pierInfo;
        };
    };
} extraDataCompound_t, ...;
```

The `special` union provides a mechanism for future extensions. Currently used for:
- **TOpier** — a single-pier overpass whose height is parameterized.
- **TOpierInfo** — multi-pier overpasses where each pier can have its own height and name.

---

### `structureData_t` — Generic Track Type Record

Structure track objects use the generic compound extra-data structure (no type-specific fields beyond the shared header). The actual geometry is always stored in a separate `turnoutInfo_p` pointer referenced via the undo system's type-specific data field.

---

## Core Functions

### `CreateNewStructure(char *scale, char *title, wIndex_t segCnt,
                        trkSeg_p segData, BOOL_T updateList)` — Allocate New Structure Definition

Creates a new structure definition record without attaching it to any track object:

1. Checks if `segCnt == 0` → returns NULL (no geometry defined).
2. Calls `FindCompound(FIND_STRUCT, scale, title)` to check for duplicates in the global array. If found, reuses the existing entry instead of allocating new memory.
3. Allocates a new `turnoutInfo_t` record and appends it to `structureInfo_da`.
4. Copies the segment pointers (via `memdup`) so modifications to the original do not affect the definition.
5. Calls `CopyPoly()` to create an independent copy of each segment's point array.
6. Calls `FixUpBezierSegs()` to ensure Bézier segments have valid control points.
7. Computes the bounding box via `GetSegBounds()`.
8. If defined with `REORIGSTRUCT` flag (defined but not used in current build), it re-centers the geometry to origin and flips coordinates — legacy code path.
9. Sets the param file index (`curParamFileIndex`) so that structures loaded from parameter files can be grouped by source file later.
10. Stores a "Custom Structures" label if loaded from the custom parameter group; otherwise uses `curSubContents`.
11. If `updateList` is TRUE, formats a title string (combining manufacturer, part number, description) and appends it to a list widget for display in the structure picker dialog.
12. Stores the bar scale factor (`barScale`) so that hotbar elements are drawn at the correct size relative to their definition scale.

Returns a pointer to the newly created structure entry or NULL if no new entry was needed (duplicate found).

---

### `StructureDelete(void *structure)` — Free a Structure Definition

Frees memory associated with a single structure definition:
- Frees the title string and segment array (`segs`).
- If the structure has special pier information, frees each pier's name strings individually. The union is tagged by `special` so that only the appropriate branch is executed.
- Calls `MyFree()` on the entire record to free the dynamically allocated block from `structureInfo_da`.

This function handles multiple pier definitions via a loop over the variable-length array of pier info records.

---

### `DeleteStructures(int fileIndex)` — Bulk Delete Structures from Parameter File

Removes all structure definitions that originated from a specific parameter file (e.g., after loading a new version or deleting a param group). It:
1. Scans the `structureInfo_da` array to find the contiguous block of entries whose `paramFileIndex` matches the given index.
2. Calls `StructureDelete()` on each entry in that block, counting how many were actually deleted.
3. Compacts the remaining valid entries by copying down from the end of the block into the gap left by deletions.
4. Updates `structureInfo_da.cnt` to reflect the new total number of structures.

This operation is safe because structure definitions are loaded contiguously in the array, and all entries sharing a parameter file index are guaranteed to be adjacent.

---

### `GetStructureCompatibility(int paramFileIndex, SCALEINX_T scaleIndex)` — Check Scale Compatibility

Determines whether structures from a given parameter file can be used with the current layout scale:
- If any structure in the file has an *exact* scale match (same gauge), returns `PARAMFILE_FIT`.
- If no exact match is found but some entries are within ±15% of the target scale, returns `PARAMFILE_COMPATIBLE` (scaled structures will be resized).
- Otherwise returns `PARAMFILE_NOTUSABLE` (structures should not be shown to the user).

This allows users to load structure libraries designed for different gauges and have them automatically filtered or flagged.

---

### `ReadStructureParam(char *firstLine)` — Parse a Structure Definition from File Format

Parses a line from a parameter file that defines a new structure:

```text
STRUCTURE "<scale>" "<title>" <segment data lines...>
```

The function:
1. Uses `GetArgs()` to extract the scale and title strings after the `"STRUCTURE "` prefix (10 characters).
2. Calls `ReadSegs()` which reads subsequent lines until a blank line or terminator, parsing each segment as either a straight line (`SEG_STRLIN`) or curve (`SEG_CRVLIN`, etc.). Segments are stored in the global `tempSegs_da` array and counted in `tempSegs_da.cnt`.
3. Calls `CreateNewStructure()` with the parsed scale/title/segments. If it returns NULL (duplicate title), aborts parsing by returning FALSE.
4. Reads a special-case field (`tempSpecial`). If non-empty, checks if it begins with `"PIER"`. If so, switches into pier-info mode and parses subsequent lines as `height name` pairs until the end of the structure block. Otherwise, signals an error via `InputError()`.
5. Reads an optional custom text string from `tempCustom` and stores a copy in `to->customInfo`.
6. Frees the allocated title string (it was copied into a static buffer by `CreateNewStructure`).

Returns TRUE on success; FALSE if any parsing field failed.

---

### `StructAdd(long mode, SCALEINX_T scale, wList_p list, coOrd *maxDim)` — Populate Structure Picker Dialog

This function is called from the structure selection dialog to populate its dropdown/list widget with all structures compatible with the current layout scale:
1. Iterates over every entry in `structureInfo_da`.
2. Skips entries whose parameter file index is not currently loaded (`IsParamValid()` returns FALSE).
3. Skips entries with zero segments (corrupted/incomplete definitions).
4. Calls `CompatibleScale(FIT_STRUCTURE, to->scaleInx, scale)` to determine if this structure fits the current gauge (exact or within 15% tolerance).
5. If compatible:
   - Calls `FormatCompoundTitle()` to build a display string combining manufacturer, part number, description fields.
   - Appends the formatted string to the provided list widget (`wListAddValue`).
   - Optionally tracks maximum width/height dimensions (passed via `maxDim`) for sizing purposes.
6. Returns the last compatible structure pointer (the one at the top of the list) so the caller can use it as the default selection.

The function is used to populate the dropdown in the "Place Structure" dialog, allowing users to pick from a library of pre-defined structures.

---

### `DrawStructure(track_p t, drawCmd_p d, wDrawColor color)` — Render a Structure Track Object

This function draws a single structure instance placed on the layout:
1. Retrieves the extra-data block via `GET_EXTRA_DATA()`.
2. Sets line style options (`DC_DASH`, `DC_DOT`, etc.) based on the `lineType` field. The dash pattern is applied to the entire polyline by setting the draw command's option flags.
3. Calls `DrawSegs()` with the stored origin, angle, and segment array. This draws each segment in turn at its transformed position.
4. Clears the `DC_NOTSOLIDLINE` flag so that subsequent drawing commands can use solid lines if needed (e.g., for pier supports).
5. If not in simple mode AND a label is configured (`labelWhen == 2` or `(labelWhen==1 && DC_PRINT)`), calls `DrawCompoundDescription()` to render the structure's description text below its bounding box.

This function draws structures on top of the normal track drawing layer (via `&screenDrawFuncs`) so they overlay correctly with curves and turnouts.

---

### `ReadStructure(char *line)` — Deserialize a Structure from File Format

Delegates to the shared compound reader: `return ReadCompound(line+10, T_STRUCTURE);`. The `"STRUCTURE "` prefix (10 chars) is stripped before parsing the generic format common to all compound track types.

---

### `GetAngleStruct(track_p trk, coOrd pos, EPINX_T *ep0, EPINX_T *ep1)` — Compute Orientation Angle for a Structure

Computes the angle at which a structure should be rotated when placed so that it aligns with existing track geometry (used primarily for pier-supported overpasses):
- Retrieves the extra-data block.
- Translates `pos` by subtracting the origin, then rotates by `-angle` to convert from global to local coordinates.
- Calls `GetAngleSegs()` which finds the angle of the first segment at point `pos`. If `pos` does not lie on a segment endpoint, this returns a default value (the angle of the first segment).
- Returns that angle normalized to `[0°, 360°)`, adding back the stored orientation offset.

The optional output pointers `ep0` and `ep1` are set to `-1` if not used by the caller. This function enables placing a structure so its geometry automatically aligns with the track at the specified point, useful for overpasses whose pier location is determined by intersecting tracks.

---

### `QueryStructure(track_p trk, int query)` — Structure Query Predicate

Called by the undo system to determine what kind of track object this is:
- **Q_HAS_DESC** → returns TRUE (structures always have a description field).
- **Q_IS_STRUCTURE** → returns TRUE (this confirms that `trk` represents a structure instance).
- Any other query code → returns FALSE.

This allows generic undo/redo logic to distinguish structures from curves or turnouts.

---

### `CompareStruct(track_cp trk1, track_cp trk2)` — Compare Two Structure Objects for Undo System

Used by the undo system to determine whether two structure track objects are identical (for comparing before/after states of an operation):
- Retrieves extra-data blocks from both tracks.
- Uses a series of `REGRESS_CHECK_*` macros to compare: origin, angle, flip flag, ungrouped flag, split index, description offset and size.
- Calls `CompareSegs()` to ensure the segment arrays (and their point coordinates) are byte-for-byte identical.

Returns TRUE if the two structure objects represent the exact same geometry and attributes; FALSE otherwise. This is used by `UndoSystem`'s diff logic to decide whether an undo/redo operation should be recorded as a "modify" or treated as a new object.

---

### `structureCmds` — Structure Command Record

A global command record that wires up all structure-related operations:

```c
static trackCmd_t structureCmds = {
    "STRUCTURE",
    DrawStructure,              // draw — renders the structure on screen
    DistanceCompound,           // distance — returns closest point to a given coordinate (generic)
    DescribeCompound,           // describe — builds human-readable description string
    DeleteCompound,             // delete — frees the track record and extra-data block
    WriteCompound,              // write — serializes to file format (compound format)
    ReadStructure,              // read — deserializes from file format
    MoveCompound,               // move — translate by offset vector
    RotateCompound,             // rotate — rotate around origin point
    RescaleCompound,            // rescale — no-op stub (reserved for future scaling logic)
    NULL,                       // undoStart — not used for structures
    GetAngleStruct,             // getAngle — compute orientation angle from a given point
    NULL,                       // split — unused
    NULL,                       // traverse — unused
    EnumerateCompound,          // enumerate — iterate over all segments (for traversing)
    NULL,                       // redraw — unused
    NULL,                       // trim — unused
    NULL,                       // merge — unused
    NULL,                       // modify — not used; structures are moved/rotated as a whole
    NULL,                       // getLength — unused
    NULL,                       // getTrkParams — unused
    NULL,                       // moveEndPt — unused
    QueryStructure,             // query — predicate for undo system
    UngroupCompound,            // ungroup — not used; structures are single objects
    FlipCompound,               // flip — mirror horizontally across a point
    NULL,                       // getAngleAtPoint — not needed for compound types
    DrawSegs,                   // drawPositionIndicator — draws selection handles (via DrawSegs)
    AdvancePositionIndicator,   // advancePositionIndicator — unused for structures
    CheckTraverse,              // checkTraverse — unused
    MakeParallel,               // makeParallel — unused
    NULL                        // drawDesc — description drawing handled separately
};
```

This record is registered via `InitObject()` and becomes accessible through the undo system as a valid track type. The `DrawSegs` function is used for the position indicator (selection handle) instead of a dedicated handler, since structures are drawn by their full geometry anyway.

---

### `CreateArrowAnchor(coOrd pos, ANGLE_T a, DIST_T len)` — Create Arrow Segment for Selection Handle

Creates two arrow-shaped line segments emanating from point `pos` at angles `a ± 135°`. Each arrow is drawn as a simple polyline (`SEG_STRLIN`) with zero width and blue color. Used when placing structures interactively to indicate the direction of placement or rotation hints.

---

### `CreateRotateAnchor(coOrd pos)` — Create Circular Rotation Handle

Creates five segments: one circular arc centered at `pos` (radius 2 units) rendered in aqua, plus three arrowheads pointing radially outward from the center. This forms a "rotate me" handle that visually indicates which point should be used as the rotation pivot for the structure being placed.

---

### `CreateMoveAnchor(coOrd pos)` — Create Translation Handle

Creates five arrows around point `pos`: two pairs of opposing arrows (horizontal and vertical) plus one diagonal arrow, all blue. This forms a crosshair-style handle that indicates "drag to move" semantics during placement mode.

---

### `PlaceStructure(coOrd p0, coOrd p1, coOrd origPos, coOrd *resPos, ANGLE_T *resAngle)` — Compute Placement Result from Drag Vector

Given the first point of a two-point drag (`p0`) and second point (`p1`), computes where to place the structure:
- If `curStructure->special == TOpierInfo`, then `p0` is interpreted as lying on track geometry. It calls `OnTrack()` to find the nearest track segment, then picks an endpoint via `PickEndPoint()`. The result position and angle are derived from the turnout's end-point angle (with a 90° offset).
- Otherwise, simply translates the structure's origin by the drag vector: `resPos = origPos + (p1 - p0)`.

Returns TRUE if pier-based placement succeeded; FALSE for ordinary translation.

---

### `NewStructure(void)` — Finalize Structure Placement with Undo Support

Called when the user clicks OK in the structure dialog or presses Enter after dragging to place:
- Checks that `Dst.state != 0` (a valid drag result is available).
- If a pier list dropdown was opened but no pier selected, aborts silently.
- Calls `UndoStart("Place Structure", "newStruct")`.
- Calls `NewCompound(T_STRUCTURE, ...)` which allocates a new track record and initializes the extra-data block with:
  - The structure definition pointer (`curStructure->segs`, `segCnt`) copied into the compound's segment field.
  - Visibility set to TRUE (visible by default).
  - No ties, no bridge flag, no roadbed flag set (plain overlay).
- Calls `DrawNewTrack(trk)` which draws the new structure on screen for preview.
- Calls `UndoEnd()` to commit the operation.
- Resets `Dst.state = 0` so a subsequent placement starts fresh.

This function is the core of interactive placement — it bridges the drag-and-drop UI with the undo system.

---

### `StructRotate(void *pangle)` — Rotate Placement Handle During Drag

Called during right-drag or Ctrl+drag on the rotation handle:
- If no structure has been selected yet (`Dst.state == 0`), exits early.
- Parses the argument pointer to extract a normalized angle value (divides by 1000.0 because angles are stored as integers in this codebase).
- Rotates `Dst.pos` around `cmdMenuPos` by that angle.
- Adds the delta to `Dst.angle`, accumulating rotation over multiple drags.
- Calls `TempRedraw()` to refresh the screen (structures drawn on a temporary canvas for preview).

---

### `CmdStructureAction(wAction_t action, coOrd pos)` — Structure Placement State Machine

Handles all mouse and keyboard events during structure placement:

| Action | Behavior |
|--------|-----------|
| **C_START** | Clears anchor arrays (`anchors_da`), resets state to 0, clears selection handles. Shows pier list if applicable. Prompts about magnetic snap modifier. |
| **wActionMove** | Creates move or rotate anchors depending on Ctrl modifier. Draws a crosshair at cursor position. |
| **C_DOWN** | First point of drag captured (`Dst.state=1`). Computes initial angle from track geometry if pier mode, otherwise stores first point as `origPos`. Shows prompt: "Left-Drag to place, Right-Drag or Ctrl+Drag to Rotate". |
| **C_MOVE** | Updates the preview position/angle based on current drag vector. Draws anchors at cursor. Informs user about magnetic snap modifier. |
| **C_RDOWN / C_RMOVE / C_RUP** | Right-click or Ctrl-drag triggers rotation mode: creates circular handle around `pos`, accumulates angle delta, draws arc + arrows indicating rotation direction. |
| **C_UP / C_FINISH** | Finalize placement by calling `NewStructure()` if valid, otherwise cancel. Resets state to 0. |
| **C_CANCEL** | Aborts entire operation: clears anchors, hides pier list, resets state. |
| **C_CMDMENU** | Opens the popup menu (for rotation options). |
| **C_REDRAW** | Draws preview segments and anchors on temporary canvas. |

This function implements a full state machine for interactive placement with multiple modes (translate vs. rotate) and modifier key support.

---

### `CmdStructure(wAction_t action, coOrd pos)` — Top-Level Command Handler

Entry point invoked by menu button or hotbar element:
- If the dialog window (`structureW`) is not yet created, builds it via `ParamCreateDialog()` with controls: structure list (dropdown), canvas preview area, hide checkbox. Registers for change notifications so that layout scale changes refresh the picker.
- On C_START: opens the dialog, loads structures into the dropdown via `structureChange()`, shows the window.
- Handles Ctrl+drag vs. Alt+drag routing to sub-actions (`CmdStructureAction`).
- On C_CANCEL: hides the dialog and cleans up handles.

The function delegates most of its work to `CmdStructureAction()`.

---

### `AddHotBarStructures(void)` — Populate Hotbar with Structure Icons

Iterates over all loaded structure definitions in `structureInfo_da`:
1. Skips entries whose parameter file is not currently loaded or whose scale is incompatible with the current layout gauge.
2. Calls `AddHotBarElement()` to create a hotbar button element that:
   - Displays the structure's contents label (e.g., "Overpass") as its icon text.
   - Stores the size and origin of the structure so it can be drawn in preview mode when selected from the hotbar.
3. Registers a callback (`CmdStructureHotBarProc`) that handles selection, list title formatting, full-name display, and drawing.

This allows users to place structures quickly without opening the dialog each time — just click the appropriate icon on the toolbar.

---

### `CmdStructureHotBarProc(hotBarProc_e op, void *data, drawCmd_p d, coOrd *origP)` — Hotbar Command Callback

Handles hotbar element operations:
- **HB_SELECT** → calls `CmdStructureAction(C_FINISH, zero)` to finalize placement of the selected structure and updates `curStructure` to point to it. Then invokes the menu button associated with this element via `DoCommandB()`.
- **HB_LISTTITLE / HB_BARTITLE** → formats short/medium-length display strings for list widgets (combining manufacturer name, part number, description).
- **HB_FULLTITLE** → returns the full title string for tooltips.
- **HB_DRAW** → draws the structure outline at `*origP` using the stored size and origin from the hotbar element data.

---

### `InitCmdStruct(wMenu_p menu)` — Register Structure Menu Button and Hotbar Element

Adds two buttons to the command menu:
1. **"Structure"** button with a building icon — invokes `CmdStructure()` which opens the structure picker dialog.
2. An unnamed hotbar element (invisible) that invokes `CmdStructureHotBar()` when clicked from the toolbar, allowing direct placement of any loaded structure without opening the dialog first.

Also registers the parameter group (`structurePG`) for use in dialogs and change notifications. Registers a popup menu ("Structure Rotate") containing rotation angle options via `AddRotateMenu()`.

---

### `InitTrkStruct(void)` — Finalize Structure Track Type Registration

Called once at application startup:
- Calls `InitObject(&structureCmds)` which registers the track type with the undo system, assigns a type index (e.g., `T_STRUCTURE`), and wires up all draw/edit/delete commands.
- Registers `ReadStructureParam()` as a parameter file parser so structures can be loaded from external files.
- Registers the pier parameter group (`pierPG`) for use when editing pier-specific fields in dialog controls.

---

## Usage Flow

1. **Initialization** — `InitTrkStruct()` registers `T_STRUCTURE` with the undo system and sets up parameter file parsers.
2. **Loading structures from files** — When a `.xtc` file containing structure definitions is loaded, each definition is parsed by `ReadStructureParam()`, which calls `CreateNewStructure()` to build an entry in the global list. Structures are displayed in the picker dialog for user selection.
3. **Placing via dialog** — User selects "Structure" from menu → dialog opens with a dropdown of compatible structures. User picks one, drags on canvas to position/rotate → `CmdStructureAction()` processes drag events and finalizes placement via `NewStructure()`.
4. **Placing via hotbar** — A structure icon is selected from the toolbar → `CmdStructureHotBar()` is invoked → user drags to place directly without opening dialog first.
5. **Rendering** — Each placed structure instance appears on screen drawn by `DrawStructure()` which renders all its segments as a polyline at the stored origin and angle, optionally with dashed/phantom styles if configured.

---

## Notes

- Structures are **not** real track segments — they do not have endpoints or curvature. They are purely visual overlays placed on top of the layout for decoration, modeling, or as placeholders for future track sections.
- The `REORIGSTRUCT` flag (defined in a header but not used currently) would have re-centered geometry to origin and flipped coordinates. This suggests structures may have been designed to support multiple coordinate systems in the past.
- Pier-supported overpasses use pier height values stored per-pier. When placed on track, the pier's top is aligned to the track surface by computing an elevation offset from the underlying segment endpoint.
