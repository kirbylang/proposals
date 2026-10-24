---
status: Draft
category: feature
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: Strings

This proposal fills the gaps in how Kirby programs read, compare and build text. A string literal can not hold a character by its Unicode code. There is no way to turn a number or a list of bytes into a string. `<` does not work on strings. `@parseInt` can not read hex. Reading one character means copying a one character string. Each gap is small, and together they stop a program from handling any text format properly: a JSON parser can not decode `\u00e9`, and a tokenizer can not tell a digit from a letter without a lookup string. What a string is made of (bytes or characters) is a separate question, settled in the [String Encoding Proposal]. This proposal assumes its recommended answer, UTF-8 text with byte positions.

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

Each of these was verified against a build of `from_commit`.

**1. A literal can not name a character by its code.** The parser's `stringLiteral` (`src/parser.c`) knows five escapes: `n`, `r`, `t`, `"` and `\`. Anything else is an error:

```kirby
@println("\u00e9");
// [line 1] Error at '"\u00e9"': Invalid escape sequence: \u
```

A character that is not on the keyboard can only be written by pasting it into the file.

**2. There is no way to build a string from a number.** `@strToBytes` turns a string into `[u8]`, and nothing turns it back. `@bytesToString` is not a native (`Undefined variable '@bytesToString'.`), and there is nothing like `@chr`. A program that decodes a text format, such as the `\uXXXX` escape in JSON, can read the number and can not make the character.

**3. Strings can not be ordered.** Comparing with `<` is a type error:

```kirby
@println("a" < "b");
// [line 1] Error at '<': '<' needs two numbers of the same type, got string and string.
```

`==` works. A range check such as "is this character a digit" has to be written as `@strContains("0123456789", c)`.

**4. `@parseInt` has one parameter and reads decimal only.** `@parseInt("0x1f")` reports `function @parseInt expects argument 1 to be a whole number but got "0x1f".`, and `@parseInt("ff", 16)` reports `Expected 1 argument(s), got 2.` Natives check their argument count (`assertArgCount`), so a second parameter would be a different signature.

**5. Reading one character copies a string.** A string can not be indexed (`s[1]` reports `Can't index into a string.`). The way to read position `i` is `@strSlice(s, i, i + 1)`, and `strSliceNative` ends with `copyString(vm->gc, string->chars + start, end - start)`, a new allocation on every call. There is also no way to read a character's code without an array: `@strToBytes(s)` copies the whole string into an array.

**6. Building a string.** `+` makes a new string each time. Three or more strings joined by `+` in one expression compile to a single `@strConcat([...])` (CHANGELOG), and `examples/stringBuilder.krb` shows a `StringBuilder` written in Kirby with a `[string]` and `@strConcat`. The [String Interpolation Proposal] (Closed) already weighed a `StringBuilder` in the standard library and chose `@strConcat`. That part is covered, and this proposal does not reopen it.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- A Unicode escape in string literals
- Natives to build a string from bytes, and from a character code
- A native to read the code of the character at a position
- Ordering strings with `<`, `<=`, `>` and `>=`
- Reading numbers in another base
- Whether Kirby gets a character type or a way to index a string ([Q-char-type])

The following is intentionally left out of scope for this proposal:

- What a string is, and what a position means. See the [String Encoding Proposal]. This proposal uses its recommended answer.
- A `StringBuilder`. See above.
- Regular expressions, formatting, padding and locale
- Case changes beyond ASCII
- Making `@parseInt` and `@parseFloat` return a `Result` instead of stopping the program. See the [Result Proposal].
- Iterating a string with `for`. See the [For-Each Proposal].

### Part A: A Unicode escape

A string literal, including an interpolated one (`$"..."`), should accept `\u{` followed by one to six hex digits and `}`:

```kirby
@println("caf\u{e9}");       // café
@println("\u{1F600}");       // 😀
@println($"price: \u{20ac}{amount}");
```

The escape stands for the UTF-8 bytes of that code point. A value that is not a character is a compile error: a surrogate (`D800` to `DFFF`) or anything above `10FFFF`. Both literal kinds use `stringLiteral`, so the change is in one place.

The braces make the length clear: four digits can not reach past `FFFF`, and a fixed six would need leading zeros. See [Q-escape-syntax].

### Part B: Building a string from numbers

Two new natives, in the style of `src/native_signatures.c`:

| Native                 | Signature              | Result                                                 |
| ---------------------- | ---------------------- | ------------------------------------------------------ |
| `@strFromBytes(bytes)` | `fun ([u8]) => string` | The string those bytes hold. Inverse of `@strToBytes`. |
| `@strFromCode(code)`   | `fun (i64) => string`  | The one character string for a code point.             |

Both stop with a runtime error on input that is not valid text: bytes that are not valid UTF-8, a code that is a surrogate or above `10FFFF`, or a negative code. That follows the [String Encoding Proposal] Part D. Once the [Result Proposal] lands they are natural candidates for returning a `Result`.

### Part C: Reading a character's code

| Native                    | Signature                  | Result                                                     |
| ------------------------- | -------------------------- | ---------------------------------------------------------- |
| `@strCodeAt(s, position)` | `fun (string, i64) => i64` | The code point of the character that starts at `position`. |

It copies nothing, so a loop can test a character without making a string. With `@strCharSize` from the [String Encoding Proposal] it walks a string:

```kirby
var i = 0;

while (i < @strLen(s)) {
  let code = @strCodeAt(s, i);

  if (code >= 48 and code <= 57) {
    // a digit
  }

  i = i + @strCharSize(s, i);
}
```

`@strCodeAt` follows the position rules of `@strCharSize`: it is an error at a position that is not the start of a character, or that equals `@strLen(s)`.

### Part D: Ordering strings

`<`, `<=`, `>` and `>=` should accept two strings. They compare byte by byte from the start, and a shorter string that is a prefix of a longer one comes first. For UTF-8 this is the same as ordering by code point, so it does not need to decode anything. See [Q-compare] for whether this is an operator rule or an `Ord` impl.

### Part E: Other bases

`@parseInt` should be able to read a number in base 2 to 36. See [Q-radix] for how, since a native can not take an optional parameter today.

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Escape (A)

- `stringLiteral` in `src/parser.c` gets a `u` case. It reads the digits, checks the range, and writes the UTF-8 bytes into `buffer`.
- `buffer` is allocated as `length + 1`, and a `\u{...}` always takes at least as many source bytes as it produces bytes, so the size stays safe.
- The scanner's `string` function already skips the character after a backslash, so the `u` is passed over without a rule. The interpolated scanner (`interpSegment`) skips the `u` the same way and then meets the `{`, which it reads as the start of a placeholder. It needs a rule so `{` right after `\u` belongs to the escape ([Q-escape-syntax]).

#### Part 2: Natives (B, C, E)

- `strFromBytesNative`, `strFromCodeNative` and `strCodeAtNative` in `src/native.c`, registered in `nativeDefinitions`, with entries in `src/native_signatures.c`.
- Added to the VS Code grammar and hover data.

#### Part 3: Operators (D)

- The type checker's rule for `<` (`needs two numbers of the same type`) accepts two strings.
- The VM's comparison instructions get a string case.

#### Part 4: Docs

- `docs/TYPES.md` and `docs/CHANGELOG.md`.

## Impacts

### Existing Syntax Or Behavior

- `\u` followed by `{` in a literal used to be an error and now means something. Nothing that compiled before changes.
- `<` on strings used to be a type error and now works.
- New native names are reserved (they start with `@`, which already can not be declared).

### Related Proposals

- [Lists Proposal] — `@strSplit` and `@strToBytes` return a list `[T]`, and `@arrJoin` is renamed `@listJoin`.
- [String Encoding Proposal] — decides what a string and a position are. Parts A to C here assume UTF-8 and byte positions ([Q-model] there).
- [Result Proposal] — `@strFromBytes`, `@strFromCode` and `@parseInt` are inputs from outside the program's control, and are natural to return a `Result`.
- [For-Each Proposal] — a `for` loop over a string would use these natives.
- [Primitive Impls Proposal] — `Ord` for `string` is the other way to get `<` ([Q-compare]). It lists primitive trait impls as unsupported today (`Primitive trait implementations aren't supported yet.`).
- [String Interpolation Proposal] — Closed. `$"..."` shares `stringLiteral`, and `@strConcat` is the answer to building strings.
- [Additional Native Functions Proposal] — Closed. It added the other `str*` natives and sets the pattern for naming.
- [Collection Methods Proposal] — if strings get methods, these natives become methods.
- [Generic Types Proposal] — none of this needs generics, which is why it can be built first.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it runs is the file of the same name without `.argv`, written `$file` in the arguments. Language tests go in `tests/run/` and run with `run` and `$file`. See `tests/README.md` in kirbylang.

##### NEW: tests/run/literals/string_unicode_escape.krb

```kirby
@println("caf\u{e9}");
@println("\u{20AC}");
@println("\u{1F600}");
@println(@strLen("\u{e9}"));
@println("\u{e9}" == "é");
```

###### Expected Outcome

Prints `café`, `€`, `😀`, then `2`, then `true`.

##### NEW: tests/run/errors/string_unicode_escape_surrogate.krb

```kirby
@println("\u{D800}");
```

###### Expected Outcome

Does not compile. The error names the escape and says it is not a character.

##### NEW: tests/run/native_functions/native_fn_str_from_code_call.krb

```kirby
@println(@strFromCode(65));
@println(@strFromCode(233));
@println(@strFromBytes([104, 105]));
@println(@strCodeAt("é", 0));
```

###### Expected Outcome

Prints `A`, `é`, `hi`, `233`.

##### NEW: tests/run/operators/string_less_than.krb

```kirby
@println("a" < "b");
@println("b" < "a");
@println("ab" < "abc");
@println("a" >= "a");
```

###### Expected Outcome

Prints `true`, `false`, `true`, `true`.

## Questions

### **Q:** What does the Unicode escape look like?

<!-- [Q-escape-syntax]: #q-what-does-the-unicode-escape-look-like -->

**Status:** Open

Options: (a) `\u{XXXX}` with one to six hex digits, as in Rust. Reaches every code point, and the braces show where it ends. (b) `\uXXXX` with exactly four digits, as in JSON and JavaScript. Familiar, but a character above `FFFF` needs a pair of escapes (a surrogate pair), which this proposal rejects as invalid. (c) Both, with `\U` plus eight digits for the long form, as in C and Python.

(a) is the proposal's recommendation. One thing to check in the scanner: the interpolated string scanner treats `{` as the start of a placeholder, so `$"\u{e9}"` must read as an escape and not as a placeholder.

### **Q:** Is string ordering an operator rule or an `Ord` impl?

<!-- [Q-compare]: #q-is-string-ordering-an-operator-rule-or-an-ord-impl -->

**Status:** Open

Options: (a) The type checker and VM accept `<` on two strings directly, as they do for two numbers. Small, and works at once. (b) `impl Ord for string`, so `<` is `a.cmp(b) < 0`. This is how `Ord` already orders a struct and it would work with generic code that asks for `Ord`. It needs the [Primitive Impls Proposal], which does not support trait impls on primitives yet.

(a) first, and (b) later makes both true: the operator keeps working, and `string` also satisfies `Ord`.

### **Q:** How does `@parseInt` take a base?

<!-- [Q-radix]: #q-how-does-parseint-take-a-base -->

**Status:** Open

Options: (a) A new native, such as `@parseIntBase(s, base)`. No change to how natives work. (b) An optional second parameter on `@parseInt`. This needs native signatures to describe an optional parameter, which they can not today (`fun (string) => i64`). (c) Wait for the [Result Proposal] and design both changes to `@parseInt` once.

(c) is tidy and slow. (a) can be shipped now and the name is easy to keep next to a `Result` version.

### **Q:** Does Kirby get a character type or string indexing?

<!-- [Q-char-type]: #q-does-kirby-get-a-character-type-or-string-indexing -->

**Status:** Open

Today a "character" is a string of one character. Options: (a) Keep it that way, and use `@strCodeAt`, `@strCharSize` and `@strSlice` as in Part C. Nothing new in the type system. (b) Add `s[i]` that returns a one character string. Short to write. It hides a copy and a cost on every read, and `i` is a byte position, which a reader may not expect from index syntax. (c) Add a `char` type, a code point with its own literal like `'a'`. It is the clearest model, and the largest change: a literal, a type, conversions, and `Display`.

(a) is the smallest and the other two can follow. (c) is worth its own proposal if programs find the one character strings awkward.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Escape**: A backslash and letters in a string literal that stand for another character, like the `n` in a newline.
- **Code point**: A number that stands for one character in Unicode. `A` is 65 and `é` is 233.
- **Surrogate**: A number in the range `D800` to `DFFF`. It is used by UTF-16 to build large code points in pairs, and is not a character in its own right.
- **UTF-8**: A way to store text as bytes, one to four per character.
- **Base** (or radix): The number of digits a number system has. Base 16 is hex.
- **Native function**: A function written in C and built into the language. Its name starts with `@`.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[String Encoding Proposal]: ../string-encoding/PROPOSAL.md
[Result Proposal]: ../result/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Additional Native Functions Proposal]: ../additional-native-functions/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-model]: ../string-encoding/PROPOSAL.md#q-are-strings-bytes-utf-8-text-or-characters

<!-- Questions -->

[Q-escape-syntax]: #q-what-does-the-unicode-escape-look-like
[Q-compare]: #q-is-string-ordering-an-operator-rule-or-an-ord-impl
[Q-radix]: #q-how-does-parseint-take-a-base
[Q-char-type]: #q-does-kirby-get-a-character-type-or-string-indexing
