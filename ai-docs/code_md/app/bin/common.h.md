# common.h

## Overview
`common.h` is a fundamental header file that defines basic types, macros, type definitions, and utility includes used throughout the XTrkCAD codebase. It establishes the foundational building blocks for almost every other source file in `app/bin/`.

---

## Includes
```c
#include <ctype.h>    // Character classification functions (isalpha, isspace, etc.)
#include <errno.h>    // Error number definitions
#include <locale.h>   // Locale-related functions
#include <math.h>     // Mathematical functions
#include <stdarg.h>   // Variable argument list handling
#include <stdint.h>   // Fixed-width integer types (int32_t, int64_t)
#include <stdio.h>    // Standard I/O (stdin, stdout, stderr)
#include <stdlib.h>   // General utilities (malloc, free, atoi, etc.)
#include <string.h>   // String manipulation functions
#include <sys/stat.h> // File stat operations
#include <sys/types.h>// System type definitions

// Conditional includes based on platform:
#  ifdef WINDOWS
#include <io.h>       // Windows I/O extensions
#include <process.h>  // Process utilities on Windows
#include "include/dirent.h"  // Directory entry handling (Windows)
#include "direct.h"   // Windows directory/path utilities
#include "getopt.h"   // Command-line option parsing
#  else
#include <dirent.h>   // Unix/Mac directory traversal
#include <unistd.h>   // Unix system calls
#  endif
```

---

## Platform-Specific Defines

### Unix/Linux/macOS
```c
#define PATH_SEPARATOR "/"
```

### Windows
```c
#define PATH_SEPARATOR "\\"

// Windows-specific re-exports of POSIX functions:
#define access _access       // File permission check
#define unlink(a)  _unlink((a))    // File removal
#define rmdir(a)   _rmdir((a))     // Directory removal
#define open(name, flag, mode)  _open((name), (flag), (mode))
#define close(file) _close((file))
#define getpid()   _getpid()

// Case-insensitive string comparison:
#define strcasecmp _stricmp
#define strncasecmp _strnicmp

// Directory creation:
# define mkdir(DIR, MODE)  _mkdir((DIR))

// Starting from Visual Studio 2015 (_MSC_VER >= 1900), round() is in math.h.
// For older compilers, a custom implementation is provided:
#if ( _MSC_VER < 1900 )
# define round(x)   floor((x)+0.5)
#endif

// Suppress Visual Studio warnings:
#pragma warning( disable : 4305 )  // Conversion of int to char (in .bmp files)
#pragma warning( disable : 6385 )  // Array reference bounds check
```

---

## Defines and Constants

| Macro | Value | Purpose |
|-------|-------|---------|
| `M_PI` | `3.14159265358979323846` | Pi constant |
| `F_OK` | `(00)` | File exists test (access) |
| `W_OK` | `(02)` | Write permission test |
| `R_OK` | `(04)` | Read permission test |
| `DIST_INF` | `2.0E9` | Infinite distance sentinel value |
| `NUM_LAYERS` | `(99)` | Maximum number of drawing layers |
| `BASE_DPI` | `(75.0)` | Base dots-per-inch for rendering |

---

## Version Descriptors

```c
#define VERSION_DESCRIPTION2  (12)   // Bezier, Cornu, Joint format changes
#define VERSION_INLINENOTE    (12)   // Inline quoted text in Notes and Cars
#define VERSION_NONAKEDENDS   (12)   // END is replaced by END$SEGS, END$TRK, etc.
```

---

## Type Definitions

### Floating Point Types
```c
typedef double FLOAT_T;    // General floating point type alias
typedef double POS_T;      // Position coordinate type
typedef double DIST_T;     // Distance measurement type
typedef double ANGLE_T;    // Angle measurement type (radians)
typedef double LWIDTH_T;   // Line width type
```

### Integer Types
```c
typedef int INT_T;         // General integer type alias
typedef int BOOL_T;        // Boolean-like integer type
typedef int EPINX_T;      // Endpoint index type
typedef int CSIZE_T;      // Coordinate size/type indicator
typedef int SIZE_T;       // Size/count type (non-Windows)
typedef int STATE_T;      // State machine state type
typedef int STATUS_T;     // Function return status type
typedef signed char TRKTYP_T;  // Track type identifier (-127 to 127)
typedef int TRKINX_T;     // Track index type
typedef long DEBUGF_T;    // Debug flag type (bitmask)
```

### Platform-Specific Size Type
```c
#ifndef WIN32
typedef int SIZE_T;       // Unix size type matches `int` width
#endif
```

### Specialized Index Types
```c
typedef long SCALEINX_T;      // Scale index/type identifier
typedef long GAUGEINX_T;      // Gauge (rail gauge) index/type identifier
typedef long SCALEDESCINX_T;  // Scale description index
```

---

## Enumerations

### Param File States
Used to indicate the compatibility status of a parameter file during load:

| Value | Constant | Meaning |
|-------|----------|---------|
| `0`   | `PARAMFILE_UNLOADED`     | Not yet loaded |
| `1`   | `PARAMFILE_NOTUSABLE`    | Incompatible/invalid |
| `2`   | `PARAMFILE_COMPATIBLE`   | Compatible but not perfect fit |
| `3`   | `PARAMFILE_FIT`          | Fully compatible/fit |

---

## DYNARRAY — Dynamic Array Macro Type

A lightweight dynamic array abstraction implemented as a single struct:

```c
typedef struct {
    int cnt;   // Current number of elements
    int max;   // Allocated capacity
    void * ptr; // Pointer to element array
} dynArr_t;
```

### Macros for DYNARRAY Usage

| Macro | Purpose |
|-------|---------|
| `DYNARR_APPEND(T,DA,INCR)` | Append `INCR` elements to the array (grows by `INCR` if needed) |
| `DYNARR_LAST(T,DA)` | Access the last element: `((T*)(DA).ptr)[(DA).cnt-1]` |
| `DYNARR_N(T,DA,N)`     | Access element at index `N` |
| `DYNARR_RESET(T,DA)`   | Reset `.cnt` to 0 (keeps `.max` and `.ptr` intact) |
| `DYNARR_SET(T,DA,N)`   | Set number of elements to `N`; grows if necessary |
| `DYNARR_INIT(T,DA)`    | Initialize: sets all fields to zero; suitable for stack-local variables |
| `DYNARR_FREE(T,DA)`    | Free the array and reset counters |
| `DYNARR_REMOVE(T,DA,N)` | Remove element at index `N` (shifts subsequent elements left) |

#### Notes on DYNARRAY:

- The struct has three fields only: `.cnt`, `.max`, `.ptr`.
- Growth is amortized — the array grows by a configurable increment (`INCR`) when capacity is exhausted.
- Memory allocation uses a custom `MyRealloc` function (presumably defined elsewhere).
- On failure, `abort()` is called immediately (no graceful error handling here).

---

## Size Constants for Strings

| Macro | Value | Use Case |
|-------|-------|----------|
| `STR_SIZE`         | `256`       | Short strings |
| `STR_SHORT_SIZE`   | `80`        | Very short labels/titles |
| `STR_LONG_SIZE`    | `1024`      | Longer descriptive text |
| `STR_HUGE_SIZE`    | `10240`     | Large blocks of text (e.g., notes) |

---

## Type Aliases for Convenience

```c
#define CAST_AWAY_CONST  (char*)   // Cast away const qualifier
#define TITLEMAXLEN      (40)       // Maximum title string length
```

---

## File Version Constants

These define the maximum allowed file version number and its sub-components:

| Macro | Value | Description |
|-------|-------|-------------|
| `VERSION_DESCRIPTION2`   | `12` | Bezier, Cornu, Joint format changes |
| `VERSION_INLINENOTE`    | `12` | Inline quoted text in Notes and Cars |
| `VERSION_NONAKEDENDS`   | `12` | Replacement of bare `END` with typed ends (`END$SEGS`, etc.) |

---

## Forward Type Declarations

```c
typedef struct drawCmd_t * drawCmd_p;
typedef struct track_t    * track_p, * track_cp;  // pointer and const-pointer aliases
typedef struct trkSeg_t   * trkSeg_p;
typedef struct traverseTrack_t * traverseTrack_p;
typedef struct trkEndPt_t * trkEndPt_p;

// Callback type definitions:
typedef void (*doSaveCallBack_p)(void);            // save callback function pointer
typedef void (*addButtonCallBack_t)(void*);        // button add callback
typedef STATUS_T (*procCommand_t)(wAction_t, coOrd);  // command handler signature
```

---

## Extra Data Base — Polymorphic Access Pattern

A base structure for all "extra data" types associated with tracks. Each concrete type (e.g., `turnoutExtraData_t`, `structureExtraData_t`) contains this struct as its **first element**, enabling safe cast-based access:

```c
typedef struct extraDataBase_t {
    TRKTYP_T trkType;      // Track type identifier shared by all extras
} extraDataBase_t;
```

### Macro for Type-Safe Access

```c
// Usage pattern:
extraDataBase_t * GetTrkExtraData(track_p, TRKTYP_T);

#define GET_EXTRA_DATA(TRK, TRKTYP, TYPE) \
    ((TYPE*)GetTrkExtraData((TRK), (TRKTYP)))
```

This macro allows code to retrieve the correct extra-data structure for a track of a given type (`TRKTYP`) and cast it directly to a specific concrete type. This works because all `extraData*_t` types begin with an `extraDataBase_t`.

---

## tieData_t — Tie (Railroad Tie) Data Structure

```c
typedef struct {
    BOOL_T valid;      // Is this tie data considered valid?
    DIST_T length;     // Length of the tie
    DIST_T width;      // Width of the tie
    DIST_T spacing;    // Spacing between ties (for track gauge calculations)
} tieData_t, *tieData_p;
```

---

## Macro: `EXPORT`

A no-op macro used as a marker for exported symbols. It is defined but does nothing on its own — presumably paired with build configuration macros that control symbol visibility at compile time.

```c
#define EXPORT   // no-op placeholder
```

---

## Macro: `COUNT`

Convenience macro to get the number of elements in an array:

```c
#define COUNT(A)  (sizeof(A)/sizeof(A[0]))
```

---

## Summary of Key Concepts

1. **`common.h` is a foundational header** — most source files include it for basic types and macros.
2. **The DYNARRAY abstraction** provides a minimal dynamic array without requiring a full heap allocator in the type definition itself (allocation happens externally via `MyRealloc`).
3. **Platform-specific defines** handle Windows vs. Unix differences cleanly, with fallback implementations for missing POSIX functions on Windows.
4. **`extraDataBase_t` enables polymorphic access** to track extra-data structures without requiring virtual-like dispatch tables at runtime — relies instead on C-style casts after retrieving the base struct.
