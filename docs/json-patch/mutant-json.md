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

# mutant-json — Technical Review

> Library: `mutant-json` v0.6.3
> Repository: [rubeniskov/mutant-json](https://github.com/rubeniskov/mutant-json)
> Location in stdlib tree: `libraries/mutant-json`

---

## 1. Repository Overview

| Property | Value |
| -------- | ----- |
| **Repository name** | mutant-json |
| **Primary language** | JavaScript |
| **Module system** | CommonJS |
| **Approximate implementation size** | `index.js` (~236 lines) + `constants.js`, `tap.js` |
| **License** | MIT |
| **Runtime dependencies** | `traverse-json` ^0.5.1, `is-promise` ^4.0.0, `jsonpatch` ^3.0.1 ⚠️ |

### Public API

- `mutantJson(target, process, [opts])` — Traverse a document and apply mutations via callback

### Key Characteristics

**This is NOT a JSON Patch implementation.** mutant-json is a **traversal-and-mutation framework** that _uses_ the `jsonpatch` library (dharmafly's jsonpatch.js) internally. It provides:

1. **Iterator-based traversal**: Walks the document using `traverse-json`, calling a user-supplied `process` callback for each value.
2. **Promise support**: Can handle async values (promises nested in the document).
3. **Callback-based mutation**: The `process` callback receives a `mutate` function that accepts patch operations.
4. **Delegates actual patching** to `jsonpatch.apply_patch`.

### Why This Doesn't Apply

mutant-json is fundamentally different from the other libraries:
- It's a **higher-level abstraction** over JSON Patch, not an implementation of it
- It depends on `jsonpatch` (dharmafly) for actual patch application
- It adds traversal, filtering, and async support on top
- It has **3 external runtime dependencies**

---

## 2. stdlib Relevance

### Overall Rating: **Not Applicable**

**stdlib Fit**: Not applicable. mutant-json is not a JSON Patch implementation — it's a traversal framework that wraps one. It has 3 external dependencies, uses async/Promise patterns extensively, and solves a different problem (document traversal with mutation callbacks) than RFC 6902 patch application.

**What to borrow**: Nothing directly. The traversal-based mutation concept is interesting but out of scope for the stdlib JSON Patch implementation.
