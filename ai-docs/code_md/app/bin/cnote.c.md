# cnote.c — Main Layout Note Management

## Overview

`cnote.c` provides the primary layout note functionality in XTrkCad. It manages a single "main" note that is stored on the layout file and displayed via a dialog box when triggered by a command. The note supports UTF-8 text (when compiled with `UTFCONVERT`) and stores its own version/size metadata for incremental saving.

## File Location

```
app/bin/cnote.c
```

## Includes & Dependencies

```c
#include "custom.h"   // custom widget types: wWin, wText, etc.
#include "dynstring.h"// dynamic memory allocation (MyMalloc)
#include "fileio.h"   // file I/O helpers
#include "misc.h"     // utility functions (GetArgs, wSystemToUTF8, etc.)
#include "param.h"    // parameter dialog framework
#include "include/utf8convert.h"  // UTF-8 conversion utilities
```

## Data Structures

### `noteTextData` (static)
A `paramTextData_t` struct holding the text control parameters for the note dialog.

```c
static paramTextData_t noteTextData = { 300, 150 };
// width = 300 pixels, height = 150 pixels
```

### `notePLs[]` (parameter list)
The parameter control list for the note dialog. Contains one text control:

```c
static paramData_t notePLs[] = {
    #define I_NOTETEXT          (0)
    #define noteT              ((wText_p)notePLs[I_NOTETEXT].control)
    { PD_TEXT, NULL, "text", PDO_DLGRESIZE, &noteTextData }  /* text control */
};
```

### `notePG` (parameter group)
The parameter group that wraps the dialog controls.

```c
static paramGroup_t notePG = { "note", 0, notePLs, COUNT(notePLs) };
```

## Functions

### `ClearNote(void)`
Releases memory allocated for the current note text. Called before reading new text to avoid leaks.

```c
void ClearNote(void)
{
    if (mainText) {
        MyFree(mainText);
        mainText = NULL;
    }
}
```

### `NoteOk(void *unused)`
Callback invoked when the user clicks **OK** in the note dialog. If the text control is marked as modified, it copies its contents into a newly-allocated buffer and assigns it to `mainText`. Then the window is hidden (`wHide`).

## Usage Notes

- The note is displayed via `DoNote()` which creates (once) or shows an existing `notePG` dialog.
- Text is always stored in `mainText`, a globally-held pointer that must be freed by `ClearNote()`.
