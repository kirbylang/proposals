---
status: Draft
category: feature
created: 2026-09-18
from_commit: 90ca32cd
---

# Proposal: Methods on Built-in Collection Types

This proposal is less settled than most in this repository, and it stays
that way on purpose. The goal isn't `arr.filter(fn)` specifically — it's
figuring out what category lists and arrays (and, eventually, tuples, `Set`,
and `Map`/`Dict`) belong to, and what methods they should have. Lists and
arrays already have a first set of methods, built with the same mechanism as
primitive types. Designing more before the open questions are answered would be
guessing. This document is
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

Lists `[T]` and fixed size arrays `[T; N]` have methods ([PR #178], [PR #184]).
The standard library writes them in Kirby, and each one calls a native:

```kirby
impl[T] [T] {
    pub fun len(self): i64 = @listLen(self);
    pub fun contains(self, item: T): bool = @listContains(self, item);
}
```

A call such as `xs.push(2)` is resolved at compile time from the static type of
`xs`, into a direct call of a function ([Primitive Impls Proposal]). The
virtual machine's `invoke()` is not involved.

| Type     | Methods                                                                                                        |
| -------- | -------------------------------------------------------------------------------------------------------------- |
| `[T]`    | `len`, `isEmpty`, `push`, `pop`, `insert`, `remove`, `clear`, `contains`, `copy`, `reverse`, `slice`, `concat` |
| `[T; N]` | `len`, `contains`, `copy`, `reverse`, `slice`, `concat`, `toList`                                              |

What is missing:

- Methods that take a function or need something of the items, such as `map`,
  `filter`, `sort` or `join`.
- Traits. The impl parameters can not have bounds, so there is no `Eq`,
  `Display` or `Ord` that uses the items' impls ([Q-list-bounds]). `==` on two
  lists compares identity ([Q-list-equality]).
- Filling a bounded type parameter with a list or array. This waits for
  monomorphization.
- Methods for `Set`, `Map` and tuples, none of which exist. `Map` is designed in
  the [Map Proposal] and tuples in the [Tuples Proposal]. `Set` has no design.

User code can not write these impls.

### Relationship to other proposals

- [Primitive Impls Proposal] designs the mechanism that lists and arrays use.
- [Generic Types Proposal] matters for `Map[K, V]` and `Set[T]`, which need
  generic type arguments. They work for structs a program declares
  (`Box[f64]`, [PR #169]), and no built-in type other than `[T]` and `[T; N]`
  takes them yet. See [Q-generics-order].
- [Map Proposal] designs `Map[K, V]`, with its operations written as natives (`@mapGet`, `@mapSet`) and a question about whether they become methods ([Q-api-shape]). It is the sketch of `Map`'s shape that option (b) of [Q-scope-order] asks for.
- [For-Each Proposal] adds `for (item in collection)`. Methods like `filter` and `map` on arrays would be built on the same iteration rule.
- [Tuples Proposal] designs the tuple type this proposal mentions. It writes
  the type `(f64, string)` ([Q-type-spelling]), and gives a tuple a fixed
  size and item types known at compile time, which bears on [Q-category].

## The Changes

Everything below is a candidate direction, not a decision — see
[Questions] for what's actually still open about each.

### Part 1 — Name the category

Introduce **collection type** as a term for this family — lists and arrays
today; tuples, `Set`, and `Map`/`Dict` prospectively — distinct from
[Primitive Impls Proposal]'s "primitive type." Whether "collection type" is
even one coherent category, or actually two with different needs, is
[Q-category].

### Part 2 — The static mechanism (built for lists and arrays)

A list or array impl declares type parameters, and the checker matches them
against the receiver, so `[i64]` and `[string]` share one method body. Tuples,
`Set` and `Map` have no subtyping either, so the same mechanism may fit them.
`Map` and `Set` would need generic built-in types first.

### Part 3 — Generic methods

A method written in Kirby that needs something of the items, such as `join`
calling `toString`, is generic code calling a method on a value that may be a
number or a string. That works after monomorphization ([Generic Types Proposal]
Part 5). See [Q-generics-order].

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

**Status:** Answered

The question was whether one named function can serve a method on `[T]` for
every `T`.

#### Answer

Yes. [PR #178] built methods on `[T]` and `[T; N]` this way. Each method is
compiled once, and the checker substitutes the item type, and the array length,
at each call.

### **Q:** Are `TYPE_SYSTEM_RFC.md` and "Phase 4b" something to reconcile with first?

<!-- [Q-rfc]: #q-are-type_system_rfcmd-and-phase-4b-something-to-reconcile-with-first -->

**Status:** Answered

A code comment on an unmerged branch, `initial-primitive-traits`, referred to a
`TYPE_SYSTEM_RFC.md` that is in neither repository.

#### Answer

No. That branch was not the one that was built. [PR #178] built primitive,
list and array impls without it.

### **Q:** Does this wait on generics, or ship an untyped version first?

<!-- [Q-generics-order]: #q-does-this-wait-on-generics-or-ship-an-untyped-version-first -->

**Status:** Answered

Arrays are typed by their items: `[i64]` is an array of `i64`. The array
natives are typed through generic parameters in their signatures, which only
native signatures can use: `@listPush` is `fun ([T], T) => unit`. A lambda
passed to a generic native gets its parameter types from earlier arguments,
which is what a typed `filter` or `map` needs. So a native such as `@arrMap`
could be typed as `fun ([T], fun (T) => U) => [U]` today. User code can
declare generic functions and methods ([PR #169]), and a type parameter can
have trait bounds ([PR #176]), so a generic body can compare, add or call
the methods of its bound. A number or string cannot yet fill a bounded
parameter, even though the standard library implements traits for them
([PR #178]).

Options: (a) wait for trait bounds ([Generic Types Proposal], Part 4), so
collection methods can be written in Kirby and typed from the start; (b) ship them as
typed natives first, the way `@listPush` and `@listContains` already work, and
turn them into methods later. (b) unblocks this sooner but means a second
pass later to move them; (a) avoids that second pass but is blocked on a
proposal with its own open questions.

#### Answer

(a): wait. Trait bounds are built, but a method such as `join` calls `toString`
on each item, and for a number or string that is a call on a primitive inside
generic code. Primitives have their impls ([Primitive Impls Proposal]).
Generic code can call them once each copy of it knows its type, which is
monomorphization ([Generic Types Proposal] Part 5). The order is in the
[roadmap](../../ROADMAP.md): monomorphization is built before this proposal.

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
- **Bound**: a trait that a type parameter has to implement, `[T: Display]`.
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
[Q-list-bounds]: ../primitive-impls/PROPOSAL.md#q-can-a-list-or-array-impl-require-something-of-its-item-type
[Q-list-equality]: ../primitive-impls/PROPOSAL.md#q-should--and-contains-on-lists-use-the-item-types-eq
[PR #178]: https://github.com/kirbylang/kirbylang/pull/178
[PR #184]: https://github.com/kirbylang/kirbylang/pull/184
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
[PR #176]: https://github.com/kirbylang/kirbylang/pull/176
