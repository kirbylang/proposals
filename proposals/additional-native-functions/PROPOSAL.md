---
status: Draft
created: 2026-09-18
from_commit: b56a2a9
---

# Proposal: Additional Native Functions

This proposal fills in Kirby's native functions.

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

Checked against a clean build, `src/native.c` defines one math function:

```c
static Value ceilNative(VM *vm, int argCount, Value *args) {
  assertArgCount(vm, "ceil", 1, argCount);
  assertArgIsNumber(vm, "ceil", args, 0);
  return NUMBER_VAL(ceil(AS_NUMBER(args[0])));
}
```

and one string function, `strIsEmpty`, which just checks `length == 0`.
Everything else that would normally live in a language's standard
library — `floor`, `abs`, `sqrt`, splitting or searching a string, changing
its case — doesn't exist as a native or anywhere else, since
`stdlib/stdlib.krb` (the one other place functions like this could live) is
currently empty and unused on `main`.

Separately, native functions can optionally carry a static type signature,
checked by the type checker before the function ever runs:

```c
// src/native_signatures.h
#define NATIVE_SIGNATURE_MAX_PARAMS 2

typedef enum {
  NATIVE_UNIT,
  NATIVE_BOOL,
  NATIVE_STRING,
  NATIVE_F64,
  NATIVE_LIST,
} NativePrimitive;

typedef struct {
  const char *name;
  NativePrimitive paramTypes[NATIVE_SIGNATURE_MAX_PARAMS];
  int paramCount;
  NativePrimitive returnType;
} NativeSignature;
```

Only 16 of the ~40 existing natives actually appear in `nativeSignatures[]`
(`clock`, `__version__`, `exit`, `rand`, `rand01`, `randBetween`, `ceil`,
`readFileToString`, `writeStringToFile`, `numberToString`, `fileExists`,
`getenv`, `setenv`, `argc`, `parseNumber`, `strIsEmpty`). The rest — the
entire `arr*` family, `argv`, `is`/`isNumber`/etc. — are checked only at
runtime, by the `assert*` helpers in `src/asserts.c`. Two patterns explain
every omission, and both are precedent this proposal follows rather than
re-litigates:

- **Functions taking more than two arguments are left out.** `arrInsert`
  (array, index, value) and `arrSlice` (array, start, end) both take three
  arguments — one more than `NATIVE_SIGNATURE_MAX_PARAMS` allows — and
  neither is in the table.
- **Functions that can return `nil` to mean "absent" are left out**, even
  when they'd otherwise fit. `argv(index)` returns `nil` for an
  out-of-range index and takes one argument, comfortably within the limit —
  but it isn't in `nativeSignatures[]` either, because there's no
  `NATIVE_LIST`-style case for "this primitive type, or nil."

## The Functions

| Function                   | Params                 | Returns    |
| -------------------------- | ---------------------- | ---------- |
| `floor(n)`                 | f64                    | f64        |
| `round(n)`                 | f64                    | f64        |
| `trunc(n)`                 | f64                    | f64        |
| `abs(n)`                   | f64                    | f64        |
| `sqrt(n)`                  | f64                    | f64        |
| `pow(base, exponent)`      | f64, f64               | f64        |
| `min(a, b)`                | f64, f64               | f64        |
| `max(a, b)`                | f64, f64               | f64        |
| `strContains(s, sub)`      | string, string         | bool       |
| `strIndexOf(s, sub)`       | string, string         | f64 or nil |
| `strSlice(s, start, end)`  | string, f64, f64       | string     |
| `strSplit(s, sep)`         | string, string         | array      |
| `strTrim(s)`               | string                 | string     |
| `strToUpper(s)`            | string                 | string     |
| `strToLower(s)`            | string                 | string     |
| `strStartsWith(s, prefix)` | string, string         | bool       |
| `strEndsWith(s, suffix)`   | string, string         | bool       |
| `strRepeat(s, count)`      | string, f64            | string     |
| `strReplace(s, old, new)`  | string, string, string | string     |

Math native functions wrap the corresponding `<math.h>` function directly (`floor`, `round`, `trunc`, `fabs`, `sqrt`, `pow`) or a direct comparison (`min`, `max`), the same way `ceilNative` wraps `ceil`. `sqrt`'s behavior outside its domain (negative input) is [Q-sqrt-domain].

Notes, each following an existing pattern rather than inventing a new one:

- **`strIndexOf` returns `nil`, not `-1`, when the substring isn't found.**
  This follows `argv`'s existing precedent for "absent" rather than
  introducing a sentinel value with no precedent elsewhere in the language.
- **`strSlice` mirrors `arrSlice` exactly**: the same
  `assertPositiveNumber` / `assertIsInArrayBounds`-equivalent bounds
  checks, and the same `start < end` requirement.
- **`strTrim` removes ` `, `\t`, `\n`, and `\r`** — the same whitespace
  characters the lexer already recognizes as escape sequences, not a
  locale-dependent `isspace()`.
- **`strToUpper`/`strToLower` are byte-oriented (ASCII), not Unicode-aware**,
  consistent with the rest of the string implementation — `ObjString` is a
  `char*` and a byte length, with no encoding tracked anywhere today.
- **`strReplace`'s replace-first-vs-replace-all behavior** is
  [Q-replace-all].
- **`strSplit`'s behavior on an empty separator** is [Q-split-empty].

## Questions

### **Q:** What should `sqrt` do outside its domain?

<!-- [Q-sqrt-domain]: #q-what-should-sqrt-do-outside-its-domain -->

**Status:** Open

`sqrt(n)` is undefined for `n < 0`. None of the existing single-argument
math helpers have a domain restriction to follow as precedent —
`assertPositiveNumber` is the closest existing assert, but it rejects `0`
too (`number <= 0`), which is one input too strict for `sqrt` (`sqrt(0)` is
`0`, a valid result).

Options: (a) raise a runtime error for negative input, which needs a new
`assertNonNegativeNumber` helper (`number < 0`) alongside the existing
`assertPositiveNumber`/`assertNonZero`; (b) return whatever C's `sqrt`
returns for a negative input (`NaN`, per IEEE 754), and let it propagate
silently until it surfaces somewhere confusing. (a) is consistent with how
every other invalid-input case in `native.c` is handled today — everything
else raises rather than returning a sentinel; (b) is cheaper to implement
and avoids the new assert helper.

### **Q:** Does `strReplace` replace the first occurrence or every occurrence?

<!-- [Q-replace-all]: #q-does-strreplace-replace-the-first-occurrence-or-every-occurrence -->

**Status:** Open

Conventions differ across languages the target audience is likely to have
used — replacing every occurrence by default is more common, but not
universal, and there's no existing Kirby precedent either way to defer to.

Options: (a) replace every occurrence, matching the majority convention and
the intuition that `arrJoin`'s inverse should handle a whole string, not
just its first match; (b) replace only the first occurrence, which is
cheaper to implement correctly (no need to re-scan the replacement for
further matches of `old`) and leaves "replace all" open for a later
`strReplaceAll` if it's wanted. If (a), whether `new` itself can contain
`old` and cause re-matching inside the replacement needs an explicit answer
(most implementations scan the _original_ string's remaining tail, not the
freshly-substituted output, to avoid infinite work — that should be stated
outright rather than left to whatever the implementation happens to do).

### **Q:** What does `strSplit` do with an empty separator?

<!-- [Q-split-empty]: #q-what-does-strsplit-do-with-an-empty-separator -->

**Status:** Open

`strSplit("abc", "")` has no single obvious answer.

Options: (a) raise a runtime error, consistent with how `native.c` generally
prefers raising over guessing at a caller's intent; (b) split into individual
one-character strings, which is a useful and fairly common behavior
elsewhere; (c) return the whole string as a single-element array, treating
"no separator" as "no split points found." (a) is the safest default and
the easiest to loosen later if (b) or (c) turns out to be wanted; loosening
a runtime error into defined behavior is backwards-compatible, the reverse
is not.

## Glossary

These are both technical and non technical terms used throughout the
proposal.

- **Changes**: Changes refer the proposed changes in this document
- **Native function**: A function implemented in C and exposed to Kirby
  code under a global name, as opposed to one written in Kirby itself.
  Defined in `src/native.c`.
- **Native signature**: An optional static type declaration for a native
  function (see `src/native_signatures.h`), checked by the type checker at
  a call site the same way a Kirby-defined function's declared types are. A
  native without one is still callable; its argument and return types are
  only checked at runtime, inside the native itself.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement

<!-- Questions -->

[Q-sqrt-domain]: #q-what-should-sqrt-do-outside-its-domain
[Q-replace-all]: #q-does-strreplace-replace-the-first-occurrence-or-every-occurrence
[Q-split-empty]: #q-what-does-strsplit-do-with-an-empty-separator
