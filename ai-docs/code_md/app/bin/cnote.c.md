# cnote.c — Main Layout Notes Utility

## Overview

`cnote.c` implements the **main layout note** feature — a single, global text area where users can paste notes about their model railroad project. This is not per-track or per-object; it's a single document that persists across sessions and is saved to the layout file (`.xtc`).

Think of it as a `README` or `CHANGELOG.md` embedded inside the CAD project itself — useful for:
- Documenting design decisions ("This turnout is N-scale, so use 16.5mm gauge")
- Recording progress notes ("Complete bridge segment A on 2026-08-01")
- Listing references ("See http://railroads.example.org for signal timing specs")

---

## Data Structures

### `mainText` — The Note Text Buffer

```c
static char * mainText = NULL;
```

A dynamically allocated string holding the current note text. It starts as `NULL` (empty) and is only allocated when the user types something in the dialog. Memory is freed on exit via `ClearNote()`.

### `notePG` — Parameter Group for the Note Dialog

```c
static paramGroup_t notePG = { "note", 0, notePLs, COUNT(notePLs) };
```

A parameter group that wraps a single text area control. The dialog is created once and reused across invocations (unlike `Describe`, which reuses controls from a larger shared pool).

### `notePLs[]` — Parameter Definition Array

```c
static paramData_t notePLs[] = {
    { PD_TEXT, NULL, "text", PDO_DLGRESIZE, &noteTextData }
};
```

A single entry defining one text area control:
- **Type:** `PD_TEXT` — a multi-line text field (not a single-line input box)
- **Label:** `"text"` — shown as the column header in the dialog layout system
- **Option flag:** `PDO_DLGRESIZE` — allows the window to resize horizontally to fit its contents

### `noteTextData` — Text Field Parameters

```c
static paramTextData_t noteTextData = { 300, 150 };
```

Width and height in pixels for the text area widget. These are passed to `wTextSetSize()` when the dialog is resized.

---

## Core Functions

### `ClearNote(void)` — Free Memory on Exit

Called from `NoteOk()` before displaying the new text, and from program shutdown. It frees the previously allocated buffer so that `mainText` can be reassigned to a newly-allocated string.

```c
void ClearNote(void) {
    if (mainText) {
        MyFree(mainText);
        mainText = NULL;
    }
}
```

---

### `NoteOk(void *unused)` — Dialog OK Callback

Invoked when the user clicks the dialog's OK button:

1. Checks whether the text area has been modified (the window tracks this internally). If not, the user just opened and closed the dialog without editing anything — do nothing.
2. Allocates a new buffer sized for the current content (`len+2` accounts for possible UTF-8 multi-byte encoding; note that `strlen()` counts bytes, so this is an upper bound on bytes, not characters).
3. Calls `wTextGetText()` to copy the contents into `mainText`.
4. Hides and destroys the dialog window.

---

### `DoNote(void *unused)` — Show the Dialog

The top-level entry point for opening the note editor:

```c
void DoNote(void * unused) {
    if (noteW == NULL) {
        // Create the parameter group window once, on first call
        noteW = ParamCreateDialog(&notePG, MakeWindowTitle(_("Note")),
                                  _("Ok"), NoteOk,
                                  ParamCancel_Current, FALSE, NULL,
                                  F_NOTTRANSIENT|F_RESIZE, NULL);
    }

    wTextClear(noteT);               // Clear the text area display
    wTextAppend(noteT, mainText ? mainText :
                _("Replace this text with your layout notes"));  // Show placeholder if empty
    wTextSetReadonly(noteT, FALSE); // Enable editing
    wShow(noteW);                   // Display the window
}
```

**Design note:** The dialog is created once and reused. On subsequent calls (e.g., from a keyboard shortcut), it simply clears its contents and shows them again. This avoids flicker and saves memory allocation overhead.

The placeholder text `_("Replace this text with your layout notes")` gives the user immediate feedback that they can start typing right away. It only appears if `mainText` is empty (i.e., no note has been set yet).

---

### `WriteMainNote(FILE *f)` — Save Note to File

Saves the current note text into an XTC layout file:

```c
BOOL_T WriteMainNote(FILE * f) {
    BOOL_T rc = TRUE;
    char *noteText = mainText;  // Local copy for possible reassignment

    if (noteText && *noteText) {
#ifdef UTFCONVERT
        char *out = NULL;
        if (RequiresConvToUTF8(mainText)) {
            size_t cnt = strlen(mainText) * 2 + 1;
            out = MyMalloc(cnt);
            wSystemToUTF8(mainText, out, (unsigned int)cnt);
            noteText = out;
        }
#endif // UTFCONVERT

        char * sText = ConvertToEscapedText( noteText );
        rc &= fprintf(f, "NOTE MAIN 0 0 0 0 0 \"%s\"\n", sText )>0;
        MyFree( sText );

#ifdef UTFCONVERT
        if (out) {
            MyFree(out);
        }
#endif // UTFCONVERT
    }
    return rc;
}
```

**Key behaviors:**

- **UTF-8 handling:** If compiled with `UTFCONVERT`, the note is checked for non-ASCII characters. If found, they are encoded to UTF-8 (since the XTC file format uses UTF-8 strings). The original string's memory is replaced by the new byte buffer.
  
- **Escaping:** `ConvertToEscapedText()` escapes special characters (`"`, `\`) so that the note can be safely stored inside a quoted C string literal in the XTC file.

- **File format line:** `"NOTE MAIN 0 0 0 0 0 \"..."` — the fields after `MAIN` are placeholders (version 0, meaning no special flags). The entire text is enclosed in double quotes and must not contain unescaped quote characters.

---

### `ReadMainNote(char *line)` — Load Note from File

Parses a line of the form:

```
NOTE MAIN version major minor patch "encoded text"
```

The version fields (`0 0 0`) allow future extensions (e.g., adding a character offset for embedded notes). The `q` flag in the format string indicates that the quoted field follows.

**Version handling:**

- **Version < 3:** Uses a legacy reader (`ReadMultilineText()`) — presumably a different file format or pre-1.x behavior.
- **Version >= 3 and < VERSION_INLINENOTE:** Same as above, for intermediate versions.
- **Version >= VERSION_INLINENOTE:** Reads the inline quoted string directly from the file.

The memory of any previously-loaded note is freed before assignment, ensuring no leaks even if the same layout file is opened twice in a row.

---

## Summary Table

| Function | Purpose | Key Behavior |
|----------|---------|--------------|
| `ClearNote()` | Free previously-allocated note text buffer | Called before reassigning `mainText` to avoid leak |
| `DoNote()` | Show the note editor dialog (create once, reuse thereafter) | Clears placeholder if empty; enables editing |
| `WriteMainNote(FILE*)` | Serialize note into XTC file format | Escapes quotes/backslashes; optionally converts non-ASCII chars to UTF-8 |
| `ReadMainNote(char*)` | Deserialize a note from an XTC file line | Handles multiple version formats; frees old buffer before reassigning |

---

## Design Decisions & Tradeoffs

### Why Not Store the Note in Memory Only?

One might argue that project notes are ephemeral — "I just wanted to jot this down for today" — and don't need persistence. But in practice, notes accumulate over years of work. A model railroad builder may want to document:
- What gauge is being used and why (e.g., "HO scale because most rolling stock available at my local hobby shop")
- Where a specific bridge segment was ordered from
- Wiring schematics or signal timing values that are unlikely to be re-derived

Embedding the note directly in the layout file ensures it travels with the project. If the user later changes their mind and deletes the note, they can simply delete that single line from the XTC file — no special "delete note" UI is needed.

### Why Not Use a Separate File?

Separate files (`notes.txt`, `README.md`) have several downsides in this context:
- Users may forget to open or edit them
- They don't automatically appear alongside the layout when opened with XTrkCAD
- Version control can get messy if notes are edited independently of the CAD data

By contrast, embedding the note as a single line in the layout file guarantees it's always present. The UI (the dialog) is merely an editor — users could also edit the `.xtc` file directly if they prefer.

### Why Not Use a Rich Text Format?

The current implementation uses plain text with escaped quotes/backslashes. This is intentional:
- **Simplicity:** No need for a rich-text editor component; a single `wText` widget suffices.
- **Portability:** Any text editor can open and edit the `.xtc` file without special software.
- **Size:** Notes are typically short (a few sentences). Even 10KB of notes is negligible compared to a full layout file.

If users want rich formatting, they could use an external wiki or documentation system linked from within the project folder structure.

---

## Summary

| Category | Content |
|----------|---------|
| **Purpose** | Single global text note attached to a model railroad layout; persists across sessions |
| **Domain** | Project documentation, embedded metadata, plain-text storage in binary/semi-binary formats |
| **Key concept** | The note is stored as a single line (`NOTE MAIN ...`) in the XTC file — no separate database or sidecar file needed |
| **Main entry point** | `DoNote()` — invoked via a palette button or keyboard shortcut; creates the dialog once and reuses it |
