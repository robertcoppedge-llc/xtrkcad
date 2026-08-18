# elev.c — Dynamic Elevation Computation for Track Layouts

## Overview

`elev.c` implements the **dynamic elevation computation** system in XTrkCAD. This is a sophisticated algorithm that assigns elevations to tracks based on their connectivity to "elevation anchors" (DefElev points), propagating elevation values through turnouts and short-circuit paths.

The core idea: instead of requiring every track endpoint to have an explicit elevation marker, the system computes elevations by treating the layout as a network graph where:

- **DefElev** = defined elevation endpoint (explicitly set by the user)
- **Fork** = turnout/junction connecting multiple elevation zones
- **Island** = group of tracks connected only to each other, not to any DefElev/Fork

The algorithm uses **weighted averaging based on path distance**, so that a track closer to one DefElev gets an elevation closer to that anchor, while tracks equidistant from two DefElevs get the midpoint.

---

## Core Data Structures

### Elevation List Entry (`elist_t`)

```c
typedef struct {
    track_p trk;   // The track being processed
    EPINX_T ep;    // Endpoint index on that track
    DIST_T len;    // Distance from the endpoint (used for weighted averaging)
} elist_t;
static dynArr_t elist_da;  /* Per-call working buffer */
```

Used in Step 5 (Island computation) to collect all tracks reachable from a starting point.

---

### Fork Definition (`fork_t`)

```c
typedef struct {
    track_p trk;   // The turnout (junction) track
    EPINX_T ep;    // Endpoint on the turnout itself
    EPINX_T ep2;   // Other endpoint of the turnout (the one leading to a DefElev)
    DIST_T dist;   // Distance from this fork to its associated DefElev
    DIST_T elev;   // The elevation value for this fork
} fork_t;
static dynArr_t fork_da;  /* Per-call working buffer */
```

Each entry represents a turnout that lies on a shortest path between two DefElev points. The `elev` field holds the computed weighted-elevation value for that junction.

---

### Pivot Definition (`pivot_t`)

```c
typedef struct {
    track_p trk;   // A track on the boundary of an Island (connected to elevation zone)
    coOrd pos;     // Center point of the track (for distance computation)
    DIST_T elev;   // Elevation value of this pivot (from a connected DefElev/Fork)
} pivot_t;
static dynArr_t pivot_da;  /* Per-call working buffer */
```

Used in Step 5 to identify which tracks form the boundary of an "Island" and what elevation they should influence.

---

### Elevation Distance (`elevdist_t`)

```c
typedef struct {
    DIST_T elev;   // Elevation of a connected DefElev/Fork
    DIST_T dist;   // Total distance from current track to that anchor
} elevdist_t;
static dynArr_t elevdist_da;  /* Per-call working buffer */
```

Used during the weighted averaging step (Step 4). Each entry records one path from a junction or island to an elevation anchor.

---

### DefElev Definition (`defelev_t`)

```c
typedef struct {
    track_p trk;   // Track where the endpoint is defined
    EPINX_T ep;    // Endpoint index
    DIST_T elev;   // Elevation value at that endpoint
} defelev_t;
static dynArr_t defelev_da;  /* Per-call working buffer */
```

Collected in Step 1 — lists all endpoints across the layout that have explicit elevation values.

---

## Algorithm Steps

### Step 1: Find DefElevs (`FindDefElev`)

Scans all tracks and collects every endpoint marked as `EndPtIsDefinedElev()`. Each entry stores the track, endpoint index, and elevation value.

```c
static void FindDefElev( void )
{
    /* Iterate all tracks; for each, scan endpoints:
       if EndPtIsDefinedElev(trk,ep) → add to defelev_da */
}
```

**Output:** `defelev_da` — a list of all elevation anchors.

---

### Step 2: Find Forks on Shortest Paths (`FindForks`)

For each DefElev, compute the **shortest path** through the track network to every other endpoint. Any turnout (turnout with ≥3 connections) encountered along that path is recorded as a "Fork" along with its distance from the DefElev and the DefElev's elevation.

The shortest-path search uses a custom callback function:

```c
static int FillElevShortestPathFunc(
        SPTF_CMD cmd,           /* Command type: MATCH, ADD_TRK, IGNORE, etc. */
        track_p trk,           /* Current track being examined */
        EPINX_T ep,            /* Endpoint we're at on that track */
        DIST_T dist,           /* Accumulated distance from start point */
        void * data )          /* Pointer to defelev_t (the source anchor) */
{
    switch (cmd) {
        case SPTC_MATCH:
            /* Check if current endpoint matches the target DefElev */
            break;
        case SPTC_ADD_TRK:
            /* Current track is a Fork: record it with distance and elev */
            DYNARR_APPEND(fork_t, fork_da, 10);
            ...;
            break;
        case SPTC_IGNNXTTRK:
            /* This endpoint is ignored (e.g., marked for elevation ignore) */
            return 1;  /* skip this endpoint */
        case SPTC_TERMINATE:
            /* Reached the target DefElev — stop searching from here */
            break;
    }
}
```

**Outputs:** `fork_da` (list of forks), plus updates to `TB_ELEVPATH` bits on tracks that lie on a shortest path.

---

### Step 3: Compute Fork Elevations (`ComputeForkElev`)

For each fork, gather all connected DefElevs and compute a **weighted average elevation**. The weight for each connection is inversely proportional to the distance from the fork to its anchor:

```
elev = Σ (dist_total / dist_to_anchor_i) * elev_i   ÷   Σ (dist_total / dist_to_anchor_i)
```

Where `dist_total` is the sum of all distances from this fork to all connected DefElevs. If a fork has only **one** connection to an elevation anchor, it simply inherits that anchor's elevation (`singlePath = TRUE`).

If there are ≥3 connections to different DefElevs (not just two endpoints on the same turnout), the track is considered a true junction and gets a computed weighted elevation.

```c
static DIST_T ComputeWeightedElev(DIST_T totalDist)
{
    for each elevdist_t *w in elevdist_da:
        if w->dist < 0.001:  /* very close to anchor */
            return w->elev;   /* no averaging needed */

    d2 = Σ (totalDist / w->dist);           /* sum of inverse distances */
    e = Σ ((totalDist / w->dist) * w->elev) / d2;   /* weighted average */
    return e;
}
```

**Outputs:** `fork_da` is modified in-place to hold the computed elevation for each fork.

---

### Step 4: Propagate Fork Elevations (`PropogateForkElevs`, `PropogateDefElevs`)

For every track that lies on a shortest path between forks/DefElevs (marked with `TB_ELEVPATH` bit), propagate the elevation along each branch.

```c
static void PropogateForkElev(track_p trk, EPINX_T ep1, DIST_T d1, DIST_T e)
{
    /* Walk along the path from the fork to the next junction/DefElev:
       - If the next endpoint is a DefElev → stop (already defined)
       - Otherwise, compute the midpoint elevation at the halfway point:
             elev_half = e + d1 * (elev_anchor - e) / d2
       - Mark intermediate tracks as ELEV_BRANCH mode */

    /* Build list of all branches leading away from this fork */
    DYNARR_RESET(elist_t, elist_da);
    while (trk != NULL && trk not already marked) {
        if (next endpoint is DefElev):
            d1 += half-distance;
            goto next_step;

        elistAppend(trk, ep1, d1);   /* record this branch */
        trk = next_track_at_this_endpoint;
        d1 += half-distance;
    }

    /* For each recorded branch, mark tracks as ELEV_BRANCH */
    for each entry in elist_da:
        SetTrkOnElevPath(entry.trk, ELEV_BRANCH, e + e1 * d2);
}
```

---

### Step 5: Find Islands (`SurveyIsland`, `ComputeIslandElev`)

After Steps 1–4 have marked all tracks connected to DefElevs/Forks with `TB_ELEVPATH`, any remaining unmarked tracks form one or more **islands**. An island is a group of tracks that are mutually reachable but have no connection (directly or via turnouts) to any elevation anchor.

For each island:
1. Collect all its boundary tracks into `pivot_da` — these are tracks adjacent to an already-elevated zone or a DefElev endpoint.
2. Compute the **weighted average elevation** for the entire island using distances from the island's center (or midpoint of bounding box) to each pivot point:

```c
static void ComputeIslandElev(track_p trk0)  /* starting track in island */
{
    SurveyIsland(trk0, TRUE);   /* find all tracks in this island + pivots */

    for each track in elist_da (the island):
        if no pivots found:
            elev = 0;           /* island is completely isolated → no elevation */
        else if only one pivot:
            elev = pivot[0].elev;   /* inherit from the single connected zone */
        else:
            totalDist = Σ distance to each pivot;
            elev = Σ (totalDist / dist_i) * pivots[i].elev ÷ Σ(totalDist/dist_i);

    for each track in island:
        SetTrkOnElevPath(trk, ELEV_ISLAND || ELEV_ALONE, elev);
}
```

**Mode bits:**

| Mode | Meaning |
|------|---------|
| `ELEV_DEF` | Explicitly defined elevation (user-set) |
| `ELEV_GRADE` | On a grade slope between two points |
| `ELEV_COMP` | Computed elevation on a curve/transition |
| `ELEV_IGNORE` | Ignored for elevation purposes |
| `ELEV_STATION` | Elevation tied to a station marker |
| `ELEV_FORK` | Junction/fork track (Step 3) |
| `ELEV_BRANCH` | Track on a branch from a fork (Step 4) |
| `ELEV_ISLAND` | Island connected to one or more elevation anchors |
| `ELEV_ALONE` | Isolated island with no connection → elevation = 0 |

---

## Key Functions Summary

| Function | Purpose |
|----------|---------|
| `ComputeElev(trk, ep)` | Compute/refresh elevation for a single endpoint (calls the full pipeline if needed) |
| `FindDefElev()` | Step 1: collect all DefElev anchors |
| `FindForks()` | Step 2: find forks on shortest paths between DefElevs |
| `ComputeForkElev()` | Step 3: compute weighted elevations for each fork |
| `PropogateForkElevs()` | Step 4a: propagate from forks along branches |
| `PropogateDefElevs()` | Step 4b: propagate directly from DefElev endpoints |
| `FindIslandElevs()` | Step 5: identify and compute elevations for isolated islands |
| `ComputeWeightedElev(totalDist)` | Utility: computes weighted average elevation given distances to anchors |
| `SetTrkOnElevPath(trk, mode, elev)` | Sets the elevation mode/elevation value on a track (internal helper) |
| `GetTrkOnElevPath(trk, *elev)` | Checks whether a track has an elevation value and returns it |

---

## Summary Table

| Step | Phase | Key Action | Output Buffer |
|------|-------|------------|---------------|
| 1 | Find DefElevs | Scan all endpoints for `DefElev` markers | `defelev_da` |
| 2 | Find Forks | Shortest-path search from each DefElev; record junction turnouts | `fork_da`, `TB_ELEVPATH` bits |
| 3 | Compute Fork Elevations | Weighted average across connected anchors | updates `fork.da.elev` |
| 4a | Propagate Forks | Walk branches away from forks, marking tracks as `ELEV_BRANCH` | updates track elevation modes |
| 4b | Propagate DefElevs | Direct propagation from marked endpoints | — |
| 5 | Find Islands | Survey remaining unmarked tracks; compute weighted average via pivots | `pivot_da`, `elist_da`; sets mode to `ELEV_ISLAND` or `ELEV_ALONE` |

---

## Design Notes

- **Lazy recomputation**: The system caches elevation values per endpoint (`GetTrkEndElevCachedHeight`) and only re-runs the expensive pipeline when something changes (track added, DefElev moved, etc.).
- **Isolated tracks get no elevation**: If a group of tracks forms an island with no connection to any elevation anchor, its elevation is set to 0. This prevents spurious elevation values from appearing for completely disconnected layout islands.
- **Weighted averaging** (inverse-distance weighting) means that a track closer to one DefElev pulls more strongly toward that value than a distant one — mimicking how real-world railroads often follow terrain constraints.
