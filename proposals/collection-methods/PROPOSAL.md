---
status: Draft
created: 2026-09-18
from_commit: b56a2a9
---

# Proposal: Methods on Built-in Collection Types

This proposal is less settled than most in this repository, and it stays
that way on purpose. The goal isn't `arr.filter(fn)` specifically — it's
figuring out what category `Array` (and, eventually, `Tuple`, `Set`, and
`Map`/`Dict`) actually belong to, and whether the mechanism already being
built for primitive types fits them or not. Until that's answered, a
concrete design for "Array methods" would be guessing. This document is
mostly [Questions] for that reason.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing
process to understand the changes being proposed and what impacts they will
have. That research is largely tracked in the form of [Questions].

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it is
defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true
  for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is
  after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c`
  code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Method-call syntax, `receiver.method(args)`, is handled by one function,
`invoke()` in `src/vm.c`. Checked against a clean `main` build, it recognizes
exactly two kinds of receiver:

```c
static bool invoke(ObjString *name, int argCount) {
  Value caller = peekStack(argCount);

  if (IS_STRUCT(caller)) {
    return invokeStatic(AS_STRUCT(caller), name, argCount);
  }

  if (!IS_INSTANCE(caller)) {
    runtimeError(&vm, "Only instances have methods.");
    return false;
  }
  // ...instance method lookup via the struct's method table...
```

Anything that isn't a struct or a struct instance — an array, a string, a
number — hits `"Only instances have methods."` and fails. `arr.push(x)` and
`"hi".trim()` both error out today for the same reason.

[Primitive Impls Proposal] is already closing this gap for `f64`, `string`,
`bool`, and `unit` — but not through `invoke()`. An unmerged branch,
`initial-primitive-traits`, implements it: `impl`/trait blocks on these four
types are resolved **statically, at compile time**, into direct calls to
mangled functions, restricted to `stdlib/stdlib.krb` (not user code). A
method body is ordinary Kirby that mostly just delegates to an existing
native:

```kirby
// stdlib/stdlib.krb, on initial-primitive-traits
impl Display for f64 {
    pub fun toString(self): string = numberToString(self);
}
```

This works because a value's primitive type is a single, fixed, compile-time
fact — there's no subtyping or polymorphism among `f64` values, so the
compiler always knows exactly which `toString` a given call site means, with
no need for the runtime method-table lookup `invoke()` does for structs.

`Array` is explicitly **not** part of that work. [Primitive Impls Proposal]
says so directly:

> Explicitly out of scope: `Array[T]` and other built-in container types.
> `Array` is ephemeral today — it isn't backed by a Kirby struct, and its
> future shape (once generics land) isn't settled.

And the implementation on `initial-primitive-traits` shows exactly where it
stops. The type-checker's gate function treats `Array` the same as the four
real primitives:

```c
static bool tokenIsPrimitiveTypeName(Token *token) {
  return tokenTextEquals(token, "unit") || tokenTextEquals(token, "bool") ||
         tokenTextEquals(token, "string") || tokenTextEquals(token, "f64") ||
         tokenTextEquals(token, "Array");
}
```

but the function that actually resolves a primitive name to a concrete
`Type*` for `impl` to bind `Self` to returns `NULL` for `"Array"`, with a
comment spelling out why:

```c
// Returns the primitive singleton `token` names (unit/bool/string/f64), or
// NULL if it names something else. Notably NULL for "Array" --
// tokenIsPrimitiveTypeName() above treats that as primitive-ish too, but
// it isn't a scalar type and can't carry impl/trait-impl methods (Phase
// 4b doesn't extend that far -- see TYPE_SYSTEM_RFC.md).
static Type *typchkPrimitiveTypeNamed(Token *token) { ... }
```

That comment references a `TYPE_SYSTEM_RFC.md` that doesn't exist in either
repository as of this branch — see [Q-rfc].

The gap is bigger than `Array` alone. `Tuple` is a single, undetailed bullet
under "Future State" in `docs/CHANGELOG.md`. `Set` and `Map`/`Dict` don't
appear anywhere — not in the changelog, not in any existing proposal, not in
the codebase. All of them will eventually want methods (`arr.filter(fn)`,
`set.has(x)`, `map.get(k)`, whatever their eventual syntax turns out to be),
and none of them are "primitive" in the sense [Primitive Impls Proposal]
uses that word — they're heap-backed and dynamically sized, closer in shape
to a struct than to `f64`, but they aren't structs either, and user code
can't `impl` them the way it can `impl` a struct.

### Relationship to other proposals

- [Primitive Impls Proposal] is the direct precedent this proposal has to
  either extend or diverge from. It also already names this exact gap
  (quoted above) and stops at its edge deliberately — this proposal exists
  to pick up where it stopped, not to re-decide anything it already settled
  for `f64`/`string`/`bool`/`unit`.
- [Generic Types Proposal] matters because `Array[T]` annotations are
  rejected today (`"Generic types aren't supported yet"`, checked against a
  clean build). A properly typed `Map[K, V]` or `Set[T]` — or even a
  generically-typed `Array[T].filter`, whose predicate needs to know its
  argument's type — depends on generics landing first. See [Q-generics-order].
- [Tuples Proposal] designs the `Tuple` type this proposal mentions. It
  writes the type `(f64, string)` rather than `Tuple[f64, string]`
  ([Q-type-spelling]), and gives a tuple a fixed size and item types known
  at compile time, which bears on [Q-category].

## The Changes

Everything below is a candidate direction, not a decision — see
[Questions] for what's actually still open about each.

### Part 1 — Name the category

Introduce **collection type** as a term for this family — `Array` today;
`Tuple`, `Set`, and `Map`/`Dict` prospectively — distinct from
[Primitive Impls Proposal]'s "primitive type." Naming it is worth doing on
its own: right now `Array` is described only negatively, as a primitive that
doesn't qualify for the primitive mechanism. Whether "collection type" is
even one coherent category, or actually two with different needs, is
[Q-category].

### Part 2 — Candidate direction: extend the static mechanism

`tokenIsPrimitiveTypeName` already accepts `"Array"`; only
`typchkPrimitiveTypeNamed` stops short. Neither `Array`, nor (presumably)
`Tuple`, `Set`, or `Map` involve subtype polymorphism — there is exactly one
`Array` type, not an open hierarchy of them, the same way there's exactly
one `f64`. That's the property that lets the primitive mechanism resolve
`x.toString()` at compile time with no runtime lookup. On that reasoning,
collection types might fit the same static/mangled mechanism
[Primitive Impls Proposal] already built, rather than needing a different
one — but "ephemeral" and "ephemeral, isn't backed by a Kirby struct" (that
proposal's own words) suggest the compiler may not currently track enough
about an `Array` value — and won't, for `Map`/`Set`'s key/value types,
without generics — to resolve a call the way it resolves one for `f64`. This
needs to be checked against the actual type-checker code, not assumed;
see [Q-static-fit].

### Part 3 — Candidate direction: extend `invoke()`'s dynamic dispatch

The alternative: give collection types a real method table and route
`arr.filter(...)` through `invoke()`, the same path structs already use,
rather than resolving it away at compile time. This doesn't depend on the
compiler knowing a value's exact type statically, but it's a different, and
by the look of `invoke()` today, larger change to the VM than Part 2 — a new
receiver kind alongside `IS_STRUCT`/`IS_INSTANCE`, and something for that
kind to look method names up in. Whether this is worth building as a
separate mechanism from Part 2, or only if Part 2 turns out not to fit, is
[Q-two-mechanisms].

### Part 4 — Explicitly out of scope

This proposal does not design `Tuple`, `Set`, or `Map`/`Dict` themselves —
none of them exist yet. It only tries to scope the _mechanism_ question, so
that whichever of them get built later — and `Array`, which already
exists — have somewhere to attach methods to.

## Questions

### **Q:** Is "collection type" one category, or two?

<!-- [Q-category]: #q-is-collection-type-one-category-or-two -->

**Status:** Open

`Array`, `Set`, and `Map`/`Dict` are all dynamically sized and mutable.
`Tuple`, once it exists, is likely fixed-size and its element types likely
fully known at compile time (`Tuple[f64, string]` and `Tuple[f64, f64]`
would be different concrete types) — closer in shape to a primitive than to
`Array`. Lumping all four together as "collection types" may be grouping two
different problems under one name.

Options: (a) one category, one mechanism, accepting that it has to handle
both the fixed-shape and dynamically-sized cases; (b) two categories —
`Tuple` follows [Primitive Impls Proposal]'s path since it's closed and
statically known, `Array`/`Set`/`Map` follow whatever this proposal lands
on. (b) is more honest about the actual differences but means committing to
two mechanisms instead of one.

### **Q:** Does `Array` actually fit the static/mangled mechanism?

<!-- [Q-static-fit]: #q-does-array-actually-fit-the-staticmangled-mechanism -->

**Status:** Open

Part 2 is a hypothesis based on `Array` not having subtype polymorphism, not
a checked fact. It needs to be checked directly against `typecheck.c`: does
the checker currently know, at a given call site, that a value is
specifically an `Array` (as opposed to just "some value"), the way it always
knows a value's type is specifically `f64`? [Primitive Impls Proposal]'s own
description of `Array` as "ephemeral" suggests maybe not, but that
proposal's authors were describing why they left `Array` out, not
necessarily making a precise claim about what the checker tracks.

### **Q:** Are `TYPE_SYSTEM_RFC.md` and "Phase 4b" something to reconcile with first?

<!-- [Q-rfc]: #q-are-type_system_rfcmd-and-phase-4b-something-to-reconcile-with-first -->

**Status:** Open

A code comment on `initial-primitive-traits` references a `TYPE_SYSTEM_RFC.md`
and a "Phase 4b" as the reason non-scalar `impl` targets aren't handled yet.
Neither the file nor any other reference to a phased plan exists in this
repository or the proposals repository as of this branch. If that RFC exists
somewhere else (or existed and was removed), this proposal should be
reconciled with it rather than duplicating or contradicting a plan that's
already been thought through — worth checking with whoever wrote that
comment before this proposal goes further.

### **Q:** Does this wait on generics, or ship an untyped version first?

<!-- [Q-generics-order]: #q-does-this-wait-on-generics-or-ship-an-untyped-version-first -->

**Status:** Open

`Array` today is itself untyped/element-agnostic (`Array[T]` annotations are
rejected — [Generic Types Proposal]). A `filter`/`map`-style method's
predicate parameter has the same "what type is an element" problem `Array`
itself already has, and already ships without an answer to today.

Options: (a) wait for [Generic Types Proposal] to land, so collection
methods can be properly typed from the start; (b) ship collection methods
untyped first (mirroring how `Array` itself already works, and how
`arrPush`/`arrContains`/etc. already take and return untyped `Value`s
today), and revisit typing once generics exist. (b) unblocks this sooner but
means a second pass later to add proper typing; (a) avoids that second pass
but is blocked on a proposal with its own open questions.

### **Q:** Should this start from `Array` alone, or wait until `Tuple`/`Set`/`Map` are at least sketched?

<!-- [Q-scope-order]: #q-should-this-start-from-array-alone-or-wait-until-tuplesetmap-are-at-least-sketched -->

**Status:** Open

Designing the mechanism against `Array` alone risks shaping it around
`Array`'s particular quirks (no keys, single element type, index-based
access) in ways that don't generalize to `Map` (two type parameters, no
inherent ordering) or `Set` (no duplicate/indexing semantics at all).

Options: (a) design and ship for `Array` now, adjust the mechanism later if
`Set`/`Map` don't fit it; (b) at least sketch `Set` and `Map`'s shape
(without fully designing them) before committing to a mechanism, so the
mechanism is checked against more than one case up front. (a) makes
progress now on the type that already exists; (b) costs more up front but
lowers the odds of a mechanism that has to be reworked for the second
collection type that uses it.

## Glossary

These are both technical and non technical terms used throughout the
proposal.

- **Changes**: Changes refer the proposed changes in this document
- **Collection type**: a term this proposal introduces for Kirby's
  built-in, dynamically-sized aggregate types — `Array` today, `Set` and
  `Map`/`Dict` prospectively. Distinct from a **primitive type**
  ([Primitive Impls Proposal]'s term for `unit`/`bool`/`string`/`f64`: fixed
  shape, no fields, resolved statically) and from a user-defined **struct**.
  Whether `Tuple` belongs in this category or with primitives is
  [Q-category].
- **Static/mangled resolution**: resolving a method call to a specific
  function at compile time, by rewriting (mangling) the call into a direct
  reference to a generated function name, rather than looking the method up
  in a runtime table. This is how [Primitive Impls Proposal] resolves calls
  like `x.toString()` for `f64`.
- **Dynamic dispatch**: resolving a method call at runtime, by looking the
  method name up in the receiver's method table. This is how `invoke()`
  already resolves method calls on struct instances.
- **Receiver**: the value a method is called on — the `x` in `x.method()`.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement

<!-- Proposals -->

[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-type-spelling]: ../tuples/PROPOSAL.md#q-is-a-tuple-type-written-a-b-or-tuplea-b

<!-- Questions -->

[Q-category]: #q-is-collection-type-one-category-or-two
[Q-static-fit]: #q-does-array-actually-fit-the-staticmangled-mechanism
[Q-rfc]: #q-are-type_system_rfcmd-and-phase-4b-something-to-reconcile-with-first
[Q-generics-order]: #q-does-this-wait-on-generics-or-ship-an-untyped-version-first
[Q-scope-order]: #q-should-this-start-from-array-alone-or-wait-until-tuplesetmap-are-at-least-sketched
