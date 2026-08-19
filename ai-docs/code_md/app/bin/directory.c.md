# directory.c — Directory Management Utilities

## Overview

`directory.c` provides two simple filesystem utilities:

- **`SafeCreateDir()`** — Attempts to create a directory with mode `0755`. If it fails because the directory already exists, the failure is silently ignored (treated as success).
- **`DeleteDirectory()`** — Recursively removes all contents of a directory and then deletes the directory itself using `rmdir()`.

These functions are used by XTrkCAD's archive handling: temporary unpack directories (`zip_in.<pid>`) and temporary save directories (`zip_out.<pid>`) are created here, and cleaned up on program exit or error.

---

## `SafeCreateDir(const char *dir)`

```c
BOOL_T SafeCreateDir(const char *dir)
{
    int err;

    err = mkdir(dir, 0755);
    if (err < 0) {
        if (errno != EEXIST) {
            NoticeMessage(MSG_DIR_CREATE_FAIL, _("Continue"), NULL, dir, strerror(errno));
            perror(dir);
            return FALSE;
        }
    }
    return TRUE;
}
```

**Behavior:**

| `mkdir()` result | Action taken |
|---|---|
| `-1` with `errno == EEXIST` | Directory already exists → treated as success (`TRUE`) |
| `-1` with any other `errno` | Error dialog shown → return `FALSE` |
| `0` (success) | Return `TRUE` |

The permission mode is set to **0755** (rwxr-xr-x). On Linux this allows read/execute for everyone and write/execute for the owner. The function does not attempt to fix a directory that exists with wrong permissions — it just reports success if it already exists.

---

## `DeleteDirectory(const char *dir_path)`

```c
BOOL_T DeleteDirectory(const char *dir_path)
{
    size_t path_len;
    char *full_path = NULL;
    DIR *dir;
    struct stat stat_path, stat_entry;
    struct dirent *entry;
    DynString path;

    /* stat for the path */
    int resp = stat(dir_path, &stat_path);

    if (resp != 0 && errno == ENOENT) {
        return TRUE;      // Does not exist — treat as success
    }

    /* If path is not a directory → error */
    if (!(S_ISDIR(stat_path.st_mode))) {
        NoticeMessage(MSG_NOT_DIR_FAIL, _("Continue"), NULL, dir_path);
        return FALSE;
    }

    /* Open the directory for iteration */
    if ((dir = opendir(dir_path)) == NULL) {
        NoticeMessage(MSG_DIR_OPEN_FAIL, _("Continue"), NULL, dir_path);
        return FALSE;
    }

    /* Allocate a buffer to build full paths of entries */
    path_len = strlen(dir_path) + 1;
    DynStringMalloc(&path, path_len + 16);

    for (entry = readdir(dir); entry != NULL; entry = readdir(dir)) {
        /* Skip "." and ".." */
        if (!strcmp(entry->d_name, ".") || !strcmp(entry->d_name, "..")) {
            continue;
        }

        /* Build full path to this entry */
        DynStringReset(&path);
        DynStringCatCStrs(&path, dir_path, FILE_SEP_CHAR, entry->d_name, NULL);
        full_path = DynStringToCStr(&path);

        stat(full_path, &stat_entry);

        if (S_ISDIR(stat_entry.st_mode) != 0) {
            /* Recursive call: delete subdirectory first */
            DeleteDirectory(full_path);
            continue;
        } else {
            /* It's a file — unlink it */
            if (unlink(full_path)) {
                NoticeMessage(MSG_UNLINK_FAIL, _("Continue"), NULL, full_path);
                DynStringFree(&path);
                closedir(dir);
                return FALSE;
            }
        }

#if DEBUG
        printf("Removed a file: %s\n", full_path);
#endif
    }

    /* Cleanup loop */
    closedir(dir);
    DynStringFree(&path);

    /* Finally, remove the now-empty directory itself */
    if (rmdir(dir_path)) {
        NoticeMessage(MSG_RMDIR_FAIL, _("Continue"), NULL, dir_path);
        return FALSE;
    } else {
#if DEBUG
        printf("Removed a directory: %s\n", dir_path);
#endif
    }

    return TRUE;
}
```

**Algorithm (recursive descent):**

1. **Check existence:** `stat()` the target path. If it doesn't exist (`ENOENT`), treat as already deleted → success.
2. **Validate type:** If it's not a directory, complain and fail.
3. **Open for iteration:** Use `opendir()` to list contents.
4. **For each entry (skipping `"."` and `"..""):**
   - Build the full path with `DynStringCatCStrs()`.
   - If it is a subdirectory → recurse (`DeleteDirectory(full_path)`).
   - If it is a regular file → call `unlink()` to remove it.
5. **After the loop:** Close the directory stream and free the buffer.
6. **Finally:** Call `rmdir()` on the original path. This only succeeds because all children have been removed.

**Notes:**

- The function uses `DynString` to build full paths, avoiding stack overflow from deeply nested paths.
- Error handling is per-entry: if any unlink fails, a dialog is shown and the caller can decide whether to continue or abort (here it returns `FALSE`).
- No attempt is made to preserve permissions — subdirectories are removed and then recreated with whatever permissions `rmdir()` needs (normally 0755).

---

## Summary Table

| Function | Purpose | Notes |
|----------|---------|-------|
| `SafeCreateDir(dir)` | Create a directory (silently ignores if it already exists) | Uses mode 0755; only reports non-EEXIST errors as failures. |
| `DeleteDirectory(path)` | Recursively remove all contents, then delete the directory itself | Recursive via internal recursive call + `rmdir()` at the end. |

---

## Design Notes

- **`SafeCreateDir`** is intentionally forgiving: it treats "already exists" as a success case because archive unpacking code expects to be able to create the same temp dir repeatedly (e.g., on re-run of an import).
- **`DeleteDirectory`** assumes that the directory is empty after all children have been removed. If `rmdir()` fails for any other reason (e.g., permission denied), it reports a generic "remove failed" dialog. It does not attempt to handle non-empty directories — that would require additional logic (and potentially elevated privileges).
- The use of `DynString` for path construction avoids stack overflow on deeply nested paths and keeps the full-path string in a single allocation rather than repeatedly concatenating into a buffer.
