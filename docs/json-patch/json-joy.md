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

# json-joy — Technical Review

> Library: `json-joy` v18.28.0
> Repository: [streamich/json-joy](https://github.com/streamich/json-joy)
> Location in stdlib tree: `libraries/json-joy`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | json-joy (monorepo) |
| **Primary language** | TypeScript |
| **Module system** | ES modules (TypeScript, compiled to CommonJS + ESM) |
| **Approximate implementation size** | Monorepo with 50+ packages. JSON Patch: `packages/json-joy/src/json-patch/` (~17 files, core `applyPatch/v3.ts` ~146 lines). JSON Pointer: `packages/json-pointer/` (~6 source files). Extremely large total codebase (~50+ packages). |
| **License** | **AGPL-3.0-only** ⚠️ |

### Public API (JSON Patch subset)

- `applyOperation(doc, operation)` — apply a single operation (returns `{ doc, old }`)
- `applyPatch(doc, patch, options)` — apply a patch array (mutate option)
- `validateOperations(ops)` — validate operation array
- `validateOperation(op)` — validate single operation
- Operation classes: `OpAdd`, `OpRemove`, `OpReplace`, `OpMove`, `OpCopy`, `OpTest`, plus many extensions

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | Object and array |
| `remove` | ✅ | Object and array |
| `replace` | ✅ | Object and array |
| `move` | ✅ | Decomposes to remove + add |
| `copy` | ✅ | Deep clones via external package |
| `test` | ✅ | Deep equality via external package |

### Extensions Beyond RFC 6902

json-joy supports a vast set of extensions:

- **JSON Predicate** (draft-snell-json-test-01): `contains`, `defined`, `ends`, `in`, `less`, `matches`, `more`, `starts`, `type`, `undefined`, `and`, `or`, `not`
- **Extended operations**: `flip`, `inc`, `str_ins`, `str_del`, `split`, `merge`, `extend`
- **Extended test**: `test_type`, `test_string`, `test_string_len`

### Runtime/Browser Support

- Node.js ≥10.0
- Browser (via ESM)

### Build Requirements

- TypeScript, Yarn, Biome (formatter/linter)
- Monorepo managed with Yarn workspaces

### External Dependencies

- **Runtime**: `@jsonjoy.com/util`, `@jsonjoy.com/json-pointer`, `@jsonjoy.com/json-equal` (all from the same monorepo, published as separate npm packages)
- **Development**: TypeScript, Biome, Jest, etc.

### License Concern

> ⚠️ **AGPL-3.0-only** — This is a copyleft license that requires derivative works to also be AGPL-licensed. This is **incompatible with stdlib's Apache-2.0 license**. Code from json-joy cannot be directly reused, even as reference for implementation patterns. Only general algorithmic ideas (which are not copyrightable) can be studied.

---

## 2. JSON Patch Specification Compliance

### JSON Pointer (RFC 6901)

| Behavior | Status | Evidence |
| -------- | ------ | -------- |
| `~0` → `~` decoding | ✅ | `util.ts`: `component.replace(r1, '/').replace(r2, '~')` |
| `~1` → `/` decoding | ✅ | Same function, correct order |
| Early exit optimization | ✅ | `if (component.indexOf('~') === -1) return component` |
| Empty pointer | ✅ | `parseJsonPointer`: returns `[]` for empty string |
| Array `-` handling | ✅ | `find.ts`: `key === '-'` → `key = length` |
| Invalid array index | ✅ | `~~key` coercion + string round-trip check |

### Operation Analysis

#### `add`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object: `obj[key] = value` | ✅ | Direct assignment |
| Array: `splice` or `push` | ✅ | `splice` for index < length, `push` for end |
| Root: `doc = operation.value` | ✅ | Returns `{ doc: operation.value, old: doc }` |

#### `remove`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object: `delete obj[key]` | ✅ | Standard |
| Array: `splice(key, 1)` | ✅ | Standard |
| Returns old value | ✅ | `{ doc, old }` |

#### `replace`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Object/Array: `obj[key] = value` | ✅ | Direct assignment |
| Returns old value | ✅ | `{ doc, old }` |

#### `move`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Decomposition | ✅ | `applyOperation(doc, {op: 'remove', ...})` then `applyOperation(result.doc, {op: 'add', ...})` |
| Ancestor check | ✅ | Validated in `validateOperationMove`: `path.indexOf(from + '/') === 0` |
| Uses removed value as add value | ✅ | `removeResult.old` becomes add value |

#### `copy`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Deep clone | ✅ | `deepClone(val)` from `@jsonjoy.com/util` |

#### `test`

| Behavior | Status | Notes |
| -------- | ------ | ----- |
| Deep equality | ✅ | `deepEqual` from `@jsonjoy.com/util` |
| Throws on failure | ✅ | `throw new Error('TEST')` |

### Atomicity

- **Not atomic.** Operations applied sequentially; no rollback.

---

## 3. Architecture Analysis

```
json-joy (monorepo)
├── packages/json-pointer/ — JSON Pointer implementation
│   ├── find.ts — Traverse to pointer (returns { val, obj, key })
│   ├── findByPointer/ — String-based find (parses then calls find)
│   ├── util.ts — escape/unescape, parse, format, isChild
│   ├── validate.ts — Pointer validation
│   ├── get.ts — Simple value getter
│   └── codegen/ — JIT-compiled pointer accessors
│
├── packages/json-equal/ — Deep equality
│
├── packages/util/ — Shared utilities (clone, hasOwnProperty)
│
└── packages/json-joy/src/json-patch/ — JSON Patch
    ├── applyPatch/
    │   ├── v1.ts — Basic implementation
    │   ├── v2.ts — Improved version
    │   ├── v3.ts — Main implementation (inline operations)
    │   └── v4.ts — Class-based implementation
    ├── op/ — Operation classes (33 files!)
    │   ├── OpAdd.ts, OpRemove.ts, OpReplace.ts, ...
    │   ├── OpTest.ts, OpDefined.ts, OpUndefined.ts, ...
    │   ├── OpStrIns.ts, OpStrDel.ts, OpSplit.ts, ...
    │   └── AbstractOp.ts, AbstractPredicateOp.ts
    ├── validate.ts — Comprehensive operation validation
    ├── opcodes.ts — Operation type definitions
    ├── types.ts — TypeScript type definitions
    ├── codec/ — Serialization/deserialization
    ├── codegen/ — JIT code generation for patches
    └── exec/ — Execution engine
```

### Key Design Decisions

1. **Multiple implementation versions** (`v1`-`v4`): The library iterates on implementation approaches. `v3` is the main one — a monolithic function with inline operation dispatch via `switch`.

2. **Inline everything in `v3`**: Unlike Fast-JSON-Patch's hash dispatch, json-joy's `v3` inlines all operations directly in the traversal loop. This is **more performant** (no function call overhead, no `this` binding) but **less maintainable**.

3. **JIT code generation**: The `codegen/` directory contains JIT-compiled pointer accessors and patch applicators — the library can generate optimized JavaScript code at runtime for specific pointer patterns. This is highly advanced but completely unsuitable for stdlib.

4. **Extensive type system**: Every operation has its own TypeScript class with serialization methods. This is far more than RFC 6902 requires.

5. **External packages for core utilities**: `json-pointer`, `json-equal`, `util` are separate packages within the monorepo.

---

## 4. Algorithm Analysis

### JSON Pointer (v3 inline)

- **Parsing**: Not using `parseJsonPointer` — instead parses inline via `indexOf('/')` in a `while` loop. No intermediate array allocation.
- **This is the most performant pointer traversal approach reviewed** — zero allocation, no regex, single pass.
- `unescapeComponent` is called only for object keys (arrays use numeric coercion).

### `add` (v3 inline)

- Array: `splice` for index < length, `push` otherwise.
- Object: Direct assignment.
- **Does NOT clone the value** before insertion — reference aliasing is possible.

### `move` (v3)

- Decomposes to `applyOperation(doc, {op: 'remove', ...})` + `applyOperation(result.doc, {op: 'add', ...})`.
- Uses `removeResult.old` as the value to add — no unnecessary cloning.

### `copy` (v3)

- Uses `deepClone` from external package — prevents reference aliasing.

### `test` (v3)

- Uses `deepEqual` from `@jsonjoy.com/util` — separate package.

### Deep Clone

- From `@jsonjoy.com/util/lib/json-clone/clone` — recursive implementation (not JSON round-trip).

---

## 5. Mutation vs Immutability

| Aspect | Behavior |
| ------ | -------- |
| Default | Depends on `options.mutate` flag |
| Mutation | When `options.mutate = true`, modifies document in-place |
| Immutable | When `options.mutate = false`, deep clones first |
| Clone strategy | Recursive `deepClone` (not JSON round-trip) |
| Value cloning | ⚠️ `add`/`replace` do **NOT** clone values — only `copy` does |

---

## 6. Dependency Analysis

### Runtime Dependencies

| Dependency | Purpose | stdlib-compatible? |
| ---------- | ------- | ------------------ |
| `@jsonjoy.com/util` | Clone, hasOwnProperty, deepEqual | ❌ External package (from same monorepo) — must replace with `@stdlib/*` utilities |
| `@jsonjoy.com/json-pointer` | JSON Pointer parsing, traversal | ❌ External package — must implement internally |
| `@jsonjoy.com/json-equal` | Deep equality | ❌ External package — use `@stdlib/assert/deep-equal` |

**Verdict**: The library has significant external runtime dependencies (3 packages from the same author's monorepo). All would need to be replaced with stdlib equivalents.

---

## 7. stdlib Compatibility Assessment

### Code Style

| Criterion | Assessment |
| --------- | ---------- |
| Language | ❌ **TypeScript** — incompatible with stdlib |
| Module system | ❌ **ES modules** — stdlib uses CommonJS |
| Architecture | ❌ **Massive monorepo** — 50+ packages, far beyond stdlib scope |
| Naming | ✅ camelCase functions |
| Operation classes | ❌ **33 operation classes** — massive over-abstraction |
| JIT codegen | ❌ **Completely inappropriate** for stdlib |
| Validation | ✅ Thorough, well-structured |

### API Style

| Criterion | Assessment |
| --------- | ---------- |
| Functional | ⚠️ Mixed — `v3` is functional, but `op/` uses heavy class hierarchy |
| Options object | ✅ `ApplyPatchOptions` with `mutate` flag |
| Return value | ✅ `{ doc, old }` — informative |

### Overall Rating: **Poor**

While json-joy is technically impressive and likely the most feature-rich implementation, it is fundamentally incompatible with stdlib in almost every dimension: TypeScript, ES modules, massive monorepo, heavy class hierarchy, JIT code generation, external dependencies, and most critically, **AGPL-3.0 license**.

---

## 8. What Should Be Borrowed

| Concept | Source | Why Useful | stdlib Adaptation |
| ------- | ------ | ---------- | ----------------- |
| **Inline pointer parsing via `indexOf`** | `v3.ts` | Zero-allocation traversal — most performant approach | Study the `indexOf` loop pattern for potential optimization |
| **`{ val, obj, key }` reference** | `find.ts` | More informative than `{ target, key }` — includes the resolved value | Consider including `val` in stdlib's reference |
| **Move ancestor validation** | `validateOperationMove` | `path.indexOf(from + '/') === 0` — simple string check | Adopt this pattern |
| **Validation separation** | `validate.ts` | Clean per-operation validation functions | Structure stdlib validation similarly |

> ⚠️ **License caveat**: Due to AGPL-3.0 licensing, only general algorithmic concepts (not copyrightable) should be referenced. No code patterns, class structures, or API designs should be borrowed.

---

## 9. What Should NOT Be Borrowed

| Approach | Why to Avoid |
| -------- | ------------ |
| **AGPL-3.0 licensed code** | Incompatible with Apache-2.0; cannot reference specific implementation patterns |
| **Monorepo architecture** | stdlib is a single repo; separate pointer/equal/util packages add complexity |
| **TypeScript + ES modules** | stdlib uses plain JavaScript + CommonJS |
| **33 operation classes** | Massive over-abstraction for 6 RFC operations |
| **JIT code generation** | Inappropriate for a standard library |
| **JSON Predicate extensions** | Beyond RFC 6902 scope; adds complexity without clear benefit for stdlib |
| **`str_ins`/`str_del`/`split`/`merge`/`extend`** | Non-standard, application-specific extensions |
| **External dependencies** | 3 separate npm packages for core functionality |
| **No value cloning in `add`/`replace`** | Reference aliasing risk — `copy` clones but `add` doesn't |
| **v1/v2/v3/v4 multiple implementations** | Confusing; stdlib needs one canonical implementation |

---

## 10. Testing Analysis

### Test Structure

- Tests use Jest/Vitest
- Community JSON Patch test suite runner exists
- Extensive tests for extended operations (predicate, str_ins, etc.)
- Tests for codegen/JIT path

### Notable

- The library runs RFC 6902 compliance tests via CLI tools
- Tests cover the 6 standard operations plus all extensions
- Automated fuzzing tests exist for JSON CRDT (different scope)

---

## 11. Performance Analysis

### Architecture-Level

| Aspect | Assessment |
| ------ | ---------- |
| Pointer traversal | ✅ **Excellent** — `v3` inline `indexOf` parsing, zero allocation |
| Operation dispatch | ✅ **Excellent** — `v3` uses `switch` (JIT-optimizable) |
| JIT codegen | ✅ **Excellent** performance, but inappropriate for stdlib |
| Clone | ✅ Recursive (not JSON round-trip) |

**Not benchmarked** by this review.

---

## 12. Security and Robustness

| Risk | Assessment |
| ---- | ---------- |
| Prototype pollution | ⚠️ No explicit `__proto__` check visible in `v3.ts`. Uses `hasOwnProperty` for traversal. |
| Deep recursion | ⚠️ No depth limit |
| Validation | ✅ Comprehensive in `validate.ts` |
| Error messages | ⚠️ Minimal — generic string errors like `'TEST'`, `'NOT_FOUND'`, `'INVALID_INDEX'` |

---

## 13. Final Library Verdict

### Strengths

1. **Most feature-rich implementation** — covers RFC 6902, JSON Predicate, and custom extensions
2. **Highly optimized** — inline pointer parsing, JIT codegen
3. **Thorough validation** — per-operation validation functions
4. **Multiple implementation approaches** — v1 through v4 show architectural exploration
5. **Active development** — version 18.x, actively maintained

### Weaknesses

1. **AGPL-3.0 license** — **incompatible with Apache-2.0**, cannot reference specific patterns
2. **Massive complexity** — 50+ packages, 33 operation classes
3. **TypeScript + ES modules** — incompatible with stdlib
4. **External dependencies** — 3 runtime packages
5. **Over-engineered** — JIT codegen, JSON CRDT, collaborative editing far beyond JSON Patch
6. **No value cloning in `add`/`replace`** — reference aliasing risk
7. **Generic error messages** — not informative

### Best Ideas

1. Inline `indexOf`-based pointer traversal (general algorithmic concept)
2. Move ancestor validation via string prefix check
3. Thorough per-operation validation structure

### stdlib Fit: **Poor**

The AGPL-3.0 license alone disqualifies json-joy as a reference implementation. Beyond licensing, the TypeScript monorepo architecture, class hierarchy, JIT code generation, and external dependencies are all incompatible with stdlib.

### Recommendation: **Reference only for specific algorithms**

Due to AGPL-3.0 licensing constraints, json-joy should only be referenced for **general algorithmic concepts** (which are not copyrightable):
- The idea of inline pointer parsing without array allocation
- The idea of move ancestor validation via string prefix
- The idea of per-operation validation functions

No specific code patterns, API designs, or architectural decisions should be borrowed.
