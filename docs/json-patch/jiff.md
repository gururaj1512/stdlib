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

# jiff — Technical Review

> Library: `jiff` v0.7.3
> Repository: [cujojs/jiff](https://github.com/cujojs/jiff)
> Location in stdlib tree: `libraries/jiff`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | jiff |
| **Primary language** | JavaScript (plain, no transpilation) |
| **Module system** | CommonJS |
| **Approximate implementation size** | ~16 source files in `lib/`, entry `jiff.js` (~241 lines). Core: `patches.js` (~382 lines), `jsonPointer.js` (~162 lines), `jsonPointerParse.js` (~46 lines), `lcs.js` (~149 lines), `clone.js` (~50 lines), `deepEquals.js` (~59 lines). Total: ~1200 lines. |
| **License** | MIT |

### Public API

- `jiff.diff(a, b, [options])` — compute JSON Patch from two objects
- `jiff.patch(changes, x, [options])` — apply patch (clones first, immutable)
- `jiff.patchInPlace(changes, x, [options])` — apply patch (mutates in-place)
- `jiff.inverse(patch)` — compute inverse of a patch
- `jiff.clone(x)` — deep clone utility

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | Object and array, root replacement |
| `remove` | ✅ | Object and array |
| `replace` | ✅ | Object and array, root replacement |
| `move` | ✅ | With ancestor check |
| `copy` | ✅ | With deep clone |
| `test` | ✅ | Custom deep equality |

### RFC Compliance

- **RFC 6902 (JSON Patch)**: Yes
- **RFC 6901 (JSON Pointer)**: Yes
- **Extensions beyond RFC 6902**: `inverse` (patch inversion), `commute` (operation reordering), `context` (fuzzy patching), LCS-based array diffing

### Runtime/Browser Support

- Node.js (tested with buster)
- Browser (via jsdelivr/bower)
- No build step required

### External Dependencies

- **Runtime**: **None** (zero `dependencies` in `package.json`)
- **Development**: `buster ~0.7`, `gent 0.6.2`, `jshint ~2`, `json-patch-test-suite ^1.0.1`

---

## 2. JSON Patch Specification Compliance

### JSON Pointer (RFC 6901)

| Behavior | Status | Evidence |
| -------- | ------ | -------- |
| Pointer parsing | ✅ RFC-compliant | `jsonPointerParse.js`: regex-based single-pass parser handles `/`, `~0`, `~1` inline |
| `~0` → `~` decoding | ✅ Correct | Regex matches `~0` and maps to `~` (escape char) |
| `~1` → `/` decoding | ✅ Correct | Regex matches `~1` and maps to `/` (separator) |
| Order of decoding | ✅ Correct | Single regex `\/|~1|~0` matches all three in one pass — no ordering issue |
| Encoding (escape) | ✅ Correct | `encodeSegment`: `~` → `~0` first, `/` → `~1` second |
| Empty pointer (root) | ✅ Handled | `find`: `path === ''` → `{ target: x, key: void 0 }` |
| `/` pointer (empty-string key) | ✅ Handled | `find`: `path === '/'` → `{ target: x, key: '' }` |
| Array indexing | ✅ | `parseArrayIndex`: regex `/^(0|[1-9]\d*)$/`, coercion via `+s` |
| `-` array append | ✅ | Checked in `_add`: `pointer.key === '-'` → `target.push(value)` |
| Invalid array index | ✅ | Throws `SyntaxError` from `parseArrayIndex` |

### Operation Analysis

#### `add`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object property addition | ✅ RFC-required | `target[pointer.key] = value` |
| Array element insertion | ✅ RFC-required | `target.splice(pointer.key, 0, value)` |
| Root replacement | ✅ RFC-required | `pointer.key === void 0` → returns `val` directly |
| `-` append | ✅ RFC-required | `pointer.key === '-'` → `target.push(value)` |
| Index > length validation | ✅ RFC-required | `pointer.key > target.length` throws |
| Deep clone of value | ✅ RFC-correct | `clone(change.value)` before insert — prevents aliasing |
| Missing value validation | ✅ | Throws `InvalidPatchOperationError` if `change.value === void 0` |

#### `remove`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object property removal | ✅ RFC-required | `delete target[pointer.key]` |
| Array element removal | ✅ RFC-required | `target.splice(parseArrayIndex(pointer.key), 1)` |
| Path existence check | ✅ RFC-required | Checks `notFound(pointer) || pointer.target[pointer.key] === void 0` |

#### `replace`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object replacement | ✅ RFC-required | `target[pointer.key] = value` |
| Array replacement | ✅ RFC-required | `target[parseArrayIndex(pointer.key)] = value` |
| Root replacement | ✅ RFC-required | Returns `value` directly when `pointer.key === void 0` |
| Existing value required | ✅ RFC-required | Checks `notFound(pointer) || missingValue(pointer)` |
| Deep clone of value | ✅ RFC-correct | `clone(change.value)` |

#### `move`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Decomposition to remove + add | ✅ RFC-required | `_add(pto, _remove(pfrom))` |
| Ancestor check | ✅ RFC-required | `jsonPointer.contains(change.path, change.from)` → throws |
| No unnecessary cloning | ✅ | Value is moved by reference (no clone needed since it's removed from source) |

#### `copy`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Deep copy of value | ✅ RFC-correct | `clone(value)` before `_add` |
| `from` validation | ✅ RFC-required | Checks `notFound(pfrom) || missingValue(pfrom)` |

#### `test`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Deep equality check | ✅ RFC-required | Uses custom `deepEquals` function |
| Throws on failure | ✅ RFC-required | Throws `TestFailedError` |
| Array comparison | ✅ | Length check + element-wise recursion |
| Object comparison | ✅ | Key count + `in` operator + recursive |
| NaN handling | ❌ Potential bug | `NaN === NaN` is `false` in JS; no special case like `a !== a && b !== b` |
| null vs object | ✅ | Explicit check: `a === null && b !== null` |

### Atomicity

- **Observed**: **Not atomic**. `patchInPlace` applies operations sequentially. On failure, already-applied operations remain.
- `patch()` (immutable version) clones the document first, so the original is preserved, but the clone is still partially patched on failure.

---

## 3. Architecture Analysis

```
jiff
├── jiff.js — Entry point and diff algorithm
│   ├── diff(a, b, options) — Top-level diff
│   ├── appendChanges() — Recursive diff dispatcher
│   ├── appendObjectChanges() — Object diff
│   ├── appendArrayChanges() — Array diff (via LCS)
│   ├── appendValueChanges() — Primitive diff
│   └── lcsToJsonPatch() — LCS matrix → patch operations
│
├── lib/
│   ├── jsonPatch.js — Patch application
│   │   ├── patch() — Immutable (clones then applies)
│   │   └── patchInPlace() — Mutable application
│   │
│   ├── patches.js — Individual operation handlers
│   │   ├── exports.test = { apply, inverse, commute }
│   │   ├── exports.add = { apply, inverse, commute }
│   │   ├── exports.remove = { apply, inverse, commute }
│   │   ├── exports.replace = { apply, inverse, commute }
│   │   ├── exports.move = { apply, inverse, commute }
│   │   └── exports.copy = { apply, inverse, commute }
│   │
│   ├── jsonPointer.js — JSON Pointer utilities
│   │   ├── find() — Traverse to pointer target
│   │   ├── parse() — Split pointer into segments
│   │   ├── contains() — Ancestor check
│   │   ├── encodeSegment() / decodeSegment()
│   │   └── parseArrayIndex() / isValidArrayIndex()
│   │
│   ├── jsonPointerParse.js — Low-level single-pass parser
│   ├── lcs.js — LCS (Longest Common Subsequence)
│   │   ├── compare() — Build LCS matrix
│   │   └── reduce() — Traverse LCS matrix
│   │
│   ├── clone.js — Deep clone (recursive)
│   ├── deepEquals.js — Deep equality
│   ├── inverse.js — Patch inversion
│   ├── commute.js — Operation commutation
│   ├── commutePaths.js — Path-based commutation
│   ├── context.js — Fuzzy patching context
│   ├── array.js — Array utilities (cons, tail, map)
│   ├── rebase.js — Patch rebasing
│   └── Error classes (3 files)
```

### Key Design Decisions

1. **Each operation is a module with `apply`, `inverse`, and `commute` methods**: This is a rich algebraic design that treats patches as transformable mathematical objects. Each operation knows how to apply itself, invert itself, and commute with other operations.

2. **JSON Pointer as a separate, reusable module**: Clean separation — `find()` returns a `{ target, key }` descriptor rather than directly modifying the target. This allows the operation handler to decide what to do.

3. **LCS-based array diffing**: Uses Longest Common Subsequence algorithm to produce optimal array diffs. This is more sophisticated than the simple property comparison in Fast-JSON-Patch.

4. **Immutable by default**: `patch()` clones before applying; `patchInPlace()` for mutation.

5. **Patch as a first-class concept**: With `inverse`, `commute`, and `rebase`, patches are treated as algebraic objects — useful for OT (Operational Transformation) scenarios.

---

## 4. Algorithm Analysis

### JSON Pointer

- **Parsing**: `jsonPointerParse.js` uses a single regex `\/|~1|~0/g` and `exec` in a loop. This is a **single-pass streaming parser** — it processes the string left-to-right, decoding escape sequences inline. More efficient than split+replace because it avoids creating an intermediate array and doing two replace passes.
- **Traversal**: `find()` iterates segments via callback from `jsonPointerParse`, descending `x = x[key]` at each step.
- **Return value**: `{ target: parent, key: key }` — the parent object/array and the final key. This is elegant because it gives the caller direct access to the parent for mutation.
- **Array handling**: `parseArrayIndex` uses regex `/^(0|[1-9]\d*)$/` — correct per RFC 6901 (no leading zeros).

### `add`

- Clone value: `clone(change.value)` — O(v) where v is value size.
- Array: `splice(key, 0, value)` — O(n) for array shift.
- Object: `target[key] = value` — O(1).
- Root: returns cloned value — O(v).

### `remove`

- Array: `splice(parseArrayIndex(pointer.key), 1)` — O(n).
- Object: `delete target[pointer.key]` — O(1).
- Returns removed value (useful for inverse/move).

### `replace`

- Clone value: `clone(change.value)` — O(v).
- Separate array/object handling with `parseArrayIndex` for arrays.

### `move`

- `_add(pto, _remove(pfrom))` — removes from source, adds to destination.
- Ancestor check: `jsonPointer.contains(change.path, change.from)` — prevents moving a node into its own subtree.
- **No unnecessary cloning** — the removed value is directly inserted.

### `copy`

- `clone(value)` for deep copy, then `_add`.

### `test`

- `deepEquals`: recursive, handles arrays and objects.
- **Does NOT handle NaN** — `NaN === NaN` is false, and there's no special case.

### Deep Clone (`clone.js`)

- **Recursive implementation** — not `JSON.parse(JSON.stringify())`.
- Handles: `null`, primitives, boxed primitives (`String`, `Number`, `Boolean`), arrays, plain objects.
- **Better than Fast-JSON-Patch's clone** for correctness (handles boxed primitives, doesn't go through JSON serialization).
- Still limited to JSON-compatible types.

### LCS Algorithm

- **DP-based**: Builds an (m+1)×(n+1) matrix comparing array elements.
- **Prefix/suffix optimization**: `findPrefix` and `findSuffix` skip common elements at the beginning and end before building the matrix. This significantly reduces matrix size for arrays with small changes.
- **Complexity**: O(m×n) time and space for the matrix, reduced by prefix/suffix optimization.
- **Hashing**: Array elements are hashed via `JSON.stringify` by default, or a custom `options.hash` function. This allows comparing complex objects.

---

## 5. Mutation vs Immutability

| Aspect | Behavior |
| ------ | -------- |
| Default model | **Immutable** — `patch()` clones first |
| Mutable option | `patchInPlace()` for in-place mutation |
| Clone strategy | Recursive `clone.js` (not JSON round-trip) |
| Values always cloned | `add`/`replace` clone `change.value` before insertion (even in `patchInPlace`) |
| Return value | Returns the (possibly new) document |

### Tradeoffs

**Immutable (default)**:
- ✅ Original preserved on failure
- ✅ Safe for multi-consumer scenarios
- ❌ Full document clone cost

**Mutable (`patchInPlace`)**:
- ✅ Faster — avoids document clone
- ⚠️ Still clones individual operation values (defensive)
- ❌ No rollback on partial failure

### Value cloning note

jiff **always clones `change.value`** before inserting, even in `patchInPlace`. This is defensive — prevents the patch operations from sharing references with the document. This is more correct than Fast-JSON-Patch (which only clones for `copy`, not for `add`/`replace`).

---

## 6. Dependency Analysis

### Runtime Dependencies

**None.** Zero `dependencies` in `package.json`.

| Internal Function | stdlib Equivalent | Assessment |
| ----------------- | ----------------- | ---------- |
| `clone.js` | `@stdlib/utils/copy` | **stdlib-compatible** — stdlib's is more comprehensive |
| `deepEquals.js` | `@stdlib/assert/deep-equal` | **stdlib-compatible** — stdlib's handles more types |
| `jsonPointer.js` | No equivalent | Needs implementation |
| `jsonPointerParse.js` | No equivalent | Needs implementation — but the streaming parser is elegant |
| `lcs.js` | No equivalent | Only needed for `diff`, not for `patch` — can be deferred |
| `patches.js` | No equivalent | Operation handlers — must implement |
| Error classes | stdlib error conventions | Use `TypeError`/`RangeError`/`Error` with `@stdlib/string/format` |

**Verdict**: All dependencies are internal. No external runtime deps. Very stdlib-compatible.

---

## 7. stdlib Compatibility Assessment

### Code Style

| Criterion | Assessment |
| --------- | ---------- |
| Language | ✅ **Excellent** — plain JavaScript, CommonJS, no transpilation |
| Naming | ✅ **Good** — `camelCase` functions, descriptive names |
| Indentation | ✅ Uses tabs (matches stdlib) |
| Function organization | ✅ **Good** — small, focused functions |
| Module boundaries | ✅ **Excellent** — clean separation of concerns |
| Documentation | ✅ **Good** — JSDoc on public functions |
| Error handling | ✅ **Good** — custom error types, clear messages |
| Defensive programming | ✅ **Good** — validates paths, values, operations |

### API Style

| Criterion | Assessment |
| --------- | ---------- |
| Functional | ✅ **Excellent** — purely functional, no classes for core logic |
| Argument ordering | ⚠️ **Note**: `patch(changes, document)` — patch first, document second. stdlib typically puts the object first. |
| Options object | ✅ **Good** — uses options object instead of positional booleans |
| Return values | ✅ **Good** — returns the patched document |
| Error semantics | ✅ **Good** — throws on invalid operations |

### Performance

| Criterion | Assessment |
| --------- | ---------- |
| Clone strategy | ✅ Recursive (avoids JSON serialization) |
| Path traversal | ✅ Single-pass parser (efficient) |
| LCS | ✅ Prefix/suffix optimization reduces matrix size |
| Value cloning | ⚠️ Clones values even in `patchInPlace` — extra overhead but safer |

### Maintainability

| Criterion | Assessment |
| --------- | ---------- |
| Complexity | ✅ **Low** — each module is small and focused |
| Coupling | ✅ **Low** — modules interact through well-defined interfaces |
| Readability | ✅ **High** — clear, well-documented code |
| Testability | ✅ **High** — each module independently testable |

### Overall Rating: **Good**

jiff is the most stdlib-compatible library in terms of code style, module design, and philosophy. It uses plain JavaScript, CommonJS, tabs, small focused functions, and functional APIs. The main deviations are argument ordering (`patch, doc` vs `doc, patch`) and the error class inheritance pattern.

---

## 8. What Should Be Borrowed

| Concept | Source | Why Useful | stdlib Adaptation |
| ------- | ------ | ---------- | ----------------- |
| **`{ target, key }` pointer resolution** | `jsonPointer.find()` | Returns parent + final key — enables clean operation dispatch without duplicating traversal | Adopt this pattern for stdlib's pointer resolver |
| **Single-pass pointer parser** | `jsonPointerParse.js` | More efficient than split+replace — handles decoding inline during traversal | Study the regex approach; consider a non-regex equivalent for stdlib |
| **Operation-as-module pattern** | `patches.js` | Each operation exports `{ apply, inverse?, commute? }` — extensible, testable | Adopt `apply` dispatch; skip `inverse`/`commute` initially |
| **Recursive clone** | `clone.js` | Better than `JSON.parse(JSON.stringify())` — handles boxed primitives, doesn't lose information | Use `@stdlib/utils/copy` instead (even better) |
| **Value cloning on insert** | `applyAdd`, `applyReplace` | Always clones `change.value` before insertion — prevents reference aliasing | Consider adopting; documents the defensive behavior |
| **Ancestor check for `move`** | `jsonPointer.contains()` | Prevents moving a node into its own subtree (RFC 6902 Section 4.4) | Must implement this check |
| **LCS-based array diffing** | `lcs.js` | Optimal array diff — prefix/suffix optimization is efficient | Useful for future `diff` feature, not needed for `patch` |
| **`parseArrayIndex` regex** | `jsonPointer.js` | `/^(0|[1-9]\d*)$/` — correct per RFC 6901 (rejects leading zeros, accepts `0`) | Adopt this validation pattern |

---

## 9. What Should NOT Be Borrowed

| Approach | Why to Avoid |
| -------- | ------------ |
| **Argument ordering: `patch(changes, x)`** | stdlib should put the document first: `apply(document, patch)`. Consistency with existing stdlib patterns like `merge(target, source)`. |
| **`commute`/`rebase` system** | Over-engineered for stdlib's use case (SSE Vega updates). Operational Transformation is a separate concern. |
| **`context`/fuzzy patching** | Non-standard extension. Adds complexity without clear benefit for strict RFC 6902 compliance. |
| **`inverse` as core feature** | Useful but not essential for the initial stdlib implementation. Can be added later. |
| **Custom error classes via prototype manipulation** | `InvalidPatchOperationError.prototype = new Error()` — legacy pattern. stdlib uses standard Error types. |
| **No prototype pollution protection** | jiff has **no protection** against `__proto__` modification. This is a significant security gap. |
| **NaN equality bug** | `deepEquals` does not handle NaN correctly. stdlib's `@stdlib/assert/deep-equal` does. |
| **`buster` test framework** | Obsolete. stdlib uses `tape`. |

---

## 10. Testing Analysis

### Test Suite Structure

| Test File | Content |
| --------- | ------- |
| `jiff-test.js` | Diff tests |
| `jsonPatch-test.js` | Patch application tests |
| `jsonPointerParse-test.js` | Pointer parsing tests |
| `commute-test.js` | Operation commutation tests |
| `commutePaths-test.js` | Path commutation tests |
| `inverse-test.js` | Patch inversion tests |
| `rebase-test.js` | Patch rebasing tests |
| `json-patch-tests-test.js` | Community RFC test suite runner |

### Coverage Areas

- ✅ All 6 RFC operations
- ✅ Community JSON Patch test suite (`json-patch-test-suite` dep)
- ✅ Pointer parsing with escaping
- ✅ Array operations
- ✅ Object diff and patch round-trip
- ✅ LCS-based array diff
- ✅ Operation commutation
- ✅ Patch inversion
- ⚠️ Limited edge case coverage compared to Fast-JSON-Patch
- ❌ No prototype pollution tests
- ❌ No explicit NaN test

---

## 11. Performance Analysis

### Architecture-Level Characteristics

| Aspect | Assessment |
| ------ | ---------- |
| Pointer parsing | ✅ **Excellent** — single-pass regex parser, no intermediate arrays |
| Operation dispatch | ✅ Hash lookup `patches[p.op]` — O(1) |
| Clone on patch | ⚠️ Values always cloned — extra allocations |
| Array diff (LCS) | ⚠️ O(m×n) — can be expensive for large arrays |
| Prefix/suffix opt | ✅ Significantly reduces LCS matrix size |
| Memory | Moderate — LCS matrix can be large for arrays |

### Perf Directory

The `perf/` directory exists but was not inspected in detail. **Not benchmarked** by this review.

---

## 12. Security and Robustness

| Risk | Assessment |
| ---- | ---------- |
| **Prototype pollution** | ❌ **No protection** — `__proto__` and `constructor.prototype` are not blocked. This is a significant vulnerability. |
| Deep recursion | ⚠️ No depth limit on `find`, `clone`, or `deepEquals` |
| Malformed operations | ✅ Caught — invalid op throws `InvalidPatchOperationError` |
| Invalid pointers | ✅ `parseArrayIndex` throws on invalid; `find` returns `undefined` on missing paths |
| Malformed patch array | ⚠️ Non-array `changes` silently returns `x` unchanged (TODO comment suggests throwing) |
| NaN handling | ❌ `deepEquals` treats `NaN !== NaN` — potential issue for `test` operation |

---

## 13. Final Library Verdict

### Strengths

1. **Plain JavaScript, CommonJS** — most stdlib-compatible language/module choice
2. **Clean modular architecture** — small focused modules with well-defined interfaces
3. **`{ target, key }` pointer resolution** — elegant pattern for operation dispatch
4. **Single-pass pointer parser** — efficient, correct
5. **Recursive clone** — better than JSON round-trip
6. **LCS-based array diffing** — sophisticated, optimal diff algorithm
7. **Immutable by default** — safe default, mutable option available
8. **Always clones operation values** — prevents reference aliasing
9. **Algebraic patch operations** — inverse, commute, rebase capabilities (advanced)
10. **Zero dependencies**

### Weaknesses

1. **No prototype pollution protection** — critical security gap
2. **NaN equality bug** in `deepEquals`
3. **Legacy test framework** (buster — obsolete)
4. **Argument ordering** (`patch, doc` instead of `doc, patch`)
5. **No validation function** — validation is inline only
6. **`commute`/`rebase` adds complexity** — not needed for stdlib's use case
7. **Appears unmaintained** — last meaningful update years ago

### Best Ideas

1. **Single-pass pointer parser** — most efficient approach reviewed
2. **`{ target, key }` resolution pattern** — clean separation of traversal from operation
3. **LCS array diffing with prefix/suffix optimization** — for future `diff` feature
4. **Value cloning on insert** — defensive, correct behavior
5. **Ancestor check for `move`** — important RFC requirement
6. **Operation-as-module** — clean, extensible dispatch pattern

### stdlib Fit: **Good**

The most stylistically compatible library reviewed so far. Plain JavaScript, CommonJS, tabs, functional style, small modules. The core algorithms are clean and correct. However, the security gap (no prototype pollution protection) and the NaN equality bug are concerning.

### Recommendation: **Strong reference**

jiff should be a **primary reference for architecture and module design**. Its pointer resolution pattern, module structure, and recursive clone are directly applicable. The LCS algorithm is valuable for future `diff` capability. However, security (prototype pollution) and correctness (NaN) issues must be addressed in the stdlib implementation.

**Best for**: Architecture, module decomposition, pointer resolution pattern, array diffing algorithm.
**Not best for**: Security model, deep equality, error handling conventions.
