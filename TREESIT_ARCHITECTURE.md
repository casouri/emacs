# Emacs Tree-Sitter Integration: Complete Architecture Guide

This document provides a comprehensive reference for the tree-sitter integration in GNU Emacs. It covers both the C-level primitives (`src/treesit.c`, `src/treesit.h`) and the Emacs Lisp layer (`lisp/treesit.el`), maintained by Yuan Fu.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [C Layer: treesit.c (5529 lines)](#2-c-layer-treesitc)
   - [Data Types / Structs](#21-data-types--structs)
   - [Parser Lifecycle](#22-parser-lifecycle)
   - [Buffer Change Tracking & Incremental Parsing](#23-buffer-change-tracking--incremental-parsing)
   - [Node Operations](#24-node-operations)
   - [Query / Pattern Matching](#25-query--pattern-matching)
   - [Range Management](#26-range-management)
   - [Notifier System](#27-notifier-system)
   - [Memory Management](#28-memory-management)
   - [Error Handling](#29-error-handling)
   - [Complete DEFUN Reference](#210-complete-defun-reference)
3. [Lisp Layer: treesit.el (5839 lines)](#3-lisp-layer-treesitel)
   - [Font-Lock Engine](#31-font-lock-engine)
   - [Indentation Engine](#32-indentation-engine)
   - [Thing System & Navigation](#33-thing-system--navigation)
   - [Embedded Language Support](#34-embedded-language-support)
   - [Major Mode Setup](#35-major-mode-setup)
   - [Imenu Integration](#36-imenu-integration)
   - [Other Subsystems](#37-other-subsystems)
4. [Key Design Decisions](#4-key-design-decisions)
5. [Integration Points in Other Files](#5-integration-points-in-other-files)
6. [How to Create a New Tree-Sitter Major Mode](#6-how-to-create-a-new-tree-sitter-major-mode)

---

## 1. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  treesit.el  (Emacs Lisp — high-level features)              │
│  ┌───────────┐ ┌────────────┐ ┌───────┐ ┌─────────────────┐ │
│  │ Font-lock │ │ Indentation│ │ Imenu │ │ Navigation      │ │
│  │ Engine    │ │ Engine     │ │       │ │ (defun/sexp/...) │ │
│  └─────┬─────┘ └──────┬─────┘ └───┬───┘ └───────┬─────────┘ │
│        └───────────────┼───────────┼─────────────┘           │
│               treesit-major-mode-setup                       │
├──────────────────────────────────────────────────────────────┤
│  treesit.c  (C bridge — 52 DEFUNs)                           │
│  ┌────────────┐ ┌────────┐ ┌─────────┐ ┌──────────────────┐ │
│  │ Parser     │ │ Node   │ │ Query   │ │ Buffer change    │ │
│  │ lifecycle  │ │ access │ │ engine  │ │ tracking         │ │
│  └─────┬──────┘ └───┬────┘ └────┬────┘ └────────┬─────────┘ │
├────────┼────────────┼───────────┼────────────────┼───────────┤
│        └────────────┴───────────┴────────────────┘           │
│               libtree-sitter (external C library)             │
└──────────────────────────────────────────────────────────────┘
```

The integration has two layers:

- **`src/treesit.c`** — The C bridge. Exposes three Lisp pseudovector types (parsers, nodes, compiled queries) and 52 `DEFUN` functions. Handles all direct communication with the `libtree-sitter` C library. Manages incremental parsing, buffer change tracking, byte-to-charpos translation, and GC integration.

- **`lisp/treesit.el`** — The Emacs Lisp layer. Builds ~15 subsystems on top of the C primitives: font-lock, indentation, navigation, imenu, embedded language support, major mode setup helpers, and interactive debugging tools.

---

## 2. C Layer: treesit.c

### 2.1 Data Types / Structs

#### `struct Lisp_TS_Parser` (treesit.h:35-125)

The central data structure. Wraps a `TSParser *` and holds the current parse tree plus metadata.

| Field | Type | Description |
|---|---|---|
| `header` | `union vectorlike_header` | Pseudovector header (`PVEC_TS_PARSER`) |
| `language_symbol` | `Lisp_Object` | Symbol identifying the language (e.g., `python`, `c`) |
| `after_change_functions` | `Lisp_Object` | List of notifier functions called after re-parse |
| `tag` | `Lisp_Object` | Symbol tag to differentiate parsers of the same language (default `nil`) |
| `last_set_ranges` | `Lisp_Object` | Charpos `(BEG . END)` cons cells of last-set included ranges |
| `embed_level` | `Lisp_Object` | Nil or non-negative integer for parser nesting level |
| `buffer` | `Lisp_Object` | The buffer this parser is associated with |
| `parser` | `TSParser *` | Raw tree-sitter parser (never NULL) |
| `tree` | `TSTree *` | Current syntax tree (NULL before first parse) |
| `input` | `TSInput` | Callback struct for reading from the Emacs buffer |
| `need_reparse` | `bool` | True when buffer has changed since last parse |
| `visible_beg` | `ptrdiff_t` | Buffer byte position of tree-sitter's visible region start (1-based) |
| `visible_end` | `ptrdiff_t` | Buffer byte position of tree-sitter's visible region end (1-based) |
| `visi_beg_linecol` | `struct ts_linecol` | Cached line/col for `visible_beg` |
| `visi_end_linecol` | `struct ts_linecol` | Cached line/col for `visible_end` |
| `timestamp` | `ptrdiff_t` | Monotonically increasing counter; incremented on every buffer change |
| `deleted` | `bool` | If true, parser has been deleted; functions signal `treesit-parser-deleted` |
| `need_to_gc_buffer` | `bool` | If true, killing the parser also kills its buffer (for `treesit-parse-string` temp buffers) |
| `within_reparse` | `bool` | Re-entrancy guard for `treesit_ensure_parsed` |

#### `struct Lisp_TS_Node` (treesit.h:128-142)

| Field | Type | Description |
|---|---|---|
| `header` | `union vectorlike_header` | Pseudovector header (`PVEC_TS_NODE`) |
| `parser` | `Lisp_Object` | Reference to owning parser (prevents GC of tree while nodes live) |
| `node` | `TSNode` | Raw tree-sitter node (value type, not a pointer) |
| `timestamp` | `ptrdiff_t` | Snapshot of parser's timestamp at creation; mismatch = outdated |

#### `struct Lisp_TS_Query` (treesit.h:155-172)

| Field | Type | Description |
|---|---|---|
| `header` | `union vectorlike_header` | Pseudovector header (`PVEC_TS_COMPILED_QUERY`) |
| `language` | `Lisp_Object` | Language symbol |
| `source` | `Lisp_Object` | Original query source (string or sexp) |
| `query` | `TSQuery *` | Compiled query (NULL if lazy, compiled on first use) |
| `cursor` | `TSQueryCursor *` | Reusable cursor (created on demand) |

#### `struct ts_linecol` (buffer.h:226-235)

Used for line/column tracking (needed by grammars like Haskell).

| Field | Type | Description |
|---|---|---|
| `bytepos` | `ptrdiff_t` | Byte position (0 = uninitialized / not tracking) |
| `line` | `ptrdiff_t` | Line number |
| `col` | `ptrdiff_t` | Column in bytes (0-based from BOL or BOB) |

### 2.2 Parser Lifecycle

#### Creation: `treesit-parser-create` (treesit.c:2323)

`Ftreesit_parser_create(language, buffer, no_reuse, tag)`:

1. Calls `treesit_initialize()` to set up tree-sitter's allocator (mapped to `xmalloc`/`xrealloc`/`xfree`).
2. Resolves indirect buffers to their base buffer.
3. Unless `no_reuse` is non-nil, checks for an existing parser with the same language + tag and returns it.
4. Loads the language grammar via `treesit_load_language()`:
   - Constructs library name candidates: `libtree-sitter-LANG.so`, `.so.0`, `.so.0.0`, versioned variants.
   - Searches `treesit-extra-load-path`, `~/.emacs.d/tree-sitter/`, and system library paths.
   - Dynamically loads the shared library and calls `tree_sitter_LANG()` to get `TSLanguage *`.
   - Validates ABI version compatibility.
5. Creates `TSParser` via `ts_parser_new()`, sets its language.
6. Allocates the `Lisp_TS_Parser` pseudovector with `make_treesit_parser()`. Sets `TSInput` callback to `treesit_read_buffer` (UTF-8).
7. If language is in `treesit-languages-require-line-column-tracking`, enables linecol tracking.
8. Prepends parser to the buffer's `ts_parser_list`.

#### Deletion: `treesit-parser-delete` (treesit.c:2430)

Removes the parser from the buffer's parser list and sets `deleted = true`. The underlying `TSParser` and `TSTree` are freed later by GC.

#### GC Finalization (alloc.c)

- `PVEC_TS_PARSER` → `treesit_delete_parser()` (treesit.c:2094): frees `TSTree`, `TSParser`; optionally kills temp buffer.
- `PVEC_TS_COMPILED_QUERY` → `treesit_delete_query()` (treesit.c:2103): frees `TSQuery`, `TSQueryCursor`.
- `PVEC_TS_NODE` → no special cleanup (value type; ref to parser prevents premature GC).

### 2.3 Buffer Change Tracking & Incremental Parsing

This is the core mechanism that makes tree-sitter efficient.

#### Flow

```
Buffer edit (insert/delete/replace/casify/transpose)
  │
  ▼
treesit_record_change(start_byte, old_end_byte, new_end_byte, linecol...)
  │  Called from: insdel.c, editfns.c, casefiddle.c
  │
  ▼
treesit_record_change_1() — for each parser in buffer's ts_parser_list:
  ├─ Clip change region to parser's visible_beg/visible_end
  ├─ Compute offsets relative to visible_beg
  ├─ Update visible_beg and visible_end for size change
  ├─ Call ts_tree_edit() with TSInputEdit struct
  └─ Set need_reparse = true, increment timestamp

  ... time passes, no immediate re-parse ...

Node or parse result requested
  │
  ▼
treesit_ensure_parsed()
  ├─ Guard: within_reparse (prevents infinite recursion)
  ├─ treesit_check_buffer_size() — rejects buffers > 4 GiB
  ├─ treesit_sync_visible_region() — reconcile with narrowing
  ├─ If !need_reparse → return
  ├─ ts_parser_parse(parser, old_tree, input) — incremental parse
  ├─ Update tree, clear need_reparse, increment timestamp
  ├─ ts_tree_get_changed_ranges(old_tree, new_tree)
  ├─ Call after-change notifiers with changed ranges
  └─ Free old tree
```

#### `treesit_record_change` (treesit.c:1541)

Called from buffer modification functions. Parameters: `start_byte`, `old_end_byte`, `new_end_byte`, plus optional linecol data. Iterates over all parsers in the buffer and records the edit.

#### `treesit_sync_visible_region` (treesit.c:1625)

Called just before parsing. Synchronizes the parser's visible region with the buffer's current narrowing (`BUF_BEGV_BYTE` / `BUF_ZV_BYTE`). Simulates inserts/deletes at the boundaries of tree-sitter's view:

- `visible_beg > BUF_BEGV_BYTE` → simulate insert at beginning
- `visible_end < BUF_ZV_BYTE` → simulate insert at end
- `visible_end > BUF_ZV_BYTE` → simulate delete at end
- `visible_beg < BUF_BEGV_BYTE` → simulate delete at beginning

Also clips `last_set_ranges` to the new visible region.

#### `treesit_ensure_parsed` (treesit.c:1898)

The core lazy-parsing function. Only actually parses when `need_reparse` is true. After parsing, calls `ts_tree_get_changed_ranges` to get affected regions and fires notifier functions.

#### `treesit_read_buffer` (treesit.c:1952)

The `TSInput` read callback provided to tree-sitter:
- Receives 0-based `byte_index` from tree-sitter
- Adds `visible_beg` to get buffer byte position
- Returns one character at a time via `BUF_BYTE_ADDRESS`
- Reading one character at a time is intentional — benchmarks showed negligible difference vs. chunk reading, and it avoids gap-buffer complications

### 2.4 Node Operations

#### Obtaining Nodes

| Function | Line | Description |
|---|---|---|
| `treesit-parser-root-node` | 2585 | Calls `treesit_ensure_parsed`, returns root via `ts_tree_root_node` |
| `treesit-node-child` | 3052 | Nth child (supports negative N). Optional NAMED flag |
| `treesit-node-child-by-field-name` | 3215 | Child by grammar field name |
| `treesit-node-parent` | 3021 | Uses `treesit_cursor_helper` (recursive descent from root) |
| `treesit-node-first-child-for-pos` | 3331 | First child extending beyond a position |
| `treesit-node-descendant-for-range` | 3369 | Smallest descendant covering a byte range |
| `treesit-parse-string` | 2842 | One-off: creates temp buffer, parses string, returns root |

#### Node Properties

| Function | Line | Description |
|---|---|---|
| `treesit-node-type` | 2945 | Grammar type as a string |
| `treesit-node-start` / `treesit-node-end` | 2963/2984 | Buffer charpos (translated from tree-sitter byte offsets via `visible_beg`) |
| `treesit-node-string` | 3004 | S-expression of subtree |
| `treesit-node-check` | 3097 | Properties: `named`, `missing`, `extra`, `outdated`, `has-error`, `live` |
| `treesit-node-field-name-for-child` | 3153 | Field name of Nth child |
| `treesit-node-child-count` | 3192 | Child count (optional NAMED) |

#### Navigation

| Function | Line | Description |
|---|---|---|
| `treesit-node-next-sibling` | 3239 | Next sibling (optional NAMED) |
| `treesit-node-prev-sibling` | 3265 | Previous sibling (optional NAMED) |
| `treesit-node-eq` | 3419 | Structural equality test |

#### Outdated Node Detection

Nodes inherit the parser's `timestamp` at creation. Each buffer change increments the parser's timestamp. When any node function is called, `treesit_check_node` verifies `node->timestamp == parser->timestamp`. Mismatch → signal `treesit-node-outdated`.

#### The `treesit_cursor_helper` Problem

`TSNode` is a value type without parent-stack information. To find a parent, Emacs creates a `TSTreeCursor` at the root and recursively descends to the target node (`treesit_cursor_helper_1`), then moves to parent. Limited to `TREESIT_RECURSION_LIMIT = 1000` levels.

### 2.5 Query / Pattern Matching

#### Query Compilation: `treesit-query-compile` (treesit.c:3848)

Creates a `Lisp_TS_Query` object. Compilation is **lazy by default** — the `TSQuery *` is NULL until first use. This is necessary because `(defvar ... (treesit-compile-query ...))` in mode files would otherwise require grammar loading at `require` time.

`treesit_ensure_query_compiled` (treesit.c:2166):
1. If already compiled (`query->query` non-NULL), return.
2. Resolve language via `treesit-language-remap-alist`.
3. Load grammar.
4. If source is sexp, expand to string via `Ftreesit_query_expand`.
5. Call `ts_query_new()` to compile.

#### Query Execution: `treesit-query-capture` (treesit.c:3980)

The main query function:
1. Accepts NODE (node, parser, or language symbol — resolved via `treesit_resolve_node`).
2. Accepts QUERY (compiled query, string, or sexp).
3. Initializes query and cursor.
4. Optionally sets byte range via `ts_query_cursor_set_byte_range`.
5. Iterates matches with `ts_query_cursor_next_match`:
   - Collects captures as `(CAPTURE_NAME . NODE)` pairs.
   - Evaluates predicates per match pattern.
   - On predicate failure, rolls back results.
6. Supports `NODE-ONLY` (return nodes without capture names) and `GROUPED` (return list of match groups) modes.

#### Built-in Predicates

| Predicate | Function | Line | Description |
|---|---|---|---|
| `#eq?` | `treesit_predicate_equal` | 3686 | String equality of two captures or a capture and a literal |
| `#match?` | `treesit_predicate_match` | 3721 | Regex match of captured node text |
| `#pred?` | `treesit_predicate_pred` | 3785 | Arbitrary Lisp function called with captured nodes |

#### Query Expansion

`treesit-query-expand` (treesit.c:3540) and `treesit-pattern-expand` (treesit.c:3484) convert sexp query notation to tree-sitter string notation:
- `:anchor` → `.`
- `:?` → `?`
- `:*` → `*`
- `:+` → `+`
- Keywords like `:match`, `:eq`, `:pred` → `#match?`, `#eq?`, `#pred?`

### 2.6 Range Management

#### `treesit-parser-set-included-ranges` (treesit.c:2701)

Sets which regions of the buffer a parser should parse. Accepts a list of `(BEG . END)` cons cells (charpos). The function:
1. Validates ranges are ordered, non-overlapping, within visible region.
2. Compares with `last_set_ranges` to skip no-op updates.
3. Converts charpos to bytepos `TSRange` arrays (subtracts `visible_beg`).
4. Calls `ts_parser_set_included_ranges`.
5. Sets `need_reparse = true`.

Ranges are stored as charpos (not bytepos) to avoid a bytepos range cutting into a multibyte character after buffer edits.

If RANGES is nil, the parser parses the entire buffer.

### 2.7 Notifier System

Each parser has an `after_change_functions` list. Notifier functions receive `(RANGES PARSER)`:
- `RANGES` — list of `(BEG . END)` cons cells representing changed regions
- `PARSER` — the parser that was re-parsed

| API | Line | Description |
|---|---|---|
| `treesit-parser-notifiers` | 2776 | Returns copy of notifier list |
| `treesit-parser-add-notifier` | 2792 | Adds a function symbol (no lambdas) |
| `treesit-parser-remove-notifier` | 2812 | Removes a function symbol |

Notifiers fire from `treesit_ensure_parsed` (treesit.c:1942) after successful re-parse. Ranges come from `ts_tree_get_changed_ranges`. First parse reports entire visible region as changed.

### 2.8 Memory Management

Tree-sitter objects are Emacs **pseudovectors** managed by GC:

- **Parsers**: GC sweep → `treesit_delete_parser()` → free `TSTree`, `TSParser`, optionally kill temp buffer.
- **Queries**: GC sweep → `treesit_delete_query()` → free `TSQuery`, `TSQueryCursor`.
- **Nodes**: No special cleanup. Node holds ref to parser (preventing parser GC while nodes alive).

Tree-sitter's allocator is mapped to Emacs's at init: `ts_set_allocator(xmalloc, treesit_calloc_wrapper, xrealloc, xfree)` (treesit.c:574).

### 2.9 Error Handling

All errors use `xsignal` / `xsignal1` / `xsignal2`. Convention: signals are only raised in DEFUNs, not internal helpers (helpers use return values/output params).

| Signal | Parent | Description |
|---|---|---|
| `treesit-error` | `error` | Generic tree-sitter error |
| `treesit-query-error` | `treesit-error` | Malformed query pattern |
| `treesit-parse-error` | `treesit-error` | Parse failure |
| `treesit-range-invalid` | `treesit-error` | Overlapping/out-of-order/out-of-range ranges |
| `treesit-buffer-too-large` | `treesit-error` | Buffer > 4 GiB |
| `treesit-load-language-error` | `treesit-error` | Cannot load grammar (`not-found`, `symbol-error`, `version-mismatch`) |
| `treesit-node-outdated` | `treesit-error` | Node from before a re-parse |
| `treesit-node-buffer-killed` | `treesit-error` | Node's buffer is dead |
| `treesit-parser-deleted` | `treesit-error` | Parser has been deleted |
| `treesit-invalid-predicate` | `treesit-error` | Invalid predicate in `treesit-thing-settings` |

### 2.10 Complete DEFUN Reference

There are **52 DEFUNs** in treesit.c. Organized by category:

#### Language / Library (5)

| Function | Line | Description |
|---|---|---|
| `treesit-available-p` | 5197 | Non-nil if tree-sitter is built-in and available |
| `treesit-language-available-p` | 898 | Non-nil if grammar is loadable. DETAIL → `(t . nil)` or `(nil . DATA)` |
| `treesit-library-abi-version` | 930 | ABI version of tree-sitter library |
| `treesit-language-abi-version` | 947 | ABI version of a specific grammar |
| `treesit-grammar-location` | 972 | Absolute path of loaded grammar's shared library |

#### Line/Column Tracking (2)

| Function | Line | Description |
|---|---|---|
| `treesit-tracking-line-column-p` | 1200 | Non-nil if buffer tracks line/column |
| `treesit-parser-tracking-line-column-p` | 1218 | Non-nil if parser tracks line/column |

#### Type Predicates (7)

| Function | Line | Description |
|---|---|---|
| `treesit-parser-p` | 2233 | t if object is a parser |
| `treesit-node-p` | 2244 | t if object is a node |
| `treesit-compiled-query-p` | 2255 | t if object is a compiled query |
| `treesit-query-p` | 2266 | t if object is any query |
| `treesit-query-eagerly-compiled-p` | 2278 | Non-nil if compiled query is actually compiled |
| `treesit-query-language` | 2293 | Language symbol of compiled query |
| `treesit-query-source` | 2303 | Source of compiled query |

#### Parser Lifecycle & Properties (12)

| Function | Line | Description |
|---|---|---|
| `treesit-parser-create` | 2323 | Create/reuse parser for language in buffer |
| `treesit-parser-delete` | 2430 | Delete parser |
| `treesit-parser-list` | 2449 | List parsers for buffer (filter by language/tag) |
| `treesit-parser-buffer` | 2504 | Parser's buffer |
| `treesit-parser-language` | 2516 | Parser's language symbol |
| `treesit-parser-tag` | 2527 | Parser's tag |
| `treesit-parser-embed-level` | 2537 | Parser's embed level |
| `treesit-parser-set-embed-level` | 2553 | Set embed level |
| `treesit-parser-root-node` | 2585 | Root node (triggers parse) |
| `treesit-parse-string` | 2842 | Parse string, return root |
| `treesit-parser-changed-regions` | 2877 | Force re-parse, return affected ranges |
| `treesit-node-parser` | 2313 | Parser that produced a node |

#### Range Management (2)

| Function | Line | Description |
|---|---|---|
| `treesit-parser-set-included-ranges` | 2701 | Set ranges parser should parse |
| `treesit-parser-included-ranges` | 2759 | Get currently set ranges |

#### Notifiers (3)

| Function | Line | Description |
|---|---|---|
| `treesit-parser-notifiers` | 2776 | List of notifier functions |
| `treesit-parser-add-notifier` | 2792 | Add notifier |
| `treesit-parser-remove-notifier` | 2812 | Remove notifier |

#### Node Accessors (16)

| Function | Line | Description |
|---|---|---|
| `treesit-node-type` | 2945 | Node type string |
| `treesit-node-start` | 2963 | Start charpos |
| `treesit-node-end` | 2984 | End charpos |
| `treesit-node-string` | 3004 | S-expression of subtree |
| `treesit-node-parent` | 3021 | Parent node |
| `treesit-node-child` | 3052 | Nth child |
| `treesit-node-check` | 3097 | Check properties |
| `treesit-node-field-name-for-child` | 3153 | Field name of Nth child |
| `treesit-node-child-count` | 3192 | Child count |
| `treesit-node-child-by-field-name` | 3215 | Child by field name |
| `treesit-node-next-sibling` | 3239 | Next sibling |
| `treesit-node-prev-sibling` | 3265 | Previous sibling |
| `treesit-node-first-child-for-pos` | 3331 | First child beyond pos |
| `treesit-node-descendant-for-range` | 3369 | Smallest descendant for range |
| `treesit-node-eq` | 3419 | Structural equality |
| `treesit-node-match-p` | 5021 | Match against predicate |

#### Query (4)

| Function | Line | Description |
|---|---|---|
| `treesit-pattern-expand` | 3484 | Expand single pattern sexp → string |
| `treesit-query-expand` | 3540 | Expand query sexp → string |
| `treesit-query-compile` | 3848 | Compile query (lazy or eager) |
| `treesit-query-capture` | 3980 | Execute query, return captures |

#### Tree Traversal (4)

| Function | Line | Description |
|---|---|---|
| `treesit-search-subtree` | 4734 | DFS search in subtree |
| `treesit-search-forward` | 4804 | Linear forward/backward search |
| `treesit-induce-sparse-tree` | 4927 | Build sparse tree of matching nodes |
| `treesit-subtree-stat` | 5074 | `(max-depth max-width count)` stats |

#### Internal/Debug (3)

| Function | Line | Description |
|---|---|---|
| `treesit--linecol-at` | 5136 | Line/col at position (testing) |
| `treesit--linecol-cache-set` | 5155 | Set linecol cache (testing) |
| `treesit--linecol-cache` | 5176 | Return linecol cache (testing) |

#### Lisp Variables (6)

| Variable | Line | Description |
|---|---|---|
| `treesit-load-name-override-list` | 5306 | Override list for unconventional library/function names |
| `treesit-extra-load-path` | 5324 | Additional directories for grammar search |
| `treesit-thing-settings` | 5338 | `(LANGUAGE . ((THING PRED) ...))` alist |
| `treesit-language-remap-alist` | 5373 | Language symbol remapping (buffer-local) |
| `treesit-languages-require-line-column-tracking` | 5386 | Languages needing line/column tracking |
| `treesit-major-mode-remap-alist` | 5397 | Mode remapping for tree-sitter modes |

---

## 3. Lisp Layer: treesit.el

### 3.1 Font-Lock Engine

The font-lock engine replaces regex-based highlighting with tree-sitter query-based highlighting.

#### Defining Rules: `treesit-font-lock-rules` (treesit.el:1237)

A macro-like function that produces a list of `treesit-font-lock-setting` structs. Uses keyword-value pairs:

```elisp
(treesit-font-lock-rules
 :language 'python
 :feature 'comment
 :override t
 '((comment) @font-lock-comment-face)

 :language 'python
 :feature 'string
 '((string) @font-lock-string-face))
```

**Keywords:**
- `:language LANG` — language symbol (required)
- `:feature FEATURE` — feature symbol for grouping (required)
- `:override OVERRIDE` — `nil` (no override), `t` (always override), `append`, `prepend`, `keep`
- `:default-language LANG` — set default language for all following rules

#### Feature Levels: `treesit-font-lock-level` (treesit.el:1117)

4-level system controlling which features are active:

| Level | Typical Features |
|---|---|
| 1 | `comment`, `definition` |
| 2 | + `keyword`, `string`, `type` |
| 3 | + `assignment`, `constant`, `escape-sequence`, `number`, `property` |
| 4 | + `bracket`, `delimiter`, `operator`, `misc-punctuation` |

Modes declare features per level via `treesit-font-lock-feature-list` (a list of 4 lists of feature symbols).

#### Fontification Function: `treesit-font-lock-fontify-region` (treesit.el:1491)

Called by Emacs's font-lock machinery. For each active font-lock setting:
1. Runs the compiled tree-sitter query over the region.
2. For each captured node, applies the face according to override rules.
3. Handles multi-language buffers by running queries per language.

**Fast mode** (treesit.el ~line 1475): When the parse tree is deeper than `treesit--font-lock-fast-mode-max-depth` (default kind), the engine switches to a simpler fontification strategy to avoid performance issues with pathologically deep trees.

#### Notifier: `treesit--font-lock-notifier` (treesit.el:1636)

A parser notifier that marks changed ranges for re-fontification. Registered automatically by `treesit-major-mode-setup`. Calls `font-lock-flush` on each changed range plus a configurable extension (`treesit--font-lock-extend-region-function`).

### 3.2 Indentation Engine

#### Defining Rules: `treesit-simple-indent-rules` (buffer-local variable)

A list of `(LANGUAGE . RULES)`. Each rule is `(MATCHER ANCHOR OFFSET)`:

```elisp
(setq treesit-simple-indent-rules
  `((python
     ((parent-is "module") column-0 0)
     ((node-is "}") parent-bol 0)
     ((parent-is "block") parent-bol 4)
     (catch-all parent-bol 0))))
```

#### Preset Matchers (treesit.el, ~lines 2450-2800)

| Matcher | Description |
|---|---|
| `parent-is TYPE` | Parent node matches TYPE |
| `node-is TYPE` | Current node matches TYPE |
| `n-p-gp NODE PARENT GRANDPARENT` | Match all three levels |
| `query QUERY` | Run a tree-sitter query |
| `match NODE-TYPE PARENT-TYPE NODE-FIELD NODE-INDEX-MIN NODE-INDEX-MAX` | Multi-field match |
| `comment-end` | At end of comment |
| `catch-all` | Always matches (fallback) |
| `no-node` | Point is not in any node |
| `and MATCHERS...` | All must match |
| `or MATCHERS...` | Any must match |
| `not MATCHER` | Negation |

#### Preset Anchors (treesit.el, ~lines 2800-3000)

| Anchor | Description |
|---|---|
| `column-0` | Column 0 |
| `parent-bol` | Beginning of parent's line |
| `parent` | Parent's start position |
| `standalone-parent` | First non-trivial ancestor on its own line |
| `first-sibling` | First sibling's start |
| `prev-sibling` | Previous sibling's start |
| `prev-line` | Indentation of previous non-blank line |
| `point-min` | Position 1 |
| `comment-start` | Start of comment body |
| `comment-start-skip` | After comment-start regex match |
| `prev-adaptive-prefix` | Previous line's adaptive fill prefix end |
| `grand-parent` | Grandparent's start |

#### Indentation Function: `treesit-simple-indent` (treesit.el:3111)

1. Gets the node at point.
2. Walks up the tree trying each rule's MATCHER.
3. On first match: computes base position from ANCHOR, adds OFFSET.
4. Returns the target column.

#### Batch Optimization: `treesit-indent-region` (treesit.el:3244)

For region-wide indentation, processes in 400-line chunks. Creates a "context" object that caches node lookups and reuses them across lines for performance.

### 3.3 Thing System & Navigation

The "thing" system is a generic abstraction for navigable syntactic entities.

#### Configuration: `treesit-thing-settings` (buffer-local, C variable)

An alist of `(LANGUAGE . ((THING PRED) ...))`:

```elisp
(setq treesit-thing-settings
  `((python
     (sexp ,(rx (or "string" "number" "identifier" "list" "tuple" "dict")))
     (sentence "statement")
     (defun ,(rx (or "function_definition" "class_definition"))))))
```

**Predicate types:**
- **String/regexp** — matches against node type
- **Function** — `(lambda (node) ...)` returning non-nil for match
- **`(or PRED...)`** — any predicate matches
- **`(and PRED...)`** — all predicates match
- **`(not PRED)`** — negation

**Built-in thing names:** `defun`, `sexp`, `sentence`, `comment`, `text`, `list`

#### Navigation API (treesit.el, ~lines 3600-4200)

| Function | Description |
|---|---|
| `treesit-beginning-of-thing` | Move to beginning of thing |
| `treesit-end-of-thing` | Move to end of thing |
| `treesit-navigate-thing` | Core navigation with direction and side |
| `treesit-thing-at` | Return the thing node at point |
| `treesit-thing-prev` / `treesit-thing-next` | Previous/next thing |
| `treesit-top-level-thing` | Return the outermost thing at point |

#### Navigation Tactics: `treesit-defun-tactic` (treesit.el)

| Tactic | Description |
|---|---|
| `nested` | Navigate into nested defuns |
| `top-level` | Jump between top-level defuns only |
| `restricted` | Navigate within the current top-level |
| `parent-first` | Visit parent before children |

#### Standard Motion Commands Powered by Things

These are set up by `treesit-major-mode-setup`:

| Command | Thing Used | Setup Variable |
|---|---|---|
| `beginning-of-defun` / `end-of-defun` | `defun` | `treesit-defun-type-regexp` or thing settings |
| `forward-sexp` / `backward-sexp` | `sexp` | `treesit-sexp-type-regexp` or thing settings |
| `forward-sentence` / `backward-sentence` | `sentence` | `treesit-sentence-type-regexp` or thing settings |
| `forward-comment` | `comment` | thing settings |
| `up-list` / `down-list` | `list` | thing settings |
| `forward-list` / `backward-list` | `list` | thing settings |

### 3.4 Embedded Language Support

For buffers with multiple languages (e.g., HTML with embedded CSS and JavaScript).

#### Range Rules: `treesit-range-rules` (treesit.el:673)

Declarative rules specifying how to compute parser ranges:

```elisp
(treesit-range-rules
 :embed 'css
 :host 'html
 '((style_element (raw_text) @capture))

 :embed 'javascript
 :host 'html
 :local t
 '((script_element (raw_text) @capture)))
```

**Keywords:**
- `:embed LANG` — the embedded language
- `:host LANG` — the host language
- `:local t` — create local parsers (one per embedded region, tracked via overlays)
- `:offset '(START-OFFSET . END-OFFSET)` — adjust range boundaries

#### Local vs Global Parsers

- **Global parsers**: One parser per embedded language. All regions of that language in the buffer are passed as included ranges.
- **Local parsers**: One parser per embedded region. Tracked via overlays with `treesit-parser` property. Used when regions are independent (e.g., separate `<script>` tags).

#### Range Update: `treesit-update-ranges` (treesit.el:829)

Called before fontification. Runs the range queries against host parsers to find embedded regions, then updates each embedded parser's included ranges. Processes up to `treesit--range-update-max-levels` (default 4) embed levels.

### 3.5 Major Mode Setup

#### `treesit-major-mode-setup` (treesit.el:4625)

The one-stop function that modes call after setting their buffer-local variables. It configures:

1. **Font-lock**: Sets `font-lock-fontify-region-function` to `treesit-font-lock-fontify-region`, registers notifiers
2. **Indentation**: Sets `indent-line-function` to `treesit-indent`, `indent-region-function` to `treesit-indent-region`
3. **Navigation**: Sets `forward-sexp-function`, `beginning-of-defun-function`, `end-of-defun-function`, `transpose-sexps-function`
4. **Imenu**: Sets `imenu-create-index-function` if `treesit-simple-imenu-settings` is non-nil
5. **Outline**: Sets `outline-search-function` and `outline-level` if `treesit-outline-predicate` is set
6. **Which-function**: Integrates with `which-func-functions`
7. **Show-paren**: Sets `show-paren--categorize-function` if defun predicates are available
8. **Hideshow**: Configures `hs-special-modes-alist` based on node types

#### `treesit-derived-mode` / `define-derived-mode` Integration

Tree-sitter major modes typically inherit from a base mode using `define-derived-mode` and call `treesit-major-mode-setup` in the mode body.

#### `treesit-ready-p` (treesit.el:4507)

Check function that modes call to verify grammar availability:
```elisp
(when (treesit-ready-p 'python)
  (add-to-list 'auto-mode-alist '("\\.py\\'" . python-ts-mode)))
```

### 3.6 Imenu Integration

#### `treesit-simple-imenu-settings` (treesit.el:4294)

A list of `(CATEGORY NAME-REGEXP PRED NAME-FN)` tuples:

```elisp
(setq treesit-simple-imenu-settings
  `(("Function" "\\`function_definition\\'" nil nil)
    ("Class" "\\`class_definition\\'" nil nil)))
```

- `CATEGORY` — string for imenu submenu grouping (nil = flat)
- `NAME-REGEXP` — regexp matching node types to include
- `PRED` — optional predicate function for additional filtering
- `NAME-FN` — optional function to compute display name from node

#### `treesit-simple-imenu` (treesit.el:4373)

Creates imenu index by calling `treesit-induce-sparse-tree` (C-level tree traversal) for each setting, then building the imenu alist.

### 3.7 Other Subsystems

#### Outline Support (treesit.el:4444)

Configured via `treesit-outline-predicate`. Uses thing system to find defun/heading boundaries. `treesit-outline-search` implements `outline-search-function`.

#### Show-Paren Support (treesit.el:4480)

`treesit--show-paren-categorize` uses the thing system to detect defun boundaries for show-paren matching.

#### Hideshow Support (treesit.el, within `treesit-major-mode-setup`)

Configures `hs-special-modes-alist` based on defun node types.

#### `treesit-explore-mode` (treesit.el:5399)

Interactive minor mode that shows the parse tree in a side buffer. Updates on `post-command-hook` to highlight the node at point. Useful for grammar development and debugging.

#### `treesit-inspect-mode` (treesit.el:5300)

Lighter-weight minor mode that shows current node info in the mode line. Shows node type and field name at point.

#### Grammar Installation: `treesit-install-language-grammar` (treesit.el:5099)

Interactive command that downloads, compiles, and installs a tree-sitter grammar from a Git repository. Uses `treesit-language-source-alist` for repository URLs.

---

## 4. Key Design Decisions

These decisions are documented in treesit.c's commentary (lines 318-485):

1. **Syntax tree not exposed to Lisp.** Held internally in the parser. Tree updates happen at C level.

2. **Tree cursor not exposed to Lisp.** Performance gain deemed negligible vs. API complexity cost. Used only internally for parent-finding.

3. **Lazy parsing.** No re-parse on every keystroke. Changes recorded immediately; parsing deferred until results are needed.

4. **Explicit query compilation (AOT).** No transparent cache. `treesit-query-compile` gives users control over when compilation happens, avoiding cache-thrashing edge cases.

5. **Indirect buffer sharing.** Indirect buffers share the base buffer's parser list, but each parser points to its own buffer.

6. **Opt-in line/column tracking.** Most grammars don't need it. Enabled via `treesit-languages-require-line-column-tracking`. Once enabled, never disabled.

7. **Position translation.** Tree-sitter: 0-based byte offsets. Emacs: 1-based character positions. `visible_beg` bridges: `tree_offset = buffer_byte_pos - visible_beg`.

8. **4 GiB buffer limit.** Enforced by `treesit_check_buffer_size` (`UINT32_MAX`).

9. **Recursion limit.** `TREESIT_RECURSION_LIMIT = 1000` for tree traversal.

10. **Ranges stored as charpos.** Avoids bytepos ranges cutting into multibyte characters after edits.

11. **Lazy query compilation.** Default lazy because `(defvar ... (treesit-compile-query ...))` in mode files would otherwise force grammar loading at `require` time.

12. **One-char-at-a-time buffer reading.** `treesit_read_buffer` returns one character per call. Benchmarks showed negligible difference vs. chunks, and it avoids gap-buffer complications.

---

## 5. Integration Points in Other Files

| File | Integration |
|---|---|
| `src/alloc.c` | GC cleanup for parsers, nodes, queries |
| `src/insdel.c` | Calls `treesit_record_change` on buffer edits |
| `src/editfns.c` | Calls `treesit_record_change` on buffer edits |
| `src/casefiddle.c` | Calls `treesit_record_change` on case changes |
| `src/print.c` | Printing tree-sitter objects |
| `src/lisp.h` | Type definitions |
| `src/data.c` | Type predicates |
| `src/buffer.h` | `struct ts_linecol` definition |
| `src/treesit.h` | Struct definitions, function declarations |
| `lisp/cl-preloaded.el` | Type registrations |

---

## 6. How to Create a New Tree-Sitter Major Mode

Standard pattern used by all built-in tree-sitter modes:

```elisp
(define-derived-mode my-ts-mode prog-mode "MyLang"
  "Major mode for MyLang using tree-sitter."
  (when (treesit-ready-p 'mylang)
    ;; 1. Create the parser
    (treesit-parser-create 'mylang)

    ;; 2. Font-lock
    (setq-local treesit-font-lock-feature-list
                '((comment definition)
                  (keyword string type)
                  (constant number property)
                  (bracket delimiter operator)))
    (setq-local treesit-font-lock-settings
                (treesit-font-lock-rules
                 :language 'mylang
                 :feature 'comment
                 '((comment) @font-lock-comment-face)
                 :language 'mylang
                 :feature 'keyword
                 '((keyword) @font-lock-keyword-face)))

    ;; 3. Indentation
    (setq-local treesit-simple-indent-rules
                `((mylang
                   ((parent-is "block") parent-bol 2)
                   (catch-all parent-bol 0))))

    ;; 4. Navigation (things)
    (setq-local treesit-thing-settings
                `((mylang
                   (defun ,(rx (or "function_definition" "class_definition")))
                   (sexp ,(rx (or "string" "number" "identifier")))
                   (sentence "statement"))))

    ;; 5. Imenu
    (setq-local treesit-simple-imenu-settings
                `(("Function" "\\`function_definition\\'" nil nil)
                  ("Class" "\\`class_definition\\'" nil nil)))

    ;; 6. Activate everything
    (treesit-major-mode-setup)))
```

Key points:
- Always check `treesit-ready-p` before creating parsers
- Set buffer-local variables *before* calling `treesit-major-mode-setup`
- `treesit-major-mode-setup` reads all the buffer-local vars and wires everything up
- For multi-language modes, use `treesit-range-rules` and create multiple parsers
