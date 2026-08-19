# XTrkCad `app/cornu/` Documentation Index

## Overview

The `app/cornu/` directory contains a port of **Raph Levien's polynomial spiral spline** library, used by XTrkCad to generate smooth easement curves between track segments. Unlike traditional clothoid (Cornu) functions which use Fresnel integrals for linearly-varying curvature, the spiro library uses piecewise polynomial spirals with $C^2$ continuity.

---

## Documented Files

| File | Summary |
|------|---------|
| [`spiro.c`](./spiro.c.md) | Polynomial spiral splines: band-diagonal matrix solver, Newton iteration for coefficient determination, subdivision to Bezier paths |
| [`spiroentrypoints.c`](./spiroentrypoints.c.md) | XTrkCad integration layer: initializes spiro objects, generates easement curves from track database segments, handles spline continuity constraints |

---

## Key Concepts

### Polynomial Spiral Spirals

A polynomial spiral is defined by a curvature function that is itself a polynomial of degree 4 (making the position a quintic polynomial in arc-length parameter):

$$\kappa(s) = c_0 + c_1 s + c_2 s^2 + c_3 s^3 + c_4 s^4$$

This yields:
- $C^2$ continuity at segment joins (continuous curvature, not just continuous tangent)
- Exact integration via precomputed coefficients — no numerical quadrature error
- Very fast evaluation (fixed number of FLOPs per point)

### Why Not Just Clothoids?

Clothoid spirals have $\kappa(s)$ linear in $s$, giving a clothoid segment between each pair of waypoints. The limitation is that joining two clothoids at an arbitrary angle generally produces a **discontinuous curvature** (only $C^1$ continuity). Polynomial spirals solve this by using higher-order polynomials whose coefficients are solved as a banded linear system to satisfy both position and tangent constraints simultaneously.

---

## Usage in XTrkCad

The spiro library is used to generate easement curves that:
- Connect a straight track segment to a circular arc (or vice versa)
- Provide smooth transitions between two different radii arcs
- Allow arbitrarily many intermediate control points for complex track geometry

See [`ccornu.c`](../bin/ccornu.c.md) for the XTrkCad-specific wrapper that integrates spiro with the track database.
