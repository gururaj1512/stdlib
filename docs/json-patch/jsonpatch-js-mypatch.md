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

# jsonpatch.js (dharmafly) — Technical Review

> Library: `jsonpatch` v3.1.0
> Repository: [dharmafly/jsonpatch.js](https://github.com/dharmafly/jsonpatch.js)
> Location in stdlib tree: `libraries/jsonpatch.js`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | jsonpatch.js (npm: `jsonpatch`) |
| **Primary language** | JavaScript (plain) |
| **Module system** | UMD (CommonJS + AMD + global) |
| **Approximate implementation size** | Single file `lib/jsonpatch.js` (~533 lines) |
| **License** | MIT |
| **Runtime dependencies** | **None** |
| **Node.js compat** | `>=0.4.0` |

### Public API

- `apply_patch(doc, patch)` — Apply a patch array to a document (immutable default)
- `JSONPatch(patch, [mutate])` — Constructor for patch object
- `JSONPointer(pathStr)` — Constructor for pointer object
- Error classes: `InvalidPatch`, `PatchApplyError`

### Architecture Highlights

**Compilation model**: Patch operations are "compiled" into closures at construction time. Each operation in the patch array is compiled into a function that captures the path, value, and `from` pointer. The compiled functions are then applied sequentially.

**Structural sharing (shallow clone)**: By default (`mutate=false`), the library creates **shallow copies** at each level of path traversal — not a deep clone of the entire document. This means it only copies the objects along the path being modified, leaving unaffected subtrees as shared references. This is a **structural sharing** approach similar to immutable-json-patch.

### Key Design Patterns

1. **JSONPointer as a class**: Parses path once at construction time, stores pre-split segments. Operations are methods on the pointer (`add`, `remove`, `replace`, `get`).

2. **`_action` method with handler callback**: All pointer operations share a common traversal method `_action(doc, handler, mutate)` that walks the path and calls the handler at the final segment. This is **elegant and DRY**.

3. **Shallow clone during traversal**: `_action` clones each node while walking the path (when `mutate=false`), and sets `node[segment] = subnode` (the cloned child) on the way back up the recursion. This creates a new path from root to target while keeping everything else shared.

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | Array splice, object assignment, root replacement |
| `remove` | ✅ | Array splice, delete, root → undefined |
| `replace` | ✅ | Array splice, object assignment, root replacement |
| `move` | ✅ | `from.get` → `from.remove` → `path.add` (with ancestor check) |
| `copy` | ✅ | `from.get` → `path.add` |
| `test` | ✅ | Custom `deepEqual` |

### Notable Behaviors

- **Immutable by default** with structural sharing — most memory-efficient approach for large documents with small patches
- **`replace` uses `splice(i, 1, value)` for arrays** — atomically replaces the element
- **`replace` requires existing value** — validates `hasOwnProperty` before replacing
- **`deepEqual` has a bug**: Uses `Object.hasOwnProperty(a, key)` (calls on `Object` constructor, not on instance via `Object.prototype.hasOwnProperty.call(a, key)`)
- **No prototype pollution protection**
- **No `-` append handling in `remove`**

### Security & Robustness

| Risk | Assessment |
| ---- | ---------- |
| Prototype pollution | ❌ **No protection** |
| Validation | ✅ `validateOp` checks op, path, required fields |
| Path not found | ✅ Throws `PatchApplyError` |
| Deep equality bug | ⚠️ `Object.hasOwnProperty(a, key)` instead of `Object.prototype.hasOwnProperty.call(a, key)` |

---

## 2. stdlib Relevance

### What Should Be Borrowed

| Concept | Why Useful |
| ------- | ---------- |
| **Structural sharing via shallow clone during traversal** | Most memory-efficient immutability approach. Only clones objects along the modified path, shares everything else. |
| **`_action` with handler callback** | Elegant DRY pattern — traversal logic written once, operations supply handlers. |
| **Compilation model** | Pre-parsing paths at patch creation time avoids re-parsing on every application. |
| **`subsetOf` for ancestor check** | Clean path prefix comparison for move validation. |

### What Should NOT Be Borrowed

| Approach | Why |
| -------- | --- |
| Class-based design (`JSONPatch`, `JSONPointer`) | stdlib avoids classes |
| `deepEqual` bug (`Object.hasOwnProperty(a, key)`) | Incorrect usage of `hasOwnProperty` |
| No prototype pollution protection | Critical security gap |
| `replace` using `splice` for arrays | Unnecessary — direct assignment works |

### Overall Rating: **Moderate**

**stdlib Fit**: Moderate. The structural sharing approach is excellent and the `_action` traversal pattern is clean. However, the class-based design, `deepEqual` bug, and missing security protections are problematic.

**Best for**: Structural sharing concept, `_action` traversal pattern.

---

# jsonpatch-js (bruth) — Technical Review

> Library: `json-patch` v0.7.0
> Repository: [bruth/jsonpatch-js](https://github.com/bruth/jsonpatch-js)
> Location in stdlib tree: `libraries/jsonpatch-js`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | jsonpatch-js (npm: `json-patch`) |
| **Primary language** | CoffeeScript (compiled to JavaScript) |
| **Module system** | UMD (CommonJS + AMD + global) |
| **Approximate implementation size** | Single file `jsonpatch.js` (~544 lines, generated from CoffeeScript) |
| **License** | BSD |
| **Runtime dependencies** | **None** |

### Public API

- `apply(document, patch)` — Apply patch (calls `compile` then executes)
- `compile(patch)` — Pre-compile patch into executable function
- `JSONPointer(path)` — Pointer class
- Error classes: `JSONPatchError`, `InvalidPointerError`, `InvalidPatchError`, `PatchConflictError`, `PatchTestFailed`

### Architecture Highlights

1. **CoffeeScript-generated**: The JavaScript is compiled from CoffeeScript, resulting in idiomatic CoffeeScript patterns (class hierarchy via `extend`, IIFE wrapping).

2. **Heavy class hierarchy**: Each operation type is a class (`AddPatch`, `RemovePatch`, `ReplacePatch`, `TestPatch`, `MovePatch`, `CopyPatch`) inheriting from a base `JSONPatch` class. Operations are dispatched via `operationMap[p.op]`.

3. **Compilation pattern**: Like jsonpatch.js (dharmafly), this library pre-compiles patches into closures.

4. **JSONPointer with `accessor` separation**: The pointer pre-parses the path into `steps` (parent segments) and `accessor` (final segment). `getReference` traverses to the parent; operations use `accessor` to act on the final key.

5. **Deep equality with circular reference detection**: `_isEqual` uses a stack to detect circular references — more robust than many implementations.

### Notable Behaviors

- **Mutates by default** — no immutability option
- **`CopyPatch` does NOT deep clone** — uses `slice(accessor, accessor+1)[0]` for arrays (shallow), direct reference for objects. **This is a bug** — RFC 6902 requires `copy` to be independent of the source.
- **`MovePatch` with same-path optimization** — if `from === path`, `apply` becomes a no-op (returns document unchanged)
- **Move ancestor check** — validates at construction time, not at application time
- **Error hierarchy**: 5 custom error classes with CoffeeScript-style inheritance

### Security & Robustness

| Risk | Assessment |
| ---- | ---------- |
| Prototype pollution | ❌ **No protection** |
| Validation | ✅ Per-operation validation |
| Deep equality | ✅ With circular reference detection |
| `copy` reference aliasing | ❌ Does NOT deep clone for objects |

---

## 2. stdlib Relevance

### What Should Be Borrowed

| Concept | Why Useful |
| ------- | ---------- |
| **`steps` + `accessor` pointer separation** | Pre-separating the final segment from the path enables clean operation handlers |
| **Circular reference detection in equality** | More robust than simple recursion |
| **Move same-path optimization** | Skip no-op moves |

### What Should NOT Be Borrowed

| Approach | Why |
| -------- | --- |
| CoffeeScript-generated code | Unreadable, non-idiomatic JS |
| Heavy class hierarchy | 6 operation classes — over-engineered for stdlib |
| Missing deep clone for `copy` | RFC violation |
| No prototype pollution protection | Security gap |
| Mutation-only (no immutability) | Too limiting |

### Overall Rating: **Poor**

**stdlib Fit**: Poor. CoffeeScript-generated code, heavy class hierarchy, and the `copy` bug (no deep clone) make this unsuitable. The `steps`/`accessor` pointer separation concept is the only noteworthy idea.

**Best for**: `steps`/`accessor` concept only.
