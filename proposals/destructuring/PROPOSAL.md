---
status: Draft
created: 2026-09-20
from_commit: 662d98b
---

# Proposal: Destructuring

This proposal adds destructuring to Kirby: taking a struct or an array apart into several new variables in a single `let` or `var` declaration, like `let Box { value } = box;` and `let [a, b, c] = values;`. It also defines the pattern syntax that the [Pattern Matching Proposal] builds on. This is a skeleton built from [Issue #86], which is a rough sketch, so the details are expected to change.

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

A `let` or `var` declaration introduces exactly one name today. Reading several fields or items out of a value takes one statement each:

```kirby
struct Box {
  pub var value: f64;
}

let box = Box { value: 123 };
let value = box.value;

let values = [1, 2, 3];
let a = values[0];
let b = values[1];
let c = values[2];
```

[Issue #86] proposes taking the value apart in one declaration:

```kirby
struct Box {
  pub var value: f64;
}

let box = Box { value: 123 };

let Box { value } = box;

print value; // 123.000000

let values = [1, 2, 3];
let [a, b, c] = values;

let [a, b, c, d] = values; // Error: TODO: Error message here
```

Neither form works today. Verified against a clean build of `from_commit`:

```
let Box { value } = box;    // Error at 'Box': 'let' binding requires an initializer.
let [a, b, c] = values;     // Error at '[': Expect variable name.
```

Several things that already exist shape the design. Each was verified:

- **A declaration holds one name.** `varDeclaration` in `src/parser.c` reads one identifier, and the AST node for a declaration (`NODE_VAR_DECL`) holds one name token.
- **Field shorthand is not supported for struct literals.** `Box { value }` reports `Expect ':' after field name.` The sketch uses that shorthand in a pattern. See [Q-shorthand].
- **Field privacy is per struct and checked at run time.** A private field can be used by the struct's own `impl` methods, and reading one from outside reports `Field 'balance' is private to 'Account'.` at run time. A pattern that names a private field has to follow the same rule.
- **An array's length is not part of its type.** The checker knows an array's item type (`let s: string = values[0];` reports `Expected string, got f64.`) but not how many items it has. So `let [a, b, c, d] = values;` can only be found wrong when the program runs. An out of range index reports `Array index out of bounds.` at run time (exit code 70).
- **The keyword decides mutability.** `let a = 1; a = 2;` reports `Cannot assign to immutable binding`. A `let` pattern should make every name in it immutable, and `var` should make them all assignable.
- **Destructuring is needed for pattern matching.** [Issue #39] says as much. The pattern syntax defined here is what a `match` arm would use.

## Proposed Changes

### Where patterns can appear

A pattern can be used in place of the name in a `let` or `var` declaration. It must have an initializer, like `let` does today. `let` makes every name in the pattern immutable and `var` makes every name assignable.

Where a type annotation goes, like `: Array`, is [Q-annotation].

### Struct patterns

```kirby
let Box { value } = box;
```

This declares `value` and sets it to `box.value`. The name in front of the braces must be the struct's type. Each field named must exist and be visible where the pattern is written: public, or private inside the struct's own `impl` methods. Each new variable has the type of its field. Renaming, shorthand, and leaving fields out are [Q-rename], [Q-shorthand], and [Q-partial].

### Array patterns

```kirby
let [a, b, c] = values;
```

This declares `a`, `b`, and `c` from the first three items in order. Each has the array's item type. If the array does not have the items the pattern asks for, the program stops with a run time error. What exactly counts as not fitting, and the error message, are [Q-array-mismatch].

### Tuple and tuple struct patterns

`let (a, b) = pair;` and `let Value(v) = val;` are the same idea for tuples and tuple structs. They are listed so the design leaves room for them. They can only be built once the [Tuples Proposal] and the [Tuple Structs Proposal] exist.

### Nesting and ignoring

A pattern can contain other patterns, and a placeholder could stand in for an item that is not needed. Both are candidates. See [Q-wildcard].

### Goals and Non Goals

What this proposal covers:

- Struct patterns and array patterns in `let` and `var` declarations
- Typed, correctly scoped variables from each pattern
- Respecting field privacy
- A clear run time error when an array pattern does not fit
- A pattern syntax that the [Pattern Matching Proposal] can reuse

The following is intentionally left out of scope for this proposal:

- `match`. See the [Pattern Matching Proposal].
- Patterns in function or lambda parameters, like `fun f(Box { value }: Box)`
- Patterns in `for` loops
- Assigning to variables that already exist, like `[a, b] = [b, a]`
- Patterns that can fail in ways other than an array's length, such as choosing an enum variant. Those belong to pattern matching.
- Adding field shorthand to struct literals. See [Q-shorthand].

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Pattern nodes

- New AST node kinds for patterns: a single name, a struct pattern (a struct name and field and pattern pairs), and an array pattern (a list of patterns). Tuple, tuple struct, and placeholder patterns are added later.
- `NODE_VAR_DECL` holds one name token. A declaration with a pattern needs either a pattern in place of the name, or a new declaration node.
- The [Pattern Matching Proposal] reuses these nodes, so the two should be designed together.

#### Part 2: Parser

- `varDeclaration` in `src/parser.c` reads one identifier with `consumeDeclarationIdentifier`. It should look at the token after `let` or `var` first. An identifier followed by `{` starts a struct pattern, and `[` starts an array pattern.
- A `{` after an identifier is also how a struct literal starts. That is not a problem here, because a pattern is only read straight after `let` or `var`.
- A pattern must have an initializer. `var [a, b];` is an error.

#### Part 3: Type checker

- The initializer is inferred first. A struct pattern must match the struct's type, each field must exist and be visible, and each new variable gets the field's type. An array pattern needs an array type and each new variable gets its item type.
- The new variables go into the scope like any other declaration, so a repeated name is the usual "already declared" error.
- `src/definite_assignment.c` must treat every name from a pattern as assigned.

#### Part 4: Compiler

- The initializer is evaluated once and kept in a hidden temporary. Each new variable is then read from it, with the existing opcodes `OP_GET_PROPERTY` for a struct field and `OP_GET_INDEX` for an array item.
- At the top level each name becomes a global (`OP_DEFINE_GLOBAL`). Inside a function or block each name takes a stack slot, and the hidden temporary takes one too. The bookkeeping that tracks stack slots needs care here. There is an existing test for locals, `tests/locals_bookkeeping.krb`.
- The array length check needs either a new opcode or a call to `@len`. It should run before any name is defined, so a pattern that does not fit does not leave some of its names defined.

#### Part 5: Docs and tooling

- `docs/TYPES.md` or a new syntax page, and `docs/CHANGELOG.md`.
- Each new variable needs its own source location, and the hidden temporary must not show up as a variable a user wrote. See [Tooling Data Proposal].

## Impacts

### Existing Syntax Or Behavior

- `let Name {` and `let [` are compile errors today, so no valid program changes meaning.
- If `_` becomes a placeholder inside patterns, that touches an existing behavior. `_` is a valid variable name today: `var _ = 3;` works (verified). See [Q-wildcard].
- Struct literals still do not accept field shorthand. Only patterns would.

### Related Proposals

- [Tuples Proposal] — tuple patterns depend on it.
- [Tuple Structs Proposal] — tuple struct patterns depend on it.
- [Pattern Matching Proposal] — depends on this proposal. It reuses the pattern nodes and adds patterns that can fail, such as literals and enum variants.
- [Enums Proposal] — an enum variant can not be taken apart by `let`, because the value might be a different variant. That is done with `match`.
- [Tooling Data Proposal] — records the names of local variables and the range of code each is alive for. A pattern declares several names from one statement, plus a hidden temporary that has no name.
- [Debugger Proposal] — its variables view lists locals, so the hidden temporary should not appear there.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

##### NEW: tests/destructuring/destructure_struct.krb

```kirby
struct Box {
  pub var value: f64;
}

let box = Box { value: 123 };

let Box { value } = box;

print value;
```

###### Expected Outcome

Prints `123.000000`. Exit code 0.

##### NEW: tests/destructuring/destructure_array.krb

```kirby
let values = [1, 2, 3];
let [a, b, c] = values;

print a;
print b;
print c;
```

###### Expected Outcome

Prints `1.000000`, `2.000000`, `3.000000`.

##### NEW: tests/destructuring/destructure_var_assignable.krb

```kirby
var [a, b] = [1, 2];

a = 5;

print a;
```

###### Expected Outcome

Prints `5.000000`.

##### NEW: tests/destructuring/destructure_let_immutable.krb

```kirby
let [a, b] = [1, 2];

a = 5;
```

###### Expected Outcome

Compile error `Cannot assign to immutable binding`. Exit code 65.

##### NEW: tests/destructuring/destructure_array_too_short.krb

```kirby
let values = [1, 2, 3];
let [a, b, c, d] = values;
```

###### Expected Outcome

Run time error. Exit code 70. The message is to be decided ([Q-array-mismatch]).

##### NEW: tests/destructuring/destructure_initializer_evaluated_once.krb

```kirby
var calls = 0;

fun make(): Array {
  calls = calls + 1;
  [1, 2]
}

let [a, b] = make();

print calls;
```

###### Expected Outcome

Prints `1.000000`.

##### NEW: tests/destructuring/destructure_struct_private_field.krb

```kirby
struct Account {
  var balance: f64;
}

let account = Account { balance: 1 };

let Account { balance } = account;
```

###### Expected Outcome

An error saying the field is private. Whether it is found at compile time or run time is to be decided, and should match how a private `account.balance` read is reported.

##### NEW: tests/destructuring/destructure_struct_unknown_field.krb

```kirby
struct Box {
  pub var value: f64;
}

let box = Box { value: 1 };

let Box { missing } = box;
```

###### Expected Outcome

Compile error that `Box` has no field `missing`. Exit code 65.

##### NEW: tests/destructuring/destructure_struct_wrong_type.krb

```kirby
struct Box {
  pub var value: f64;
}

struct Other {
  pub var value: f64;
}

let box = Box { value: 1 };

let Other { value } = box;
```

###### Expected Outcome

Compile error that `Other` was expected and `Box` was given. Exit code 65.

##### NEW: tests/destructuring/destructure_without_initializer.krb

```kirby
var [a, b];
```

###### Expected Outcome

Compile error. Exit code 65. Today this reports `Expect variable name.`, and the message may improve.

## Questions

### **Q:** Should struct patterns allow field shorthand when struct literals do not?

<!-- [Q-shorthand]: #q-should-struct-patterns-allow-field-shorthand-when-struct-literals-do-not -->

**Status:** Open

The sketch writes `let Box { value } = box;`, which uses shorthand: `value` means `value: value`. Building the same struct today needs `Box { value: value }`, because `Box { value }` reports `Expect ':' after field name.`

Options: (a) Allow shorthand in patterns only, as sketched. A pattern and the literal it takes apart then look different. (b) Allow shorthand in both. Struct literal shorthand is useful on its own, and would be its own small proposal. (c) Require `value: value` in patterns until (b) exists, which is repetitive.

### **Q:** How is a field bound to a different name?

<!-- [Q-rename]: #q-how-is-a-field-bound-to-a-different-name -->

**Status:** Open

The sketch has no way to name a variable differently from its field, which matters when the field name is already taken. Options: (a) `let Box { value: v } = box;`, which mirrors a struct literal `Box { value: 123 }` where the position after the colon is now a pattern. This is also the form nested patterns would use, like `let Outer { inner: Box { value } } = outer;`. (b) Some other keyword, like `value as v`. (a) needs no new keyword and matches the literal.

### **Q:** Must a pattern list every field and every item?

<!-- [Q-partial]: #q-must-a-pattern-list-every-field-and-every-item -->

**Status:** Open

Options: (a) Yes, always. A struct pattern names every field and an array pattern gives every item. This is strict but easy to explain. (b) Struct patterns may leave fields out, and array patterns may take the first few items. Convenient, but a struct pattern that quietly ignores a field that is added later is easy to miss. (c) Everything must be listed unless the pattern ends with a marker for "and the rest", such as `..`. This is the Rust approach. `..` is not a token in Kirby today (a single `.` is), so this needs a scanner change.

Only the struct case is in the sketch, where the struct has one field.

### **Q:** What happens when an array pattern does not fit?

<!-- [Q-array-mismatch]: #q-what-happens-when-an-array-pattern-does-not-fit -->

**Status:** Open

The length of an array is not in its type, so the checker cannot see a mismatch. The sketch ends with an error and a TODO for its message. Two separate points:

- **Which lengths fit.** Options: (a) The array must have exactly as many items as the pattern. `[a, b]` fails on a three item array. (b) The array must have at least as many items. Extra items are ignored, which quietly drops data.
- **Whether `let` should accept a pattern that can fail at all.** Options: (a) Yes, with a run time error, as sketched. The error could reuse the existing `Array index out of bounds.` or be its own message that names the expected and actual lengths. (b) No. `let` only accepts patterns that always fit, so array patterns would only be usable in `match`, where a failed pattern just tries the next arm. (c) Allow it and add a way to say what to do on failure, like `let [a, b] = values else { ... }`.

(a) with exact length is the smallest change. (b) and (c) are the choices to reach for if run time errors from `let` are not wanted.

### **Q:** Is the underscore a placeholder in patterns?

<!-- [Q-wildcard]: #q-is-the-underscore-a-placeholder-in-patterns -->

**Status:** Open

An item that is not needed, like the second in `let [a, _, c] = values;`, needs a name that says "ignore this". `_` is the usual choice. But `_` is a valid variable name in Kirby today (`var _ = 3; print _;` prints `3.000000`, and no `.krb` file in the repository uses it).

Options: (a) Reserve `_` so it can never be a variable name. Simple, but it breaks any program that uses `_`. (b) `_` only means "ignore" inside patterns, and stays a normal name elsewhere. Nothing breaks, but `let _ = 1;` then means something different in each place. (c) No placeholder. Every item gets a real name, and repeated names are an "already declared" error.

The [Pattern Matching Proposal] needs the same answer for a catch-all arm.

### **Q:** Where does a type annotation go?

<!-- [Q-annotation]: #q-where-does-a-type-annotation-go -->

**Status:** Open

A declaration can have a type annotation after the name, like `let count: f64 = 1;`. With a pattern there are several names, and no single name to put it after. Options: (a) After the whole pattern, describing the initializer's type, like `let [a, b]: Array = values;`. (b) Not allowed on patterns. The types come from the initializer, and each name can be checked when it is used.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Destructuring**: Taking a value apart into its pieces and giving each piece a name, in one step.
- **Pattern**: A description of the shape of a value, written with names where the pieces should go. `Box { value }` and `[a, b, c]` are patterns.
- **Struct pattern**: A pattern that takes a struct apart by field.
- **Array pattern**: A pattern that takes an array apart by position.
- **Shorthand**: Writing `value` to mean `value: value`.
- **Initializer**: The expression after `=` in a declaration.
- **Hidden temporary**: A variable the compiler creates to hold the initializer while its pieces are read. The user never writes or sees it.
- **Can fail**: A pattern can fail if some values of the right type do not fit it. An array pattern can fail because arrays have different lengths.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Tuples Proposal]: ../tuples/PROPOSAL.md
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md

<!-- External -->

[Issue #86]: https://github.com/kirbylang/kirbylang/issues/86
[Issue #39]: https://github.com/kirbylang/kirbylang/issues/39

<!-- Questions -->

[Q-shorthand]: #q-should-struct-patterns-allow-field-shorthand-when-struct-literals-do-not
[Q-rename]: #q-how-is-a-field-bound-to-a-different-name
[Q-partial]: #q-must-a-pattern-list-every-field-and-every-item
[Q-array-mismatch]: #q-what-happens-when-an-array-pattern-does-not-fit
[Q-wildcard]: #q-is-the-underscore-a-placeholder-in-patterns
[Q-annotation]: #q-where-does-a-type-annotation-go
