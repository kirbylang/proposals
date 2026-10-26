---
status: Draft
category: feature
created: 2026-09-20
from_commit: 2f96a2e
---

# Proposal: Tuples

This proposal adds tuples to Kirby: fixed size groups of values where each value can have a different type. A tuple is written `(a, b, c)`, its type is written `(A, B, C)`, and its items are read by position, like `values.0`. This is a skeleton built from [Issue #33], which is a rough sketch, so the details are expected to change.

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

Kirby has two ways to group values today. Neither fits the case of "a few values of different types, with no declaration":

- **Arrays** (`[T]`) hold any number of items, but every item must have the same type. Verified against `from_commit`: `let a = [1, "a"];` is the compile error `Expected string, got i64.`
- **Structs** can hold different types, but every group needs a `struct` declaration, named fields, and a `Name { field: value }` literal. A function that wants to return two values has to declare a struct first.

[Issue #33] proposes tuples:

```kirby
let values: (number, number, number, number) = (100, 405, 10, 84);

@println(values.0); // 100
@println(values.1); // 405
@println(values.2); // 10
@println(values.3); // 84
@println(values.4); // Compile Error: (number, number, number, number) does not have field `4` (or similar wording)
```

`number` is not a type in user code: `let x: number = 1;` reports `Unknown type.` Kirby has ten number kinds instead ([Sized Number Types Proposal]). A whole number literal such as `100` is an `i64` and a decimal literal such as `1.5` is an `f64`, so `(100, 405)` would be an `(i64, i64)`. Examples in this proposal use `i64` and `f64`.

None of the sketch works today. Verified against a clean build of `from_commit`:

```
let t = (1, 2);                 // Error at ',': Expect ')' after expression.
let t: (f64, f64) = (1, 2);     // Error at '(': Expect type name.
@println((0,));                 // Error at ',': Expect ')' after expression.
@println(s.0);                  // Error at '0': Expect property name after '.'.  (s is a struct)
```

Two things that already exist shape the design:

1. **`()` is already a value.** It is the unit value, added in 0.4.0. The parser function `grouping()` in `src/parser.c` checks for `)` straight after `(` and returns a unit literal. At run time unit is `nil`: `let u = (); @println(u);` prints `nil`.
2. **`(x)` is already a grouping**, meaning parentheses around one expression, like `(1 + 2) * 3`. A tuple with one item has to be told apart from it. The issue does this with a trailing comma: `(0,)` is a tuple and `(0)` is a grouping.

## Proposed Changes

### Literals and types

- A tuple literal is two or more expressions in parentheses: `(1, "a")`. One item needs a trailing comma: `(1,)`.
- `(x)` stays a grouping and `()` stays the unit value. Neither changes.
- A tuple type is written the same way: `(f64, string)`, and `(f64,)` for one item.
- Tuple types are compared by their parts. `(f64, string)` is the same type wherever it is written, and order matters. This is the same rule Kirby already uses for function types and array types. Structs are different: two structs are the same type only if they are the same declaration.

```kirby
fun sumAndProduct(a: f64, b: f64): (f64, f64) = (a + b, a * b);

let result = sumAndProduct(3, 4);

@println(result.0); // 7
@println(result.1); // 12

let entry: (string, f64) = ("apples", 3);
```

### Reading items

- `values.0`, `values.1`, and so on read an item by position. The position is written in the source, not calculated, so the compiler knows which item is read and what its type is.
- A position past the end is a compile error. The issue suggests wording like "does not have field `4`".
- `values[0]` is a compile error: "Tuples can not be indexed". There is already a compile time error of this kind for structs: `let s = S { x: 1 }; @println(s[0]);` reports `Can't index into a S.`
- Whether an item can be assigned to, like `values.0 = 5`, is [Q-mutability].

### Equality

`==` and `!=` compare two tuples item by item:

```kirby
@assert((1, 2, 3) == (1, 2, 3), "Tuple equality is based on the items equality");
@assert((1, 2, 3) != (4, 5, 5), "Tuple equality is based on the items equality");
```

How each item is compared is [Q-equality].

### Length

The sketch asks for the number of items. Kirby has `@strLen`, typed `fun (string) => i64`, and `@listLen`, typed `fun ([T]) => i64` (`src/native_signatures.c`). There is also `@arrLen`, typed `fun ([T; N]) => i64`, for the fixed size array. None of them accepts a tuple. Whether a tuple gets a length native of its own is [Q-len].

### Goals and Non Goals

What this proposal covers:

- Tuple literals, including the trailing comma rule for one item
- Tuple types, compared by their parts
- Reading an item by position, checked at compile time
- `==` and `!=` between tuples of the same type
- A length native for tuples, if [Q-len] keeps it

The following is intentionally left out of scope for this proposal:

- Taking a tuple apart into named variables, such as `let (a, b) = t;`. See the [Destructuring Proposal].
- Matching on a tuple's shape. See the [Pattern Matching Proposal].
- Methods and `impl` blocks on tuples. See the [Collection Methods Proposal].
- Naming the items of a tuple
- Reading an item by a position that is calculated at run time
- Joining or splitting tuples
- Tuple types inside other types, like `[(f64, f64)]` or `Box[(f64, f64)]`. See the [Generic Types Proposal].

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Scanner and parser

- In `grouping()` in `src/parser.c`, after the first expression, a `,` starts a tuple literal instead of expecting `)`.
- The type parser accepts `(` at the start of a type.
- `.` followed by a number is accepted as a property access. There is a scanner problem here. `number()` in `src/scanner.c` reads digits and then an optional `.` and more digits, so `pair.0.1` is scanned as `pair`, `.`, and the single number `0.1` (verified with `krb lex`). See [Q-nested-access].

#### Part 2: AST and type checker

- New AST node kinds for a tuple literal and a tuple type. The property access node accepts a number as its name.
- A new type kind, `TYPE_TUPLE`, in `src/types.h`. It holds the item types. `typesEqual` and `typeToString` in `src/types.c` get a case for it.
- `typchkInferGet` only allows `.name` on a struct today and reports `Can't access '.foo' on a f64.` for anything else. It needs a tuple branch that checks the position and returns the item's type.
- `==` and `!=` are allowed when both sides are the same tuple type.

#### Part 3: Compiler and virtual machine

- A new kind of heap object for a tuple (see [Q-representation]), which the garbage collector in `src/gc.c` must know how to visit, the same as `ObjArray`.
- One opcode to build a tuple from the top N values on the stack, and one to read the item at a fixed position.
- The array opcode `OP_ARRAY` carries its count in one byte, and an array literal with more than 255 items is a compile error (`Too many elements in array literal.`). A tuple opcode built the same way would have the same limit.
- `valuesEqual` in `src/value.c` compares objects by identity. Tuple equality needs an item by item comparison.
- `@print`, `@println` and `@typeof` each get a tuple case, and so does a length native if [Q-len] adds one. The proposed print format is `(1, 2)`, matching how arrays print (`[1, 2, 3]`), and `(1,)` for one item.

#### Part 4: Docs and tooling

- `docs/TYPES.md` gains a Tuples section, and "Tuples" moves from Future State to a release in `docs/CHANGELOG.md`.
- Check the VS Code extension for anything that needs updating.

## Impacts

### Existing Syntax Or Behavior

- Every new form is a compile error today (see Problem Statement), so no valid program changes meaning. `(x)` and `()` are unchanged.
- `==` on two objects compares identity. `@println([1, 2] == [1, 2]);` prints `false`. Tuples would be the first object type whose `==` looks inside it, so tuples and arrays would behave differently. See [Q-equality].
- A length native may be added ([Q-len]).

### Related Proposals

- [Collection Methods Proposal] — treats tuples as a possible collection type. Both proposals spell the type `(f64, string)` ([Q-type-spelling]). The question there about whether collection types are one category or two ([Q-category]) is informed by this proposal: a tuple has a fixed size and its item types are known at compile time.
- [Primitive Impls Proposal] — asks whether `unit` gets impls ([Q-unit]). Whether `unit` is the empty tuple is [Q-unit-empty] here, and the two answers should agree.
- [Generic Types Proposal] — 10.4 is about `==` on objects, which tuple equality depends on. Tuple types also need to be handled by the type substitution helper in 3.6. See 10.9 there.
- [Tuple Structs Proposal] — depends on this proposal for reading an item by position.
- [Destructuring Proposal] — tuple patterns like `let (a, b) = t;` depend on this proposal.
- [Pattern Matching Proposal] — tuple patterns depend on this proposal.
- [Map Proposal] and [For-Each Proposal] — looping over a map gives a pair of key and value, which is a tuple. `for ((key, value) in map)` needs this proposal and the [Destructuring Proposal].
- [Sized Number Types Proposal] — Closed. It added ten number kinds and no `number` type, so the sketch's `number` becomes `i64` or `f64`.
- [Top-Level Declarations Proposal] — a tuple literal only builds data, so it would count as a comptime value when its items do, and could start a top-level `let` or `var`. The samples here that use top-level statements are converted when [Q-samples] there is settled.
- [Formatter Proposal] — the trailing comma in a one-item tuple, `(1,)`, is what tells it apart from a grouping, so the formatter always keeps it. Whether other lists may end with a comma is [Q-trailing-format] there. A tuple literal is a list for its line breaking rules.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

##### NEW: tests/run/tuples/tuple_literal.krb

```kirby
let entry: (string, f64) = ("apples", 3);

@println(entry.0);
@println(entry.1);
```

###### Expected Outcome

Prints `apples` then `3`. Exit code 0. The `3` is an `f64`, because the type says so.

##### NEW: tests/run/tuples/tuple_single_item.krb

```kirby
let one = (7,);
let grouped = (7);

@println(one.0);
@println(grouped);
```

###### Expected Outcome

Prints `7` twice. `one` is a tuple with one item and `grouped` is a plain `i64`.

##### NEW: tests/run/tuples/tuple_unit_unchanged.krb

```kirby
@println(());
```

###### Expected Outcome

Prints `nil`, the same as today. Guards the unit value against the new tuple parsing.

##### NEW: tests/run/tuples/tuple_nested_position.krb

```kirby
let nested = ((1, 2), 3);

@println(nested.0.1);
```

###### Expected Outcome

Prints `2`. Guards the `0.1` scanning problem in [Q-nested-access].

##### NEW: tests/run/tuples/tuple_position_out_of_range.krb

```kirby
let t = (1, 2);

@println(t.2);
```

###### Expected Outcome

Compile error naming the tuple type and the position. Exit code 65. The exact message is to be decided.

##### NEW: tests/run/tuples/tuple_index_with_brackets.krb

```kirby
let t = (1, 2);

@println(t[0]);
```

###### Expected Outcome

Compile error saying tuples can not be indexed. Exit code 65.

##### NEW: tests/run/tuples/tuple_equality.krb

```kirby
@println((1, 2, 3) == (1, 2, 3));
@println((1, 2, 3) != (4, 5, 5));
@println((1, 2) == (1, 3));
```

###### Expected Outcome

Prints `true`, `true`, `false`.

##### NEW: tests/run/tuples/tuple_equality_different_types.krb

```kirby
@println((1, 2) == (1, "a"));
```

###### Expected Outcome

Compile error because the two sides are different types. Exit code 65.

##### NEW: tests/run/tuples/tuple_wrong_item_type.krb

```kirby
let t: (f64, string) = (1, 2);
```

###### Expected Outcome

Compile error that item 1 expected `string` and got `i64`. Exit code 65.

##### NEW: tests/run/tuples/tuple_len.krb

Only if [Q-len] adds a length native. It is called `@tupleLen` here.

```kirby
@println(@tupleLen((1, 2, 3)));
```

###### Expected Outcome

Prints `3`.

## Questions

### **Q:** How should `pair.0.1` be scanned?

<!-- [Q-nested-access]: #q-how-should-pair01-be-scanned -->

**Status:** Open

Reading an item of an item, like `nested.0.1`, is the natural way to use nested tuples. It does not work with the current scanner, which reads `0.1` as one number (verified: `a.0.1` scans as `a`, `.`, `0.1`).

Options: (a) after a `.` token, the scanner reads only whole number digits, so `0.1` becomes `0`, `.`, `1`. (b) The parser splits a number token like `0.1` into two positions when it follows a `.`. (c) Require parentheses, `(nested.0).1`, and leave the scanner alone. (a) is the smallest change but the scanner then has to remember the previous token. (c) is the simplest to build but is surprising to write.

[Tuple Structs Proposal] needs the same answer for `self.0.1`.

### **Q:** What is a tuple at run time?

<!-- [Q-representation]: #q-what-is-a-tuple-at-run-time -->

**Status:** Open

Options: (a) A new kind of heap object that holds a fixed list of items. `ObjArray` (from the [Lists Proposal]) is already a heap object whose count never changes, with no `capacity`, so this would be a second object of the same shape. Every place that looks at objects (`@typeof`, printing, `==`, the garbage collector) gets an explicit tuple case. (b) Reuse `ObjArray` and rely on the type checker to stop a tuple being used as an array. This is the least new code, but at run time a tuple and an array look the same, so `@typeof` and `@println` cannot tell them apart. The `@arr*` natives take `[T; N]`, so the type checker stops `@arrLen(tuple)`, but nothing at run time would. (c) A struct that the compiler generates with fields named `0`, `1`, and so on, which shares work with [Tuple Structs Proposal]. But tuple types are compared by their parts and struct types by name, so each tuple type would need its own generated struct.

### **Q:** How does tuple equality compare items?

<!-- [Q-equality]: #q-how-does-tuple-equality-compare-items -->

**Status:** Open

The sketch says equality is "based on the items equality". Today `==` on two objects asks whether they are the same object (`valuesEqual` in `src/value.c`). `@println([1, 2] == [1, 2]);` prints `false`. Two structs that both `impl Eq` also compare by identity under `==`, even though calling `.equals` returns `true` (verified against `from_commit`, and described in the [Generic Types Proposal] Part 1.4).

Options: (a) Items are compared the way `==` compares that item's type today: numbers, bools, and strings by value, nested tuples item by item, and anything else by identity. (b) Items are compared through `Eq` where the item's type has one. This needs the decision in the [Generic Types Proposal] 10.4 first.

Either way, tuples and arrays would compare differently. Changing arrays is out of scope here.

### **Q:** Is `unit` the empty tuple?

<!-- [Q-unit-empty]: #q-is-unit-the-empty-tuple -->

**Status:** Open

Options: (a) No. `unit` and `()` stay as they are and no tuple type has zero items. (b) Yes. `()` becomes the tuple with no items and `unit` becomes another way to write its type.

At run time `()` is `nil` today. Making it a tuple object would change what `@println(());` shows and how `nil ?? x` behaves. (a) avoids that, and matches the [Primitive Impls Proposal] treating `unit` as a primitive.

### **Q:** Can an item of a tuple be assigned to?

<!-- [Q-mutability]: #q-can-an-item-of-a-tuple-be-assigned-to -->

**Status:** Open

The sketch does not say. Array items can be assigned today, even through a `let` binding: `let a = [1, 2]; a[0] = 9;` works (verified). `let` only stops the name being pointed somewhere else. Struct fields can be assigned only when declared with `var`: `let s = S { x: 1 }; s.x = 2;` works for a `var x` field and is a run time error (`Field 'x' is immutable`) for any other.

Options: (a) Tuple items cannot be assigned. `(1, 2)` never changes, which makes item by item equality safer. (b) Items can be assigned, like struct fields. [Tuple Structs Proposal] makes fields immutable unless marked `var`, the same as regular struct fields, so the two proposals should agree.

### **Q:** Is a tuple type written `(A, B)` or `Tuple[A, B]`?

<!-- [Q-type-spelling]: #q-is-a-tuple-type-written-a-b-or-tuplea-b -->

**Status:** Answered

The sketch writes `(number, number)`. A spelling like `Tuple[f64, string]` was also considered. Square brackets after a name are how Kirby writes generic arguments (`Box[T]`), and a tuple has a different number of type arguments each time, which generics do not support. The parenthesized form matches the value syntax.

#### Answer

`(A, B)`. The type looks like the value: `(1, "a")` has the type `(i64, string)`. Square brackets are taken. After a name they are generic arguments (`Box[f64]`), and on their own they are an array type (`[f64]`). `[i64, string]` would read as an odd array type, and `[T; n]` is planned for fixed size arrays (see the [Lists Proposal]). The [Collection Methods Proposal] uses the same spelling.

### **Q:** Should a tuple have a length native?

<!-- [Q-len]: #q-should-a-tuple-have-a-length-native -->

**Status:** Open

The sketch asks for it. But the length of a tuple is part of its type and is always known at compile time, so the answer is never a surprise. Options: (a) add a native such as `@tupleLen`. `@arrLen` can't simply take a tuple,, because its signature is `fun ([T; N]) => i64`. (b) Leave it out, and keep `@strLen` and `@listLen` for values whose length can change.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Tuple**: A fixed size group of values. Each value can have a different type.
- **Item**: One value inside a tuple.
- **Position**: The number that says which item of a tuple is meant. The first item is at position `0`.
- **Grouping**: Parentheses around a single expression, like `(1 + 2)`. The parentheses only change the order things are worked out. They do not make a value.
- **Unit**: The value written `()`, which stands for "nothing useful". Its type is `unit`.
- **Heap object**: A value that lives in memory managed by Kirby's garbage collector, such as a string, array, or struct instance.
- **Compared by their parts**: Two types are the same when their parts are the same, whatever they are called. Function types, array types, and (proposed) tuple types work this way. Struct types are compared by name instead.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Map Proposal]: ../map/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Sized Number Types Proposal]: ../sized-number-types/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md
[Formatter Proposal]: ../formatter/PROPOSAL.md
[Q-trailing-format]: ../formatter/PROPOSAL.md#q-should-lists-broken-over-lines-end-with-a-comma

<!-- Other proposals' questions -->

[Q-category]: ../collection-methods/PROPOSAL.md#q-is-collection-type-one-category-or-two
[Q-unit]: ../primitive-impls/PROPOSAL.md#q-does-unit-get-impls-too
[Q-samples]: ../top-level-declarations/PROPOSAL.md#q-when-do-code-samples-in-other-proposals-change

<!-- External -->

[Issue #33]: https://github.com/kirbylang/kirbylang/issues/33

<!-- Questions -->

[Q-nested-access]: #q-how-should-pair01-be-scanned
[Q-representation]: #q-what-is-a-tuple-at-run-time
[Q-equality]: #q-how-does-tuple-equality-compare-items
[Q-unit-empty]: #q-is-unit-the-empty-tuple
[Q-mutability]: #q-can-an-item-of-a-tuple-be-assigned-to
[Q-type-spelling]: #q-is-a-tuple-type-written-a-b-or-tuplea-b
[Q-len]: #q-should-a-tuple-have-a-length-native
