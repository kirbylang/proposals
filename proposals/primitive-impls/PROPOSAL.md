---
status: Draft
category: feature
created: 2026-09-18
from_commit: 90ca32cd
---

# Proposal: `impl` and Trait Implementations for Primitive Types

Kirby has methods and trait impls on primitive types: the number kinds (`i64`,
`f64`, `u8` and the rest), `string`, `bool` and `unit`, and also lists (`[T]`)
and fixed size arrays (`[T; N]`). The standard library writes them in Kirby.
Programs call them but can not write their own.

This document records what is built, and what is still undecided: mainly whether
a program may implement its own traits for these types.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing process to understand the changes being proposed and what impacts they will have. That research is largely tracked in the form of [Questions].

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it is defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true for the current implementation. Examples were run on the `from_commit`.
- When the proposal text says Kirby "should" or "will" do something, that is true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Before [PR #178], only a struct could have methods or implement a trait. There
was no way to call `x.toString()` or `x.equals(y)` on a number or a string, and
the standard library could not offer `"abc".len()`.

That is solved for calls on a value whose type is known where the call is
written. What remains:

- A program can not implement its own trait for a primitive ([Q-user-impls]).
  In Rust it can, because the trait is the program's own.
- A primitive can not fill a bounded type parameter. This waits for
  monomorphization ([Generic Types Proposal] Part 5).
- Some behavior of the built-in impls is unsettled: what `unit`'s methods
  return, how lists compare, and how floats order ([Questions]).

## What Is Built

Built in [PR #178]. Later changes are named where they apply.

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
- A method has to be called: `let f = "a".len;` is the error
  `Method 'len' of 'string' has to be called.` ([Q-method-values]).
- A method of one number kind can not be called on another.
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

Two limits come with it. Two traits can not give one type the same method name
([Q-method-names]). A static trait method that has the same name as one of the
type's own can not be called ([Q-shadowed-static]).

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

### 5. Only the standard library writes these impls

```kirby
impl f64 {
    pub fun double(self): f64 = self * 2;
}
```

```
[line 1] Error at 'f64': You can't write an impl for a primitive type in user code.
```

The parser rejects it by the target's spelling, for both impl forms. It also
rejects an impl for a program's own struct that is named like a primitive, such
as `struct string`. An impl can not name a primitive through a type alias either:
`type S = string; impl S { }` is `Unknown type 'S'.` The type checker has its own
messages (`Only the standard library can add methods to 'f64'.`), which the
parser's check comes before.

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
  a trait's method under both `method` and `Trait.method` (`OP_TRAIT_METHOD`).

### 7. Limits that remain

- A primitive, list or array can not fill a bounded type parameter:

  ```
  string implements Display, but a primitive can't be used with a bound yet, required by T.
  ```

  A list or array gets `[i64; 1] doesn't implement Display, required by T.`
- An `impl` without a trait has no static methods:
  `Static methods on 'string' aren't supported yet.`
- `@println` and string interpolation use built-in formatting for numbers,
  strings and `bool` ([Q-print-display]). Interpolating `unit` is an error,
  even though `unit` implements `Display`.
- Numbers keep two built-in pieces for operators inside bounded generics:
  `typchkNumberImplementsTrait` and `invokeNumber`. Monomorphization removes
  them.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- Methods and trait impls on `bool`, `string`, `unit`, the number kinds, lists
  and arrays (built)
- Whether a program may implement its own traits for them ([Q-user-impls])
- The behavior of the standard library's impls ([Questions])

The following is intentionally left out of scope for this proposal:

- Filling a bounded type parameter with a primitive. See the
  [Generic Types Proposal] Part 5.
- A trait type that holds a primitive. See [Q-primitives].
- What more methods lists and other collections get. See the
  [Collection Methods Proposal].
- Which module may write an impl. See [Modules Proposal] and the orphan rule in
  the [Generic Types Proposal].

### Implementation Plan

#### Part 1: Programs implement their own traits

Only if [Q-user-impls] is answered yes.

```kirby
trait Squarable { fun squared(self): f64; }

impl Squarable for f64 {
    fun squared(self): f64 = self * self;
}
```

- The parser stops rejecting primitive targets, so one check, in the type
  checker, decides.
- The checker rejects a built-in trait (`impl Display for f64`) and a plain
  `impl f64` in a program, and allows a program's own trait.
- The existing coherence rule (a trait once per type) applies.

## Impacts

### Existing Syntax Or Behavior

- Part 1 changes the message for a rejected primitive impl in programs, from the
  parser's to the type checker's.
- A program's own struct named like a primitive could then have impls.

### Related Proposals

- [Generic Types Proposal] — monomorphization lets primitives fill bounds and
  removes the built-in number methods.
- [Collection Methods Proposal] — builds on the list and array impls.
- [Generic Traits Proposal] — `impl Iterator[T] for [T]` puts a generic trait on
  a list.
- [Trait Types Proposal] — whether a primitive converts to a trait type.
- [Tuples Proposal] — whether `unit` is the empty tuple should agree with
  [Q-unit-methods].
- [Strings Proposal] — `Ord` for `string`.
- [String Interpolation Proposal] — which placeholder types convert.
- [Enums Proposal] — an enum value that is a plain number would be called the way
  a primitive is.

### Testing Plan

The built behavior is covered by `tests/run/primitive_methods/`,
`tests/run/structs/trait_qualified_*` and `tests/parse/impls/`.

#### E2E Tests

##### NEW: tests/run/primitive_methods/user_trait_on_number.krb

```kirby
trait Squarable { fun squared(self): f64; }

impl Squarable for f64 {
    fun squared(self): f64 = self * self;
}

fun main(): unit {
    let three: f64 = 3;
    @println(three.squared());
    @println(Squarable.squared(three));
}
```

###### Expected Outcome

Prints `9` twice, if [Q-user-impls] is answered yes.

## Questions

### **Q:** Should bare (non-trait) `impl` on primitives be allowed too, or should primitives stay trait-impl-only?

[Q-inherent]: #q-should-bare-non-trait-impl-on-primitives-be-allowed-too-or-should-primitives-stay-trait-impl-only

**Status:** Answered

Options: (a) only trait impls on primitives, so every method added to a type is named by a trait. (b) both forms.

#### Answer

(b), in the standard library only. `impl string { ... }` is how `len` is written. A program can write neither form. Whether a program may write the trait form is [Q-user-impls].

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

(b). It implements `Display`, `Eq`, `Ord` and `Default`. What those methods return is [Q-unit-methods].

### **Q:** What should `unit`'s trait methods return?

[Q-unit-methods]: #q-what-should-units-trait-methods-return

**Status:** Open

Today `toString` is `"nil"`, `equals` is `true`, `cmp` is `0` and `default` is `()`. Options: (a) keep all four. (b) keep only `Default`. (c) remove the impls. `$"{()}"` is rejected today, which (a) would suggest allowing. The answer should agree with [Q-unit-empty].

### **Q:** Should printing use a primitive's `Display` impl?

[Q-print-display]: #q-should-printing-use-a-primitives-display-impl

**Status:** Open

`@println` and string interpolation call a struct's `Display` impl. For numbers, strings and `bool` they use built-in formatting, so a primitive's `Display` impl runs only when called by name.

Options: (a) use the impl. One consistent rule. (b) keep built-in formatting, which is what is built. Only the standard library writes these impls, so the two print the same text today.

### **Q:** Should the design anticipate more numeric primitive kinds now, or wait for that proposal?

[Q-numeric-family]: #q-should-the-design-anticipate-more-numeric-primitive-kinds-now-or-wait-for-that-proposal

**Status:** Answered

Options: (a) design for any primitive kind. (b) only `f64`, `bool` and `string`.

#### Answer

(a). [Sized Number Types][sized-number-types] is Closed, and all ten number kinds have impls.

### **Q:** May a program implement its own traits for primitive types?

[Q-user-impls]: #q-may-a-program-implement-its-own-traits-for-primitive-types

**Status:** Open

The first request for this feature was that Kirby code can `impl` and implement traits for primitives. The first version allows only the standard library.

Options: (a) a program may implement a trait it declares, but not a built-in trait, and may not add methods without a trait. This is Rust's [orphan rules] and its rule against [methods on primitives]. (b) keep it to the standard library.

(a) is the recommendation.

### **Q:** Can a list or array impl require something of its item type?

[Q-list-bounds]: #q-can-a-list-or-array-impl-require-something-of-its-item-type

**Status:** Open

`impl[T: Eq] Eq for [T]` is the error `A list or array impl's parameters can't have bounds.`, and `impl Eq for [i64]` is rejected too. One impl covers every list, so `Eq`, `Ord` and `Display` for lists can not use the items' methods. Rust writes `impl<T: PartialEq> PartialEq for Vec<T>`, and Swift calls this [conditional conformance].

Options: (a) allow bounds on these parameters. (b) leave lists without these traits. (a) is of little use for `[i64]` or `[string]` until primitives can fill bounds.

### **Q:** Should `==` and `contains` on lists use the item type's `Eq`?

[Q-list-equality]: #q-should--and-contains-on-lists-use-the-item-types-eq

**Status:** Open

`==` on two lists or arrays compares identity, so `[1] == [1]` is `false`. `contains` compares items the same way, so a list of structs does not find an equal struct, even with `impl Eq`. Rust compares contents with the item's equality.

Options: (a) write `contains` and `equals` in Kirby under `impl[T: Eq]`, which needs [Q-list-bounds] and monomorphization, and decide whether `==` compares contents. (b) the C code calls the program's `equals`, which means a native calling Kirby code. (c) keep identity.

### **Q:** How are floats ordered when a value is `NaN`?

[Q-float-order]: #q-how-are-floats-ordered-when-a-value-is-nan

**Status:** Open

`nan <= 1.0` and `nan >= 1.0` are `true` today, because `<=` is compiled as "not greater". `nan.cmp(1.0)` is `0`, which means equal, so a generic `a <= b` is also `true`. In Rust all four comparisons with `NaN` are `false`, and `f64` implements [PartialOrd] and not `Ord`.

The direct operators can be fixed alone, with instructions for "less or equal" and "greater or equal". For `Ord`, options: (a) add a `PartialOrd` trait, which needs a way to answer "no order". (b) remove `Ord` from floats. (c) give `cmp` an order that includes `NaN`, which then differs from the operators.

### **Q:** Can a primitive's method be used as a value?

[Q-method-values]: #q-can-a-primitives-method-be-used-as-a-value

**Status:** Open

A struct's method can be: `let f = p.get;`. A primitive's can not. Rust allows neither form but allows `str::len`.

Options: (a) allow it for primitives. (b) remove it for structs, which breaks programs. (c) keep the difference and document it.

### **Q:** Can two traits give one type the same method name?

[Q-method-names]: #q-can-two-traits-give-one-type-the-same-method-name

**Status:** Open

It is the error `'name' is already provided to 'P' by another trait.` Rust allows it: a plain `p.name()` is then ambiguous, and `A::name(&p)` picks one.

Options: (a) allow it, make the plain call an error, and use `A.name(p)`. (b) keep the error.

### **Q:** How is a static trait method called when the type has its own of that name?

[Q-shadowed-static]: #q-how-is-a-static-trait-method-called-when-the-type-has-its-own-of-that-name

**Status:** Open

`P.make()` calls the type's own `make`, and `A.make()` is the error `'make' is a static method of 'A'. Call it on a type instead.` So the trait's `make` can not be called. Rust writes `<P as A>::make()`.

This needs a syntax. No option is proposed yet.

## Glossary

- **Changes**: Changes refer to the proposed changes in this document
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
[Q-compare]: ../strings/PROPOSAL.md#q-is-string-ordering-an-operator-rule-or-an-ord-impl
[Q-primitives]: ../trait-types/PROPOSAL.md#q-can-a-number-string-or-bool-convert-to-a-trait-type
[Q-unit-empty]: ../tuples/PROPOSAL.md#q-is-unit-the-empty-tuple

<!-- External -->

[PR #178]: https://github.com/kirbylang/kirbylang/pull/178
[PR #183]: https://github.com/kirbylang/kirbylang/pull/183
[PR #184]: https://github.com/kirbylang/kirbylang/pull/184
[PR #186]: https://github.com/kirbylang/kirbylang/pull/186
[orphan rules]: https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
[methods on primitives]: https://doc.rust-lang.org/error_codes/E0390.html
[conditional conformance]: https://swift.org/blog/conditional-conformance/
[PartialOrd]: https://doc.rust-lang.org/core/cmp/trait.PartialOrd.html
