# ccontrol.c — Track Control Objects (Signals, Points, Detectors)

## Overview

`ccontrol.c` implements **track control objects** — special track elements that can trigger actions when a train passes over them. Controls include:
- **Signal controls** — display aspects to approaching trains
- **Point switches** — route trains through turnouts
- **Speed detectors** — measure train speed and enforce limits
- **Sensors / occupancy detectors** — detect the presence of rolling stock

Each control is stored as a `track` object (type `T_CONTROL`) with associated data that specifies:
- A position on the layout (the point where it activates)
- An optional "on script" — code to execute when triggered
- An optional "off script" — code to execute when deactivating

---

## Data Structures

### Global Constants and Variables

```c
EXPORT TRKTYP_T T_CONTROL = -1;   // Track type constant for controls
static int log_control = 0;        // Logging channel index for control messages
```

### `controlData_t` — Per-Track Control Data Structure

This struct is embedded in the extra data area of every track that has type `T_CONTROL`:

```c
typedef struct controlData_t {
    extraDataBase_t base;          // Base structure linking to track_p::extra_data
    coOrd orig;                    // 2D position (x,y) — the activation point for the control
    BOOL_T IsHilite;               // Flag indicating whether this control is currently highlighted (e.g., during editing)
    char * name;                   // Human-readable label (e.g., "Signal A", "Point SW1")
    char * onscript;              // Script/command string to execute when the control is triggered ON
    char * offscript;             // Script/command string to execute when deactivated OFF
} controlData_t, *controlData_p;
```

The `extraDataBase_t base` member links this data structure back to its owning track object. All XTrkCAD objects (tracks, signals, controls) use the same mechanism: a small header (`extraDataBase_t`) followed by type-specific data.

---

### `descData_t controlDesc[]` — Description Descriptor Table

```c
typedef enum { NM, PS, ON, OF } controlDesc_e;
static descData_t controlDesc[] = {
    /* NM */ { DESC_STRING, N_("Name"),      &controlProperties.name, sizeof(controlProperties.name) },
    /* PS */ { DESC_POS,    N_("Position"),  &controlProperties.pos },
    /* ON */ { DESC_STRING, N_("On Script"), &controlProperties.onscript, sizeof(controlProperties.onscript) },
    /* OF */ { DESC_STRING, N_("Off Script",&controlProperties.offscript, sizeof(controlProperties.offscript) },
    { DESC_NULL }
};
```

This table is used by the generic description system (`DoDescribe()`) to populate a property dialog when the user right-clicks on a control. Each entry maps a field name ("Name", "Position") to an actual string or coordinate buffer and its maximum size.

---

### `controlProperties` — Per-Object Property Buffers

```c
static struct {
    char name[STR_SHORT_SIZE];           // Buffer for the control's label string
    coOrd pos;                           // Current position of the control (used during editing)
    char onscript[STR_LONG_SIZE];        // Buffer for the ON action script
    char offscript[STR_LONG_SIZE];       // Buffer for the OFF action script
} controlProperties;
```

These buffers hold the *current* values displayed in a description dialog. They are populated by `DescribeControl()` and then used by `UpdateControlProperties()` to apply user changes back into the track's extra data.

---

## Core Functions

### `DrawControl(track_p t, drawCmd_p d, wDrawColor color)`

Renders a control as a small circle with three radial "spokes" (lines extending outward). The shape resembles a mechanical switch or signal indicator:

- A filled circle at the center (`orig`) of radius ~6 pixels.
- Three lines radiating from the center at 45°, 135°, and 225° angles, each with length ~8 pixels.

The drawing is scaled by `control_SF` (a factor of 3.0) and divided by the current layout scale ratio so that controls appear the same physical size regardless of zoom level.

```c
static void DrawControl(track_p t, drawCmd_p d, wDrawColor color) {
    controlData_p xx = GetcontrolData(t);
    DDrawControl(d, xx->orig, GetScaleRatio(GetTrkScale(t)), color);
}
```

---

### `DistanceControl(track_p t, coOrd *p)` — Distance from Activation Point

Returns the Euclidean distance between the control's activation point and a query point. This is used by collision detection or proximity-based logic (e.g., "trigger this signal when a train approaches within 10 meters").

```c
static DIST_T DistanceControl(track_p t, coOrd *p) {
    controlData_p xx = GetcontrolData(t);
    return FindDistance(xx->orig, *p);
}
```

---

### `DescribeControl(track_p trk, char *str, CSIZE_T len)` — Populate a Description Dialog

Gathers all editable properties of the selected control into a single string (e.g., `"Signal A (Layer 1): at 123.456,-789.012"`). It then calls `DoDescribe()` with the `controlDesc` table so that each field can be edited via a property dialog.

This function populates global buffers (`controlProperties.name`, etc.) which are later used to apply edits back into the track object.

---

### `UpdateControlProperties(track_p trk, int inx, descData_p descUpd, BOOL_T needUndoStart)`

Applies changes from a description dialog back into the control's extra data. It handles:
- Updating the name string (freeing old memory, allocating new)
- Moving the activation point (`orig.x`, `orig.y`)
- Swapping in new ON/OFF script strings
- Recomputing the bounding box so that collision detection works correctly

The `needUndoStart` flag indicates whether an undo transaction should be started before modifying. This is used when the user explicitly edits a property vs. automatic updates during other operations.

---

### `DeleteControl(track_p trk)` — Cleanup Before Deletion

Frees any dynamically allocated strings stored in the control's extra data (`name`, `onscript`, `offscript`). The base track object itself is freed by the caller via `DeleteTrack()`.

```c
static void DeleteControl(track_p trk) {
    controlData_p xx = GetcontrolData(trk);
    MyFree(xx->name); xx->name = NULL;
    MyFree(xx->onscript); xx->onscript = NULL;
    MyFree(xx->offscript); xx->offscript = NULL;
}
```

---

### `WriteControl(track_p t, FILE *f)` — Serialize to XTC File

Writes a control track to the layout file. The output line includes:
- Index and layer
- Scale name (e.g., "HO")
- Visibility flag
- Activation point coordinates (`orig.x`, `orig.y`)
- Name string (escaped)
- ON script (escaped, or empty if none)
- OFF script (escaped, or empty if none)

```c
static BOOL_T WriteControl(track_p t, FILE *f) {
    controlData_p xx = GetcontrolData(t);
    char *controlName = MyStrdup(xx->name);

#ifdef UTFCONVERT
    controlName = Convert2UTF8(controlName);   // Encode non-ASCII to UTF-8 for safe storage
#endif // UTFCONVERT

    rc &= fprintf(f, "CONTROL %d %u %s %d %0.6f %0.6f \"%s\" \"%s\" \"%s\"\n",
                  GetTrkIndex(t), GetTrkLayer(t), GetTrkScaleName(t),
                  GetTrkVisible(t), xx->orig.x, xx->orig.y, controlName,
                  xx->onscript, xx->offscript) > 0;

    MyFree(controlName);
    return rc;
}
```

The name and scripts are stored as plain strings. If `UTFCONVERT` is defined at compile time, they are encoded to UTF-8 before writing (so that special characters like accents or non-Latin letters don't corrupt the file).

---

### `ReadControl(char *line)` — Deserialize from XTC File

Parses a line starting with "CONTROL" and constructs a new track object:

```c
static BOOL_T ReadControl(char *line) {
    wIndex_t index;
    track_p trk;
    char *name;
    char *onscript, *offscript;
    coOrd orig;
    BOOL_T visible;
    char scale[10];
    wIndex_t layer;
    controlData_p xx;

    // Parse: CONTROL index layer scale visible x y "name" "onscript" "offscript"
    if (!GetArgs(line+7, "dLsdpqqq", &index, &layer, scale, &visible, &orig,
                 &name, &onscript, &offscript)) {
        return FALSE;
    }

#ifdef UTFCONVERT
    ConvertUTF8ToSystem(name);   // Decode UTF-8 back to system encoding
#endif // UTFCONVERT

    trk = NewTrack(index, T_CONTROL, 0, sizeof(controlData_t));
    SetTrkVisible(trk, visible);
    SetTrkScale(trk, LookupScale(scale));
    SetTrkLayer(trk, layer);

    xx = GetcontrolData(trk);
    xx->name = name;
    xx->orig = orig;
    xx->onscript = onscript;   // These strings are allocated by the reader and owned by the track
    xx->offscript = offscript;

    ComputeControlBoundingBox(trk);  // Pre-compute bounding box for collision detection
    return TRUE;
}
```

Note: The parsed string pointers (`name`, `onscript`, `offscript`) are stored directly into the extra data without being copied. This means ownership is transferred to the track object — if the track is deleted, those strings will be freed along with it. However, for safety (and because these could point into static buffers in some callers), a defensive copy might be needed depending on usage patterns.

---

### `MoveControl(track_p trk, coOrd orig)` — Translate Activation Point

Adds an offset to the control's position. This is used internally when computing transformations for a group of controls that move together (e.g., if you flip or rotate a selection). The bounding box is recomputed afterwards so collision detection remains accurate.

```c
static void MoveControl(track_p trk, coOrd orig) {
    controlData_p xx = GetcontrolData(trk);
    xx->orig.x += orig.x;
    xx->orig.y += orig.y;
    ComputeControlBoundingBox(trk);
}
```

---

### `RotateControl(track_p trk, coOrd orig, ANGLE_T angle)` — Rotate Around a Point

Currently empty. Future work may allow rotating the entire control (including its activation point) around an arbitrary pivot. This could be useful for placing controls along curved track sections where their local coordinate system should align with the tangent of the track.

```c
static void RotateControl(track_p trk, coOrd orig, ANGLE_T angle) {
    // Not yet implemented
}
```

---

### `RescaleControl(track_p trk, FLOAT_T ratio)` — Uniform Scaling

Currently empty. Could be used to scale controls when the layout scale changes (e.g., from HO to N gauge). Controls might need their activation points adjusted and perhaps resized if they contain graphical elements.

```c
static void RescaleControl(track_p trk, FLOAT_T ratio) {
    // Not yet implemented
}
```

---

### `FlipControl(track_p trk, coOrd orig, ANGLE_T angle)` — Reflect Across a Line

Reflects the control's activation point across a line defined by `(orig, angle)`. Useful for mirroring controls when flipping a track layout horizontally or vertically.

```c
static void FlipControl(track_p trk, coOrd orig, ANGLE_T angle) {
    controlData_p xx = GetcontrolData(trk);
    FlipPoint(&xx->orig, orig, angle);   // Reflect the point across (orig,angle) line
    ComputeControlBoundingBox(trk);       // Update bounding box after transformation
}
```

---

## Command Handler: `CmdControl(wAction_t action, coOrd pos)`

This is the top-level event dispatcher for the **Place Control** command palette entry. It handles placing a new control at a point on the layout by clicking:

- **C_START / C_DOWN** — Enter "place mode" and start drawing preview geometry.
- **C_MOVE** — Draw a small circle with spokes that follows the mouse (preview).
- **C_UP** — Finalize placement by calling `CreateNewControl(pos)`.
- **C_CANCEL** — Discard the pending control.

```c
static STATUS_T CmdControl(wAction_t action, coOrd pos) {
    static coOrd control_pos;   // Temporary storage for mouse position during preview
    static BOOL_T create;       // Flag indicating whether we are in "create new" mode

    switch (action) {
        case C_START:
            InfoMessage(_("Place control"));
            SetAllTrackSelect(FALSE);
            create = FALSE;
            return C_CONTINUE;

        case C_DOWN:
            create = TRUE;           // Mark that we are now in creation mode
        /* no break */
        case C_MOVE:
            SnapPos(&pos);          // Apply snapping constraints (grid, track, etc.)
            control_pos = pos;      // Store the mouse position for preview
            return C_CONTINUE;

        case C_UP:
            SnapPos(&pos);          // Re-apply snapping on release
            CreateNewControl(pos);  // Create the actual track object
            return C_TERMINATE;     // Exit command mode

        case C_REDRAW:
            if (create) {
                DDrawControl(&tempD, control_pos, GetScaleRatio(GetLayoutCurScale()), wDrawColorBlack);
            }
            return C_CONTINUE;      // Redraw the preview circle on top of existing graphics

        case C_CANCEL:
            return C_CONTINUE;

        default:
            return C_CONTINUE;
    }
}
```

---

## Control Management (Hotbar) — `ControlMgmProc`

The control management function is invoked from the hotbar when a user right-clicks or double-clicks on a control. It handles editing and deletion:

- **CONTMGM_CAN_EDIT** — Return true to allow opening an edit dialog.
- **CONTMGM_DO_EDIT** — Call `EditControl(trk)` which shows the property dialog.
- **CONTMGM_CAN_DELETE** — Always return true (controls can be deleted).
- **CONTMGM_DO_DELETE** — Delete the track object via `DeleteTrack()`.
- **CONTMGM_GET_TITLE** — Return a short description string for the context menu.

```c
static int ControlMgmProc(int cmd, void *data) {
    track_p trk = (track_p)data;
    controlData_p xx = GetcontrolData(trk);

    switch (cmd) {
        case CONTMGM_CAN_EDIT: return TRUE; break;
        case CONTMGM_DO_EDIT: EditControl(trk); return TRUE; break;
        case CONTMGM_CAN_DELETE: return TRUE; break;
        case CONTMGM_DO_DELETE: DeleteTrack(trk, FALSE); return TRUE; break;
        case CONTMGM_GET_TITLE: sprintf(message,"\t%s\t",xx->name); return 0; break;
    }
}
```

---

## Edit Dialog — `EditControl()`

When the user clicks "Edit" on a control in the hotbar, this function is called. It populates the dialog with current values and shows it:

```c
static void EditControl(track_p trk) {
    controlData_p xx;

    if (!controlEditW) {
        // Initialize or load the parameter group for editing
        ParamRegister(&controlEditPG);
        controlEditW = ParamCreateDialog(&controlEditPG,
                                         MakeWindowTitle(_("Edit control")),
                                         _("Ok"), ControlEditOk,
                                         ParamCancel_Current, TRUE, NULL, F_BLOCK, NULL);
    }

    if (controlEditTrack == NULL) {
        // No track selected yet — clear the dialog fields
        controlEditName[0] = '\0';
        controlEditOnScript[0] = '\0';
        controlEditOffScript[0] = '\0';
    } else {
        xx = GetcontrolData(controlEditTrack);
        strncpy(controlEditName, xx->name, STR_SHORT_SIZE-1);
        strncpy(controlEditOnScript, xx->onscript, STR_LONG_SIZE-1);
        strncpy(controlEditOffScript, xx->offscript, STR_LONG_SIZE-1);
        controlEditOrig = xx->orig;
    }

    ParamLoadControls(&controlEditPG);  // Fill the dialog fields with current values
    wShow(controlEditW);                 // Show the window
}
```

The dialog has four fields: name (string), X/Y position (two floats), ON script (multi-line text), OFF script (multi-line text). When the OK button is pressed, `ControlEditOk()` applies those changes back into the track.

---

## Initialization

### `InitTrkControl()` — Register Track Type and Initialize Logging

```c
EXPORT void InitTrkControl(void) {
    T_CONTROL = InitObject(&controlCmds);   // Register this as a new track type in the system
    log_control = LogFindIndex("control");  // Get a logging channel for debugging
}
```

`InitObject()` registers `T_CONTROL` with all subsystems (draw, collision detection, undo, etc.) using the command descriptor (`controlCmds`). The logging index is used for verbose debug output.

### `ControlMgmLoad()` — Build Management List

Iterates over all tracks of type `T_CONTROL` and adds them to a management list (used by the hotbar). Each control gets an icon (`wIconCreatePixMap`) so it can be represented visually in dialogs.

```c
EXPORT void ControlMgmLoad(void) {
    track_p trk;
    static wIcon_p controlI = NULL;

    if (controlI == NULL) {
        // Load a bitmap for the control icon from resources
        controlI = wIconCreatePixMap(control_image3[iconSize]);
    }

    TRK_ITERATE(trk) {
        if (GetTrkType(trk) != T_CONTROL) continue;
        ContMgmLoad(controlI, ControlMgmProc, trk);  // Add to management list
    }
}
```

---

## Summary

| Feature | Detail |
|---------|--------|
| **Track type** | `T_CONTROL = -1` — a special track subtype representing an active device |
| **Data storage** | Extra data area (`controlData_t`) holds position, label, and action scripts |
| **Drawing** | A small circle with three radial spokes; scaled by layout scale |
| **Actions** | ON/OFF scripts can trigger arbitrary commands (e.g., run a program, write to serial port) |
| **Editing** | Property dialog allows editing name, position, and scripts |
| **Persistence** | Serialized as `CONTROL` lines in the XTC file; UTF-8 encoding applied if compiled with `UTFCONVERT` |

Controls are essentially **scripted triggers** placed at arbitrary positions on the layout. When a train passes over them (detected by collision or proximity), the associated ON/OFF script is executed, allowing integration with external systems or internal actions.
