# doption.c — Option Dialogs (Display, Command Options, Preferences)

## Overview

`doption.c` implements several **dialog boxes** used in XTrkCAD for configuring global settings:

- **Display Options dialog** (`displayPG`) — controls what is drawn on the layout canvas (track colors, tunnel rendering, tie drawing mode, label visibility, etc.)
- **Command Options dialog** (`cmdoptPG`) — sets default command behavior (selection modes, right-click actions)
- **Preferences dialog** (`prefPG`) — global application settings like icon size, measurement units, distance format, connection tolerances
- **Color dialog** (`colorPG`) — sets drawing colors for tracks, grid, markers

The file also handles **measurement unit conversion**, **distance format selection** (English fractions or metric decimals), and the **"On Startup"** behavior (load last layout vs. start new).

---

## Core Data Structures

### Display Options Parameter Group (`displayPG`)

```c
typedef struct {
    char * name;
    long   fmt;  // distance format bitmask: DISTFMT_FMT_NONE, DISTFMT_FRACT_NUM|_MM|_CM|_M, etc.
} dstFmts_t;

static dstFmts_t englishDstFmts[] = {
    { N_("999.999"),         DISTFMT_FMT_NONE|DISTFMT_FRACT_NUM|3 },  // 3 decimal places
    { N_("999.99"),          DISTFMT_FMT_NONE|DISTFMT_FRACT_NUM|2 },
    { N_("999 7/8"),        DISTFMT_FMT_NONE|DISTFMT_FRACT_FRC|3 },  // fraction with denominator 2^n
    { N_("999' 11.99\""),   DISTFMT_FMT_SHRT|DISTFMT_FRACT_NUM|2 },  // feet & inches decimal
    { NULL, 0 }
};

static dstFmts_t metricDstFmts[] = {
    { N_("999.999"),         DISTFMT_FMT_NONE|DISTFMT_FRACT_NUM|3 },
    { N_("999.99mm"),       DISTFMT_FMT_MM|DISTFMT_FRACT_NUM|2 },
    { N_("999.9cm"),        DISTFMT_FMT_CM|DISTFMT_FRACT_NUM|1 },
    { N_("999.999m"),       DISTFMT_FMT_M|DISTFMT_FRACT_NUM|3 },
    { NULL, 0 }
};

static dstFmts_t *dstFmts[] = { englishDstFmts, metricDstFmts };
```

The `dstFmts` array maps a unit system (English vs. Metric) to an array of available format strings and their corresponding bitmasks.

### Color Palette Structure

```c
typedef struct {
    char * name;           // Human-readable name (e.g., "Normal Track")
    wDrawColor color;      // The actual GTK drawing color
} color_t, *color_p;

static color_t normalColorTab[] = {
    { N_("Red"),            wDrawFindColor("#aa3300") },
    { N_("Green"),          wDrawFindColor("#689457") },
    // ... more colors
};
```

Colors are stored as `wDrawColor` values (internal GTK color handles) and have a human-readable name for display.

---

## Key Functions

### Display Options Dialog Initialization

```c
EXPORT addButtonCallBack_t DisplayInit( void )
{
    ParamRegister( &displayPG );
    wEnableBalloonHelp( (int)enableBalloonHelp );
#ifdef LATER
    RegisterChangeNotification( DisplayChange );
#endif
    return &DoDisplay;
}
```

Registers the `displayPG` parameter group and returns a callback (`&DoDisplay`) that can be attached to a toolbar button. The dialog is created on first call (lazy creation pattern).

---

### Opening the Display Options Dialog

```c
static void DoDisplay( void *junk )
{
    if (displayW == NULL) {
        displayW = ParamCreateDialog( &displayPG, MakeWindowTitle(_("Display Options")),
                                      _("Ok"), DisplayOk, ParamCancel_Restore, TRUE, NULL, 0, OptionDlgUpdate );

        /* Populate the hotbar labels drop-down */
        wListAddValue( (wList_p)displayPLs[I_HOTBARLABELS].control, _("Proto"), NULL, I2VP(0x0002) );
        wListAddValue( (wList_p)displayPLs[I_HOTBARLABELS].control, _("Proto/Manuf"), NULL, I2VP(0x0012) );
        /* ... more entries */
    }

    ParamLoadControls( &displayPG );  // Load current values into widgets
    wShow( displayW );
}
```

The `I_HOTBARLABELS` entry is a drop-down that lets users choose which fields are shown in the car hotbar (Proto only, Proto/Manuf, etc.). The list entries include bitmasks like `0x4312` meaning "show all four levels: Proto, Manuf, Part Number, Item."

---

### Display Options Dialog Update Handler

```c
static void OptionDlgUpdate( paramGroup_p pg, int inx, void *valueP )
{
    if ( inx < 0 ) { return; }
    if ( pg->paramPtr[inx].valueP == &enableBalloonHelp ) {
        wEnableBalloonHelp((wBool_t)*(long*)valueP);
    } else {
        if (pg->paramPtr[inx].valueP == &labelEnable) {
            long new_labels = wRadioGetValue( (wChoice_p)pg->paramPtr[inx].control );
            labelEnable = new_labels;
            ParamLoadControl(&displayPG,labelSelect);  // Rebuild label enable list based on selection
        }
        if (pg->paramPtr[inx].valueP == &units) {
            UpdatePrefD();
        }
        if (pg->paramPtr[inx].valueP == &distanceFormatInx) {
            UpdateMeasureFmt();
        }
    /* ... more handlers */
    }
}
```

This is a **switch statement** that routes each control by its stored `valueP` pointer. When the user changes any control, this function is called and dispatches to appropriate sub-handlers:

- **Balloon help toggle** — enables/disables GTK's built-in balloon help system
- **Label enable radio buttons** — switching between "Track Descriptions", "Lengths", etc. rebuilds a sub-list showing which label types are active
- **Units change** — triggers `UpdatePrefD()` to reload the distance format list with English or Metric entries
- **Distance format change** — calls `UpdateMeasureFmt()` to redraw measurement labels on the layout

---

### Saving Display Options

```c
static void DisplayOk( void *junk )
{
    long changes;
    changes = GetChanges( &displayPG );  // Returns bitmask of which fields changed
    wHide( displayW );
    DoChangeNotification(changes);       // Dispatch changes to interested parties
}
```

The `GetChanges()` helper scans all parameters in the group and returns a bitfield indicating which ones were modified since last save. This is used by other modules (e.g., drawing routines) to decide whether they need to redraw.

---

### Command Options Dialog

The command options dialog controls how commands behave:

```c
static char * preSelectLabels[] = { N_("Properties"), N_("Select"), NULL };
static char * selectLabels[] = { N_("Single item selected, +Ctrl Add to selection"),
                                  N_("Add to selection, +Ctrl Single item selected"), NULL };
static char * selectZeroLabels[] = { N_("Deselect all on select nothing"), NULL };
```

| Control | Purpose |
|---------|----------|
| **Default Command** (radio) | When clicking a track piece without any command active, should it open the Properties dialog or just select? |
| **Select Mode** (radio) | How Ctrl+Click behaves — either adds to selection or toggles single-select mode |
| **Right Click** (radio) | Whether right-click opens the Command list or the Options dialog |

---

### Preferences Dialog (`prefPG`)

The preferences dialog contains global application settings:

```c
static paramData_t prefPLs[] = {
    /* 0 */ { PD_RADIO, &iconSize, "iconsize", PDO_NOPSHUPD, iconSizeLabels, N_("Icon Size"), BC_HORZ },
    /* 1 */ { PD_RADIO, &angleSystem, "anglesystem", PDO_NOPSHUPD, angleSystemLabels, N_("Angles"), BC_HORZ },

    #define I_UNITS (2)
    /* 2 */ { PD_RADIO, &units, "units", PDO_NOPSHUPD|PDO_NOUPDACT, unitsLabels, N_("Units"), BC_HORZ },

    #define I_DSTFMT (3)
    /* 3 */ { PD_DROPLIST, &distanceFormatInx, "dstfmt", PDO_DIM|PDO_NOPSHUPD|PDO_LISTINDEX, I2VP(150), N_("Length Format") },

    /* minLength — minimum track segment length allowed before a break occurs */
    /* connectDistance — maximum gap distance that two segments can be bridged */
    /* connectAngle — maximum angle difference between adjacent tracks for connection to succeed */
    /* turntableAngle — the angle threshold for detecting a turntable turnout */

    #define I_CHKPT (15)
    /* 15 */ { PD_LONG, &checkPtInterval, "checkpoint", PDO_NOPSHUPD|PDO_FILE, &i0_10000, N_("Check Point Frequency") },

    #define I_AUTOSAVE (16)
    /* 16 */ { PD_LONG, &autosaveChkPoints, "autosave", PDO_NOPSHUPD|PDO_FILE, &i0_99, N_("Autosave Checkpoint Frequency") },

    /* On Startup — whether to load the last layout or start with a new blank one */
};
```

Key notes:

- `PDO_NOUPDACT` on the units radio means changing it does *not* immediately update dependent fields (like the length format list) — instead, it waits until the user clicks OK. This prevents race conditions where the drop-down is being repopulated while the user is selecting a value.
- **Check Pointing** (`checkPtInterval`) stores how often XTrkCAD should save an autosave checkpoint to disk (in seconds). If set to 0, auto-saving is disabled.
- **Autosave Checkpoint Frequency** (`autosaveChkPoints`) — if the check point interval is non-zero *and* this frequency > 0, an additional checkpoint is written every N minutes. This provides a safety net in case the program crashes between autosaves.

---

### Updating Preferences on Unit Change

```c
static void UpdatePrefD( void )
{
    long newUnits, oldUnits;
    int inx;

    if ( prefW==NULL || (!wWinIsVisible(prefW))
         || prefPLs[I_UNITS].control==NULL ) {
        return;   // Dialog not open or control not yet created
    }

    newUnits = wRadioGetValue( (wChoice_p)prefPLs[I_UNITS].control );
    if (newUnits != displayUnits) {
        oldUnits = units;  // remember what the user had before we switched
        units = newUnits;  // switch to new unit system temporarily

        LoadDstFmtList();  // reload the drop-down with entries for the new system
        distanceFormatInx = 0;  // select first entry as default

        /* Re-populate all dimension fields (length, width, spacing) with their ranges */
        for (inx = 0; inx < COUNT( prefPLs ); inx++) {
            if ((prefPLs[inx].option&PDO_DIM)) {
                ParamLoadControl(&prefPG, inx);
            }
        }

        units = oldUnits;  // restore original unit system for internal computations
        displayUnits = newUnits;
    }
}
```

When the user switches between English and Metric, this function:

1. Saves the current `units` value as a temporary backup
2. Loads the appropriate array of distance format strings (`dstFmts[units]`)
3. Selects the first entry (usually "999.99") as default
4. Reloads all dimension-related fields so their ranges are consistent with the new units

The original unit system is restored at the end because internal calculations still use a canonical unit internally (likely inches or millimeters).

---

### Updating Measurement Format Labels

```c
static void UpdateMeasureFmt()
{
    int inx;

    distanceFormatInx = wListGetIndex((wList_p)prefPLs[I_DSTFMT].control);
    units = wRadioGetValue((wChoice_p)prefPLs[I_UNITS].control);

    for (inx = 0; inx < COUNT( prefPLs ); inx++) {
        if ((prefPLs[inx].option&PDO_DIM)) {
            ParamLoadControl(&prefPG, inx);
        }
    }
}
```

When the distance format changes, all fields that have the `PDO_DIM` option (dimension fields like track length, width) are reloaded. This ensures their displayed units and ranges match the selected measurement system.

---

### Color Dialog

```c
static paramData_t colorPLs[] = {
    /* 0 */ { PD_COLORLIST, &snapGridColor, "snapgrid", PDO_NOPSHUPD, NULL, N_("Snap Grid"), 0 },
    /* 1 */ { PD_COLORLIST, &markerColor,       "marker",     PDO_NOPSHUPD, NULL, N_("Marker"), 0 },
    /* 2 */ { PD_COLORLIST, &borderColor,       "border",     PDO_NOPSHUPD, NULL, N_("Border"), 0 },
    /* 3 */ { PD_COLORLIST, &crossMajorColor,   "crossmajor", PDO_NOPSHUPD, NULL, N_("Primary Axis"), 0 },
    /* 4 */ { PD_COLORLIST, &crossMinorColor,   "crossminor", PDO_NOPSHUPD, NULL, N_("Secondary Axis"), 0 },

    /* Track colors — these are the main drawing colors for track pieces */
    /* normalColor      — regular track segments */
    /* selectedColor    — track currently under mouse cursor (hover) */
    /* exceptionColor   — track that violates constraints or is broken */

    /* Other element colors */
    /* profilePathColor — centerline/path of a turnout/switch */
    /* tieColor         — ties (sleepers) under the rails */
    /* bridgeColor      — bridge deck color */
};
```

The color palette supports both **track segments** and **drawing elements**. Each `PD_COLORLIST` entry uses GTK's built-in color picker widget. Colors are stored as `wDrawColor` which is a pointer to an internal GTK color structure, allowing the drawing system to look up RGB values efficiently.

---

### Color Dialog Save Handler

```c
static void ColorOk( void *junk )
{
    long changes;
    changes = GetChanges( &colorPG );
    wHide( colorW );

    /* If the grid is visible and the user changed snap-grid color, redraw it */
    if ( (changes&CHANGE_GRID) && GridIsVisible() ) {
        changes |= CHANGE_MAIN;
    }

    DoChangeNotification(changes);
}
```

The `GetChanges()` call determines whether any colors were modified. The grid-change notification is special — if the snap-grid color changed and the grid is currently visible, a redraw is also triggered.

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `DisplayInit()` | Register display options dialog with change notifications | Lazily creates dialog on first call |
| `DoDisplay(junk)` | Open the Display Options dialog and populate controls | Populates hotbar labels drop-down; loads current state |
| `OptionDlgUpdate(pg, inx, valueP)` | Per-control event handler for display options | Routes by `valueP` pointer to specific sub-handlers |
| `DisplayOk(junk)` | Save changes from Display Options dialog | Calls `GetChanges()` to determine what changed |
| `CmdoptInit()` | Register command options dialog with change notifications | — |
| `PrefInit()` | Initialize Preferences dialog, validate connection parameters | Enforces min/max bounds on connectAngle/connectDistance |
| `PrefOk(junk)` | Save changes from Preferences dialog | Shows warnings if parameters are out of valid range; restarts if icon size changed |
| `ColorInit()` | Register color dialog with change notifications | — |
| `ColorOk(junk)` | Save color changes and notify relevant systems | Triggers redraw if grid is visible and its color changed |

---

## Domain & Design Notes

- **Change notification system**: All dialogs register themselves as change listeners via `RegisterChangeNotification()`. When a parameter change occurs (e.g., user moves to another dialog or the program receives an external event), all registered handlers are invoked with the appropriate bitmask of changes. This decouples the UI from the drawing/rendering code.

- **Lazy initialization**: All dialogs use the "create on first call" pattern (`if (displayW == NULL) { create... }`). This avoids creating windows until they're actually needed, which is important for startup performance and also simplifies dialog hierarchy management.

- **`PDO_NOUPDACT` flag**: Used on the units radio button to delay updating dependent fields until the user clicks OK. This prevents a race condition where `UpdatePrefD()` might try to read a control that hasn't been fully initialized yet.

- **Distance format bitmasks**: The enum values encode both the unit system (None, Millimeters, Centimeters, Meters) and the number of decimal places or fraction denominator in a single integer. This allows compact storage and easy lookup when formatting display strings.
