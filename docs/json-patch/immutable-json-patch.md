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

# immutable-json-patch — Technical Review

> Library: `immutable-json-patch` v6.0.3
> Repository: [josdejong/immutable-json-patch](https://github.com/josdejong/immutable-json-patch)
> Location in stdlib tree: `libraries/immutable-json-patch`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | immutable-json-patch |
| **Primary language** | TypeScript |
| **Module system** | ES modules (type: "module", compiled to ESM + UMD + CJS) |
| **Approximate implementation size** | `src/immutableJSONPatch.ts` (~209 lines), `src/immutabilityHelpers.ts` (~299 lines), `src/revertJSONPatch.ts` (~223 lines), `src/jsonPointer.ts` (~35 lines), `src/utils.ts` (~55 lines), `src/typeguards.ts` (~60 lines). Total: ~880 lines |
| **License** | ISC |
| **Runtime dependencies** | **None** |

### Public API

- `immutableJSONPatch(document, operations, [options])` — Apply patch immutably with structural sharing
- `revertJSONPatch(document, operations, previousDocument)` — Generate revert patch
- Immutability helpers: `getIn`, `setIn`, `updateIn`, `deleteIn`, `insertAt`, `existsIn`, `transform`
- JSON Pointer: `parseJSONPointer`, `compileJSONPointer`
- Type guards: `isJSONObject`, `isJSONArray`, `isJSONPatchOperation`, etc.

### Key Design Characteristic: **Structural Sharing**

This is the **defining feature** of this library. Unlike all other libraries reviewed:

- **Mutation-based**: Mutates in-place → fast but destructive
- **Full-clone immutability**: Clones entire document → safe but O(n)
- **Structural sharing (this library)**: Only shallow-clones objects along the modified path → safe AND efficient

#### How Structural Sharing Works

The `setIn` function implements structural sharing:

```
setIn(object, ['a', 'b', 'c'], newValue)
```

1. Recursively descends from root to `object.a.b.c`
2. At the leaf, creates the new value
3. On the way back up, calls `applyProp(object, key, updatedValue)`:
   - If `object[key] === updatedValue` (no change), returns `object` unchanged
   - Otherwise, creates a **shallow clone** of `object` and sets `clone[key] = updatedValue`
4. Result: only `root`, `root.a`, `root.a.b` are new objects; everything else (e.g., `root.x`, `root.a.y`) is shared

This is the same approach used by React/Redux immutable state management.

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | Array insert via `insertAt`, object via `setIn` |
| `remove` | ✅ | `deleteIn` |
| `replace` | ✅ | `setIn` (only if path exists — checked via `existsIn`) |
| `move` | ✅ | `deleteIn` + `setIn`/`insertAt` |
| `copy` | ✅ | `getIn` + `setIn`/`insertAt` (no deep clone — shares reference) |
| `test` | ✅ | Custom `isEqual` |

### Hooks: `before` and `after`

The `options` parameter supports `before` and `after` hooks:
- `before(document, operation)` — Called before each operation, can modify the operation or document
- `after(document, operation, previousDocument)` — Called after each operation, can modify the result

These hooks enable middleware-style processing (logging, transformation, undo tracking).

### JSON Pointer

Minimal implementation in `jsonPointer.ts` (~35 lines):
- `parseJSONPointer(pointer)` — Splits on `/`, unescapes `~1`→`/` and `~0`→`~`
- `compileJSONPointer(path)` — Joins with `/`, escapes `~`→`~0` and `/`→`~1`

### Deep Equality

`isEqual` in `utils.ts` — basic recursive equality. Does NOT handle NaN or circular references.

### Revert Patch Generation

`revertJSONPatch.ts` generates a reverse patch by comparing the result document with the original. For each operation:
- `add` → `remove` (or `replace` if overwriting)
- `remove` → `add` (with the removed value from the original document)
- `replace` → `replace` (with the original value)
- `move` → `move` (reversed from/path)
- `copy` → `remove` (or `replace`)

---

## 2. Architecture Analysis

```
immutable-json-patch
├── immutableJSONPatch.ts — Main patch application
│   ├── immutableJSONPatch() — Apply patch with hooks
│   ├── add() — Immutable add (setIn/insertAt)
│   ├── remove() — Immutable remove (deleteIn)
│   ├── replace() — Immutable replace (setIn)
│   ├── copy() — Immutable copy (getIn + setIn)
│   ├── move() — Immutable move (deleteIn + setIn)
│   ├── test() — Deep equality check
│   └── validateJSONPatchOperation()
│
├── immutabilityHelpers.ts — Core structural sharing primitives
│   ├── shallowClone() — Clone array/object/value
│   ├── applyProp() — Set property with change detection
│   ├── getIn() — Get nested value
│   ├── setIn() — Set nested value (structural sharing)
│   ├── updateIn() — Update nested value with transform
│   ├── deleteIn() — Delete nested value (structural sharing)
│   ├── insertAt() — Insert into array (structural sharing)
│   ├── existsIn() — Check path existence
│   └── transform() — Deep transform with structural sharing
│
├── revertJSONPatch.ts — Revert patch generation
├── jsonPointer.ts — JSON Pointer parsing
├── typeguards.ts — Type checking utilities
├── utils.ts — isEqual, initial, last
└── types.ts — TypeScript type definitions
```

---

## 3. Mutation vs Immutability

| Aspect | Behavior |
| ------ | -------- |
| Default model | **Immutable with structural sharing** |
| Mutation option | None — always immutable |
| Clone strategy | Shallow clone only along modified path |
| Change detection | `applyProp` returns original if value unchanged |
| Memory efficiency | ✅ Excellent — shares unmodified subtrees |
| Time efficiency | ✅ O(d) where d = path depth (not O(n) for document size) |

### Structural Sharing vs Other Approaches

| Approach | Time | Space | Safety | Library |
| -------- | ---- | ----- | ------ | ------- |
| In-place mutation | O(1) per op | O(0) | ❌ Destructive | Fast-JSON-Patch (default) |
| Full deep clone + mutate | O(n) clone | O(n) | ✅ Safe | Fast-JSON-Patch (mutate=false) |
| Structural sharing | O(d) per op | O(d) | ✅ Safe | **immutable-json-patch** |

Where d = path depth, n = document size. Structural sharing is **optimal** for large documents with small, deep patches — exactly the Vega specification use case.

---

## 4. Security & Robustness

| Risk | Assessment |
| ---- | ---------- |
| Prototype pollution | ⚠️ No explicit `__proto__` check |
| Validation | ✅ `validateJSONPatchOperation` checks op, path, from |
| Path errors | ✅ `existsIn` check for `replace`; throws on missing path |
| `copy` reference aliasing | ⚠️ **Does NOT deep clone** — `getIn(doc, from)` returns reference, `setIn` stores it. Structural sharing means the copied value is shared (reference), not cloned. |
| Symbol preservation | ✅ `shallowClone` preserves symbols |
| NaN handling | ❌ `isEqual` does not handle NaN |

### Copy Reference Aliasing

> **Important**: The `copy` operation does **not** deep clone the value. It uses `getIn` to get a reference and `setIn` to store it. Since the library is immutable (original is never mutated), this is acceptable — the shared reference can't be corrupted through mutations. However, if external code mutates the copied value, both locations would be affected.

---

## 5. stdlib Relevance

### What Should Be Borrowed

| Concept | Source | Why Useful | stdlib Adaptation |
| ------- | ------ | ---------- | ----------------- |
| **Structural sharing via `setIn`/`deleteIn`** | `immutabilityHelpers.ts` | Optimal for Vega SSE: large doc, small patches. O(d) vs O(n). | Implement `setIn`/`deleteIn` as stdlib utility or internal helper. |
| **`applyProp` with change detection** | `immutabilityHelpers.ts` | Returns original if value unchanged — minimizes unnecessary cloning. | Essential optimization for structural sharing. |
| **`insertAt` for array insertion** | `immutabilityHelpers.ts` | Clean immutable array insertion with structural sharing. | Adapt for stdlib. |
| **`existsIn` path check** | `immutabilityHelpers.ts` | Necessary for `replace` validation. | Implement. |
| **`resolvePathIndex` for `-`** | `immutableJSONPatch.ts` | Resolves `-` to actual array length before operations. | Adopt this approach. |
| **`before`/`after` hooks concept** | `immutableJSONPatch.ts` | Extensibility without modifying core. | Consider for future stdlib enhancement. |
| **Revert patch generation** | `revertJSONPatch.ts` | Useful for undo/redo. | Consider for future stdlib enhancement. |

### What Should NOT Be Borrowed

| Approach | Why |
| -------- | --- |
| TypeScript + ES modules | stdlib uses plain JS + CommonJS |
| No deep clone for `copy` | Acceptable with immutability but risky if stdlib offers mutable mode |
| Missing prototype pollution protection | Security gap |
| Missing NaN handling in `isEqual` | Use `@stdlib/assert/deep-equal` instead |
| `path.slice(1)` at every recursion in `setIn` | Creates intermediate arrays — could be optimized with index tracking |
| Always immutable (no mutable option) | stdlib should offer both modes |

---

## 6. Final Verdict

### Strengths

1. **Structural sharing** — the most sophisticated immutability approach among all reviewed libraries
2. **Zero runtime dependencies**
3. **Clean architecture** — well-separated helpers, typeguards, pointer, main logic
4. **Revert patch generation** — built-in undo support
5. **`before`/`after` hooks** — extensible
6. **Active development** — v6.0.3, actively maintained
7. **Good test coverage** — comprehensive test files in `src/`

### Weaknesses

1. **TypeScript + ES modules** — incompatible with stdlib
2. **No deep clone for `copy`** — reference aliasing under certain conditions
3. **No prototype pollution protection**
4. **No NaN handling** in equality
5. **`path.slice(1)` creates garbage** at each recursion level in `setIn`
6. **No mutable mode** — always clones along the path

### Best Ideas

1. **Structural sharing via `setIn`/`deleteIn`/`applyProp`** — THE key algorithm for stdlib
2. **Change detection in `applyProp`** — returns original if unchanged
3. **`resolvePathIndex`** — pre-resolves `-` before operations
4. **Revert patch generation** — for future undo feature

### stdlib Fit: **Good** (algorithms), **Poor** (code style)

The algorithms and data structures are excellent and directly applicable to the Vega SSE use case. The structural sharing approach is the optimal solution for large documents with small patches. However, the TypeScript/ES module code must be completely reimplemented in stdlib conventions.

### Recommendation: **Primary reference for immutability strategy**

immutable-json-patch should be the **primary reference for structural sharing and immutable operations** in the stdlib implementation. Its `setIn`/`deleteIn`/`applyProp` primitives should be adapted as internal stdlib helpers, providing O(d) immutable updates instead of O(n) full clones.
