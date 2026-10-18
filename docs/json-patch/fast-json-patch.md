<!--

@license Apache-2.0

Copyright (c) 2025 The Stdlib Authors.

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

# Fast-JSON-Patch — Technical Review

> Library: `fast-json-patch` v3.1.1
> Repository: [Starcounter-Jack/JSON-Patch](https://github.com/Starcounter-Jack/JSON-Patch)
> Location in stdlib tree: `libraries/JSON-Patch`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | JSON-Patch (npm: `fast-json-patch`) |
| **Primary language** | TypeScript (compiled to JS) |
| **Module system** | CommonJS + ES modules (dual output) |
| **Approximate implementation size** | ~3 source files: `core.ts` (~522 lines), `helpers.ts` (~165 lines), `duplex.ts` (~213 lines). Total ~900 lines. |
| **License** | MIT |

### Public API

- `applyOperation(document, operation, [validate], [mutate], [banProto], [index])` — apply a single operation
- `applyPatch(document, patch, [validate], [mutate], [banProto])` — apply an array of operations
- `applyReducer(document, operation, index)` — reducer-compatible function
- `validate(sequence, [document], [validator])` — validate a patch sequence
- `validator(operation, index, [document], [existingPath])` — default validator
- `getValueByPointer(document, pointer)` — retrieve value by JSON Pointer
- `deepClone(obj)` — deep clone via `JSON.parse(JSON.stringify(obj))`
- `observe(obj, [callback])` — observe object changes (duplex module)
- `unobserve(root, observer)` — detach observer
- `generate(observer, [invertible])` — generate patches from observed changes
- `compare(tree1, tree2, [invertible])` — generate diff patches between two objects

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | Object and array |
| `remove` | ✅ | Object and array |
| `replace` | ✅ | Object and array |
| `move` | ✅ | Delegates to `remove` + `add` |
| `copy` | ✅ | Deep clones copied value |
| `test` | ✅ | Uses custom deep equality |
| `_get` | ✅ | Non-standard extension (private) |

### RFC Compliance

- **RFC 6902 (JSON Patch)**: Yes
- **RFC 6901 (JSON Pointer)**: Yes
- **Extensions beyond RFC 6902**: `_get` operation (internal); `observe`/`generate`/`compare` (duplex diff generation)

### Runtime/Browser Support

- Node.js and browser (includes webpack config)
- Tested via SauceLabs (cross-browser)
- No polyfill requirements observed beyond `Object.keys` fallback

### Build Requirements

- TypeScript ~3.5
- Webpack 4 (for browser bundle)

### External Dependencies

- **Runtime**: **None** (zero dependencies in `dependencies` field)
- **Development**: `benchmark`, `jasmine`, `typescript`, `webpack`, etc.

---

## 2. JSON Patch Specification Compliance

### JSON Pointer (RFC 6901)

| Behavior | Status | Evidence |
| -------- | ------ | -------- |
| Pointer escaping `~0` → `~`, `~1` → `/` | ✅ RFC-compliant | `unescapePathComponent`: replaces `~1` → `/` then `~0` → `~` (correct order) |
| Escape order (unescape `~1` before `~0`) | ✅ Correct | `helpers.ts:76` |
| Empty pointer (root reference) | ✅ Handled | Explicit `if (operation.path === "")` block handles root operations |
| Path must start with `/` (if non-empty) | ✅ Validated | `validator`: checks `path.indexOf('/') !== 0 && path.length > 0` |
| Array indexing | ✅ | `isInteger` check; numeric conversion via `~~key` |
| `-` array append | ✅ | Converts to `obj.length` for arrays |

### Operation Analysis

#### `add`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object property addition | ✅ RFC-required | `obj[key] = this.value` |
| Array element insertion | ✅ RFC-required | `arr.splice(i, 0, this.value)` |
| Root replacement | ✅ RFC-required | Returns `{ newDocument: operation.value }` |
| `-` append semantics | ✅ RFC-required | `-` → `arr.length` |
| Index > length validation | ✅ RFC-required | Throws `OPERATION_VALUE_OUT_OF_BOUNDS` (only when validation enabled) |
| Non-integer array key as property | ⚠️ Implementation-specific | Falls through to `arr[i] = this.value` when key is not integer — allows non-index properties on arrays |

#### `remove`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object property removal | ✅ RFC-required | `delete obj[key]` |
| Array element removal | ✅ RFC-required | `arr.splice(i, 1)` |
| Root removal | ⚠️ Ambiguous | Sets `newDocument = null` — RFC 6902 doesn't clearly specify root removal semantics |
| Missing path validation | ✅ RFC-required | Validated via `OPERATION_PATH_UNRESOLVABLE` (when validation enabled) |

#### `replace`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object value replacement | ✅ RFC-required | `obj[key] = this.value` |
| Array value replacement | ✅ RFC-required | `arr[i] = this.value` |
| Root replacement | ✅ RFC-required | Returns `{ newDocument: operation.value, removed: document }` |
| Existing value required | ✅ RFC-required | Validated via `OPERATION_PATH_UNRESOLVABLE` (when validation enabled) |

#### `move`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Decomposition to `remove` + `add` | ✅ RFC-required | Calls `applyOperation(document, { op: "remove", path: this.from })` then `applyOperation(document, { op: "add", ... })` |
| `from` must exist | ✅ RFC-required | Validated via `OPERATION_FROM_UNRESOLVABLE` |
| Captures removed value at destination | ✅ Implementation-specific | Deep clones existing destination value before overwrite — enables returning removed value |
| Array index edge cases | ⚠️ Potential issue | Moving within same array can cause index shift since `remove` happens before `add`. This is actually RFC-correct behavior (sequential application). |

#### `copy`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Deep copy of value | ✅ RFC-correct | `_deepClone(valueToCopy)` before `add` — prevents reference aliasing (addresses issue #177) |
| `from` validation | ✅ RFC-required | Validated via `OPERATION_FROM_UNRESOLVABLE` |

#### `test`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Deep equality check | ✅ RFC-required | Uses `_areEquals` function |
| Throws on failure | ✅ RFC-required | Throws `TEST_OPERATION_FAILED` |
| Object/array comparison | ✅ | Key-count check, recursive descent |
| NaN equality | ✅ | `a !== a && b !== b` handles NaN |

### Atomicity

- **Observed**: **Not atomic**. Operations are applied sequentially. If operation N fails, operations 0..N-1 have already mutated the document.
- The RFC 6902 Section 5 says: "If a normative requirement is violated by a JSON Patch document, or if an operation is not successful, evaluation of the JSON Patch document SHOULD terminate and application of the entire patch document SHALL NOT be considered successful." The library does stop on failure but does NOT roll back already-applied operations.
- When `mutateDocument=false`, the original is cloned first, so failure doesn't corrupt the original — but the clone is still partially patched.

---

## 3. Architecture Analysis

```
Fast-JSON-Patch
├── core.ts — Patch application engine
│   ├── applyOperation() — Single operation application
│   │   ├── Root operations (path === "")
│   │   │   └── Direct switch on op type
│   │   └── Non-root operations
│   │       ├── JSON Pointer path splitting: path.split('/')
│   │       ├── Path traversal loop (while true)
│   │       │   ├── Unescape path components
│   │       │   ├── Prototype pollution check
│   │       │   ├── Validation (optional)
│   │       │   ├── Array/Object branching
│   │       │   └── Dispatch to arrOps/objOps
│   │       └── Target resolution: obj = obj[key]
│   ├── applyPatch() — Batch operation application
│   │   ├── Optional deep clone
│   │   └── Sequential applyOperation calls
│   ├── applyReducer() — Array.reduce-compatible wrapper
│   ├── validator() — Operation validation
│   │   └── Structural validation (type checks, required fields)
│   ├── validate() — Batch validation (clones document first)
│   ├── getValueByPointer() — Value retrieval via _get op
│   └── _areEquals() — Deep equality (from fast-deep-equal)
│
├── helpers.ts — Utility functions
│   ├── _deepClone() — JSON.parse(JSON.stringify())
│   ├── isInteger() — Character-code-based integer check
│   ├── escapePathComponent() — ~0/~1 encoding
│   ├── unescapePathComponent() — ~0/~1 decoding
│   ├── hasOwnProperty() — Safe hasOwnProperty wrapper
│   ├── _objectKeys() — Object.keys with array fallback
│   ├── hasUndefined() — Recursive undefined detection
│   └── PatchError — Custom error class
│
└── duplex.ts — Diff generation (observe/compare)
    ├── observe() — Set up mutation observer (DOM event-based dirty checking)
    ├── unobserve() — Tear down observer
    ├── generate() — Generate patch from observed changes
    ├── compare() — Generate diff between two objects
    └── _generate() — Recursive diff algorithm
```

### Key Design Decisions

1. **Operation dispatch via hash objects** (`objOps`/`arrOps`): Operations are dispatched as `objOps[operation.op].call(operation, obj, key, document)`. The operation object is used as `this` context to access `this.value`, `this.from`.

2. **Separate object/array operation tables**: Array operations use `splice` for insertion/removal; object operations use assignment/`delete`. Move, copy, test, and _get are shared.

3. **Path traversal is inline**: JSON Pointer parsing and traversal happen in the main `applyOperation` function rather than being separated into a reusable pointer resolution function.

4. **Validation is optional**: By default, no validation occurs. Validation is triggered by passing `true` or a custom validator function.

---

## 4. Algorithm Analysis

### JSON Pointer

- **Parsing**: Simple `path.split('/')` — O(n) where n is path length.
- **Tokenization**: Split by `/`, skip empty first element (index 1).
- **Escaping**: `replace(/~/g, '~0').replace(/\//g, '~1')` — correct per RFC 6901.
- **Unescaping**: `replace(/~1/g, '/').replace(/~0/g, '~')` — order is correct (must replace `~1` before `~0`).
- **Path traversal**: Linear walk via `obj = obj[key]`, O(d) where d is pointer depth.
- **Array handling**: `isInteger` check via character codes (O(k) per component), `~~key` for numeric coercion, `-` → `obj.length`.

### `add`

- **Object**: `obj[key] = this.value` — O(1).
- **Array**: `arr.splice(i, 0, this.value)` — O(n) due to element shifting.
- **Root**: Returns new document directly — O(1).

### `remove`

- **Object**: `delete obj[key]` — O(1).
- **Array**: `arr.splice(i, 1)` — O(n) due to element shifting.
- **Root**: Returns `{ newDocument: null }` — O(1).

### `replace`

- **Object/Array**: Direct assignment `obj[key] = this.value` — O(1).
- **Root**: Returns new value — O(1).

### `move`

- Decomposes to `applyOperation(remove)` + `applyOperation(add)`.
- Captures existing value at destination via `getValueByPointer` + `_deepClone` before removal.
- **Complexity**: Two full path traversals + potential array splice operations.

### `copy`

- `getValueByPointer` to find source value, `_deepClone`, then `applyOperation(add)`.
- **Deep clone**: `JSON.parse(JSON.stringify(obj))` — prevents reference aliasing.
- **Complexity**: Path traversal + O(v) clone where v is value size.

### `test`

- `_areEquals` — recursive deep comparison.
- Arrays: Length check + element-wise recursion.
- Objects: Key count check + `hasOwnProperty` + key-wise recursion.
- NaN: `a !== a && b !== b` special case.
- **Complexity**: O(v) where v is value size.

### Deep Clone (`_deepClone`)

- `JSON.parse(JSON.stringify(obj))` — simple but lossy.
- **Does not preserve**: `undefined` values (converted to `null` in arrays, omitted in objects), `Date` objects (converted to strings), `RegExp`, functions, circular references (throws), `Infinity`, `NaN` (converted to `null`).
- **Acceptable for JSON Patch** because RFC 6902 operates on JSON-compatible values only.

---

## 5. Mutation vs Immutability

| Aspect | Behavior |
| ------ | -------- |
| Default mutation model | **Mutates** the input document in-place |
| Immutable option | `mutateDocument=false` triggers deep clone before patching |
| Clone strategy | `JSON.parse(JSON.stringify())` (full deep clone) |
| Structural sharing | None — either full mutation or full clone |
| Return value | Returns `{ newDocument }` which is the same reference when mutating, new reference for root replacement |

### Tradeoffs

**Mutation (default)**:
- ✅ Fast — no allocation overhead
- ✅ Memory-efficient — no copies
- ❌ Caller must be careful about shared references
- ❌ No rollback on partial failure

**Immutable mode** (`mutateDocument=false`):
- ✅ Original document preserved
- ❌ Full deep clone via JSON serialization — O(n) where n is document size
- ❌ No structural sharing — wasteful for large documents with small patches

### stdlib Recommendation

For the Vega SSE use case:
- **Mutation is likely more appropriate**: Vega specs can be large, patches are typically small, and the browser doesn't need to retain the old spec.
- However, an **optional clone** parameter is useful for safety.
- stdlib already has `@stdlib/utils/copy` for deep cloning, which handles more types than `JSON.parse(JSON.stringify())`.

---

## 6. Dependency Analysis

### Runtime Dependencies

**None.** The `package.json` has an empty `dependencies` field.

All functionality is self-contained:
- Deep clone: internal `_deepClone` (JSON round-trip)
- Deep equality: internal `_areEquals` (derived from `fast-deep-equal`)
- Object keys: internal `_objectKeys`
- hasOwnProperty: internal wrapper
- Integer check: internal `isInteger`
- Path escaping: internal functions
- Error class: internal `PatchError`

### Dependency Compatibility Assessment

| Internal Function | stdlib Equivalent | Assessment |
| ----------------- | ----------------- | ---------- |
| `_deepClone` | `@stdlib/utils/copy` | stdlib-compatible — stdlib's is more comprehensive |
| `_areEquals` | `@stdlib/assert/deep-equal` | stdlib-compatible — stdlib's handles more types |
| `_objectKeys` | `@stdlib/utils/keys` | stdlib-compatible |
| `hasOwnProperty` | `@stdlib/assert/has-own-property` | stdlib-compatible |
| `isInteger` | Could use `@stdlib/assert/is-nonnegative-integer` (partial) | Needs custom implementation — the library checks string representation of non-negative integer |
| `PatchError` | No direct equivalent | Needs custom implementation — stdlib uses standard Error types with `@stdlib/string/format` |
| `escapePathComponent` | No equivalent | Needs custom implementation |
| `unescapePathComponent` | No equivalent | Needs custom implementation |

**Verdict**: All dependencies can be satisfied by existing `@stdlib/*` utilities or small internal functions.

---

## 7. stdlib Compatibility Assessment

### Code Style

| Criterion | Assessment |
| --------- | ---------- |
| Naming conventions | **Moderate** — uses camelCase (good), but also uses `_` prefix for internal functions and TypeScript-style interfaces/generics |
| Function organization | **Poor** — large monolithic `applyOperation` function (~130 lines) mixes path traversal, validation, and operation dispatch |
| Module boundaries | **Moderate** — reasonable split into core/helpers/duplex but helpers is a grab-bag |
| Documentation | **Poor** — minimal JSDoc, no usage examples in source |
| Error handling | **Moderate** — custom `PatchError` class with error codes, but uses class-based inheritance |
| Defensive programming | **Moderate** — validation is optional (off by default), prototype pollution check is good |

### API Style

| Criterion | Assessment |
| --------- | ---------- |
| Functional vs class-oriented | **Mixed** — mostly functional exports but uses `this` binding in operation dispatch, `PatchError` class, `Observer` interface |
| Argument ordering | **Good** — `(document, operation, ...options)` |
| Return values | **Moderate** — returns `{ newDocument, removed?, test? }` object — verbose but informative |
| Error semantics | **Good** — throws on invalid operations (when validation enabled) |
| Option handling | **Poor for stdlib** — too many positional boolean arguments; stdlib prefers explicit type-checked parameters |

### Performance

| Criterion | Assessment |
| --------- | ---------- |
| Path traversal | **Good** — single-pass split + linear walk |
| Operation dispatch | **Good** — hash lookup O(1) |
| Clone strategy | **Moderate** — `JSON.parse(JSON.stringify())` is fast but lossy |
| Allocation | **Good when mutating** — minimal allocation |
| `isInteger` | **Good** — character code comparison, faster than regex |

### Maintainability

| Criterion | Assessment |
| --------- | ---------- |
| Complexity | **Moderate** — core `applyOperation` is complex but linear |
| Coupling | **Low** — helpers are independent, duplex is layered on core |
| Readability | **Moderate** — some clever patterns (operation dispatch via `call`) reduce readability |
| Testability | **Good** — functions are exported and independently testable |

### Overall Rating: **Moderate**

The library is well-engineered for performance and has zero external dependencies, which aligns with stdlib. However, the TypeScript source, class-based patterns, `this`-binding dispatch, and positional boolean arguments conflict with stdlib conventions. The monolithic `applyOperation` function would need to be decomposed for stdlib.

---

## 8. What Should Be Borrowed

| Concept | Source | Why Useful | stdlib Adaptation |
| ------- | ------ | ---------- | ----------------- |
| **Operation dispatch via hash table** | `objOps`/`arrOps` objects | O(1) dispatch, clean separation of object vs array logic | Use plain object mapping from op name to handler function; avoid `this` binding |
| **Separate object/array operation handlers** | `objOps`/`arrOps` split | Arrays need `splice`; objects need `delete`. Clean separation | Maintain this separation in stdlib |
| **Prototype pollution protection** | `banPrototypeModifications` check | Critical security feature — blocks `__proto__` and `constructor.prototype` modification | Must include; make it non-optional (always enabled) |
| **`isInteger` via character codes** | `helpers.ts` | Faster than regex for validating array indices | Implement similar approach in stdlib |
| **JSON Pointer unescape order** | `unescapePathComponent` | Must replace `~1` before `~0` per RFC 6901 | Follow same ordering |
| **Root operation special-casing** | `applyOperation` root handling | Root operations (`path === ""`) require fundamentally different handling — no traversal, direct document replacement | Handle root operations as early return before path traversal |
| **Deep clone for `copy` operation** | `copy` handler | Prevents reference aliasing between source and destination (issue #177) | Use `@stdlib/utils/copy` instead of `JSON.parse(JSON.stringify())` |
| **`compare`/`_generate` diff algorithm** | `duplex.ts` | Useful for potential future `diff` capability in stdlib | Study recursive diff algorithm structure |
| **Validation as separate concern** | `validator` function | Separating validation from application allows both validated and unvalidated paths | Provide separate `validate` function; always validate by default in stdlib |

---

## 9. What Should NOT Be Borrowed

| Approach | Why to Avoid |
| -------- | ------------ |
| **`this` binding for operation dispatch** | `objOps[op].call(operation, ...)` uses the operation object as `this` context. This is fragile, non-obvious, and incompatible with stdlib's functional style. Pass operation properties as explicit arguments instead. |
| **Positional boolean parameters** | `applyOperation(doc, op, validate, mutate, banProto, index)` — 6 positional arguments, 3 of which are booleans. Nearly impossible to read at call sites. stdlib should use an options object or separate functions. |
| **`JSON.parse(JSON.stringify())` for deep clone** | Lossy (drops `undefined`, functions, special objects), throws on circular refs, and allocates a string intermediary. stdlib already has `@stdlib/utils/copy`. |
| **Class-based `PatchError`** | Uses ES6 `class extends Error` with `Object.setPrototypeOf`. stdlib avoids classes; use factory functions or plain `TypeError`/`RangeError` with descriptive messages. |
| **`_get` operation** | Non-standard operation (`op: '_get'`). It's used internally to implement `getValueByPointer` by applying a fake operation. This is clever but convoluted — a direct pointer traversal function is simpler and more efficient. |
| **Optional validation (off by default)** | Validation should be on by default for safety. Performance-sensitive callers can opt out. |
| **DOM event listeners in `observe`** | `duplex.ts` attaches `mouseup`/`keyup`/`mousedown`/`keydown`/`change` listeners to `window` for dirty checking. This is inappropriate for stdlib — completely browser-specific and tightly coupled to DOM. |
| **`~~key` for integer conversion** | Bitwise NOT trick is non-obvious. Use `parseInt(key, 10)` or a clearer conversion. |
| **Non-atomic patch application** | No rollback mechanism. For stdlib, consider optional atomicity (clone-then-apply or snapshot-and-rollback). |

---

## 10. Testing Analysis

### Test Suite Structure

| Test File | Size | Content |
| --------- | ---- | ------- |
| `coreSpec.mjs` | ~53KB | Core patch application tests |
| `duplexSpec.mjs` | ~51KB | Observe/generate/compare tests |
| `validateSpec.mjs` | ~18KB | Validation tests |
| `jsonPatchTestsSpec.mjs` | ~2KB | Runner for shared JSON Patch test suite |
| `json-patch-tests/` | Directory | Shared RFC compliance test suite (community standard) |

### Test Coverage Areas

- ✅ All 6 RFC operations
- ✅ Root-level operations
- ✅ Array operations (add, remove, replace at various indices)
- ✅ `-` array append
- ✅ Nested path traversal
- ✅ Pointer escaping (`~0`, `~1`)
- ✅ Invalid operations
- ✅ Validation error codes
- ✅ Deep equality for `test` operation
- ✅ `move` within same object
- ✅ `copy` reference independence
- ✅ Community JSON Patch test suite (`json-patch-tests`)
- ✅ Prototype pollution prevention
- ✅ Duplex: observe/generate/compare
- ✅ Benchmarks (core + duplex)

### Notable Test Cases for stdlib

The `json-patch-tests` directory contains the community-standard RFC 6902 compliance test suite. **stdlib should use this as a baseline test suite.**

### Proposed Test Matrix (Derived)

| Category | Tests Required |
| -------- | -------------- |
| Operation validity | Valid/invalid `op`, `path`, `from`, `value` |
| Path parsing | Empty, root, nested, escaped (`~0`/`~1`), invalid |
| Array operations | Insert at index, append (`-`), out-of-bounds, remove, replace |
| Object operations | Add new key, replace existing, remove existing, missing key |
| Root operations | Add, replace, remove, move-to-root, copy-to-root, test at root |
| `move` | Same-parent move, cross-branch move, array index shift |
| `copy` | Deep copy verification, reference independence |
| `test` | Equal values, unequal values, nested objects, arrays, NaN |
| Error handling | Invalid path, unresolvable path, missing `from`, type errors |
| Security | `__proto__` mutation, `constructor.prototype` mutation |
| Atomicity | Partial failure behavior |
| Deep equality | Objects with same keys different order, nested equality |
| Edge cases | Empty patch array, empty document, `null` document |

---

## 11. Performance Analysis

### Benchmark Methodology

The library includes two benchmark files:

1. **`coreBenchmark.js`** (~2.7KB): Benchmarks `applyPatch` and `applyOperation` with various patch sizes.
2. **`duplexBenchmark.js`** (~6KB): Benchmarks `observe`/`generate`/`compare` with various document sizes.

### Architecture-Level Performance Characteristics

| Aspect | Assessment |
| ------ | ---------- |
| Path traversal | O(d) where d = pointer depth — efficient |
| Operation dispatch | O(1) hash lookup — efficient |
| Array insert/remove | O(n) due to `splice` — inherent to arrays |
| Deep clone (immutable mode) | O(n) where n = document size — expensive for large documents |
| Deep equality (test) | O(v) where v = value size |
| Memory (mutable mode) | Minimal — patches are applied in-place |
| Memory (immutable mode) | Full document duplication |

**Not benchmarked** by this review. The library's own benchmarks were not executed.

---

## 12. Security and Robustness

### Prototype Pollution

✅ **Protected** via `banPrototypeModifications` (default: `true`):
- Blocks `__proto__` as a path component
- Blocks `constructor/prototype` combination
- Can be disabled (risky but documented)

### Other Security Considerations

| Risk | Assessment |
| ---- | ---------- |
| Deep recursion | ⚠️ No depth limit on path traversal or deep equality — extremely deep objects could cause stack overflow |
| Malformed operations | ✅ Caught by validator (when enabled); without validation, behavior is undefined |
| Invalid pointers | ✅ Throws `OPERATION_PATH_UNRESOLVABLE` when validation enabled |
| Denial of service | ⚠️ No limit on patch array size, value size, or document depth |
| Unexpected JS values | ⚠️ `_deepClone` via JSON serialization silently converts `undefined` → `null`, drops functions, converts `Date` to string |
| Mutation of unexpected objects | ⚠️ When `mutateDocument=true` (default), the caller's object is modified. If the caller doesn't expect this, it can cause bugs. |

---

## 13. Final Library Verdict

### Strengths

1. **Zero runtime dependencies** — entirely self-contained
2. **Clean operation dispatch architecture** — hash-table-based separation of object/array operations
3. **Prototype pollution protection** — built-in and enabled by default
4. **Community RFC compliance tests** — uses the standard `json-patch-tests` suite
5. **Comprehensive test coverage** — ~120KB of test code
6. **Duplex/diff capability** — observe changes and generate patches (bonus, not core)
7. **Root operation handling** — explicit and correct
8. **Correct JSON Pointer escaping** — proper `~0`/`~1` ordering

### Weaknesses

1. **TypeScript source** — cannot be directly used in stdlib (CommonJS, plain JS)
2. **Monolithic `applyOperation` function** — ~130 lines mixing traversal, validation, dispatch
3. **`this` binding for dispatch** — fragile, non-obvious pattern
4. **Positional boolean arguments** — poor API ergonomics
5. **`JSON.parse(JSON.stringify())` for cloning** — lossy, not suitable for general use
6. **Non-atomic by default** — no rollback on partial failure
7. **Validation off by default** — unsafe for production use without explicit opt-in
8. **Class-based `PatchError`** — incompatible with stdlib conventions
9. **DOM event listeners in duplex** — tightly coupled to browser environment

### Best Ideas

1. **Operation dispatch via hash tables** — fast, extensible
2. **Separate object/array handlers** — clean separation of concerns
3. **Prototype pollution check** — essential security measure
4. **`isInteger` via character codes** — micro-optimization for hot path
5. **JSON Pointer escape ordering** — correct implementation to reference
6. **Root operation early return** — prevents unnecessary path traversal
7. **Community test suite** — must-use for any implementation

### stdlib Fit: **Moderate**

The library is technically strong and dependency-free, making it a valuable reference. However, the TypeScript source, class patterns, `this` binding, and positional booleans require substantial reworking for stdlib. The core algorithms (dispatch, traversal, escaping) are well-suited for adaptation.

### Recommendation: **Strong reference**

Fast-JSON-Patch should be the **primary architectural reference** for stdlib's JSON Patch implementation. Its operation dispatch pattern, path traversal approach, and prototype pollution protection are well-engineered and directly adaptable. The RFC compliance test suite should be adopted. However, the API design, error handling, and module structure must be completely reimagined for stdlib conventions.
