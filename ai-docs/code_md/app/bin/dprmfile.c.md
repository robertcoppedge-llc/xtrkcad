# dprmfile.c — Parameter Files Dialog

## Overview

`dprmfile.c` implements the **Parameter Files dialog**, which manages external `.xtp` parameter files that supplement or extend the layout data. The dialog allows users to:

- **Load** one or more parameter files (multiple selection supported)
- **Unload** individual files or all at once
- **Refresh** files to re-read from disk (useful after making edits outside XTrkCAD)
- **Mark as Favorite** — starred files appear with a star icon and are sorted to the top of the list
- **Browse** to locate `.xtp` files on disk
- **Search** for specific parameters within loaded files

The dialog uses icons (green dot = compatible, yellow dot = needs fit adjustment, red dot = not usable/unloaded) to indicate each file's compatibility state.

---

## Core Data Structures

### Parameter File Dialog (`paramFilePG`)

```c
static wWin_p paramFileW;           // Window handle for the dialog
static long paramFileSel;           // Toggle: FALSE = show file names, TRUE = show contents (short name)

/* Icon indicators */
typedef enum {
    PARAMFILE_UNLOADED   = 0,       // File has been unloaded / deleted
    PARAMFILE_NOTUSABLE  = 1,       // File cannot be loaded (incompatible or broken)
    PARAMFILE_COMPATIBLE = 2,       // File is fully compatible and usable
    PARAMFILE_FIT        = 3         // File needs some fitting adjustments
} paramFileState_t;

static wIcon_p indicatorIcons[ 2 ][PARAMFILE_MAXSTATE];
// [STANDARD_PARAM / FAVORITE_PARAM][UNLOADED|NOTUSABLE|COMPATIBLE|FIT]
```

### Parameter List (`paramFilePLs`)

| Index | Name | Type | Field | Purpose |
|-------|------|------|-------|---------|
| 0 | `I_PRMFILLIST` | DropList | — | Lists parameter files with icon indicators and text labels |
| 1 | `I_PRMFILTOGGLE` | ToggleButton | `paramFileSel` | Toggles between "Show File Names" and "Show Contents" view mode |
| 2 | Message control | Static text | — | Displays status messages (e.g., "One parameter file reloaded.") |
| 4 | Favorite button | Button | — | Marks selected files as favorite / remove star |
| 5 | Unload button | Button | — | Unloads selected parameter file(s) |
| 6 | Refresh button | Button | — | Reloads selected files from disk |
| 7 | Browse button | Button | — | Opens a file dialog to locate `.xtp` files |

---

## Key Functions

### Loading and Sorting the Parameter File List

```c
void ParamFileListLoad(int paramFileCnt, dynArr_t *paramFiles)
{
    DynString description;
    DynStringMalloc(&description, STR_SHORT_SIZE);
    int *sortedIndex = MyMalloc(sizeof(int)*paramFileCnt);
    int log_params = LogFindIndex("params");

    /* Sort by compatibility state first (loaded > fit needed > compatible > unloaded),
       then alphabetically by contents name */
    SortParamFileList(paramFileCnt, paramFiles, sortedIndex);

    wControlShow((wControl_p)paramFileL, FALSE);  // Hide list while repopulating
    wListClear(paramFileL);

    for (int i = 0; i < paramFileCnt; i++) {
        paramFileInfo_t paramFileInfo = DYNARR_N(paramFileInfo_t, (*paramFiles), sortedIndex[i]);
        if (paramFileInfo.valid) {
            DynStringClear(&description);
            /* If in "Show Contents" mode, show the short name; otherwise show full path */
            DynStringCatCStr(&description,
                (!paramFileSel && paramFileInfo.contents) ?
                paramFileInfo.contents : paramFileInfo.name);

            wListAddValue(paramFileL,
                           DynStringToCStr(&description),
                           indicatorIcons[paramFileInfo.favorite][paramFileInfo.trackState],
                           I2VP(sortedIndex[i]));
        }
    }
    wControlShow((wControl_p)paramFileL, TRUE);  // Show list again

    DynStringFree(&description);
    MyFree(sortedIndex);
}
```

**Sort order:** Files are sorted by `trackState` (compatibility status) first, then alphabetically by their contents name. This ensures that fully loaded files appear at the top, followed by those needing adjustment, and finally unloaded/incompatible ones at the bottom. The sort uses an index array rather than moving elements in the underlying array, preserving the original order if needed elsewhere.

---

### Sorting Comparator

```c
int CompareParameterFiles(const void *index1, const void *index2)
{
    paramFileInfo_t paramFile1 = DYNARR_N(paramFileInfo_t, (*sortFiles), *(int*)index1);
    paramFileInfo_t paramFile2 = DYNARR_N(paramFileInfo_t, (*sortFiles), *(int*)index2);

    if (paramFile2.trackState != paramFile1.trackState) {
        return (paramFile2.trackState - paramFile1.trackState);
    } else {
        return strcmp(paramFile1.contents, paramFile2.contents);
    }
}
```

The comparator returns a positive value when file 2 should come before file 1 (descending by `trackState`), and sorts alphabetically as a tiebreaker.

---

### Updating the Favorite Button State

```c
static void UpdateParamFileButton(void)
{
    wIndex_t selcnt = wListGetSelectedCount(paramFileL);

    // Nothing selected → disable favorite button
    if (selcnt <= 0) { return; }

    paramFilePLs[I_PRMFILEFAVORITE].context = FALSE;  // Default to "unset" state

    wIndex_t cnt = wListGetCount(paramFileL);
    for (wIndex_t inx = 0; inx < cnt; inx++) {
        if (wListGetItemSelected((wList_p)paramFileL, inx)) {
            wIndex_t fileInx = VP2L(wListGetItemContext(paramFileL, inx));
            if (!IsParamFileFavorite(fileInx)) {
                paramFilePLs[I_PRMFILEFAVORITE].context = I2VP(TRUE);  // Enable button
            }
        }
    }
}
```

The favorite button is enabled only when at least one selected file is **not** already a favorite. This prevents redundant clicks and provides immediate visual feedback that the action will change something.

---

### Marking Files as Favorite / Un-favoriting

```c
void UpdateParamFileProperties(bool newState)
{
    wIndex_t inx, cnt;
    wIndex_t fileInx;

    cnt = wListGetCount(paramFileL);
    for (inx = 0; inx < cnt; inx++) {
        if (wListGetItemSelected((wList_p)paramFileL, inx)) {
            fileInx = VP2L(wListGetItemContext(paramFileL, inx));
            SetParamFileFavorite(fileInx, newState);
        }
    }
    DoChangeNotification(CHANGE_PARAMS);  // Notify other modules of favorite status change
}

static void ParamFileFavorite(void *setFavorite)
{
    wIndex_t selcnt = wListGetSelectedCount(paramFileL);
    wMessageSetValue(MESSAGETEXT, "");
    if (selcnt) {
        UpdateParamFileProperties(setFavorite ? TRUE : FALSE);  // Toggle favorite state
    }
}
```

The "Favorite" button toggles the favorite flag on all selected files. Favorite status is stored in a file registry and persisted across sessions (the icon changes from green/yellow/red dot to a corresponding star). The `CHANGE_PARAMS` notification ensures any UI that relies on favorite status (e.g., sorting) stays in sync.

---

### Unloading and Refreshing Parameter Files

```c
static void ParamChangeSelectedFiles(unsigned paramFileChange)
{
    wIndex_t cnt = wListGetCount(paramFileL);
    for (wIndex_t inx = 0; inx < cnt; inx++) {
        if (wListGetItemSelected((wList_p)paramFileL, inx)) {
            wIndex_t fileInx = VP2L(wListGetItemContext(paramFileL, inx));

            switch (paramFileChange) {
                case PARAMFILE_UNLOAD:
                    if (IsParamFileFavorite(fileInx)) {
                        SetParamFileDeleted(fileInx, TRUE);  // Mark as deleted for favorites
                    } else {
                        UnloadParamFile(fileInx);
                    }
                    break;

                case PARAMFILE_REFRESH:
                    /* If it's a favorite and marked as deleted, "refresh" means re-adding it */
                    if (IsParamFileFavorite(fileInx) && IsParamFileDeleted(fileInx)) {
                        SetParamFileDeleted(fileInx, FALSE);
                    } else {
                        ReloadParamFile(fileInx);  // Re-read from disk
                    }
                    break;

                default:
                    CHECKMSG(FALSE, ("Invalid change type %d in ParamChangeSelectedFiles", paramFileChange));
            }
        }
    }
    ParamFileListLoad(paramFileInfo_da.cnt, &paramFileInfo_da);  // Rebuild list
    DoChangeNotification(CHANGE_PARAMS);
}

static void ParamRefreshSelectedFiles(void *action)
{
    wIndex_t selcnt = wListGetSelectedCount(paramFileL);
    if (selcnt) {
        DynString reloadMessage;
        DynStringMalloc(&reloadMessage, 16);
        ParamChangeSelectedFiles(PARAMFILE_REFRESH);

        if (selcnt > 1) {
            DynStringPrintf(&reloadMessage, _("%d parameter files reloaded."), selcnt);
        } else {
            DynStringCatCStr(&reloadMessage, _("One parameter file reloaded."));
        }
        wMessageSetValue(MESSAGETEXT, DynStringToCStr(&reloadMessage));
        DynStringFree(&reloadMessage);
    } else {
        wBeep();  // No files selected — beep to warn user
    }
}

static void ParamUnloadSelectedFiles(void *action)
{
    wIndex_t selcnt = wListGetSelectedCount(paramFileL);
    wMessageSetValue(MESSAGETEXT, "");
    if (selcnt) {
        ParamChangeSelectedFiles(PARAMFILE_UNLOAD);
    } else {
        wBeep();  // No selection — beep to warn user
    }
}
```

**Key design decisions:**

- **Favorite files are treated specially when unloading:** They're marked as "deleted" rather than immediately unloaded. This allows them to be re-added later by a simple "refresh" operation without requiring the full load path again.
- **`ReloadParamFile()` vs `SetParamFileDeleted(FALSE)`:** For non-favorite files, `ReloadParamFile` is called (which reads from disk and rebuilds internal structures). For favorite files that were previously marked deleted, simply clearing the "deleted" flag reactivates them without reloading.
- **User feedback:** If no files are selected, a beep sounds to alert the user. Status messages inform how many files were reloaded/unloaded.

---

### Dialog Update Handler

```c
static void ParamFileDlgUpdate( paramGroup_p pg, int inx, void *valueP )
{
    switch (inx) {
        case I_PRMFILLIST:          // List selection changed
            UpdateParamFileButton(); // Re-evaluate favorite button enable state
            break;
        case I_PRMFILTOGGLE:        // Toggle between "Show Names" and "Show Contents" view
            DoChangeNotification(CHANGE_PARAMS);  // Notify that the displayed names changed
            break;
    }
}
```

The handler is simple — it only re-evaluates whether the favorite button should be enabled after a selection change. The toggle control's callback triggers `DoChangeNotification(CHANGE_PARAMS)` so that any code listening for parameter changes will see the view mode switch.

---

### Opening the Dialog

```c
void DoParamFiles(void *junk)
{
    void *data;
    if (paramFileW == NULL) {
        /* Create icons */
        indicatorIcons[STANDARD_PARAM][PARAMFILE_UNLOADED]     = wIconCreatePixMap(greydot_image1);
        indicatorIcons[STANDARD_PARAM][PARAMFILE_NOTUSABLE]   = wIconCreatePixMap(reddot_image1);
        indicatorIcons[STANDARD_PARAM][PARAMFILE_COMPATIBLE]  = wIconCreatePixMap(yellowdot_image1);
        indicatorIcons[STANDARD_PARAM][PARAMFILE_FIT]         = wIconCreatePixMap(greendot_image1);
        indicatorIcons[FAVORITE_PARAM][PARAMFILE_UNLOADED]    = wIconCreatePixMap(greystar_image1);
        indicatorIcons[FAVORITE_PARAM][PARAMFILE_NOTUSABLE]   = wIconCreatePixMap(redstar_image1);
        indicatorIcons[FAVORITE_PARAM][PARAMFILE_COMPATIBLE]  = wIconCreatePixMap(yellowstar_image1);
        indicatorIcons[FAVORITE_PARAM][PARAMFILE_FIT]         = wIconCreatePixMap(greenstar_image1);

        ParamRegister(&paramFilePG);

        paramFileW = ParamCreateDialog(&paramFilePG,
                                       MakeWindowTitle(_("Parameter Files")), _("Done"),
                                       ParamFileOk, ParamCancel_Null, TRUE, NULL,
                                       F_RESIZE | F_RECALLSIZE,
                                       ParamFileDlgUpdate);

        /* File chooser for the "Browse..." button */
        paramFile_fs = wFilSelCreate(mainW, FS_LOAD, FS_MULTIPLEFILES,
                                     _("Load Parameters"), _("Parameter files (*.xtp)|*.xtp"),
                                     LoadParamFile, NULL);
    }
    ParamLoadControls(&paramFilePG);  // Populate list with current contents
    ParamGroupRecord(&paramFilePG);   // Record user's selection for later restore
    if ((wListGetValues(paramFileL, NULL, 0, NULL, &data)) >= 0) {
        UpdateParamFileButton();      // Enable/disable favorite button based on current selection
    }
    wShow(paramFileW);
}
```

Notes:
- Icons are created lazily on first dialog open.
- `F_RECALLSIZE` allows the dialog to restore a previously set window size (useful if the user resized it and reopens it).
- The file chooser (`paramFile_fs`) is also created lazily — used by the "Browse..." button to let users find `.xtp` files on disk.

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `CompareParameterFiles()` | Sort comparator for qsort | Primary key: trackState (descending); secondary: contents name (alphabetical) |
| `SortParamFileList()` | Build a sorted index array | Avoids moving elements in the underlying array; uses an index permutation |
| `ParamFileListLoad(cnt, files)` | Repopulate the list box with current parameter files | Filters out invalid entries; applies sort order |
| `UpdateParamFileButton()` | Evaluate whether favorite button should be enabled | Enabled only if at least one selected file is not already a favorite |
| `ParamFileFavorite(void *)` | Toggle favorite status on selected files | Calls `SetParamFileFavorite()` for each selected item |
| `UpdateParamFileProperties(bool)` | Apply favorite/unfavorite to all selected items | Triggers `CHANGE_PARAMS` notification |
| `ParamChangeSelectedFiles(unsigned change)` | Unload or refresh selected parameter files | Handles favorites specially (mark as deleted vs. reload) |
| `ParamRefreshSelectedFiles(void *)` | Reload selected files from disk | Shows a message; beeps if nothing selected |
| `ParamUnloadSelectedFiles(void *)` | Unload selected files from memory | Marks favorites as "deleted"; others are unloaded |
| `ParamFileBrowse(void *)` | Open file dialog to browse for `.xtp` files | Uses `wFilSelCreate()` with `FS_MULTIPLEFILES` flag |
| `ParamFileSelectAll(void *)` | Select all entries in the list | Also re-evaluates favorite button state |
| `ParamFileOk(void *)` | Close dialog and trigger change notifications | Calls `SearchUiOk()` (shared search logic); notifies listeners |

---

## Domain & Design Notes

- **Icon indicators:** The four-state enum (`PARAMFILE_UNLOADED`, `NOTUSABLE`, `COMPATIBLE`, `FIT`) maps to a 2×4 icon matrix — standard icons for non-favorite files, star variants for favorites. This gives immediate visual feedback about file health without reading text.

- **Toggle mode:** The second column of the list box can show either full file paths or just short names (e.g., `"mytrack.xtp"` instead of `/home/user/mylayout/params/mytrack.xtp`). This is useful when paths are long and clutter the dialog.

- **Favorites persist across sessions:** Favorite status is stored in a persistent data structure (`paramFileInfo_t.favorite` flag), so starred files reappear at the top of the list after restarting XTrkCAD.

- **Change notifications:** Every modification (favorite toggle, unload, reload) triggers `DoChangeNotification(CHANGE_PARAMS)`, which propagates to any module that needs to know about parameter file changes (e.g., updating a toolbar indicator, refreshing a status bar).

- **Beep on empty selection:** A consistent UX pattern here — if the user clicks "Favorite" or "Refresh" with nothing selected, a system beep warns them rather than silently doing nothing.
