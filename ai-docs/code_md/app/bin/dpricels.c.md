# dpricels.c — Price List Dialog

## Overview

`dpricels.c` implements the **Price List dialog**, a utility for editing turnout (turnout/switch) price entries. The dialog allows users to:

- Select a scale from a list of defined scales
- Edit the unit cost for that turnout type
- Specify additional flex track length and its associated cost

Prices are persisted using `wPrefSetFloat()` under the key `"price list <scale_name>"`, allowing per-scale price lists.

---

## Core Data Structures

### Price List Parameter Group (`priceListPG`)

```c
static wWin_p priceListW;          // Dialog window handle
static turnoutInfo_t * priceListCurrent;  // Currently selected turnoutInfo entry

/* Global variables for editing */
DIST_T priceListCostV;             // Current unit cost being edited
char priceListEntryV[STR_SIZE];    // Formatted display string (manuf|descr|partno)
DIST_T priceListFlexLengthV;       // Flex track length (feet or meters depending on units)
DIST_T priceListFlexCostV;         // Cost per unit of flex track

static paramFloatRange_t priceListCostData = { 0.0, 9999.99, 80 };
static wWinPix_t priceListColumnWidths[] = { -60, 200 };
static const char * priceListColumnTitles[] = { N_("Price"), N_("Item") };
static paramListData_t priceListListData = { 10, 400, 2, priceListColumnWidths, priceListColumnTitles };
```

### Parameter List (`priceListPLs`)

| Index | Name | Control Type | Field | Purpose |
|-------|------|--------------|-------|---------|
| 0 | `I_PRICELSCOST` | FloatEdit | `priceListCostV` | Unit cost for the selected turnout |
| 1 | `I_PRICELSENTRY` | ReadonlyString | `priceListEntryV` | Display string showing turnout type (manuf\|descr\|partno) |
| 2 | `I_PRICELSLIST` | DropList | — | Scale selector; each item is a pointer to a `turnoutInfo_t` entry |
| 3 | `I_PRICELSFLEXLEN` | FloatEdit (label only) | `priceListFlexLengthV` | Label for flex track length field |
| 6 | `I_PRICELSFLEXCOST` | FloatEdit | `priceListFlexCostV` | Cost per unit of flex track |

Note: Index 4 and 5 are intentionally skipped in the list, likely reserving room for future entries. The flex length control at index 3 has no associated field pointer — it's just a label displayed alongside the actual input box (which would be at a higher index if present).

---

## Key Functions

### `PriceListUpdate()`

```c
static void PriceListUpdate()
{
    DIST_T oldPrice;
    ParamLoadData( &priceListPG );  // Read current values from widgets into global vars

    if (priceListCurrent == NULL) {
        return;   // Nothing selected — nothing to update
    }

    /* Build a display string like "Montrachet #106 33/32 90° Switch" */
    FormatCompoundTitle( LABEL_MANUF|LABEL_DESCR|LABEL_PARTNO,
                         priceListCurrent->title );

    wPrefGetFloat( "price list", message, &oldPrice, 0.0 );
    if (oldPrice == priceListCostV) {
        return;   // Price hasn't changed — nothing to do
    }

    /* Persist the new price under a per-scale key */
    wPrefSetFloat( "price list", message, priceListCostV );
    FormatCompoundTitle( listLabels|LABEL_COST, priceListCurrent->title );
    if (message[0] != '\0') {
        wListSetValues( priceListSelL, wListGetIndex(priceListSelL), message, NULL, priceListCurrent );
    }
}
```

**Key points:**

- `FormatCompoundTitle()` with `LABEL_MANUF|LABEL_DESCR|LABEL_PARTNO` reconstructs the turnout's title from its constituent fields (manufacturer description, item type, part number). For example: `"Montrachet #106 33/32 90° Switch"`.
- The price is saved to a persistent preferences file using `wPrefSetFloat("price list", ...)`. The key includes the scale name, so different scales can have their own independent price lists.
- If the displayed string changes (e.g., because the user switched to a different turnout type), it's re-inserted into the drop-down list at its current position.

---

### `PriceListOk(void *action)`

```c
static void PriceListOk( void *action )
{
    PriceListUpdate();  // Save cost and update display string in drop-down

    sprintf( message, "price list %s", curScaleName );
    wPrefSetFloat( message, "flex length", priceListFlexLengthV );
    wPrefSetFloat( message, "flex cost", priceListFlexCostV );

    wHide( priceListW );  // Close dialog
}
```

On OK: the current cost is persisted (via `PriceListUpdate()`), and both flex track length and flex cost are saved under a per-scale key (`"price list <scale_name>"`). The dialog then closes.

---

### `PriceListSel(turnoutInfo_t *to)`

```c
static void PriceListSel( turnoutInfo_t * to )
{
    FLOAT_T price;
    PriceListUpdate();
    priceListCurrent = to;
    if (priceListCurrent == NULL) { return; }

    FormatCompoundTitle( LABEL_MANUF|LABEL_DESCR|LABEL_PARTNO,
                         priceListCurrent->title );
    wPrefGetFloat( "price list", message, &price, 0.00 );
    priceListCostV = price;
    strcpy( priceListEntryV, message );

    ParamLoadControl( &priceListPG, I_PRICELSCOST );   // Set cost field to saved value
    ParamLoadControl( &priceListPG, I_PRICELSENTRY );  // Show formatted title string
}
```

When a user selects an entry from the drop-down:

1. The dialog is updated with that turnout's details (formatted title shown in the readonly label).
2. The saved price is read back from preferences and placed into the cost edit box.
3. `ParamLoadControl()` restores the widget's value — this also re-enables its internal validation logic if needed.

---

### `PriceListChange(long changes)`

```c
static void PriceListChange( long changes )
{
    turnoutInfo_t * to1, * to2;
    if ((changes & (CHANGE_SCALE|CHANGE_PARAMS)) == 0 ||
        priceListW == NULL || !wWinIsVisible( priceListW ) ) {
        return;
    }

    /* Clear the list */
    wListClear( priceListSelL );

    to1 = TurnoutAdd( listLabels|LABEL_COST, GetLayoutCurScale(), priceListSelL, NULL, -1 );
    to2 = StructAdd( listLabels|LABEL_COST, GetLayoutCurScale(), priceListSelL, NULL );
    if (to1 == NULL) {
        to1 = to2;
    }

    priceListCurrent = NULL;  // Reset selection until user picks something

    /* If a scale was added or removed, refresh the list with its entries */
    if ((changes & CHANGE_SCALE) == 0) {
        return;
    }

    sprintf( message, "price list %s", curScaleName );
    wPrefGetFloat( message, "flex length", &priceListFlexLengthV, 0.0 );
    wPrefGetFloat( message, "flex cost", &priceListFlexCostV, 0.0 );
    ParamLoadControls( &priceListPG );
}
```

This is a **change-notification handler** that runs whenever a scale or parameter change occurs (e.g., the user switches scales in the main toolbar). It:

1. Clears the drop-down list and rebuilds it by calling `TurnoutAdd()` and `StructAdd()`. These functions iterate over all turnouts of the current scale and add them to the price list drop-down, storing a pointer back into each item so that selection can be resolved later.
2. If a new scale was created (`CHANGE_SCALE`), it re-reads flex track parameters from preferences under `"price list <scale_name>"`.

Note: The dialog is only refreshed if `!wWinIsVisible(priceListW)` — i.e., the user hasn't already opened the dialog, in which case they can see stale data until they click OK. This is a subtle but important detail: **the dialog contents may be stale if it's open during a scale change**, and the next save will overwrite whatever was edited.

---

### `PriceListDlgUpdate(paramGroup_p pg, int inx, void *valueP)`

```c
static void PriceListDlgUpdate( paramGroup_p pg, int inx, void * valueP )
{
    turnoutInfo_t * to;
    switch( inx ) {
        case I_PRICELSCOST:
            PriceListUpdate();  // Save cost when the edit box is modified
            break;
        case I_PRICELSLIST:
            to = (turnoutInfo_t*)wListGetItemContext( (wList_p)pg->paramPtr[inx].control,
                            *(long*)valueP );
            PriceListSel( to );  // Populate cost field and display string for the selected turnout
            break;
    }
}
```

The dialog update handler handles two events:

- **Cost edit box changed** — calls `PriceListUpdate()` which writes the new price back to preferences.
- **Scale drop-down selection changed** — retrieves the corresponding `turnoutInfo_t` pointer from the list item's context and loads that turnout's details into the dialog (title string, cost field).

---

### Opening the Dialog

```c
static void DoPriceList( void *junk )
{
    if (priceListW == NULL) {
        priceListW = ParamCreateDialog( &priceListPG, MakeWindowTitle(_("Price List")),
                                        _("Done"), PriceListOk, ParamCancel_Null, TRUE, NULL, F_RESIZE,
                                        PriceListDlgUpdate );
    }
    wShow( priceListW );
    PriceListChange( CHANGE_SCALE|CHANGE_PARAMS );  // Populate list with current scale's turnouts
}
```

The dialog is lazily created on first call. On startup (`PriceListInit`), it registers itself as a parameter group and returns a callback that attaches the window to a toolbar button or menu item. The `CHANGE_SCALE|CHANGE_PARAMS` event triggers a rebuild of the drop-down list with entries for all turnouts defined in the current scale.

---

## Summary Table

| Function | Purpose | Key Notes |
|----------|---------|-----------|
| `PriceListInit()` | Register parameter group, return callback | — |
| `DoPriceList(junk)` | Open dialog; refresh list with current scale's turnouts | Lazily creates window; rebuilds drop-down on every call if not visible |
| `PriceListDlgUpdate(pg, inx, valueP)` | Dialog control event handler | Routes by index: cost edit saves price, list selection loads turnout details |
| `PriceListSel(turnoutInfo_t *to)` | Populate dialog fields for a selected turnout | Reads saved price from prefs; formats title string |
| `PriceListUpdate()` | Save current cost to prefs and update display label | Calls `FormatCompoundTitle()` with the three component flags |
| `PriceListOk(void *)` | Close dialog after saving flex length/cost under per-scale key | — |
| `PriceListChange(long changes)` | Refresh list when scale/params change | Clears list, rebuilds via `TurnoutAdd()`, reads flex params from prefs if scale changed |

---

## Domain & Design Notes

- **Per-scale price lists:** Prices are stored under keys like `"price list Standard"` or `"price list Prototype"`. This means a user can maintain separate pricing for different scales without conflict. The key is built with `sprintf("price list %s", curScaleName)`.
  
- **Flex track parameters:** Flex track length and its unit cost are also stored per-scale, allowing flexibility in modeling scenarios where flex track isn't free or costs differently than standard piece-to-piece sections.

- **`FormatCompoundTitle()` flags:** The `LABEL_MANUF|LABEL_DESCR|LABEL_PARTNO` combination reconstructs the full turnout name from three separate fields (manufacturer description, item type code, and part number). This is used both for display in the dialog's label field and for updating the drop-down list with a meaningful string instead of just an index.

- **Change-notification coupling:** Because `PriceListChange()` runs on scale/parameter changes, opening the dialog while a scale change is happening can leave it in an intermediate state. The next save will overwrite whatever was edited — this is acceptable because price lists are typically static data that users edit infrequently.
