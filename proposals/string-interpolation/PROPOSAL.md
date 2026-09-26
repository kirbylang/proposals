---
status: Draft
created: 2026-09-19
from_commit: 8d21cbd
---

# Proposal: String Interpolation

This proposal adds string interpolation to Kirby: a `$`-prefixed string
literal whose body can contain `{expression}` placeholders, e.g.
`$"Hello {name}!"`. Each interpolated string compiles to one call to a new
native, `@strConcat`, which joins its pieces in a single step. There is no new
opcode and nothing is added to `stdlib/stdlib.krb`. The same join is used for
chains of three or more strings joined with `+`.

It is implemented in [PR #95]. The sections below describe that
implementation.

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing
process to understand the changes being proposed and what impacts they will
have. That research is largely tracked in the form of [Questions].

### Linking

[Links] in this document are defined as link references.

---

## Problem Statement

Building a string out of parts today means chaining `+` and converting
non-string values by hand:

```kirby
var name = "World";
print "Hello " + name + "!";

var pi = 3.14;
print "pi is " + @numberToString(pi) + " roughly";
```

That works — `string + string` is accepted by the type-checker
(`typchkInferBinary` in `src/typecheck.c`) and `OP_ADD` concatenates strings
in the VM (`concatenate()` in `src/vm.c`) — but the seams between literal
text and values get buried in operator syntax, and every non-string segment
needs a manual `@numberToString`.

It is also slow for long chains. Each `+` makes a new string, copying
everything built so far, and every in-between string is hashed and interned.
A chain of `n` strings copies the start of the result `n - 1` times.

[Issue #15] proposes:

```kirby
var name = "World";

print $"Hello {name}!";
```

The `$` prefix exists so the scanner can tell an interpolated string from an
ordinary one at the start of the token, without having to scan the string
for delimiters.

At `from_commit` that example fails at scan time, because `$` is not a token
in `src/scanner.c` and falls through to the error default:

```
$ ./build/krb -f test.krb
[line 2] Error: Unexpected character.
```

The issue's acceptance criteria: "If it's possible to add the StringBuilder
implementation to stdlib.krb then construct bytecode to instantiate and build
the interpolated string. Or if there is a better way of doing it." This
proposal takes the "better way": a native, compared against the
`StringBuilder` in [Q-strategy].

## Proposed Changes

### Part 1 — Syntax

An interpolated string is a `$` immediately followed by a string literal.
The `$` must be directly adjacent to the opening quote (no whitespace); a
lone `$`, or a `$` followed by anything but `"`, remains an "Unexpected
character" error.

The body of an interpolated string is a sequence of **segments**: literal
text and `{expression}` placeholders.

- A placeholder can contain any Kirby expression, so calls, indexing, block
  expressions, and struct literals with their own braces are legal:
  `$"sum={p.x + p.y}"`, `$"first={items[0]}"`. An interpolated string can
  nest inside a placeholder of another, up to 16 deep.
- A placeholder's value must be a `string`, `f64`, `bool`, or a type that
  implements `Display` ([Q-segment-types]). Numbers, bools, and `Display`
  values are converted to strings.
- Literal text uses the same escapes as ordinary string literals (`\n`, `\r`,
  `\t`, `\"`, `\\`). Literal braces are doubled: `{{` is a `{` and `}}` is a
  `}` ([Q-braces]).
- A single `}` in literal text is an error ([Q-bare-brace]).
- Like ordinary strings, a body may span multiple lines.
- An interpolated string is an **expression** of type `string`: it appears
  anywhere a string literal does.
- `$"text"` with no placeholders is allowed, and is the same as `"text"`
  except that `{{` and `}}` are literal braces.

```kirby
let name = "World";
let count = 3;
let ok = true;

print $"Hello {name}! count={count} ok={ok}"; // Hello World! count=3 ok=true
print $"{{name}} is {name}";                   // {name} is World
print $"{{{count}}}";                          // {3}
```

### Part 2 — `@strConcat` and `@boolToString` natives

Two natives are added to `src/native.c`.

**`@strConcat(strings)`** joins an array of strings, with nothing between
them:

```kirby
print @strConcat(["Hello", ", ", "World"]); // Hello, World
print @strConcat([]);                       // an empty string
```

It shares its body with `@arrJoin` through a `joinStrings` helper, which
checks every element is a string while adding up the total length, then
allocates the result once and copies each string into it. Every string is
copied once.

`@strConcat` has no type signature yet, like `@arrJoin` and the other array
natives, so its argument is checked when the program runs. A `[string]`
parameter would reject arrays whose element type is unknown, such as the
result of `@strSplit`, and the type-checker has no "array of anything" until
there are generics ([Generic Types Proposal]).

**`@boolToString(b)`** returns `"true"` or `"false"`, matching
`@numberToString` for numbers. It has a signature, `(bool) -> string`.

### Part 3 — Compiler lowering

An interpolated string compiles to one `@strConcat` call, with each
placeholder converted first:

```kirby
$"Hello {name}! count={count} ok={ok}"

// compiles as
@strConcat(["Hello ", name, "! count=", @numberToString(count), " ok=", @boolToString(ok)])
```

- Literal text becomes string constants. Empty text is left out, so
  `$"{a}{b}"` joins two parts, not three.
- A single part needs no join: `$"{name}"` compiles to `name`, and `$"{n}"`
  to `@numberToString(n)`.
- The conversion for each placeholder is chosen by the type-checker, because
  the compiler has no types ([Part 4]). A value whose type implements
  `Display` is converted by calling its `toString()` method.
- Conversions happen before the call, so `@strConcat` only ever receives
  strings. A native can't call a Kirby method, so keeping conversions outside
  it is what lets a placeholder use a type's `Display` impl: the compiler
  calls `toString()` on the value itself.

For `$"Hello {name}!"` the bytecode is:

```
OP_GET_GLOBAL       '@strConcat'
OP_CONSTANT         'Hello '
OP_GET_GLOBAL       'name'
OP_CONSTANT         '!'
OP_ARRAY            3
OP_CALL             1
```

**Limit: 255 parts.** `OP_ARRAY` holds its item count in one byte, so an
interpolated string can have at most 255 parts (text pieces and
placeholders). More is a compile error, "Too many parts in interpolated
string." Array literals already have the same limit.

#### `+` chains of strings

The same join is used for `+`. When both sides of a `+` are strings, the
type-checker marks the `+` node, and the compiler flattens a chain of marked
`+` nodes into a list of the strings they join:

```kirby
a + b + c + d        // parsed as ((a + b) + c) + d
a + (b + c) + d      // parentheses are looked through

// both compile as
@strConcat([a, b, c, d])
```

- **Two strings keep a single `OP_ADD`**, which is cheaper than building an
  array and calling a native.
- **Chains longer than 255 strings** join the first 255 and add the rest with
  `OP_ADD`.
- **A `+` whose operand types aren't known** is not marked, and compiles to
  `OP_ADD` as before, so the VM still checks it at runtime.

Measured with 200,000 loop iterations, before and after (the join was
measured as `@arrJoin`, which shares `@strConcat`'s implementation):

| Strings joined | `OP_ADD` chain | One join |
|---|---|---|
| 8 × 200 characters | 2.12s | 0.54s |
| 16 × 5 characters | 0.31s | 0.12s |
| 3 × 200 characters | 0.32s | 0.21s |
| 3 × 5 characters | 0.033s | 0.047s |
| 2 strings (stays `OP_ADD`) | 0.131s | 0.152s |

The join only loses on three or four very short strings, by a few
nanoseconds each, which is why the cut-off is three.

### Part 4 — Scanner, parser, and type-checker changes

#### Scanner

Four new tokens split an interpolated string into its text pieces, with
ordinary tokens for each placeholder's expression between them:

| Token | Text | Example |
|---|---|---|
| `TOKEN_INTERP_STRING` | a whole string with no placeholders | `$"Hello"` |
| `TOKEN_INTERP_START` | from `$"` to the first `{` | `$"Hello {` |
| `TOKEN_INTERP_MIDDLE` | from a placeholder's `}` to the next `{` | `}, {` |
| `TOKEN_INTERP_END` | from the last placeholder's `}` to the closing `"` | `}!"` |

`print $"Hello {"World"}! {b} {n}";` scans as:

```
TOKEN_PRINT
TOKEN_INTERP_START    $"Hello {
TOKEN_STRING          "World"
TOKEN_INTERP_MIDDLE   }! {
TOKEN_IDENTIFIER      b
TOKEN_INTERP_MIDDLE   } {
TOKEN_IDENTIFIER      n
TOKEN_INTERP_END      }"
TOKEN_SEMICOLON
```

Tokens keep pointing into the source, so line numbers stay correct.

In the text, a single `{` starts a placeholder, and a doubled `{{` is literal
text. Inside a placeholder the scanner scans ordinary code, so `"World"` is an
ordinary string. It tracks how many `{` are open inside each placeholder
(`interpBraces` in `Scanner`), so a `}` only ends the placeholder when none
are open: in `{Point { x: 1 }.x}`, the first `}` closes the struct literal
and the second closes the placeholder. Up to 16 interpolated strings can nest
(`MAX_INTERP_DEPTH`); deeper is the error "Interpolated strings nested too
deeply."

`TOKEN_INTERP_END` is needed, rather than ending with an ordinary
`TOKEN_STRING`, because an ordinary string can follow a placeholder's
expression. In `$"{a "b"}"`, a separate end token lets the parser report
"Expect '}' after placeholder expression." at `"b"`. If the end were a
`TOKEN_STRING`, `"b"` would be taken as the end of the string.

#### AST

- **`NODE_INTERP_STRING`** holds an `InterpStringNode`: an arena-allocated
  array of `InterpPart`, in source order.
- **`InterpPart`** is an expression (a string literal for text, or the
  placeholder's expression) and a `StringConversion`.
- **`StringConversion`** is `STRING_CONVERSION_NONE` (already a string),
  `STRING_CONVERSION_NUMBER` (`@numberToString`), `STRING_CONVERSION_BOOL`
  (`@boolToString`), or `STRING_CONVERSION_DISPLAY` (the value's `toString()`).
  The parser sets `NONE`; the type-checker sets the rest.
- **`BinaryNode.isStringConcat`** marks a `+` on two strings, set by the
  type-checker for the compiler's `+` chain lowering.

`$"text"` with no placeholders produces an ordinary string `NODE_LITERAL`.

#### Parser

A prefix rule for `TOKEN_INTERP_START` parses an expression after it and each
`TOKEN_INTERP_MIDDLE`, until `TOKEN_INTERP_END`. Text pieces are unescaped by
the same code as ordinary strings, which also turns `{{` and `}}` into single
braces, and rejects a single `}`, when the text comes from an interpolated
string. The errors are:

- `$"a {} b"`: "Expect expression inside '{}'."
- `$"a {x "b"} c"`: "Expect '}' after placeholder expression."
- `$"a } b"`: "Single '}' in interpolated string. Write '}}' for a literal
  '}'."
- `$"\{a"`: "Invalid escape sequence: \{", as in an ordinary string.

#### Type-checker

`typchkInferInterpString` infers `string` for the node and records each
placeholder's conversion from its type, using `chooseStringConversion`. Any
other type is an error, for example "Can't interpolate [f64]. Only string,
f64, bool, and types that implement Display can be interpolated."

The compiler turns a part into a string with `compileToString`, which compiles
the value followed by its conversion.

A placeholder whose type the checker can't tell is also an error
([Q-unknown-types]):

```
Can't tell the type of this placeholder. Declare it before this line, or give it a type, e.g. 'let n: f64 = ...;'.
```

## Impacts

### Existing Syntax Or Behavior

- `$` is an error character at `from_commit`, so reserving `$`+`"` breaks no
  existing program. A lone `$`, or `$` followed by anything but a quote,
  still errors.
- Ordinary string literals are untouched; no interpolation happens inside a
  plain `"..."`, and `{{` there is just two braces.
- Chains of three or more strings joined with `+` compile to `@strConcat`.
  Their output is unchanged; only their bytecode differs.
- Nothing is added to `stdlib/stdlib.krb`, so no global name can collide with
  user code.
- Each join adds constants for the natives it names, and a function has at
  most 256 constants ("Too many constants in one chunk."). A function with
  many interpolated strings reaches that limit sooner.

### Limitations

- **Placeholders must have a known type** ([Q-unknown-types]). This rejects
  calls to natives with no signature yet, such as `$"{@len(items)}"`, and
  globals used in a function before they're declared in the file.
- **At most 255 parts** per interpolated string ([Part 3]).
- **At most 16 levels** of interpolated strings nested inside placeholders.

### Related Proposals

- [Primitive Impls Proposal] — adds `impl Display for f64` with a `toString`
  method. Once merged, number (and other) placeholders could convert through
  the trait instead of `@numberToString`.
- Display for structs and enums — a placeholder whose type implements the
  built-in `Display` trait converts with its `toString()` method, and so do
  `print` and the `@print` natives. Values whose parts are only known at
  runtime, such as an array of `Display` structs, aren't converted yet: that
  needs a native to call a Kirby method.
- [Generic Types Proposal] — `@strConcat` gets a type signature once arrays
  of unknown element type can be checked. Signatures for natives such as
  `@len` also remove most of the unknown-type limitation.
- [Top-Level Declarations Proposal] — makes globals known before function
  bodies are checked, which removes the rest of the unknown-type limitation
  ([Q-unknown-types]).
- [Macros Proposal] — interpolation could alternatively be implemented as a
  built-in macro. This proposal lowers directly and does not depend on
  macros.
- [Testing Proposal] — acceptance tests for this feature would land with the
  test framework.
- [Debugger Proposal] — interpolated strings compile to native calls, so
  stepping into one never enters stdlib code. The generated instructions carry
  the line of the interpolated string.
- [Tooling Data Proposal] — the generated code should carry the span of the
  interpolated string (or of the placeholder, for code inside `{...}`). A
  string that runs over several lines is currently given the line where its
  first piece _ends_; that proposal changes strings to where they start.
- [Embedded Library Proposal] — interpolation no longer adds anything to the
  stdlib, so it costs new instances nothing.

## Questions

### **Q:** Which construction strategy should the compiler use?

<!-- [Q-strategy]: #q-which-construction-strategy-should-the-compiler-use -->

**Status:** Answered

The candidates were:

- **(a) stdlib `StringBuilder` chain**: `StringBuilder.default().add(...)
  ...toString()`, with a `StringBuilder` struct in `stdlib/stdlib.krb`.
- **(b) `+` chain**: compile `$"a {x} b"` to `"a " + x + " b"`.
- **(c) a new native**: `@strConcat([...])`.
- **(d) a dedicated opcode**: a new instruction that joins strings.

**(a) `StringBuilder`**

- Pros: written in Kirby, with no new native. Matches the issue's suggestion,
  and verified to work at the proposal's original `from_commit`.
- Cons:
  - Puts a `StringBuilder` global in every program, which collides with a
    user's own `StringBuilder` ("Already declared in this scope.").
  - Depends on the stdlib being found and loaded. Without it, no program
    using interpolation can run.
  - Much more work per string: creating the struct, one method call and one
    `@arrPush` per part, then a final `@arrJoin`.
  - Stepping into an interpolated string in a debugger enters stdlib code,
    and a program transpiled to Lua has to carry a translated copy of the
    stdlib.

**(b) `+` chain**

- Pros: no dependencies at all.
- Cons: each `OP_ADD` makes a new string, copying everything built so far,
  so a string with `n` parts copies its start `n - 1` times.

**(c) `@strConcat` native** (chosen)

- Pros:
  - One call; the result is allocated once and each part copied once.
  - No global name, and no dependency on the stdlib.
  - Small, simple bytecode.
  - Also speeds up `+` chains of strings.
- Cons:
  - A new native in every program's namespace. Its name is `@`-prefixed, so
    it can't collide with user code.
  - A native can't call Kirby methods, so conversions must be chosen by the
    compiler. That's workable, including for `Display`.
  - Builds a temporary array for every join.
  - Limited to 255 parts by `OP_ARRAY`.
  - No type signature until generics.

**(d) Opcode**

- Pros: fastest, with no array and no native call.
- Cons: a new opcode is not desirable.

#### Answer

(c): `@strConcat`. It keeps most of (d)'s speed without a new opcode, and
avoids (a)'s global name and stdlib dependency.

### **Q:** Which placeholder types should be accepted?

<!-- [Q-segment-types]: #q-which-placeholder-types-should-be-accepted -->

**Status:** Answered

At the proposal's original `from_commit` the only conversion from a
non-string type to a string was `@numberToString` for `f64`. `bool`, `nil`,
and struct values had none — `Display` impls on primitives are unmerged work
in [Primitive Impls Proposal].

#### Answer

`string`, `f64`, and `bool` for now. `@boolToString` is added for `bool`
([Part 2]).

Update: types that implement the built-in `Display` trait are also accepted,
converting with their `toString()` method.

### **Q:** How do literal braces in segments work?

<!-- [Q-braces]: #q-how-do-literal-braces-in-segments-work -->

**Status:** Answered

Literal segments already reuse ordinary string escapes. A literal brace that
must not start a placeholder needs a rule: (a) new escapes `\{` and `\}`;
(b) doubled braces `{{`/`}}` (Go-style); (c) literal braces in segments are
simply not allowed. (a) is consistent with the existing escape handling in
`string_()`; (b) avoids new escapes but adds a second dialect of string
syntax.

#### Answer

Escaping `\{` and `\}` for now. If `{{expr}}` is supported in the future, it will be a new proposal.

Revised: (b), doubled braces, as in C# and Python. `{{` is a literal `{` and
`}}` is a literal `}`, so `$"{{{n}}}"` wraps a value in braces. `\{` and `\}`
are invalid escapes, as in ordinary strings.

### **Q:** Should a single `}` in literal text be an error?

<!-- [Q-bare-brace]: #q-should-a-single--in-literal-text-be-an-error -->

**Status:** Answered

With doubled braces ([Q-braces]), `}}` means one literal `}`. A single `}`
could still be read as a literal brace, as JavaScript and Swift do with their
own escapes.

#### Answer

It is an error, "Single '}' in interpolated string. Write '}}' for a literal
'}'.", as in C#, Python, and Rust. Since `}}` means one brace, allowing a
single `}` too would make text like `a}}` ambiguous: one brace or two. The
message says how to write a literal brace.

### **Q:** What happens to a placeholder whose type isn't known?

<!-- [Q-unknown-types]: #q-what-happens-to-a-placeholder-whose-type-isnt-known -->

**Status:** Answered

The type-checker allows expressions of unknown type elsewhere, and leaves
them to runtime checks (a `+` of unknown operands compiles to `OP_ADD`,
which the VM checks). A placeholder's conversion has to be chosen at compile
time, though. Unknown types come from natives with no signature yet
(`$"{@len(items)}"`) and from globals used in a function before they're
declared:

```kirby
fun describe(): string = $"later={later}";

let later = "x";
```

Options: (a) reject them with a compile error; (b) pass them to `@strConcat`
unconverted, so a string works and anything else fails at runtime, with an
error that names `@strConcat` though the program never calls it.

#### Answer

(a), with an error that says how to fix it: "Can't tell the type of this
placeholder. Declare it before this line, or give it a type, e.g. 'let n: f64
= ...;'." Accepting more programs later breaks nothing, while going from (b)
to (a) later would. [Generic Types Proposal] and [Top-Level Declarations
Proposal] remove most of these cases.

### **Q:** Is `StringBuilder` the right name for a stdlib global?

<!-- [Q-stdlib-name]: #q-is-stringbuilder-the-right-name-for-a-stdlib-global -->

**Status:** Answered

The original design put `StringBuilder` in every program's global namespace
(the stdlib loads first in every run mode), where a user program that
declares its own `StringBuilder` collides with it.

#### Answer

This is ok right now. I do wonder if how this will work as the language scales, and [[modules]] are added.

No longer applies: the chosen strategy ([Q-strategy]) adds nothing to the
stdlib.

## Glossary

These are both technical and non-technical terms used throughout the
proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Interpolated string**: a `$`-prefixed string literal whose body contains
  literal-text and `{expression}` segments; its value is the concatenation
  of the segments, with each placeholder replaced by the string form of its
  expression's value.
- **Segment**: one element of an interpolated string's body — either literal
  text or a single `{expression}` placeholder.
- **Part**: one element of an `InterpStringNode` after parsing: a text
  segment (empty ones are left out) or a placeholder's expression, each with
  its conversion.
- **Conversion**: how a part becomes a string — none for strings,
  `@numberToString` for `f64`, `@boolToString` for `bool`, and `toString()`
  for a type that implements `Display`.
- **Lowering**: the compiler's rewrite of an interpolated string, or of a `+`
  chain of strings, into a single `@strConcat([...])` call.
- **`+` chain**: a `+` expression whose operands are strings, together with
  any `+` of strings nested in its operands, e.g. `a + (b + c) + d`.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement
[Part 2]: #part-2--strconcat-and-booltostring-natives
[Part 3]: #part-3--compiler-lowering
[Part 4]: #part-4--scanner-parser-and-type-checker-changes

<!-- Proposals -->

[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Testing Proposal]: ../testing/PROPOSAL.md
[Modules]: ../modules/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md

<!-- External -->

[Issue #15]: https://github.com/kirbylang/kirbylang/issues/15
[PR #95]: https://github.com/kirbylang/kirbylang/pull/95

<!-- Questions -->

[Q-strategy]: #q-which-construction-strategy-should-the-compiler-use
[Q-segment-types]: #q-which-placeholder-types-should-be-accepted
[Q-braces]: #q-how-do-literal-braces-in-segments-work
[Q-bare-brace]: #q-should-a-single--in-literal-text-be-an-error
[Q-unknown-types]: #q-what-happens-to-a-placeholder-whose-type-isnt-known
[Q-stdlib-name]: #q-is-stringbuilder-the-right-name-for-a-stdlib-global
