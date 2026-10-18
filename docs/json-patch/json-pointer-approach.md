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

# JSON Pointer Implementation Approach

> Detailed plan for implementing an RFC 6901-compliant JSON Pointer package in stdlib.

---

## 1. What is JSON Pointer (RFC 6901)

JSON Pointer is a simple string format for pointing to a specific value inside a JSON document. A pointer is a series of tokens separated by `/`, where each token is a key name (for objects) or an index (for arrays).

### Examples

Given this document:

```json
{
  "foo": ["bar", "baz"],
  "": 0,
  "a/b": 1,
  "m~n": 8
}
```

| Pointer | Points to |
| ------- | --------- |
| `""` (empty string) | The whole document |
| `"/foo"` | `["bar", "baz"]` |
| `"/foo/0"` | `"bar"` |
| `"/"` | `0` (the key is an empty string `""`) |
| `"/a~1b"` | `1` (the key is `"a/b"` — `/` is escaped as `~1`) |
| `"/m~0n"` | `8` (the key is `"m~n"` — `~` is escaped as `~0`) |

### Rules from the RFC

1. **Empty string** = points to the root (the entire document)
2. **Must start with `/`** if non-empty
3. **Escaping**: `~` → `~0`, `/` → `~1`. When decoding, **`~1` must be unescaped before `~0`** (otherwise `~01` would wrongly become `/` instead of `~1`)
4. **Arrays**: Token must be either a number (no leading zeros, except `"0"` itself) or `"-"` (means "past the last element" — used by JSON Patch for appending)
5. **Errors**: If a pointer refers to something that doesn't exist, that's an error. The RFC doesn't say *how* to handle it — that's up to the application

---

## 2. How Other Libraries Handle It

I reviewed the pointer implementation in every library. Here's what each one does differently and what we can learn from it.

### jiff — Single-pass parsing with `{ target, key }` return

jiff's `jsonPointerParse.js` walks the pointer string character by character and calls back with each unescaped token. It doesn't create an intermediate array of tokens — it processes them as it finds them.

More importantly, jiff's `jsonPointer.find()` resolves a pointer against a document and returns `{ target, key }` — the **parent object** and the **final key**. This is the cleanest pattern I found because:

- The caller can do `target[key] = value` for add/replace
- The caller can do `delete target[key]` for remove
- The caller can read `target[key]` for get/test
- One traversal function serves all operations

**What to take**: The `{ target, key }` return pattern. This is what our `resolve` function should return.

### Fast-JSON-Patch — Prototype pollution protection

Fast-JSON-Patch is the only library that blocks dangerous path segments like `__proto__`, `constructor`, and `prototype`. During pointer traversal, it checks each token against a blocklist. If a token is `__proto__` (or similar), it throws an error instead of traversing into it.

Without this check, an attacker could craft a pointer like `/__proto__/isAdmin` and set `isAdmin = true` on every single object in the JavaScript runtime. This is called prototype pollution, and it's a real vulnerability that has caused CVEs in popular libraries.

**What to take**: The security check. We should reject `__proto__` and `constructor` as tokens. This is non-negotiable.

### Fast-JSON-Patch — Root document special-casing

When the pointer is an empty string (pointing to root), Fast-JSON-Patch returns immediately without any traversal. This avoids unnecessary work and handles the root case cleanly since there's no parent object to return.

**What to take**: Handle `""` as a special case at the top of the function. Return something like `{ target: null, key: null, value: document }` so the caller knows they're dealing with the root.

### json-joy — Inline `indexOf` traversal (no array allocation)

json-joy's v3 implementation doesn't split the pointer into an array. Instead, it uses a `while` loop with `indexOf('/')` to walk through the string, extracting one token at a time. This means zero array allocation — the fastest possible approach.

**What to take**: The general idea. For stdlib, the simpler `split('/')` approach is fine since it's clearer and the performance difference is negligible for typical Vega specs. But if we ever need to optimize, this is the path.

### jsonpatch.js (dharmafly) — Pre-parsing at construction time

jsonpatch.js parses the pointer string into an array of tokens once (at construction), and then reuses those tokens every time the pointer is applied. This amortizes the parsing cost when the same pointer is used multiple times.

**What to take**: Consider offering a `parse` function that returns a token array, separate from `resolve`. This way callers who apply the same pointer repeatedly can parse once and resolve many times.

### immutable-json-patch — `resolvePathIndex` for `-`

immutable-json-patch resolves the special `-` token (meaning "append to array") into the actual array length before any operation runs. So `/foo/-` on an array of length 3 becomes `/foo/3`. This simplifies operation handlers — they never have to deal with `-` themselves.

**What to take**: Resolve `-` to the array's `.length` during traversal when the context requires it (add operations). For get/test/remove, `-` should be an error since it refers to a nonexistent element.

### json-joy — Separate `find.ts` with typed references

json-joy's `find.ts` returns a typed reference: `{ val, obj, key }` where `val` is the resolved value, `obj` is the parent, and `key` is the final key. It also distinguishes array references from object references with helper functions (`isArrayReference`, `isObjectReference`).

**What to take**: Including the resolved `value` in the return object is useful. Our return type should be `{ target, key, value }` — the parent, the final key, and the value at that location (or `undefined` if it doesn't exist).

---

## 3. What We Need to Build

Based on the RFC and library analysis, here are the functions we need:

### 3.1 `parse( pointer )` — Parse a pointer string into tokens

**Input**: A JSON Pointer string (e.g., `"/foo/0/bar"`)
**Output**: An array of unescaped tokens (e.g., `["foo", "0", "bar"]`)
**Errors**: Throws if the pointer doesn't start with `/` (and isn't empty)

This function handles:
- Splitting on `/`
- Unescaping `~1` → `/` then `~0` → `~` (in that order!)
- Validating syntax

```
parse( '/foo/bar' )     → [ 'foo', 'bar' ]
parse( '/a~1b/c~0d' )  → [ 'a/b', 'c~d' ]
parse( '' )             → []
parse( '/foo/-' )       → [ 'foo', '-' ]
parse( 'foo' )          → THROWS (no leading slash)
```

### 3.2 `resolve( document, pointer )` — Resolve a pointer against a document

**Input**: A JSON document and a pointer (string or pre-parsed token array)
**Output**: `{ target, key, value }` — the parent object, final key, and the value at that location
**Errors**: Throws if the path doesn't exist in the document

This function handles:
- Walking the document following each token
- Array index coercion (string token → integer)
- Array index validation (no leading zeros, must be non-negative, within bounds)
- `-` token handling (returns array length as key)
- `__proto__` / `constructor` blocking
- Root pointer (`""`) special case

```
var doc = { foo: [ 'bar', 'baz' ] };

resolve( doc, '/foo/0' )
// → { target: doc.foo, key: 0, value: 'bar' }

resolve( doc, '/foo' )
// → { target: doc, key: 'foo', value: ['bar', 'baz'] }

resolve( doc, '' )
// → { target: null, key: null, value: doc }  (root)

resolve( doc, '/missing' )
// → THROWS
```

### 3.3 `get( document, pointer )` — Get the value at a pointer

**Input**: A JSON document and a pointer
**Output**: The value at that location
**Errors**: Throws if the path doesn't exist

A convenience wrapper around `resolve` that just returns `value`.

```
get( { foo: 'bar' }, '/foo' )  →  'bar'
get( { foo: 'bar' }, '' )      →  { foo: 'bar' }
```

### 3.4 `has( document, pointer )` — Check if a pointer resolves

**Input**: A JSON document and a pointer
**Output**: `true` if the path exists, `false` if not

Doesn't throw on missing paths — that's the whole point.

### 3.5 `format( tokens )` — Convert token array back to pointer string

**Input**: An array of tokens (e.g., `["a/b", "c~d"]`)
**Output**: An escaped pointer string (e.g., `"/a~1b/c~0d"`)

Inverse of `parse`. Escapes `~` → `~0` first, then `/` → `~1`.

---

## 4. Stdlib Package Structure

Following stdlib conventions (tabs, `'use strict'`, SPDX headers, `lib/main.js` + `lib/index.js` pattern):

```
lib/node_modules/@stdlib/json/pointer/
├── lib/
│   ├── index.js          # Re-exports main + attaches sub-functions
│   ├── main.js           # Main export: resolve( document, pointer )
│   ├── parse.js          # parse( pointer ) → token array
│   ├── format.js         # format( tokens ) → pointer string
│   ├── get.js            # get( document, pointer ) → value
│   ├── has.js            # has( document, pointer ) → boolean
│   └── validate.js       # Internal: pointer syntax + security checks
├── test/
│   ├── test.js           # Main tests
│   ├── test.parse.js     # parse() tests
│   ├── test.format.js    # format() tests
│   ├── test.get.js       # get() tests
│   ├── test.has.js       # has() tests
│   └── test.resolve.js   # resolve() tests
├── benchmark/
│   └── benchmark.js      # Performance benchmarks
├── docs/
│   ├── repl.txt          # REPL help text
│   └── types/
│       ├── index.d.ts    # TypeScript declarations
│       └── test.ts       # Type tests
├── examples/
│   └── index.js          # Usage examples
├── README.md
└── package.json
```

### What `lib/index.js` exports

```js
var resolve = require( './main.js' );
var parse = require( './parse.js' );
var format = require( './format.js' );
var get = require( './get.js' );
var has = require( './has.js' );

// Main export is `resolve`, with other functions as properties:
resolve.parse = parse;
resolve.format = format;
resolve.get = get;
resolve.has = has;

module.exports = resolve;
```

So you can use it like:

```js
var pointer = require( '@stdlib/json/pointer' );

// Resolve (main export):
var ref = pointer( doc, '/foo/0' );
// ref.target, ref.key, ref.value

// Convenience:
var val = pointer.get( doc, '/foo/0' );
var exists = pointer.has( doc, '/foo/0' );

// Parse/format:
var tokens = pointer.parse( '/a~1b' );  // [ 'a/b' ]
var str = pointer.format( [ 'a/b' ] );  // '/a~1b'
```

---

## 5. Implementation Details

### 5.1 `parse` — Step by Step

```
function parse( pointer ):
    1. If pointer is empty string → return []
    2. If pointer[0] !== '/' → throw TypeError
    3. Split pointer on '/' → tokens (skip first empty element)
    4. For each token:
       a. Replace '~1' with '/' (MUST be first)
       b. Replace '~0' with '~' (MUST be second)
    5. Return token array
```

Why step 4a before 4b: The RFC explicitly says this. If you did it the other way, the string `~01` would first become `~1` (wrong — you turned the escaped tilde into nothing) and then become `/` (double-wrong). The correct result is `~1` (the literal string "tilde-one").

### 5.2 `resolve` — Step by Step

```
function resolve( document, pointer ):
    1. Parse pointer → tokens (via parse())
    2. If tokens is empty → return { target: null, key: null, value: document }
    3. Set current = document
    4. For each token EXCEPT the last:
       a. SECURITY: if token is '__proto__' or 'constructor' → throw Error
       b. If current is an array:
          - Validate token is a valid array index (digits, no leading zeros)
          - Coerce token to integer
          - If index >= array.length → throw RangeError
          - current = current[ index ]
       c. Else if current is an object:
          - If !hasOwnProperty( current, token ) → throw Error
          - current = current[ token ]
       d. Else → throw TypeError (can't traverse into a primitive)
    5. Handle the LAST token:
       a. Same security check
       b. If current is an array and token is '-' → key = current.length
       c. Else if current is an array → validate + coerce to integer, key = integer
       d. Else → key = token (string)
    6. Return { target: current, key: key, value: current[key] }
```

Why we handle the last token separately: Because the last token's parent (`target`) is what operations need. The traversal stops one level before the end, so we have both the parent and the key.

### 5.3 Security checks — What to block

Based on Fast-JSON-Patch's approach:

```js
function isSafe( token ) {
    if ( token === '__proto__' ) return false;
    if ( token === 'constructor' ) return false;
    if ( token === 'prototype' ) return false;
    return true;
}
```

This runs on every token during traversal. It's cheap (three string comparisons) and prevents prototype pollution completely.

### 5.4 Array index validation

The RFC says array indices must be:
- The character `"0"` alone, OR
- A sequence of digits that doesn't start with `"0"` (i.e., no leading zeros like `"01"`)
- Non-negative

```js
function isValidArrayIndex( token ) {
    if ( token === '0' ) return true;
    if ( token === '-' ) return true;  // special append token
    return /^[1-9][0-9]*$/.test( token );
}
```

### 5.5 stdlib dependencies

| Need | stdlib package | Where used |
| ---- | -------------- | ---------- |
| `hasOwnProperty` check | `@stdlib/assert/has-own-property` | Traversal — checking if object has the key |
| Type checking (is object) | `@stdlib/assert/is-plain-object` | Traversal — knowing whether to use string key or integer index |
| Type checking (is array) | `@stdlib/assert/is-array` | Traversal — array index handling |
| Type checking (is string) | `@stdlib/assert/is-string` | Input validation |

---

## 6. Edge Cases to Handle

These come from the RFC spec and from the community test suite:

| Case | Pointer | Expected Behavior |
| ---- | ------- | ----------------- |
| Root | `""` | Return the whole document |
| Empty key | `"/"` | Object member with key `""` |
| Nested empty keys | `"//"` | Object member `""` inside object member `""` |
| Escaped tilde | `"/m~0n"` | Key `"m~n"` |
| Escaped slash | `"/a~1b"` | Key `"a/b"` |
| Double escape `~01` | `"/~01"` | Key `"~1"` (NOT `"/"`) |
| Array index | `"/foo/0"` | First element of array `foo` |
| Array append `-` | `"/foo/-"` | Past-the-end (for add operations) |
| Leading zeros | `"/foo/01"` | **Error** — not a valid index |
| Negative index | `"/foo/-1"` | **Error** — not valid per RFC |
| Non-numeric on array | `"/foo/bar"` | **Error** — array needs numeric index |
| Missing key | `"/nonexistent"` | **Error** — key doesn't exist |
| Traverse into primitive | `"/foo/0/bar"` where `foo[0]` is a string | **Error** — can't traverse into primitive |
| `__proto__` | `"/__proto__"` | **Error** — blocked for security |
| Unicode keys | `"/café"` | Should work — RFC says pointers are Unicode strings |
| NUL character | Key containing `\u0000` | Should work — RFC specifically mentions this |

---

## 7. Testing Strategy

### 7.1 Community test suite

The [json-patch-test-suite](https://github.com/json-patch/json-patch-tests) includes pointer evaluation tests. Both Fast-JSON-Patch and jiff run against this. We should too.

### 7.2 Our own tests

Organized by function, covering:
- **Happy path**: Standard RFC examples from Section 5
- **Escaping**: `~0`, `~1`, combined sequences, double-escape `~01`
- **Arrays**: Valid indices, `-` token, out-of-bounds, leading zeros, non-numeric
- **Security**: `__proto__`, `constructor`, `prototype` tokens
- **Edge cases**: Root pointer, empty key, nested empty keys, Unicode, NUL
- **Input validation**: Non-string pointer, null document, missing arguments
- **Error messages**: Verify error messages are descriptive and include the pointer + token that caused the error

### 7.3 Benchmarks

Benchmark `resolve()` against:
- Shallow pointers (`/foo`)
- Deep pointers (`/a/b/c/d/e/f`)
- Array-heavy pointers (`/0/1/2/3`)
- Long key names
- Pointers with escaping

---

## 8. Order of Implementation

1. **`parse.js`** — Straightforward string splitting + unescaping. No document needed. Easy to test in isolation.
2. **`format.js`** — Inverse of parse. Test round-trip: `format(parse(ptr)) === ptr`.
3. **`validate.js`** — Internal security checks + index validation helpers.
4. **`main.js` (resolve)** — The core function. Depends on parse + validate.
5. **`get.js`** — Thin wrapper over resolve.
6. **`has.js`** — Wrapper that catches resolve errors and returns boolean.
7. **`index.js`** — Wire everything together.
8. **Tests** — Write alongside each file.
9. **README.md, docs, examples** — After the implementation is solid.

---

## 9. Open Questions

These are things to discuss/decide before or during implementation:

1. **Package path**: Is `@stdlib/json/pointer` the right location? Or should it be `@stdlib/utils/json-pointer` to match stdlib's utility conventions?

2. **Return shape for root**: When the pointer is `""`, there's no parent and no key. Should `resolve` return `{ target: null, key: null, value: doc }` or should it be a different signal?

3. **`-` token behavior**: Should `resolve` accept `-` and return `key = array.length`? Or should `-` always be an error in `resolve` (and only handled specially by JSON Patch's `add` operation)?

4. **Strict vs lenient mode**: Should we offer an option to skip the security checks (for trusted internal use)? Or always enforce them?

5. **Pre-parsed token input**: Should `resolve` accept both strings and pre-parsed token arrays? (jiff does this — it checks `typeof pointer` and skips parsing if it's already an array.)
