# ctext.c — Text Command Implementation

## Overview

`ctext.c` implements the **Text** command in XTrkCAD, which allows users to place multi-line text labels (annotations) on track plans. The command uses a popup dialog for entering/editing text, with controls for font size, color, and boxed/unboxed display options.

---

## Command State Machine

The command maintains an internal state tracked by `Dt.state`:

| State | Description |
|-------|-------------|
| `POSITION_TEXT` | Initial/finished state — waiting to place new text. The cursor is positioned at the last created text or center of screen. |
| `SHOW_TEXT` | Text entry mode — user types characters into a preview area; pressing Enter/Ctrl+Enter commits the text. |

---

## Data Structure: `Dt`

```c
static struct {
    STATE_T state;                          /* command state (POSITION_TEXT / SHOW_TEXT) */
    CSIZE_T len;                            /* current character count in buffer */
    coOrd cursPos0, cursPos1;              /* cursor position for preview drawing */
    POS_T cursHeight;                       /* height of the last rendered line */
    POS_T textLen;                          /* total width of the multi-line text */
    POS_T lastLineLen;                      /* width of the most recent (last) line */
    POS_T lastLineOffset;                   /* vertical offset from previous lines */
    coOrd pos;                              /* user-placed insertion point (selected by mouse click or Ctrl+click for centering) */
    ANGLE_T angle;                          /* optional rotation angle for the text block */
    long size;                              /* current font size in points */
    long fontSizeInx;                       /* index into the font-size dropdown list */
    char text[STR_HUGE_SIZE];              /* null-terminated buffer for user-entered text */
    wDrawColor color;                       /* stroke color (e.g., black, white, etc.) */
    BOOL_T boxed;                           /* whether to draw a rectangular box around the text */
} Dt;
```

---

## Parameter Group (`textPG`)

The command exposes three user-editable parameters via a `paramGroup`:

| Parameter | Type | Control | Description |
|-----------|------|---------|-------------|
| `fontsize` | Dropdown list (indexed by `Dt.fontSizeInx`) | `wList_p` | Font size in points; changing it updates the preview live. |
| `color` | Color picker (`wDrawColor`) | Color list control | Stroke color for the text label. |
| `boxed` | Toggle button | Boolean flag | Whether to draw a rectangular box around the entire multi-line block. |

These are registered with `ParamRegister(&textPG)` and later loaded/recorded via the standard param system (`ParamLoadControls`, `ParamGroupRecord`).

---

## Dialog Update Callback: `TextDlgUpdate`

Whenever a parameter changes (e.g., font size changed), this callback is invoked to update the preview:

```c
static void TextDlgUpdate(paramGroup_p pg, int inx, void *context) {
    coOrd size, lastline;

    switch (inx) {
        case 0:   // fontsize
        case 1:   // color
        case 2:   // boxed
            UpdateFontSizeList( &Dt.size, (wList_p)textPLs[0].control, Dt.fontSizeInx );
            if ( Dt.state == SHOW_TEXT ) {
                DrawMultiLineTextSize( &mainD, Dt.text, NULL, Dt.size, TRUE, &size, &lastline);
                Dt.textLen = size.x;
                Dt.lastLineLen = lastline.x;
                Dt.lastLineOffset = lastline.y;
            }
            wSetSelectedFontSize((wFontSize_t)Dt.size);   //Update for next time
            DrawTextSize( &mainD, "Aquilp", NULL, Dt.size, TRUE, &size );
            Dt.cursHeight = size.y;
            if ( Dt.state == SHOW_TEXT ) {
                Dt.cursPos0.x = Dt.cursPos1.x = Dt.pos.x+Dt.lastLineLen;
                Dt.cursPos1.y = Dt.pos.y+Dt.cursHeight+Dt.lastLineOffset;
            }
            break;
    }
}
```

Key points:

- `UpdateFontSizeList(...)` maps the current font size to an index.
- If we're in `SHOW_TEXT` state, it redraws the multi-line preview using `DrawMultiLineTextSize`.
- The cursor position for text entry is recomputed based on the new line width and height.
- A sample string `"Aquilp"` is drawn to show what the font size looks like.

---

## Command Actions (`CmdText`)

| Action | Behavior |
|--------|----------|
| `C_START` | Initialize state, create param controls (if not already), load current settings from the parameter system, display the preview with a sample string ("Aquilp"), show info controls for the dropdown/list/color picker. Returns `C_CONTINUE`. |
| `C_DOWN` | User pressed the **Down** arrow key — advance to `SHOW_TEXT` state, position cursor below the last line (or at center if no text yet), redraw preview with sample string. Returns `C_CONTINUE`. |
| `C_MOVE` | Handle mouse movement: update `Dt.pos`, recompute cursor position based on line wrapping info (`lastLineLen`, `lastLineOffset`). Returns `C_CONTINUE`. |
| `C_UP` | Up arrow — currently a no-op (placeholder). Returns `C_CONTINUE`. |
| `C_TEXT` | Process typed character. Handle backspace/delete, newline (`\n`) to wrap to next line, carriage return (`\015`, Enter) to commit the text via `NewText()`. Truncate if buffer too large (> 248 chars). Redraw preview with updated text and recompute cursor position. Returns `C_CONTINUE` or `C_TERMINATE` after committing. |
| `C_REDRAW` | Called during redraw loops — draw a single-line cursor at the current caret position using `DrawLine`. Also redraw the full multi-line string (`DrawMultiString`) so it's visible behind the cursor. Returns `C_CONTINUE`. |
| `C_CANCEL` | User pressed Esc or clicked Cancel: restore state to `POSITION_TEXT`, hide info controls. Returns `C_TERMINATE`. |
| `C_OK` | User clicked OK (or typed Enter). If in `SHOW_TEXT` with non-empty text, call the same logic as `C_TEXT`'s CR case (create a track object via `NewText`). Otherwise cancel. Returns `C_TERMINATE`. |
| `C_FINISH` | Called when the command is finishing — if there's pending text (state != POSITION_TEXT and len > 0), commit it; otherwise cancel. Returns `C_TERMINATE`. |
| `C_CMDMENU` | Display a popup menu (`textPopupM`) containing font selection options. Returns `C_CONTINUE`. |

---

## Text Creation: `NewText()`

When the user commits text, the command calls:

```c
t = NewText( 0, Dt.pos, Dt.angle, Dt.text, (CSIZE_T)Dt.size, Dt.color, Dt.boxed );
```

This allocates a new track object of type "text" with the given position, angle, content string, font size, color, and boxed flag. The returned pointer `t` is then drawn via `DrawNewTrack(t)` to give immediate visual feedback.

---

## Initialization: `InitCmdText`

```c
void InitCmdText( wMenu_p menu ) {
    AddMenuButton( menu, CmdText, "cmdText", _("Text"),
                   wIconCreatePixMap(text_image3[iconSize]), LEVEL0_50,
                   IC_STICKY|IC_CMDMENU|IC_POPUP2, ACCL_TEXT, NULL );
    textPopupM = MenuRegister( "Text Font" );
    wMenuPushCreate( textPopupM, "", _("Fonts..."), 0, SelectFont, NULL );
    Dt.size = (CSIZE_T)wSelectedFontSize();
    Dt.color = wDrawColorBlack;
}
```

- Adds a menu button to the main command menu.
- Registers a popup submenu (`"Text Font"`) containing font-selection commands.
- Initializes `Dt.size` from the current system-wide selected font size (which may have been changed by other modules).
- Default color is black.

---

## Initialization: `InitTrkText`

```c
void InitTrkText( void ) { }
```

An empty stub — no per-track initialization is needed for text objects.

---

## Summary Table

| Component | Purpose | Key Detail |
|-----------|---------|------------|
| `CmdText` | Main command handler | State machine with preview; accepts typed input via Ctrl+click or mouse click to position, then types characters with Enter/OK to commit. |
| `NewText()` | Factory for text track objects | Called when the user commits text; allocates a new track entry of type TEXT. |
| `TextDlgUpdate` | Live preview update | Invoked on parameter changes (font size/color) while in SHOW_TEXT state; redraws sample string and cursor position. |
| `InitCmdText` | Registration | Adds menu button, registers popup font submenu, initializes default settings. |

---

## Design Notes

- **Multi-line text is supported.** The command buffers characters until the user presses Enter (`\015`) or clicks OK. Each newline inserts a line feed character into the buffer and advances the cursor to the next visual line in the preview.
- **The `boxed` parameter** draws a rectangle around all lines of the multi-line text block — useful for grouping labels that belong together (e.g., a legend or caption).
- **Font size is global.** The current font size comes from the system-wide `"fontsize"` preference, which can be changed in other contexts. This command respects that setting via `wSelectedFontSize()`.
- **Truncation** is handled: if the user types more than 248 characters (buffer size minus room for terminator), it's truncated and a beep/alert is emitted.
