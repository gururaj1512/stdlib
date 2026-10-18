<!--

@license Apache-2.0

Copyright (c) 2026 The Stdlib Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

-->

# JSON Patch Implementation Approach

> Detailed plan for implementing an RFC 6902-compliant JSON Patch package suite in stdlib, built on top of the JSON Pointer (`@stdlib/utils/json-pointer/*`) foundation.

---

## 1. What is JSON Patch (RFC 6902)

JSON Patch is a format for expressing a sequence of operations to apply to a JSON document. Instead of transmitting an entire updated document, only the list of changes (deltas) is sent. This is especially useful for incrementally updating large Vega visualization specs over SSE connections.

A JSON Patch document is a JSON array of operation objects. Each operation has an `op` field specifying the operation type, a `path` field (a JSON Pointer, RFC 6901), and operation-specific fields like `value` or `from`.

### The Six Operations

```json
[
  { "op": "add",     "path": "/foo/1",    "value": "bar" },
  { "op": "remove",  "path": "/foo/0" },
  { "op": "replace", "path": "/foo/0",    "value": "baz" },
  { "op": "move",    "from": "/foo/0",    "path": "/bar" },
  { "op": "copy",    "from": "/foo/0",    "path": "/baz" },
  { "op": "test",    "path": "/foo/0",    "value": "baz" }
]
```

### Operation Semantics (RFC 6902 §4)

#### 4.1 `add`
- **Required fields**: `op`, `path`, `value`
- If `path` targets an existing object member → the member's value is **replaced**
- If `path` targets a nonexistent object member → the member is **added**
- If `path` targets an array element → the value is **inserted before** that index (shifts elements right)
- If `path` uses `-` on an array → the value is **appended** to the array
- If `path` is `""` (root) → the entire document is replaced by `value`
- The target location's containing object/array MUST exist

#### 4.2 `remove`
- **Required fields**: `op`, `path`
- If `path` targets an object member → the member is **deleted**
- If `path` targets an array element → the element is **removed** and remaining elements shift left
- The target location MUST exist (error otherwise)
- `path` MUST NOT be `""` (cannot remove the root)

#### 4.3 `replace`
- **Required fields**: `op`, `path`, `value`
- Functionally equivalent to a `remove` followed by an `add` at the same location
- The target location MUST exist
- If `path` is `""` (root) → the entire document is replaced

#### 4.4 `move`
- **Required fields**: `op`, `from`, `path`
- Functionally equivalent to a `remove` at `from` followed by an `add` at `path`
- The `from` location MUST exist
- **Critical constraint**: `path` MUST NOT be a child of `from` (e.g., moving `/a/b` to `/a/b/c` is invalid — you'd be moving something into itself)

#### 4.5 `copy`
- **Required fields**: `op`, `from`, `path`
- Functionally equivalent to an `add` at `path` with the value from `from`
- The `from` location MUST exist
- The value MUST be **deep cloned** — modifying the copy must not affect the original

#### 4.6 `test`
- **Required fields**: `op`, `path`, `value`
- Verifies the value at `path` is **deeply equal** to `value`
- If the test fails → the entire patch application fails
- Used for conditional patching and optimistic concurrency control

### Atomicity (RFC 6902 §5)

> Operations are applied sequentially in the order they appear in the array. If any operation fails, the entire patch MUST have no effect — the document MUST be left in its original state.

---

## 2. How Other Libraries Handle It (Key Insights)

We reviewed 9 libraries in detail. Here's what matters for the patch implementation:

### Operation Dispatch — Hash-Table (jiff, Fast-JSON-Patch)

The cleaner libraries use a plain object to map operation names to handler functions:

```js
var handlers = {
    'add': doAdd,
    'remove': doRemove,
    'replace': doReplace,
    'move': doMove,
    'copy': doCopy,
    'test': doTest
};
handlers[ op.op ]( document, op );
```

This gives O(1) lookup, clean separation of concerns, and easy extensibility. jiff's version is the best fit for stdlib because it uses pure functions without `this` binding.

**What to take**: Hash-table dispatch with pure handler functions. No `this`, no classes.

### Structural Sharing — immutable-json-patch, jsonpatch.js

Instead of cloning the entire document before patching (O(n) time and space), structural sharing only shallow-copies the objects **along the path being modified**. Everything else keeps its original object references.

```
Original:  { a: { b: { c: 1, d: 2 }, e: 3 }, f: 4 }
Patch:     replace /a/b/c → 99

Result:    { a: { b: { c: 99, d: 2 }, e: 3 }, f: 4 }
                  ↑     ↑ ← new objects
                  ↑ ← new object
           ↑ ← new object
           original.a.b.d === result.a.b.d  ← shared reference (not cloned)
           original.f === result.f           ← shared reference
```

This is O(d) where d is the depth of the modified path (typically 3–5 levels), which is dramatically better than O(n) full cloning for large Vega specs.

**What to take**: Structural sharing as the default mode via `setIn` / `deleteIn` / `insertIn` helper primitives.

### Move Ancestor Validation — jiff, json-joy

When processing `move`, both jiff and json-joy check that the target `path` is not a descendant of the source `from`. The check is:

```js
if ( path.indexOf( from + '/' ) === 0 ) {
    throw new Error( 'Cannot move into own child' );
}
```

Without this, a move like `{ "from": "/a", "path": "/a/b/c" }` would remove `/a` and then try to add at `/a/b/c` which no longer exists.

**What to take**: Mandatory ancestor validation for `move`.

### Atomicity via Structural Sharing — immutable-json-patch

With structural sharing, atomicity is inherently guaranteed: the original document is never mutated. If any operation throws, we simply discard the partial result and the original is intact. No rollback logic needed.

For an in-place mutation mode (if we offer it), we'd need JSON8 Patch's approach: record inverse operations as we go, and if anything fails, apply the revert patch.

**What to take**: Structural sharing gives us atomicity for free. In-place mutation can be a future option.

### Value Cloning — jiff, RFC requirement

The RFC requires that `copy` produces an independent deep clone. Several libraries (jsonpatch-js, jsonpatch.js) get this wrong by inserting a direct reference. We must deep clone using `@stdlib/utils/copy`.

For `add` and `replace` operations, we should also clone the incoming `value` to prevent the caller's reference from being shared with the document.

**What to take**: Deep clone values on `copy` (mandatory), and on `add`/`replace` (defensive).

---

## 3. Architecture — Granular Subpackages

Following the same pattern we established with `@stdlib/utils/json-pointer/*`, each operation and utility is its own leaf package under the `@stdlib/utils/json-patch/` namespace.

### 3.1 Why Granular Subpackages

This is the same decision we made for json-pointer. Each package:
- Can be independently `require()`'d (tree-shaking friendly)
- Has its own test suite, benchmarks, docs, and README
- Follows stdlib's one-export-per-package convention
- Can be consumed without pulling in the entire patch suite

### 3.2 Full Namespace Layout

```
lib/node_modules/@stdlib/utils/json-patch/
├── lib/
│   └── index.js                      # Namespace aggregator — exports all subpackages
├── test/
│   └── test.js
├── docs/
│   ├── repl.txt
│   └── types/
│       ├── index.d.ts
│       └── test.ts
├── examples/
│   └── index.js
├── README.md
├── package.json
│
├── apply/                            # Core: apply a full patch array
│   ├── lib/
│   │   ├── index.js
│   │   ├── main.js                   # apply( document, operations ) → newDocument
│   │   ├── validate_operation.js     # Internal: validates op shape
│   │   └── handlers.js              # Internal: hash-table dispatch map
│   ├── test/
│   │   └── test.js
│   ├── benchmark/
│   │   └── benchmark.js
│   ├── docs/
│   │   ├── repl.txt
│   │   └── types/
│   │       ├── index.d.ts
│   │       └── test.ts
│   ├── examples/
│   │   └── index.js
│   ├── README.md
│   └── package.json
│
├── validate/                         # Validate a patch document structure
│   ├── lib/
│   │   ├── index.js
│   │   └── main.js                   # validate( operations ) → null | Error
│   ├── ...
│
├── ops/                              # All six RFC 6902 operations
│   ├── add/                          # Single operation: add
│   │   ├── lib/
│   │   │   ├── index.js
│   │   │   └── main.js               # add( document, path, value ) → newDocument
│   │   ├── test/
│   │   │   └── test.js
│   │   ├── benchmark/
│   │   │   └── benchmark.js
│   │   ├── docs/ ...
│   │   ├── examples/ ...
│   │   ├── README.md
│   │   └── package.json
│   │
│   ├── remove/                       # Single operation: remove
│   │   ├── lib/
│   │   │   ├── index.js
│   │   │   └── main.js               # remove( document, path ) → newDocument
│   │   ├── test/ ...
│   │   ├── ...
│   │
│   ├── replace/                      # Single operation: replace
│   │   ├── lib/
│   │   │   ├── index.js
│   │   │   └── main.js               # replace( document, path, value ) → newDocument
│   │   ├── ...
│   │
│   ├── move/                         # Single operation: move
│   │   ├── lib/
│   │   │   ├── index.js
│   │   │   └── main.js               # move( document, from, path ) → newDocument
│   │   ├── ...
│   │
│   ├── copy/                         # Single operation: copy
│   │   ├── lib/
│   │   │   ├── index.js
│   │   │   └── main.js               # copy( document, from, path ) → newDocument
│   │   ├── ...
│   │
│   └── test-op/                      # Single operation: test (named test-op to avoid collision)
│       ├── lib/
│       │   ├── index.js
│       │   └── main.js               # testOp( document, path, value ) → boolean
│       ├── ...
│
└── _lib/                             # Shared internal helpers (private)
    ├── set_in.js                     # Structural sharing: immutable set
    ├── delete_in.js                  # Structural sharing: immutable delete
    └── insert_in.js                  # Structural sharing: immutable array insert
```

### 3.3 Internal Shared Helpers (`_lib/`)

These are **private** helpers shared across the operation subpackages. They are not exported as public API.

#### `set_in.js` — Immutable Set via Structural Sharing

Given a document, a token array (from `json-pointer/parse`), and a value, returns a new document where only the objects along the path are shallow-cloned:

```js
// Pseudocode:
function setIn( document, tokens, value ) {
    if ( tokens.length === 0 ) {
        return value;  // root replacement
    }
    var root = shallowClone( document );
    var current = root;
    for ( i = 0; i < tokens.length - 1; i++ ) {
        child = shallowClone( current[ tokens[i] ] );
        current[ tokens[i] ] = child;
        current = child;
    }
    current[ tokens[ tokens.length - 1 ] ] = value;
    return root;
}
```

This is the core of structural sharing — only d objects are cloned (d = path depth).

#### `delete_in.js` — Immutable Delete via Structural Sharing

Same as `setIn` but deletes the final key. For arrays, uses splice on the cloned array. For objects, uses `delete` on the cloned object.

#### `insert_in.js` — Immutable Array Insert via Structural Sharing

Same path-cloning as `setIn`, but the final operation is `splice( index, 0, value )` to insert before the given index, shifting elements right.

---

## 4. Package-by-Package Specification

### 4.1 `@stdlib/utils/json-patch/validate` (top-level)

**Purpose**: Validate the structure of a JSON Patch operations array before applying it.

**Signature**: `validate( operations )` → `null | Error`

**Logic**:
1. `operations` must be an array
2. Each element must be a plain object
3. Each object must have a string `op` field with value in `{ 'add', 'remove', 'replace', 'move', 'copy', 'test' }`
4. Each object must have a string `path` field that is a valid JSON Pointer
5. `add`, `replace`, `test` must have a `value` field (any type)
6. `move`, `copy` must have a string `from` field that is a valid JSON Pointer
7. Returns `null` if valid, or an `Error` describing the first validation failure

**Why standalone**: Allows callers to validate untrusted input before execution, separating validation from application (unlike most libraries that combine them).

### 4.2 `@stdlib/utils/json-patch/ops/add`

**Purpose**: Apply a single `add` operation immutably.

**Signature**: `add( document, path, value )` → `newDocument`

**Logic**:
1. Parse `path` into tokens via `@stdlib/utils/json-pointer/parse`
2. If tokens is empty (root pointer) → return deep clone of `value`
3. Resolve all tokens except the last against the document to find the target container
4. Clone the incoming `value` via `@stdlib/utils/copy`
5. If target is an **object**:
   - Use `setIn` to immutably set `target[lastToken] = value`
   - (Adds if not present, replaces if present — per RFC)
6. If target is an **array**:
   - If lastToken is `'-'` → use `insertIn` with index = `array.length` (append)
   - Else → validate as numeric index, use `insertIn` at that index (inserts before, shifts right)
   - Index must be `0 <= index <= array.length` (not strictly less than length — inserting at length is appending)
7. Return the new document (original untouched)

**Errors**:
- Path references a nonexistent intermediate container → `Error`
- Invalid array index → `Error`
- Path traverses into a primitive → `TypeError`

### 4.3 `@stdlib/utils/json-patch/ops/remove`

**Purpose**: Apply a single `remove` operation immutably.

**Signature**: `remove( document, path )` → `newDocument`

**Logic**:
1. Parse `path` into tokens
2. Root pointer (`""`) → error (cannot remove root)
3. Resolve the pointer — target must exist
4. If target is an **object** → use `deleteIn` to immutably remove the property
5. If target is an **array** → use `deleteIn` which clones the array and splices out the element
6. Return the new document

**Errors**:
- `path` is `""` → `Error` (cannot remove root)
- Target does not exist → `Error`

### 4.4 `@stdlib/utils/json-patch/ops/replace`

**Purpose**: Apply a single `replace` operation immutably.

**Signature**: `replace( document, path, value )` → `newDocument`

**Logic**:
1. Parse `path` into tokens
2. If tokens is empty (root) → return deep clone of `value`
3. Resolve the pointer — target MUST exist (unlike `add`, `replace` requires it)
4. Clone the value
5. Use `setIn` to immutably replace the value at that location
6. Return the new document

**Errors**:
- Target does not exist → `Error` (use `add` for new properties)

### 4.5 `@stdlib/utils/json-patch/ops/move`

**Purpose**: Apply a single `move` operation immutably.

**Signature**: `move( document, from, path )` → `newDocument`

**Logic**:
1. Parse both `from` and `path` into tokens
2. **Ancestor check**: if `path` starts with `from + '/'` → throw `Error` (cannot move into own child)
3. Resolve `from` → get the value
4. Apply `remove( document, from )` → intermediate document
5. Apply `add( intermediate, path, value )` → final document
6. Return the final document

**Errors**:
- `from` does not exist → `Error`
- `path` is a descendant of `from` → `Error`

### 4.6 `@stdlib/utils/json-patch/ops/copy`

**Purpose**: Apply a single `copy` operation immutably.

**Signature**: `copy( document, from, path )` → `newDocument`

**Logic**:
1. Resolve `from` → get the value
2. Deep clone the value via `@stdlib/utils/copy` (RFC requirement)
3. Apply `add( document, path, clonedValue )` → new document
4. Return the new document

**Errors**:
- `from` does not exist → `Error`

### 4.7 `@stdlib/utils/json-patch/ops/test-op`

**Purpose**: Test whether a value at a path deeply equals a given value.

**Signature**: `testOp( document, path, value )` → `boolean`

**Logic**:
1. Resolve `path` → get the current value
2. Compare current value to `value` using `@stdlib/assert/deep-equal`
3. Return `true` if equal, `false` otherwise

**Note**: Named `test-op` (package) and `testOp` (function) to avoid collision with JavaScript's `test` keyword / test infrastructure conventions.

### 4.8 `@stdlib/utils/json-patch/apply` (top-level)

**Purpose**: Apply an entire patch (array of operations) to a document atomically.

**Signature**: `apply( document, operations )` → `newDocument`

**Logic**:
1. Validate `operations` via `@stdlib/utils/json-patch/validate` (sibling package)
2. Set `current = document`
3. For each operation in order:
   - Look up the handler in the dispatch table
   - Execute: `current = handler( current, op )`
4. If any operation throws → the original `document` is untouched (structural sharing guarantees this — each step produces a new document, so on error we just stop and let the original remain)
5. Return `current` (the fully patched document)

**Dispatch Table**:

```js
var add = require( '@stdlib/utils/json-patch/ops/add' );
var remove = require( '@stdlib/utils/json-patch/ops/remove' );
var replace = require( '@stdlib/utils/json-patch/ops/replace' );
var move = require( '@stdlib/utils/json-patch/ops/move' );
var copy = require( '@stdlib/utils/json-patch/ops/copy' );
var testOp = require( '@stdlib/utils/json-patch/ops/test-op' );

var handlers = {
    'add': function handleAdd( doc, op ) {
        return add( doc, op.path, op.value );
    },
    'remove': function handleRemove( doc, op ) {
        return remove( doc, op.path );
    },
    'replace': function handleReplace( doc, op ) {
        return replace( doc, op.path, op.value );
    },
    'move': function handleMove( doc, op ) {
        return move( doc, op.from, op.path );
    },
    'copy': function handleCopy( doc, op ) {
        return copy( doc, op.from, op.path );
    },
    'test': function handleTest( doc, op ) {
        if ( !testOp( doc, op.path, op.value ) ) {
            throw new Error( 'Test operation failed...' );
        }
        return doc;
    }
};
```

### 4.9 `@stdlib/utils/json-patch` (Namespace Aggregator)

**Purpose**: Top-level namespace that re-exports all subpackages.

**`lib/index.js`**:

```js
var setReadOnly = require( '@stdlib/utils/define-read-only-property' );

var ns = {};

setReadOnly( ns, 'apply', require( '@stdlib/utils/json-patch/apply' ) );
setReadOnly( ns, 'validate', require( '@stdlib/utils/json-patch/validate' ) );

// Operations (under ops/):
setReadOnly( ns, 'add', require( '@stdlib/utils/json-patch/ops/add' ) );
setReadOnly( ns, 'copy', require( '@stdlib/utils/json-patch/ops/copy' ) );
setReadOnly( ns, 'move', require( '@stdlib/utils/json-patch/ops/move' ) );
setReadOnly( ns, 'remove', require( '@stdlib/utils/json-patch/ops/remove' ) );
setReadOnly( ns, 'replace', require( '@stdlib/utils/json-patch/ops/replace' ) );
setReadOnly( ns, 'test', require( '@stdlib/utils/json-patch/ops/test-op' ) );

module.exports = ns;
```

---

## 5. Structural Sharing — Deep Dive

### 5.1 How `setIn` Works

The key insight: to immutably update a value at depth d, we only need to create d new objects (one per level of the path). Everything else is shared by reference.

```
Before:
doc = {
    a: {
        b: {
            c: 1,       ← we want to change this to 99
            d: 2
        },
        e: 3
    },
    f: 4
}

setIn( doc, ['a', 'b', 'c'], 99 )

After — what happens internally:
1. Clone doc       → newDoc = { a: ..., f: 4 }         // f is SHARED
2. Clone doc.a     → newA = { b: ..., e: 3 }            // e is SHARED
3. Clone doc.a.b   → newB = { c: 1, d: 2 }              // d is SHARED
4. Set newB.c = 99
5. Set newA.b = newB
6. Set newDoc.a = newA
7. Return newDoc

Only 3 new objects created (newDoc, newA, newB).
Original doc is completely untouched.
doc.f === newDoc.f  // true — same reference
doc.a.e === newDoc.a.e  // true — same reference
```

### 5.2 How `deleteIn` Works

Same path-cloning, but the final step is either:
- `delete newObj[lastKey]` for objects
- `newArr.splice( lastIndex, 1 )` for arrays (on the cloned array)

### 5.3 How `insertIn` Works (for `add` on arrays)

Same path-cloning, but the final step is:
- `newArr.splice( index, 0, value )` to insert before the given index

### 5.4 Shallow Clone Helper

For objects: `Object.assign( {}, obj )` or manual key copy
For arrays: `obj.slice()` or `Array.prototype.slice.call( obj )`

We should use a dedicated stdlib-compatible shallow clone that handles both. Check if `@stdlib/utils/copy` with `level = 1` satisfies this, or write a small internal helper.

---

## 6. Dependency Graph

```
                    @stdlib/utils/json-patch/apply
                    ┌────────────┬──────────────┐
                    │            │              │
                    ▼            ▼              ▼
          json-patch/ops/add  json-patch/ops/remove  json-patch/ops/replace  ...
                    │            │              │
                    ▼            ▼              ▼
                    └────────────┴──────────────┘
                                 │
                    ┌────────────┼──────────────┐
                    │            │              │
                    ▼            ▼              ▼
            _lib/set_in   _lib/delete_in  _lib/insert_in
                    │            │              │
                    └────────────┴──────────────┘
                                 │
                                 ▼
                    @stdlib/utils/json-pointer/parse
                    @stdlib/utils/json-pointer/resolve

                    ┌─────────────────────────────────────────┐
                    │           stdlib dependencies           │
                    ├─────────────────────────────────────────┤
                    │ @stdlib/utils/copy                      │
                    │ @stdlib/assert/deep-equal               │
                    │ @stdlib/assert/is-array                 │
                    │ @stdlib/assert/is-plain-object          │
                    │ @stdlib/assert/has-own-property         │
                    │ @stdlib/string/format                   │
                    │ @stdlib/utils/define-read-only-property │
                    └─────────────────────────────────────────┘
```

---

## 7. stdlib Dependencies

| Package | Usage |
| ------- | ----- |
| `@stdlib/utils/json-pointer/parse` | Parse JSON Pointer strings into token arrays |
| `@stdlib/utils/json-pointer/resolve` | Resolve pointers to find target + key + value |
| `@stdlib/utils/copy` | Deep clone values for `copy` op and defensive `add`/`replace` cloning |
| `@stdlib/assert/deep-equal` | `test` operation — deep structural comparison |
| `@stdlib/assert/is-array` | Type checking during structural sharing helpers |
| `@stdlib/assert/is-plain-object` | Type checking during structural sharing helpers |
| `@stdlib/assert/is-string` | Input validation |
| `@stdlib/assert/has-own-property` | Property existence checks |
| `@stdlib/string/format` | Error message formatting |
| `@stdlib/utils/define-read-only-property` | Namespace aggregator wiring |

---

## 8. Edge Cases & Error Handling

### 8.1 Per-Operation Edge Cases

| Case | Operation | Behavior |
| ---- | --------- | -------- |
| Add to root (`""`) | `add` | Replace entire document with value |
| Add to existing object key | `add` | Replace the value (not an error) |
| Add to nonexistent intermediate | `add` | Error — container must exist |
| Add at array index = length | `add` | Append (equivalent to `-`) |
| Add at array index > length | `add` | Error — out of bounds |
| Remove root | `remove` | Error — cannot remove root |
| Remove nonexistent path | `remove` | Error |
| Replace nonexistent path | `replace` | Error (unlike `add`) |
| Replace root | `replace` | Replace entire document |
| Move into own child | `move` | Error — ancestor check |
| Move root | `move` | Valid if `from` is `""` and `path` is valid |
| Copy without deep clone | `copy` | Bug — must deep clone |
| Test with `NaN` | `test` | `NaN === NaN` should be true per deep-equal |
| Test with nested objects | `test` | Full structural comparison required |
| Empty patch array | `apply` | Return document unchanged |
| Invalid `op` string | `apply` | Error (validation catches this) |
| Missing `value` on `add` | `apply` | Error (validation catches this) |
| Prototype pollution in path | all | Blocked at json-pointer/resolve level |

### 8.2 Array Index Rules for `add`

The `add` operation has slightly different array index rules than `resolve`:
- Valid range for `add`: `0 <= index <= array.length` (can equal length — that's appending)
- Valid range for `remove`/`replace`/`test`: `0 <= index < array.length` (must be existing element)
- The `-` token is only valid for `add` (append)

This means `add` needs its own index bounds check, not the one from `resolve` (which requires `index < length`).

### 8.3 Atomicity Guarantees

With structural sharing, atomicity is automatic:
- Each operation produces a completely new document
- If operation N fails, the result of operation N-1 is simply discarded
- The original document's object references are never modified
- No rollback logic needed

---

## 9. Testing Strategy

### 9.1 Per-Package Tests

Each subpackage has its own `test/test.js` covering:
- Happy path with various document shapes
- Error cases (invalid input, missing paths, out of bounds)
- Type validation (wrong argument types)
- Structural sharing verification (original document untouched, shared references intact)

### 9.2 Community Test Suite

The [json-patch-test-suite](https://github.com/json-patch/json-patch-tests) contains ~80 tests that major libraries (Fast-JSON-Patch, jiff) run against. We should create a dedicated test file that runs our `apply` package against this suite.

### 9.3 Structural Sharing Verification

For each operation, verify:
1. Original document is **not** mutated (`deepEqual( original, beforeCopy )`)
2. Unmodified subtrees share references (`original.unchanged === result.unchanged`)
3. Modified path has new references (`original.modified !== result.modified`)

### 9.4 Security Tests

- Prototype pollution paths rejected at all levels
- All paths validated before execution
- No accidental reference sharing between patch values and document

---

## 10. Order of Implementation

Build bottom-up, testing each layer before moving to the next:

### Phase 1: Structural Sharing Primitives

1. **`_lib/set_in.js`** — Core immutable set. Path-cloning with structural sharing.
2. **`_lib/delete_in.js`** — Immutable delete for objects and arrays.
3. **`_lib/insert_in.js`** — Immutable array insert (splice with shift-right).

### Phase 2: Individual Operation Packages

Build and test one at a time, in dependency order:

4. **`ops/add/`** — Most complex single operation. Handles objects, arrays, root, `-` token.
5. **`ops/remove/`** — Simpler. Object delete + array splice.
6. **`ops/replace/`** — Thin: resolve + setIn.
7. **`ops/test-op/`** — Pure read: resolve + deep-equal.
8. **`ops/move/`** — Composed: remove + add + ancestor check.
9. **`ops/copy/`** — Composed: resolve + deep clone + add.

### Phase 3: Orchestration

10. **`validate/`** — Standalone operation structure validator.
11. **`apply/`** — Full patch applicator with dispatch table + validation.

### Phase 4: Namespace & Integration

12. **`@stdlib/utils/json-patch/` (namespace)** — Wire all subpackages into the aggregator.
13. **Community test suite integration** — Run against json-patch-test-suite.

---

## 11. API Usage Examples

### Apply a Full Patch

```js
var apply = require( '@stdlib/utils/json-patch/apply' );  // top-level

var doc = {
    'title': 'My Chart',
    'data': {
        'values': [ 1, 2, 3 ]
    }
};

var ops = [
    { 'op': 'replace', 'path': '/title', 'value': 'Updated Chart' },
    { 'op': 'add', 'path': '/data/values/-', 'value': 4 },
    { 'op': 'remove', 'path': '/data/values/0' }
];

var result = apply( doc, ops );
// result.title → 'Updated Chart'
// result.data.values → [ 2, 3, 4 ]
// doc.title → 'My Chart'  (unchanged)
// doc.data.values → [ 1, 2, 3 ]  (unchanged)
```

### Use Individual Operations

```js
var add = require( '@stdlib/utils/json-patch/ops/add' );
var remove = require( '@stdlib/utils/json-patch/ops/remove' );
var testOp = require( '@stdlib/utils/json-patch/ops/test-op' );

var doc = { 'foo': [ 1, 2, 3 ] };

// Add to array:
var doc2 = add( doc, '/foo/-', 4 );
// doc2.foo → [ 1, 2, 3, 4 ]

// Remove from array:
var doc3 = remove( doc2, '/foo/0' );
// doc3.foo → [ 2, 3, 4 ]

// Test a value:
var bool = testOp( doc3, '/foo/0', 2 );
// bool → true
```

### Validate Before Applying

```js
var validate = require( '@stdlib/utils/json-patch/validate' );
var apply = require( '@stdlib/utils/json-patch/apply' );

var ops = [
    { 'op': 'add', 'path': '/foo' }  // missing 'value'!
];

var err = validate( ops );
if ( err ) {
    console.error( 'Invalid patch:', err.message );
} else {
    var result = apply( doc, ops );
}
```

---

## 12. Open Questions

These should be decided before or during implementation:

1. **`test-op` naming**: The operation is called `test` in the RFC, but `test` collides with testing conventions. The package is `test-op`, the export function is `testOp`. Does the namespace aggregator expose it as `ns.test` or `ns.testOp`?

2. **`_lib/` sharing mechanism**: The structural sharing helpers in `_lib/` need to be importable by the operation packages under `ops/`. Options:
   - (a) Relative path requires from ops: `require( '../../_lib/set_in.js' )`
   - (b) Inline in each package (duplication, but zero coupling)
   - (c) Publish as a private stdlib package like `@stdlib/utils/json-patch/_lib`

3. **Value cloning on `add`/`replace`**: Should we defensively clone values passed to `add` and `replace`? This prevents the caller's reference from being shared with the document, but adds overhead. The RFC doesn't require it for these operations (only `copy`). Could offer an `options.clone` flag.

4. **In-place mutation mode**: Should we offer a `mutate: true` option for maximum performance in trusted contexts? If so, this would bypass structural sharing and mutate directly. This is a future concern — start with immutable-only.

5. **Root `add` semantics**: When `add` targets root (`""`), do we deep clone the value or return it directly? The RFC says the result is the value, but for consistency with our defensive cloning, we should probably clone it.

---

## 13. Reference: `json-pointer/parse` Branch Structure

Each json-patch subpackage should follow the **exact file structure** established on the `json-pointer/parse` git branch. This is our canonical template.

**Branch**: `json-pointer/parse` (local + `origin/json-pointer/parse`)

### Per-Package File Layout

```
<package-name>/
├── lib/
│   ├── index.js       # Module docblock + re-exports main.js
│   └── main.js        # Core implementation with JSDoc
├── test/
│   └── test.js        # tape tests with values/expected arrays
├── benchmark/
│   └── benchmark.js   # @stdlib/bench benchmarks
├── docs/
│   ├── repl.txt       # REPL help text ({{alias}} format)
│   └── types/
│       ├── index.d.ts # TypeScript declarations
│       └── test.ts    # TypeScript compile-time tests
├── examples/
│   └── index.js       # Usage examples with console.log
├── README.md          # stdlib-style markdown with <section> tags
└── package.json       # stdlib package.json format
```

### Key Conventions from the Branch

**`lib/index.js`** — Module-level docblock with `@module` tag and usage examples, then re-exports `./main.js`:
```js
/**
* @module @stdlib/utils/json-patch/ops/add
*
* @example
* var add = require( '@stdlib/utils/json-patch/ops/add' );
* ...
*/
var main = require( './main.js' );
module.exports = main;
```

**`lib/main.js`** — Section comments (`// MODULES //`, `// MAIN //`, `// EXPORTS //`), full JSDoc with `@param`, `@throws`, `@returns`, `@example`. Uses stdlib imports (`@stdlib/assert/is-string`, `@stdlib/string/format`, etc).

**`test/test.js`** — Uses `tape`. Test cases use `values` and `expected` arrays iterated with a `for` loop. All assertion messages are `'returns expected value'`. Error tests use `badValue()` closure pattern.

**`benchmark/benchmark.js`** — Uses `@stdlib/bench`. Multiple named benchmarks (`format( '%s:shallow', pkg )`, `format( '%s:deep', pkg )`, etc).

**`package.json`** — `name`, `version: "0.0.0"`, `description`, `license: "Apache-2.0"`, `author`, `contributors`, `main: "./lib"`, `directories`, `types: "./docs/types"`, `engines: { node: ">=0.10.0" }`, `os` array, `keywords`.

**`README.md`** — `<!--license-->` header, `<section class="usage">`, `<section class="notes">`, `<section class="examples">`, `<section class="references">`, `<section class="links">`.

**`docs/repl.txt`** — Uses `{{alias}}` placeholder, documents parameters, return value, and examples.

**`docs/types/index.d.ts`** — TypeScript `declare function` with JSDoc `@example` blocks.

**`docs/types/test.ts`** — `$ExpectType` and `$ExpectError` assertions.

---

## 14. Related Documents

- [JSON Patch & Pointer Research Index](./README.md) — Cross-library comparison and evaluation
- [JSON Pointer Implementation Approach](./json-pointer-approach.md) — RFC 6901 engineering spec
- [Proposal](./proposal.md) — High-level proposal and rationale
- [Individual Library Reviews](./fast-json-patch.md) — Detailed per-library analysis

