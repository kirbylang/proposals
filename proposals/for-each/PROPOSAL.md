---
status: Draft
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: For-Each Loops

This proposal adds a loop that visits each item of a collection: `for (item in items) { ... }`. Today the only `for` loop is the C-style one with a counter, so reading an array means keeping an index, asking for the length and indexing by hand. The proposal starts with arrays, which need no other proposal, and shows how ranges, strings, maps and user types can follow through one rule.

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

The `for` statement has one form (`forStatement` in `src/parser.c`): an initializer, a condition and an increment, separated by semicolons.

```c
consume(p, TOKEN_LEFT_PAREN, "Expect '(' after 'for'.");
...
consume(p, TOKEN_SEMICOLON, "Expect ';' after loop condition.");
...
consume(p, TOKEN_RIGHT_PAREN, "Expect ')' after for clauses.");
```

Visiting every item of an array takes a counter, the length, and an index on every read:

```kirby
let names: [string] = ["a", "b", "c"];

for (var i = 0; i < @arrLen(names); i = i + 1) {
  @println(names[i]);
}
```

The program says three things (a counter, a limit, a step) for one idea ("each name"), and the counter can be got wrong. The form people try first does not work. Verified against a build of `from_commit`:

```kirby
for (name in names) { @println(name); }
// [line 1] Error at 'name': Expect ';' after expression.
```

Other facts that shape the design:

- **`in` is not a keyword.** `var in = 1; @println(in);` prints `1`.
- **There is no range.** `0..3` is not valid: `@println(1..2);` reports `Expect property name after '.'.` A counted loop needs the C-style form.
- **The parser can look two tokens ahead.** `tsPeek` and `tsPeekNext` are in `src/token_stream.h`. `for (name in` can be told from `for (var name` or `for (i = 0` by looking at the token after the name.
- **The loop node has four parts.** `NODE_FOR` holds `init`, `condition`, `increment` and `body` (`src/ast.h`, `src/parser.c`), and `break` and `continue` already work in it (`tests/run/flow_control/for_continue.krb`).
- **Arrays can change while read.** `@arrPush`, `@arrPop`, `@arrInsert`, `@arrRemove` and `@arrClear` all change an array in place, so a loop over an array has to say what happens if the body does.

Collections that are coming make the gap bigger. The [Map Proposal] has no way to look at all entries without `@mapKeys`, the [String Encoding Proposal] needs a character by character walk that is easy to get wrong, and the [Tuples Proposal] and [Destructuring Proposal] design patterns that a loop should be able to use.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- `for (item in collection) body` for arrays
- `break` and `continue` inside it
- What happens if the body changes the array
- A path to ranges, strings, maps and user types through one rule

The following is intentionally left out of scope for this proposal:

- Changing or removing the C-style `for` loop
- `while let` and loops that use patterns to stop
- List comprehensions. See the future state list in `docs/CHANGELOG.md`.
- Parallel iteration, `zip`, `enumerate`, and other helpers on iterators
- Iterating while removing items

### Part A: Arrays

```kirby
for (name in names) {
  @println(name);
}
```

- The collection expression is evaluated once, before the loop.
- `name` is a new variable for each pass, with the type of the array's items. It lives inside the loop only.
- The loop ends after the last item. An empty array runs the body zero times.
- `break` leaves the loop, and `continue` goes to the next item. Both mean what they mean in the C-style form.
- The collection must be an array. Anything else is a compile error that names the type: `Can't loop over a value of type f64.`
- The loop is an ordinary statement and has no value.

The variable can not be assigned to unless it is declared with `var` ([Q-binding]).

### Part B: Ranges

A counted loop is the most common use of the C-style form. A range would cover it:

```kirby
for (i in 0..3) {
  @println(i);   // 0, 1, 2
}
```

`0..3` stands for the whole numbers from 0 up to but not including 3. A range needs a `..` token, and a rule so `1..2` is not read as the number `1.` followed by `.2`. It is listed as a separate part because it adds syntax outside the loop. See [Q-ranges].

### Part C: One rule for everything else

Strings, maps and types written by users should not each get their own special case. The rule: a loop over a value works when the value's type can produce its items one at a time. For an array, the compiler knows how and can use the counter it would otherwise write by hand. For other types, a trait says how:

```kirby
trait Iterator[T] {
  fun next(self): Option[T];
}

trait Iterable[T] {
  fun iter(self): Iterator[T];
}
```

- `for (x in v)` calls `v.iter()` once, then calls `next()` on the result until it gives `None`.
- A user type implements `Iterable` to be loopable, and the iterator it returns can be a separate struct that remembers where it is.
- A map would give a pair of key and value. With a pattern in place of the name, `for ((key, value) in map)` takes the pair apart ([Q-pattern]).
- A string gives what [Q-string-iteration] decides.

This part depends on generic traits and `Option[T]`. Parts A and B need neither, which is why they come first and why the array loop does not go through `Iterator`.

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Scanner and parser

- Whether `in` becomes a keyword is [Q-keyword]. If it does, a `TOKEN_IN` in `src/token.h` and a case in `identifierType` in `src/scanner.c`. If not, the parser checks the token text.
- `forStatement` looks at the token after `(`: a name followed by `in` is a for-each loop. Otherwise it goes on as today.
- A new AST node, `NODE_FOR_EACH`, with the binding, the collection and the body.

#### Part 2: Type checker

- The collection is checked to be an array, and the binding gets the item type.
- A binding without `var` is not assignable, and the existing "assigning to a `let`" error is reused.

#### Part 3: Compiler

- The loop compiles to the C-style shape with two hidden locals: the array and the index. The condition compares the index to the array's length each time, reads the item into the binding, runs the body, and adds one.
- `continue` jumps to the step that adds one, as it does in the C-style loop.
- The hidden locals are the same kind as the ones the [Destructuring Proposal] and [Pattern Matching Proposal] need, and the [Tooling Data Proposal] and [Debugger Proposal] should not show them.
- No new instruction is expected. If reading the length needs one, it is a small addition.

#### Part 4: Ranges (B)

- `..` in the scanner, the number scanner rule, a range expression or a special case in the loop. Loops over a range compile to the counted loop with no array made.

#### Part 5: Iterator traits (C)

- The two traits and the call of `iter()` and `next()` in the compiled loop. Needs generics ([Generic Types Proposal]) and enums ([Enums Proposal]).

#### Part 6: Docs and tooling

- `docs/TYPES.md` or a flow control page, the changelog, `in` in the VS Code grammar.

## Impacts

### Existing Syntax Or Behavior

- The C-style `for` is unchanged.
- If `in` is reserved ([Q-keyword]), a program that uses `in` as a name stops compiling. Nothing in this repository does.
- `for (x in y)` was a syntax error and now means something.

### Related Proposals

- [Map Proposal] — the loop is how a program visits all entries. A map yields pairs.
- [String Encoding Proposal] and [Strings Proposal] — the natives for walking a string by character, and [Q-string-iteration], which asks what a loop over a string gives.
- [Tuples Proposal] and [Destructuring Proposal] — a pair is a tuple, and a pattern in the loop head takes it apart ([Q-pattern]).
- [Pattern Matching Proposal] — the same hidden local and keyword question ([Q-keyword-match]).
- [Enums Proposal] — `Option[T]` is the result of `next()`, and `enum` has the same keyword question ([Q-keyword-enum]).
- [Generic Types Proposal] — `Iterator[T]` and `Iterable[T]` are generic traits.
- [Collection Methods Proposal] — methods like `map` and `filter` on arrays would be built from the same iteration rule.
- [Lists Proposal] — the loop works on a list `[T]` and a fixed size array `[T; n]` alike.
- [Trait Types Proposal] — a function that takes any `Iterable` is a use of a trait type, though generic traits would need to be allowed ([Q-iterator-trait]).
- [Tooling Data Proposal] and [Debugger Proposal] — the binding is a local variable with a short life, and the hidden locals have no name.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it runs is the file of the same name without `.argv`, written `$file` in the arguments. Language tests go in `tests/run/` and run with `run` and `$file`. See `tests/README.md` in kirbylang.

##### NEW: tests/run/flow_control/for_each_array.krb

```kirby
let names: [string] = ["a", "b", "c"];

for (name in names) {
  @println(name);
}
```

###### Expected Outcome

Prints `a`, `b`, `c` on separate lines.

##### NEW: tests/run/flow_control/for_each_empty.krb

```kirby
let none: [i64] = [];

for (n in none) {
  @println(n);
}

@println("done");
```

###### Expected Outcome

Prints `done` only.

##### NEW: tests/run/flow_control/for_each_break_continue.krb

```kirby
for (n in [1, 2, 3, 4, 5]) {
  if (n == 2) { continue; }
  if (n == 4) { break; }

  @println(n);
}
```

###### Expected Outcome

Prints `1`, then `3`.

##### NEW: tests/run/flow_control/for_each_nested.krb

Two loops, one inside the other, over `[1, 2]` and `["x", "y"]`, printing each pair.

###### Expected Outcome

Prints the four pairs in order, and the inner binding does not clash with the outer one.

##### NEW: tests/run/flow_control/for_each_push_while_looping.krb

```kirby
let items = [1, 2];

for (n in items) {
  if (n < 3) { @arrPush(items, n + 2); }
  @println(n);
}
```

###### Expected Outcome

Follows [Q-mutation]. Under the recommended answer it prints `1`, `2`, `3`, `4`: the length is read on every pass, so items added during the loop are visited.

##### NEW: tests/run/errors/for_each_not_iterable.krb

```kirby
for (n in 3.5) {
  @println(n);
}
```

###### Expected Outcome

Does not compile. The error says a `f64` can not be looped over.

##### NEW: tests/run/errors/for_each_assign_to_binding.krb

```kirby
for (n in [1, 2]) {
  n = 5;
}
```

###### Expected Outcome

Does not compile. The error says `n` can not be assigned to ([Q-binding]).

## Questions

### **Q:** Is `in` a reserved word?

<!-- [Q-keyword]: #q-is-in-a-reserved-word -->

**Status:** Open

`in` is a valid variable name today (`var in = 1;` works, verified). This is the same choice `enum` has ([Q-keyword-enum]) and `match` has ([Q-keyword-match]): (a) reserve it, like every other keyword, and any program that uses `in` as a name stops compiling. (b) Treat it as a keyword only where a for-each head can have it, after the first name in a `for (` head. Nothing breaks, and the parser has a special case.

All three proposals should make the same choice.

### **Q:** Can the loop variable be assigned to, and can it have a type?

<!-- [Q-binding]: #q-can-the-loop-variable-be-assigned-to-and-can-it-have-a-type -->

**Status:** Open

Options: (a) The variable is read only, as a `let`. The loop gives a value and the body uses it. (b) The variable can be written as `for (var n in items)`, and changing it changes nothing in the array. (c) Both: a bare name is read only, and `var` makes it assignable.

A type is never needed, since the array has one. A pattern in place of the name is [Q-pattern].

(c) is the recommendation and matches how `let` and `var` already work. Either way, assigning to `n` does not change the array, and the docs should say so.

### **Q:** What if the body changes the array?

<!-- [Q-mutation]: #q-what-if-the-body-changes-the-array -->

**Status:** Open

Options: (a) The length is read on each pass. Items added in the body are visited, items removed are not. This is what the hand-written C-style loop does today, and needs no copy. (b) The length is read once, before the loop. Items added are skipped, and removing items can read past the end, which is an out of bounds error. (c) The loop works on a copy. Safe and predictable, and costs a copy of every array that is looped over.

(a) is the recommendation. It matches the loop it replaces, and the docs should say what it does.

### **Q:** What does a `for` loop over a string give?

<!-- [Q-string-iteration]: #q-what-does-a-for-loop-over-a-string-give -->

**Status:** Open

The answer follows [Q-model] in the [String Encoding Proposal]. Under the recommended answer there (UTF-8 text with byte positions): (a) The loop gives characters, each as a one character string. What most people expect. (b) The loop gives bytes. Matches `@strToBytes`, and is almost never what a text program wants. (c) A string is not loopable and a program uses `@strCharSize` as in that proposal.

(a) is the recommendation, and (c) is the safe first step.

### **Q:** Does Kirby get ranges?

<!-- [Q-ranges]: #q-does-kirby-get-ranges -->

**Status:** Open

Options: (a) No. Counted loops stay in the C-style form. (b) `a..b` stops before `b`, as in Rust. Needs the scanner to handle `..` next to numbers. (c) `a..=b` as well, to include the end. (d) A native, `@range(a, b)`, that returns an array. No syntax, and makes an array for the loop.

(b) is the recommendation. A range might later be a value of its own (for slicing), which is out of scope here.

### **Q:** Can a pattern replace the name?

<!-- [Q-pattern]: #q-can-a-pattern-replace-the-name -->

**Status:** Open

`for ((key, value) in map)` and `for (Point { x, y } in points)` take each item apart. Options: (a) Not in this proposal. The name is a plain name. (b) Yes, any pattern the [Destructuring Proposal] allows in `let` that cannot fail. (c) Yes, and patterns that can fail skip the items they do not fit. A bigger change that needs the [Pattern Matching Proposal].

(b) is the recommendation, built when the Destructuring Proposal lands.

### **Q:** How does a user type become loopable?

<!-- [Q-iterator-trait]: #q-how-does-a-user-type-become-loopable -->

**Status:** Open

Part C sketches `Iterator[T]` and `Iterable[T]`. Options: (a) Two generic traits as written. Familiar from Rust, and needs generic traits and `Option`. (b) One trait, `Iterable[T]`, with a `next(self): Option[T]` method on the type itself. The type can only be walked once at a time, since it holds its own place. (c) Nothing: only built-in collections are loopable.

(a) is the recommendation, and (c) is where this proposal can stop if generics are far away. Arrays do not use either trait.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **For-each loop**: A loop that runs its body once for each item of a collection.
- **Binding**: A name given to a value for some stretch of code. The loop variable is a binding.
- **Range**: A run of whole numbers from a start up to an end, written `a..b`.
- **Iterator**: Something that gives the items of a collection one at a time and remembers where it is.
- **Hidden local**: A variable the compiler makes for itself that the program can not name.
- **Pattern**: A shape that a value is matched against to pull parts out of it, like `(key, value)`.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Map Proposal]: ../map/PROPOSAL.md
[String Encoding Proposal]: ../string-encoding/PROPOSAL.md
[Strings Proposal]: ../strings/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-model]: ../string-encoding/PROPOSAL.md#q-are-strings-bytes-utf-8-text-or-characters
[Q-keyword-enum]: ../enums/PROPOSAL.md#q-is-enum-a-reserved-word
[Q-keyword-match]: ../pattern-matching/PROPOSAL.md#q-is-match-a-reserved-word

<!-- Questions -->

[Q-keyword]: #q-is-in-a-reserved-word
[Q-binding]: #q-can-the-loop-variable-be-assigned-to-and-can-it-have-a-type
[Q-mutation]: #q-what-if-the-body-changes-the-array
[Q-string-iteration]: #q-what-does-a-for-loop-over-a-string-give
[Q-ranges]: #q-does-kirby-get-ranges
[Q-pattern]: #q-can-a-pattern-replace-the-name
[Q-iterator-trait]: #q-how-does-a-user-type-become-loopable
