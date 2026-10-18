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

# JSON Patch & Pointer Evaluation and Research Guide

> Comprehensive research index, glossary, cross-library comparison, and evaluation of existing JSON Patch (RFC 6902) and JSON Pointer (RFC 6901) implementations to inform a stdlib-native implementation.

---

## Table of Contents

1. [Purpose & Context](#1-purpose--context)
2. [Individual Library Reviews & Repository Locations](#2-individual-library-reviews--repository-locations)
3. [Key stdlib Conventions](#3-key-stdlib-conventions)
4. [Glossary & Technical Concepts](#4-glossary--technical-concepts)
   - [Core Specifications](#core-specifications)
   - [Architecture & Dispatch Patterns](#architecture--dispatch-patterns)
   - [Pointer Traversal Patterns](#pointer-traversal-patterns)
   - [Immutability Strategies](#immutability-strategies)
   - [Security Concepts](#security-concepts)
   - [Algorithms](#algorithms)
   - [Module & Build Concepts](#module--build-concepts)
5. [Cross-Library Comparison Matrix](#5-cross-library-comparison-matrix)
6. [Best Algorithms by Category](#6-best-algorithms-by-category)
   - [JSON Pointer Parsing](#json-pointer-parsing)
   - [Operation Dispatch](#operation-dispatch)
   - [Deep Clone](#deep-clone)
   - [Immutability Strategy](#immutability-strategy)
   - [Atomicity](#atomicity)
   - [Security (Prototype Pollution)](#security-prototype-pollution)
7. [Best Ideas to Adopt & Anti-Patterns to Avoid](#7-best-ideas-to-adopt--anti-patterns-to-avoid)
   - [Best Ideas by Tier](#best-ideas-to-adopt-ordered-by-priority)
   - [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
8. [Individual Library Summaries & Verdicts](#8-individual-library-summaries--verdicts)
9. [Related Documents](#9-related-documents)

---

## 1. Purpose & Context

This directory contains structured technical reviews, algorithmic comparisons, and design evaluations of 9 existing JSON Patch (RFC 6902) and JSON Pointer (RFC 6901) libraries. The goal is to analyze existing implementations, identify best practices and performance bottlenecks, and architect an independent, stdlib-native implementation with zero external runtime dependencies.

### Target Use Case

The primary driving use case is supporting incremental Vega visualization specification updates in the browser over Server-Sent Events (SSE). Currently, entire Vega specifications are re-transmitted and re-rendered on each change; JSON Patch allows transmitting only minimal deltas and applying them immutably and efficiently without full editor re-initialization.

---

## 2. Individual Library Reviews & Repository Locations

### Review Documents

- [x] [Fast-JSON-Patch](./fast-json-patch.md) — `fast-json-patch` (Starcounter-Jack/JSON-Patch)
- [x] [jiff](./jiff.md) — `jiff` (cujojs/jiff)
- [x] [json-joy](./json-joy.md) — `json-joy` (streamich/json-joy)
- [x] [jsonpatch.js](./jsonpatch-js-mypatch.md) — `jsonpatch.js` (dharmafly/jsonpatch.js)
- [x] [jsonpatch-js](./jsonpatch-js.md) — `jsonpatch-js` (bruth/jsonpatch-js)
- [x] [UltraPatch](./ultra-patch.md) — `UltraPatch` (FluentCoding/UltraPatch)
- [x] [JSON8 Patch](./json8-patch.md) — `JSON8/patch` (JSON8/patch)
- [x] [mutant-json](./mutant-json.md) — `mutant-json` (rubeniskov/mutant-json)
- [x] [immutable-json-patch](./immutable-json-patch.md) — `immutable-json-patch` (josdejong/immutable-json-patch)

### Repository Locations

| Library | Directory | Primary Language | License |
| ------- | --------- | ---------------- | ------- |
| Fast-JSON-Patch | `libraries/JSON-Patch` | TypeScript | MIT |
| jiff | `libraries/jiff` | JavaScript | MIT |
| json-joy | `libraries/json-joy` | TypeScript | AGPL-3.0 |
| jsonpatch.js | `libraries/jsonpatch.js` | JavaScript | MIT |
| jsonpatch-js | `libraries/jsonpatch-js` | JavaScript / CoffeeScript | BSD |
| UltraPatch | `libraries/UltraPatch` | TypeScript | MIT |
| JSON8 Patch | `libraries/patch` | JavaScript | ISC |
| mutant-json | `libraries/mutant-json` | JavaScript | MIT |
| immutable-json-patch | `libraries/immutable-json-patch` | TypeScript | ISC |

---

## 3. Key stdlib Conventions

Any stdlib-native implementation must adhere to the following project conventions:

- **Module system**: CommonJS (`require`/`module.exports`)
- **Indentation**: Tabs
- **Strict mode**: `'use strict';` in every file
- **License header**: Apache-2.0 block in every file
- **Package structure**: `lib/` (source), `test/` (tests), `benchmark/`, `docs/`, `examples/`
- **Entry point**: `lib/index.js` re-exports `lib/main.js`
- **Testing**: `tape` + `chai.assert`
- **Benchmarking**: `@stdlib/bench`
- **Dependencies**: Only `@stdlib/*` packages; zero external runtime dependencies
- **Error handling**: `TypeError`/`RangeError` with `@stdlib/string/format` messages
- **Type checking**: `@stdlib/assert/is-*` family
- **Deep copy**: `@stdlib/utils/copy`
- **Deep equality**: `@stdlib/assert/deep-equal`
- **Node.js compatibility**: `>=0.10.0`
- **Naming**: `snake-case` for package directories, `camelCase` for functions/variables
- **API style**: Functional (no classes), positional arguments, explicit input validation

---

## 4. Glossary & Technical Concepts

### Core Specifications

#### JSON Patch (RFC 6902)
A standard format for expressing a sequence of operations to apply to a JavaScript Object Notation (JSON) document (`add`, `remove`, `replace`, `move`, `copy`, `test`). Instead of sending a full updated document, only the list of deltas is transmitted.

#### JSON Pointer (RFC 6901)
A string syntax for identifying a specific value within a JSON document. For example, `/data/values/0/name` targets the `name` property of the first element in the `values` array under `data`. Every JSON Patch operation uses a pointer in its `path` (and optionally `from`) property.

#### Pointer Escaping
Because `/` is used as a segment delimiter, literal `/` and `~` characters in object keys must be escaped:
- `~` is encoded as `~0`
- `/` is encoded as `~1`

When decoding, implementations **must** unescape `~1` &rarr; `/` first, then `~0` &rarr; `~`. Inverting the order results in corrupted strings (e.g. `~01` would erroneously decode to `/` instead of `~1`).

---

### Architecture & Dispatch Patterns

#### Hash-Table Dispatch
Instead of large `switch` blocks or `if/else` chains, operation handlers are stored in an object dictionary:
```js
var handlers = {
    'add': doAdd,
    'remove': doRemove,
    'replace': doReplace,
    'move': doMove,
    'copy': doCopy,
    'test': doTest
};
handlers[ op.op ]( doc, path, op.value );
```
- **Where used**: Fast-JSON-Patch, jiff, JSON8 Patch
- **Why it matters**: O(1) lookup, clean separation of concerns, easily extensible without modifying a central branching block.

#### Switch Dispatch
Direct `switch ( op.op )` statements. While JIT-optimizable, it couples all operation implementations into a single monolithic loop and makes adding custom operations rigid.
- **Where used**: json-joy (v3), UltraPatch, immutable-json-patch

#### Class-Based Dispatch
Each operation type is modeled as a class (`AddPatch`, `RemovePatch`, etc.) inheriting from an abstract base class.
- **Where used**: jsonpatch-js (bruth)
- **stdlib assessment**: Inappropriate and over-engineered for stdlib's functional paradigm.

#### Patch Compilation
Pre-compiling or pre-parsing a patch array into executable closures or pre-tokenized structures at construction time. This avoids repeated path parsing and syntax validation when applying the same patch across many documents.
- **Where used**: jsonpatch.js (dharmafly), jsonpatch-js (bruth)

---

### Pointer Traversal Patterns

#### `{ target, key }` Resolution
When resolving a pointer (e.g. `/a/b/c`), traversal halts at the **parent object** (`doc.a.b`) and returns both the parent container (`target`) and the final segment (`key` = `'c'`).
- **Where used**: jiff
- **Why it matters**: One traversal routine serves all operations: `target[key] = value` for add/replace, `delete target[key]` for remove, and reading `target[key]` for get/test.

#### `_action` with Handler Callback
A shared recursive traversal function walks the document once and executes a supplied callback at the final segment.
- **Where used**: jsonpatch.js (dharmafly)
- **Why it matters**: Elegant DRY pattern for tree walking.

#### Inline Pointer Parsing
Walking the pointer string directly in a loop using `indexOf('/')` without allocating intermediate token arrays.
- **Where used**: json-joy (v3)
- **Why it matters**: Zero allocations; high raw throughput.

---

### Immutability Strategies

#### In-Place Mutation
Modifies the document directly in memory.
- **Time**: O(1) per operation
- **Space**: O(0)
- **Trade-off**: Destructive; cannot recover original document on error or perform change comparisons.

#### Full Deep Clone + Mutate
Deeply clones the entire document before applying modifications in-place on the clone.
- **Time**: O(n) where n is total document size
- **Space**: O(n)
- **Trade-off**: Non-destructive, but prohibitive for large documents (e.g. 10k-line Vega specs) with tiny single-property edits.

#### Structural Sharing
Only shallow-clones the objects **along the path of modification**. Unmodified subtrees maintain shared object references with the original document.
- **Time**: O(d) where d is path depth (typically 3–5 levels)
- **Space**: O(d) new objects created
- **Safety**: Fully immutable; original document is never mutated
- **Where used**: immutable-json-patch, jsonpatch.js (dharmafly)
- **stdlib recommendation**: Primary strategy for stdlib immutable mode.

---

### Security Concepts

#### Prototype Pollution
A vulnerability where untrusted JSON Patch payloads contain paths targeting `__proto__`, `constructor`, or `prototype` (e.g. `{"op": "add", "path": "/__proto__/isAdmin", "value": true}`). If unblocked, properties are injected onto `Object.prototype`, affecting all objects in the runtime.
- **Where used**: Only Fast-JSON-Patch includes explicit protection among the 9 libraries reviewed.
- **stdlib requirement**: Mandatory non-optional blocklist for `__proto__`, `constructor`, and `prototype` in JSON Pointer resolution.

#### Reference Aliasing
Occurs when `add` or `copy` operations insert an object reference directly from the payload or from another part of the document without cloning. Subsequent mutations to that object inadvertently mutate both locations.
- **Fix**: Deep clone values on `copy` (RFC requirement) and clone values on `add`/`replace` in immutable mode.

---

### Algorithms

#### LCS (Longest Common Subsequence) Diffing
An algorithm used during patch generation (`diff`) to find the longest sequence of elements common to two arrays. Allows generating minimal `add` and `remove` operations rather than replacing the entire array.
- **Where used**: jiff (with prefix/suffix optimization)
- **Relevance**: Crucial for future `diff()` implementation.

#### Deep Equality
Structural comparison across primitive types, arrays, nested objects, and edge cases (NaN, circular references). Required by the RFC `test` operation.
- **stdlib solution**: Use `@stdlib/assert/deep-equal`.

#### Atomicity / Rollback
RFC 6902 specifies that patch application must be atomic: if any operation fails, the entire patch must have no effect.
- **Where used**: JSON8 Patch implements true rollback by recording inverse steps and applying a revert patch upon encountering an error.
- **Note**: With structural sharing, atomicity is inherently guaranteed because the original document is untouched.

---

### Module & Build Concepts

#### UMD (Universal Module Definition)
Enables compatibility across CommonJS, AMD, and browser globals in a single file. (Used in older libraries like jsonpatch.js, jsonpatch-js).

#### CommonJS (CJS)
Standard synchronous module system (`require` / `module.exports`) used natively by stdlib.

#### ES Modules (ESM)
Modern JavaScript `import` / `export` syntax. (Used by UltraPatch, immutable-json-patch, json-joy).

---

## 5. Cross-Library Comparison Matrix

| Library | Lang | Deps | License | In-Place Mutation | Immutable Clone | Structural Sharing | Prototype Pollution Guard | Atomic Rollback | Array Diffing | stdlib Fit |
| ------- | ---- | ---- | ------- | :---------------: | :-------------: | :----------------: | :-----------------------: | :-------------: | :-----------: | :--------: |
| **Fast-JSON-Patch** | TS | 0 | MIT | ✅ (Default) | ✅ (Full clone) | ❌ | ✅ | ❌ | ✅ | **Moderate** |
| **jiff** | JS | 0 | MIT | ✅ (`patchInPlace`) | ✅ (Full clone) | ❌ | ❌ | ❌ | ✅ (LCS) | **Good** |
| **json-joy** | TS | 3 | **AGPL-3.0** | ✅ (Option) | ✅ (Option) | ❌ | ⚠️ | ❌ | ❌ | **Poor** |
| **jsonpatch.js** | JS | 0 | MIT | ✅ (Option) | ❌ | ✅ (Shallow) | ❌ | ❌ | ❌ | **Moderate** |
| **jsonpatch-js** | CS &rarr; JS | 0 | BSD | ✅ Only | ❌ | ❌ | ❌ | ❌ | ❌ | **Poor** |
| **UltraPatch** | TS | 0 | MIT | ✅ Only | ❌ | ❌ | ❌ | ❌ | ✅ | **Poor** |
| **JSON8 Patch** | JS | 2 | ISC | ✅ | ❌ | ❌ | ⚠️ | ✅ **(Rollback)** | ✅ | **Moderate** |
| **mutant-json** | JS | 3 | MIT | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | **N/A** |
| **immutable-json-patch** | TS | 0 | ISC | ❌ | ❌ | ✅ **(Full)** | ❌ | ❌ | ❌ | **Good (Algo)** |

---

## 6. Best Algorithms by Category

### JSON Pointer Parsing

| Approach | Library | Complexity | Allocations | Assessment |
| -------- | ------- | ---------- | ----------- | ---------- |
| `path.split('/') + replace` | Fast-JSON-Patch | O(n) | Intermediate array + strings | Simple, easy to maintain |
| Regex single-pass (`\/\|~1\|~0`) | jiff | O(n) | Zero array allocations | **Best balance**: streaming, clean |
| Inline `indexOf` loop | json-joy (v3) | O(n) | Zero allocations | **Fastest raw performance** |
| Pre-parsed split | jsonpatch.js | O(n) | Parsed once at compile time | Ideal for repeated patch execution |

**Recommendation for stdlib**: Use single-pass parsing with early syntax validation and support pre-parsed token arrays for reusable patches.

---

### Operation Dispatch

| Approach | Library | Lookup Time | Extensibility | Assessment |
| -------- | ------- | ----------- | ------------- | ---------- |
| Hash table (`ops[op].call(...)`) | Fast-JSON-Patch | O(1) | High | Clean, but uses `this` binding |
| Hash table (`patches[op].apply(...)`) | jiff | O(1) | High | **Best**: pure functional, no `this` |
| `switch (op)` | json-joy, UltraPatch | O(1) | Low | Fast JIT compilation, but rigid |
| Class hierarchy | jsonpatch-js | O(1) + alloc | Moderate | Over-engineered for functional standard library |

**Recommendation for stdlib**: Hash-table dispatch mapping operation names to pure handler functions without `this` contexts.

---

### Deep Clone

| Approach | Library | Type Fidelity | Performance | Assessment |
| -------- | ------- | ------------- | ----------- | ---------- |
| `JSON.parse(JSON.stringify())` | Fast-JSON-Patch | Lossy (`Date`, `RegExp`, `undefined`) | Fast | Acceptable only for strict JSON |
| Recursive object copier | jiff | High | Moderate | Accurate |
| `structuredClone` | UltraPatch | Full | Native | Incompatible with Node < 17 |
| `@stdlib/utils/copy` | stdlib Native | Full | Optimized | **Recommended stdlib choice** |

**Recommendation for stdlib**: Leverage existing `@stdlib/utils/copy` for complete type preservation and compatibility.

---

### Immutability Strategy

| Strategy | Library | Time per Op | Space per Op | Safety | Assessment |
| -------- | ------- | ----------- | ------------ | ------ | ---------- |
| In-place mutation | Fast-JSON-Patch | O(1) | O(0) | ❌ Unsafe | Fastest; destroys source document |
| Full clone + mutate | Fast-JSON-Patch, jiff | O(n) | O(n) | ✅ Safe | Inefficient for large documents |
| Structural sharing | immutable-json-patch, jsonpatch.js | O(d) | O(d) | ✅ Safe | **Optimal for large Vega specs** |

**Recommendation for stdlib**: Implement structural sharing as the default immutable mode (`setIn`, `deleteIn`, `applyProp`), with an optional `mutate: true` flag for maximum throughput.

---

### Atomicity

| Approach | Library | Behavior on Failure | Assessment |
| -------- | ------- | ------------------- | ---------- |
| No rollback | Most libraries | Partially modified document | ❌ Corrupts state |
| Clone first | jiff, Fast-JSON-Patch | Original intact; clone discarded | ⚠️ Safe, but memory heavy |
| **Active Rollback** | JSON8 Patch | Generates & executes revert patch | ✅ **Best for in-place mode** |

**Recommendation for stdlib**: With structural sharing, atomicity is automatic (original is never modified). For in-place mutation mode, apply JSON8-style rollback on caught exceptions.

---

### Security (Prototype Pollution)

| Approach | Library | Assessment |
| -------- | ------- | ---------- |
| Block `__proto__`, `constructor`, `prototype` | Fast-JSON-Patch | **Only secure implementation** |
| No checks | All other 8 libraries | Vulnerable to prototype poisoning |

**Recommendation for stdlib**: Enforce non-optional blocklist validation during pointer segment evaluation.

---

## 7. Best Ideas to Adopt & Anti-Patterns to Avoid

### Best Ideas to Adopt (Ordered by Priority)

#### Tier 1: Must-Have
1. **Prototype pollution protection** (*Fast-JSON-Patch*): Mandatory security blocklist for `__proto__`, `constructor`, `prototype`.
2. **Hash-table operation dispatch** (*Fast-JSON-Patch, jiff*): Clean, extensible O(1) function map.
3. **`{ target, key }` pointer resolution** (*jiff*): Cleanly decouples tree traversal from operation logic.
4. **RFC 6901 compliant unescaping order** (*Fast-JSON-Patch, jiff*): Always `~1` &rarr; `/` prior to `~0` &rarr; `~`.
5. **Root operation special casing** (*Fast-JSON-Patch*): O(1) handling for root path `""`.
6. **Move ancestor validation** (*jiff, json-joy*): Verify target path is not a child of source path (`to.indexOf(from + '/') !== 0`).
7. **Compliance test suite integration** (*Fast-JSON-Patch, jiff*): Test against official `json-patch-test-suite`.

#### Tier 2: Should-Have
8. **Structural sharing for immutability** (*immutable-json-patch, jsonpatch.js*): O(d) updates preserving unmodified subtrees.
9. **Value cloning on `add`/`replace`** (*jiff*): Prevent accidental object reference aliasing.
10. **Standalone validation API** (*Fast-JSON-Patch, json-joy*): Ability to validate patch structure prior to execution.
11. **Rollback-based atomicity** (*JSON8 Patch*): Complete error recovery for in-place operations.
12. **Deep clone for `copy`** (*Fast-JSON-Patch, jiff*): Strict RFC conformance ensuring independent copy instances.

#### Tier 3: Nice-to-Have
13. **LCS-based array diffing** (*jiff*): Prefix/suffix trimmed Myers-like diffing for patch generation.
14. **Reversible patch generation** (*JSON8 Patch, immutable-json-patch*): Generate inverse patches for undo/redo workflows.
15. **Shared `_action` traversal** (*jsonpatch.js*): Unified recursive walker with operation delegates.

---

### Anti-Patterns to Avoid

1. **`this`-context binding for dispatch** (*Fast-JSON-Patch*): Creates hidden state coupling and breaks functional purity.
2. **Positional boolean parameters** (*Fast-JSON-Patch*): `applyPatch(doc, patch, true, false)` leads to unreadable call sites.
3. **Lossy JSON serialization for cloning** (*Fast-JSON-Patch*): `JSON.parse(JSON.stringify())` drops non-JSON types.
4. **Class hierarchies for standard operations** (*jsonpatch-js, json-joy*): Unnecessary OOP abstraction for 6 atomic functions.
5. **Disabled-by-default validation** (*Fast-JSON-Patch*): Unsafe for untrusted network inputs.
6. **DOM event integration in core library** (*Fast-JSON-Patch duplex*): Pollutes pure data manipulation libraries with browser UI concerns.
7. **External runtime dependencies** (*JSON8 Patch, mutant-json*): Violates stdlib zero-dependency policy.
8. **Transpiled CoffeeScript output** (*jsonpatch-js*): Unmaintainable runtime output.
9. **JIT runtime code generation** (*json-joy*): Excessive complexity unsuitable for standard library core utilities.
10. **Copyleft licensing (AGPL-3.0)** (*json-joy*): Legally incompatible with Apache-2.0.

---

## 8. Individual Library Summaries & Verdicts

### Fast-JSON-Patch
The most widely used JSON Patch library (5M+ weekly downloads) and the only one with prototype pollution protection. Its hash-table dispatch and pointer traversal are well-architected. However, it uses TypeScript, `this`-binding for dispatch, and `JSON.parse(JSON.stringify())` for cloning. Best used as an **architecture and security reference**.

### jiff
The most stdlib-compatible library in code style — plain JavaScript, CommonJS, functional design, zero dependencies. Its single-pass pointer parser and `{ target, key }` resolution pattern are elegant and efficient. It also includes an LCS-based array diff algorithm. The main gap is missing prototype pollution protection. Best used as the **primary style and module design reference**.

### json-joy
A massive TypeScript monorepo (50+ packages) covering JSON Patch, JSON CRDTs, and collaborative editing. Highly optimized with JIT code generation and inline pointer parsing. However, its **AGPL-3.0 license is incompatible with stdlib's Apache-2.0**, and the 33-class hierarchy is extreme overkill. **Ruled out for code reuse; reference for conceptual algorithms only**.

### jsonpatch.js (dharmafly)
A clean, single-file implementation that defaults to immutability via **structural sharing** — shallow-cloning only along the modified path. Its `_action` traversal pattern is elegantly DRY. Downsides include a bug in `deepEqual` (`Object.hasOwnProperty` call), missing prototype pollution protection, and class-based design. Best used as a **structural sharing reference**.

### jsonpatch-js (bruth)
Generated from CoffeeScript, resulting in verbose JavaScript with a 6-class inheritance hierarchy. The `copy` operation does **not** deep clone values, violating RFC 6902. Its only notable idea is separating the pointer into `steps` (parents) + `accessor` (final key) at parse time. **Ruled out due to code quality and RFC non-compliance**.

### UltraPatch
Self-described as "extremely fast (but naive)" — it achieves speed by skipping **all validation**, causing invalid paths to crash with uncaught TypeErrors. Requires TypeScript, Bun, and `structuredClone` (Node &ge;17). Its iterative stack-based diff avoids recursion. **Ruled out due to its unsafe "no validation" design**.

### JSON8 Patch
The **only library that implements true atomic rollback**: when an operation fails mid-patch, it builds a revert patch and restores the document state. Features a clean per-file module structure (`add.js`, `remove.js`, etc.). The downside is its 2 external runtime dependencies (`json8`, `json8-pointer`). Best used as the **atomicity pattern reference**.

### mutant-json
**Not a JSON Patch implementation** — it is a document traversal and mutation wrapper built on top of dharmafly's `jsonpatch` library with async/Promise handling and 3 runtime dependencies. **Ruled out as out of scope**.

### immutable-json-patch
The most sophisticated immutability implementation reviewed: **structural sharing** via `setIn`/`deleteIn`/`applyProp` primitives that only clone objects along the modified path. Includes revert patch generation and `before`/`after` middleware hooks with zero dependencies. The main drawbacks are TypeScript/ESM and lack of deep cloning for `copy`. Best used as the **primary reference for immutable operations**.

---

## 9. Related Documents

- [JSON Pointer Implementation Approach](./json-pointer-approach.md) — Detailed engineering specification for `@stdlib/json/pointer` (RFC 6901).
- [Mentor 1:1 Talking Points](./mentor_talking_points.md) — Concise summary and roadmap script for mentor syncs.
- [Proposed stdlib Architecture](./proposed-design.md) — *(Upcoming)* End-to-end package design and API blueprints for `@stdlib/json/patch`.
