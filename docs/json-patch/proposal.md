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

# Adding JSON Patch Support to stdlib

## What I've Done So Far

I went through nine existing JSON Patch libraries, read their source code, and wrote detailed technical reviews for each one. The goal was to understand how people actually implement this — what works, what doesn't, what patterns we should follow, and what mistakes to avoid.

Out of the nine libraries, four turned out to be genuinely useful references:

**jiff** is the closest match to how stdlib code looks — plain JavaScript, CommonJS, zero dependencies, functional style. It has a clean way of resolving JSON Pointers where the resolver gives you back the parent object and the final key, so the operation code is just one line like `parent[key] = value`. We should follow this pattern.

**Fast-JSON-Patch** is the most popular one (5M+ weekly downloads) and it's the only library that protects against prototype pollution — where an attacker could craft a path like `/__proto__/isAdmin` to inject properties onto every object in the runtime. Every other library we looked at is vulnerable to this. We absolutely need this protection.

**immutable-json-patch** has the smartest approach to immutability. Instead of cloning the entire document before making changes (which is slow for big Vega specs), it only copies the few objects along the path being modified. Everything else stays untouched. So changing one property in a huge spec creates maybe 3-4 new objects instead of copying the whole thing. This is exactly what we need for our use case.

**JSON8 Patch** is the only library that implements true atomic rollback — if one operation in a patch fails, it actually undoes everything it already applied, leaving the document exactly as it was. The spec says patches should be all-or-nothing, and this is the only library that actually does that.

The other five were ruled out for clear reasons: json-joy has an AGPL license (incompatible with Apache-2.0), jsonpatch-js is generated from CoffeeScript and has RFC violations, UltraPatch skips all validation, jsonpatch.js has bugs in its equality check, and mutant-json isn't even a JSON Patch implementation — it's a wrapper around another library.

All the detailed reviews, a cross-library comparison matrix, and a glossary of technical concepts are documented in the `docs/json-patch/` directory.

## How We Should Build It

We should build this as two separate stdlib packages — one for JSON Pointer and one for JSON Patch — since JSON Pointer is useful on its own and is the foundation everything else depends on.

### Step 1: JSON Pointer (`@stdlib/utils/json-pointer`)

This is the first thing we should build. JSON Pointer (RFC 6901) is a simple string format for pointing to a specific value inside a JSON document. It's a standalone spec — useful on its own — and it's the foundation that JSON Patch depends on for every operation.

A pointer is a series of tokens separated by `/`. Each token is either an object key name or an array index. An empty string points to the root (the whole document).

#### How It Works — Examples

Given this document:

```json
{
  "foo": ["bar", "baz"],
  "": 0,
  "a/b": 1,
  "m~n": 8
}
```

| Pointer | Points To | Explanation |
| ------- | --------- | ----------- |
| `""` | `{ "foo": [...], ... }` | Empty string = the whole document |
| `"/foo"` | `["bar", "baz"]` | Key `"foo"` |
| `"/foo/0"` | `"bar"` | First element of the `foo` array |
| `"/"` | `0` | Key is an empty string `""` |
| `"/a~1b"` | `1` | Key `"a/b"` — `/` is escaped as `~1` |
| `"/m~0n"` | `8` | Key `"m~n"` — `~` is escaped as `~0` |

The escaping is important: since `/` is the separator, literal `/` in key names must be written as `~1`, and literal `~` as `~0`. When decoding, we must unescape `~1` → `/` first, then `~0` → `~` (the RFC is specific about this order — doing it backwards would turn `~01` into `/` instead of the correct `~1`).

For arrays, the token must be either a valid number (no leading zeros — so `/foo/01` is invalid) or the special `-` character which means "past the end" (used by JSON Patch for appending).

#### What We Need to Build

We need five functions:

**`parse( pointer )`** — takes a pointer string and returns an array of unescaped tokens.

```js
var tokens = parse( '/foo/0/bar' );
// returns [ 'foo', '0', 'bar' ]

var tokens = parse( '/a~1b/c~0d' );
// returns [ 'a/b', 'c~d' ]

var tokens = parse( '' );
// returns []
```

**`resolve( document, pointer )`** — walks the document following the pointer and returns the parent object, the final key, and the value at that location. This is the pattern from jiff — returning the parent + key means operations can just do `target[key] = value` or `delete target[key]` directly.

```js
var doc = { foo: [ 'bar', 'baz' ] };

var ref = resolve( doc, '/foo/0' );
// ref.target → doc.foo  (the parent array)
// ref.key    → 0        (the index)
// ref.value  → 'bar'    (the value at that spot)

var ref = resolve( doc, '' );
// ref.target → null     (root has no parent)
// ref.key    → null
// ref.value  → doc      (the whole document)
```

**`get( document, pointer )`** — convenience wrapper that just returns the value.

```js
var val = get( { foo: 'bar' }, '/foo' );
// returns 'bar'
```

**`has( document, pointer )`** — returns true/false without throwing on missing paths.

```js
var bool = has( { foo: 'bar' }, '/foo' );     // true
var bool = has( { foo: 'bar' }, '/missing' ); // false
```

**`format( tokens )`** — converts a token array back into a pointer string (inverse of parse).

```js
var str = format( [ 'a/b', 'c~d' ] );
// returns '/a~1b/c~0d'
```

#### Package Structure

Following stdlib conventions, the package would look like:

```
lib/node_modules/@stdlib/utils/json-pointer/
├── lib/
│   ├── index.js          # Main export: resolve, with parse/format/get/has attached
│   ├── main.js           # resolve( document, pointer )
│   ├── parse.js          # parse( pointer ) → token array
│   ├── format.js         # format( tokens ) → pointer string
│   ├── get.js            # get( document, pointer ) → value
│   ├── has.js            # has( document, pointer ) → boolean
│   └── validate.js       # Internal: security checks + index validation
├── test/
│   ├── test.js
│   ├── test.parse.js
│   ├── test.format.js
│   ├── test.get.js
│   ├── test.has.js
│   └── test.resolve.js
├── benchmark/
│   └── benchmark.js
├── docs/
│   ├── repl.txt
│   └── types/
│       ├── index.d.ts
│       └── test.ts
├── examples/
│   └── index.js
├── README.md
└── package.json
```

The main export would be `resolve`, with the other functions attached as properties:

```js
var pointer = require( '@stdlib/utils/json-pointer' );

// Main usage:
var ref = pointer( doc, '/foo/0' );

// Sub-functions:
var val    = pointer.get( doc, '/foo/0' );
var exists = pointer.has( doc, '/foo/0' );
var tokens = pointer.parse( '/a~1b' );
var str    = pointer.format( [ 'a/b' ] );
```

#### Security

The security checks should live here at the pointer level — we reject `__proto__`, `constructor`, and `prototype` as path segments during traversal. That way anything built on top (JSON Patch, or any future consumer) gets protection automatically without needing its own checks.

### Step 2: JSON Patch (`@stdlib/utils/json-patch`) — Initial Thinking

This part still needs more research and design work, but here's my initial thinking based on what I've seen across the libraries.

Once the pointer package is solid, we'd build the patch package on top of it. The rough idea is a function like `applyPatch( document, operations, options )` that takes a JSON document and an array of RFC 6902 operations and returns the patched result.

From what I've seen so far, there are a few directions we should probably go:

- **Operation dispatch** — most of the better libraries (jiff, Fast-JSON-Patch) use an object lookup to pick the right handler function instead of big if/else chains. That seems like the right approach for stdlib too, but we need to figure out the exact function signatures and how options flow through.

- **Immutability** — the structural sharing approach from immutable-json-patch (only copy objects along the modified path) looks really promising for our Vega use case, but I need to dig deeper into how their `setIn`/`deleteIn` helpers actually work and how we'd adapt that to stdlib's style. There might also be edge cases around arrays and the `-` append token that need more thought.

- **Atomicity** — JSON8 Patch's rollback pattern is interesting, but I haven't fully traced through how it handles complex cases like `move` operations that fail halfway. This needs more study.

- **Validation** — we should probably validate by default (most libraries don't, which feels unsafe), but the exact validation rules and error messages need to be designed.

- **Existing stdlib utilities** — for things like deep equality (`test` operation) and deep cloning (`copy` operation), we should be able to lean on `@stdlib/assert/deep-equal` and `@stdlib/utils/copy` rather than writing our own, but I haven't verified that they cover all the edge cases RFC 6902 requires.

Basically, the research so far has given me a good sense of the landscape and the patterns that work, but the actual package design — API shape, file structure, how structural sharing integrates with the operation handlers, how we handle errors — still needs to be worked out once the pointer package is done and we have a clearer picture.

## What We Should Take From Where

| What | From Where | Why |
| ---- | ---------- | --- |
| Code style and module layout | jiff | Plain JS, CommonJS, functional, closest to stdlib |
| Prototype pollution blocking | Fast-JSON-Patch | Only library that does it — security is non-negotiable |
| `{ target, key }` pointer resolution | jiff | Clean separation of traversal from operation logic |
| Structural sharing for immutability | immutable-json-patch | O(d) instead of O(n) — perfect for large Vega specs |
| Atomic rollback on failure | JSON8 Patch | True RFC compliance for all-or-nothing semantics |
| Operation dispatch | Fast-JSON-Patch, jiff | Object lookup instead of if/else chains |
| LCS array diffing (future) | jiff | For a future `diff()` function |

## Order of Work

1. Build the JSON Pointer package first — it stands alone and everything else depends on it
2. Test against the community RFC test suite that other libraries use
3. Build the JSON Patch package — `add`, `remove`, `replace` first, then `move`/`copy`/`test`
4. Test with the full JSON Patch test suite
5. Wire it into the Vega SSE workflow
