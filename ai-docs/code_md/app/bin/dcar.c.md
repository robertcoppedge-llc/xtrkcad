# dcar.c — Car Part & Item Data Management

## Overview

`dcar.c` implements the data model and management functions for cars, car parts (chassis), and car items in XTrkCAD. It handles:

- **Car Parts** (`carPart_t`) — chassis/loco definitions indexed by manufacturer + prototype
- **Car Items** (`carItem_t`) — individual car instances with geometry, prices, dates, and track placement
- **Car Prototypes** (handled in `ccornu.c` / `dproto.c`)

The file uses a two-level hierarchy:
1. **CarPartParents** — grouped by manufacturer + prototype type + scale
2. **CarParts** — individual part definitions within each parent group

---

## Core Data Structures

### Car Part (`carPart_t`)

```c
struct carPart {
    char        *title;           // Full title string (e.g., "Araldo 4-4-0 #1")
    int         paramFileIndex;   // Source parameter file index
    SCALEINX_T scaleInx;          // Scale this part applies to
    long        options;          // Flags: CAR_DESC_IS_LOCO, etc.
    long        type;             // Car prototype type code
    carDim_t    dim;              // Dimensions (carLength, carWidth, truckCenter, coupledLength)
    wDrawColor  color;            // Color of the chassis
    char       *partnoP;          // Part number substring within title
    int         partnoL;
    carPartParent_p parent;       // Pointer to CarPartParent containing this part
};
```

### Car Part Parent (`carPartParent_t`)

Groups parts by manufacturer and prototype:

```c
struct carPartParent {
    char        *manuf;           // Manufacturer name (e.g., "Araldo")
    char        *proto;           // Prototype name/number (e.g., "20734-15")
    SCALEINX_T  scale;            // Scale for this parent group
    dynArr_t    parts_da;         // Array of carPart_p pointers
};
```

### Car Item (`carItem_t`)

Represents a single car placed on the track layout:

| Field | Type | Description |
|-------|------|-------------|
| `index` | long | Unique item index in parameter file |
| `scaleInx` | SCALEINX_T | Scale of this car |
| `title` | char* | Full title (e.g., "Araldo 4-4-0 #1") |
| `contentsLabel` | char* | Human-readable label ("Car Item") |
| `barScale` | DIST_T | Scaling factor for bar charts |
| `options` | long | Bit flags: CAR_DESC_IS_LOCO, etc. |
| `type` | long | Prototype type code |
| `dim` | carDim_t | Car dimensions |
| `color` | wDrawColor | Chassis color |
| `data.number` | char* | Numbering (e.g., "10", "A") |
| `segPtr` | trkSeg_p* | Geometry segments for the chassis |
| `segCnt` | int | Number of segments |

---

## Key Functions

### Car Part Creation & Management

```c
carPart_p CarPartNew(
    carPart_p partP,          // NULL to create new; reuse existing if same key
    int paramFileIndex,       // Which parameter file this comes from
    SCALEINX_T scaleInx,      // Scale index
    char *title,              // Full title string
    long options,             // Option flags
    long type,                // Prototype type code
    carDim_t *dim,            // Dimensions
    wDrawColor color          // Color
);
```

**Behavior:**
- Looks up existing part by (manuf, proto, scale, title) — if found and not from CUSTOM file, reuses it.
- Creates a new `carPartParent` entry for the manufacturer/prototype group.
- Stores the part in that parent's `parts_da`.

```c
void CarPartDelete(carPart_p partP);
void DeleteCarPart(int fileIndex);  // Remove all parts from a parameter file
```

---

### Car Item Creation & Management

```c
carItem_p CarItemNew(
    carItem_p item,           // NULL for new; reuse existing if same title/index
    int paramFileIndex,       // Parameter file source
    long itemIndex,          // Unique index within this file
    SCALEINX_T scaleInx,     // Scale
    char *title,             // Full title
    long options,            // Flags including CAR_ITEM_HASNOTES, etc.
    long type,               // Prototype type
    carDim_t *dim,           // Dimensions
    wDrawColor color,        // Chassis color
    FLOAT_T purchPrice,      // Purchase price
    FLOAT_T currPrice,       // Current market value
    long condition,          // Condition rating (0–100)
    long purchDate,          // Date purchased
    long serviceDate         // Date serviced/overhauled
);
```

**Behavior:**
- Reuses an existing item if the title matches and it's not from a custom file.
- Loads roadname list and reporting marks from parameter file metadata.
- If `CAR_ITEM_ONLAYOUT` is set, parses geometry segments and creates a track object via `NewCar()`.

```c
void CarItemGetSegs(carItem_p item);  // Load/chunk car geometry into segPtr
BOOL_T WriteCars(FILE *f);             // Serialize all items to file
void DeleteCarPart(int fileIndex);     // Remove parts from one parameter file
```

---

### Compatibility Checking

```c
enum paramFileState GetCarPartCompatibility(
    int paramFileIndex,   // Which parameter file
    SCALEINX_T scaleInx   // Current layout scale
);
```

Checks whether car parts in a given parameter file are compatible with the current track gauge/scale:
- `PARAMFILE_NOTUSABLE` — no compatible parts found
- `PARAMFILE_FIT` — exact match on scale/gauge
- `PARAMFILE_COMPATIBLE` — some compatible parts exist (different scale)

---

### Car Item Description Formatting

```c
char *CarItemDescribe(carItem_p item, long mode, long *index);
```

Formats a human-readable description of a car item. The `mode` parameter controls which fields to show:

- Bit 0–3: Include manufacturer (bit 1), prototype type (bit 2), part number (bit 3)
- Bit 4+: Show index, road name, reporting mark, car number
- Returns a formatted string suitable for drop-down lists or info messages.

---

### Coupler Mount Point Calculation

```c
void CarItemFindCouplerMountPoint(
    carItem_p item,
    traverseTrack_t trvTrk0,  // Reference traverse track (e.g., from coupler end)
    coOrd pos[2]              // Outputs: coupler mount positions on each truck
);
```

Computes where the coupler pivots attach to a car. Handles both **body-mounted** and **truck-mounted** couplers, accounting for:
- Single vs. dual-truck cars (`truckCenterOffset`)
- Coupler length relative to coupledLength dimension
- Truck angle (for truck-mounted couplers)

---

### Item Placement on Track

```c
void CarItemPlace(
    carItem_p item,
    traverseTrack_p trvTrk,   // Reference track traversal point
    DIST_T *dists             // Outputs: distances to each coupler end
);
```

Computes the item's center position and angle relative to a given track traversal point. Used when snapping a car onto existing track geometry.

---

### Drawing Functions

```c
void CarItemDraw(
    drawCmd_p d,              // Drawing context
    carItem_p item,           // Item being drawn
    wDrawColor color,         // Color override (e.g., selection color)
    int direction,            // 0 = left-to-right, 1 = right-to-left
    BOOL_T locoIsMaster,      // TRUE for locomotive master (headlight draws)
    vector_t *coupler,        // Coupler mount positions
    track_p traverse          // Track used for clearance checks
);
```

Draws:
- The car body geometry (from `segPtr` or a default rectangle if not loaded)
- Truck circles on the rails (if `drawCarTrucks` is set)
- Headlight glow (for locomotives, drawn as a filled circle at the front truck center)
- Coupler rods connecting the two trucks

```c
void CarItemUpdate(carItem_p item);  // Called when scale changes — refreshes geometry
```

---

### Hotbar Integration

```c
void AddHotBarCarDesc(void);         // Populate hotbar with available car parts
carItem_p currCarItemPtr;            // Global pointer to currently selected car part
long carHotbarModeInx = 1;           // Controls how items are displayed in the drop-down

int CarAvailableCount(void);          // Count of available cars at current scale
```

The hotbar system lets users quickly select a car by manufacturer, prototype, and type. Items are sorted by configurable criteria (index, manufacturer name, part number, etc.).

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `CarPartNew()` | Create or reuse a car part definition | title, options, type, dim, color |
| `CarItemNew()` | Create or reuse a car item instance | paramFileIndex, scaleInx, title, prices, dates |
| `CarItemGetSegs()` | Load/chunk geometry for drawing | — |
| `WriteCars()` | Serialize all items to parameter file | FILE* |
| `DeleteCarPart()` | Remove parts from one parameter file | file index |
| `CarItemDescribe()` | Format a human-readable item description | mode bitmask |
| `CarItemFindCouplerMountPoint()` | Find coupler pivot positions on trucks | traverseTrack reference |
| `CarItemPlace()` | Compute position/angle of an item on track | traverseTrack, returns distances to ends |
| `CarItemDraw()` | Render a car onto the drawing canvas | drawCmd_p, color override |

---

## Domain & Design Notes

- **Parameter file versioning**: The code checks `paramVersion` to decide whether notes are stored as a separate multiline block or as a single escaped string.
- **Scale compatibility**: Car parts are only usable on layouts that match their scale exactly; otherwise they may be compatible (usable but at wrong size) or unusable.
- **Geometry caching**: When an item is selected, `CarItemGetSegs()` loads the full geometry from prototype definitions and caches it in `segPtr` to avoid repeated computation during drawing.
- **Hotbar modes**: The hotbar supports multiple display modes controlled by a bitmask — e.g., show only available cars at current scale, or show all known parts regardless of scale compatibility.
