---
status: Draft
category: feature
created: 2026-11-24
from_commit: 89a0443
---

# Proposal: Generic Traits

A trait that has type parameters of its own, such as `Iterator[T]`, and trait
impls on generic types, such as `impl Display for Box[T]`. Nothing else
designs these. The [For-Each Proposal] sketches `trait Iterator[T]` and says it
depends on them, and the [Enums Proposal] and [Collection Methods Proposal] need
methods on generic types.

This is a skeleton. It records what the code does today and the questions that
must be answered before it is built. It does not yet choose a design.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing process to understand the changes being proposed and what impacts they will have. That research is largely tracked in the form of [Questions].

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it is defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Type parameters work on functions, structs and aliases ([Generic Types Proposal]
Part 3), and a parameter can be bound to traits (Part 4, [PR #176]). Two things
are still missing, and both show up as soon as the standard library is written
in Kirby.

**A trait cannot have type parameters.** This is what `Iterator[T]` needs:

```kirby
trait Source[T] {
  fun next(self): T;
}
```

```
[line 1] Error at '[': Expect '{' before trait body.
```

A trait's `Type` has no list of parameters. It holds static methods, instance
methods, an optional supertrait and a built-in flag (`src/types.h`).

**A trait cannot be implemented for a generic struct.** A plain impl works:

```kirby
struct Box[T] { pub var value: T; }
impl Box[T] { pub fun get(self): T = self.value; }
```

A trait impl does not. Written without parameters it is rejected, and written
with them it does not parse:

```kirby
impl Display for Box { pub fun toString(self): string = "box"; }
// [line 2] Error at 'Box': Trait implementations on generic structs aren't supported yet.

impl Display for Box[T] { pub fun toString(self): string = "box"; }
// [line 2] Error at '[': Expect '{' before impl body.
```

The first message comes from `typecheck.c`. The second comes from the parser,
which reads the target of a trait impl as one name.

Until both exist, a method like `join` or `map` on a list of any item type has
to be a native written in C, and `for (x in v)` over a user type cannot be
written.

## Proposed Changes

No design is chosen yet. This section lists what the design has to cover, so the
questions below can be answered one at a time.

### Goals and Non Goals

What this proposal covers:

- Declaring a trait with type parameters: `trait Iterator[T] { ... }`
- Implementing a trait for a type, with the trait's arguments:
  `impl Iterator[f64] for Counter`
- Implementing a trait for a generic type: `impl Display for Box[T]`
- Naming such a trait in a bound: `I: Iterator[f64]`
- Method signatures inside the trait that use the trait's parameters

The following is intentionally left out of scope for this proposal:

- Making each generic function a separate copy (see [Generic Types Proposal] Part 5)
- A new kind of trait, such as one with associated types, unless a question below
  needs it
- Trait types (a trait name where a type is expected). See the [Trait Types Proposal]

### Implementation Plan

The parts below are in the order they can be built. Each is its own patch.

#### Part 1: Parse the parameters

- `trait Name[T, U] { ... }` and a target with arguments in a trait impl, as
  `impl Trait for Box[T]` and `impl Trait[A] for Type`.
- A bound with arguments: `T: Iterator[f64]` ([Q-bound-syntax]).

#### Part 2: A trait keeps its parameters

- The trait `Type` gets a list of parameter names. A method signature inside the
  trait can use them, the same way a generic struct's fields can.
- A use of a trait with the wrong number of arguments is an error, worded like
  the existing `'Box' takes 1 type parameter(s), but this impl declares 0.`

#### Part 3: Trait impls on generic types

- Remove `Trait implementations on generic structs aren't supported yet.` The
  struct's parameters are in scope inside the impl, as they are for a plain
  impl today.
- `typeStructImplementsTrait` has to answer for `Box[f64]` as well as `Box`
  ([Q-impl-match]).

#### Part 4: Bounds and calls

- A bound with arguments is checked at the call site, like the bounds of
  [PR #176].
- A method called through such a bound has the trait's parameters replaced by
  the arguments, using the substitution helper from the [Generic Types Proposal]
  3.6.

## Impacts

### Existing Syntax Or Behavior

- `trait Name[...]` and `impl Trait for Type[...]` are errors today, so no valid
  program changes meaning.
- The message `Trait implementations on generic structs aren't supported yet.`
  goes away. `docs/TYPES.md` lists it as a limitation.

### Related Proposals

- [Generic Types Proposal] — type parameters and bounds are built. Monomorphization
  ([Part 5]) must copy generic trait impls, and the orphan rule (6.4) must say who
  may write `impl Iterator[f64] for Counter`.
- [For-Each Proposal] — Part C defines `Iterator[T]` and `Iterable[T]` with this.
- [Enums Proposal] — Part C: methods on `Option[T]` and `Result[T, E]` need trait
  impls on generic types.
- [Collection Methods Proposal] — methods built from `Iterator[T]`.
- [Primitive Impls Proposal] — `impl Iterator[T] for [T]` puts a generic trait on
  an array, which that proposal leaves out.
- [Trait Types Proposal] — a function taking any `Iterable[T]`.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

##### NEW: tests/run/generic_traits/trait_with_parameter.krb

```kirby
trait Source[T] {
  fun next(self): T;
}

struct Counter { pub var n: i64; }

impl Source[i64] for Counter {
  pub fun next(self): i64 = self.n;
}

fun main(): unit {
  let c = Counter { n: 7 };
  @println(c.next());
}
```

###### Expected Outcome

Prints `7`.

##### NEW: tests/run/generic_traits/trait_impl_on_generic_struct.krb

```kirby
struct Box[T: Display] { pub var value: T; }

impl Display for Box[T] {
  pub fun toString(self): string = "Box(" + self.value.toString() + ")";
}

fun main(): unit {
  @println(Box { value: 1 });
}
```

###### Expected Outcome

Prints `Box(1)`. The impl takes `T: Display` from the struct, as a plain impl does today ([Q-impl-bounds]).

##### NEW: tests/run/errors/generic_trait_wrong_argument_count.krb

```kirby
trait Source[T] {
  fun next(self): T;
}

struct Counter { pub var n: i64; }

impl Source for Counter {
  pub fun next(self): i64 = self.n;
}
```

###### Expected Outcome

A compile error that says `Source` takes 1 type parameter and none were given.
The exact text is chosen when the test is written.

## Questions

### **Q:** How is a trait's argument written in an impl and in a bound?

**Status:** Open

`impl Source[i64] for Counter` and `I: Source[i64]` are the obvious forms, as in
Rust. The existing `impl Box[T]` already puts the parameters after the name.
Whether the impl has to repeat the struct's parameters (`impl Display for
Box[T]`) or may leave them out is part of the same choice.

### **Q:** Does `Box[f64]` implement a trait when `Box[T]` does?

**Status:** Open

`typeStructImplementsTrait` takes a type and a trait name. A generic struct's
impl is written once for all arguments, but it may be written only for some
(`impl Display for Box[string]`). Options: (a) one impl per generic struct,
for every argument, with bounds on the parameters; (b) impls for specific
arguments too. (b) needs a rule for two impls that both match.

### **Q:** Can one type implement the same trait with different arguments?

**Status:** Open

`impl From[f64] for X` together with `impl From[string] for X`. Allowing it
means a method call needs the argument types to choose. Not allowing it means a
method name always finds one impl. `Iterator[T]` needs only one.

### **Q:** Does a trait impl on a generic struct keep taking its bounds from the struct?

**Status:** Open

`impl Display for Box[T]` needs `T: Display` to call `self.value.toString()`.
A plain impl already takes its bounds from the struct, and writing them on the
impl is an error (`tests/run/errors/generic_bound_on_impl.krb`):

```
[line 5] Error at ':': An impl's type parameters take their bounds from the struct.
```

Options: (a) the same rule for trait impls, so the struct has to declare the
bound; (b) a trait impl may add bounds the struct does not have, so
`Box[string]` and `Box[Point]` can implement `Display` while `Box[T]` with no
bound does not. (b) is what Rust does, and it needs the check from [Q-impl-match].

### **Q:** Do trait types accept a generic trait?

**Status:** Open

The [Trait Types Proposal] asks whether `Iterable[f64]` can stand where a type is
expected. A trait that mentions `Self` outside its receiver cannot be a trait
type, and a trait with parameters may need a second rule.

### **Q:** Does a generic trait need something other than parameters?

**Status:** Open

Rust's `Iterator` has an associated type, `Item`, rather than a parameter. With
a parameter, `I: Iterator[T]` names the item type in every signature that uses it.
Parameters are enough for the For-Each sketch. This asks whether to stop there.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Generic trait**: A trait with type parameters of its own, like `Iterator[T]`
- **Bound**: The traits a type parameter must implement, as in `T: Display`
- **Orphan rule**: The rule that decides which module may write an impl, so two modules cannot write the same one

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Part 5]: ../generic-types/PROPOSAL.md#part-5--monomorphization
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md

<!-- Questions -->

[Q-bound-syntax]: #q-how-is-a-traits-argument-written-in-an-impl-and-in-a-bound
[Q-impl-match]: #q-does-boxf64-implement-a-trait-when-boxt-does
[Q-impl-bounds]: #q-does-a-trait-impl-on-a-generic-struct-keep-taking-its-bounds-from-the-struct

<!-- Pull requests -->

[PR #176]: https://github.com/kirbylang/kirbylang/pull/176
