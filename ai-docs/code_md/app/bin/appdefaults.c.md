# appdefaults.c — Application Defaults & Region-Aware Configuration

## Overview

`appdefaults.c` implements a **configuration defaults system** for XTrkCAD. It provides:

- A sorted lookup table of default values keyed by section.name pairs.
- Binary search over that table to find defaults when a configuration value is missing.
- Region-aware defaults (US/Canada = imperial, GB = metric) via function-pointer callbacks.
- A first-run detection mechanism using a special `"misc.firstrun"` boolean flag in the preferences file.

The design allows defaults to be either:
- **Static constants** (e.g., `1` for "pre-select command mode")
- **Region-dependent functions** (e.g., return `HO` scale in US, `OO` in UK)
- **Path generators** (e.g., construct a full path like `"libdir/paramsubdir/nmra-ho.xtp"`)

---

## Core Data Structure: `appDefault`

```c
struct appDefault {
    const char *defaultKey;              /* "section.name" lookup key, sorted alphabetically */
    bool wasUsed;                         /* flag to avoid re-fetching same default once used per run */
    enum defaultTypes valueType;          /* INTEGERCONSTANT | FLOATCONSTANT | STRINGCONSTANT |
                                             INTEGERFUNCTION | FLOATFUNCTION | STRINGFUNCTION */
    union {
        int  intValue;                    /* static integer default (e.g., 1 = "pre-select") */
        double floatValue;                /* static float default */
        const char *stringValue;          /* pointer to a literal string constant */
        int (*intFunction)(struct appDefault *, const void *);  /* function returning an int */
        double (*floatFunction)(struct appDefault *, const void *);  /* function returning a float */
        const char *(*stringFunction)(struct appDefault *, const void *);  /* function returning a string */
    } defaultValue;
    const void *additionalData;           /* extra data passed to function callbacks */
};
```

The `union` allows each entry to carry either a **static value** or a **function pointer**. The caller must set the appropriate field and never read an unused union member.

---

## Default Entry Table (`xtcDefaults[]`)

Each row is sorted by its full key string (section + "." + name), enabling binary search. Example entries:

| Key | Type | Value / Function |
|-----|------|------------------|
| `DialogItem.cmdopt-preselect` | INTEGERCONSTANT | `1` (pre-select command mode) |
| `DialogItem.pref-dstfmt` | INTEGERFUNCTION | calls `GetLocalDistanceFormat()` → returns `8` for metric, `4` for imperial |
| `draw.roomsizeX` | FLOATFUNCTION | calls `GetLocalRoomSize()` → returns `125/2.54` (m) or `48` (inches) depending on region |
| `misc.scale` | STRINGFUNCTION | calls `GetLocalPopularScale()` → returns `"HO"` in US, `"OO"` in GB |

The table is stored as a static global array; no dynamic allocation is needed. The header defines:

```c
#define DEFAULTCOUNT COUNT(xtcDefaults)  /* number of entries */
```

---

## Binary Search Lookup (`FindDefault`)

```c
struct appDefault *
FindDefault(struct appDefault *defaultValues, const char *section,
            const char *name)
{
    char *searchString = malloc(strlen(section) + strlen(name) + 2);
    int res;
    sprintf(searchString, "%s.%s", section, name);

    res = binarySearch(defaultValues, 0, DEFAULTCOUNT-1, searchString);
    free(searchString);

    if (res != -1 && defaultValues[res].wasUsed == FALSE) {
        defaultValues[res].wasUsed = TRUE;   /* mark as used so it's not fetched again */
        return (defaultValues + res);
    } else {
        return NULL;                          /* no matching entry or already consumed */
    }
}
```

The `wasUsed` flag ensures that once a default has been applied (and its value stored into the preferences file), it is never fetched again — avoiding redundant function calls and memory allocations on subsequent accesses.

---

## Region Detection (`InitializeRegionCode`)

On first run, the program detects the user's locale region:

```c
static void InitializeRegionCode(void)
{
    strcpy(regionCode, "US");  /* default fallback */

#ifdef WINDOWS
    {
        LCID lcid = GetThreadLocale();
        char iso3166[10];
        GetLocaleInfo(lcid, LOCALE_SISO3166CTRYNAME, iso3166, sizeof(iso3166));
        strncpy(regionCode, iso3166, 2);   /* e.g., "US" or "GB" */
    }
#else
    {
        char *pLang = getenv("LANG");

        if (pLang) {
            char *ptr = strpbrk(pLang, "_-");
            if (ptr) {
                strncpy(regionCode, ptr + 1, 2);   /* extract "US", "GB", etc. */
            }
        }
    }
#endif

    regionCode[2] = '\0';
}
```

This runs **once**, after the preferences file is confirmed to not exist (first run). On subsequent runs, `bFirstRun` remains false and the function is skipped.

---

## Region-Aware Default Functions

### Metric vs Imperial

```c
static bool UseMetric(void)
{
    return (strcmp(regionCode, "US") != 0 &&
            strcmp(regionCode, "CA") != 0);
}
```

This returns `true` for all regions except `"US"` and `"CA"`, so metric units are used everywhere else.

---

### Room Size Defaults (`GetLocalRoomSize`)

The default layout room size depends on the region:

| Region | X dimension (pixels) | Y dimension (pixels) |
|--------|----------------------|----------------------|
| US/CA  | `96` inches          | `48` inches          |
| Metric | `200 cm / 2.54 = 78.74 px` | `125 cm / 2.54 = 49.21 px` |

```c
static double GetLocalRoomSize(struct appDefault *ptrDefault, const void *data)
{
    if (!strcmp(ptrDefault->defaultKey, "draw.roomsizeX")) {
        return UseMetric() ? 200.0 / 2.54 : 96;
    }

    if (!strcmp(ptrDefault->defaultKey, "draw.roomsizeY")) {
        return UseMetric() ? 125.0 / 2.54 : 48;
    }

    return 0.0;   /* should never reach here */
}
```

---

### Popular Scale (`GetLocalPopularScale`)

The most common model scale by region:

```c
static const char *GetLocalPopularScale(struct appDefault *ptrDefault, const void *data)
{
    return (strcmp(regionCode, "GB") ? "HO" : "OO");
}
```

In the UK (`regionCode == "GB"`), `OO` (1:76.2) is the standard; elsewhere HO (1:87) is returned.

---

### Prototype Map Defaults

Prototype maps are region-specific: US users get `"North American Prototypes"`, while GB users get `"British stock"`. The function simply returns a string literal based on `regionCode`.

---

## Wrapper Functions (`wPrefGet*Ext`)

The public API is split into two layers:

```c
/* Basic access — no defaults consulted */
bool wPrefGetIntegerBasic(const char *section, const char *name, long *result, long defaultValue);
double wPrefGetFloatBasic(...);
char   *wPrefGetStringBasic(...);

/* Extended access — consults the default table first */
bool wPrefGetIntegerExt(const char *section, const char *name, long *result, long defaultValue);
double wPrefGetFloatExt(...);
char   *wPrefGetStringExt(...);
```

On first run (`bFirstRun == TRUE`), `InitAppDefaults()` sets a function pointer that points to the basic getter. On subsequent runs (after `"misc.firstrun"` was written as `FALSE`), it switches to the extended version, which looks up defaults in `xtcDefaults[]`.

---

## Initialization (`InitAppDefaults`)

```c
void InitAppDefaults(void)
{
    wPrefGetIntegerBasic("misc", "firstrun", &bFirstRun, TRUE);

    if (bFirstRun) {
        wPrefSetInteger("misc", "firstrun", FALSE);   /* mark first run as done */
        InitializeRegionCode();                        /* detect locale region */
    } else {
        GetIntegerPref = wPrefGetIntegerBasic;
        GetFloatPref  = wPrefGetFloatBasic;
        GetStringPref = wPrefGetStringBasic;
    }
}
```

On the **first run**, `firstrun` is written as `FALSE` and region detection happens. On all later runs, the basic getters are used directly because the user's preferences file already exists and can override defaults naturally.

---

## Summary Table

| Component | Purpose | Key Detail |
|-----------|---------|------------|
| `appDefault` struct | Defines a single default entry with key, type, and value/func pointer | Uses a union to store either static data or a function pointer |
| `xtcDefaults[]` array | Global lookup table of all defaults (sorted by key) | Each row has a `.wasUsed` flag to avoid re-fetching already-applied defaults |
| `FindDefault()` | Binary-searches the table for a given `"section.name"` key | Returns pointer or NULL; marks entry as used if found |
| `InitializeRegionCode()` | Detects locale on first run (Windows API vs `LANG` env var) | Sets global `regionCode[3]` to something like `"US"`, `"GB"`, etc. |
| Region-aware functions (`GetLocal*`) | Return values that differ by region (metric/imperial, HO/OO scale, prototype map) | Called through function-pointer callbacks in the default table |
| `wPrefGet*Ext()` wrappers | Public API that first checks defaults, then falls back to user preferences | On first run, `"misc.firstrun"` is written and then basic getters are used; on later runs, extended getters consult the default table |

---

## Design Notes

- **Binary search** over a static array avoids scanning all entries linearly — O(log n) lookup instead of O(n).
- The `wasUsed` flag prevents redundant function calls (e.g., recomputing region-dependent values that have already been stored in the preferences file).
- Region detection runs only once on first start; subsequent starts skip it because `"misc.firstrun"` is now `FALSE`.
- The design cleanly separates **first-run initialization** from **subsequent runtime**, avoiding unnecessary locale detection on every launch.
