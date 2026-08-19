# archive.c — ZIP Archive Creation and Extraction

## Overview

`archive.c` implements ZIP-based file packaging for XTrkCad layout files. It uses the MinizIP library (`libzip`) to:
- **Create** archives containing an entire directory tree (all `.xtp`, images, fonts, etc.)
- **Extract** archives into a temporary working directory, filtering to only the requested layout file

This is used for:
- Saving layouts to disk in `.xtp` format
- Loading layouts from archive files
- Bundling resources (fonts, bitmaps) with the project

## File Location

```
app/bin/archive.c  (441 lines)
```

## Includes & Dependencies

| Header | Purpose |
|--------|----------|
| `fcntl.h` | File descriptor operations (`dup`, etc.) |
| `zip.h` | MinizIP library for ZIP archive manipulation |
| `archive.h` | Local declarations |
| `directory.h` | Directory creation utilities |
| `dynstring.h` | Dynamic string buffer (for path concatenation) |
| `misc.h` | General utilities (`strdup`, `MyFree`) |
| `paths.h` | Path manipulation utilities (`MakeFullpath`, `FindFilename`) |
| `utf8convert.h` | UTF-8 encoding conversion for non-UTF8 filesystems |
| `common-ui.h` | UI widget types (`NoticeMessage`) |
| `fileio.h` | File I/O utilities |

## External Variables

```c
int log_zip = 0;   // Log level for ZIP operations (when > 0, uses LOG macro)
```

## Data Structures

### `GetZipDirectoryName()` returns a dynamically allocated string containing the temporary directory path used during archive operations. The format is:

```text
<working_dir>/zip_<in|out>.<pid>
```

Where `<in>` means "read" (unpacking), `<out>` means "write" (packing), and `.pid` ensures uniqueness per process.

## Core Functions

### `GetZipDirectoryName(enum ArchiveOps op)`

Constructs the full path for a temporary directory used during archive operations.

**Parameters:**
- `op` — Operation type:
  - `ARCHIVE_READ` → "zip_in.<pid>" (unpacking target)
  - `ARCHIVE_WRITE` → "zip_out.<pid>" (packing source)
- **Returns:** Dynamically allocated path string (must be freed by caller)

### `AddDirectoryToArchive(struct zip *za, const char *dir_path, const char *prefix)`

Recursively adds all files and subdirectories under `dir_path` into the ZIP archive. The optional `prefix` parameter prepends a string to each archived entry name (e.g., "my_layout/" instead of bare filenames).

**Steps:**
1. Stat the directory — if it's not actually a directory, report error and return `FALSE`
2. Open the directory with `opendir()`
3. Iterate entries via `readdir()`:
   - Skip "." and ".."
   - Construct full system path to entry
   - If prefix given, construct archive-relative name (`prefix/entry`)
   - **If subdirectory:** call `zip_dir_add(za, arch_path)` then recurse
   - **If file:** convert paths to UTF-8 if needed, create a ZIP source via `zip_source_file()`, add with `zip_file_add()`

**Parameters:**
- `za` — Open ZIP archive handle (`struct zip *`)
- `dir_path` — Full filesystem path to the directory being archived
- `prefix` — Optional string prepended to all archived entries (NULL = no prefix)

**Returns:** `BOOL_T` — `TRUE` if successful, `FALSE` on error (reports via `NoticeMessage` and returns early)

### `CreateArchive(const char *dir_path, const char *file_name)`

Creates a ZIP archive containing the entire contents of `dir_path`, then renames it to `file_name`.

**Steps:**
1. Duplicate `file_name` (since it's `const`) — this is the desired final name
2. Extract just the filename from the full path using `FindFilename()`
3. Build a full absolute path for the archive using `MakeFullpath()`
4. Convert to UTF-8 if needed
5. Call `zip_open(..., ZIP_CREATE)` — creates or overwrites the file
6. Call `AddDirectoryToArchive(za, dir_path, "")` — add everything at root level
7. Close and finalize with `zip_close()`
8. Remove any old file with that name (`unlink`)
9. Rename temp archive to final name via `rename()` (or fall back to `Copyfile()`)

**Parameters:**
- `dir_path` — Directory whose contents are being archived
- `file_name` — Desired filename for the resulting ZIP (e.g., "my_layout.xtp")

**Returns:** `BOOL_T` — `TRUE` on success, `FALSE` on error with message via `NoticeMessage`

### `UnpackArchiveFor(const char *pathName, const char *fileName, const char *tempDir, BOOL_T file_only)`

Extracts a ZIP archive into a temporary directory, optionally filtering to extract only one specific file.

**Steps:**
1. Duplicate and convert the path name to UTF-8 if needed
2. Open with `zip_open()` — report error if it fails
3. Iterate over all entries in the archive:
   - If entry is a **directory** (`sb.name` ends with '/') and `file_only == FALSE`: create that directory with `SafeCreateDir()`
   - If entry is a **regular file**:
     - If `file_only == TRUE`, check whether this is the requested file; skip others
     - Convert archive name to system encoding if needed
     - Open output file for writing (`fopen(..., "wb")`)
     - Read from ZIP in 100-byte chunks until all bytes consumed
     - Close output file and close ZIP entry

4. Close the ZIP and return success

**Parameters:**
- `pathName` — Full path to the archive file (e.g., "/path/to/layout.xtp")
- `fileName` — Just the filename portion (used when filtering)
- `tempDir` — Where extracted files are placed
- `file_only` — If `TRUE`, only extract a single specific file; otherwise unpack everything

**Returns:** `BOOL_T` — `TRUE` if all entries processed successfully, `FALSE` on error

## Error Handling

The code uses `NoticeMessage()` to report ZIP errors (open fail, create fail, add fail, rename fail) and prints additional details to stderr. The MinizIP library's `zip_error_strerror()` provides human-readable error messages that are passed directly into the message dialog.

When an operation fails partway through (e.g., during extraction), the function returns early without cleaning up partial state — the caller typically checks the return value and handles cleanup if needed.

## Notes

- This module depends on MinizIP (`libzip`) being linked at compile time.
- On non-UTF8 filesystems (Windows with legacy code pages, old macOS HFS+, etc.), paths are converted to UTF-8 before adding to the archive and back again when extracting — this prevents corruption of filenames containing non-ASCII characters.
- The "file_only" mode is used during layout loading: extract everything first into a temp dir, then locate and move only the requested layout file plus its subdirectories to their final destination.
