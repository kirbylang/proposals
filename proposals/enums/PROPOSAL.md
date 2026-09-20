---
status: Draft
created: 2026-09-20
from_commit: 662d98b
---

# Proposal: Enums

This proposal adds enums to Kirby: a type that is exactly one of a fixed set of named variants. A variant can be a plain name (`Red`), can carry values (`Bool(bool)`, or `Function { args: ..., body: ... }`), and an enum can take type parameters (`Option[T]`). It is heavily influenced by Rust. This is a skeleton built from [Issue #39], a first pass in three parts: Simple, With Values, and Generic Params. The three parts depend on different proposals, so the plan builds them in stages.

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

Kirby has no way to define a type that is "one of these alternatives". The workarounds all leave a gap:

- **Numbers or strings as names.** `let red = 0;` says nothing about which other numbers are valid colors, and nothing stops a color being added to a count.
- **A struct with a tag field.** It works, but the tag is again a number or a string, and every place that uses the struct must remember which fields go with which tag.
- **`nil` for "no value".** Several natives may return nothing. The changelog lists `@argv`, `@prompt`, `@stdin`, and `@strIndexOf` as still unchecked, "pending `Option[T]`", because there is no type to describe a result that might be missing.

[Issue #39] sketches enums in three parts. Simple:

```kirby
enum Colors {
  Red, // 0
  Green, // 1
  Blue, // 2
}

impl Display for Colors {
  fun toString(self): string {
    // Nonexistant pattern matching syntax
    match self {
      Red => "Red",
      Green => "Green",
      Blue => "Blue",
    }
  }
}

@assert(Colors.Red == Colors.Red, "Enum variants equal themselves");
@assert(Colors.Red == Colors.Blue, "Enum variants do equal each other");
```

The last line says variants "do equal each other". `Colors.Red == Colors.Blue` would be `false` for two different variants, so the assertion would fail. This proposal reads it as a typo for `!=`, in the same way the tuples issue asserts `(1, 2, 3) != (4, 5, 5)`.

With Values:

```kirby
enum AstNode {
  Bool(bool),
  String(string),
  Number(f64),
  Function {
    args: Array[AstNode],
    body: AstNode,
    returnType: AstNode,
  }
}
```

Generic Params:

```kirby
enum Option[T] {
  None,
  Some(T),
}

enum Result[T, E] {
  Err(E),
  Ok(T),
}
```

None of this works today. Verified against a clean build of `from_commit`, `enum Enum { A, B, C }` reports `Error at 'enum': Expect ';' after expression.` and then `Error at '}': Expect expression.`

Several things that already exist shape the design. Each was verified:

- **`enum` is a valid variable name.** `var enum = 1;` works. No `.krb` file in the repository uses it.
- **Declarations are top level only.** A `struct`, `impl`, `trait`, or `type` inside a function or block is a compile error: `'struct', 'impl', 'trait', and 'type' are declarations and can only appear at the top level.` The check is in `declaration()` in `src/parser.c`.
- **`impl` and traits work on structs only.** `impl Colors` for a name that is not a struct reports `Unknown type 'Colors'.`, and `docs/TYPES.md` lists "Traits can only be implemented for structs currently" as a limitation. The sketch needs `impl Display for Colors`.
- **The type checker has no enum kind.** There are nine kinds of type in `src/types.h`. Struct types are compared by name, and this proposal expects enum types to work the same way.
- **A value has no room for a tag.** A value is a bool, `nil`, a number, or a pointer to a heap object (`src/value.h`). Objects are the only kind that can carry extra data.
- **`Type.name` is already the way to reach a static method.** `Point.origin()` and `Struct.new(123)` work today. `Colors.Red` in the sketch looks the same. There is no `::` token.
- **`Array[T]` is rejected.** `struct Node { pub var args: Array[Node]; }` reports `Generic types aren't supported yet.` But a struct can refer to itself as a field type: `struct Node { pub var next: Node; }` compiles.
- **`==` on objects compares identity.** For structs, `==` also requires `Eq` at compile time (see the [Generic Types Proposal] Part 1.4).
- **`nil` has the type `unit`.** `let x: f64 = nil;` reports `Expected f64, got unit.` So `nil` cannot stand for a missing `f64` in the type system today. `??` exists, and is accepted on a typed value (`let x: f64 = 1; print x ?? 2;` prints `1.000000`).
- **The standard library is empty.** `stdlib/stdlib.krb` has no content, and `src/main.c` runs it before user code in each of its three run modes.

## Proposed Changes

### Part A: Enums with plain variants

```kirby
enum Colors {
  Red,
  Green,
  Blue,
}
```

- An enum is declared at the top level with `enum`, a name, and a list of variants in braces. A trailing comma is allowed, as in the sketch.
- `Colors.Red` is a value of type `Colors`. The type is compared by name, the same as a struct: two enums with the same variants are different types.
- `impl Colors { ... }` and `impl Trait for Colors { ... }` work as they do for structs.
- `==` and `!=` between two values of the same enum are true when they are the same variant. This works without writing an `impl Eq` ([Q-equality]).
- How a variant is written and named is [Q-variant-access]. What a value is at run time is [Q-representation]. Whether a variant has a visible number, like the `// 0` comments in the sketch, is [Q-discriminants]. How a value prints is [Q-print].

A `Display` for an enum can be written by hand with `if` before `match` exists:

```kirby
impl Display for Colors {
  fun toString(self): string =
    if (self == Colors.Red) "Red" else if (self == Colors.Green) "Green" else "Blue";
}
```

### Part B: Variants that carry values

A variant can hold values, in two shapes:

```kirby
enum Shape {
  Circle(f64),
  Rect { width: f64, height: f64 },
  Empty,
}
```

- **Tuple-like:** `Circle(f64)` holds values by position.
- **Struct-like:** `Rect { width: f64, height: f64 }` holds values by name.
- A single enum can mix plain, tuple-like, and struct-like variants, as `AstNode` does.
- An enum can refer to itself, as `body: AstNode` does. Structs can already do this.
- A value is built with the enum and variant names: `Shape.Circle(1)`, `Shape.Rect { width: 1, height: 2 }`. How this is written is [Q-variant-fields].
- The values inside a variant can only be read with a pattern, because the value might be a different variant. This is why Part B depends on the [Pattern Matching Proposal]: without it a value can be built and can never be taken apart.
- `Array[AstNode]` in the sketch needs generics. Until the [Generic Types Proposal] lands it has to be written as `Array`, which is accepted today, and the items are not checked.

### Part C: Enums with type parameters

```kirby
enum Option[T] {
  None,
  Some(T),
}
```

- Type parameters are written in square brackets, as for generic structs (`Struct[T]`), which parse today but are rejected by the checker.
- `Option[T]` and `Result[T, E]` are the two the sketch names. Where they are defined is [Q-option-home], and how `Option[T]` relates to `nil` is [Q-option-nil].
- Part C depends entirely on the [Generic Types Proposal]. It is listed here so Parts A and B are designed so it can follow.
- Once `Option[T]` exists, the natives that the changelog lists as pending it (`@argv`, `@prompt`, `@stdin`, `@strIndexOf`) could be given types.
- The sketch lists `Err` before `Ok` in `Result`, where Rust lists `Ok` first. That only matters if the order of variants can be seen ([Q-discriminants]).

### Goals and Non Goals

What this proposal covers:

- `enum` declarations with plain variants, and `impl` and trait impls on enums
- `==` and `!=` on enums
- Variants with values, in both shapes, and enums that refer to themselves
- Enums with type parameters, once generics exist
- Building the parts in an order where each can ship without the next

The following is intentionally left out of scope for this proposal:

- `match` and checking that every variant is handled. See the [Pattern Matching Proposal].
- Generating `Display` or `Eq` for an enum. The sketch writes `Display` by hand. See the [Macros Proposal].
- Turning an enum into a number and back
- Using a variant without writing the enum's name in front of it. See [Q-bare-variants] and the [Modules Proposal].
- Declaring an enum inside a function or block. The top level rule for declarations stays.
- Replacing `nil` and `??` with `Option[T]` ([Q-option-nil])

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible. Part A needs nothing else. Part B needs the [Pattern Matching Proposal]. Part C needs the [Generic Types Proposal].

#### Part 1: Scanner and parser (A)

- A new keyword token, `TOKEN_ENUM`, in `src/token.h`, recognised in `identifierType` in `src/scanner.c` (see [Q-keyword]).
- An `enumDeclaration` next to `structDeclaration` in `src/parser.c`, and a new AST node for it.
- `declaration()` checks a list of four keywords to give the top level error. `enum` joins that list, and the message changes to name it.

#### Part 2: Type checker (A)

- A new type kind, `TYPE_ENUM`, in `src/types.h`. It is compared by name and holds its variants.
- `Colors.Red` is checked as a value of type `Colors`, and an unknown variant is a compile error.
- `impl` and `impl Trait for` accept an enum as the target. Today the target must be a struct, and the type records which traits it implements. An enum type needs the same record.
- `==` and `!=` are allowed between two values of one enum type ([Q-equality]).

#### Part 3: Compiler and virtual machine (A)

- Declaring an enum, and creating its variants, according to [Q-representation].
- If variants are heap objects, the garbage collector in `src/gc.c` visits them, and `print`, `@typeof`, and `@instanceOf` each get a case.

#### Part 4: Variants with values (B)

- The parser accepts the tuple-like and struct-like shapes in a declaration.
- Building a value. The `{` after an expression currently only accepts a struct name (`Only a struct name can be initialized with '{'.`), so building a struct-like variant needs it to accept the enum and variant names.
- Each variant with values is checked like a function of those values that returns the enum type.
- Values are stored in the heap object, and read by the instructions the [Pattern Matching Proposal] adds.

#### Part 5: Enums with type parameters (C)

- Type parameters on an enum declaration, and type arguments where an enum type is written.
- The type substitution helper in the [Generic Types Proposal] (3.6) handles enum types, the same as struct types.

#### Part 6: Docs and tooling

- `docs/TYPES.md` gets an Enums section, and the limitation about traits only being for structs changes. `docs/CHANGELOG.md` gets an entry.
- `enum` in the VS Code grammar (`vsc/syntaxes/kirby.tmLanguage.json`), and a hover.

## Impacts

### Existing Syntax Or Behavior

- `enum` becomes a reserved word ([Q-keyword]). It is a valid variable name today, so a program that uses it as one would stop compiling. Nothing in this repository does.
- The top level declaration error message changes to include `enum`.
- `impl` and traits are no longer only for structs. The limitation in `docs/TYPES.md` changes.
- `Type.name` starts to mean a variant as well as a static method ([Q-variant-access]).
- Existing structs, and everything in the Problem Statement, are unchanged.

### Related Proposals

- [Pattern Matching Proposal] — Part B depends on it. Part A is useful without it, using `==` and `if`. It also decides how a variant is written in a pattern ([Q-bare-variants]).
- [Destructuring Proposal] — `let` can not take an enum apart, because the value might be a different variant. That is done with `match`.
- [Tuple Structs Proposal] — a variant like `Circle(f64)` is a close relative of a tuple struct. Both need positional values, so how they are built ([Q-constructor]) and stored should agree.
- [Generic Types Proposal] — Part C depends on it. The type substitution helper in 3.6 needs to handle enum types, and 10.4 (`==` on objects) affects [Q-equality].
- [Primitive Impls Proposal] — calling a method on a value that is a plain number has "nowhere to keep a method" (1.3 there). That is the problem Part A would have if variants were numbers ([Q-representation]).
- [Macros Proposal] — the sketch writes `impl Display for Colors` by hand. Derive-style macros (7.7 in the [Generic Types Proposal]) could write it. The macros question about what form syntax takes ([Q-syntax]) could be answered by an enum much like `AstNode`.
- [Modules Proposal] — how an enum and its variants are made visible outside a file.
- [Debugger Proposal] — its variables view shows arrays and struct values, so a new kind of object needs a display.
- [Collection Methods Proposal] — methods that look something up, like finding an item, are natural users of `Option[T]`.
- [Top-Level Declarations Proposal] — `enum` would join `fun`, `struct`, `impl`, `trait`, and `type` in the list of declarations a file's top level may hold. A variant value such as `Colors.Red` or `Shape.Circle(1)` only builds data, so it would count as a comptime value when its contents do, once [Q-variant-access] and [Q-variant-fields] settle how it is written. `Option` and `Result` in `stdlib/stdlib.krb` ([Q-option-home], option (a)) are declarations only, so they fit. The samples here that use top-level statements are converted when [Q-samples] there is settled.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases. Tests for `match` on enums are here because that is where enums are used, and they need the [Pattern Matching Proposal].

##### NEW: tests/enums/enum_variants_equal.krb

```kirby
enum Colors {
  Red,
  Green,
  Blue,
}

@assert(Colors.Red == Colors.Red, "Enum variants equal themselves");
@assert(Colors.Red != Colors.Blue, "Different variants are not equal");

print "ok";
```

###### Expected Outcome

Prints `ok`. Exit code 0.

##### NEW: tests/enums/enum_impl_method.krb

```kirby
enum Colors {
  Red,
  Green,
}

impl Colors {
  pub fun isRed(self): bool = self == Colors.Red;
}

print Colors.Red.isRed();
print Colors.Green.isRed();
```

###### Expected Outcome

Prints `true` then `false`.

##### NEW: tests/enums/enum_trait_impl.krb

```kirby
enum Colors {
  Red,
  Green,
}

impl Display for Colors {
  fun toString(self): string = if (self == Colors.Red) "Red" else "Green";
}

print Colors.Red.toString();
```

###### Expected Outcome

Prints `Red`.

##### NEW: tests/enums/enum_unknown_variant.krb

```kirby
enum Colors {
  Red,
}

print Colors.Purple;
```

###### Expected Outcome

Compile error that `Colors` has no variant `Purple`. Exit code 65.

##### NEW: tests/enums/enum_different_types_not_comparable.krb

```kirby
enum A {
  X,
}

enum B {
  X,
}

print A.X == B.X;
```

###### Expected Outcome

Compile error because the two sides are different types. Exit code 65.

##### NEW: tests/enums/enum_variant_and_method_share_name.krb

```kirby
enum Colors {
  Red,
}

impl Colors {
  pub fun Red(): Self = Colors.Red;
}
```

###### Expected Outcome

Compile error that the name is already used. Exit code 65. See [Q-variant-access].

##### NEW: tests/enums/enum_declared_in_block.krb

```kirby
{
  enum Colors {
    Red,
  }
}
```

###### Expected Outcome

Compile error that `enum` is a declaration and can only appear at the top level. Exit code 65.

##### NEW: tests/enums/enum_keyword_reserved.krb

```kirby
var enum = 1;
```

###### Expected Outcome

Compile error. Exit code 65. Today this program compiles, so the change is documented in `docs/CHANGELOG.md`.

##### NEW: tests/enums/enum_variant_with_values.krb

```kirby
enum Value {
  Bool(bool),
  Number(f64),
}

let a = Value.Bool(true);
let b = Value.Number(1);

print "built";
```

###### Expected Outcome

Prints `built`. Part B.

##### NEW: tests/enums/enum_variant_wrong_value_type.krb

```kirby
enum Value {
  Number(f64),
}

let a = Value.Number("a");
```

###### Expected Outcome

Compile error that `f64` was expected and `string` was given. Exit code 65.

##### NEW: tests/enums/enum_struct_like_variant.krb

```kirby
enum Shape {
  Rect { width: f64, height: f64 },
}

let r = Shape.Rect { width: 1, height: 2 };

print "built";
```

###### Expected Outcome

Prints `built`. The exact way to build a struct-like variant is [Q-variant-fields].

##### NEW: tests/enums/enum_recursive.krb

```kirby
enum List {
  End,
  Item(f64, List),
}

let list = List.Item(1, List.Item(2, List.End));

print "built";
```

###### Expected Outcome

Prints `built`.

##### NEW: tests/enums/enum_match_payload.krb

```kirby
enum Value {
  Bool(bool),
  Number(f64),
}

fun describe(v: Value): string {
  match (v) {
    Value.Bool(b) => if (b) "yes" else "no",
    Value.Number(n) => @numberToString(n),
  }
}

print describe(Value.Bool(true));
print describe(Value.Number(3));
```

###### Expected Outcome

Prints `yes` then `3`. Needs the [Pattern Matching Proposal].

##### NEW: tests/enums/enum_match_non_exhaustive.krb

```kirby
enum Colors {
  Red,
  Green,
}

fun name(c: Colors): string {
  match (c) {
    Colors.Red => "Red",
  }
}
```

###### Expected Outcome

Compile error that `Colors.Green` is not handled. Exit code 65. Needs the [Pattern Matching Proposal].

##### NEW: tests/enums/enum_generic_option.krb

```kirby
enum Option[T] {
  None,
  Some(T),
}

let a: Option[f64] = Option.Some(1);

print match (a) {
  Option.Some(n) => n,
  Option.None => 0,
};
```

###### Expected Outcome

Prints `1.000000`. Needs the [Pattern Matching Proposal] and the [Generic Types Proposal].

##### NEW: tests/enums/enum_generic_wrong_type.krb

```kirby
enum Option[T] {
  None,
  Some(T),
}

let a: Option[string] = Option.Some(1);
```

###### Expected Outcome

Compile error that `string` was expected and `f64` was given. Exit code 65. Needs the [Generic Types Proposal].

## Questions

### **Q:** How is a variant written, and what else shares its name?

<!-- [Q-variant-access]: #q-how-is-a-variant-written-and-what-else-shares-its-name -->

**Status:** Open

The sketch writes `Colors.Red`. Kirby already uses `Type.name` for static methods (`Point.origin()`, `Struct.new(123)`), and has no `::` token, so the sketch's form needs no new syntax. But a variant and a static method then share one set of names. `impl Colors { pub fun Red(): Self }` would clash with the variant `Red`, and building a variant with values, `Shape.Circle(1)`, is written exactly like calling a static method.

Options: (a) One shared set of names. A variant and a static method with the same name is a compile error. Simple, and easy to explain. (b) Two sets, told apart by context. Nothing clashes, but reading the code gets harder. (c) A different form for variants, such as `Colors::Red`. Clear, but a new token, and unlike everything else in Kirby.

Whether a variant can be used without the enum's name is [Q-bare-variants].

### **Q:** What is an enum value at run time?

<!-- [Q-representation]: #q-what-is-an-enum-value-at-run-time -->

**Status:** Open

A value is a bool, `nil`, a number, or a pointer to an object. There is no room in a value for a variant name.

Options: (a) Plain variants are numbers, and only variants with values are objects. No memory is used and `==` already works on numbers. But at run time `Colors.Red` is just `0`, so `@typeof` cannot tell it from a count, and a method call on it can only be resolved from the type the checker knows, the way the [Primitive Impls Proposal] does for `f64`. Two representations must also be kept in step. (b) Every variant is an object. Plain variants are created once, when the enum is declared, and shared, so `==` works by identity. A variant with values is a new object each time it is built. One representation, and a link from the object to its enum lets the virtual machine find methods, as it does for struct instances. (c) A new object for every use, even plain variants. Simplest to describe, wasteful, and `==` would need to look inside.

(b) is the option that needs the fewest exceptions.

### **Q:** How does equality work on enums?

<!-- [Q-equality]: #q-how-does-equality-work-on-enums -->

**Status:** Open

The sketch asserts that `Colors.Red == Colors.Red` with no `impl Eq` anywhere. For structs, `==` needs an `Eq` impl to compile, and then compares identity when it runs, so two equal-looking structs are not equal (verified; see the [Generic Types Proposal] Part 1.4).

Options: (a) `==` works on any two values of one enum type with no `impl Eq`. Plain variants are equal if they are the same variant. Variants with values are equal if the variant is the same and each value is equal, item by item, as proposed for tuples ([Q-equality-tuples]). (b) The same as structs: an `impl Eq` is needed, and identity is used when it runs. Then two separately built `Value.Number(1)` are not equal. (c) Only plain enums can be compared until `Eq` can be generated by a macro.

(a) is closest to the sketch. Any change to how `==` works on objects comes from the [Generic Types Proposal] 10.4.

### **Q:** Can a program see or set the number of a variant?

<!-- [Q-discriminants]: #q-can-a-program-see-or-set-the-number-of-a-variant -->

**Status:** Open

The sketch's comments (`// 0`, `// 1`, `// 2`) suggest every variant has a number, from its position. Options: (a) No. The order has no meaning to a program, and the comments only explain the sketch. This keeps the order free to change, and makes it the simplest representation ([Q-representation]). (b) Yes, there is a way to read it, such as a native function. Then the order of variants can never change without breaking users, and `Result` would number `Err` before `Ok`, unlike Rust. (c) Yes, and a variant can be given its own number, like `Red = 5`. This is C-like, and it makes the enum a set of numbers with names, which does not fit variants that carry values.

(a) is the smaller commitment. It can become (b) or (c) later.

### **Q:** How are the values inside a variant declared, built, and read?

<!-- [Q-variant-fields]: #q-how-are-the-values-inside-a-variant-declared-built-and-read -->

**Status:** Open

The sketch has two shapes, and each raises questions:

- **Declaring.** A struct-like variant writes `name: Type,` with commas. A struct writes `pub var name: Type;` with semicolons, and marks each field `pub` or private. Should the values in a variant be public, and can they be assigned to? In Rust they are public and only change through a mutable binding. Here, options: (a) always public, and never assignable. Simple, and no `pub` or `var` to write. (b) The same as struct fields, with `pub` and `var`. More to write for something that is usually only read.
- **Building.** A tuple-like variant is built with a call, `Shape.Circle(1)`. That is the same call syntax the [Tuple Structs Proposal] asks about ([Q-constructor]), and the changelog says to deprecate calling structs. A struct-like variant would be built with braces, `Shape.Rect { width: 1, height: 2 }`. The parser only allows a struct name before `{` today.
- **Reading.** Only with a pattern. A `.0` is not offered, because the value might be a different variant.

The two proposals should agree on how a call builds a value.

### **Q:** Where do `Option` and `Result` live?

<!-- [Q-option-home]: #q-where-do-option-and-result-live -->

**Status:** Open

Options: (a) In `stdlib/stdlib.krb`, as ordinary Kirby enums. That file is empty today and is run before user code in every run mode. It is simple and visible. But it puts `Option` and `Result` in every program's set of global names (the [String Interpolation Proposal] raises the same worry for `StringBuilder`), and native functions written in C would have to build a value of an enum declared in Kirby. (b) Built into the compiler, the way `Display`, `Eq`, `Ord`, and `Default` are already built in. C code can then create them, so natives could return `Option`. More C to write, and more to keep in step with Kirby's own rules. (c) Start with (a) for use in Kirby code, and give the natives an `Option` later once a way to build one from C is decided.

### **Q:** How does `Option[T]` relate to `nil`?

<!-- [Q-option-nil]: #q-how-does-optiont-relate-to-nil -->

**Status:** Open

`nil` has the type `unit` today, so it cannot stand for a missing `f64` (verified: `let x: f64 = nil;` reports `Expected f64, got unit.`). That is why natives that may return nothing have no type. `??` picks the right side when the left is `nil`, and is accepted on typed values such as an `f64`.

Options: (a) Keep `nil` and `??` as they are. `Option[T]` is an ordinary enum next to them, and code chooses which to use. (b) Have `??` also work on `Option[T]`, choosing the right side for `None`. (c) Later, move "maybe no value" wholly to `Option[T]` and change what `nil` is.

This proposal only needs the answer to be (a). (b) and (c) are here so they are decided on purpose and not by accident.

### **Q:** How does an enum value print?

<!-- [Q-print]: #q-how-does-an-enum-value-print -->

**Status:** Open

A struct value prints as its name followed by `instance` (`S instance`, verified), and `print` does not use `Display` today. Options: (a) The same for enums, `Colors instance`. No work, and not much use. (b) The variant, `Colors.Red`, and for a variant with values, `Shape.Circle(1.000000)`. Friendlier and matches how a value is written.

### **Q:** Is `enum` a reserved word?

<!-- [Q-keyword]: #q-is-enum-a-reserved-word -->

**Status:** Open

`enum` is a valid variable name today (`var enum = 1;` works, verified). The choice is the same one the [Pattern Matching Proposal] has for `match` ([Q-keyword-match]): (a) reserve it, like every other keyword, and any program using `enum` as a name stops compiling. (b) Treat it as a keyword only where a declaration can start. Nothing breaks, but the parser has more to decide.

Both proposals should make the same choice.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Enum**: A type whose values are each exactly one of a fixed set of named variants.
- **Variant**: One of the named alternatives of an enum. `Red` is a variant of `Colors`.
- **Plain variant**: A variant that carries no values.
- **Tuple-like variant**: A variant that carries values by position, like `Circle(f64)`.
- **Struct-like variant**: A variant that carries values by name, like `Rect { width: f64, height: f64 }`.
- **Type parameter**: A placeholder for a type that is filled in where the enum is used, like `T` in `Option[T]`.
- **Compared by name**: Two types are the same only if they are the same declaration. Structs, and enums as proposed, work this way.
- **Heap object**: A value that lives in memory managed by Kirby's garbage collector, such as a string, array, or struct instance.
- **Static method**: A function in an `impl` block that has no `self` and is called on the type, like `Point.origin()`.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-bare-variants]: ../pattern-matching/PROPOSAL.md#q-are-variant-names-written-in-full-in-patterns
[Q-keyword-match]: ../pattern-matching/PROPOSAL.md#q-is-match-a-reserved-word
[Q-constructor]: ../tuple-structs/PROPOSAL.md#q-how-is-a-tuple-struct-constructed
[Q-equality-tuples]: ../tuples/PROPOSAL.md#q-how-does-tuple-equality-compare-items
[Q-syntax]: ../macros/PROPOSAL.md#q-what-form-of-syntax-value-do-macros-receive-and-return
[Q-samples]: ../top-level-declarations/PROPOSAL.md#q-when-do-code-samples-in-other-proposals-change

<!-- External -->

[Issue #39]: https://github.com/kirbylang/kirbylang/issues/39

<!-- Questions -->

[Q-variant-access]: #q-how-is-a-variant-written-and-what-else-shares-its-name
[Q-representation]: #q-what-is-an-enum-value-at-run-time
[Q-equality]: #q-how-does-equality-work-on-enums
[Q-discriminants]: #q-can-a-program-see-or-set-the-number-of-a-variant
[Q-variant-fields]: #q-how-are-the-values-inside-a-variant-declared-built-and-read
[Q-option-home]: #q-where-do-option-and-result-live
[Q-option-nil]: #q-how-does-optiont-relate-to-nil
[Q-print]: #q-how-does-an-enum-value-print
[Q-keyword]: #q-is-enum-a-reserved-word
