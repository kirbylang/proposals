---
status: Draft
category: feature
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: String Encoding

This proposal decides what a Kirby `string` is made of: bytes, or characters. Today a string is a run of bytes, and every native function counts and cuts it by bytes. That is fine for plain English text, and it quietly breaks for anything else. The proposal keeps strings as UTF-8 text with byte positions, makes cutting a character in half an error, and adds the small set of natives a program needs to walk a string one character at a time. The [Strings Proposal] builds on the answer.

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

A string in Kirby is a length and a pointer to bytes (`src/object.h`):

```c
struct ObjString {
  Obj obj;
  int length;
  char *chars;
  uint32_t hash;
};
```

`length` is a count of bytes. Nothing records an encoding, and nothing checks that the bytes mean anything. Each of these was verified against a build of `from_commit`:

```kirby
let s = "aéb";

@println(@strLen(s));          // 4, not 3
@println(@strSlice(s, 1, 3));  // é
@println(@strSlice(s, 1, 2));  // half of é: one byte that is not valid text
@println(@strIndexOf(s, "b")); // 3, a byte position
@println(@strToBytes("é"));    // [195, 169]
```

- **Source text passes through as it is.** The scanner copies the bytes between the quotes. A file saved as UTF-8 gives strings of UTF-8 bytes.
- **Every position is a byte position.** `@strLen`, `@strSlice`, `@strIndexOf` and `@strRepeat` all work in bytes. `assertStringPosition` in `src/native.c` accepts any whole number from `0` up to `string->length`, including a number that lands in the middle of a character.
- **Cutting a character in half is allowed.** `@strSlice(s, 1, 2)` above succeeds and returns a string that no text tool can read.
- **Case changes only touch ASCII.** The [Additional Native Functions Proposal] (Closed) records that `@strToUpper` and `@strToLower` are byte-oriented. `@strToUpper("éa")` returns `éA`.
- **There is no way to check for valid text.** `@readFileToString` returns whatever bytes the file holds.

The result is a trap. A program that walks a string one position at a time works on every test string written in English and fails on the first name with an accent. A parser for a text format (JSON, TOML, source code) is exactly such a program.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- Stating what a `string` holds: UTF-8 text
- Keeping every position a byte position, so existing code keeps its meaning
- Making `@strSlice` refuse a position inside a character
- Natives to find where the next character starts, and to count characters
- Where the text is checked for validity

The following is intentionally left out of scope for this proposal:

- Escapes like a Unicode code in a string literal, a way to build a string from a code number, and string comparison with `<`. See the [Strings Proposal].
- Unicode normalization, so `é` as one code point and `e` plus an accent are different strings
- Grapheme clusters (what a reader sees as one symbol, which can be several code points)
- Case mapping beyond ASCII, and anything locale dependent
- Encodings other than UTF-8
- Iterating a string with a `for` loop. See the [For-Each Proposal] ([Q-string-iteration]).

### Part A: Strings are UTF-8 text

A `string` should hold valid UTF-8. A position in a string should stay a count of bytes from the start. A "character" in this proposal means one Unicode code point, which is one to four bytes.

Byte positions keep `@strLen`, `@strIndexOf`, `@strSlice`, `@strRepeat` and `@strToBytes` meaning exactly what they mean now, and keep each of them as fast as it is. Nothing is scanned to find position `n`.

### Part B: Positions must fall between characters

`@strSlice` should report a runtime error when `start` or `end` is inside a character:

```kirby
@strSlice("aéb", 1, 2);
// function @strSlice expects argument 3 to be at the start of a character but got 2.
```

This follows the other position errors in `assertStringPosition`, which already name the argument and the bad number. A position of `0` or `@strLen(s)` is always valid. Whether it is an error or rounds to a character start is [Q-boundary].

`@strIndexOf` needs no change: it returns the start of a match, and a valid string only matches at a character start.

### Part C: Walking a string by character

Two new natives, with signatures in the style of `src/native_signatures.c`:

| Native                      | Signature                  | Result                                                         |
| --------------------------- | -------------------------- | -------------------------------------------------------------- |
| `@strCharSize(s, position)` | `fun (string, i64) => i64` | Bytes taken by the character that starts at `position`: 1 to 4 |
| `@strCharCount(s)`          | `fun (string) => i64`      | Number of characters in `s`                                    |

`@strCharSize` makes a character-by-character loop short and keeps it fast:

```kirby
var i = 0;

while (i < @strLen(s)) {
  let size = @strCharSize(s, i);
  let ch = @strSlice(s, i, i + size);

  // use ch

  i = i + size;
}
```

`@strCharSize` reports an error at a position that is not the start of a character, or that is equal to `@strLen(s)`, in the same style as the other position errors.

`@strCharCount` is a scan from the start, so it is linear. It is named for what it counts so that nobody reads `@strLen` as a character count.

### Part D: Where text is checked

Text becomes invalid UTF-8 in only a few places. Each should be checked once, at the border, so the rest of the language can trust a `string`:

- **Source files.** The scanner should reject invalid UTF-8 in a file, with the line.
- **`@readFileToString`.** See [Q-invalid-input].
- **`@strFromBytes`.** The new native in the [Strings Proposal] that builds a string from `[u8]`.
- **`@getenv`, `@argv`, `@prompt`, `@stdin`.** Operating systems hand these over as bytes. See [Q-invalid-input].

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Validity helpers

A small function in `src/object.c` that says whether a byte run is valid UTF-8, and one that gives the size of the character at a position. Both are plain C with no allocation, and are used by every part below.

#### Part 2: Position checks

- `assertStringPosition` in `src/native.c` gains a flag for "must be at a character start". `@strSlice` sets it for both positions.
- The error text follows the existing one, naming the argument.

#### Part 3: New natives

- `@strCharSize` and `@strCharCount` in `src/native.c`, with entries in `nativeDefinitions` and signatures in `src/native_signatures.c`.
- Added to the VS Code grammar and hover data, as for every native.

#### Part 4: Border checks

- The scanner checks string literal bytes. The check is skipped for text that is plain ASCII.
- Natives that return outside text follow [Q-invalid-input].

#### Part 5: Docs

- `docs/TYPES.md` gets a Strings section saying a string is UTF-8 and positions are bytes.
- `docs/CHANGELOG.md` gets an entry.

## Impacts

### Existing Syntax Or Behavior

- `@strSlice` with a position inside a character stops with an error where it used to return broken text. Programs that did this were already getting invalid strings.
- A file with invalid UTF-8 in a string literal stops compiling. Nothing in this repository has one.
- `@strLen`, `@strIndexOf`, `@strRepeat`, `@strToBytes` and every other native keep their behavior.
- Equality is still byte equality, so two strings that look the same but are built differently (see Normalization above) compare unequal.

### Related Proposals

- [Strings Proposal] — builds on this one. It adds a Unicode escape for literals, `@strFromBytes`, a way to make a string from a code point, and string comparison. All of them need the answer to [Q-model].
- [Additional Native Functions Proposal] — Closed. It noted that case natives are byte-oriented. That stays true here.
- [String Interpolation Proposal] — Closed. `@strConcat` joins valid strings into a valid string, so it needs no change.
- [For-Each Proposal] — whether a `for` loop over a string gives bytes or characters ([Q-string-iteration]). The answer follows from [Q-model].
- [Primitive Impls Proposal] — string methods, if `string` gets impls, would carry the same position rules.
- [Result Proposal] — the checks in Part D that can fail on outside input are natural users of `Result`.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it runs is the file of the same name without `.argv`, written `$file` in the arguments. Language tests go in `tests/run/` and run with `run` and `$file`. See `tests/README.md` in kirbylang.

##### NEW: tests/run/native_functions/native_fn_str_slice_call_inside_character.krb

```kirby
@println(@strSlice("aéb", 1, 2));
```

###### Expected Outcome

The program stops with exit code 70 and an error naming `@strSlice`, argument 3, and the position. Nothing is printed to standard output.

##### NEW: tests/run/native_functions/native_fn_str_char_size_call.krb

```kirby
let s = "aéb€😀";

var i = 0;

while (i < @strLen(s)) {
  let size = @strCharSize(s, i);

  @println(@strSlice(s, i, i + size));

  i = i + size;
}

@println(@strCharCount(s));
```

###### Expected Outcome

Prints `a`, `é`, `b`, `€`, `😀` on separate lines, then `5`.

##### NEW: tests/run/native_functions/native_fn_str_char_size_call_inside_character.krb

```kirby
@println(@strCharSize("é", 1));
```

###### Expected Outcome

The program stops with exit code 70 and an error saying position 1 is not the start of a character.

##### NEW: tests/run/errors/string_literal_invalid_utf8.krb

A file whose string literal holds the single byte `0xE9`.

###### Expected Outcome

The program does not compile, and the error names the line of the literal.

## Questions

### **Q:** Are strings bytes, UTF-8 text, or characters?

<!-- [Q-model]: #q-are-strings-bytes-utf-8-text-or-characters -->

**Status:** Open

Options:

- (a) **Bytes.** Keep strings as they are and document that all natives work in bytes. Nothing to build. The trap in the Problem Statement stays, and every program that handles text has to know about it.
- (b) **UTF-8 text with byte positions.** What this proposal describes. Existing positions keep their meaning and speed. The cost is a validity check at the borders and an error for a position inside a character.
- (c) **A sequence of characters.** `@strLen` counts characters and position `n` means the `n`th character. This is the easiest to read. Finding position `n` needs a scan from the start unless the string also stores an index of its characters, and every position in every native changes meaning, which breaks `@strToBytes` round trips.

(b) is the proposal's recommendation. It can grow into (c) later by adding natives, but (c) cannot shrink back to (b).

### **Q:** Where is invalid UTF-8 rejected, and how?

<!-- [Q-invalid-input]: #q-where-is-invalid-utf-8-rejected-and-how -->

**Status:** Open

Source files can be rejected at compile time. Outside input is harder, because the bytes come from the world: a file, an environment variable, a program argument, a line typed at a prompt.

Options: (a) A runtime error, like a missing file today. Simple, and wrong for a program that wants to read any file and decide for itself. (b) Wait for the [Result Proposal] and return `Result[string, E]` from the natives that read outside input. (c) Do not check, and let `@strCharSize` and `@strCharCount` report the error when they meet a bad byte.

(a) is the smallest first step. (b) is where it should end up.

### **Q:** Should a position inside a character be an error, or round to the nearest start?

<!-- [Q-boundary]: #q-should-a-position-inside-a-character-be-an-error-or-round-to-the-nearest-start -->

**Status:** Open

Options: (a) An error, as Part B describes. A wrong position is almost always a bug, and the error points at it. (b) Round down to the start of the character. No program stops, and a bug becomes a quietly wrong string.

(a) matches how `@strSlice` already treats a position past the end.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Byte**: The smallest unit a string is stored in. A number from 0 to 255.
- **UTF-8**: A way to store text as bytes. The characters of English are one byte each. Other characters take two to four bytes.
- **Code point**: A number that stands for one character in Unicode. `é` is code point 233.
- **Character**: In this proposal, one code point. A reader may see several code points as one symbol, which is out of scope.
- **Position**: A count of bytes from the start of a string.
- **Normalization**: Writing the same text in one agreed way, because some characters can be built from one code point or from several.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Strings Proposal]: ../strings/PROPOSAL.md
[Additional Native Functions Proposal]: ../additional-native-functions/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Result Proposal]: ../result/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-string-iteration]: ../for-each/PROPOSAL.md#q-what-does-a-for-loop-over-a-string-give

<!-- Questions -->

[Q-model]: #q-are-strings-bytes-utf-8-text-or-characters
[Q-invalid-input]: #q-where-is-invalid-utf-8-rejected-and-how
[Q-boundary]: #q-should-a-position-inside-a-character-be-an-error-or-round-to-the-nearest-start
