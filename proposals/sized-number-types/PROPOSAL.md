---
status: Draft
created: 2026-09-18
from_commit: b56a2a9
---

# Proposal: Sized Number Types

This proposal adds `u8`, `u16`, `i32`, and `i64` alongside Kirby's existing
`f64`, with implicit widening between compatible kinds, explicit narrowing
the other direction, and a literal suffix (`10u8`) to pin a literal's kind
at the token itself.

This is a **living document**: it captures what's already decided and lays
out, as [Questions], the parts that aren't. Following the project's
convention, when this document says Kirby "does" or "has" something, that
is a checked fact about `main` (commit `b56a2a9`). When it says Kirby
"should" or "will" do something, that is what this proposal asks for.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. Most of what
isn't yet settled is tracked as [Questions] rather than decided in the
prose.

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it is
defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true
  for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that
  is true after the changes presented here.

### Code & Changes

- C code from the language's implementation is shown in `c` blocks.
- Kirby code — including syntax this proposal asks for, which does not
  compile on `main` — is shown in `kirby` blocks.

---

## Problem Statement

Kirby has exactly one numeric type today, and that fact runs deeper than
"there's only one `TypeKind` for it" — the single type is load-bearing in
several places that would all need to change together.

**One runtime representation.** Every number, regardless of how it's
declared or used, is a plain C `double` (`src/value.h`):

```c
typedef enum { VAL_BOOL, VAL_NIL, VAL_NUMBER, VAL_OBJ } ValueType;

typedef struct {
  ValueType type;
  union {
    bool boolean;
    double number;
    Obj *obj;
  } as;
} Value;
```

There is no width or signedness tag anywhere at runtime — `VAL_NUMBER` is
`VAL_NUMBER`, full stop.

**One type-checker type.** `TypeKind` (`src/types.h`) has exactly one
numeric member, `TYPE_F64`, and `typesEqual` treats it (like `bool` and
`string`) as trivially equal to itself and nothing else — there's no
"numeric-ish" category, only exact identity.

**Every arithmetic and comparison site hardcodes `f64` individually,
rather than checking against a shared "is numeric" predicate.** Unary
negate:

```c
if (!typchkCheck(env, u->operand, typeF64()))
  return NULL;
return typeF64();
```

`+` (which also special-cases `string`), `- * / %` (checked together), and
the four comparison operators are three more separate hardcoded blocks in
`typchkInferBinary` (`src/typecheck.c`), each independently written against
`typeF64()`. Literal inference is a fourth site:

```c
case LITERAL_NUMBER:
  return typeF64();
```

Adding a second numeric kind is therefore not "add a `TypeKind` value" in
isolation — it's a change to the equality-based checking model these five
sites all share today, since none of them currently distinguishes "the
right kind" from "some numeric kind."

**No literal suffix syntax exists**, and the scanner's `number()` doesn't
look for one — it consumes digits and an optional fractional part and
stops (`src/scanner.c`). Confirmed on a clean build:

```shell
$ echo 'print 10u8;' > /tmp/suffix.krb && ./build/krb -f /tmp/suffix.krb
[line 1] Error at 'u8': Expect ';' after value.
```

`10` is scanned as a complete `NUMBER` token, `u8` is scanned separately as
an `IDENTIFIER`, and the parser chokes on the second token immediately
after the first expression — not because `u8` is a bad suffix, but because
nothing tells the scanner a suffix is a thing.

**No new type name is recognized either.** `typchkResolveType`
(`src/typecheck.c`) matches type-annotation tokens against a fixed list —
`unit`, `bool`, `string`, `f64`, `Array`, `Self`, then structs and aliases —
and falls through to "Unknown type." for anything else:

```shell
$ echo 'let x: u8 = 5;' > /tmp/u8type.krb && ./build/krb -f /tmp/u8type.krb
[line 1] Error at 'u8': Unknown type.
```

**A third, independent enumeration of primitive kinds already exists** in
`src/native_signatures.h`, alongside `TypeKind` and the
`tokenIsPrimitiveTypeName` text-matching in `typecheck.c`:

```c
typedef enum {
  NATIVE_UNIT,
  NATIVE_BOOL,
  NATIVE_STRING,
  NATIVE_F64,
  NATIVE_LIST,
} NativePrimitive;
```

Any new numeric kind that natives should type-check against (`ceil`,
`numberToString`, and whatever the [Additional Native Functions Proposal]
adds) needs a place in this list too, independent of `TypeKind`.

**Printing doesn't distinguish "whole" from "fractional" today**, which
matters once integer-flavored kinds exist. Every number, `f64` or
otherwise, prints via `%f` (`src/value.c`):

```c
case VAL_NUMBER:
  snprintf(buffer, size, "%f", AS_NUMBER(value));
  break;
```

so `print 1234567890;` prints `1234567890.000000` today (confirmed by
`tests/primitives/number.krb`), and `docs/TYPES.md`'s own `Display`
example shows `numberToString` producing `"123.000000"`. A `u8` printing
`3.000000` would read strangely to anyone coming from another language.

**There is no conversion syntax between primitives at all** — no `as`
keyword, no cast functions. The closest existing thing, `numberToString` /
`parseNumber`, converts between `string` and the one numeric type, not
between numeric kinds. Explicit narrowing (`u16` → `u8`) therefore isn't
"use the existing cast mechanism with a new target type" — the mechanism
itself doesn't exist yet.

**This isn't a surprise to the project.** `docs/CHANGELOG.md`'s Future
State section already names it —

```
- Number types
  - `u#` (e.g. `123u8`), `i#`, `f#`, `number`
```

— and there's an open tracking issue, [Issue #63], titled "Sized number
types," whose own notes already show the central tension this proposal has
to resolve:

> Current idea: The size is compile time enforced with the runtime value
> being f64

against the type-system tracking issue, [Issue #17], which frames the same
idea slightly differently:

> I think the underlying number being represented as `f64` but compile
> time **and runtime** checked for size.

"Compile-time only" and "compile-time and runtime" aren't the same
design — see [Q-runtime-repr].

The [Primitive Impls Proposal] was written anticipating this one: its
glossary defines "primitive kind" instead of hardcoding `f64` specifically,
and its own [Q-numeric-family] question was left open pending "a
numeric-type-family proposal." This is that proposal.

## Proposed Changes

- **Four new numeric kinds join `f64`:** `u8`, `u16`, `i32`, `i64`. Each is
  its own type, not an alias — `let x: u8 = 5;` and `let y: u16 = 5;`
  produce values of different, non-interchangeable types, the same way
  `f64` and `bool` are non-interchangeable today. Whether exactly this set
  of five is the right set to ship first is [Q-widths-in-scope].
- **A literal suffix pins a numeric literal's kind at the token**: `10u8`,
  `500u16`, `2i32`, `9000000000i64`. An unsuffixed literal's kind is
  decided some other way — see [Q-unsuffixed-literal-inference], since
  today's literal inference (always `typeF64()`) doesn't yet have a notion
  of "ask the surrounding context." The exact grammar of a suffix — where
  digits stop and the suffix starts, and whether a fractional literal can
  carry one — is [Q-literal-suffix-grammar].
- **Widening is implicit.** A `u8` value flows into a `u16`-typed slot
  (assignment, parameter, return, comparison) with no explicit conversion
  needed, because widening a smaller unsigned kind into a larger one can
  never lose information. The full graph of which kinds implicitly widen
  into which others — including whether the signed kinds widen the same
  way, and what (if anything) widens into `f64` — is [Q-widening-graph].
- **Narrowing is explicit.** Going the other direction (`u16` → `u8`) is
  never implicit; a program that wants it has to say so. What syntax says
  so, and what happens when the value in hand doesn't actually fit in the
  narrower kind, is [Q-narrowing] — genuinely open, and the reason this is
  a living document rather than a finished design.
- **Whether the runtime carries the distinction at all is open.** Every
  numeric kind could keep sharing today's single `VAL_NUMBER`/`double`
  representation, with size and signedness purely a compile-time fiction
  the type checker enforces and the VM never sees again — or a kind could
  become a real runtime property a value carries. This is [Q-runtime-repr],
  and it's this proposal's biggest open question: it decides how big the
  implementation is, and it decides what "narrowing failed" can even mean
  at runtime.

Two consequences of adding non-`f64` numeric kinds that this proposal
doesn't have a full answer for yet, flagged rather than silently
assumed away:

- Once `typchkInferBinary`'s per-operator blocks stop being able to say
  "both sides `f64`," something has to say what a mixed-kind expression
  like `someU8 + someU16` means — [Q-operator-result-kind].
- `native_signatures.h`'s `NativePrimitive` enum, the numeric-only native
  functions it types (`ceil`, and whatever [Additional Native Functions
  Proposal] adds), and `f64`'s three builtin trait impls (`Display`, `Eq`,
  `Ord` — real once [Primitive Impls Proposal] lands, `Default` already
  spelled in `docs/TYPES.md`) all currently assume "the one numeric type."
  Whether extending all of that to the new kinds is in this proposal's
  scope or deferred to those proposals is [Q-scope-native-and-traits].

## Related Proposals

- [Primitive Impls Proposal] — its `Q-numeric-family` question explicitly
  deferred to "a numeric-type-family proposal," which is this one. Once
  numeric kinds beyond `f64` exist, that proposal's `impl`/`impl Trait for`
  design needs to reach all of them, not just `f64` — see
  [Q-scope-native-and-traits].
- [Generic Types, Macros, and Tooling Proposal][generic-types] — Part 1.1
  documents "one numeric type, `f64`" as today's checked baseline; that
  stops being true once this proposal lands. Its Part 4.4/10.3 discussion
  of primitive operator dispatch is also numeric-kind-shaped once `+`
  isn't just "two `f64`s."
- [Additional Native Functions Proposal] — proposes `floor`, `round`,
  `trunc`, `abs`, `sqrt`, `pow`, `min`, `max`, all typed `f64` in and
  `f64` out. If those land before this proposal's kinds do, their
  signatures are one more thing [Q-scope-native-and-traits] has to decide
  whether to revisit.

## Questions

### **Q:** What happens when an explicit narrowing conversion doesn't fit?

[Q-narrowing]: #q-what-happens-when-an-explicit-narrowing-conversion-doesnt-fit

**Status:** Open

Kirby has no conversion syntax between any two primitive types today (see
[Problem Statement]), so this question is really two: what spells "convert
this `u16` to a `u8`," and what happens when the value doesn't fit in the
target (`300u16` narrowed to `u8`, which only holds 0–255).

**Syntax options:** (a) a cast-like construct new to the language — an
`as` keyword (`x as u8`) or a call-like form (`u8(x)`); either is a first
for Kirby, which currently converts between types only via named natives
like `numberToString`/`parseNumber`. (b) a named native per target kind
(`toU8(x)`), consistent with the existing native-function convention and
requiring no new syntax at all, at the cost of one more native per kind.

**Failure options:** (a) wrap (C's own truncating/modular behavior for
narrowing integer conversions, cheapest to implement and the most
"invisible" failure mode); (b) a runtime error, consistent with how every
other invalid-input case in `native.c`/the VM is handled today (division by
zero, `assertPositiveNumber`, etc. all raise rather than silently
producing a different number); (c) saturate to the target kind's min/max.

**Why open:** this is the question the user request that started this
proposal asked directly, and it doesn't have an existing Kirby precedent to
settle it — the language has never had a lossy conversion of any kind
before. (b) reads as the most consistent choice given every other "this
input doesn't work" case in the codebase already raises rather than
guessing, but it depends on [Q-runtime-repr]: a runtime error requires the
VM to know, at the moment of narrowing, what value it actually has to
check against 0–255, which is trivial if kinds are real at runtime and
requires re-deriving the source value's kind some other way if they
aren't.

### **Q:** Does the runtime represent numeric kinds, or is sizing purely a compile-time fiction?

[Q-runtime-repr]: #q-does-the-runtime-represent-numeric-kinds-or-is-sizing-purely-a-compile-time-fiction

**Status:** Open

Every `Value` is `VAL_NUMBER` wrapping a `double`, with no room today for a
width or signedness tag (see [Problem Statement]). [Issue #63]'s own note
leans toward keeping it that way ("the runtime value being f64"); [Issue
#17] frames it as "compile time and runtime checked for size" — the two
aren't quite the same design, and neither issue resolves it.

**Options:** (a) purely compile-time: every numeric kind still compiles
down to the same `VAL_NUMBER`/`double`, and the type checker is the only
thing that ever knows a value is a `u8` rather than an `f64`. Cheapest by
far — no `Value` layout change, no VM change — but a value that escapes
static checking (e.g. crosses through `Array`, which drops element types
today per `docs/TYPES.md`) has no way to be checked or even
`typeof`'d at runtime, and [Q-narrowing]'s "runtime error" option has
nothing to check against once execution starts. (b) real runtime
representation: `Value` grows a kind tag (a few new `ValueType`
variants, or a sub-tag alongside `VAL_NUMBER`), so `typeof`, narrowing
checks, and anything crossing an `Array` boundary can all ask "what kind is
this, really." More invasive — every `BINARY_OP`/`AS_NUMBER` site in
`src/vm.c` currently assumes a bare `double` — but the only option that
makes a runtime narrowing error (rather than a wrap) fully honest.

**Why open:** this is the question the user request driving this proposal
asked directly, and it's the one everything else in this document is
downstream of: it decides how big the implementation is, and it decides
which of [Q-narrowing]'s failure-mode options are even implementable
faithfully. No recommendation is made here on purpose.

### **Q:** Which widths and signs ship first?

[Q-widths-in-scope]: #q-which-widths-and-signs-ship-first

**Status:** Open

The user request behind this document names `u8`, `u16`, `i32`, `i64`.
[Issue #63]'s own examples are `u8`, `u16`, `i64` (no `i32`). Neither list
mentions `i8`, `u32`, `u64`, or an architecture-sized `usize`/`isize`,
which most languages with sized integers eventually want.

**Options:** (a) ship exactly `u8`, `u16`, `i32`, `i64` (plus existing
`f64`) as a first, deliberately incomplete family — matches the user
request as given, and every design decision elsewhere in this document
(widening graph, suffix grammar, runtime representation) is written to
generalize to more kinds later without redesign; (b) fill out the full
conventional set (`i8`/`u8`/`i16`/`u16`/`i32`/`u32`/`i64`/`u64`) up front,
so the widening graph is complete on day one instead of growing awkward
gaps (e.g. `u16` widens to `u32`... which doesn't exist yet).

**Why open:** low-stakes relative to [Q-narrowing]/[Q-runtime-repr], but
worth pinning down before [Q-widening-graph] is answered, since the graph's
shape depends on exactly which kinds are in it. Recommendation: (a) — the
user request is explicit about this set, and every other kind is strictly
additive later provided [Q-runtime-repr] doesn't get answered in a way
that special-cases today's four.

### **Q:** What is the complete implicit widening graph?

[Q-widening-graph]: #q-what-is-the-complete-implicit-widening-graph

**Status:** Open

The user request names one edge explicitly (`u8` → `u16` widens
implicitly) and one direction generally (narrowing is always explicit).
Left unstated: whether `i32` → `i64` widens the same way (presumably yes,
by the same lossless-conversion logic); whether any unsigned kind widens
implicitly into a signed one (`u8` → `i32` is lossless for every `u8`
value, `u16` → `i32` also is, but `u16` → `i16` — if `i16` ever exists —
would not be); and whether any integer kind widens implicitly into `f64`,
which is a different kind of "lossless" (every `i64` value is
representable as a nearby `f64`, but not always exactly, since `f64` only
has 53 bits of exact integer precision against `i64`'s 63).

**Options:** (a) widen within same-signedness families only
(`u8`→`u16`, `i32`→`i64`), and treat every cross-signedness or
integer-to-`f64` conversion as requiring the explicit narrowing syntax
from [Q-narrowing] even where it happens to be lossless — simplest rule,
errs toward requiring more explicit conversions than strictly necessary;
(b) widen wherever a conversion is provably lossless for every value of
the source kind (unsigned-to-larger-signed included), and require an
explicit conversion only where some value could lose information —
matches the general meaning of "widening" more closely, but `i64`→`f64`
sits right at the edge of "provably lossless" and needs its own explicit
call in whichever direction this proposal picks.

**Why open:** this proposal's driving request establishes the pattern
(`u8`→`u16`) but not the general rule, and the general rule is what an
implementation actually needs before it can accept or reject a single
mixed-kind expression.

### **Q:** What is the exact grammar of a literal suffix?

[Q-literal-suffix-grammar]: #q-what-is-the-exact-grammar-of-a-literal-suffix

**Status:** Open

`scanner.c`'s `number()` currently stops scanning the moment it sees a
non-digit, non-`.` character, so a suffix has to be scanned as part of the
same token deliberately, not fall out of existing behavior. Two things
need pinning down: whether a suffix is legal on a fractional literal
(`1234.10f64`, per [Issue #63]'s own example — meaning `f64` needs a
spellable suffix too, not just the four new kinds), and what happens when
the letters after a number aren't one of the recognized suffixes (is
`10x` a scan error at the literal, or does it scan as `10` followed by an
`x` identifier the way it does today, just with `u8`/`u16`/`i32`/`i64`/`f64`
carved out as special cases?).

**Options:** (a) suffixes apply to both integer and fractional literals
uniformly, and any letter sequence immediately following a number's digits
that isn't a recognized suffix is a scan error — most consistent, closes
off silently-wrong programs like a typo'd suffix falling back to being
read as two separate tokens; (b) suffixes apply only to integer-literal
syntax (no `.`), since `1234.10u8` describing a whole-number kind reads as
contradictory on its face, and non-suffix letters after a number keep
today's fallback-to-identifier behavior, accepting that a typo'd suffix
becomes a confusing "Expect ';' after value" error like the one in
[Problem Statement] rather than a clearer one.

**Why open:** purely a syntax-design question with no existing precedent
in the scanner to defer to; either answer is implementable, and the choice
mostly affects how good the error message is for a typo'd suffix.

### **Q:** How does an unsuffixed numeric literal get a concrete kind?

[Q-unsuffixed-literal-inference]: #q-how-does-an-unsuffixed-numeric-literal-get-a-concrete-kind

**Status:** Open

`typchkInferLiteral` always returns `typeF64()` for `LITERAL_NUMBER` today,
which is why `let age: f64 = 100;` works — the literal's inferred type
happens to equal the annotation's. That stops working once `100` could
mean five different things depending on where it's written. Bidirectional
checking already exists for two node kinds — `typchkCheck` special-cases
`NODE_ARRAY` (each element is checked against the array's element type,
not just inferred and compared) and lambda parameters — but numeric
literals have no such special case; they're always inferred first, then
compared.

**Options:** (a) extend `typchkCheck`'s existing special-casing to
`LITERAL_NUMBER`: when a numeric literal is checked against an expected
numeric kind, it takes that kind directly, the same way an array literal
already takes its element type from context; falls back to some default
(see (c)) only when there's no expected type to check against (e.g. a bare
`print 100;`). This is the smallest addition, since the mechanism it
extends already exists for a different node kind. (b) every unsuffixed
literal is `f64`, full stop — matches today's behavior exactly and never
needs context, but means `let x: u8 = 100;` would need `100` written as
`100u8` even though the context already says `u8`, which reads as
needlessly verbose. (c) whatever the default is when there's no context
(a bare `print 100;`, or `let x = 100;` with no annotation) needs its own
answer regardless of (a) vs (b) — most likely `f64`, continuing today's
behavior, but that's worth saying outright rather than leaving implicit.

**Why open:** (a) is the more ambitious but more ergonomic answer and
needs the `typchkCheck`/`typchkInfer` split to grow one more special case;
(b) is nearly free but reintroduces exactly the suffix-on-every-literal
verbosity this proposal's suffix syntax was meant to avoid needing in
annotated contexts. No recommendation made here since it trades
implementation cost against ergonomics in a way this document can't settle
unilaterally.

### **Q:** What does a mixed-kind arithmetic or comparison expression mean?

[Q-operator-result-kind]: #q-what-does-a-mixed-kind-arithmetic-or-comparison-expression-mean

**Status:** Open

Today, `typchkInferBinary`'s arithmetic and comparison cases each check
"both sides `f64`" as one condition and return one fixed result type. Once
"both sides the same numeric kind" is no longer the only legal shape (per
[Q-widening-graph], a `u8` may be allowed to widen into a `u16` context),
something has to decide what `someU8 + someU16` produces, and separately,
whether it's even legal without an explicit narrowing first.

**Options:** (a) mixed-kind operands are legal exactly when one side
widens into the other per [Q-widening-graph], and the result is the wider
of the two kinds — the usual behavior in languages with implicit
widening; (b) every binary operator requires both operands to already be
the exact same kind, and mixing kinds — even along a legal widening
edge — is a type error requiring an explicit widen or narrow at the call
site first; widening being implicit would then only apply to
assignment/parameter/return positions, not to operator operands directly.

**Why open:** (a) is more convenient to write but means `typchkInferBinary`
needs a "wider of these two kinds" helper that doesn't exist in any form
today; (b) is a smaller change to the existing per-operator blocks (each
still checks "both sides equal to some numeric kind," just no longer
hardcoded to `f64` specifically) but pushes more explicit conversions into
ordinary arithmetic than a user coming from most other typed languages
would expect.

### **Q:** Do native function signatures and builtin trait impls extend to the new kinds in this proposal, or later?

[Q-scope-native-and-traits]: #q-do-native-function-signatures-and-builtin-trait-impls-extend-to-the-new-kinds-in-this-proposal-or-later

**Status:** Open

`native_signatures.h`'s `NativePrimitive` enum, every native function
typed against `NATIVE_F64` (`ceil` today; `floor`/`round`/`sqrt`/etc. per
the [Additional Native Functions Proposal] if it lands), and `f64`'s
builtin trait coverage (`Display`/`Eq`/`Ord` once real per
[Primitive Impls Proposal], `Default` already documented) are all written
against "the one numeric type" as a live assumption, not just as an
artifact of there being nothing else to write them against.

**Options:** (a) this proposal's scope includes updating
`NativePrimitive` and every existing `f64`-typed native signature to also
accept/return the new kinds (likely via the same "wider of these" logic as
[Q-operator-result-kind], or by requiring an explicit widen to `f64` at
the call site) — keeps the language internally consistent the moment this
proposal lands, at the cost of a noticeably larger implementation; (b)
this proposal ships the type system and literal syntax only, and updating
natives/trait impls to cover the new kinds is explicitly deferred to
follow-up work (tracked against [Primitive Impls Proposal] and
[Additional Native Functions Proposal] respectively) — smaller, faster to
land, but leaves `ceil(someU8)` needing an explicit widen to `f64` first,
indefinitely, until that follow-up happens.

**Why open:** genuinely a sequencing question, not a design disagreement —
recorded here so the three proposals don't each assume a different answer
independently. Recommendation: (b), on the same reasoning
[Q-sequencing] in [Primitive Impls Proposal] uses for a similar
question — shipping the smaller piece first doesn't foreclose the larger
one.

## Glossary

- **Changes**: Changes refer to the proposed changes in this document.
- **Numeric kind**: one of Kirby's numeric primitive types — `f64` today,
  plus `u8`, `u16`, `i32`, and `i64` per this proposal (see
  [Q-widths-in-scope] for whether more join later). Deliberately narrower
  than [Primitive Impls Proposal]'s "primitive kind," which also covers
  `bool` and `string`.
- **Widening**: an implicit conversion from a numeric kind to another kind
  that can represent every value the first kind can, used automatically at
  assignment, parameter-passing, and (per [Q-operator-result-kind]) maybe
  operator operands.
- **Narrowing**: a conversion from a numeric kind to one that cannot
  represent every value the first kind can. Never implicit under this
  proposal; see [Q-narrowing] for what makes it happen and what happens
  when the source value doesn't actually fit.
- **Suffix**: the letters immediately following a numeric literal's digits
  that pin the literal's kind at the token itself, e.g. `u8` in `10u8`.
  See [Q-literal-suffix-grammar] for the exact grammar.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Q-numeric-family]: ../primitive-impls/PROPOSAL.md#q-should-the-design-anticipate-more-numeric-primitive-kinds-now-or-wait-for-that-proposal
[Q-sequencing]: ../primitive-impls/PROPOSAL.md#q-should-this-land-before-after-or-independent-of-the-generic-types-proposals-operator-dispatch-decision
[generic-types]: ../generic-types/PROPOSAL.md
[Additional Native Functions Proposal]: ../additional-native-functions/PROPOSAL.md

<!-- External references -->

[Issue #63]: https://github.com/kirbylang/kirbylang/issues/63
[Issue #17]: https://github.com/kirbylang/kirbylang/issues/17

<!-- Questions -->

[Q-narrowing]: #q-what-happens-when-an-explicit-narrowing-conversion-doesnt-fit
[Q-runtime-repr]: #q-does-the-runtime-represent-numeric-kinds-or-is-sizing-purely-a-compile-time-fiction
[Q-widths-in-scope]: #q-which-widths-and-signs-ship-first
[Q-widening-graph]: #q-what-is-the-complete-implicit-widening-graph
[Q-literal-suffix-grammar]: #q-what-is-the-exact-grammar-of-a-literal-suffix
[Q-unsuffixed-literal-inference]: #q-how-does-an-unsuffixed-numeric-literal-get-a-concrete-kind
[Q-operator-result-kind]: #q-what-does-a-mixed-kind-arithmetic-or-comparison-expression-mean
[Q-scope-native-and-traits]: #q-do-native-function-signatures-and-builtin-trait-impls-extend-to-the-new-kinds-in-this-proposal-or-later
