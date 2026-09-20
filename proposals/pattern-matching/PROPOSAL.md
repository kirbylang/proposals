---
status: Draft
created: 2026-09-20
from_commit: 662d98b
---

# Proposal: Pattern Matching

This proposal adds a `match` expression to Kirby. It picks the first arm whose pattern fits a value, and the value of the whole `match` is the value of that arm. It is heavily influenced by Rust. The only sketch of it is a short example in [Issue #39], the enums issue, so this skeleton is mostly a set of questions. It builds on the patterns from the [Destructuring Proposal].

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

Kirby chooses between alternatives with `if` and `else`, which is already an expression. Choosing between many values means a chain of comparisons, and there is no way at all to choose by the shape of a value:

```kirby
fun describe(n: f64): string =
  if (n == 0) "zero" else if (n == 1) "one" else "many";
```

That works today (verified against `from_commit`). It does not scale, it cannot take a value apart, and nothing checks that every case was handled.

[Issue #39] has the only sketch of `match`, inside an enum example that says the syntax does not exist yet:

```kirby
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
```

It also says "Destructuring needed for pattern matching also doesn't exist yet." That is the [Destructuring Proposal].

The sketch does not work today. Verified against a clean build of `from_commit`, `match x { 1 => print "one", }` reports `Error at 'match': Expect ';' after expression.` because `match` is an ordinary word.

Several things that already exist shape the design. Each was verified:

- **`if` is an expression, and its condition is in parentheses.** `if (cond) a else b` has a value, and its branches are expressions. `while (...)` and `for (...)` also use parentheses.
- **A `{` after an expression starts a struct literal.** The parser treats `{` as an infix operator of call strength (`[TOKEN_LEFT_BRACE] = {blockExpr, struct_, PREC_CALL}` in `src/parser.c`). So `var x = 1; x { y: 2 };` reports `Unknown struct 'x'`, and `self { x: 1 }` inside a method reports `Only a struct name can be initialized with '{'.` The sketch's `match self {` would read `self {` the same way if the value after `match` is parsed as an ordinary expression. See [Q-scrutinee].
- **`=>` is already a token,** `TOKEN_FAT_ARROW`, used in function types like `fun (f64) => f64`.
- **Some pattern symbols are not tokens.** A single `|` is a scanner error (`||` is the `or` operator) and `..` does not exist (a single `.` does). `@` at the start of a word means a native function, so `name @ pattern` from Rust would clash.
- **`match`, `enum`, and `_` are valid variable names today.** `var match = 2;` and `var _ = 3;` both work. No `.krb` file in the repository uses `match` or `_` as a name.
- **The type checker compares types by exact equality.** There is no subtype rule that would let two arms of different types meet at a common type.
- **`==` on numbers and strings compares by value.** `"a" + "b" == "ab"` is `true`. Objects such as arrays and structs compare by identity.

## Proposed Changes

### The `match` expression

```kirby
match (value) {
  pattern => expression,
  pattern => expression,
}
```

- The value is worked out once. The arms are tried from the top. The first arm whose pattern fits gives the value of the `match`.
- It is an expression, so it can be a function's last expression or the initializer of a `let`.
- Every arm's expression must have the same type, and that is the type of the `match`.
- Arms are separated by commas and a trailing comma is allowed, as in the sketch.
- The parentheses around the value are one option. See [Q-scrutinee]. More about arm syntax is [Q-arm-syntax].

```kirby
fun describe(n: f64): string {
  match (n) {
    0 => "zero",
    1 => "one",
    _ => "many",
  }
}
```

### Patterns

Patterns are the ones from the [Destructuring Proposal], plus patterns that can fail to fit. They can be added in stages, according to what each one depends on:

- **Needs nothing new.** A literal (`1`, `"a"`, `true`), a name that takes whatever value is there, and the placeholder `_`.
- **Needs the destructuring work.** Struct and array patterns, and tuple and tuple struct patterns once those exist, all of which can be nested.
- **Needs enums.** A variant, with or without patterns for the values it carries, like `Colors.Red` or `Shape.Circle(radius)`. See the [Enums Proposal]. Whether the enum name is needed in front of the variant is [Q-bare-variants].
- **Later candidates.** A condition on an arm (`pattern if condition`), several patterns for one arm, a range of values, and a name that also captures the whole value. See [Q-first-version].

### Checking that every case is handled

A `match` should have an arm for every possible value, or the program should not compile. A name or `_` catches everything not handled before it. Whether this is an error, and how it works for each type, is [Q-exhaustive].

### Goals and Non Goals

What this proposal covers:

- A `match` expression with arms of `pattern => expression`
- Literal, name, and placeholder patterns
- The destructuring patterns, and enum variant patterns, in `match` arms
- Type checking the patterns against the value, and the arms against each other
- Checking that every case is handled
- Compiling to the instructions Kirby already has where possible

The following is intentionally left out of scope for this proposal:

- `if let`, `while let`, and `let ... else`
- Arm conditions, several patterns per arm, ranges, and capturing patterns, until [Q-first-version] is answered
- Making matching fast, for example by building a decision tree instead of trying arms in order
- Matching on the type of a value at run time, like `@instanceOf`

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Scanner

- A `match` keyword, `TOKEN_MATCH`, added in `identifierType` in `src/scanner.c` (see [Q-keyword]).
- `_` and any symbols chosen in [Q-first-version], such as a single `|` or `..`.

#### Part 2: Parser and AST

- A `NODE_MATCH` node holding the value and a list of arms, each a pattern and an expression. It is read as a prefix expression, the same way `if` is.
- The pattern nodes come from [Destructuring Proposal] Part 1, plus new kinds for literals, the placeholder, and variants.

#### Part 3: Type checker

- Infer the type of the value. Check each pattern against it. A literal must have the value's type, a name takes the value's type, and a placeholder takes anything.
- Check each arm's expression, with the pattern's names in scope, and require one type across all arms. When the `match` has an expected type, whether the arms are checked against it is [Q-arms-expected].
- Check that every case is handled ([Q-exhaustive]).

#### Part 4: Compiler

- Keep the value in a hidden temporary, shared with the destructuring work. Compile each arm as a test followed by a jump: a literal test is `OP_EQUAL` and `OP_JUMP_IF_FALSE`, and the names in a pattern become locals that only exist inside the arm. The result is left on the stack like an `if` expression.
- Patterns for enum variants and for values inside them need new instructions. Those come with the [Enums Proposal].
- If no arm fits, the program stops with a run time error. The checker should make that impossible for a well formed `match`.

#### Part 5: Docs and tooling

- `match` and `=>` in the VS Code grammar (`vsc/syntaxes/kirby.tmLanguage.json`), and a hover for `match`.
- Names bound in an arm are local variables for the [Tooling Data Proposal] and the [Debugger Proposal].

## Impacts

### Existing Syntax Or Behavior

- `match` becomes a reserved word, and `_` may too ([Q-keyword], [Q-wildcard]). Both are valid variable names today, so a program that uses either as a name would stop compiling. Nothing in this repository does.
- `=>` is already a token, so no change to the scanner is needed for it.
- `if`, `else`, and the comparison operators are unchanged, and `if` chains keep working.
- A `{` after an expression is still a struct literal. Anything else that follows a value with `{` must be made unambiguous ([Q-scrutinee]).

### Related Proposals

- [Destructuring Proposal] — this proposal depends on it. The patterns and their AST nodes are shared. It also settles the meaning of `_` for patterns ([Q-wildcard]). Its question about array patterns that do not fit ([Q-array-mismatch]) has a natural answer here: in a `match` an arm that does not fit just falls through to the next.
- [Enums Proposal] — the main reason for this proposal. An enum with values inside a variant cannot be used without it. Fieldless enums can be used with `==` alone.
- [Tuples Proposal] and [Tuple Structs Proposal] — their patterns depend on them.
- [Unimplemented Proposal] — `???` type-checks only where an expected type exists, and `if` branches are not such a place. Whether arms are one is [Q-arms-expected].
- [Generic Types Proposal] — matching an `Option[T]` needs generic enums. A literal pattern compares with `==`, so it depends on how `==` behaves (10.4 there).
- [Sized Number Types Proposal] — a literal pattern like `1` must take the kind of number being matched, following that proposal's literal rules.
- [Macros Proposal] — macros are described as ordinary Kirby code working on syntax as data. `match` and enums would be the natural way to write that code.
- [Tooling Data Proposal] and [Debugger Proposal] — each arm introduces local variables.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases. Tests for enum patterns are in the [Enums Proposal].

##### NEW: tests/match/match_literal_number.krb

```kirby
fun describe(n: f64): string {
  match (n) {
    0 => "zero",
    1 => "one",
    _ => "many",
  }
}

print describe(0);
print describe(1);
print describe(7);
```

###### Expected Outcome

Prints `zero`, `one`, `many`. Exit code 0.

##### NEW: tests/match/match_literal_string.krb

```kirby
let s = "a" + "b";

let result = match (s) {
  "ab" => 1,
  _ => 0,
};

print result;
```

###### Expected Outcome

Prints `1.000000`. The string was built at run time, so this checks that strings are compared by value.

##### NEW: tests/match/match_bool.krb

```kirby
let flag = true;

print match (flag) {
  true => "yes",
  false => "no",
};
```

###### Expected Outcome

Prints `yes`. Compiles without `_`, because `true` and `false` are every value a `bool` can have ([Q-exhaustive]).

##### NEW: tests/match/match_name_binding.krb

```kirby
print match (5) {
  0 => 0,
  n => n + 1,
};
```

###### Expected Outcome

Prints `6.000000`. The name `n` takes the value.

##### NEW: tests/match/match_first_arm_wins.krb

```kirby
print match (1) {
  1 => "first",
  _ => "second",
};
```

###### Expected Outcome

Prints `first`.

##### NEW: tests/match/match_non_exhaustive.krb

```kirby
print match (1) {
  0 => "zero",
};
```

###### Expected Outcome

Compile error that not every case is handled. Exit code 65. See [Q-exhaustive].

##### NEW: tests/match/match_arm_type_mismatch.krb

```kirby
print match (1) {
  0 => "zero",
  _ => 1,
};
```

###### Expected Outcome

Compile error that the arms have different types. Exit code 65.

##### NEW: tests/match/match_pattern_type_mismatch.krb

```kirby
print match (1) {
  "a" => 1,
  _ => 2,
};
```

###### Expected Outcome

Compile error that a `string` pattern can not match an `f64`. Exit code 65.

##### NEW: tests/match/match_keyword_reserved.krb

```kirby
var match = 1;
```

###### Expected Outcome

Compile error. Exit code 65. Today this program compiles, so the change is documented in `docs/CHANGELOG.md`.

##### NEW: tests/match/match_destructure_array.krb

```kirby
let values = [1, 2];

print match (values) {
  [a, b] => a + b,
  _ => 0,
};
```

###### Expected Outcome

Prints `3.000000`. Needs the [Destructuring Proposal]. An array with a different length takes the `_` arm.

##### NEW: tests/match/match_destructure_struct.krb

```kirby
struct Box {
  pub var value: f64;
}

let box = Box { value: 4 };

print match (box) {
  Box { value } => value,
};
```

###### Expected Outcome

Prints `4.000000`. Needs the [Destructuring Proposal]. Compiles without `_`, because a struct pattern that lists every field always fits.

## Questions

### **Q:** Does the value after `match` need parentheses?

<!-- [Q-scrutinee]: #q-does-the-value-after-match-need-parentheses -->

**Status:** Open

The sketch writes `match self {`. Read as an ordinary expression, `self {` is the start of a struct literal, and fails today with `Only a struct name can be initialized with '{'.` (verified with `self { x: 1 }` in a method). Rust has the same trap and avoids it by not allowing struct literals in that position.

Options: (a) Require parentheses, `match (self) {`, like `if (...)`, `while (...)`, and `for (...)`. It needs no special rule in the parser and is consistent with the rest of Kirby. (b) No parentheses, and the parser does not treat `{` as a struct literal while reading the value. It matches the sketch and Rust, but the parser needs a mode for it, and `match Box { value: 1 } {` would not work without wrapping the literal in parentheses. (c) No parentheses, and the value can only be a name or a call. Simple, and limiting.

The examples in this proposal use (a).

### **Q:** Are variant names written in full in patterns?

<!-- [Q-bare-variants]: #q-are-variant-names-written-in-full-in-patterns -->

**Status:** Open

The sketch writes `Red => "Red"`, with the variant name on its own. But a name on its own in a pattern already means "take the value and call it this", as in `n => n + 1`. `Red => "Red"` would then match everything, and `Red` would be a new variable. Rust has the same trap and requires the enum name (`Colors::Red`) unless the variants are brought into scope.

Options: (a) Always write the enum name, `Colors.Red => "Red"`. It is clear and matches how the value is written elsewhere. (b) Let the checker resolve a bare name as a variant when the value being matched is an enum with a variant of that name. It is short, but a typo in a variant name silently becomes a catch-all variable. (c) A separate way to bring variants into scope, which depends on the [Modules Proposal].

The [Enums Proposal] has a related question about how variants are written when a value is built ([Q-variant-access]).

### **Q:** What does it mean for a `match` to handle every case?

<!-- [Q-exhaustive]: #q-what-does-it-mean-for-a-match-to-handle-every-case -->

**Status:** Open

Options for a `match` that does not handle everything: (a) It is a compile error, listing what is missing. (b) It compiles and stops with a run time error if the value does not fit any arm. (c) A compile error for some types and a run time error for others.

How each type counts: `bool` has two values, so `true` and `false` together handle everything. An enum has its variants. An `f64` and a `string` have too many values to list, so they always need a name or `_` at the end. An array pattern has a fixed length and arrays vary, so a list of array patterns needs a catch-all. A struct or tuple pattern that lists every field always fits.

(a) is the reason to have `match` rather than a chain of `if`. It needs an analysis in the type checker that (b) does not. It is also the part of this proposal that grows most as more kinds of pattern are added.

### **Q:** Which patterns are in the first version?

<!-- [Q-first-version]: #q-which-patterns-are-in-the-first-version -->

**Status:** Open

Rust has several more patterns than this proposal lists. Each needs something Kirby does not have today:

- **A condition on an arm,** `pattern if condition => ...`. No new symbol, since `if` already exists. The checker would need to leave conditions out of its analysis of handled cases.
- **Several patterns for one arm,** like `1 | 2 => ...`. A single `|` is a scanner error today, and `||` is `or`. Kirby's logical `or` is a word, so `1 or 2 => ...` would need no new token, but it reads like the logical operator.
- **Ranges,** like `1..5`. `..` is not a token.
- **A name that also captures the whole value,** Rust's `name @ pattern`. `@` at the start of a word already means a native function, so this clashes.

Options: (a) Only what the sections above list, and add the rest later as their own proposals. (b) Add the arm condition now, since it needs nothing new, and leave the rest. (c) Decide each of these here.

### **Q:** What are the details of arm syntax?

<!-- [Q-arm-syntax]: #q-what-are-the-details-of-arm-syntax -->

**Status:** Open

The sketch has commas between arms, no braces around arm expressions, and a trailing comma. Still to decide:

- Can an arm's expression be a block, `pattern => { ... }`? A block is already an expression in Kirby. Does an arm with a block still need a comma after it?
- Can a `match` be used as a statement? Kirby has separate parsing for an `if` used as an expression (`ifExpr`) and as a statement (`ifStatement`), so a `match` written on its own line may need the same, including whether a `;` follows it.
- Does a name inside a pattern shadow a variable of the same name outside it? For example `n => n + 1` when `n` is already declared. Most likely yes, for the length of the arm.

### **Q:** Are arms checked against the type the `match` is expected to have?

<!-- [Q-arms-expected]: #q-are-arms-checked-against-the-type-the-match-is-expected-to-have -->

**Status:** Open

Kirby's checker has two modes: inferring a type with no context, and checking against an expected type. The [Unimplemented Proposal] says `???` type-checks only in the second mode, and lists the branches of an `if` as a place where it is a compile error, because those branches are inferred, not checked against an expectation.

Options: (a) Infer every arm and require them to be equal, the same as `if`. It is simpler, but `???` is an error in an arm, which is exactly where a stub for an unfinished case is wanted. (b) When the `match` has an expected type, such as a function's declared return type, check each arm against it. `???` then works in an arm. (b) would raise the same question for `if` too.

### **Q:** Is `match` a reserved word?

<!-- [Q-keyword]: #q-is-match-a-reserved-word -->

**Status:** Open

`match` is a valid variable name today (`var match = 2;` works, verified), and the same is true of `enum`, which the [Enums Proposal] would reserve.

Options: (a) A reserved word, like every other keyword in Kirby's scanner. It is simple, and a program that uses `match` as a name stops compiling. (b) Only a keyword where a statement or expression is expected to start, so `match` can still be a variable name elsewhere. Nothing breaks, but the parser has more to decide, and `match (x)` could be a call of a function named `match`.

The changelog can record whichever is chosen. The same choice for `_` is [Q-wildcard].

## Glossary

These are both technical and non-technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Pattern matching**: Choosing what to do based on the shape and contents of a value.
- **Arm**: One line of a `match`, made of a pattern, `=>`, and an expression.
- **Pattern**: A description of the shape of a value, written with names where the pieces should go. See the [Destructuring Proposal].
- **Fit**: A pattern fits a value when the value has the shape the pattern describes.
- **Placeholder**: The pattern `_`, which fits any value and does not name it.
- **Handling every case**: A `match` handles every case when every possible value fits at least one arm. Also called being exhaustive.
- **Variant**: One of the named alternatives of an enum. See the [Enums Proposal].
- **Value being matched**: The expression right after `match`.
- **Hidden temporary**: A variable the compiler creates to hold the value being matched. The user never writes or sees it.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Unimplemented Proposal]: ../unimplemented/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Sized Number Types Proposal]: ../sized-number-types/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-wildcard]: ../destructuring/PROPOSAL.md#q-is-the-underscore-a-placeholder-in-patterns
[Q-array-mismatch]: ../destructuring/PROPOSAL.md#q-what-happens-when-an-array-pattern-does-not-fit
[Q-variant-access]: ../enums/PROPOSAL.md#q-how-is-a-variant-written-and-what-else-shares-its-name

<!-- External -->

[Issue #39]: https://github.com/kirbylang/kirbylang/issues/39

<!-- Questions -->

[Q-scrutinee]: #q-does-the-value-after-match-need-parentheses
[Q-bare-variants]: #q-are-variant-names-written-in-full-in-patterns
[Q-exhaustive]: #q-what-does-it-mean-for-a-match-to-handle-every-case
[Q-first-version]: #q-which-patterns-are-in-the-first-version
[Q-arm-syntax]: #q-what-are-the-details-of-arm-syntax
[Q-arms-expected]: #q-are-arms-checked-against-the-type-the-match-is-expected-to-have
[Q-keyword]: #q-is-match-a-reserved-word
