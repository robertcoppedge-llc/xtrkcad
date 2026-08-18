# dcmpnd.c — Compound Tracks (Turnouts & Structures) Management

## Overview

`dcmpnd.c` handles the **compound track** data model in XTrkCAD, which includes turnouts (switches), structures (buildings/bridges), and other complex track elements that have multiple end points and custom geometry. The file implements:

- Track refresh logic for updating compound tracks from parameter files
- Custom management operations (rename, edit, delete)
- File I/O serialization of compound track definitions
- Utility functions for list lookup by context pointer

---

## Core Concepts

### Compound Tracks vs. Simple Tracks

A **simple track** is a single continuous loop or line with one traversal endpoint per end. A **compound track** has:

- Multiple **end points** (for multi-track turnouts like 4-way switches)
- Custom geometry defined by segments and paths
- Associated metadata (manufacturer, description, part number, scale)
- Optional custom data stored in a text field (`customInfo`)

### Turnout Types

| Type | Description | End Points |
|------|-------------|------------|
| `T_TURNOUT` | Switches, frogs, turnouts | 2–4 (depending on configuration) |
| `T_STRUCTURE` | Buildings, bridges, tunnels | 1+ |

---

## Key Data Structures

### Turnout Info (`turnoutInfo_t`)

Located in `compound.h`, but referenced here:

```c
struct turnoutInfo {
    char        *title;           // e.g. "Araldo 4-4-0 #1"
    SCALEINX_T  scaleInx;         // Scale this turnout applies to
    long        paramFileIndex;   // Which parameter file (PARAM_CUSTOM, etc.)
    int         segCnt;           // Number of segments in geometry
    trkSeg_p * segs;              // Array of track segments
    int         endCnt;           // Number of end points
    struct endPt_t *endPt;       // End point definitions
    char       *customInfo;       // Free-form custom data (for editable turnouts)
    turnoutType_t type;           // Turnout type code
};
```

### Structure Info (`structureInfo_t`)

Similar structure but for buildings, bridges, etc. stored in `structureInfo_da` dynamic array.

---

## Key Functions

### Refresh Compound Tracks

The refresh system re-reads geometry from parameter files when a compound track is selected and refreshed via the command menu. This updates outdated turnout/structure definitions.

```c
int refreshCompoundCnt;  // Global counter tracking number of tracks refreshed
```

#### `CheckCompoundEndPoint(track_p trk, EPINX_T trkEp, turnoutInfo_t *to, EPINX_T toEp, BOOL_T flip)`

Validates that a track end point matches a turnout's expected endpoint:

1. **Position check**: The physical position of the track end must be within `connectDistance` (default ~30 pixels) of the turnout's defined end point.
2. **Angle check**: The angle difference between tracks must not exceed `connectAngle` (default ~15°). This ensures the switch rails are properly aligned with the turnout frog/switch rail geometry.

If a mismatch is found, an error message explains that endpoints aren't close or aligned.

#### `RefreshCompound1(track_p trk, turnoutInfo_t *to)`

Main refresh logic for one compound track:

- Validates all end points match
- Reuses existing geometry if compatible; otherwise replaces it from the parameter file
- Handles **flip** cases (e.g., a 3-point turnout where endpoints are swapped)
- Updates the track's extra data (`extraDataCompound_t`) with new segments and paths
- Clears the `TB_SELECTED` bit on success

#### `RefreshCompound(track_p trk, BOOL_T junk)`

Dispatch function that:
- Skips non-compound tracks (simple loops)
- Looks up the matching turnout by title
- If no match found in parameter files, opens a dialog allowing the user to manually choose a replacement turnout/structure from the available list
- Handles multi-endpoint turnouts (e.g., 4-way switches with 3 endpoints)

---

### Custom Management

#### `CompoundCustomSave(FILE *f)`

Serializes custom/editable turnouts and structures to a parameter file. Only writes entries where:

- `paramFileIndex == PARAM_CUSTOM`
- `segCnt > 0` (has geometry defined)

Writes the turnout/structure header line, then optionally a `U <customInfo>` line if editable data is present.

#### `CompoundCustMgmProc(int cmd, void *data)`

Command dispatch function for custom management operations:

| Command | Action |
|---------|--------|
| `CUSTMGM_DO_COPYTO` | Copy the turnout/structure to the custom management file stream |
| `CUSTMGM_CAN_EDIT` | Returns TRUE if the object can be edited (has end points and custom data) |
| `CUSTMGM_DO_EDIT` | Opens rename dialog; parses title into manuf/desc/partno fields |
| `CUSTMGM_GET_TITLE` | Formats a human-readable description including scale, part number, etc. |

#### `EditCustomTurnout(turnoutInfo_t *to, turnoutInfo_t *to2)`

Compares the current custom object (`to`) with an editable prototype (`to2`), and copies geometry/path data from the prototype to the custom instance. This allows users to customize a turnout by editing its parameters while reusing the same base geometry.

---

### File I/O for Compound Tracks

#### `WriteCompoundPathsEndPtsSegs(FILE *f, paramPathList_t *paths, int segCnt, trkSeg_p *segs, int endCnt, struct endPt_t *endPt)`

Writes turnout/structure data to a parameter file:
- Writes the header line with scale and title
- Optionally writes `U <customInfo>` if present
- Calls `WriteCompoundPathsEndPtsSegs` (in another source file) to serialize paths, endpoints, and segments

---

## Utilities

### `FindListItemByContext(wList_p listP, void *context)`

Generic utility that searches a drop-down list by its item context pointer. Used in the refresh dialog to locate which list entry corresponds to the currently selected track element.

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `CheckCompoundEndPoint()` | Validate track end vs. turnout endpoint match | Uses position + angle tolerance checks |
| `RefreshCompound1()` | Replace track geometry with parameter file data | Handles flip logic for 3/4-point turnouts |
| `RefreshCompound()` | Top-level refresh dispatcher; opens dialog if no match | User can manually select a replacement |
| `CompoundCustomSave()` | Serialize custom objects to file | Only writes editable (PARAM_CUSTOM) entries |
| `CompoundCustMgmProc()` | Custom management command dispatcher | Handles copy, edit, delete operations |
| `EditCustomTurnout()` | Apply prototype geometry to custom instance | Enables parameter-based customization |

---

## Domain & Design Notes

- **Refresh count**: `refreshCompoundCnt` tracks how many compound tracks were successfully refreshed. This counter is used in the info message ("X Track(s) refreshed").
- **Flip handling**: Multi-endpoint turnouts (3-way, 4-way switches) may need endpoint reordering. The flip logic checks if reversing an endpoint's order yields a valid match.
- **Custom data field**: Turnouts/structures with `customInfo` set are editable; their geometry is stored in the parameter file but their parameters can be modified interactively via the custom management dialog.
