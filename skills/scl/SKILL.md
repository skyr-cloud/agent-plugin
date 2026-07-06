---
name: scl
description: >-
    Write SCL (Skyr Configuration Language) and SCLE. Use when creating or
    editing .scl or .scle files, authoring infrastructure-as-code targeting
    Skyr, editing Main.scl or Package.scle, or looking up SCL syntax, types,
    or standard-library documentation.
---

# Writing SCL

SCL is a statically-typed, purely functional language for describing
infrastructure. You declare *what* resources should exist; Skyr computes the
dependency graph and handles ordering, creation, updates, and destruction.
Configuration deploys by pushing the repository to its `skyr` git remote —
the entrypoint is always `Main.scl` at the repo root.

Two file kinds share the language:

- `.scl` — a module: imports, `let` bindings, type declarations, exports, and
  top-level resource expressions.
- `.scle` — an SCL Expression: a single typed value (imports, then one type
  expression, then one body expression). Used for manifests like
  `Package.scle`.

## Mental model

- **No mutation, no loops, no side-effecting statements.** Everything is an
  expression; iteration is `List.map`/comprehensions; "variables" are
  immutable `let` bindings.
- **Resource calls look like function calls** (`Artifact.File({...})`) and
  return records of *outputs*. Referencing an output of resource A in the
  inputs of resource B is what creates the dependency edge A → B. There is no
  explicit `depends_on`.
- **Types are structural.** Records match by shape, not name. Inference is
  strong; annotate only when the compiler asks or for documentation. It never
  guesses: an `if`/`try`/list/dict whose parts share no common type, or a
  generic type parameter used two incompatible ways, is a direct error at the
  construct or call site — not a silent widening to `Any`.
- **Two stdlib namespaces**: `Std/*` is the pure standard library built into
  the compiler (strings, lists, dicts, time, hashing, encoding, …). `Skyr/*`
  modules are served by resource plugins installed on the Skyr instance you
  deploy to — their exact signatures are instance-defined, so look them up
  (see "Looking up documentation" below) rather than guessing.

## Values and types

```scl
let count = 42                             // Int (64-bit signed)
let ratio = 3.14                           // Float (digits required both sides of .)
let name = "my-app"                        // Str
let enabled = true                         // Bool
let nothing = nil                          // Never? — only assignable to optionals
let conf = ./config.json                   // Path (repo-relative file reference)
let items = [1, 2, 3]                      // [Int]
let server = { port: 8080, debug: false }  // { port: Int, debug: Bool }
let lookup = #{ "key": "value" }           // #{ Str: Str } — dict, computed keys
```

- **Records** `{ field: value }` have fixed, identifier-named fields. Field
  shorthand works: `{ name, port }` ≡ `{ name: name, port: port }`.
- **Dicts** `#{ key: value }` have computed keys of one type: in
  `#{ key: 42 }` the key is the *value of the variable* `key`, not `"key"`.
- **Paths** are literals: `./src`, `../shared`, `/abs/from/repo/root`.
  Relative paths resolve against the current module's directory. Quote odd
  segments: `./dir/"file with spaces.txt"`.
- Type syntax: `Int`, `Float`, `Str`, `Bool`, `Path`, `Any`, `T?`, `[T]`,
  `#{ K: V }`, `{ f: T }`, `fn(A, B) R`, `fn<T>(T) T`,
  `fn<T <: { name: Str }>(T) Str`.
- **Collection element types** are inferred across *all* entries,
  order-independently: `[1, nil]` is `[Int?]`, `#{1: "x", 2: nil}` is
  `#{Int: Str?}`. Elements with no common type are an error — ascribe
  `as [Any]` (or `as #{ Any: Any }`) to force a heterogeneous literal.

Strings interpolate expressions with `{...}` — no `$`:

```scl
let greeting = "Hello, {name}!"                 // any expression inside {}
let info = "port: {server.port}, next: {count + 1}"
let literal = "escaped \{not interpolated}"     // \n \r \t \\ \{ are the escapes
```

## Optionals

`T?` holds a value or `nil`. `T` auto-widens to `T?`, never the reverse.
Indexing lists/dicts always yields an optional (`items[99]` may be `nil`).

```scl
let port: Int? = nil
port ?? 3000                    // nil-coalescing default → 3000

let user: { name: Str }? = nil
user?.name ?? "anonymous"       // optional chaining; result is Str
```

There is no null-pointer error path: the type system forces you through
`?.`, `??`, or `Std/Option` (`Option.unwrap` raises `Option.UnexpectedNil`).

## Expressions

Everything is an expression; there are no statements inside function bodies.

```scl
let status = if (enabled) "on" else "off"   // if-expression; parens required
let maybe = if (count > 0) count            // no else → type is Int?
let two = let x = 1; x + 1                  // inline let: binding; body

// Anonymous functions (closures). Param types inferred when context knows them.
let double = fn(x: Int) x * 2
let doubled = List.map([1, 2, 3], fn(x) x * 2)

// Generics, with optional subtype bounds
let identity = fn<T>(x: T) x
let getName = fn<T <: { name: Str }>(item: T) item.name

// List comprehensions: for iterates, if filters; clauses stack
let evens = [for (x in items) if (x / 2 * 2 == x) x]
let pairs = [for (x in xs) for (y in ys) x + y]

// Exceptions
let ParseError = exception(Str)             // payload optional: `exception` alone
// raise may sit in either branch — an if types as what covers both branches,
// and raise produces no value, so guard clauses work in either position
let risky = fn(s: Str) if (s == "") raise ParseError("empty") else s
let safe = try risky(input)
    catch ParseError(msg): "fallback: {msg}"
// catch targets may be dotted paths to module-owned exceptions:
//   try Option.unwrap(x) catch Option.UnexpectedNil: fallback
```

Operator notes: `+` concatenates strings; `Int`+`Float` arithmetic yields
`Float`; integer division truncates (`10 / 3` → `3`); `as` (type cast) binds
tighter than every binary operator, so `(1 + x) as Int` needs the parens.

## Modules and imports

```scl
import Std/List                       // stdlib
import Skyr/Container                 // instance plugin module
import Self/Config                    // ./Config.scl (or .scle) in this repo
import Self/Utils/Network             // ./Utils/Network.scl
import Std/Time as T                  // alias — binds T, not Time
import acme/platform/Database         // cross-repo (needs Package.scle, below)
```

- The **last path segment** becomes the in-scope binding unless aliased with
  `as`. Hyphenated final segments must be aliased
  (`import acme/repo/my-file as File`).
- `Self` is the current repo's package (`org/repo` when deployed; inferred
  from the git remote by local tooling).
- `export let` / `export type` make bindings importable. Type and value
  namespaces are separate — a module can export both `type Config` and
  `let Config`.

```scl
type Port Int
export type Config { host: Str, port: Port }
export let defaults: Config = { host: "localhost", port: 8080 }
```

### SCLE modules

A `.scle` file is one self-contained typed expression: imports, then a type
expression, then the body. When imported, the module *is* its body value.

```scl
// Limits.scle
{ maxPods: Int, burst: Int }

{
    maxPods: 20,
    burst: 5,
}
```

```scl
// Main.scl
import Self/Limits

let cap = Limits.maxPods        // plain property access on the module's value
```

A given module path may exist as `.scl` *or* `.scle`, never both — defining
both is an ambiguous-module error.

## Package.scle and cross-repo imports

To import modules from another repository in the same organisation, declare
the dependency in `Package.scle` at the repo root — itself an SCLE file whose
value is a `Std/Package.Manifest`:

```scl
import Std/Package

Package.Manifest

{
    dependencies: #{
        "acme/platform":    "main",
        "acme/shared-libs": "tag:v1.2.0",
        "acme/pinned":      "b50d18287a6a3b86c3f45e3a973a389784d353dd",
    },
}
```

Specifiers pin a git ref: a bare name is a **branch** (follows that branch's
active deployment), `tag:<name>` is a tag, and a 40-hex string is a **commit
hash** (fully deterministic). Branch/tag pins are *volatile*: the deployment
keeps reconciling foreign changes and stays in the `Desired` state forever,
never settling into `Up`. Pin hashes to settle. Edit the manifest directly or
via the CLI: `skyr deps list` / `skyr deps add acme/platform main` /
`skyr deps rm acme/platform`.

Once declared, import by qualified path:

```scl
import acme/platform/Database

let url = Database.primaryUrl               // remote state: read their outputs
let bucket = Database.makeBucket({ ... })   // remote module: resource is YOURS
```

Ownership rule: a resource belongs to the deployment whose own code path
reaches the resource call. Reading a foreign repo's top-level resource output
is remote state; calling a function it exports creates a resource owned by
*your* deployment.

## Resources

A resource call takes a record of inputs and returns a record of outputs.
Identity is structural — derived from the resource type, its `name`, its
region, and (for some types) other inputs — so renaming or moving a resource
means destroy + create, not update.

```scl
import Skyr/Artifact

let readme = Artifact.File({
    name: "readme.txt",
    contents: "Hello, world!",
    mediaType: "text/plain",
})

// Using an output creates the dependency: this file is only written after
// `readme` exists and its time-limited download URL is known.
Artifact.File({
    name: "index.txt",
    contents: "readme lives at {readme.url}",
})
```

Rules that matter in practice:

- **Dependencies are output references.** If B's inputs mention `a.someOutput`,
  B waits for A. No reference, no ordering.
- **Idempotent repeats**: declaring the same resource twice with identical
  inputs is fine (both resolve to one resource); twice with different inputs
  is an eval-time error.
- **Regions**: resources that are region-placeable take an optional
  `region: Str?` input (a label like `"stockholm"`), defaulting to the
  repository's region. Region is part of identity — changing it is
  destroy + create.
- **Lifecycle across deploys**: new declarations are created, changed ones
  updated (or replaced when identity changed), removed ones destroyed,
  untouched ones preserved.
- Deployment identity (org / repo / environment names) is available at eval
  time via `Std/Env`: `Env.environment.name`, `Env.repository.name`, etc.

**Do not guess `Skyr/*` signatures.** Plugin modules (`Skyr/Container`,
`Skyr/DNS`, `Skyr/IAM`, `Skyr/PKI`, `Skyr/HTTP`, `Skyr/Rollout`,
`Skyr/Random`, `Skyr/Artifact`, …) are served by the instance you deploy to,
and their inputs/outputs are precise. Look them up before use, and let
`skyr check` confirm.

## Looking up documentation

The Skyr docs are served as raw markdown, ideal for fetching and grepping.
Current canonical location (if the repo deploys to a different Skyr instance,
use that instance's host instead):

```sh
curl -s https://skyr.foo/llms.txt                        # index of all doc pages
curl -s https://skyr.foo/~docs/scl/stdlib.md             # full stdlib + plugin resource reference
curl -s https://skyr.foo/~docs/scl/syntax.md             # complete syntax reference
curl -s https://skyr.foo/~docs/scl/types.md              # type system in depth
curl -s https://skyr.foo/~docs/cross-repo-imports.md     # Package.scle details
```

Typical lookup — find a module or resource's exact fields:

```sh
curl -s https://skyr.foo/~docs/scl/stdlib.md | grep -n -A 40 '^### Container.Pod'
curl -s https://skyr.foo/~docs/scl/stdlib.md | grep -n '^## \|^### '   # table of contents
```

`stdlib.md` documents every `Std/*` module (`Str`, `List`, `Dict`, `Option`,
`Num`, `Float`, `Time`, `Path`, `Encoding`, `Crypto`, `Dyn`, `Env`,
`Package`) and every `Skyr/*` resource with input/output tables. For exact
language semantics beyond the docs (typing rules, evaluation order), the
formal SCL specification PDF ships alongside Skyr releases on dl.skyr.cloud.

## Verifying your work

Always finish by formatting and type-checking; never hand back unchecked SCL.

```sh
skyr fmt --write Main.scl   # canonical formatting, per file (omit --write to preview)
skyr check                  # parse + resolve + type-check the whole package (no deploy)
skyr repl                   # interactive: evaluate expressions, inspect types
```

- `skyr check` checks the package rooted at `Main.scl` in the current
  directory (override with `--root`). It stops before evaluation, so it
  catches syntax, resolution, and type errors — not eval-time errors like
  conflicting duplicate resource declarations.
- The package name is inferred from the `skyr`/`origin` git remote; override
  with `skyr check --package org/repo` when the inference fails (you'll see
  it default to `Local`).
- `Skyr/*` imports are type-checked against the frontends served by the
  instance's API: the first such check fetches them over the network (then
  disk-caches). Pure-`Std/` packages check fully offline. If a `Skyr/*`
  import won't resolve and the machine is offline with a cold cache, that's
  why — it is not necessarily a typo.
- Cross-repo imports resolve through the instance too, and need the
  dependency repos to exist there.

Fix diagnostics in source order — SCL errors carry causal chains for nested
type mismatches, and an early resolution failure often cascades.
