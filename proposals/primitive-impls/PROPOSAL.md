---
status: Closed
category: feature
created: 2026-09-18
from_commit: 9726657a
---

# Proposal: `impl` and Trait Implementations for Primitive Types

Kirby has methods and trait impls on primitive types: the number kinds (`i64`,
`f64`, `u8` and the rest), `string`, `bool` and `unit`, and also lists (`[T]`)
and fixed size arrays (`[T; N]`). The standard library writes them in Kirby. A
program can implement its own traits for all of these except lists and arrays.

It was implemented and merged in [PR #178], [PR #192] and the other changes
named below. The sections below describe that implementation. Work that is left
belongs to other proposals ([What Is Left](#what-is-left)).

---

## How to read this document

### Living Document

This proposal is Closed. It records what was built and why.

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it is defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true for the current implementation. Examples were run on the `from_commit`.
- When the proposal text says Kirby "will" do something, another proposal plans it.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Before [PR #178], only a struct could have methods or implement a trait. There
was no way to call `x.toString()` or `x.equals(y)` on a number or a string, and
the standard library could not offer `"abc".len()`.

That is solved for calls on a value whose type is known where the call is
written. Generic code calling a primitive's method needs monomorphization
([Generic Types Proposal] Part 5).

## What Is Built

First built in [PR #178]. Later changes are named where they apply.

### 1. Methods are written in the standard library

`stdlib/stdlib.krb` holds ordinary `impl` blocks:

```kirby
impl string {
    // Length in bytes
    pub fun len(self): i64 = @strLen(self);
}

impl Display for bool {
    fun toString(self): string = @boolToString(self);
}
```

| Type                                | Methods of the type                                                                                                         | Traits                                                                              |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `i8`, `i16`, `i32`, `i64`           | `abs`, `max`, `min`                                                                                                         | `Display`, `Eq`, `Ord`, `Default`, `Add`, `Sub`, `Mul`, `Div`, `Mod`, `Neg`         |
| `u8`, `u16`, `u32`, `u64`           | `max`, `min`                                                                                                                | the same without `Neg`                                                              |
| `f32`, `f64`                        | `abs`, `ceil`, `floor`, `max`, `min`, `pow`, `round`, `sqrt`, `trunc`                                                       | the same as the signed integers                                                     |
| `string`                            | `len`                                                                                                                       | `Display`, `Eq`, `Default`, `Add`                                                   |
| `bool`                              | none                                                                                                                        | `Display`, `Eq`, `Default`                                                          |
| `unit`                              | none                                                                                                                        | `Display`, `Eq`, `Ord`, `Default`                                                   |
| `[T]`                               | `len`, `isEmpty`, `push`, `pop`, `insert`, `remove`, `clear`, `contains`, `copy`, `reverse`, `slice`, `concat`              | none                                                                                |
| `[T; N]`                            | `len`, `contains`, `copy`, `reverse`, `slice`, `concat`, `toList`                                                           | none                                                                                |

`string` has no `Ord`, because there is no `<` on strings ([Q-compare]).

### 2. Calls

```kirby
fun main(): unit {
    @println("abc".len());          // 3
    @println(2.5.floor());          // 2
    @println(7.toString());         // 7
    @println(i64.default());        // 0
    @println(Display.toString(7));  // 7
}
```

- A static trait method is called on the type's name, or on an alias of it
  (`type Meters = i64;` then `Meters.default()`, [PR #186]). A variable with the
  same name as the type wins.
- A method can be used as a value ([PR #193]). `let length = text.len;` keeps
  the value `text` had at that moment, and `f64.default` is the function itself.
- A method of one number kind can not be called on another.
- A number literal that a method is called on takes the number kind of the
  argument, as in `==`: with `let x: u8 = 5;`, `5.equals(x)` is a `u8` call.
- A type's own method wins over a trait method of the same name.

### 3. Calling a trait's method by name

`Trait.method(value)` calls the trait's method with the value as its first
argument, even when the type has its own method of that name. It works for
structs, for primitives that implement the trait, and for a value whose type
parameter is bounded by the trait or by a trait that has it as a super trait.

```kirby
struct Point { pub var x: i64; }

impl Point {
    pub fun toString(self): string = "own";
}

impl Display for Point {
    fun toString(self): string = "trait";
}

fun show[T: Display](value: T): string = value.toString();

fun main(): unit {
    let p = Point { x: 1 };
    @println(p.toString());          // own
    @println(Display.toString(p));   // trait
    @println(show(p));               // trait
    @println(p);                     // trait
}
```

The same change fixed a type's own method running in place of the trait's
through a bound, an operator or printing. [PR #183] made the type's own method
keep the plain name when the trait impl is loaded later.

Two traits can give one type instance methods of the same name ([PR #191]). A
plain `value.name()` is then ambiguous and is an error, unless the type has its
own method of that name. `A.name(value)` picks one. Two static methods of one
name are still an error.

A type's own static method hides a trait's static method of the same name, and
the trait's can not be called yet
([a hidden static method][Q-hidden-static]).

### 4. Lists and arrays

An impl for a list or an array declares its parameters after `impl`
([PR #184]):

```kirby
impl[T] [T] {
    pub fun len(self): i64 = @listLen(self);
}

impl[T, const N] [T; N] {
    pub fun concat[const M](self, other: [T; M]): [T] = @arrConcat(self, other);
}
```

`const N` is an array length. An impl that does not need it writes `_`. Only the
standard library can declare `const` parameters, and these parameters can not
have bounds. A trait impl such as `impl[T] Display for [T]` is accepted in the
standard library and applies to every item type. `Eq.equals(a, b)` and
`a.equals(b)` would call such an `Eq` impl, but `==` on two lists would not.
The standard library has no such impl today.

### 5. What a program can write

A program can implement a trait it declares for `bool`, `string`, `unit` and
the number kinds ([PR #192]):

```kirby
trait Squarable { fun squared(self): f64; }

impl Squarable for f64 {
    fun squared(self): f64 = self * self;
}

fun main(): unit {
    let three: f64 = 3;
    @println(three.squared());       // 9
    @println(Squarable.squared(three)); // 9
}
```

It can not do the following:

| Code | Error |
| --- | --- |
| `impl Display for f64 { ... }` | `Only the standard library can implement 'Display' for 'f64'.` |
| `impl f64 { ... }` | `Only the standard library can add methods to 'f64'.` |
| `impl[X] T for [X] { ... }` | `You can't write an impl for a list or array in user code.` |

A struct, trait or type alias can not be named like a primitive: `trait i64 {}`
is `'i64' is a built-in type and can't be the name of a trait.` ([PR #195]). A
type parameter can take such a name. An impl can not name a primitive through a
type alias: `type S = string; impl S { }` is `Unknown type 'S'.`

### 6. How it works

- **Type checker.** It keeps a list of primitive methods and a list of (type,
  trait) pairs in its environment (`PrimitiveMethod` and `PrimitiveTraitImpl`
  in `src/typecheck.c`). When a call matches, it records the owner on the call's
  syntax tree node, because the compiler has no type information.
- **Compiler.** Each method becomes a global function named `owner.method`, or
  `owner.Trait.method` for a trait impl. A `.` can not appear in a program's own
  names, so these can not collide with them. `"ab".len()` is a plain call of
  `string.len` with the value as the first argument.
- **Virtual machine.** Primitive methods need no method table. A struct stores
  a trait's method under `Trait.method` (`OP_TRAIT_METHOD`), and also under
  `method` when no other method has that name. A primitive's method used as a
  value is one object holding the value and the function (`OP_BIND_METHOD`).

### 7. `NaN`

Every direct comparison with `NaN` is `false`. `cmp` sorts `NaN` below every
number and level with `NaN` ([PR #190]).

## What Is Left

| Item | Today | Now in |
| --- | --- | --- |
| A primitive, list or array filling a bounded type parameter | `string implements Display, but a primitive can't be used with a bound yet, required by T.` | [Generic Types Proposal] Part 5 |
| Ordering operators with `NaN` in generic code | numbers can not be used there yet. Generic code compares through `cmp`, where `NaN` is below every number | [Generic Types Proposal] Part 5 |
| Calling a hidden static trait method | no way to call it | [Generic Types Proposal] [10.10][Q-hidden-static] |
| Bounds on list and array impls | not allowed | [Collection Methods Proposal] ([Q-list-bounds]) |
| `==` and `contains` using the items' `Eq` | `[1] == [1]` is `false` | [Collection Methods Proposal] ([Q-list-equality]) |
| `unit`'s `Display` | `().toString()` is `"nil"` | [Enums Proposal] Part C ([Q-unit-methods]) |
| Reaching a built-in type by a path | no paths | [Modules Proposal] ([Q-builtin-names]) |

Two smaller limits have no proposal:

- An `impl` without a trait has no static methods:
  `Static methods on 'string' aren't supported yet.`
- `@println` and string interpolation use built-in formatting for numbers,
  strings and `bool` ([Q-print-display]).

## Impacts

### Related Proposals

- [Generic Types Proposal] — monomorphization lets primitives fill bounds and
  removes the built-in number methods for the operator traits.
- [Collection Methods Proposal] — builds on the list and array impls.
- [Generic Traits Proposal] — `impl Iterator[T] for [T]` puts a generic trait on
  a list.
- [Trait Types Proposal] — whether a primitive converts to a trait type
  ([Q-primitives]).
- [Tuples Proposal] — whether `unit` is the empty tuple ([Q-unit-empty]).
- [Strings Proposal] — `Ord` for `string`.
- [String Interpolation Proposal] — which placeholder types convert.
- [Enums Proposal] — an enum value that is a plain number would be called the way
  a primitive is.
- [Modules Proposal] — which module may write an impl.

### Testing Plan

The behavior is covered by `tests/run/primitive_methods/`,
`tests/run/structs/trait_qualified_*`, `tests/run/types/` and
`tests/parse/impls/`.

## Questions

### **Q:** Should bare (non-trait) `impl` on primitives be allowed too, or should primitives stay trait-impl-only?

[Q-inherent]: #q-should-bare-non-trait-impl-on-primitives-be-allowed-too-or-should-primitives-stay-trait-impl-only

**Status:** Answered

Options: (a) only trait impls on primitives, so every method added to a type is named by a trait. (b) both forms.

#### Answer

(b), in the standard library only. `impl string { ... }` is how `len` is written. A program can write only the trait form, for its own traits ([Q-user-impls]).

### **Q:** Where should a primitive kind's methods and implemented-trait list actually live?

[Q-storage]: #q-where-should-a-primitive-kinds-methods-and-implemented-trait-list-actually-live

**Status:** Answered

Options: (a) a new part of `Type` for primitives. (b) a table outside `Type`. (c) one shape shared by structs and primitives.

#### Answer

(b). The type checker's environment keeps the lists (section 6).

### **Q:** How should the run-time attach and dispatch methods for kinds with no heap object?

[Q-dispatch]: #q-how-should-the-run-time-attach-and-dispatch-methods-for-kinds-with-no-heap-object

**Status:** Answered

Numbers and `bool` are values with no object, and a string object has no method table.

#### Answer

No table is needed. A call on a value whose type is known where the call is written is a direct call of a global function (section 6). Generic code calls a primitive's method only after monomorphization ([Generic Types Proposal] Part 5), which makes those calls direct too. What stays open is a trait type that holds a primitive ([Q-primitives]), the one case where the type is not known at compile time.

### **Q:** What is the receiver syntax for calling a static trait method on a primitive kind?

[Q-static-receiver]: #q-what-is-the-receiver-syntax-for-calling-a-static-trait-method-on-a-primitive-kind

**Status:** Answered

Primitive type names are not values. Options: (a) allow the name before a method, `f64.default()`. (b) a new syntax. (c) no static methods.

#### Answer

(a). The name has this meaning only in front of a method. `@println(f64);` still fails at run time with `Undefined variable 'f64'.`

### **Q:** Should this land before, after, or independent of the generic-types proposal's operator-dispatch decision?

[Q-sequencing]: #q-should-this-land-before-after-or-independent-of-the-generic-types-proposals-operator-dispatch-decision

**Status:** Answered

The [Generic Types Proposal] considered making every operator a trait method, which needs primitive impls.

#### Answer

Independent. Operators on numbers stay built-in instructions ([Generic Types Proposal] 10.3), and primitive impls were built without changing them.

### **Q:** Does `unit` get impls too?

[Q-unit]: #q-does-unit-get-impls-too

**Status:** Answered

`unit` has one value. Options: (a) exclude it. (b) include it, to avoid a special case.

#### Answer

(b). It implements `Display`, `Eq`, `Ord` and `Default`. `Display` is to be removed along with `nil` ([Q-unit-methods]).

### **Q:** Should printing use a primitive's `Display` impl?

[Q-print-display]: #q-should-printing-use-a-primitives-display-impl

**Status:** Answered

`@println` and string interpolation call a struct's `Display` impl. For numbers, strings and `bool` they use built-in formatting, so a primitive's `Display` impl runs only when called by name.

Options: (a) use the impl. One consistent rule. (b) keep built-in formatting, which is what is built. Only the standard library writes these impls, so the two print the same text today.

#### Answer

(b). Built-in formatting stays. Programs can not implement `Display` for these
types ([Q-user-impls]), so the output can not differ.

### **Q:** Should the design anticipate more numeric primitive kinds now, or wait for that proposal?

[Q-numeric-family]: #q-should-the-design-anticipate-more-numeric-primitive-kinds-now-or-wait-for-that-proposal

**Status:** Answered

Options: (a) design for any primitive kind. (b) only `f64`, `bool` and `string`.

#### Answer

(a). [Sized Number Types][sized-number-types] is Closed, and all ten number kinds have impls.

### **Q:** May a program implement its own traits for primitive types?

[Q-user-impls]: #q-may-a-program-implement-its-own-traits-for-primitive-types

**Status:** Answered

The first request for this feature was that Kirby code can `impl` and implement traits for primitives. The first version allows only the standard library.

Options: (a) a program may implement a trait it declares, but not a built-in trait, and may not add methods without a trait. This is Rust's [orphan rules] and its rule against [methods on primitives]. (b) keep it to the standard library.

#### Answer

(a), built in [PR #192]. It covers `bool`, `string`, `unit` and the number
kinds. Impls for lists and arrays stay in the standard library, because their
parameters can not have bounds yet ([Q-list-bounds]).

### **Q:** How are floats ordered when a value is `NaN`?

[Q-float-order]: #q-how-are-floats-ordered-when-a-value-is-nan

**Status:** Answered

Every direct comparison with `NaN` is `false`, because `<=` and `>=` have their own instructions (`OP_LESS_EQUAL` and `OP_GREATER_EQUAL`). `Ord` did not match them: `nan.cmp(1.0)` was `0`, which means equal, so `a <= b` through `cmp`, as in a function with `T: Ord`, would be `true`. In Rust all four comparisons with `NaN` are `false`, and `f64` implements [PartialOrd] and not `Ord`.

Options for `Ord`: (a) add a `PartialOrd` trait, which needs a way to answer "no order". (b) remove `Ord` from floats. (c) give `cmp` an order that includes `NaN`, which then differs from the operators.

#### Answer

(c), built in [PR #190]. Floats keep `Ord`, and `cmp` sorts `NaN` below every
number, as Go's [cmp package] does. The ordering operators in generic code are
to behave like the direct ones after monomorphization ([Generic Types Proposal]
Part 5).

### **Q:** Can a primitive's method be used as a value?

[Q-method-values]: #q-can-a-primitives-method-be-used-as-a-value

**Status:** Answered

A struct's method could be: `let f = p.get;`. A primitive's could not. Rust allows neither form but allows `str::len`.

Options: (a) allow it for primitives. (b) remove it for structs, which breaks programs. (c) keep the difference and document it.

#### Answer

(a), built in [PR #193]. This differs from Rust, and matches what structs do.

### **Q:** Can two traits give one type the same method name?

[Q-method-names]: #q-can-two-traits-give-one-type-the-same-method-name

**Status:** Answered

It was the error `'name' is already provided to 'P' by another trait.` Rust allows it: a plain `p.name()` is then ambiguous, and `A::name(&p)` picks one.

Options: (a) allow it, make the plain call an error, and use `A.name(p)`. (b) keep the error.

#### Answer

(a), built in [PR #191], as in Rust. It is for methods that take `self`. Two
static methods of one name are still an error ([Q-hidden-static]).

## Glossary

- **Changes**: Changes refer to the changes in this document
- **Primitive type**: a built-in value type that is not a struct: `unit`,
  `bool`, `string` and the ten number kinds. Lists and arrays are not primitive
  types, but get methods the same way.
- **Trait impl**: an `impl SomeTrait for SomeType { ... }` block.
- **Bounded type parameter**: a type parameter that has to implement a trait,
  `[T: Display]`.
- **Monomorphization**: compiling a separate copy of a generic function for each
  type it is used with.
- **Coherence**: the rule that a trait is implemented for a type at most once.

---

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions

<!-- Proposals -->

[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Generic Traits Proposal]: ../generic-traits/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Strings Proposal]: ../strings/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[sized-number-types]: ../sized-number-types/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-compare]: ../strings/PROPOSAL.md#q-is-string-ordering-an-operator-rule-or-an-ord-impl
[Q-primitives]: ../trait-types/PROPOSAL.md#q-can-a-number-string-or-bool-convert-to-a-trait-type
[Q-unit-empty]: ../tuples/PROPOSAL.md#q-is-unit-the-empty-tuple
[Q-unit-methods]: ../enums/PROPOSAL.md#q-what-should-units-trait-methods-return
[Q-list-bounds]: ../collection-methods/PROPOSAL.md#q-can-a-list-or-array-impl-require-something-of-its-item-type
[Q-list-equality]: ../collection-methods/PROPOSAL.md#q-should--and-contains-on-lists-use-the-item-types-eq
[Q-hidden-static]: ../generic-types/PROPOSAL.md#1010-calling-a-static-trait-method-hidden-by-the-types-own
[Q-builtin-names]: ../modules/PROPOSAL.md#q-can-a-declaration-take-a-built-in-types-name

<!-- External -->

[PR #178]: https://github.com/kirbylang/kirbylang/pull/178
[PR #183]: https://github.com/kirbylang/kirbylang/pull/183
[PR #184]: https://github.com/kirbylang/kirbylang/pull/184
[PR #186]: https://github.com/kirbylang/kirbylang/pull/186
[PR #190]: https://github.com/kirbylang/kirbylang/pull/190
[PR #191]: https://github.com/kirbylang/kirbylang/pull/191
[PR #192]: https://github.com/kirbylang/kirbylang/pull/192
[PR #193]: https://github.com/kirbylang/kirbylang/pull/193
[PR #195]: https://github.com/kirbylang/kirbylang/pull/195
[orphan rules]: https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
[methods on primitives]: https://doc.rust-lang.org/error_codes/E0390.html
[PartialOrd]: https://doc.rust-lang.org/core/cmp/trait.PartialOrd.html
[cmp package]: https://pkg.go.dev/cmp#Compare
