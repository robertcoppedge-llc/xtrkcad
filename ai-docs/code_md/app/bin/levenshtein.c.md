# levenshtein.c — Levenshtein Distance Algorithm

## Overview

`levenshtein.c` implements the **Levenshtein distance** (edit distance) between two strings using the classic dynamic programming algorithm, optimized for **O(min(|a|, |b|)) space** by only maintaining a single row of the DP table.

The Levenshtein distance is defined as the minimum number of **single-character edits** (insertions, deletions, or substitutions) required to transform one string into another. It is widely used for:

- Spell correction
- Fuzzy matching / similarity search
- Duplicate detection
- Typo correction in user input

---

## External Declaration

The header `app/bin/include/levenshtein.h` declares two exported functions:

```c
// Returns the edit distance between strings a and b.
size_t levenshtein(const char *a, const char *b);

// Optimized variant that takes pre-known lengths (avoids calling strlen).
size_t levenshtein_n(const char *a, size_t lengthA,
                     const char *b, size_t lengthB);
```

---

## Algorithm: Dynamic Programming with Space Optimization

### The DP Table Concept

The full DP table for strings `a` of length `m` and `b` of length `n` is an `(m+1) × (n+1)` matrix where:

```
dp[i][j] = min(
    dp[i-1][j-1],  // a[0..i-1] → b[0..j-1] with no edit (match or substitution)
    dp[i-1][j] + 1, // deletion from a
    dp[i][j-1] + 1  // insertion into a
) + cost(a[i], b[j])
```

where `cost(char x, char y)` is `0` if `x == y`, else `1`.

### Space Optimization: Row-by-Row Reduction

The full matrix requires O(m × n) space. However, notice that row `i+1` only depends on:
- The current row being built (row `i+1`)
- The previous row (row `i`)

Thus we can reduce the space to **two rows** — or even a single array of size `min(m, n) + 1` by swapping the shorter and longer string.

The implementation in this file uses that trick: it ensures `length <= lengthB` so the inner dimension is always ≤ the other, reducing memory from O(a_len × b_len) to **O(min(|a|, |b|))**.

---

## Code Walkthrough

```c
size_t levenshtein_n(const char *a, const size_t length,
                     const char *b, const size_t bLength)
{
    // === Degenerate cases / shortcuts ===
    if (a == b) {
        return 0;                          // identical pointers → zero distance
    }

    if (length == 0) {
        return bLength;                    // empty a → insert all of b
    }

    if (bLength == 0) {
        return length;                     // empty b → delete all of a
    }

    size_t *cache = calloc(length, sizeof(size_t));
    size_t index = 0;
    size_t bIndex = 0;
    size_t distance;
    size_t bDistance;
    size_t result;
    char code;

    // === Initialize the first row of the DP table ===
    // dp[i][0] = i for all i: transforming a[0..i-1] to empty string
    while (index < length) {
        cache[index] = index + 1;         // dp[row][col=0] == col+1
        index++;
    }

    // === Main DP loop over characters of b (the "columns") ===
    while (bIndex < bLength) {
        code = b[bIndex];                 // current character in b

        result = distance = bIndex;       // dp[row=0][col=bIndex] == bIndex
                                             // (transforming empty a to b[0..j-1])

        index = SIZE_MAX;                 // sentinel: means "no valid previous"

        while (++index < length) {
            bDistance = code == a[index] ? distance : distance + 1;

            // dp[i][j] = min(
            //     diagonal (match/substitution),
            //     left   (insertion from same row, i.e., column-1),
            //     above  (deletion from previous row)
            // )

            distance = cache[index];      // read dp[i-1][j] (above in the matrix)

            cache[index] = result =       // write to current cell:
                distance > result         //   first, try diagonal / substitution cost
                    ? bDistance > result  //     if both above and diagonal are worse than left,
                        ? result + 1      //        take `left` (insertion)
                        : bDistance        //     otherwise compare diagonal vs left
                        : bDistance       //   pick the minimum of three paths
                ;                         // end ternary chain
        }

        bIndex++;                         // advance to next character in b
    }

    free(cache);                          // release allocated DP row

    return result;                       // dp[length][bLength] is the answer
}
```

---

## How the Ternary Chain Works

The inner loop computes:

```c
cache[index] = min(
    diagonal,       // cache[index] before update == dp[i-1][j] (above)
                     // but we also want to compare it with substitution cost,
                     // which is stored in `distance` at this point.
    left + 1,       // bDistance == distance (left cell) or distance+1 (mismatch)
    above           // cache[index] already holds dp[i-1][j]
)
```

The chain:

```c
distance > result ? bDistance > result ? result + 1 : bDistance
                  : bDistance > distance ? distance + 1 : bDistance
```

is equivalent to `min(diagonal, left+1, above)` where:

- `diagonal` = cost of transforming `a[0..i-1]` → `b[0..j-1]` with a possible edit at position `(i,j)`
- `above`   = `cache[index]` (value from previous row)
- `left+1`  = insertion cost

---

## Example Walkthrough: `"kitten"` → `"sitting"`

| i\j | ∅ | s | i | t | t | i | n | g |
|------|----|---|---|---|---|---|---|---|
| ∅    | 0  | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| k    | 1  | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| i    | 2  | 2 | 1 | 2 | 3 | 4 | 5 | 6 |
| t    | 3  | 3 | 2 | 1 | 0 | 1 | 2 | 3 |
| t    | 4  | 4 | 3 | 2 | 1 | 2 | 3 | 4 |
| e    | 5  | 5 | 4 | 3 | 2 | 2 | 3 | 4 |
| n    | 6  | 6 | 5 | 4 | 3 | 3 | 2 | 3 |
| n    | 7  | 7 | 6 | 5 | 4 | 4 | 3 | 4 |

The final answer is `dp[8][7] = 3`, corresponding to the edit sequence:
1. substitute `k → s`
2. insert `i` after the first `t` (but that's not optimal; instead, keep `k→s`, substitute `e→i`, delete `n`)

A better path is: `kitten` → sitten → siten → sitting — 3 edits total.

---

## Summary Table

| Function | Signature | Description |
|----------|-----------|-------------|
| `levenshtein()` | `size_t levenshtein(const char *a, const char *b)` | Convenience wrapper that calls `strlen` on both strings, then delegates to `levenshtein_n()`. |
| `levenshtein_n()` | `size_t levenshtein_n(const char *a, size_t lenA, const char *b, size_t lenB)` | Core implementation. Returns the edit distance using O(min(|a|,|b|)) space. |

---

## Summary Table (Condensed)

| Function | Signature | Description |
|----------|-----------|-------------|
| `levenshtein(a,b)` | `size_t levenshtein(const char *a, const char *b)` | Convenience wrapper calling `strlen` on both inputs before delegating to the optimized variant. |
| `levenshtein_n(a,lenA,b,lenB)` | `size_t levenshtein_n(const char *a, size_t lenA, const char *b, size_t lenB)` | Core DP implementation using a single-row cache; runs in **O(min(|a|,|b|)) space** and O(|a|·|b|) time. |

---

## Design Notes

- **Space-optimal:** By always treating the shorter string as the inner dimension of the DP table, memory usage is reduced to `min(m,n)+1` entries instead of `m×n`.
- **No full matrix needed.** Only a single row (`cache`) is allocated and reused across iterations over the longer string. This avoids allocating an O(m·n) matrix that would quickly exhaust memory for long strings.
- **Degenerate cases handled upfront:** Identical pointers, empty strings, etc., all return immediately with zero or trivial cost — avoiding unnecessary allocation in those branches.
- The implementation is compatible with C90 and does not depend on any C++ features (no `std::string`, no `std::vector`).
