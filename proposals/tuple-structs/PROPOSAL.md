---
status: Draft
created: 2026-09-20
from_commit: 662d98b
---

# Proposal: Tuple Structs

This proposal adds tuple structs to Kirby: structs whose fields have no names and are read by position. `struct Value(f64);` declares one, `Value(123)` builds one, and `val.0` reads its field. Fields are private and cannot be assigned to unless marked otherwise. This is a skeleton built from [Issue #23], which is a rough sketch, so the details are expected to change.

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

Wrapping one value, or a small group of values, in its own type needs a full struct with named fields today:

```kirby
struct Value {
  var value: f64;
}

let val = Value { value: 123 };

print val.value;
```

When the name of the field says nothing more than the name of the struct, it is noise. It also does not help to use a `type` alias. An alias is only another name for the same type. Verified against `from_commit`, this compiles and runs:

```kirby
type Meters = f64;
type Feet = f64;

let m: Meters = 1;
let f: Feet = m;
```

So an alias cannot stop a length in meters being used as a length in feet. A tuple struct is a new type with its own identity and almost no ceremony.

[Issue #23] lists these requirements:

- Tuple struct fields are private and immutable by default
- `pub` makes the field public
- `var` makes the field mutable
- Fields are referenced by numeric position, `self.0`, `self.1`

and gives these examples (shown here with the output Kirby really prints for a number, `123.000000`):

```kirby
struct Value(f64);

impl Value {
  pub fun value(self) = self.0;
}

let val = Value(123);

print val.value(); // 123.000000
```

```kirby
struct Value(pub f64);

let val = Value(123);

print val.0; // 123.000000

val.0 = 456; // Error. Fields on tuple structs are immutable by default
```

```kirby
struct Value(pub var f64);

let val = Value(123);

val.0 = 456;

print val.0; // 456.000000
```

```kirby
struct Value(var f64);

impl Value {
  pub fun new(value: f64): Self = Self(value);

  pub fun get(self): f64 = self.0;

  pub fun set(self, value: f64): unit {
    self.0 = value;
  }
}

let val = Value.new(123);

val.set(456);

print val.get(); // 456.000000
```

None of this works today. Verified against a clean build of `from_commit`, `struct Value(f64);` reports `Expect '{' before struct body.` and then `Expect '}' after struct body.`

One detail in the sketch needs fixing to run at all. The first example's `pub fun value(self) = self.0;` has no return type, and Kirby methods need one today (verified with a method named `get`, which reports `'get' needs a return type.`). This proposal uses `pub fun value(self): f64 = self.0;`.

Several rules for regular structs at `from_commit` affect this design. Each was verified:

- **Calling a struct.** `Struct(123)` fails at run time with `Can only call functions and structs.` (exit code 70), and the type checker does not stop it. Even `Self(1)` inside an `impl` block compiles and only fails when it runs. The 0.4.0 entry in `docs/CHANGELOG.md` says "Deprecate the call syntax e.g. `Point()`". See [Q-constructor].
- **Field mutability.** A field that can be reached can always be assigned. `let` fields do not exist (`pub let x: f64;` reports `Expect field declaration after 'pub'.`), and a `let` binding does not protect the fields of what it holds: `let s = S { x: 1 }; s.x = 2;` works. Immutable fields would be new. See [Q-immutability].
- **Field privacy.** A private field can be used by the methods in the struct's own `impl` block, on any instance of that struct. Visibility is per struct, not per instance. Reading one from outside is a run time error, not a compile error: `Field 'balance' is private to 'Account'.` See [Q-enforcement].
- **Printing.** `print S { x: 1 };` prints `S instance`.

## Proposed Changes

### Declaration

A tuple struct is declared with a type for each field in parentheses. Each field can have `pub` and `var` in front of its type, in that order:

```kirby
struct Meters(f64);             // private, cannot be assigned
struct Point(pub f64, pub f64); // public, cannot be assigned
struct Counter(pub var f64);    // public, can be assigned
```

| Declared as | Read outside `impl` | Assign |
| ----------- | ------------------- | ------ |
| `f64`       | no                  | no     |
| `pub f64`   | yes                 | no     |
| `var f64`   | no                  | yes, inside `impl` only |
| `pub var f64` | yes               | yes    |

Like a regular struct, a tuple struct has its own type, and two tuple structs with the same fields are still different types. The declaration must be at the top level, the same as `struct`.

### Building and reading

- `Value(123)` builds a value. Inside an `impl` block, `Self(value)` does the same. The number and types of the arguments are checked. How this is written is [Q-constructor].
- `val.0`, `val.1`, and so on read a field by position, using the syntax from the [Tuples Proposal].
- `impl Value { ... }` and `impl Trait for Value { ... }` work as they do for regular structs.
- How a tuple struct is stored is [Q-representation]. Whether `==` compares its fields is [Q-equality]. How it prints is [Q-print].

```kirby
struct Meters(pub f64);
struct Feet(pub f64);

let m: Meters = Meters(1);
let f: Feet = m; // Compile error: a Meters is not a Feet
```

### Goals and Non Goals

What this proposal covers:

- Declaring a tuple struct with positional fields
- `pub` and `var` on each field
- Building a value and reading a field by position
- `impl` blocks and trait impls on tuple structs

The following is intentionally left out of scope for this proposal:

- Taking a tuple struct apart, such as `let Value(v) = x;`. See the [Destructuring Proposal] and the [Pattern Matching Proposal].
- Unit structs with no fields at all, such as `struct Marker;`
- Generic tuple structs, such as `struct Wrapper[T](T);`. See the [Generic Types Proposal].
- Structs with both named and positional fields
- Making regular struct fields immutable. This is raised in [Q-immutability] but would be its own proposal.

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Parser

- `structDeclaration` in `src/parser.c` currently expects `{` after the struct name. It should also accept `(`, then a list of fields, each `[pub] [var] Type`, then `)` and `;`.

#### Part 2: Reading a field by position

- Reading `.0` is shared with the [Tuples Proposal], Part 1. This proposal depends on it, including the fix for `self.0.1` ([Q-nested-access-tuples]).

#### Part 3: Type checker

- A tuple struct is a struct type whose fields are named `0`, `1`, and so on. These are not valid names in Kirby code, so they cannot clash with anything the user writes.
- A struct type today records each field's name and type, and nothing else. It needs to record whether each field can be assigned so the checker can reject `val.0 = 456` (see [Q-enforcement]).
- The struct name becomes callable. The call is checked like a function call: one argument per field, each of the field's type, returning the struct's type. `Self(...)` in an `impl` block is the same.

#### Part 4: Compiler and virtual machine

- Reuse the struct opcodes. `OP_STRUCT` and `OP_FIELD` declare the struct and its fields, and `OP_FIELD` already carries a flag for public or private. A flag for assignable would sit next to it if the check is done at run time.
- Building a value from a call needs the virtual machine to accept a struct in call position. Today it reports `Can only call functions and structs.` for a call with arguments.

#### Part 5: Docs and tooling

- `docs/TYPES.md` gets a Tuple Structs section under Structs. `docs/CHANGELOG.md` gets an entry.
- Check the VS Code extension for anything that needs updating.

## Impacts

### Existing Syntax Or Behavior

- `struct Name(` is a compile error today, so no valid program changes meaning.
- Calling a regular struct name with `()` behaves as it does today. Tuple structs would be the first structs where `Name(...)` is valid, which needs care while the call syntax on structs is being deprecated ([Q-constructor]).
- Regular structs, and every rule in the Problem Statement, are unchanged.
- Immutable fields would exist only on tuple structs, so the two kinds of struct would differ ([Q-immutability]).

### Related Proposals

- [Tuples Proposal] — this proposal depends on it for reading an item by position, and for the answer to [Q-nested-access-tuples]. If tuple items can be assigned to ([Q-mutability-tuples]), the rules for the two proposals should agree.
- [Destructuring Proposal] — `let Value(v) = x;` is a destructuring pattern. It depends on this proposal.
- [Pattern Matching Proposal] — tuple struct patterns depend on this proposal.
- [Enums Proposal] — a variant like `Bool(bool)` is a close relative of a tuple struct. Both need positional fields, and the two should share how they are stored and how they are written.
- [Generic Types Proposal] — generic tuple structs would need the same substitution work as generic structs (Part 3.5 there). Out of scope here.
- [Top-Level Declarations Proposal] — a tuple struct declaration is a declaration, and it must be at the top level anyway. A value built as `Meters(1)` only builds data, so it would count as a comptime value when its arguments do, the same as a struct literal, once [Q-constructor] settles how it is written. The samples here that use top-level statements are converted when [Q-samples] there is settled.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

##### NEW: tests/tuple_structs/tuple_struct_method.krb

```kirby
struct Value(f64);

impl Value {
  pub fun value(self): f64 = self.0;
}

let val = Value(123);

print val.value();
```

###### Expected Outcome

Prints `123.000000`. Exit code 0.

##### NEW: tests/tuple_structs/tuple_struct_multiple_fields.krb

```kirby
struct Pair(pub f64, pub string);

let p = Pair(1, "a");

print p.0;
print p.1;
```

###### Expected Outcome

Prints `1.000000` then `a`.

##### NEW: tests/tuple_structs/tuple_struct_private_field_outside.krb

```kirby
struct Value(f64);

let val = Value(123);

print val.0;
```

###### Expected Outcome

An error saying the field is private. Whether it is a compile error (exit code 65) or a run time error (exit code 70) is [Q-enforcement].

##### NEW: tests/tuple_structs/tuple_struct_immutable_assign.krb

```kirby
struct Value(pub f64);

let val = Value(123);

val.0 = 456;
```

###### Expected Outcome

An error saying the field cannot be assigned to. Whether it is a compile error or a run time error is [Q-enforcement].

##### NEW: tests/tuple_structs/tuple_struct_var_assign.krb

```kirby
struct Value(pub var f64);

let val = Value(123);

val.0 = 456;

print val.0;
```

###### Expected Outcome

Prints `456.000000`.

##### NEW: tests/tuple_structs/tuple_struct_new_get_set.krb

```kirby
struct Value(var f64);

impl Value {
  pub fun new(value: f64): Self = Self(value);

  pub fun get(self): f64 = self.0;

  pub fun set(self, value: f64): unit {
    self.0 = value;
  }
}

let val = Value.new(123);

val.set(456);

print val.get();
```

###### Expected Outcome

Prints `456.000000`. Covers `Self(...)`, and assigning a `var` field from inside `impl`.

##### NEW: tests/tuple_structs/tuple_struct_wrong_argument_count.krb

```kirby
struct Value(f64);

let val = Value(1, 2);
```

###### Expected Outcome

Compile error about the number of arguments. Exit code 65.

##### NEW: tests/tuple_structs/tuple_struct_wrong_argument_type.krb

```kirby
struct Value(f64);

let val = Value("a");
```

###### Expected Outcome

Compile error that `f64` was expected and `string` was given. Exit code 65.

##### NEW: tests/tuple_structs/tuple_struct_position_out_of_range.krb

```kirby
struct Value(pub f64);

let val = Value(1);

print val.1;
```

###### Expected Outcome

Compile error that the struct has no field at position `1`. Exit code 65.

##### NEW: tests/tuple_structs/tuple_struct_distinct_types.krb

```kirby
struct Meters(pub f64);
struct Feet(pub f64);

let m: Meters = Meters(1);
let f: Feet = m;
```

###### Expected Outcome

Compile error that a `Feet` was expected and a `Meters` was given. Exit code 65.

## Questions

### **Q:** How is a tuple struct constructed?

<!-- [Q-constructor]: #q-how-is-a-tuple-struct-constructed -->

**Status:** Open

The sketch builds a value with a call, `Value(123)`, and `Self(value)` inside `impl`. The 0.4.0 entry in `docs/CHANGELOG.md` says to deprecate "the call syntax e.g. `Point()`". That deprecation presumably means calling a regular struct like a function, which `Struct(123)` fails at run time today (verified). Tuple structs would give the same shape a new meaning for one kind of struct.

Options: (a) Use the call, as sketched. It is short and familiar from other languages, and the checker can treat a tuple struct's name as a function of its fields. The deprecation is then only for regular structs, and the changelog wording should say so. (b) Use braces with positions, `Value { 0: 123 }`, which follows how regular structs are built and does not touch the call syntax. It is more to type and looks odd. (c) Provide no literal and require an `impl` block to define a `new` function. This makes tuple structs much less useful.

### **Q:** Should fields be immutable by default when regular struct fields cannot be?

<!-- [Q-immutability]: #q-should-fields-be-immutable-by-default-when-regular-struct-fields-cannot-be -->

**Status:** Open

The sketch makes a tuple struct field immutable unless it is marked `var`. Regular structs have no such thing. Every field of a regular struct is declared with `var`, there is no immutable form (`pub let x: f64;` is a parse error), and a `let` binding does not stop the fields of the value it names from being assigned. So `var` would mean "this field can be assigned" on a tuple struct and only "this is a field" on a regular struct.

Options: (a) As sketched. The two kinds of struct differ, and the difference is documented. (b) Tuple struct fields can always be assigned, like regular struct fields, and `var` is not part of tuple struct syntax. This is consistent but drops a requirement in the sketch. (c) Do (a) now, and propose immutable fields for regular structs separately so the two kinds match later.

The [Tuples Proposal] asks the same thing about tuple items ([Q-mutability-tuples]).

### **Q:** When are visibility and mutability checked?

<!-- [Q-enforcement]: #q-when-are-visibility-and-mutability-checked -->

**Status:** Open

Privacy is checked when the program runs today: `print a.balance;` on a private field reports `Field 'balance' is private to 'Account'.` and exits with code 70. The struct type the checker builds only records each field's name and type, so it cannot report this earlier.

Options: (a) Check both at run time, the same as privacy. It is the smallest change. It needs a flag next to the public flag on `OP_FIELD` and in `ObjStruct`. A mistake is only found when the line runs. (b) Check both at compile time, by recording them in the struct type. Errors are found earlier and the sketch's "Error." comments read naturally. (c) Do (b) for these new checks and leave regular struct privacy as it is.

### **Q:** Is a tuple struct a regular struct with numbered fields?

<!-- [Q-representation]: #q-is-a-tuple-struct-a-regular-struct-with-numbered-fields -->

**Status:** Open

Options: (a) Yes. The compiler declares a struct whose fields are named `0`, `1`, and so on. The existing struct opcodes, the garbage collector, `impl` blocks, and trait impls all work with no new code. (b) A new kind of heap object. More code, but positions could be looked up by number rather than by name.

(a) is the smaller change. It also means anything that lists a struct's fields would see fields named `0` and `1`, which should be checked before choosing it.

### **Q:** Does equality compare tuple structs by their fields?

<!-- [Q-equality]: #q-does-equality-compare-tuple-structs-by-their-fields -->

**Status:** Open

A wrapper type like `Meters(1)` is most useful when `Meters(1) == Meters(1)` is `true`. For regular structs, `==` checks at compile time that the struct implements `Eq` but compares identity at run time (verified; described in the [Generic Types Proposal] Part 1.4), so two equal-looking values compare as different.

Options: (a) Same as regular structs. It is consistent, and any change comes from the [Generic Types Proposal] 10.4. (b) Compare fields item by item, like the proposed rule for tuples ([Q-equality-tuples]). More useful, but differs from regular structs.

### **Q:** How does a tuple struct print?

<!-- [Q-print]: #q-how-does-a-tuple-struct-print -->

**Status:** Open

A regular struct value prints as its name followed by `instance`, so `print S { x: 1 };` prints `S instance` (verified). Options: (a) Do the same, `Value instance`. (b) Print the name and the fields, `Value(123.000000)`, which matches how the value is built. (a) needs no work, and (b) is friendlier.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Tuple struct**: A struct whose fields have no names and are read by position.
- **Position**: The number that says which field is meant. The first field is at position `0`.
- **Field**: One value stored in a struct.
- **Private field**: A field that can only be used inside the `impl` blocks of its own struct.
- **Immutable**: Cannot be assigned to after the value is built.
- **Alias**: Another name for an existing type, made with `type`. It does not make a new type.
- **Call syntax**: Writing a name followed by arguments in parentheses, like `Value(123)`.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Tuples Proposal]: ../tuples/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-nested-access-tuples]: ../tuples/PROPOSAL.md#q-how-should-pair01-be-scanned
[Q-mutability-tuples]: ../tuples/PROPOSAL.md#q-can-an-item-of-a-tuple-be-assigned-to
[Q-equality-tuples]: ../tuples/PROPOSAL.md#q-how-does-tuple-equality-compare-items
[Q-samples]: ../top-level-declarations/PROPOSAL.md#q-when-do-code-samples-in-other-proposals-change

<!-- External -->

[Issue #23]: https://github.com/kirbylang/kirbylang/issues/23

<!-- Questions -->

[Q-constructor]: #q-how-is-a-tuple-struct-constructed
[Q-immutability]: #q-should-fields-be-immutable-by-default-when-regular-struct-fields-cannot-be
[Q-enforcement]: #q-when-are-visibility-and-mutability-checked
[Q-representation]: #q-is-a-tuple-struct-a-regular-struct-with-numbered-fields
[Q-equality]: #q-does-equality-compare-tuple-structs-by-their-fields
[Q-print]: #q-how-does-a-tuple-struct-print
