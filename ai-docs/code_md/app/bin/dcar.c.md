# dcar.c — Car/Train Database & Serialization (Complete)

## Overview

`dcar.c` handles **database operations for train cars and locomotives** in XTrkCAD. It provides:

- `read_track()` / `write_track()` — Serialize track data (including cars) to/from text-based database files
- Car prototype management (loading/saving custom prototypes from parameter files)
- Custom property fields (`custom.c`) integration
- Compound parts grouping (grouping multiple car components into a single entity)

The "dcar" prefix indicates this is the **data-access layer** for cars/trains — analogous to how `drawgeom.c` handles drawing geometry or `cselect.c` handles selection.

---

## Data Structures

### `tabString_t / tabString_p` — Tab-Separated String Parser

```c
typedef struct {
    char * ptr;   // Pointer into a string at some position
    int len;      // Length of the substring
} tabString_t, *tabString_p;
```

Used by `TabStringExtract()` to parse tab-delimited lines from database files. Each field (manufacturer name, prototype description, scale index) is stored as a **substring reference** rather than a full copy — efficient for large datasets. Only when a field needs independent ownership does `TabStringDup()` allocate a new heap buffer.

---

### `carDim_t` — Car Dimension Data

```c
typedef struct {
    DIST_T carLength;       // Overall length of the car
    DIST_T carWidth;        // Overall width
    DIST_T truckCenter;     // Distance from front to center of trucks (single-truck cars)
    DIST_T truckCenterOffset;  // Offset for dual-truck center point
    DIST_T coupledLength;   // Length when two cars are coupled together
} carDim_t;
```

---

### `carData_t` — Car Inventory / Property Data

```c
typedef struct {
    char * number;              // Car number (e.g., "A1 234 567")
    FLOAT_T purchPrice;         // Purchase price (currency)
    FLOAT_T currPrice;          // Current market value
    long condition;             // Condition rating
    long purchDate;             // Date purchased
    long serviceDate;           // Date last serviced
    char * notes;               // Free-form notes field
} carData_t;
```

---

### `carItem_t` — In-Layout Car Instance

```c
struct carItem_t {
    long index;                 // Unique identifier for this instance
    SCALEINX_T scaleInx;       // Scale of the prototype it references
    char * contentsLabel;       // Label text displayed in the UI
    char * title;              // Title (used for grouping/filtering)
    carProto_p proto;          // Pointer to the car prototype definition
    DIST_T barScale;           // Display scale factor for the icon/outline
    wDrawColor color;          // Color of the car outline/draw
    long options;              // Bitmask: CAR_DESC_IS_LOCO, CAR_ITEM_HASNOTES, etc.
    long type;                 // Car type (locomotive, freight, passenger, M-O-W, other)
    carDim_t dim;             // Physical dimensions of the car
    carData_t data;           // Inventory/property data fields
    wIndex_t segCnt;          // Number of segments in the outline geometry
    trkSeg_p segPtr;         // Pointer to first segment of outline
    track_p car;             // Track object associated with this car item
    coOrd pos;               // Position (relative to car's local origin)
    ANGLE_T angle;          // Rotation angle of the car
};
```

**Note:** `carItem_t` is distinct from `track_p`. It represents a *logical* car entity that can exist independently of any track geometry. When placed on a layout, it becomes associated with a track object via the `car` field.

---

### `carProto_t` — Car Prototype Definition

```c
struct carProto_t {
    char * contentsLabel;   // Label used in dropdown lists ("Freight Car", "Steam Loco")
    wIndex_t paramFileIndex;  // Which parameter file this prototype came from
    char * desc;           // Human-readable description (e.g. "Baldwin DL-109")
    long options;          // Options bitmask (loco, non-loco flags)
    long type;             // Type: diesel, steam, electric freight, passenger, MOW, other
    carDim_t dim;         // Physical dimensions
    int segCnt;           // Number of segments in the outline polygon
    trkSeg_p segPtr;     // Pointer to first segment of the outline geometry
    coOrd size;          // Bounding box width/height (used for icon sizing)
    coOrd orig;          // Origin point of the prototype's local coordinate system
};
```

Prototypes are loaded from parameter files (`PARAMFILE_...`) or created dynamically when a new car type is needed. The `paramFileIndex` tracks which file the data came from so that deleting a parameter file can clean up its prototypes automatically.

---

### `carPart_t / carPartParent_t` — Compound Parts (Grouped Objects)

```c
struct carPart_t {
    carPartParent_p parent;  // Pointer to parent grouping container
    wIndex_t paramFileIndex;  // Which parameter file this part came from
    char * title;            // Part name ("Boiler", "Tender", etc.)
    long options;           // Options bitmask (color, visibility flags)
    long type;              // Type of part
    carDim_t dim;          // Dimensions
    wDrawColor color;      // Color for drawing
    char * partnoP;       // Part number string (for lookups)
    int partnoL;         // Length of part number string
};

struct carPartParent_t {
    char * manuf;        // Manufacturer name ("Baldwin", "Fleischmann")
    char * proto;       // Prototype description
    SCALEINX_T scale;   // Scale (e.g. 1:87, 1:90)
    dynArr_t parts_da;  // Array of carPart_t members
};
```

These structures implement a **compound object** model: multiple `carPart_t` elements are grouped together under a single "parent" (`carPartParent_t`). The parent can be displayed as a single icon, and individual parts can have their own colors/properties. This is used for grouping related components (e.g., the body + cab of a locomotive) into a single selectable unit.

---

### `cmp_key_t` — Comparison Key Structure

```c
typedef struct {
    char * name;  // String to compare against
    int len;      // Length of that string
} cmp_key_t;
```

Used as a generic comparison key for binary search lookups in sorted arrays (e.g., searching part numbers).

---

### `roadnameMap_t` — Road Name / Replacement Mark Lookup

```c
struct roadnameMap_t {
    char * roadname;   // The actual road name (e.g. "Freight and Passenger")
    char * repmark;   // Short replacement mark ("F&P") displayed in the UI
};
```

Maps full road names to their short marks, loaded from parameter files for use in labeling or filtering.

---

## Core Functions

### `TabStringExtract(string, count, tabs)` — Parse Tab-Delimited Fields

Parses a single line of tab-delimited text into an array of `tabString_t` structs. Each element points to a substring within the original string and stores its length. The function handles trailing fields (pads with empty entries if fewer than `count` fields are present). It also defaults "Unknown" for missing manufacturer names.

---

### `TabStringDup(tab)` — Duplicate a Tab-Field String

Allocates a new heap buffer and copies the contents of a tab field. Used to convert temporary substring references (which point into the original line) into owned strings that can be freed independently.

```c
char * ret = MyMalloc( tab->len+1 );
memcpy( ret, tab->ptr, tab->len );
ret[tab->len] = '\0';
return ret;
```

---

### `TabStringCpy(dst, tab)` — Copy Into a Buffer

Copies the contents of a tab field into an existing buffer. Used when you already have space allocated and want to fill it (e.g., into a structure member).

---

### `TabStringCmp(src, tab)` — Case-Insensitive Compare with Partial Match

Compares two strings where one is stored as a `tabString_t` substring reference. It handles partial matches: if the source string is longer than the tab field, it treats them as equal (prefix match). Returns -1, 0, or +1 like `strcmp`. Used for lookups in parameter files where a user might type "Baldwin" and match against "Baldwin Locomotive Works".

---

### `TabGetLong(tab)` / `TabGetFloat(tab)` — Extract Numeric Values from Tab Fields

Temporarily null-terminates the string at its stored length, calls `atol()` or `atof()`, then restores the original character. This avoids needing to make a full copy of the field just for numeric conversion.

---

### `CarProtoFind(desc)` — Lookup Prototype by Description

Performs a binary search (using `CmpCarProto`) over the sorted `carProto_da` array to find a prototype matching the given description string. If not found, returns NULL. The comparison function simply calls `strcasecmp()` on the `.desc` field — prototypes are stored sorted alphabetically by their description.

---

### `CarProtoLookup(desc, createMissing, isLoco, length, width)` — Create or Find a Prototype

This is a **factory method** that either:
1. Looks up an existing prototype by its description in the global array, OR
2. If `createMissing` is true and the prototype doesn't exist yet, creates one dynamically (allocates memory, initializes fields with defaults).

If it's already been created (e.g., from a previous call), it returns the same pointer — ensuring that multiple callers don't create duplicate prototypes for the same car type.

When creating a new prototype:
- Allocates a new struct via `LookupListElem()`
- Duplicates the description string
- Sets default options based on whether it's a locomotive or not
- Initializes dimensions from arguments
- Calls `CarProtoDlgCreateDummyOutline()` to generate a placeholder polygon outline (a simple rectangle)
- Computes bounding box with `GetSegBounds()`
- Marks `carProtoListChanged = TRUE` so that the list can be serialized to disk on next save

---

### `DeleteCarProto(protoP)` — Remove a Prototype from Memory

Removes a prototype entry from the `carProto_da` array and frees its memory. Used when deleting a custom car type or when cleaning up after loading an old database file with obsolete entries.

---

### `GetCarProtoCompatibility(paramFileIndex, scaleIndex)` — Check Which Parameter Files Are Loaded

Iterates over all loaded prototypes and checks if any came from the given parameter file index. Returns:
- `PARAMFILE_UNLOADED` if no prototypes are loaded at all
- `PARAMFILE_NOTUSABLE` if none match the requested file
- `PARAMFILE_FIT` if at least one prototype was found in that file

Used to decide whether a particular database file should be loaded into memory.

---

### `CarProtoNew(proto, paramFileIndex, desc, options, type, dim, segCnt, segPtr)` — Create/Update a Prototype Entry

This is essentially an **upsert**: it either finds an existing prototype with the given description and updates its fields (freeing old strings first), or creates a new one. The `paramFileIndex` is used to tag prototypes so that they can be cleaned up when their source parameter file is unloaded.

Notably, it uses `memdup()` to allocate space for the segment array — this copies the segments from a temporary buffer into a per-prototype structure. Then `CloneFilledDraw()` duplicates all draw commands (filling polygons, etc.) and computes bounding boxes.

---

### `CarProtoDelete(protoP)` — Free a Prototype's Memory

Removes the entry from the global list (`RemoveListElem`), frees the duplicated description string, and calls `MyFree()` on the struct itself.

---

### `DeleteCarProto(fileIndex)` — Bulk Delete by Parameter File Index

Finds all prototypes whose `paramFileIndex` matches the given value and deletes them one-by-one. It also compacts the array afterward: after deleting all matching entries, it shifts remaining elements left to fill the gaps. This is an **in-place deletion** pattern that avoids reallocating a new array.

---

### `CarProtoRead(line)` — Parse a Single Prototype Definition Line

Parses a line from a parameter file into fields using `GetArgs()` with format `"qllff0lff"` (string, long, long, float, float, optional char*, float, float). The first nine fields are:
1. Description string
2. Options bitmask
3. Type code
4. Length
5. Width
6. Center offset (scaled by 1000)
7. Truck center
8. Coupled length

Then it calls `ReadSegs()` to read the segment array from disk, and finally creates a prototype via `CarProtoNew()`. The description string is freed after use since it will be re-allocated in the new struct.

---

### `CarProtoWrite(f, proto)` — Write a Single Prototype to File

Outputs one line per prototype:
```text
CARPROTO "Description" options type length width centerOffset truckCenter coupledLength
<tab-separated segment data...>
```

It sets C locale for decimal point handling (writes "." not comma), then writes the header line and all segments. Finally restores user locale and returns success/failure of both operations via `&`.

---

### `CarProtoCustomSave(f)` — Write Only Custom Prototypes

Iterates over all prototypes in the global array and only writes those with `paramFileIndex == PARAM_CUSTOM`. This is used when exporting a curated list of custom car types that the user has defined manually.

---

### `CmpPart()` / `CmpPartParent()` / `CmpRoadnameMap()` — Comparison Functions for Binary Search

These are the comparator functions used with `LookupListElem()` to find elements in sorted arrays:
- `CmpPart`: Compares part numbers case-insensitively, then by parameter file index (DEMO last, CUSTOM before LAYOUT), then by paramFileIndex numeric value.
- `CmpPartParent`: Sorts manufacturers alphabetically (reversed!), then by scale, then prototype description. The reversal on the first field is a quirk that ensures consistent ordering across lookups.
- `CmpRoadnameMap`: Compares road name strings with prefix-match logic (if the key string is longer than the stored entry, they are considered equal).

---

### `LoadRoadnameList(roadnameTab, repmarkTab)` — Load Road Name Mappings

Takes a tab-delimited pair of strings ("Full Name<TAB>ShortMark") and inserts each into a sorted lookup table. Uses binary search to find an existing entry; if not found, allocates a new `roadnameMap_t` with duplicated strings. If the full name is empty (e.g., "undecorated"), returns NULL — this is treated as a special case meaning "no road name decorations."

---

### `CarPartFind(manufP, manufL, partnoP, partnoL, scale)` — Find a Part by Number

Searches through all compound parents (manufacturers × scales) for a part whose number matches the given string and length. Returns NULL if not found. The comparison uses both case-sensitive prefix matching and parameter file indexing to handle partial matches gracefully.

---

### `CarPartParentDelete(parentP)` — Remove a Compound Parent from Memory

Removes the parent entry from its array, frees the manufacturer name and prototype description strings (these were duplicated during creation), then frees the struct itself. The contained parts are *not* freed here — they must be cleaned up separately or via another deletion path.

---

## Design Decisions & Tradeoffs

### Why Use `tabString_t` Instead of Full String Copies?

When parsing a line from a database file, each field is stored as a pointer into the original buffer with its length. This avoids:
- Allocating ~10 small strings per car prototype on load
- Copying data unnecessarily during initial parse

Only when a field needs to be owned independently (e.g., written back to disk) does `TabStringDup()` allocate a new copy. This is the classic **string pooling** / **substring reference** optimization used in performance-critical parsers.

### Why Binary Search for Prototype Lookup?

The global car prototype array (`carProto_da`) is kept sorted by description string (via `CmpCarProto`). Lookup is O(log n) instead of O(n), which matters when the user types a search term and the application must find whether that exact prototype already exists before creating a new one.

### Why Keep `paramFileIndex` on Each Prototype?

When the user deletes or edits a parameter file, the program needs to know *which* prototypes came from that file so it can free their memory safely without affecting prototypes loaded from other files or created manually (custom). The index acts as a tag for garbage collection.

### Why Store Compound Parts in Nested Dynamic Arrays?

A `carPartParent_t` holds an array of `carPart_t` children. This allows multiple parents — e.g., "Baldwin 1:87 Steam Locomotive" and "Fleischmann N-Scale Diesel Loco" are distinct groups. The nesting (`dynArr_t carPartParent_da[ ][ ]`) is a sparse matrix representation: not every manufacturer/scale combination exists, so we store only the parents that have at least one part.

---

## Summary Table

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `TabStringExtract(...)` | Parse tab-delimited line into substring references | input string, number of fields expected, output array |
| `TabStringDup(tab)` | Allocate and copy a field's content into heap memory | pointer to tab field struct |
| `CarProtoFind(desc)` | Binary-search lookup of existing prototype by description | description string |
| `CarProtoLookup(...)` | Factory method: find or create a prototype entry | description, whether to auto-create, dimensions |
| `DeleteCarProto(fileIndex)` | Bulk delete all prototypes that came from a given parameter file | file index |
| `CarProtoNew(...)` | Create/Update a single prototype entry (upsert) | existing pointer (NULL=create), param file ID, description, options, type, dimensions, segment count & pointer |
| `CarPartFind(manuf, partnoL, scale)` | Look up a compound part by number within its parent group | manufacturer name (prefix), part number string length, scale index |
| `LoadRoadnameList(...)` | Load road name → replacement mark mappings from tab-delimited text | two tab fields containing full names and short marks |

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Load/save car/train database records; manage custom prototypes (locomotives, freight cars, passenger cars); implement compound grouping for multi-part models; provide road name/rep mark lookup tables |
| **Domain** | Data persistence: reading/writing tab-delimited text files; managing a global registry of car types and their dimensions/colors; supporting user-defined custom entries alongside built-in prototypes |
| **Key concept** | Car prototype = a reusable definition (dimensions, color outline, segment geometry) that can be instantiated multiple times on the layout. Compound parts allow grouping several outlines into one compound object for unified selection/manipulation. |
| **Main entry points** | `read_track()` — reads an entire database file and populates global arrays; `write_track(f)` — writes all loaded prototypes/parts to a text file |
