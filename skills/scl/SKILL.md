---
name: scl
description: >-
    Write SCL (Skyr Configuration Language) and SCLE. Use when creating or
    editing .scl or .scle files, authoring infrastructure-as-code targeting
    Skyr, editing Main.scl or Package.scle, writing or running a module's own
    tests, or looking up SCL syntax, types, or standard-library documentation.
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
  expression; iteration is `List.map`/comprehensions, and collapsing a list to
  one value is `List.fold`/`List.reduce`; "variables" are immutable `let`
  bindings.
- **Resource calls look like function calls** (`Artifact.File({...})`) and
  return records of *outputs*. Referencing an output of resource A is what
  creates the dependency edge A → B — whether B's inputs are built from it,
  or it only decided that B is declared at all (`if (a.ready) B({...})`).
  There is no explicit `depends_on`: data and control flow are both tracked
  for you.
- **Types are structural.** Records match by shape, not name. Inference is
  strong; annotate only when the compiler asks or for documentation. It never
  guesses: an `if`/`try`/list/dict whose parts share no common type, or a
  generic type parameter used two incompatible ways, is a direct error at the
  construct or call site — not a silent widening to `Any`.
- **Two kinds of modules**: `Std/*` is the pure standard library built into
  the compiler (strings, lists, dicts, time, hashing, encoding, …). Platform
  modules — everything else — are served by resource plugins installed on the
  Skyr instance you deploy to; first-party modules live under `Skyr/*`, and
  plugins may serve other namespaces too (e.g. `HashiCorp/Random`). Their
  exact signatures are instance-defined, so look them up (see "Looking up
  documentation" below) rather than guessing.

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
  A dict holds each key once — writing a key the literal already holds
  replaces its value **in the first write's position**, so `#{"a": 1, "b": 2,
  "a": 3}` is `#{"a": 3, "b": 2}`, size 2. Two plain entries with the same
  constant key are a warning: the first entry's *value* can never be observed,
  only the position it fixes for the key. The warning points at the later
  entry, so collapse the pair by moving its value into the **first** entry —
  deleting the first instead reorders the keys, and dict equality is
  order-sensitive. A collision between *generated* or *spliced* entries is not
  a warning (see "Dict entries" below).
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

**`T?` is an optional type, never an omittable argument.** Call arity is exact:
every parameter takes an argument, and an optional one takes `nil` to mean
"nothing" (`Option.map(nil, f)`). The case worth remembering is a constructor
whose only parameter is a record with no required fields — the record is still
written, as `{}`: `Rollout.Rollback({})` compiles and `Rollout.Rollback()` does
not. Record *fields* are the omittable thing; arguments are not.

## Atoms and enums

An **atom** `.name` is a first-class symbolic value (bare-identifier label,
camelCase by convention). An **enum type** `enum { .a, .b }` is a fixed set of
them — the way to type "one of N choices" so a wrong option is a compile error,
not a runtime failure. `enum` is a keyword; the variants are a set (order-free,
no duplicates, no empty `enum {}`).

```scl
type Mode enum { .dev, .staging, .prod }
let mode: Mode = .prod              // .prod alone has type enum { .prod }
```

- **Structural**, like every SCL type: an enum *is* its label set; a `type`
  alias adds no nominal identity.
- **Subtyping is subset inclusion** — the dual of record width: `enum { .a }`
  is assignable to `enum { .a, .b }` (fewer → more). A lone atom therefore
  fits anywhere its variant is listed.
- **Join is union**: `if (c) .a else .b` is `enum { .a, .b }`; same across list
  elements, dict values, record fields. Joining an enum with a non-enum is an
  error (no widening), like `Int` vs `Str`.
- **Equality** is `==`/`!=` by name — and, for tagged atoms, by payload too
  (ordering ops are numeric-only). Assigning a non-member atom to an enum-typed
  slot is a hard error; a comparison that can never hold — a disjoint variant,
  or an atom vs a string (`.prod` is not `"prod"`) — is flagged at compile time
  as always-`false`.
- **Narrowing** fires against a *literal* atom: `if (m == .prod)` refines `m`
  to `enum { .prod }` in the `then`, and *subtracts* the variant in the `else`,
  so an `if`/`else if` chain gets progressively tighter (exhausting the set
  leaves `Never`). For `m: Mode?`, a positive match also drops the `nil`.
  Comparing two enum *variables* checks but narrows nothing. The same
  narrowing (nil checks included) fires in a collection element's `if` guard
  (below).
- **String boundary**: a *bare* atom's interpolation and `Encoding.toJson` drop
  the dot (`.prod` → `"prod"`); `fromJson` never yields an atom. Like `Path`,
  atom-ness does not survive leaving the system. A *tagged* atom has no plain
  form — interpolation renders it whole (`.ok(1)`); JSON/YAML/TOML encode it,
  lossily, as a one-key object carrying the payload (`.literal("x")` →
  `{"literal":"x"}`, `.pair(1, 2)` → `{"pair":[1,2]}`).

```scl
// Optional enum field with a default; ?? joins the default's singleton back in.
type Curve enum { .p256, .p384, .p521 }
let keyCurve = fn(c: Curve?) c ?? .p256                    // Curve
let planeLabel = fn(m: Mode) if (m == .prod) "live" else "preview"
```

### Tagged values and `switch`

An atom may carry a positional **payload**, so enums are full tagged unions
(sum types), including recursive ones. Write the payload types in the variant
(`.name(T, …)`) and give the atom arguments to construct one:

```scl
type Status enum { .active(Int), .idle, .error(Str) }
let running: Status = .active(8080)          // .active(1) synthesizes enum { .active(Int) }
type List enum { .cons(Int, List), .empty }  // recursive; terminator is .empty (.nil is reserved)
let nums: List = .cons(1, .cons(2, .empty))
```

- Empty parens `.name()` are rejected — a nullary variant is bare `.name`. A
  nullary `.a` and a tagged `.a(Int)` are different-arity, incompatible variants.
- **Payload subtyping** is width (on labels) plus slotwise **covariance**: a
  shared variant keeps its arity and each slot widens, so `enum { .ok(enum { .x }) }`
  flows into `enum { .ok(enum { .x, .y }), .err }`.

Destructure with `switch` — the elimination form, and the only way to recover a
payload. It is an **expression** (its type is the join of the arm bodies) and it
is **total**: the `case`s must cover the subject's static type or the compiler
errors. The catch-all is `case _:` (there is no `else`); zero cases is valid
only on an uninhabited subject.

```scl
let describe = fn(s: Status)
    switch s
        case .active(port): "on {port}"     // binds the payload slot
        case .idle: "idle"
        case .error(msg): "error: {msg}"

let head = fn(l: List)
    switch l
        case .cons(first, _): first          // _ ignores a slot; arity must match
        case .empty: 0
```

Patterns (v1): variant `.name(<pat>, …)`, a variable binding, wildcard `_`, and
`nil` (only on an optional subject — adds the none case to coverage). A variant
pattern's arity must equal the payload's; ignore a slot with an explicit `_`.
Patterns may nest and overlap — **first match wins**, top to bottom — and a fully
shadowed arm is an unreachable-case error. A top-level binder types at the
*residual* (the subject minus already-consumed variants). Literals, ranges,
list/cons patterns, `@`-bindings, and `case … if` guards are **not yet
supported**.

## Expressions

Everything is an expression; there are no statements inside function bodies.
`;` is not a statement terminator — it is the **discard operator**, itself an
expression: `a; b` evaluates both operands and yields `b`. There is no trailing
`;` anywhere.

```scl
let status = if (enabled) "on" else "off"   // if-expression; parens required
let maybe = if (count > 0) count            // no else → type is Int?
let two = let x = 1; x + 1                  // inline let: binding; body
let last = (prepare(); result)              // discard: both run, value is `result`
let gated = with (db) { url: db.url }       // exists only once `db` does (below)

// Anonymous functions (closures). Param types inferred when context knows them.
let double = fn(x: Int) x * 2
let doubled = List.map([1, 2, 3], fn(x) x * 2)

// Aggregating: fold carries an accumulator, reduce starts from the first
// element and is nil-valued on an empty list. Prefer these over recursion.
let total = List.fold([1, 2, 3], 0, fn(acc: Int, x: Int) acc + x)  // 6
let biggest = List.reduce([3, 1, 2], fn(a: Int, b: Int) if (a > b) a else b)

// Generics, with optional subtype bounds
let identity = fn<T>(x: T) x
let getName = fn<T <: { name: Str }>(item: T) item.name

// A recursive binding: annotate it, and the recursive call carries the
// annotated type everywhere in the body. Unannotated, the compiler asks for
// the annotation wherever it cannot infer the call's result ("the type of a
// recursive binding cannot be inferred in this position").
let sum: fn([Int]) Int = fn(xs)
    switch List.first(xs)
        case nil: 0
        case x: x + sum(List.skip(xs, 1))

// Comprehensions: for iterates, if filters, in spreads a whole collection;
// clauses chain and everything splices in flat (see "List elements" and
// "Dict entries" below)
let evens = [for (x in items) if (x / 2 * 2 == x) x]
let pairs = [for (x in xs) for (y in ys) x + y]
let byName = #{for (x in items) "k{x}": x}  // same forms, whole entries
let flat = [for (xs in xss) in xs]          // spread under a for: flatten
let merged = #{in defaults, "region": "eu"} // splice a dict; later writes win

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
Narrowing passes through an ascription to the type a value already has (`if ((x
!= nil) as Bool) x` narrows `x`); one that *changes* the type refines only the
cast view, so a narrowing cast in a condition leaves its operand alone. Neither
carries a fact proved about a *field* through a cast of the record around it. `;`
binds loosest and is the only right-associative one — `a; b; c` is `a; (b; c)`,
its type is the last operand's, and any type may be discarded.

A discard is a textual sequence, not an ordering: the left operand runs (that is
the point — only its *value* is dropped), but `;` adds no dependency edge, a
pending left operand neither blocks nor infects the result, and the result's
dependencies are the right operand's alone. Order resources by referencing their
outputs, as everywhere else.

Where a `;` lands is settled by one rule: **it belongs to the nearest enclosing
binding still awaiting one; only an unclaimed `;` is a discard.** So a `let`'s
bound value and module scope reserve theirs — `let x = A(); B()` at module scope
is an inline let (`x` = `A()`, in scope over `B()` only, so a later mention of
`x` is an undefined-variable error), and `export let x = A(); B()` is a hard
error suggesting `export let x = (A(); B())`. Brackets of every kind reset that
(`f(a; b, c)` is two arguments), keyword bounds do not — parenthesize to chain
in a then-branch, a `switch` arm or a `try` body. A trailing body swallows a
following `; e`, so `if (c) A(); B()` makes `B()` conditional; write
`(if (c) A()); B()` for an unconditional one.

`with (subject) body` gates the body on the subject's **existence**. A pending
value anywhere inside the subject makes the whole expression pending — the body
does not run. Once the subject fully exists, the body evaluates carrying the
subject's dependencies: resources declared while it runs (even inside functions
it calls) depend on whatever the subject depends on, and so does the resulting
value. It is the manual form of the edge `if (a.ready) …` creates from control
flow — same gating, no condition to invent, and the type is simply the body's
(no optional wrapping like an else-less `if`). Grammatically it sits with `if`:
parens required around the subject, and the trailing body extends rightward, so
`with (a) A(); B()` gates `B()` too.

### List elements: `for` and `if` generate, `in` spreads, all splice in flat

Inside a `[…]` literal, `for`, `if`, and `in` are *element forms*, not
expressions. A plain expression element contributes exactly one value; a
`for`/`if` element *generates* zero or more; a spread `in xs` splices every
element of `xs` in at its position. The body of a `for`/`if` element is itself
an element, so the forms chain arbitrarily — and everything a chain generates
is spliced into the surrounding list at that position, in order. The result is
always one flat list; comprehension elements never introduce nesting.

```scl
[
    1,                           // plain element — exactly one value
    for (e in [2, 3, 4]) e,      // generates 2, 3, 4 — spliced, not nested
    if (false) 5,                // generates nothing (no nil placeholder)
    for (e in [10, 20, 30])      // chained clauses multiply out:
        if (e > 15)              //   drops e = 10
            for (l in List.range(e))
                l + 1,           // runs once per (e, l) pair: 20 + 30 = 50 times
]
// = [1, 2, 3, 4, 1, 2, …, 20, 1, 2, …, 30] — one flat [Int], 54 elements
```

- **A `for` under a `for` is a cross product**: the innermost expression runs
  once per surviving combination of the binders in scope, each run
  contributing one element. To *keep* nesting, make the body a list literal
  of its own: `[for (x in xs) [x]]` is `[[Int]]`.
- **`in xs` splices a whole list** — the `for` item with the binder dropped:
  `[in xs]` copies, `[1, in xs, 4]` concatenates in place,
  `[for (inner in nested) in inner]` flattens one level, and
  `[if (extra) in extras]` includes a block conditionally. The operand must
  itself be a list; its element type joins like any element's. Spreading is
  always the keyword `in` — there is no `...`/`..` symbol (`..` is a path
  literal, so `[..]` is a one-element list holding a path).
- `for (x in e)` iterates a list only (`e: [T]`); any other iterable type is
  a compile error.
- **The element `if` takes no `else` and is not the optional-typed else-less
  `if` expression**: `[if (c) n]` is `[Int]` with zero or one elements — no
  `nil` enters the list. With an `else` it parses as an ordinary expression
  element again, contributing exactly one value: `[if (c) a else b]`.
- **The guard narrows like an `if`'s then-branch**: the condition is assumed
  true inside the guarded element, so `[if (x != nil) x]` is `[Int]` for
  `x: Int?`. No `else` means no negation side, and the fact is scoped to the
  guarded element — siblings and code after the literal still see `Int?`.
- An `if` can guard any element, including a whole `for` chain — so
  `[if (extra) for (x in xs) x]` splices `xs` conditionally.
- Each generated value's type joins into the list's element type exactly like
  a plain element's does.

### Dict entries: the same forms, writing whole `key: value` entries

A `#{…}` literal takes the same item forms — `for` and `if` around a whole
entry rather than a value, and the spread `in d` moving whole entries by
itself. They chain and nest the same way, and everything a chain writes
lands in the enclosing dict — never a dict of dicts.

```scl
#{
    "first": 123,                // plain entry — exactly one key
    if (false) "second": 345,    // writes nothing (no nil-valued key)
    for (n in [1, 2, 3])         // chained clauses combine:
        if (n > 1)               //   drops 1
            "v{n}": n,           // writes v2 and v3
}
// = #{ "first": 123, "v2": 2, "v3": 3 }
```

- The binder is in scope for the **key and the value alike**, so a generated
  key is usually built from it: `#{for (p in ports) "port-{p}": p}`.
- **Last write wins, in the first write's position.** A generated key landing
  on one the dict already holds replaces that value in place — no duplicate
  key, no error, and `Dict.keys` order unchanged:
  `#{"v1": 0, for (n in [1, 1, 2]) "v{n}": n * 10}` is `#{"v1": 10, "v2": 20}`.
  Only a *hand-written* repeat of a constant key warns.
- **`in d` splices a whole dict**: the operand's entries land at that
  position, in order, indistinguishable from hand-written ones —
  last-write-wins applies across written and spliced keys alike, first write
  fixing the position, and never warns. `#{in defaults, "b": 3}` overrides
  `defaults`' `"b"` in place; `#{in a, in b}` is exactly `Dict.merge(a, b)`;
  `#{if (c) in overrides}` splices conditionally. The operand must be a dict;
  its key and value types join like an entry's.
- **A dict's shape is its key set; values decide nothing about it.** A
  pending entry *value* (an unmaterialized resource output, say) stays in its
  slot: `Dict.length`/`Dict.keys` and sibling entries answer while that entry
  alone defers. Only what decides the keys makes the whole literal pending —
  a pending key, `if` condition, `for` iterable, or spread operand.
- Neither `for` nor `if` adds optionality: `#{if (c) "k": 1}` is `#{Str: Int}`
  with zero or one entry, and a generator over an empty list still fixes both
  types.
- The guard narrows exactly as in a list, with the fact in scope for the key
  and the value alike: `#{if (x != nil) "p{x}": x}` is `#{Str: Int}` for
  `x: Int?`.
- An `else` makes the `if` an ordinary expression, which in item position is
  the entry's **key**: `#{if (c) "a" else "b": 1}` always writes one entry.
- `for (x in e)` iterates a list only, exactly as in a list literal. To drive
  one from a dict, bridge through `Std/Dict.entries` (or reach for
  `Dict.map`/`Dict.filter`, which transform a dict directly).

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
- **Doc comments are markdown**: `///` documents the item that follows it, and
  a leading `//!` block at the top of a file documents the module itself.
  Editor tooling shows a `///` doc on hover, and the module reference served
  by the instance is generated from both.

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

To import modules from another repository, declare the dependency in
`Package.scle` at the repo root — itself an SCLE file whose value is a
`Std/Package.Manifest`:

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

The manifest is the only way in: importing a repo that `dependencies` does not
name is a `module not found` compile error, even if that repo is deployed and
running. Add the entry first.

A dependency may be in **any** org. Declaring it only asks — the importing
repo's deployment role must hold `repository:View` on the dependency's
`org/repo`, checked live on every pass before any of its source is read. Within
your own org the default `Super` deployment role already covers it; across orgs
the *dependency's* org grants it, by naming the importer's deployment-role QID
in an `IAM.Policy` of its own (policies are always evaluated in the object's
org, so no org can grant itself access to another's repos). Missing the grant
fails the deployment with an incident naming the repo and the verb.

A policy subject must name its org outright: a `*` in the subject's organization
identity — the prefix before the first `/` or `::`, a bare `*` included — is
rejected at authoring time (a `*` past it, like `"acme::*"` or `"acme/*::…"`, is
fine). The one exception is
the bare `Anonymous` sentinel, which an org names as a subject to publish reads
— and, via `repository:View`, cloneable source over `ssh://nobody@…` — to
*anyone*: it is consulted whoever the caller acts as, signed-out callers, other
orgs' roles and this org's own members alike. Only a small read allowlist ever
takes effect for it.

One exception to "any org": `Skyr/Resource` custom-resource instances must be
registered against a definition in the *same* org — a cross-org consumer
compiles but its instance transitions are rejected.

Ownership rule: a resource belongs to the deployment whose own code path
reaches the resource call. Reading a foreign repo's top-level resource output
is remote state; calling a function it exports creates a resource owned by
*your* deployment.

`Std/Env` deliberately follows the opposite rule: an `Env.*` read answers for
the package its source is *written in*, so a branch-/tag-pinned dependency's
code sees the dependency's own current deployment even when you call it. In a
hash-pinned dependency there is no deployment behind the package, and an
`Env.*` read raises the catchable `Env.NoDeployment`. (`Std/Secret` is
caller-based like effects: lookups always resolve the deploying repo's
secrets.)

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

- **Dependencies are output references — in inputs *or* in control flow.** If
  B's inputs mention `a.someOutput`, B waits for A; so does a B declared inside
  `if (a.someOutput …)`, since whatever decided B exists is a dependency of B
  too (and outlives it on teardown). No reference anywhere, no ordering — and
  `with (a) B({...})` adds the edge by hand, gating B on `a`'s existence with
  no condition to invent (see Expressions).
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
  Each read answers for the package it is written in — foreign-package code
  reports the foreign repo's own deployment, not the evaluating one.
- **A few resources are commands: declaring one is the request, and there is
  nothing to read back.** `Skyr/Rollout.Rollback({ reason: "…" })` asks the
  platform to roll this deployment back. Whether to do it is ordinary control
  flow — write `if (c) Rollout.Rollback({ reason: "…" }) else nil` and the
  request exists only on the deployments whose evaluation reaches it. Its
  identity is *this deployment plus the reason*, so the same reason twice is one
  request and a reconcile retry keeps asking for the same one, while the same
  reason in a later deployment is a new request. Rolling back needs an explicit
  `environment:Rollback` grant on the repository's deployment role, and a
  deployment with no rollback target tears its environment down instead of
  restoring anything — read the `deploy` skill's rollback section before wiring
  one up.

**Do not guess plugin-module signatures.** Plugin modules (`Skyr/Container`,
`Skyr/DNS`, `Skyr/IAM`, `Skyr/PKI`, `Skyr/HTTP`, `Skyr/Rollout`,
`Skyr/Random`, `Skyr/Artifact`, `Skyr/Resource`, `Skyr/AWS/*`,
`HashiCorp/Random`, …) are served by the instance you deploy to, and their
inputs/outputs are precise. Look them up before use, and let `skyr check`
confirm.

You can also define your *own* resource types: `Skyr/Resource.Definition<T>`
declares a type whose instances (created via the constructor the definition
returns, typically re-exported to consumer repos) register themselves with the
defining deployment, which collects the live set and folds it into its own
resources. Look up `Skyr/Resource` for the exact shape.

When a function of yours wraps a resource call and returns a record built from
it, wrap the return in `with`:

```scl
export let makeFile = fn(name: Str, contents: Str)
    let made = Artifact.File({ name, contents });
    with (made) { name, url: made.url }
```

Gated on the resource's existence, every field of the record carries the edge —
fields that merely echo inputs (`name` here) included, so a consumer can depend
on any of them and still wait for the file. First-party plugin constructors
follow the same idiom.

## Secrets

Read a deployment's secrets with `Std/Secret`. A secret's plaintext never enters
SCL — `Secret.get(name)` returns an opaque `Ref { name, createdAt, qid }` whose
`qid` is a Secret Version QID string, and a name that doesn't resolve (not set,
or the deployment's role lacks `secret:View` on it) raises the catchable
`Secret.NotFound`:

```scl
import Std/Secret

let db = Secret.get("db-password")   // raises Secret.NotFound if unresolved
// db.qid — pass this to a consumer; it is never the plaintext
```

Hand the `qid` to something that resolves it at deploy time. In the container
plugin, a pod's or container's `env` and an ephemeral volume's `files` seed are
maps whose values are `.literal("…")` or `.secret(qid)` — you write
`.secret(Secret.get(name).qid)`, which the `deploy` skill covers. The values
themselves are set out of band with `skyr secrets set|list|delete`, never
committed to git.

## Tests

A module carries its own tests inline, marked with the `@test` annotation.
Marked statements are type-checked wherever the module is compiled and left
out of the program a deployment evaluates, so they cost a deployment nothing.
The annotation only marks; the vocabulary comes from `Std/Test`.

```scl
import Std/Option

@test
import Std/Test                 // mark the import too — it is test-only

let podName = fn(environment: Str, service: Str?)
    "{environment}-{Option.unwrap(service)}"

@test
Test.group("podName", fn()
    Test.it("joins the environment and the service", fn()
        Test.expect(podName("main", "api")).toEqual("main-api")
    );
    Test.it("refuses a service that is not set", fn()
        Test.expect(fn() podName("main", nil)).toRaise(Option.UnexpectedNil)
    )
)
```

- `Test.group(name, fn() …)` nests; `Test.it(name, fn() …)` declares one
  **case** — spelled `it` because `case` is a keyword. Sibling calls inside a
  body are joined with `;`, the discard operator (see Expressions).
- Matchers are `Test.expect(v).toEqual(expected)` and
  `Test.expect(fn() …).toRaise(SomeException)`. `toEqual`'s argument is typed
  as the value's own type, so a mismatched literal is a *compile* error, not a
  failing case.
- A case fails if it raises uncaught, if an assertion is not satisfied, or if
  it reaches no verdict at all. A failure fails that case only — the rest run.
- **Any module statement can be marked** — `import`, `let`, `type`, or a bare
  expression — but `@test export` is a hard error, and a statement *without*
  the mark may not reference a binding, type, or import a `@test` statement
  introduced. Test code reading ordinary code is free.
- **An unmarked test statement runs nowhere** — a deployment no-ops it, and a
  test run drops it as an unmarked bare expression. Mark it.

**Tests are verdicts over values, never over resources.** A test run holds no
resource state, so every resource output reads pending and a case that reaches
for one *fails*, naming the resource it awaited. `Secret.get` and `Time.now`
raise `Secret.Unavailable` / `Time.Unavailable`, and `Std/Env` reports a fixed
placeholder identity; `Path.read` still reads the package's own files. So
hoist the logic worth checking into pure functions over plain values — deriving
a name, assembling an env map, validating a configuration — and let the
resource declarations pass the results in. Anything that has to *happen* at
deploy time (a probe, a smoke job) is not a test's business.

## Looking up documentation

The Skyr docs are served as raw markdown, ideal for fetching and grepping.
Current canonical location (if the repo deploys to a different Skyr instance,
use that instance's host instead):

```sh
curl -s https://skyr.foo/llms.txt                        # index of all doc pages
curl -s https://skyr.foo/~docs/scl/reference.md          # every documented module, one line each
curl -s https://skyr.foo/~docs/scl/syntax.md             # complete syntax reference
curl -s https://skyr.foo/~docs/scl/types.md              # type system in depth
curl -s https://skyr.foo/~docs/cross-repo-imports.md     # Package.scle details
curl -s https://skyr.foo/~docs/resources.md              # what declaring a resource means at deploy time
curl -s https://skyr.foo/~docs/iam.md                    # roles and policies: Skyr/IAM, in words and examples
curl -s https://skyr.foo/~docs/terraform.md              # Skyr/AWS and the other provider-backed modules
```

A module's documentation is generated from its source and reached in two
curls. The reference index above groups every documented module by namespace —
`Std/*` and every first-party `Skyr/*` module — one line each with a summary,
each linking to that module's own markdown.

A module's markdown is shaped for grepping: the module's narrative runs from
the `# <Namespace>/<Module>` title down to the `## Types` /
`## Functions & Values` sections, carrying `##` headings of its own, and each
export then gets one `### <ExportName>` section — types first, then functions
and values — with its signature, its documentation, and a bullet per field or
parameter.

```sh
# the whole module
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md

# every export it has, in page order
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md | grep -n '^### '

# one export's whole section, ending at the next export's heading
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md | sed -n '/^### Pod$/,/^### /p'
```

Export names in the headings are unqualified — `### Pod`, not
`### Container.Pod` — and a name that is both a type and a constructor gets a
section apiece, so the range above prints both. Sections run long (a resource
constructor's is routinely 60+ lines), so prefer that form over `grep -A <n>`,
which cuts off mid-section. Modules generated from a Terraform provider's live
schema (`Skyr/AWS/*`, `HashiCorp/Random`) are not in the index —
`~docs/terraform.md` covers how they work, and their field names come from the
provider's own schema, so trust editor completions over guessing.

For exact language semantics beyond the docs (typing rules, evaluation order),
the formal SCL specification PDF ships alongside Skyr releases on dl.skyr.cloud.

## Verifying your work

Always finish by formatting and type-checking, and by running the package's
tests when it has any; never hand back unchecked SCL.

```sh
skyr fmt --write Main.scl   # canonical formatting, per file (omit --write to preview)
skyr check                  # parse + resolve + type-check the whole package (no deploy)
skyr test                   # run the package's @test cases (no deploy, no backends)
skyr repl                   # interactive: evaluate expressions, inspect types
```

- `skyr check` checks the package rooted at `Main.scl` in the current
  directory (override with `--root`). It stops before evaluation, so it
  catches syntax, resolution, and type errors — not eval-time errors like
  conflicting duplicate resource declarations.
- `skyr test` takes the same `--root`/`--package` flags. It prints one line
  per case — `pass`/`fail`/`raise`/`await`, then the case's group path and
  name — with the expectation and the source trace under each failure, and
  exits nonzero if any case failed, so it drops into a pre-commit hook or a CI
  step. It deploys nothing and holds no resource state. Run it after adding or
  changing test code, and after changing anything a case covers: a pushed
  commit is gated on the same verdict before it applies anything, so a red
  package will not roll out.
- The package name is inferred from the `skyr`/`origin` git remote; override
  with `skyr check --package org/repo` when the inference fails (you'll see
  it default to `Local`).
- `Skyr/*` imports are type-checked against the frontends served by the
  instance's API: the first such check fetches them over the network (then
  disk-caches). Pure-`Std/` packages check fully offline. If a `Skyr/*`
  import won't resolve and the machine is offline with a cold cache, that's
  why — it is not necessarily a typo.
- Cross-repo imports resolve through the instance too, and need the
  dependency repos to exist there *and* be declared in `Package.scle`. An
  undeclared repo is a `module not found` error, not a typo.

Fix diagnostics in source order — SCL errors carry causal chains for nested
type mismatches, and an early resolution failure often cascades.
