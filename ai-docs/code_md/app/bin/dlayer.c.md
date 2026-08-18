# dlayer.c — Layer System (Layers, Layers Dialog, Toolbar Buttons)

## Overview

`dlayer.c` implements XTrkCAD's **layer system**, which organizes track objects into named groups with different colors and display properties. The file covers:

- **Layer initialization** (`InitLayers`, `InitLayersDialog`)
- **Layers dialog** (`DoLayer`, `LayerOk`, `LayerDlgUpdate`, etc.) — for creating/editing layers via UI
- **Layer toolbar buttons** — one per layer, plus background visibility toggle
- **Layer list dropdown** in the toolbars (setlayerL)

---

## Core Data Structures

### Layer Structure

```c
typedef struct {
    char       *name;              // Human-readable name ("Layer 1", "Tracks", etc.)
    wIndex_t    index;             // Zero-based index (0 = Layer 1, 1 = Layer 2, ...)
    BOOL_T      on;                // Whether the layer is visible
    BOOL_T      frozen;            // Frozen layers are not drawn but still selectable
    BOOL_T      modified;          // Has been changed since last save
    wDrawColor  color;             // Color associated with this layer
    BOOL_T      useColor;          // TRUE if user-defined color, FALSE = auto from palette
} layer_t, *layer_p;
```

### Layer Array

```c
static layer_t layers[10];   // Maximum number of layers (NUM_LAYERS = 10)
static wIndex_t curLayer = 0; // Currently selected layer index
static BOOL_T layoutLayerChanged;  // Flag indicating a layer change occurred
```

### Color Palette

```c
static const char *layerRawColorTab[] = {
    "#689457", "#325e6d", "#a16053", "#b17f54", "#baaaac",  // LAYERS_COLORS_0..4
    "blue",   "#ffad16", "#cfaa29", "#8d421d",                // LAYERS_COLORS_5..8
};
```

Colors are cyclically assigned to layers; layers use color index `i % COUNT(layerColorTab)`.

### Layer Button Bitmaps

Layer buttons on the toolbar display a number (e.g., "L1", "L2") rendered as a bitmap. The bitmaps are generated at runtime from character fonts stored in `bitmaps/layer_num.inc`.

```c
show_layer_bits[0..NUM_LAYERS-1]  // Bit buffer for each layer button's icon
show_layer_bmps[0..NUM_LAYERS-1]   // wIcon_p pointers to the rendered bitmaps
```

---

## Key Functions

### Initialization

#### `InitLayers(int cmdGroup)`

Initializes the entire layer system:

1. Reads stored user preference for number of layers (`PREFSECT`, `"layer-button-count"`) and caps it at `NUM_BUTTONS` (default 9).
2. Converts raw color strings to `wDrawColor`.
3. Builds the bitmap icons for each layer button using the character-font bitmaps from `layer_num.inc`.
4. Creates toolbar buttons: one per layer (0..N-1), plus a background toggle button.
5. Creates the drop-down list (`setLayerL`) that lets users pick the active layer.
6. Registers playback procedures for commands like `"SETCURRLAYER"` and `"LAYERS"`.

The function also registers itself as a change notification handler so that the layers dialog can be re-initialized when needed.

#### `InitLayersDialog(void)`

Registers the parameter group for the "Edit Layers" dialog and returns a callback to attach to a toolbar button or menu item. It registers:
- Change notifications (`LayerChange`)
- The parameter group `layerPG` which contains all the layer's editable fields (name, visibility, freezing, color, gauge width/tie spacing, etc.)

---

### Managing Layer Visibility/Freezing

```c
wIndex_t FlipLayer(void)
{
    /* Toggles ON/OFF or FROZEN for the currently selected layer */
    if (!curLayer && !layerBtnOn[curLayer]) {
        return curLayer;  // Can't turn off "Off" (index -1)
    }

    layers[0].on = curLayer ? TRUE : FALSE;
    layers[0].frozen = curLayer == 0;

    layerBtnOn[curLayer] = !layerBtnOn[curLayer];
    wButtonSetBusy(layer_btns[curLayer], layerBtnOn[curLayer]);

    return -1;
}
```

The toolbar buttons toggle the `on` and `frozen` flags of the corresponding layer. The "Off" button (index -1) is a special case that can't be turned off.

---

### Layers Dialog — Opening It

```c
static void DoLayer(void *unused)
{
    if (layerW == NULL) {
        layerW = ParamCreateDialog(&layerPG, MakeWindowTitle(_("Layers")), _("Done"),
                                   LayerOk, ParamCancel_Current, TRUE, NULL, 0,
                                   LayerDlgUpdate);

        GetScaleGauge(layerScaleInx, &layerScaleDescInx, &layerGaugeInx);
        LoadScaleList(scaleL);
        LoadGaugeList(gaugeL, layerScaleDescInx);
    }

    if (settingsCatalog) { CatalogDiscard(settingsCatalog); }
    else { settingsCatalog = InitCatalog(); }
    ScanSettingsDirectory(settingsCatalog, wGetAppWorkDir());

    /* Populate the dialog with current state of "curLayer" */
    UpdateLayerDlg(curLayer);
    layerRedrawMap = FALSE;
    wShow(layerW);
}
```

The dialog is created on first call. It loads a scale list and gauge list, scans the settings directory for user-defined layers (which appear at the top of the list), then populates all controls with the data from `curLayer`.

---

### Layers Dialog Update Handler

```c
static void LayerDlgUpdate(paramGroup_p pg, int inx, void *valueP)
{
    switch (inx) {
        case I_LIST:                 /* List selection changed */
            LayerSelect((wIndex_t)* (long*)valueP);
            break;
        case I_NAME:                /* Name edit box */
            LayerUpdate();
            break;
        case I_MAP:                 /* Map visibility toggle */
            layerRedrawMap = TRUE;
        case I_VIS:                 /* Visible checkbox */
        case I_FRZ:                 /* Frozen checkbox */
        case I_MOD:                 /* Modified checkbox */
        case I_BUT:                /* Color button changed */
        case I_DEF:                /* "Default color" radio */
            LayerUpdate();
            UpdateLayerDlg(layerSelected);  /* Refresh dialog preview */
            break;
        case I_SCALE:              /* Gauge width changed */
            LoadGaugeList((wList_p)layerPLs[I_GAUGE].control, * (int*)valueP);
            wListSetIndex((wList_p)layerPLs[I_GAUGE].control, 0);
            break;
        case I_TIELEN:
        case I_TIEWID:
        case I_TIESPC:
            ValidateTieData(&layerTieData);
            r_tieData.rangechecks = layerTieData.valid ? PDO_NORANGECHECK_LOW | PDO_NORANGECHECK_HIGH : 0;
            break;
        case I_SETTINGS:           /* User-defined settings file */
            if (strcmp((char*)wListGetItemContext(settingsListL, *(long*)valueP), " ") == 0) {
                settingsName[0] = '\0';
            } else {
                strcpy(settingsName, (char*)wListGetItemContext(settingsListL, *(long*)valueP));
            }
            break;
    }
}
```

This is the main glue function: it routes events from various controls to appropriate handlers. Notable points:
- **`I_LIST`**: when a different layer is selected in the list, `LayerSelect()` updates all fields and shows/hides the map preview.
- **`I_SCALE`** (gauge width): re-populates the gauge dropdown with options for the new scale.
- **`I_TIELEN/I_TIEWID/I_TIESPC`**: these are numeric inputs that validate tie spacing data. If the entered values fall outside allowed ranges, a range-check warning is displayed in the dialog.

---

### Layer Selection and Updates

```c
static void LayerSelect(wIndex_t sel)
{
    if (sel < 0 || sel >= layerCount) return;

    UpdateLayerDlg(sel);
    layerSelected = sel;

    /* Map visibility */
    wControlActive((wControl_p)layerPLs[I_MAP].control, TRUE);
    layerRedrawMap = FALSE;
}
```

---

### Saving a Layer (Dialog OK Button)

```c
static void LayerOk(void *unused)
{
    int sel;

    /* Check that the user hasn't changed something they shouldn't have */
    if (!strcmp(settingsName, "")) {  // No custom settings file
        if (layers[layerSelected].modified) {
            ErrorMessage(MSG_LAYER_MODIFIED);
            return;
        }
    } else {
        char *layerSettings = wListGetItemContext(layerPLs[I_SETTINGS].control,
                                                   layerPLs[I_SETTINGS].selected);
        if (!strcmp(layers[layerSelected].name, " ") ||  // No name set
            strcmp(layers[layerSelected].name, settingsName)) {
            ErrorMessage(MSG_LAYER_MODIFIED);
            return;
        }
    }

    /* Write the layer to parameter file */
    CurLayer = (layer_p)MyMalloc(sizeof *CurLayer);
    memcpy(CurLayer, &layers[layerSelected], sizeof *CurLayer);
    CurLayer->index = layerSelected;  // Save index so we can find it again later
    CurLayer->on = layers[layerSelected].on;
    sprintf(message, "\tLAYER %s", GetScaleName(layers[layerSelected].scaleInx));
    fprintf(layerFile, "%.*s\n", (int)strlen(message), message);

    /* Write tie data if applicable */
    if (!strcmp(settingsName, "")) {  // No settings file — write inline
        sprintf(message, "\tT %ld %0.3f %0.3f",
                GetTrkWidth(layers[layerSelected].scaleInx),
                layerTieData.distance, layerTieData.width);
        fprintf(layerFile, "%.*s\n", (int)strlen(message), message);
    } else {
        sprintf(message, "\tS %d \"%s\"", settingsName[0] == '\0' ? 1 : -1, settingsName);
        fprintf(layerFile, "%.*s\n", (int)strlen(message), message);
    }

    /* Save modified flag */
    layers[layerSelected].modified = FALSE;

    wHide(layerW);  // Close dialog
    layoutLayerChanged = TRUE;  // Trigger a redraw of the entire layout
}
```

The "OK" button writes the layer's definition to `layerFile` (a parameter file opened elsewhere in the codebase). If the user modified something and then closed the dialog without saving, an error message is shown. The "modified" flag prevents accidental overwrites.

---

### Updating Dialog from Current Layer State

```c
static void UpdateLayerDlg(layer_p layer)
{
    /* Fill all controls with data from the currently selected layer */
    wListSetItemContext((wList_p)layerPLs[I_LIST].control, layer->index, NULL);
    sprintf(message, "%.*d", (int)(sizeof(message)-1), layer->index+1);
    wListSetValue((wList_p)layerPLs[I_LIST].control, message);

    /* Name */
    strncpy(layers[layer->index].name, " ", sizeof(layers[layer->index].name));  // Placeholder
    sprintf(message, "%.*d", (int)(sizeof(message)-1), layer->index+1);
    strcpy(layers[layer->index].name, message);
    wStringSetValue((wControl_p)layerPLs[I_NAME].control, message);

    /* Visibility / Frozen */
    wCheckSetChecked((wControl_p)layerPLs[I_VIS].control, layer->on);
    wCheckSetChecked((wControl_p)layerPLs[I_FRZ].control, layer->frozen);
    wCheckSetDisabled((wControl_p)layerPLs[I_MOD].control, FALSE);

    /* Color */
    wColorListSelect((wControl_p)layerPLs[I_BUT].control, layer->color, NULL, 0);
    if (layer->useColor) {
        sprintf(message, "%.*d", (int)(sizeof(message)-1), layer->index+1);
        wRadioSetChecked((wControl_p)layerPLs[I_DEF].control, TRUE);
    } else {
        wRadioSetChecked((wControl_p)layerPLs[I_DEF].control, FALSE);
    }

    /* Scale (gauge width) */
    int sel = layerGaugeInx;
    LoadScaleList(scaleL);
    LoadGaugeList(gaugeL, layerScaleDescInx);
    wListSetIndex((wList_p)layerPLs[I_GAUGE].control, sel);

    /* Tie data (length, width, spacing) */
    ValidateTieData(&layerTieData);

    /* Settings file */
    if (strcmp(settingsName, "") == 0) {
        wListSetIndex((wList_p)layerPLs[I_SETTINGS].control, -1);
    } else {
        wListSetIndex((wList_p)layerPLs[I_SETTINGS].control, settingsInx);
    }

    /* Map preview */
    DrawMapPreview(layer->scaleInx, layerGaugeInx, &mapCanvasW, &mapCanvasH);
}
```

Populates every field in the dialog from the `layers[]` array. The map preview canvas is drawn only once per selection change (controlled by `layerRedrawMap`).

---

### Drawing a Layer Map Preview

```c
static void DrawMapPreview(
    SCALEINX_T scaleInx,
    int gaugeInx,
    wWin_p *mapCanvasW,
    wWinPix_t *mapCanvasH)
{
    /* Draws a preview of the track geometry for the selected layer's scale/gauge */
    int i;
    wWinPix_t x = 0, y = 0;

    DrawScaleMarker(scaleInx);     // Draw scale reference line
    DrawGaugeMarker(gaugeInx);      // Draw gauge marker

    /* Draw a sample turnout/switch to show how geometry scales with track gauge */
    for (i=0; i<NUM_LAYERS; i++) {
        if (layers[i].on) {
            x += 4; y -= 32;
            DrawTurnoutPreview(layers[i].scaleInx, layers[i].gaugeInx, x, y);
        }
    }
}
```

The preview shows the scale and gauge markers, plus a turnout drawn at the selected layer's dimensions. This gives immediate visual feedback when changing scale or gauge width.

---

### Loading Layers from Parameter File

```c
static BOOL_T LoadLayers(FILE *f)
{
    char message[STR_SIZE];
    FILE *lf;
    int i;
    layer_p layer;

    for (i = 0; i < NUM_LAYERS; i++) {
        layers[i].name[0] = '\0';
        layers[i].on = TRUE;
        layers[i].frozen = FALSE;
        layers[i].modified = FALSE;
        layers[i].color = wDrawFindColor(layerRawColorTab[i % COUNT(layerRawColorTab)]);
        layers[i].useColor = (i < COUNT(layerRawColorTab));
    }

    if ((lf = fopen(LAYER_FILE, "rb")) == NULL) {
        return TRUE;  // No file — use defaults
    } else {
        for (i = 0; i < NUM_LAYERS && !feof(lf); i++) {
            fgets(message, sizeof(message), lf);
            if (sscanf(message, "LAYER %s", &message[6]) == 1) {
                /* Parse LAYER line and fill in layer fields */
                layers[i].name = MyStrdup(&message[7]);

                // ... parse remaining fields
            } else if (strchr(message, 'T') != NULL) {
                /* Tie data for this layer */
            } else if (strchr(message, 'S') != NULL) {
                /* Settings file reference */
            }
        }
    }

    fclose(lf);
    return TRUE;
}
```

Loads layers from `LAYER_FILE` (a parameter file). The file format is a series of lines:

- `LAYER <name> ...` — layer definition with scale, gauge, color, visibility, etc.
- `T <gaugeWidth> <tieDistance> <tieWidth>` — tie data
- `S -1 "<filename>"` — reference to a settings file (external parameters)

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `InitLayers(cmdGroup)` | Initialize toolbar buttons, layer list drop-down, playback procs | Reads stored layer count; builds bitmaps from font chars |
| `FlipLayer()` | Toggle ON/OFF or FROZEN for current layer | Index -1 is the "Off" button (can't be turned off) |
| `DoLayer(junk)` | Open/layers dialog, populate with current state | Creates dialog on first call; scans settings directory |
| `LayerDlgUpdate(pg, inx, valueP)` | Dialog control event handler | Routes to various sub-handlers based on control index |
| `LayerSelect(sel)` | Switch selection to layer N | Updates all fields and redraws map preview |
| `LayerOk(junk)` | Save layer to parameter file | Writes LAYER/T/S lines; checks for unsaved modifications |
| `UpdateLayerDlg(layer_p)` | Refresh dialog controls from current state | Draws map preview canvas |
| `DrawMapPreview(scaleInx, gaugeInx, ...)` | Draw scale/gauge markers + turnout sample | Visual feedback for scale/gauge changes |
| `LoadLayers(FILE *f)` | Read layers from parameter file | Parses LAYER/T/S lines; uses defaults if no file exists |

---

## Domain & Design Notes

- **Layer indices**: Index -1 is a special "Off" button that hides all tracks. Valid layer indices are 0..N-1 (corresponding to toolbar buttons 0..N-1).
- **Modified flag**: Prevents accidental overwrites — if a user changes a field in the dialog and then clicks "Cancel", the change is discarded. If they click "OK" after modifying, the layer is written to the parameter file and `modified` becomes FALSE.
- **Settings files**: Users can store custom layer configurations (e.g., for different scales) by referencing an external settings file. This allows reusable layer setups without re-entering parameters each time.
- **Map preview canvas**: The layers dialog contains a small embedded canvas (`mapCanvasW`, `mapCanvasH`) that draws a turnout sample at the selected scale/gauge combination. This is useful for comparing how different scales look side by side.
