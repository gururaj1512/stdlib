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

# UltraPatch — Technical Review

> Library: `ultrapatch` v0.1.4
> Repository: [FluentCoding/UltraPatch](https://github.com/FluentCoding/UltraPatch)
> Location in stdlib tree: `libraries/UltraPatch`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | UltraPatch |
| **Primary language** | TypeScript |
| **Module system** | ES modules (`type: "module"`, dual CJS/ESM output) |
| **Approximate implementation size** | Single file `src/index.ts` (~358 lines) + utils |
| **License** | MIT |
| **Runtime dependencies** | **None** |
| **Build tool** | Bun |

### Public API

- `diff(origin, destination)` — Compute patch from two values
- `patch(target, operations)` — Apply operations in-place (no cloning)

### Key Design Characteristics

1. **Self-described as "naive"**: The library explicitly states it's "extremely fast (but naive)" — optimizes for speed by skipping validation entirely.

2. **Iterative (stack-based) diff**: Unlike recursive approaches, `diff` uses an explicit stack (`while (stack.length)`) to avoid recursion. Values are pushed/popped from a flat array (origin, destination, path triples). This prevents stack overflow on deep objects.

3. **No validation whatsoever**: The `patch` function **never validates** paths, operations, or values. It trusts the caller completely. The only "security" is the `test` operation.

4. **In-place mutation only**: No immutability option. No cloning.

5. **Uses `structuredClone` for copy**: Modern API — `structuredClone(fromValue)` for the `copy` operation. Handles more types than `JSON.parse(JSON.stringify())` but requires modern runtime.

6. **Simple array diffing**: For arrays, UltraPatch compares by index — if arrays have different lengths, it adds/removes from the end. For matching indices, it compares values recursively. **No LCS algorithm** — this produces non-optimal diffs for array reordering.

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | Array splice/push, object assignment |
| `remove` | ✅ | Array splice, delete |
| `replace` | ✅ | Direct assignment |
| `move` | ✅ | Inline remove + add (no decomposition to `applyOperation`) |
| `copy` | ✅ | `structuredClone` for objects, direct for primitives |
| `test` | ✅ | Custom `isEqual` |

### Security & Robustness

| Risk | Assessment |
| ---- | ---------- |
| Prototype pollution | ❌ **No protection** |
| Validation | ❌ **None** — completely trusts input |
| Path errors | ❌ Will throw TypeError/undefined access on invalid paths |
| `structuredClone` requirement | ⚠️ Not available in Node.js < 17 |

---

## 2. stdlib Relevance

### What Should Be Borrowed

| Concept | Why Useful |
| ------- | ---------- |
| **Stack-based iterative diff** | Avoids recursion, prevents stack overflow on deep objects |
| **`structuredClone` for copy** | Handles more types than JSON round-trip (but stdlib has `@stdlib/utils/copy`) |

### What Should NOT Be Borrowed

| Approach | Why |
| -------- | --- |
| No validation | Unsafe for production use |
| No prototype pollution protection | Security gap |
| In-place mutation only | Too limiting |
| `structuredClone` dependency | Not available in Node.js < 17; stdlib targets `>=0.10.0` |
| TypeScript + Bun | Incompatible with stdlib build toolchain |
| Naive array diffing | Non-optimal patches for reordered arrays |

### Overall Rating: **Poor**

**stdlib Fit**: Poor. The "naive" design philosophy (no validation, no safety) is fundamentally incompatible with stdlib's approach. TypeScript + Bun + ES modules + `structuredClone` requirement all conflict with stdlib conventions.

**Best for**: Stack-based diff concept (avoids recursion); nothing else is directly useful.

---

# JSON8 Patch — Technical Review

> Library: `json8-patch` v0.4.1
> Repository: [JSON8/patch](https://github.com/JSON8/patch)
> Location in stdlib tree: `libraries/patch`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | JSON8/patch (npm: `json8-patch`) |
| **Primary language** | JavaScript (ES2015+ with Babel) |
| **Module system** | CommonJS |
| **Approximate implementation size** | 19 files in `lib/` (~15KB total) |
| **License** | ISC |
| **Runtime dependencies** | `json8` ^0.9.0, `json8-pointer` ^0.7.1 ⚠️ |

### Public API

- `apply(doc, patch, [options])` — Apply patch with optional revert data
- `run(doc, patch)` — Apply single operation
- Individual: `add`, `remove`, `replace`, `move`, `copy`, `test`
- Utilities: `diff`, `revert`, `buildRevertPatch`, `pack`, `unpack`, `valid`, `compile`
- `get`, `has`, `walk` — Pointer utilities

### Key Design Characteristics

1. **Clean modular architecture**: Each operation is a separate file (`add.js`, `remove.js`, etc.) — very clean separation. The `apply.js` file dispatches via `operations[patch.op]`.

2. **Atomic patch application**: **Most notable feature** — `apply.js` implements **rollback on failure**. When an operation fails, it builds a revert patch from the operations already applied and rolls back the document. This is the **only library that implements true atomicity**.

3. **Revert/reversible patches**: The `options.reversible` flag causes `apply` to return revert data alongside the patched document. `buildRevertPatch` can convert this data into a reverse patch.

4. **External dependencies**: Depends on `json8` (type-checking utilities) and `json8-pointer` (JSON Pointer implementation). These are external npm packages from the same author — unlike stdlib which requires zero external deps.

5. **Pre-decoding paths**: Uses `json8-pointer.decode` to convert path strings to token arrays before dispatching.

### Supported Operations

| Operation | Supported | Notes |
| --------- | --------- | ----- |
| `add` | ✅ | With proper array/object handling |
| `remove` | ✅ | Returns previous value for revert |
| `replace` | ✅ | Returns previous value for revert |
| `move` | ✅ | Via remove + add |
| `copy` | ✅ | Via get + add |
| `test` | ✅ | Via json8 deep equality |

### Security & Robustness

| Risk | Assessment |
| ---- | ---------- |
| Prototype pollution | ⚠️ Depends on json8-pointer implementation |
| Validation | ✅ `valid.js` for operation validation |
| Atomicity | ✅ **Rollback on failure** — unique among all libraries |
| Error handling | ✅ Try/catch with revert on failure |

---

## 2. stdlib Relevance

### What Should Be Borrowed

| Concept | Why Useful |
| ------- | ---------- |
| **Atomic patch application with rollback** | The only library implementing true RFC 6902 atomicity. Essential for production safety. |
| **Revert patch generation** | Useful for undo/redo functionality |
| **Clean per-file operation modules** | Maps well to stdlib's package structure |
| **Pre-decoding paths to token arrays** | Clean separation of parsing from traversal |

### What Should NOT Be Borrowed

| Approach | Why |
| -------- | --- |
| External runtime dependencies (`json8`, `json8-pointer`) | stdlib requires zero external deps |
| Babel + ES2015 transpilation | Incompatible with stdlib build |
| Separate npm packages for core utilities | stdlib packages are self-contained |

### Overall Rating: **Moderate-Good**

**stdlib Fit**: Moderate. The modular architecture and atomic application pattern are excellent. The external dependencies are the main issue. The atomicity pattern (rollback on failure) is the single most important design idea not found in any other library.

**Best for**: Atomicity/rollback pattern, per-file module structure, revert patch generation.
