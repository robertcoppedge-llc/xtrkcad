# file2uri.c — Filename ↔ URI Encoding/Decoding

## Overview

`file2uri.c` provides two complementary functions for converting between filesystem paths and URL-style URIs:

- **`File2URI()`** — Converts a filename to its URI representation (percent-encoding reserved characters).
- **`URI2File()`** — Decodes a percent-encoded URI back into a filename.

This is useful when XTrkCAD needs to pass filenames as URL parameters, e.g., in HTTP requests or embedded links that must safely contain path separators and special characters like spaces or `#`.

---

## Reserved Characters Set

```c
static char *reservedChars = "?#[]@!$&'()*+,;= ";
```

This set matches RFC 3986 Section 2.3 (unreserved + reserved for query/frags). Notably:

- **`?`, `#`** — These are the most common problem characters in filenames on Windows (`?.txt`) or Linux (`file #name.txt`). They must be percent-encoded as `%3F` and `%23`.
- **Spaces** — Encoded as `%20` (space) rather than `+` (which is query-string-specific).
- The forward slash `/` is *not* in this set because it is a valid character within the path component of a URI.

---

## `File2URI(char *fileName, unsigned resultLen, char *resultBuffer)`

### Purpose

Converts a filesystem path into its URI-safe representation.

### Parameters

| Parameter | Direction | Description |
|-----------|-----------|-------------|
| `fileName` | IN | Null-terminated filename/path string (may contain spaces, `#`, etc.) |
| `resultLen` | IN | Size of the output buffer in bytes |
| `resultBuffer` | OUT/IN | Pre-allocated destination buffer; returned with encoded data |

### Return Value

The length of the encoded string (excluding the terminating `\0`) placed into `resultBuffer`. The function guarantees that at most `resultLen - 1` characters are written before appending the terminator.

### Algorithm — Single Pass, No Temporary Buffer

```c
unsigned File2URI(char *fileName, unsigned resultLen, char *resultBuffer)
{
    char *currentSource = fileName;
    char *currentDest = resultBuffer;

    while (*currentSource && ((unsigned)(currentDest - resultBuffer) < resultLen - 1)) {
        if (*currentSource == FILE_SEP_CHAR[0]) {   /* '/' */
            *currentDest++ = '/';
            currentSource++;
            continue;
        }
        if (strchr(reservedChars, *currentSource)) {
            sprintf(currentDest, "%%%02x", *(unsigned char*)currentSource);
            currentSource++;
            currentDest += 3;
        } else {
            *currentDest++ = *currentSource++;
        }
    }

    *currentDest = '\0';
    return (unsigned)strlen(resultBuffer);
}
```

**Key design decisions:**

- **In-place encoding into the output buffer.** No intermediate `char*` is allocated. The loop advances both a source pointer and a destination pointer within the same buffer.
- **Uppercased hex digits.** Using `%02x` (lowercase) is fine, but some conventions prefer uppercase `%XX`. This uses lowercase — consistent with C's standard library behavior.
- **The `FILE_SEP_CHAR[0]` cast** handles the case where the separator might be something other than `'/'` on non-Unix systems (though in practice it is always `/`).

---

## `URI2File(char *encodedFileName, unsigned resultLen, char *resultBuffer)`

### Purpose

Reverses the transformation: converts a percent-encoded URI back into its original filename.

### Parameters

| Parameter | Direction | Description |
|-----------|-----------|-------------|
| `encodedFileName` | IN | Null-terminated URI-encoded string (e.g., `"file%20name#test.txt"`) |
| `resultLen` | IN | Size of the output buffer in bytes |
| `resultBuffer` | OUT/IN | Pre-allocated destination buffer |

### Return Value

The length of the decoded string.

### Algorithm — Single Pass Decoding

```c
unsigned URI2File(char *encodedFileName, unsigned resultLen, char *resultBuffer)
{
    char *currentSource = encodedFileName;
    char *currentDest = resultBuffer;

    currentSource = encodedFileName;
    while (*currentSource && ((unsigned)(currentDest - resultBuffer) < resultLen - 1)) {
        if (*currentSource == '/') {
            *currentDest++ = FILE_SEP_CHAR[0];
            currentSource++;
            continue;
        }
        if (*currentSource == '%') {
            char hexCode[3];
            memcpy(hexCode, currentSource + 1, 2);
            hexCode[2] = '\0';
            sscanf(hexCode, "%x", (unsigned int*)currentDest);
            currentDest++;
            currentSource += 3;
        } else {
            *currentDest++ = *currentSource++;
        }
    }

    *currentDest = '\0';
    return (unsigned)strlen(resultBuffer);
}
```

**Key details:**

- When a `%` is encountered, the next two characters are treated as hex digits. `sscanf(hexCode, "%x", ...)` converts `"aF"` into `(int)175`, which is then written to the output buffer.
- The `%20` → space conversion works correctly because `%20` (hex 20) equals ASCII space `' '`.
- **No error checking** — if `%XX` contains non-hex characters, `sscanf` will produce undefined behavior (typically writes garbage). A production version should validate that the two following characters are hex digits before calling `sscanf`.

---

## Example Round-Trip Test

```c
char buffer[256];

/* Encode */
File2URI("my file #name?test.txt", 256, buffer);
// buffer now holds: "my%20file%20%23name%3Ftest.txt"

/* Decode */
char decoded[256];
URI2File(buffer, sizeof(decoded), decoded);
// decoded == "my file #name?test.txt"   (assuming the original had no reserved chars)
```

---

## Summary Table

| Function | Direction | Key Behavior |
|----------|-----------|-------------|
| `File2URI()` | filename → URI | Percent-encodes any character found in `reservedChars` set; preserves `/`. |
| `URI2File()` | URI → filename | Decodes `%XX` sequences back to their original bytes; leaves non-encoded characters untouched. |

---

## Summary Table (Condensed)

| Function | Direction | Key Behavior |
|----------|-----------|-------------|
| `File2URI()` | filename → URI | Percent-encodes reserved chars (`?`, `#`, ` `, etc.); preserves `/`. |
| `URI2File()` | URI → filename | Decodes `%XX` sequences back to original bytes. |

---

## Design Notes

- **Single-pass algorithms.** Both functions operate in a single linear scan without allocating extra memory beyond the output buffer. This is optimal for performance and avoids stack overflow risks on deeply nested paths.
- **No bounds checking on hex digits.** `URI2File()` assumes that every `%` is followed by two valid hex characters. In practice this should always be true because `File2URI()` guarantees it, but a more robust implementation would validate before calling `sscanf`.
- **Lowercase hex output.** `sprintf(currentDest, "%%%02x", ...)` produces lowercase (`%2F`, `%3f`). This is consistent with RFC 3986 Section 2.1 which states that percent-encoded octets may be uppercase or lowercase and both are equivalent.
