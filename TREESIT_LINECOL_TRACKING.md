# Tree-Sitter Line/Column Tracking in Emacs: A Complete Guide

This document explains how Emacs tracks line and column numbers for the tree-sitter integration. It covers **why** the tracking exists, **how** caches work, **when** caches are valid versus stale, and the exact algorithms used to keep everything in sync without expensive full-buffer scans.

---

## Table of Contents

1. [Why Line/Column Tracking Exists](#1-why-linecol-tracking-exists)
2. [The Core Data Structure: `struct ts_linecol`](#2-the-core-data-structure)
3. [Where Caches Live](#3-where-caches-live)
4. [Enabling and Disabling Tracking](#4-enabling-and-disabling-tracking)
5. [The Sentinel Convention: Empty vs Valid](#5-the-sentinel-convention)
6. [Computing Line/Column from a Cache: `treesit_linecol_of_pos`](#6-computing-linecol-from-a-cache)
7. [The Cache Update Algorithm: `compute_new_linecol_by_change`](#7-the-cache-update-algorithm)
8. [How a Buffer Edit Flows Through the System](#8-how-a-buffer-edit-flows)
9. [Narrowing Sync: `treesit_sync_visible_region`](#9-narrowing-sync)
10. [Translation to Tree-Sitter's TSPoint: `treesit_make_ts_point`](#10-translation-to-tspoint)
11. [Call Sites: Where Edits Enter the System](#11-call-sites)
12. [Debug and Test Infrastructure](#12-debug-and-test-infrastructure)
13. [Summary of Invariants](#13-summary-of-invariants)

---

## 1. Why Line/Column Tracking Exists

Tree-sitter's `TSInputEdit` struct requires three `TSPoint` values (row/column pairs) for each edit: the start, old end, and new end of the change. In most grammars, tree-sitter doesn't actually *use* these values for parsing — it just carries them around and returns them when reporting node positions. So for years, Emacs simply sent dummy values (`{1, 0}`) and everything worked.

Then came grammars like **Haskell's** ([tree-sitter issue #4001](https://github.com/tree-sitter/tree-sitter/issues/4001)), which use **layout-sensitive parsing** — the grammar literally inspects column positions to determine how to parse indentation-sensitive syntax. With dummy values, Haskell parsing breaks.

The problem: Emacs uses a **gap buffer** and does **not** natively track line/column positions. Computing the line number of a byte position requires scanning from the beginning of the buffer, counting newlines — an O(n) operation that would be unacceptable at every keystroke.

The solution: **A caching system** that maintains a small number of known-good `(bytepos, line, column)` triples. When we need the line/column of a new position, we scan from the *nearest* cache point rather than from the beginning of the buffer. And when a buffer edit occurs, we update the caches arithmetically (without scanning) whenever possible.

**Key design constraints:**
- Tracking is **opt-in** per language, since most grammars don't need it.
- Once enabled for a buffer, tracking is **never disabled** (simplifies the code).
- A parser's tracking status is set at creation time and never changes.
- The system must **never scan long distances** — caches are chosen to be near the positions we need.

---

## 2. The Core Data Structure

Defined in `src/buffer.h:226-235`:

```c
struct ts_linecol
{
  ptrdiff_t bytepos;  // The byte position in the buffer (1-based)
  ptrdiff_t line;     // Line number at this position (1-based)
  ptrdiff_t col;      // Column in bytes, 0-based offset from BOL (or BOB)
};
```

**Key facts about the values:**

- `bytepos` is a **buffer byte position** (1-based, like all Emacs byte positions).
- `line` is the **absolute line number** in the buffer (1-based, counting from the beginning of the buffer, ignoring narrowing).
- `col` is the **byte offset from the beginning of the current line** (0-based). It's measured in bytes, not characters, because tree-sitter works with byte offsets.
- Line and column numbers **do not respect narrowing** — they're always relative to the actual beginning of the buffer.

---

## 3. Where Caches Live

There are **two kinds** of cache locations:

### 3.1 Buffer-Level Caches (3 caches per buffer)

Stored in `struct buffer` (buffer.h:735-737):

```c
struct ts_linecol ts_linecol_begv;   // Cache near BEGV (beginning of visible region)
struct ts_linecol ts_linecol_point;  // Cache near point (where edits happen)
struct ts_linecol ts_linecol_zv;     // Cache near ZV (end of visible region)
```

Accessor macros:

| Getter | Setter |
|---|---|
| `BUF_TS_LINECOL_BEGV(buf)` | `SET_BUF_TS_LINECOL_BEGV(buf, val)` |
| `BUF_TS_LINECOL_POINT(buf)` | `SET_BUF_TS_LINECOL_POINT(buf, val)` |
| `BUF_TS_LINECOL_ZV(buf)` | `SET_BUF_TS_LINECOL_ZV(buf, val)` |

**Important subtlety from the buffer.h comment (line 725-734):** These caches are "always up-to-date" in the sense that their `bytepos` and `line`/`col` values are **internally consistent** — they correctly describe the same position. But the `bytepos` stored in a cache does **not** have to match the actual current position of point/BEGV/ZV. After an edit, `ts_linecol_point.bytepos` will typically be at the `new_end` of the last edit (which is *near* point but not necessarily *at* point). The cache is a **nearby reference point**, not a live tracker.

### 3.2 Parser-Level Caches (2 caches per parser)

Stored in `struct Lisp_TS_Parser` (treesit.h:106-108):

```c
struct ts_linecol visi_beg_linecol;  // Line/col at visible_beg
struct ts_linecol visi_end_linecol;  // Line/col at visible_end
```

These track the line/column of the parser's **visible region boundaries** (`visible_beg` and `visible_end`). Unlike the buffer caches, these **must** always have `bytepos` exactly equal to `visible_beg`/`visible_end` (enforced by assertions). This is because when we tell tree-sitter about a narrowing change via `treesit_sync_visible_region`, we need to provide `TSPoint` values relative to `visible_beg`, so `visi_beg_linecol` must exactly match the current `visible_beg`.

---

## 4. Enabling and Disabling Tracking

### When tracking is enabled

Tracking is enabled when `treesit-parser-create` creates a parser whose language is in the list `treesit-languages-require-line-column-tracking` (treesit.c:2401-2422):

```c
const bool lang_need_linecol_tracking
  = !NILP (Fmemq (remapped_lang,
                   Vtreesit_languages_require_line_column_tracking));
```

If this is true, two things happen:

1. **The parser** gets its `visi_beg_linecol` and `visi_end_linecol` initialized by scanning from `TREESIT_BOB_LINECOL` (which is always valid at `{bytepos=1, line=1, col=0}`).

2. **The buffer** (if not already tracking) gets its three caches initialized to `TREESIT_BOB_LINECOL`. This is a safe starting point — BOB is always at line 1, column 0, regardless of buffer content.

### Permanence

- Once a **buffer** starts tracking, it **never stops** — even if all parsers that need tracking are deleted.
- A **parser's** tracking status is determined at creation time and never changes, regardless of later modifications to `treesit-languages-require-line-column-tracking`.

### Checking tracking status

```c
// Buffer-level: is ts_linecol_begv initialized (non-zero bytepos)?
static bool treesit_buf_tracks_linecol_p (struct buffer *buf) {
  return BUF_TS_LINECOL_BEGV (buf).bytepos != 0;
}

// Parser-level: is visi_beg_linecol initialized?
XTS_PARSER (parser)->visi_beg_linecol.bytepos == 0  // not tracking
XTS_PARSER (parser)->visi_beg_linecol.bytepos != 0  // tracking
```

Exposed to Lisp as `treesit-tracking-line-column-p` and `treesit-parser-tracking-line-column-p`.

---

## 5. The Sentinel Convention: Empty vs Valid

Two constants define the sentinel values (treesit.c:490-493):

```c
// Always valid — represents beginning of buffer (line 1, col 0, bytepos 1)
static struct ts_linecol const TREESIT_BOB_LINECOL = { 1, 1, 0 };

// "Not initialized / tracking disabled"
const struct ts_linecol TREESIT_EMPTY_LINECOL = { 0, 0, 0 };
```

The convention throughout the code:

- **`bytepos == 0`** means "this linecol is empty / not tracking". Functions check this to skip linecol processing.
- **`bytepos != 0`** means "this linecol is valid and contains a real cached position".

This convention is used everywhere:
- `treesit_linecol_maybe()` returns `TREESIT_EMPTY_LINECOL` when there's no parser or tracking is disabled.
- `treesit_record_change_1()` checks `start_linecol.bytepos != 0` to decide whether to process linecol.
- `treesit_record_change()` checks `new_end_linecol.bytepos != 0` to decide whether to update buffer caches.

---

## 6. Computing Line/Column from a Cache: `treesit_linecol_of_pos`

This is the fundamental function that computes the line/column of an arbitrary byte position, given a nearby cached position to scan from.

**Signature** (treesit.c:1080):

```c
static struct ts_linecol
treesit_linecol_of_pos (ptrdiff_t target_bytepos,
                        struct ts_linecol cache)
```

**Precondition:** `cache` must be valid (non-empty).

**How it works:**

### Case 1: Same position
If `target_bytepos == cache.bytepos`, return `cache` immediately.

### Case 2: Forward scan (target is after cache)
```
cache.bytepos -----> target_bytepos
```

1. Walk forward from `cache.bytepos` to `target_bytepos`, counting newlines using `treesit_count_lines`.
2. Track the position of the last newline found (`byte_pos_1`) — this is needed for computing the column.
3. Compute:
   - `line = cache.line + line_delta`
   - `col = target_bytepos - byte_pos_1` (if we crossed newlines — distance from last newline)
   - `col = (target_bytepos - cache.bytepos) + cache.col` (if no newlines crossed — just extend the column)

### Case 3: Backward scan (target is before cache)
```
target_bytepos <----- cache.bytepos
```

1. Walk backward counting newlines.
2. Compute:
   - `line = cache.line + line_delta` (line_delta is negative)
   - If no newlines crossed: `col = cache.col - (cache.bytepos - target_bytepos)` (just shrink the column)
   - If newlines crossed: find the previous newline before target and compute `col = target_bytepos - newline_pos`

### Performance

The cost is **proportional to the distance between `cache` and `target_bytepos`**, not the distance from the beginning of the buffer. The whole caching scheme is designed so that the cache is always *near* the position we need to compute, keeping scans short.

### `treesit_count_lines`

A helper (treesit.c:1030) that wraps `display_count_lines` but:
- **Ignores narrowing** (temporarily widens the buffer).
- **Ignores selective display** (temporarily disables it).
- Has slightly different backward-search semantics from `display_count_lines`.

---

## 7. The Cache Update Algorithm: `compute_new_linecol_by_change`

This is the clever part. When a buffer edit occurs, we have caches that are now potentially stale (their `bytepos` has shifted, and their `line`/`col` may be wrong). Rather than re-scanning from scratch, this function computes the new linecol **arithmetically** when possible.

**Signature** (treesit.c:1282):

```c
static struct ts_linecol
compute_new_linecol_by_change (
    struct ts_linecol pos_linecol,       // The cache we want to update
    struct ts_linecol start_linecol,     // Where the edit starts
    struct ts_linecol old_end_linecol,   // Where the old text ended
    struct ts_linecol new_end_linecol,   // Where the new text ends
    ptrdiff_t target_bytepos)            // The actual bytepos we want linecol for
```

**Key insight:** The function has **three cases** depending on where `pos_linecol` (the cache) is relative to the edit region:

### Case 1: Cache is BEFORE the edit start

```
pos       start ... old_end
 |          |         |
 ▼          ▼         ▼
 ┌──────────┬─────────┐
 │ unchanged│  edit    │
 └──────────┴─────────┘
```

If `start_linecol.bytepos >= pos_linecol.bytepos`, the edit is entirely after the cache. The cache is **unaffected** — its line and column haven't changed. Return `pos_linecol` as-is.

### Case 2: Cache is AFTER the edit end

```
start ... old_end    pos
  |         |         |
  ▼         ▼         ▼
  ┌─────────┬─────────┐
  │  edit    │unchanged│
  └─────────┴─────────┘
```

If `old_end_linecol.bytepos <= pos_linecol.bytepos`, the cache is beyond the edit. We can compute the new linecol **arithmetically**:

- **Line:** `new_line = pos.line + (new_end.line - old_end.line)`
  The shift in line count equals the difference in line count between new and old end of the edit.

- **Bytepos:** `new_bytepos = pos.bytepos + (new_end.bytepos - old_end.bytepos)`
  The shift in byte position equals the size change of the edit.

- **Column:** This is the tricky part. Two sub-cases:
  - If `old_end` and `pos` are on the **same line**: the column of pos shifts by the column shift at old_end → new_end:
    ```
    new_col = new_end.col + (pos.col - old_end.col)
    ```
    Visually:
    ```
    ########|OOOOO
    OOOOOOOOOO|########|
              ne       pos'
    ```
    `pos'`'s column = `new_end.col` + distance from `old_end` to `pos`.

  - If they're on **different lines**: the column of pos is unchanged.
    ```
    new_col = pos.col
    ```
    Because the newline before pos hasn't moved.

### Case 3: Cache is INSIDE the edit region

```
start    pos    old_end
  |       |       |
  ▼       ▼       ▼
  ┌───────┬───────┐
  │  edit region   │
  └───────┴───────┘
```

If `start.bytepos < pos.bytepos < old_end.bytepos`, we can't compute things arithmetically — the text that `pos` pointed to has been deleted or replaced. We fall back to **scanning** from whichever of `start_linecol` or `new_end_linecol` is closer to `target_bytepos`:

```c
if (target_bytepos - start.bytepos < |target_bytepos - new_end.bytepos|)
    scan from start_linecol
else
    scan from new_end_linecol
```

### Final step

After cases 1-2 produce a valid `new_linecol` (which may not be at `target_bytepos` exactly), a final scan brings it to the target:

```c
if (new_linecol.bytepos != target_bytepos)
    new_linecol = treesit_linecol_of_pos (target_bytepos, new_linecol);
```

This final scan is expected to be **short** because `pos_linecol` was chosen to be near `target_bytepos`.

---

## 8. How a Buffer Edit Flows Through the System

Here's the complete flow when the user types a character (or any edit happens):

### Step 1: Buffer edit function captures the "before" state

In functions like `insert_1_both` (insdel.c), `replace_range` (insdel.c), `casify_region` (casefiddle.c), or `Ftranspose_regions` (editfns.c):

```c
// Before the edit:
struct ts_linecol start_linecol
  = treesit_linecol_maybe (PT, PT_BYTE,
                           BUF_TS_LINECOL_POINT (current_buffer));
```

`treesit_linecol_maybe` does two things:
1. Checks if the buffer has parsers and tracks linecol. If not, returns `TREESIT_EMPTY_LINECOL`.
2. If tracking, calls `treesit_linecol_of_pos(PT_BYTE, point_cache)` to compute the linecol of the edit start by scanning from the point cache. This is fast because the point cache is near point.

For replacements/deletions, we also capture `old_end_linecol` the same way.

For **pure insertions** (no old text), `old_end_linecol == start_linecol` (the old end is the same position as the start).

### Step 2: The actual edit happens

The buffer is modified (gap buffer operations, etc.).

### Step 3: `treesit_record_change` is called

```c
treesit_record_change (start_byte, old_end_byte, new_end_byte,
                       start_linecol, old_end_linecol, new_end_charpos);
```

This outer wrapper:

1. **Computes `new_end_linecol`** by scanning from `start_linecol` to `new_end_byte`:
   ```c
   struct ts_linecol new_end_linecol
     = treesit_linecol_maybe (new_end, new_end_byte, start_linecol);
   ```
   This scan is short because `start_linecol` is right before the edit, and `new_end` is right after.

2. **Calls `treesit_record_change_1`** with all six pieces of information.

3. **Updates the three buffer caches:**
   - `BEGV` cache: updated via `compute_new_linecol_by_change` (arithmetic, since BEGV is usually far from the edit → Case 1 or 2)
   - `point` cache: set directly to `new_end_linecol` (the most recently computed position, which is where point will be after the edit)
   - `ZV` cache: updated via `compute_new_linecol_by_change` (arithmetic, since ZV is usually far from the edit → Case 1 or 2)

### Step 4: `treesit_record_change_1` updates each parser

For each parser in the buffer's parser list:

1. **Check if parser tracks linecol.** If the parser's `visi_beg_linecol.bytepos == 0`, skip all linecol work for this parser (just do byte-level tree edits with dummy TSPoints).

2. **Clip the edit region** to the parser's `visible_beg`/`visible_end`.

3. **Update `visible_beg`/`visible_end`** byte positions.

4. **If parser tracks linecol:**
   - Compute new `visi_beg_linecol` via `compute_new_linecol_by_change` (arithmetic — visible_beg is usually before or after the edit)
   - Compute new `visi_end_linecol` via `compute_new_linecol_by_change` (arithmetic — visible_end is usually far from the edit)
   - Compute `TSPoint` values via `treesit_make_ts_point` using the now-updated `visi_beg_linecol` as the reference point

5. **Call `treesit_tree_edit_1`** to record the edit in the tree-sitter tree (with proper TSPoints).

6. **Set `need_reparse = true`** on the parser.

### The flow visualized:

```
User types 'x' at point
    │
    ▼
insert_1_both (insdel.c)
    ├── 1. Capture start_linecol = linecol_of_pos(PT_BYTE, point_cache)
    │      [short scan from nearby cache]
    ├── 2. Perform the actual insertion
    └── 3. treesit_record_change(PT_BYTE, PT_BYTE, PT_BYTE+1, ...)
            ├── Compute new_end_linecol = linecol_of_pos(PT_BYTE+1, start)
            │   [scan of 1 byte]
            ├── treesit_record_change_1()
            │   └── For each parser:
            │       ├── Clip to visible region
            │       ├── Update visible_beg/end bytepos
            │       ├── If parser tracks linecol:
            │       │   ├── visi_beg_linecol = compute_new...() [arithmetic]
            │       │   ├── visi_end_linecol = compute_new...() [arithmetic]
            │       │   └── Make TSPoints from updated caches
            │       ├── ts_tree_edit(tree, edit)
            │       └── need_reparse = true
            └── Update buffer caches:
                ├── begv_cache = compute_new...(begv, ...) [arithmetic]
                ├── point_cache = new_end_linecol         [direct set]
                └── zv_cache = compute_new...(zv, ...)   [arithmetic]
```

---

## 9. Narrowing Sync: `treesit_sync_visible_region`

When the user narrows/widens the buffer, the parser's `visible_beg`/`visible_end` may no longer match `BUF_BEGV_BYTE`/`BUF_ZV_BYTE`. This is reconciled in `treesit_sync_visible_region` (treesit.c:1625), called just before parsing.

This function also needs to provide proper TSPoints when simulating inserts/deletes at the visible region boundaries. It:

1. Gets the parser's current `visi_beg_linecol`/`visi_end_linecol`.
2. Computes `buffer_begv_linecol` and `buffer_zv_linecol` by scanning from the buffer caches:
   ```c
   buffer_begv_linecol = treesit_linecol_of_pos(BUF_BEGV_BYTE, buf->ts_linecol_begv);
   buffer_zv_linecol   = treesit_linecol_of_pos(BUF_ZV_BYTE,   buf->ts_linecol_zv);
   ```
   These scans are short because the buffer caches are near BEGV/ZV.

3. For each narrowing adjustment (4 possible cases — expand/shrink at beginning/end), provides proper `TSPoint` values computed via `treesit_make_ts_point`.

4. After all adjustments, stores the updated `visi_beg_linecol` and `visi_end_linecol` back into the parser.

---

## 10. Translation to Tree-Sitter's TSPoint: `treesit_make_ts_point`

Tree-sitter's `TSPoint` is `{ uint32_t row, uint32_t column }` — a **0-based row/column pair relative to the beginning of the tree-sitter tree** (which starts at `visible_beg`).

**Signature** (treesit.c:1181):
```c
static TSPoint
treesit_make_ts_point (struct ts_linecol visible_beg,
                       struct ts_linecol pos)
```

**How it works:**

Both `visible_beg` and `pos` are absolute `ts_linecol` values (line/col relative to beginning of buffer). We need to compute a position **relative to visible_beg**:

- **If same line** (`visible_beg.line == pos.line`):
  ```c
  point.row = 0;
  point.column = pos.col - visible_beg.col;
  ```
  Both are on line 0 of the tree, column is the byte difference.

- **If different lines** (`pos.line > visible_beg.line`):
  ```c
  point.row = pos.line - visible_beg.line;
  point.column = pos.col;  // col is already relative to BOL
  ```
  Row is the line difference, column is the absolute column (since the line starts at the left margin).

---

## 11. Call Sites: Where Edits Enter the System

Every buffer modification function that can affect tree-sitter follows the same pattern:

| Function | File | Edit Type | Notes |
|---|---|---|---|
| `insert_1_both` | insdel.c:894 | Insert at point | `old_end == start` (pure insert) |
| `insert_from_string_1` | insdel.c:1036 | Insert string at point | `old_end == start` |
| `insert_from_gap_1` | insdel.c:1131 | Insert from gap at GPT | Used by process output |
| `insert_from_buffer` | insdel.c:1219 | Insert from another buffer | `old_end == start` |
| `replace_range` | insdel.c:1525 | Replace region (delete+insert) | Both `start` and `old_end` captured |
| `del_range_1` | insdel.c:2003 | Delete region | `new_end_byte == start_byte` |
| `Fsubst_char_in_region` | editfns.c:2318 | Replace characters in region | Same-size replacement |
| `casify_region` | casefiddle.c:547 | Change case of region | May change byte length |
| `Ftranspose_regions` | editfns.c:4602 | Transpose two regions | Reports entire span as changed |
| `casify_region` (inner loop) | editfns.c:2614 | Per-character case change | Manual `treesit_record_change` call |

The pattern is always:
1. **Before** the edit: capture `start_linecol` (and `old_end_linecol` for deletions/replacements) using `treesit_linecol_maybe`, scanning from `BUF_TS_LINECOL_POINT`.
2. **After** the edit: call `treesit_record_change(start_byte, old_end_byte, new_end_byte, start_linecol, old_end_linecol, new_end_charpos)`.

`treesit_linecol_maybe` is a guard function that returns `TREESIT_EMPTY_LINECOL` if there's no parser or no tracking. This means the linecol computation is completely skipped for buffers that don't use tree-sitter or don't need linecol tracking — zero overhead.

---

## 12. Debug and Test Infrastructure

### Debug Mode

Set `TREESIT_DEBUG_LINECOL` to `true` at compile time (treesit.c:997) to enable assertions in every linecol computation:

```c
#define TREESIT_DEBUG_LINECOL false
```

When enabled, `treesit_debug_validate_linecol` (treesit.c:1065) verifies that a linecol's `line` field matches the actual newline count from `BEG_BYTE` to `bytepos`:

```c
static void treesit_debug_validate_linecol (struct ts_linecol linecol) {
  eassert (linecol.bytepos <= Z_BYTE);
  ptrdiff_t true_line_count = treesit_count_lines(BEG_BYTE, linecol.bytepos, ...) + 1;
  eassert (true_line_count == linecol.line);
}
```

### Debug Print

```c
void treesit_debug_print_linecol (struct ts_linecol linecol) {
  printf ("{ line=%td col=%td bytepos=%td }\n",
          linecol.line, linecol.col, linecol.bytepos);
}
```

### Lisp-Level Test Functions

Three internal functions (treesit.c:5136-5192):

| Function | Description |
|---|---|
| `treesit--linecol-at POS` | Compute linecol at POS using the point cache. Returns `(LINE . COL)`. |
| `treesit--linecol-cache-set LINE COL BYTEPOS` | Manually set the point cache (for testing). |
| `treesit--linecol-cache` | Return the current point cache as `(:line LINE :col COL :bytepos BYTEPOS)`. |

These are for internal testing and debugging ONLY — they let tests verify that caches are being maintained correctly after edits.

---

## 13. Summary of Invariants

Here are all the invariants the system maintains, and why they matter:

### Always true (when tracking is enabled):

1. **Buffer caches are internally consistent.** Each `ts_linecol` value's `bytepos`, `line`, and `col` fields describe the same position. But the `bytepos` in a cache does *not* have to match any particular buffer position (like point or BEGV). It's just a "known good reference point" near that position.

2. **Parser `visi_beg_linecol.bytepos == visible_beg`** (enforced by assertions at treesit.c:1676, 1753). The parser's visible_beg cache must exactly track the `visible_beg` byte position. This is necessary because `treesit_make_ts_point` needs the line/col of the exact visible_beg to compute tree-sitter-relative coordinates.

3. **Parser `visi_end_linecol.bytepos == visible_end`** (enforced by assertion at treesit.c:1754). Same reason as above but for the end.

4. **`bytepos == 0` means "not tracking"**, `bytepos != 0` means "valid cache".

5. **Buffer caches are refreshed on every edit** (in `treesit_record_change`). After every edit:
   - `point_cache` = linecol at `new_end` of the edit
   - `begv_cache` = arithmetically updated
   - `zv_cache` = arithmetically updated

6. **Parser caches are refreshed on every edit** (in `treesit_record_change_1`). After every edit:
   - `visi_beg_linecol` = arithmetically updated to match new `visible_beg`
   - `visi_end_linecol` = arithmetically updated to match new `visible_end`

### When is a cache "expired"?

A cache is never truly "expired" in the sense that its internal `(bytepos, line, col)` triple becomes inconsistent. The system ensures that:

- **Before each edit:** caches are used to compute the linecol of the edit endpoints by scanning from the nearest cache. This relies on the caches being internally consistent.
- **After each edit:** caches are updated either arithmetically (cases 1-2 of `compute_new_linecol_by_change`) or by re-scanning (case 3). The arithmetic update adjusts `bytepos`, `line`, and `col` together to maintain consistency.

A cache can become **suboptimal** (i.e., its `bytepos` drifts away from the position it was intended to be near, making future scans slightly longer). But it never becomes **wrong**. The only scenario where a cache's values are rebuilt from scratch is case 3 of `compute_new_linecol_by_change` — when the cache position falls inside the edited region.

### Why three buffer caches?

The three caches are positioned near three common positions:

- **`point` cache:** Near where the user is editing. Edit endpoints are scanned from this cache. Since edits happen at point, this cache is almost always very close to the edit position. After each edit, it's set to `new_end`, which is exactly where point will be after the edit.

- **`begv` cache:** Near the beginning of the visible region. Needed to update `visi_beg_linecol` for each parser, and to compute linecol during narrowing sync.

- **`zv` cache:** Near the end of the visible region. Same as above but for `visible_end`.

This three-cache scheme means that no linecol computation ever needs to scan across a large fraction of the buffer — there's always a nearby reference point.
