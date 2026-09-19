---
status: Draft
created: 2026-09-19
from_commit: 61b7cc5
---

# Proposal: String Interpolation

This proposal adds string interpolation to Kirby: a `$`-prefixed string
literal whose body can contain `{expression}` placeholders, e.g.
`$"Hello {name}!"`. Per the acceptance criteria of [Issue #15], the design
lowers each interpolated string to a `StringBuilder` (a new type in
`stdlib/stdlib.krb`) that is instantiated and built with the struct-call
bytecode the compiler already has — no new opcode, no new native.

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
  true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c`
  code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Building a string out of parts today means chaining `+` and converting
non-string values by hand:

```kirby
var name = "World";
print "Hello " + name + "!";

var pi = 3.14;
print "pi is " + @numberToString(pi) + " roughly";
```

That works today — `string + string` is accepted by the type-checker
(`typchkInferBinary` in `src/typecheck.c`) and `OP_ADD` concatenates strings
in the VM (`concatenate()` in `src/vm.c`) — but the seams between literal
text and values get buried in operator syntax, and every non-string segment
needs a manual `@numberToString`.

[Issue #15] proposes:

```kirby
var name = "World";

print $"Hello {name}!";
```

The `$` prefix exists so the scanner can tell an interpolated string from an
ordinary one at the start of the token, without having to scan the string
for delimiters.

Today that example fails at scan time, because `$` is not a token in
`src/scanner.c` and falls through to the error default:

```
$ ./build/krb -f test.krb
[line 2] Error: Unexpected character.
```

(verified against a clean build of `from_commit`)

The issue's acceptance criteria: "If it's possible to add the StringBuilder
implementation to stdlib.krb then construct bytecode to instantiate and build
the interpolated string. Or if there is a better way of doing it." The rest
of this proposal checks that route against the implementation and designs
around it, with the construction strategy itself left open in [Q-strategy].

## Proposed Changes

### Part 1 — Syntax

An interpolated string is a `$` immediately followed by a string literal.
The `$` must be directly adjacent to the opening quote (no whitespace); a
lone `$`, or a `$` followed by anything but `"`, remains an "Unexpected
character" error.

The body of an interpolated string is a sequence of **segments**: literal
text and `{expression}` placeholders.

- A placeholder can contain any Kirby expression, so calls, indexing, and
  even closures or struct literals with their own braces are legal:
  `$"count={@len(items)}"`, and an interpolated string can nest inside a
  placeholder of another.
- Literal segments use the same escape rules as ordinary string literals
  (`\n`, `\r`, `\t`, `\"`, `\\` — the unescape in `string_()` in
  `src/parser.c`).
- How a literal brace is written in a segment is [Q-braces].
- Like ordinary strings, a body may span multiple lines (the scanner only
  tracks line numbers, it does not reject newlines).
- An interpolated string is an **expression**: it appears anywhere a string
  literal does (arguments, assignments, `+` operands, return values) and has
  type `string`. `print $"..."` needs no special-casing — the value is a
  plain string.

### Part 2 — `StringBuilder` in `stdlib/stdlib.krb`

`stdlib/stdlib.krb` is currently empty, and it is already loaded before user
code in every run mode (`-r`, `-f`, `-c`) via `runFile("stdlib/stdlib.krb")`
in `src/main.c`. Anything defined there is visible to user programs.

The design adds a `StringBuilder` to that file, modeled on
`examples/stringBuilder.krb`, which was verified to compile and run against
a clean `from_commit` build:

```kirby
// stdlib/stdlib.krb
struct StringBuilder {
    var value: Array;
}

impl StringBuilder {
    pub fun add(self, add: string): Self {
        @arrPush(self.value, add);

        self
    }
}

impl Default for StringBuilder {
    fun default(): Self = Self { value: [] };
}

impl Display for StringBuilder {
    fun toString(self): string = @arrJoin(self.value, "");
}
```

The `Display` and `Default` traits used here are built in — the type-checker
registers `Display`, `Eq`, `Ord`, and `Default` as builtin traits
(`typchkTypeEnvDefineBuiltinTraits` in `src/typecheck.c`).

### Part 3 — Compiler lowering

An interpolated string lowers to a `StringBuilder` chain.
`$"Hello {name}! World {count}"` becomes:

```kirby
StringBuilder.default()
    .add("Hello ")
    .add(name)
    .add("! World ")
    .add(@numberToString(count))
    .toString()
```

- Literal segments become string constants on the same path as ordinary
  literals (`CONST_STRING` + `OP_CONSTANT` in `src/compiler.c`).
- Placeholder expressions compile as ordinary expressions.
- A placeholder whose type is not `string` is wrapped in a conversion. At
  `from_commit`, `f64` is the only non-string type with a conversion to
  string (`@numberToString` in the `src/native.c` registry). Which other
  types should be accepted is [Q-segment-types].
- The type-checker infers `string` for the whole node.

No new opcode or VM change is required: everything in the lowered form
already compiles at `from_commit` — the static call
(`StringBuilder.default()`), the chained instance calls (`.add(...)`,
`OP_INVOKE` in `src/compiler.c`), and the `@`-prefixed native call.
Verified end to end: a file defining the Part 2 type and calling
`StringBuilder.default().add("Hello ").add(name).toString()` compiles,
type-checks, and runs on a clean build — including with the definition in
`stdlib/stdlib.krb` and the use in a separate user file.

### Part 4 — Scanner, parser, and type-checker changes

- **Scanner** (`src/scanner.c`): recognize `$` followed by `"` and scan a new
  token type (tentatively `TOKEN_INTERP_STRING`). The scan runs to the
  closing quote at placeholder depth zero — unescaped `{`/`}` are counted —
  so a placeholder containing a closure or struct literal does not end the
  string, and an unterminated body errors the same way `string()` does
  today. The recognition sits next to the existing `@` prefix rule:

  ```diff
    if (c == '@' && isAlpha(peek(scanner)))
      return identifier(scanner);

  + if (c == '$' && peek(scanner) == '"')
  +   return interpString(scanner);
  +
    if (isDigit(c))
      return number(scanner);
  ```

- **Parser** (`src/parser.c`): a new `NODE_` kind for interpolated strings,
  holding an ordered list of segments. Each segment is either a literal
  (unescaped with the same rules as `string_()`) or a parsed expression.
  Placeholder bodies sit inside the token's text, so the parser re-lexes the
  text between `{` and the matching `}` and parses it as an ordinary
  expression.
- **Type-checker** (`src/typecheck.c`): infer `string` for the new node. No
  other changes — placeholder expressions check as ordinary expressions, and
  any `@numberToString` wrapping checks against the existing native
  signature.

## Impacts

### Existing Syntax Or Behavior

- `$` is an error character today, so reserving `$`+`"` breaks no existing
  program. A lone `$`, or `$` followed by anything but a quote, still errors.
- Ordinary string literals are untouched; no interpolation happens inside a
  plain `"..."`, so existing code with braces in string literals is
  unaffected.
- `stdlib/stdlib.krb` gains a `StringBuilder` global. A user program that
  declares its own top-level `StringBuilder` will now fail with "Already
  declared in this scope." (verified against a clean `from_commit` build
  with the Part 2 definition in stdlib). See [Q-stdlib-name].
- An interpolated string with more than 256 literal segments in one function
  would hit the existing "Too many constants in one chunk." limit
  (`makeConstant` in `src/compiler.c` caps chunk constants at `UINT8_MAX`).
  Not a realistic limit for hand-written code; noted for completeness.

### Related Proposals

- [Primitive Impls Proposal] — that proposal adds `impl Display for f64`
  with a `toString` method. Once merged, f64 (and eventually other) segments
  could convert through the trait instead of `@numberToString`. This
  proposal uses `@numberToString` in the meantime.
- [Macros Proposal] — interpolation could alternatively be implemented as a
  built-in macro emitting the Part 3 chain. This proposal takes the direct
  lowering route and does not depend on macros.
- [Collection Methods Proposal] — the Part 2 `StringBuilder` uses `@arrPush`
  on its internal `Array`. If array methods land, the stdlib definition could
  switch to a method call.
- [Testing Proposal] — acceptance tests for this feature (scanning, parsing,
  lowering, runtime output) would land with the test framework.

## Questions

### **Q:** Which construction strategy should the compiler use?

<!-- [Q-strategy]: #q-which-construction-strategy-should-the-compiler-use -->

**Status:** Open

The issue's acceptance criteria name a stdlib `StringBuilder` but leave the
door open to "a better way." The candidates:

- **(a) stdlib `StringBuilder` chain** — the Part 2/Part 3 design. N pushes
  (amortized) plus one final `@arrJoin` allocation. Requires the stdlib
  definition to land with this feature. Verified end to end at
  `from_commit`.
- **(b) `+` chain** — compile `$"a {x} b"` to `CONST "a " + x + CONST " b"
`. Zero stdlib dependency, but each `OP_ADD` allocates a fresh string, so
  a long interpolation copies the whole accumulated result per segment.
- **(c) a dedicated opcode or new native** — a `@strConcat`-style native was
  considered and set aside: none exists at `from_commit`, and adding one is
  not part of this proposal's direction. A dedicated opcode remains a
  possible later optimization if profiling ever calls for it.

The lean is (a): it is what the acceptance criteria describe, and it is the
route verified to work. (b) stays as the fallback if the stdlib definition
turns out to be unwanted.

### **Q:** Which placeholder types should be accepted?

<!-- [Q-segment-types]: #q-which-placeholder-types-should-be-accepted -->

**Status:** Open

At `from_commit` the only conversion from a non-string type to a string is
`@numberToString` for `f64`. `bool`, `nil`, and struct values have none —
`Display` impls on primitives are unmerged work in [Primitive Impls
Proposal].

Options: (a) accept `string` and `f64` placeholders now, and type-error on
everything else; (b) accept only `string` now and open up other types as
conversions become available (e.g. `impl Display for bool` once primitive
impls land). (a) is simpler and more honest about what the compiler can
lower today.

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

### **Q:** Is `StringBuilder` the right name for a stdlib global?

<!-- [Q-stdlib-name]: #q-is-stringbuilder-the-right-name-for-a-stdlib-global -->

**Status:** Answered

Part 2 puts `StringBuilder` in every program's global namespace (the stdlib
loads first in every run mode), and a user program that declares its own
`StringBuilder` collides with it (verified: "Already declared in this
scope."). The name follows the existing example, but there is no stdlib
naming convention to follow yet — the file is empty. Worth settling before
the definition lands, since renaming later is a breaking change for anyone
who wrote against it.

#### Answer

This is ok right now. I do wonder if how this will work as the language scales, and [[modules]] are added.

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
- **Lowering**: the compiler's rewrite of an interpolated string into the
  equivalent `StringBuilder.default().add(...)...toString()` expression.
- **StringBuilder**: the struct from Part 2, in `stdlib/stdlib.krb`, that
  accumulates string segments in an `Array` and joins them in `toString`.
  Modeled on `examples/stringBuilder.krb`.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement

<!-- Proposals -->

[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Testing Proposal]: ../testing/PROPOSAL.md
[Modules]: ../modules/PROPOSAL.md

<!-- External -->

[Issue #15]: https://github.com/kirbylang/kirbylang/issues/15

<!-- Questions -->

[Q-strategy]: #q-which-construction-strategy-should-the-compiler-use
[Q-segment-types]: #q-which-placeholder-types-should-be-accepted
[Q-braces]: #q-how-do-literal-braces-in-segments-work
[Q-stdlib-name]: #q-is-stringbuilder-the-right-name-for-a-stdlib-global
