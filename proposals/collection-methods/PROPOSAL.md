---
status: Draft
category: feature
created: 2026-09-18
from_commit: 2f96a2e
---

# Proposal: Methods on Built-in Collection Types

This proposal is less settled than most in this repository, and it stays
that way on purpose. The goal isn't `arr.filter(fn)` specifically — it's
figuring out what category arrays (and, eventually, tuples, `Set`, and
`Map`/`Dict`) actually belong to, and whether the mechanism already being
built for primitive types fits them or not. Until that's answered, a
concrete design for array methods would be guessing. This document is
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

Method-call syntax, `receiver.method(args)`, is handled at run time by one
function, `invoke()` in `src/vm.c`. Checked against a clean `main` build, it
recognizes exactly two kinds of receiver:

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
number — would hit `"Only instances have methods."` and fail. The type checker
stops such a call before it runs: `arr.push(2)` on an array of `i64` reports
`Can't access '.push' on a [i64].`, and `"hi".trim()` reports
`Can't access '.trim' on a string.` (`typchkInferGet` in `src/typecheck.c`).

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

The branch was written when `f64` was Kirby's only number type and `Array`
was a type name. On main today there are ten number kinds, and arrays are
written `[T]`.

This works because a value's primitive type is a single, fixed, compile-time
fact — there's no subtyping or polymorphism among `f64` values, so the
compiler always knows exactly which `toString` a given call site means, with
no need for the runtime method-table lookup `invoke()` does for structs.

Arrays are explicitly **not** part of that work. [Primitive Impls Proposal]
says so directly:

> Explicitly out of scope: arrays (`[T]`) and other built-in container
> types. They aren't backed by a Kirby struct, and how methods on them
> should work isn't settled.

And the implementation on `initial-primitive-traits` shows exactly where it
stopped. The branch's type-checker gate function treats `Array`, the array
type name at the time, the same as the four real primitives:

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
repository — see [Q-rfc].

On main, the same gate function lives in `src/token.c` and recognizes only
`unit`, `bool`, `string` and `f64`. `Array` is not a type name, so
`impl Display for Array` reports `Unknown type 'Array'.`

The gap is bigger than arrays alone. Tuples are designed in the
[Tuples Proposal] but not built. `Map` is designed in the [Map Proposal]
(from [Issue #70]) but not built. `Set` is not in the changelog or the
codebase, and no proposal designs it (the [Map Proposal] lists it as out of
scope). All of them will eventually want methods (`arr.filter(fn)`,
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
- [Generic Types Proposal] matters for `Map[K, V]` and `Set[T]`, which need
  generic type arguments. They work for structs a program declares
  (`Box[f64]`, [PR #169]), and no built-in type takes them yet. Arrays don't
  need them: `[i64]` is an array of `i64` and the checker checks its items.
  See [Q-generics-order].
- [Map Proposal] designs `Map[K, V]`, with its operations written as natives (`@mapGet`, `@mapSet`) and a question about whether they become methods ([Q-api-shape]). It is the sketch of `Map`'s shape that option (b) of [Q-scope-order] asks for.
- [For-Each Proposal] adds `for (item in collection)`. Methods like `filter` and `map` on arrays would be built on the same iteration rule.
- [Tuples Proposal] designs the tuple type this proposal mentions. It writes
  the type `(f64, string)` ([Q-type-spelling]), and gives a tuple a fixed
  size and item types known at compile time, which bears on [Q-category].

## The Changes

Everything below is a candidate direction, not a decision — see
[Questions] for what's actually still open about each.

### Part 1 — Name the category

Introduce **collection type** as a term for this family — arrays today;
tuples, `Set`, and `Map`/`Dict` prospectively — distinct from
[Primitive Impls Proposal]'s "primitive type." Naming it is worth doing on
its own: right now arrays are described only negatively, as something that
doesn't qualify for the primitive mechanism. Whether "collection type" is
even one coherent category, or actually two with different needs, is
[Q-category].

### Part 2 — Candidate direction: extend the static mechanism

On the branch, `tokenIsPrimitiveTypeName` already accepted `"Array"`; only
`typchkPrimitiveTypeNamed` stopped short. Neither arrays, nor (presumably)
tuples, `Set`, or `Map` involve subtype polymorphism — `[i64]` is exactly one
type, not an open hierarchy of them, the same way there's exactly one
`f64`. That's the property that lets the primitive mechanism resolve
`x.toString()` at compile time with no runtime lookup. On that reasoning,
collection types might fit the same static/mangled mechanism
[Primitive Impls Proposal] already built, rather than needing a different
one. The checker knows each array's item type, so it has what a call needs
to be resolved the way one for `f64` is. `Map`/`Set`'s key and value types
would need generics first. See [Q-static-fit].

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

This proposal does not design tuples, `Set`, or `Map`/`Dict` themselves —
none of them exist yet. `Map` is designed in the [Map Proposal]. It only tries to scope the _mechanism_ question, so
that whichever of them get built later — and arrays, which already
exist — have somewhere to attach methods to.

## Questions

### **Q:** Is "collection type" one category, or two?

<!-- [Q-category]: #q-is-collection-type-one-category-or-two -->

**Status:** Open

Arrays, `Set`, and `Map`/`Dict` are all dynamically sized and mutable.
A tuple, once it exists, is likely fixed-size and its element types likely
fully known at compile time (`(f64, string)` and `(f64, f64)` would be
different concrete types) — closer in shape to a primitive than to an
array. Lumping all four together as "collection types" may be grouping two
different problems under one name.

Options: (a) one category, one mechanism, accepting that it has to handle
both the fixed-shape and dynamically-sized cases; (b) two categories —
tuples follow [Primitive Impls Proposal]'s path since they're closed and
statically known, arrays/`Set`/`Map` follow whatever this proposal lands
on. (b) is more honest about the actual differences but means committing to
two mechanisms instead of one.

Arrays have a third shape in them: `[T]` is a growable list, and `[T; n]` is a
fixed size array with one item type (see the [Lists Proposal]). A fixed size array sits between the two groups: its size is fixed
like a tuple's, but every item has the same type, like a list's.

### **Q:** Do arrays actually fit the static/mangled mechanism?

<!-- [Q-static-fit]: #q-do-arrays-actually-fit-the-staticmangled-mechanism -->

**Status:** Open

Part 2 rests on arrays not having subtype polymorphism. The checker does
know, at a call site, that a value is an array and what its items are, the
way it knows a value is an `f64`: `arr.push(2)` reports
`Can't access '.push' on a [i64].` What's left is whether the mangled-name
scheme on `initial-primitive-traits` can name a method on `[T]` for every
`T`, which is a generics question, since `[i64]` and `[string]` are different
types.

### **Q:** Are `TYPE_SYSTEM_RFC.md` and "Phase 4b" something to reconcile with first?

<!-- [Q-rfc]: #q-are-type_system_rfcmd-and-phase-4b-something-to-reconcile-with-first -->

**Status:** Open

A code comment on `initial-primitive-traits` references a `TYPE_SYSTEM_RFC.md`
and a "Phase 4b" as the reason non-scalar `impl` targets aren't handled yet.
Neither the file nor any other reference to a phased plan exists in
kirbylang or the proposals repository. If that RFC exists
somewhere else (or existed and was removed), this proposal should be
reconciled with it rather than duplicating or contradicting a plan that's
already been thought through — worth checking with whoever wrote that
comment before this proposal goes further.

### **Q:** Does this wait on generics, or ship an untyped version first?

<!-- [Q-generics-order]: #q-does-this-wait-on-generics-or-ship-an-untyped-version-first -->

**Status:** Open

Arrays are typed by their items: `[i64]` is an array of `i64`. The array
natives are typed through generic parameters in their signatures, which only
native signatures can use: `@listPush` is `fun ([T], T) => unit`. A lambda
passed to a generic native gets its parameter types from earlier arguments,
which is what a typed `filter` or `map` needs. So a native such as `@arrMap`
could be typed as `fun ([T], fun (T) => U) => [U]` today. User code can
declare generic functions and methods ([PR #169]), but without trait bounds
([Generic Types Proposal]), so a generic body can't compare, add or call
anything on a `T`.

Options: (a) wait for trait bounds ([Generic Types Proposal], Part 4), so
collection methods can be written in Kirby and typed from the start; (b) ship them as
typed natives first, the way `@listPush` and `@listContains` already work, and
turn them into methods later. (b) unblocks this sooner but means a second
pass later to move them; (a) avoids that second pass but is blocked on a
proposal with its own open questions.

### **Q:** Should this start from arrays alone, or wait until tuples/`Set`/`Map` are at least sketched?

<!-- [Q-scope-order]: #q-should-this-start-from-arrays-alone-or-wait-until-tuplessetmap-are-at-least-sketched -->

**Status:** Open

Designing the mechanism against arrays alone risks shaping it around
arrays' particular quirks (no keys, single element type, index-based
access) in ways that don't generalize to `Map` (two type parameters, no
inherent ordering) or `Set` (no duplicate/indexing semantics at all).

Options: (a) design and ship for arrays now, adjust the mechanism later if
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
  built-in, dynamically-sized aggregate types — arrays (`[T]`) today, `Set` and
  `Map`/`Dict` prospectively. Distinct from a **primitive type**
  ([Primitive Impls Proposal]'s term for `unit`, `bool`, `string` and the number kinds: fixed
  shape, no fields, resolved statically) and from a user-defined **struct**.
  Whether tuples belong in this category or with primitives is
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
[Map Proposal]: ../map/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md

<!-- External -->

[Issue #70]: https://github.com/kirbylang/kirbylang/issues/70

<!-- Other proposals' questions -->

[Q-type-spelling]: ../tuples/PROPOSAL.md#q-is-a-tuple-type-written-a-b-or-tuplea-b
[Q-api-shape]: ../map/PROPOSAL.md#q-are-the-operations-natives-or-methods

<!-- Questions -->

[Q-category]: #q-is-collection-type-one-category-or-two
[Q-static-fit]: #q-do-arrays-actually-fit-the-staticmangled-mechanism
[Q-rfc]: #q-are-type_system_rfcmd-and-phase-4b-something-to-reconcile-with-first
[Q-generics-order]: #q-does-this-wait-on-generics-or-ship-an-untyped-version-first
[Q-scope-order]: #q-should-this-start-from-arrays-alone-or-wait-until-tuplessetmap-are-at-least-sketched
[PR #169]: https://github.com/kirbylang/kirbylang/pull/169
