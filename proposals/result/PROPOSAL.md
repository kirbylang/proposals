---
status: Draft
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: Error Handling with Result

This proposal gives Kirby programs a way to deal with failure without stopping. Today every error ends the program. This proposal uses `Result[T, E]`, an enum that is either `Ok(value)` or `Err(error)`, for operations that can fail because of something outside the program, such as text that is not a number or a file that is not there. It adds a `?` operator that passes an error up to the caller in one character. The enum itself is sketched in the [Enums Proposal]. This proposal is about how it is used.

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

A failure in Kirby stops the program. Each of these was verified against a build of `from_commit`:

```kirby
@println(@parseInt("12x"));
// function @parseInt expects argument 1 to be a whole number but got "12x".
// [line 1] in script          (exit code 70)

@println(@readFileToString("/nonexistent"));
// File does not exist: '/nonexistent'   (exit code 70)

@println(@getenv("NOPE_NOT_SET"));
// Environment variable not found: 'NOPE_NOT_SET'   (exit code 70)
```

In C each of these natives reports the error and exits, the same way `strSliceNative` does:

```c
runtimeError(vm, "function @strSlice expects argument 2 to be less than or "
                 "equal to argument 3.");
exit(EXIT_CODE_RUNTIME_ERR);
```

There is nothing a program can do to ask "is this a number?" first, except to check every character itself. A parser or a program that reads user input can not report "line 3: expected a comma" and carry on. `@panic(message)` and `@assert(condition, message)` are the only error tools, and both stop the program.

A program can build its own result with a struct. This works today:

```kirby
struct R { pub var ok: bool; pub var value: string; pub var error: string; }
fun f(): R { R { ok: false, value: "", error: "bad" } }
@println(f().error);
```

It needs a new struct for each value type, and nothing makes the caller look at `ok` before reading `value`. `Result[T, E]` is sketched in [Issue #39] and in the [Enums Proposal] (Part C, with `Ok` and `Err` as variants) and named by the [Generic Types Proposal] (10.9). No proposal says which natives return one, how an error travels up through several calls, or what the error is.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- A rule for which failures are a `Result` and which stop the program
- Which natives return a `Result` or an `Option`
- A `?` operator that returns an `Err` to the caller early
- What the error type is for natives and for user code

The following is intentionally left out of scope for this proposal:

- Defining `Result`, `Option` and enum syntax. See the [Enums Proposal].
- Generic type syntax. See the [Generic Types Proposal].
- Exceptions: a `throw` and a `catch` that cross function calls without the signature showing it
- Recovering from `@panic`, `@assert`, array bounds errors, integer overflow and stack overflow
- Stack traces for errors
- Converting one error type to another, beyond what [Q-try-mismatch] decides

### Part A: One rule for what is a `Result`

A failure should be a `Result` when the program can not know in advance that it will happen, because it depends on the outside world: text typed by a user, the contents of a file, whether a file exists, a variable in the environment, bytes that must be valid text. A failure should stop the program when it is a mistake in the program itself: a position past the end of a string, an index outside an array, an integer that overflows, a failed `@assert`.

This is the rule Rust uses, and it keeps `Result` out of every array read.

### Part B: Natives that can fail

Under the rule, these natives change. Signatures are written in the style of `src/native_signatures.c`:

| Native                      | Today                          | Proposed                                    |
| --------------------------- | ------------------------------ | ------------------------------------------- |
| `@parseInt`                 | `fun (string) => i64`          | `fun (string) => Result[i64, string]`       |
| `@parseFloat`               | `fun (string) => f64`          | `fun (string) => Result[f64, string]`       |
| `@readFileToString`         | `fun (string) => string`       | `fun (string) => Result[string, string]`    |
| `@writeStringToFile`        | `fun (string, string) => unit` | `fun (string, string) => Result[unit, string]` |
| `@getenv`                   | `fun (string) => string`       | `fun (string) => Option[string]`            |
| `@strFromBytes`, `@strFromCode` | see the [Strings Proposal] | `Result[string, string]`                    |

- `Err` holds the message the native prints today, so no information is lost. What type that is, for natives and for user code, is [Q-error-type]. This table assumes `string`. `@parseInt("12x")` gives `Err("expected a whole number but got \"12x\"")`.
- `@getenv` returns an `Option` because a missing variable is a normal answer and not an error.
- `@argv`, `@prompt` and `@stdin` are already waiting for `Option[T]` (`docs/TYPES.md`) and are covered by the [Enums Proposal]. `@strIndexOf` returns `-1` when there is no match and does not need one.

How this change reaches existing programs is [Q-native-migration].

### Part C: The `?` operator

`?` after an expression of type `Result[T, E]` gives the `T` when it is `Ok`, and returns the `Err` from the function it is in when it is not:

```kirby
fun readNumber(path: string): Result[i64, string] {
  let text = @readFileToString(path)?;
  let number = @parseInt(@strTrim(text))?;

  Ok(number * 2)
}
```

This means the same as:

```kirby
fun readNumber(path: string): Result[i64, string] {
  let text = match (@readFileToString(path)) {
    Ok(value) => value,
    Err(error) => return Err(error),
  };
  ...
}
```

- The function that contains `?` must return a `Result` with the same error type `E` ([Q-try-mismatch]). Anything else is a compile error that names both types.
- `?` has the same precedence as a call or a property read, so `a.b()?.c()?` reads left to right.
- In a lambda, `?` returns from the lambda.
- The scanner today makes a lone `?` an error token (`case '?'` in `src/scanner.c` returns `TOKEN_ERROR` unless a second `?` follows). `??` is the nullish operator and stays as it is. Two `?` in a row always read as `??`, so `f()??` needs a space to try twice.
- `?` at the top level of a script, outside any function, is [Q-try-top-level].
- Whether `?` also works on `Option` is [Q-try-option]. Whether it is a postfix `?` or a `try` word is [Q-try-syntax].
- The samples write `Ok(value)` and `Err(error)` without the enum name for readability. How a variant is written is [Q-variant-access] in the [Enums Proposal], and whether patterns can drop the enum name is [Q-bare-variants].

### Part D: Taking a `Result` apart

`match` ([Pattern Matching Proposal]) is the main way. A few methods that do not need a pattern should be offered once enums and generic methods allow them:

- `isOk()`, `isErr()`
- `unwrapOr(default)`, which gives the `default` for an `Err`
- `unwrap()`, which stops the program with the error text for an `Err`

`unwrap()` is how a script that does not care about failure can still use the new signatures in one word. The exact spelling depends on how generic `impl` blocks are written in the [Generic Types Proposal].

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible. It depends on enums, generics and pattern matching.

#### Part 1: Natives create a `Result`

- A native in C needs to build an `Ok` or an `Err`. That is the question the [Enums Proposal] leaves open in [Q-option-home]. Option (b) there, a built-in `Option` and `Result` that C code can create, is what this proposal needs. Option (a), enums written in `stdlib.krb`, would need a way for C to build a value of a Kirby declared enum.
- Native signatures (`src/native_signatures.c`) are parsed from text such as `fun (string) => i64`. They would need to parse `Result[i64, string]`.

#### Part 2: Scanner and parser

- A `TOKEN_QUESTION` for a lone `?`, in `src/scanner.c`.
- A postfix rule for it in the parse rule table in `src/parser.c`, next to the other postfix rules, and a new AST node.

#### Part 3: Type checker

- `?` is checked against the type of its operand and the return type of the enclosing function.
- The enclosing function is already tracked for `return`, so the same state gives the expected return type.

#### Part 4: Compiler

- `?` compiles as a hidden temporary, a test of the variant, and a `return` of the `Err` on one path. This is the same shape as a `match` arm in the [Pattern Matching Proposal] (Part 4 of its plan), so it should reuse that work.

#### Part 5: Docs and tooling

- `docs/TYPES.md`, `docs/CHANGELOG.md`, and `?` in the VS Code grammar.
- The `examples/` programs that call the changed natives, including `cat.krb` and `env.krb`.

## Impacts

### Existing Syntax Or Behavior

- The natives in Part B change their result type. A program that uses one directly as a value stops compiling until it handles the `Result`. See [Q-native-migration].
- A lone `?` was a scanner error and becomes an operator. `??` is unchanged.
- Programs that expect a missing file or a bad number to stop the program with exit code 70 will now receive an `Err` and continue.

### Related Proposals

- [Enums Proposal] — defines `Result[T, E]` and `Option[T]`. This proposal needs natives to create them ([Q-option-home]) and uses `Option` for `@getenv`. Variant order (`Err` before `Ok` in the sketch) only matters if it can be seen ([Q-discriminants]).
- [Generic Types Proposal] — `Result[T, E]` needs generic enums. 10.9 there covers enum types in substitution.
- [Pattern Matching Proposal] — the way to take a `Result` apart, and the model for how `?` compiles.
- [Strings Proposal] — `@strFromBytes` and `@strFromCode` return a `Result`. The [String Encoding Proposal] ([Q-invalid-input]) asks the same question about reading invalid text.
- [Trait Types Proposal] — an error that any struct can implement is a use of a trait type ([Q-error-type]).
- [Unimplemented Proposal] — `???` and `unwrap()` both stop the program with a message.
- [Top-Level Declarations Proposal] — decides where a script's statements live, which `?` at the top level depends on ([Q-try-top-level]).

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it runs is the file of the same name without `.argv`, written `$file` in the arguments. Language tests go in `tests/run/` and run with `run` and `$file`. See `tests/README.md` in kirbylang.

##### NEW: tests/run/result/parse_int_ok_and_err.krb

```kirby
fun show(r: Result[i64, string]): string {
  match (r) {
    Ok(n) => $"ok {n}",
    Err(e) => $"err {e}",
  }
}

@println(show(@parseInt("42")));
@println(show(@parseInt("12x")));
```

###### Expected Outcome

Prints `ok 42`, then `err` followed by the message for `"12x"`. The program exits with code 0.

##### NEW: tests/run/result/try_operator.krb

```kirby
fun twice(s: string): Result[i64, string] {
  let n = @parseInt(s)?;

  Ok(n * 2)
}
```

The program calls `twice("21")` and `twice("x")` and prints both.

###### Expected Outcome

Prints `Ok(42)` for the first call and an `Err` for the second. Nothing after the failing `?` in `twice` runs.

##### NEW: tests/run/errors/try_operator_outside_result_function.krb

```kirby
fun f(s: string): i64 {
  @parseInt(s)?
}
```

###### Expected Outcome

Does not compile. The error says `?` needs a function that returns a `Result`, and names the function's return type.

##### NEW: tests/run/errors/try_operator_error_type_mismatch.krb

A function returning `Result[i64, string]` uses `?` on a `Result[i64, ParseError]`.

###### Expected Outcome

Does not compile. The error names both error types ([Q-try-mismatch]).

##### NEW: tests/run/result/unwrap_err_stops.krb

A call to `unwrap()` on an `Err`.

###### Expected Outcome

The program stops with exit code 70 and prints the error text.

## Questions

### **Q:** What is the error type?

<!-- [Q-error-type]: #q-what-is-the-error-type -->

**Status:** Open

Natives need a fixed answer, because C code builds the value. User code needs a choice.

Options: (a) `string`. The native error text goes in as it is, and a program can print it. Simple, and a program can not tell kinds of error apart except by comparing text. (b) A user struct for user code and `string` for natives. Natives stay simple and user code can be precise. (c) A built-in `Error` trait with a `message()` method, and `Result[T, Error]` where any struct that implements it can be the error. This needs [Trait Types Proposal]. (d) A built-in `Error` struct with a message and a kind.

(a) for natives and (b) for user code is the smallest start. (c) is the likely end if trait types land. The heading of this question is linked from the [Trait Types Proposal].

### **Q:** Do the natives change in place, or get new names?

<!-- [Q-native-migration]: #q-do-the-natives-change-in-place-or-get-new-names -->

**Status:** Open

Options: (a) Change the signatures in place. One name per job, and every call site must be updated. The changelog has precedent for this kind of change: `@parseNumber` was replaced by `@parseInt` and `@parseFloat`, and `@len` by `@strLen` and `@arrLen`. (b) Add new names, such as `@tryParseInt`, and keep the old ones. Nothing breaks, and the language keeps two names for each job, one of which stops the program. (c) Change in place and add `unwrap()` so the old behavior is one word away.

(c) is the recommendation. It takes the break once and gives a short migration.

### **Q:** Is it `expr?` or `try expr`?

<!-- [Q-try-syntax]: #q-is-it-expr-or-try-expr -->

**Status:** Open

Options: (a) A postfix `?`, as Rust has it. It chains left to right (`a()?.b()?`), and costs one character. (b) A prefix `try` keyword, as Swift and Zig have it. It is easier to see, and a chain needs brackets. (c) No operator, and use `match` every time.

The scanner already treats a lone `?` as an error, so (a) needs no change to what existing programs mean. (b) reserves a word that is a valid variable name today.

### **Q:** Must the error types match?

<!-- [Q-try-mismatch]: #q-must-the-error-types-match -->

**Status:** Open

Options: (a) They must be the same type. Easy to check and to read, and a function that calls two things with different error types must convert one by hand with `match`. (b) The error is converted automatically if the caller's error type can be built from the callee's (Rust's `From`). Convenient, and needs a trait for it and generics.

(a) is the recommendation. It does not stop (b) later.

### **Q:** Does the early return operator work on `Option` too?

<!-- [Q-try-option]: #q-does-the-early-return-operator-work-on-option-too -->

**Status:** Open

`expr?` on an `Option[T]` would give the `T` for `Some` and return `None` from a function that returns an `Option`. Options: (a) Yes. The same character for the same idea. (b) No, `Result` only. Simpler, and `match` or `??` covers `Option`.

`??` would already cover a default for an `Option` ([Q-option-nil] in the [Enums Proposal]). (a) is the recommendation.

### **Q:** What does the early return operator do at the top level of a script?

<!-- [Q-try-top-level]: #q-what-does-the-early-return-operator-do-at-the-top-level-of-a-script -->

**Status:** Open

A script's statements are not in a function that returns a `Result`. Options: (a) A compile error, as in Part C. (b) The script ends with the error text and exit code 70, the same as an uncaught failure does today. (c) Same as (b), but only in a `main` function once the [Top-Level Declarations Proposal] lands.

(b) matches what failures do today, and makes a short script easy to write. (a) is the safest start.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Result**: An enum with two variants: `Ok(value)` for success and `Err(error)` for failure.
- **Option**: An enum with two variants: `Some(value)` and `None`.
- **Early return**: Leaving a function before its last statement with a value.
- **Native function**: A function written in C and built into the language. Its name starts with `@`.
- **Panic**: Stopping the program with a message because of a mistake in the program.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Enums Proposal]: ../enums/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Strings Proposal]: ../strings/PROPOSAL.md
[String Encoding Proposal]: ../string-encoding/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[Unimplemented Proposal]: ../unimplemented/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-variant-access]: ../enums/PROPOSAL.md#q-how-is-a-variant-written-and-what-else-shares-its-name
[Q-bare-variants]: ../pattern-matching/PROPOSAL.md#q-are-variant-names-written-in-full-in-patterns
[Q-option-home]: ../enums/PROPOSAL.md#q-where-do-option-and-result-live
[Q-option-nil]: ../enums/PROPOSAL.md#q-how-does-optiont-relate-to-nil
[Q-discriminants]: ../enums/PROPOSAL.md#q-can-a-program-see-or-set-the-number-of-a-variant
[Q-invalid-input]: ../string-encoding/PROPOSAL.md#q-where-is-invalid-utf-8-rejected-and-how

<!-- External -->

[Issue #39]: https://github.com/kirbylang/kirbylang/issues/39

<!-- Questions -->

[Q-error-type]: #q-what-is-the-error-type
[Q-native-migration]: #q-do-the-natives-change-in-place-or-get-new-names
[Q-try-syntax]: #q-is-it-expr-or-try-expr
[Q-try-mismatch]: #q-must-the-error-types-match
[Q-try-option]: #q-does-the-early-return-operator-work-on-option-too
[Q-try-top-level]: #q-what-does-the-early-return-operator-do-at-the-top-level-of-a-script
