---
status: Draft
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: Lists and Fixed Size Arrays

This proposal splits today's "array" into two types. The type Kirby calls an array today, `[T]`, grows and shrinks, so it becomes a **list**. The name **array** moves to a new type, `[T; n]`, whose length is part of its type and never changes. This is a first sketch. The details are expected to change.

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

Kirby has one sequence type, written `[T]` and called an array:

```kirby
let names: [string] = ["a", "b"];
@arrPush(names, "c");
@println(@arrLen(names));
```

It can grow, because `@arrPush`, `@arrInsert`, `@arrPop`, `@arrRemove` and `@arrClear` all change its length. In C it is an `ObjArray` with a `count`, a `capacity` and a pointer to the values (`src/object.h`). The type checker only knows the type of the items. It does not know how many there are:

- `let [a, b, c, d] = values;` can only be found wrong when the program runs ([Destructuring Proposal]).
- An index that is too big reports `Array index out of bounds.` at run time, even when the length could be known when the program is compiled.
- Other proposals already assume a second, fixed size type. The [Generic Types Proposal] and the [Tuples Proposal] both say "`[T]` is a growable array now and becomes a list later, `[T; n]` is a fixed size array". Nothing writes that plan down. This proposal does.

Most languages that have both types give them different names, so a reader can tell from the word whether the size can change. In Kirby "array" is the word for the growable one, which is the opposite of what most programmers expect.

## Proposed Changes

The two types are written like this:

```kirby
let list: [i64] = [1, 2, 3];       // a list: can grow
let array: [i64; 3] = [1, 2, 3];   // an array: always 3 items
```

### Goals and Non Goals

What this proposal covers:

- Renaming today's array type to list. The syntax `[T]` and every program that uses it stay the same.
- A new fixed size array type, `[T; n]`.
- Which operations each type has.
- How a literal gets one type or the other.

The following is intentionally left out of scope for this proposal:

- Methods such as `list.push(1)`. That belongs to the [Collection Methods Proposal].
- Renaming the `@arr*` natives. See [Q-native-names].
- Multi-dimensional arrays, slices that borrow, and arrays inside structs with a size taken from a constant. A size is a number literal in this version.
- Generic sizes such as `fun sum[N](a: [i64; N])`. That needs the [Generic Types Proposal] to accept values as parameters.

### Implementation Plan

#### Part 1: Rename array to list inside the compiler

<!-- [Q1-rename]: #part-1-rename-array-to-list-inside-the-compiler -->

No behavior changes. `TYPE_ARRAY`, `NODE_TYPE_ARRAY` and `typeArray()` become `TYPE_LIST`, `NODE_TYPE_LIST` and `typeList()`. Every snapshot stays the same, so this part can land first and proves nothing else moved. Messages that say "array" for a growable value (`Cannot pop empty array.`) change to "list" in the same part, and those snapshots are updated.

The `ObjArray` object that holds the values keeps its name until [Q-representation] is settled.

#### Part 2: Parse and check `[T; n]`

<!-- [Q2-array-type]: #part-2-parse-and-check-t-n -->

The type parser reads `[`, a type, and then `]`:

```c
if (match(p, TOKEN_LEFT_BRACKET)) {
    int line = p->previous.line;
    AstNode *elementType = parseType(p);
    consume(p, TOKEN_RIGHT_BRACKET, "Expect ']' after array element type.");
```

After the element type it should also accept `;` and a whole number literal, and make an array type node that holds both. The new type kind `TYPE_ARRAY` holds an element type and a length. Two array types are the same type when both parts are the same, so `[i64; 3]` and `[i64; 4]` are different types, and neither is the same as `[i64]`.

#### Part 3: Literals

<!-- [Q3-literals]: #part-3-literals -->

An array literal is written the same as a list literal. The type checker decides which one it is from the type it expects, the way it already decides whether a number literal is an `f64` (`typchkCheck` in `src/typecheck.c`):

```diff
 if (node->kind == NODE_ARRAY && expected != NULL) {
-    if (expected->kind != TYPE_ARRAY) {
+    if (expected->kind != TYPE_LIST && expected->kind != TYPE_ARRAY) {
```

- With no expected type, or with an expected type `[T]`, the literal is a list. Every program that works today keeps working.
- With an expected type `[T; n]`, the literal must have exactly `n` items, and a different count is a compile error.

#### Part 4: What each type can do

<!-- [Q4-operations]: #part-4-what-each-type-can-do -->

| Operation | List `[T]` | Array `[T; n]` |
| --- | --- | --- |
| Read `a[i]` | yes | yes |
| Write `a[i] = x` | yes | yes |
| Index out of range | run time error | compile error when `i` is a literal, run time error otherwise |
| `@arrLen(a)` | yes | yes |
| Change length: `@arrPush`, `@arrPop`, `@arrInsert`, `@arrRemove`, `@arrClear` | yes | compile error |
| Read only natives: `@arrContains`, `@arrJoin`, `@arrSlice`, `@arrCopy`, `@arrEqual`, `@arrConcat`, `@arrReverse`, `@arrIsEmpty` | yes | yes ([Q-natives-both]) |
| Destructuring `let [a, b] = x;` | run time check of the length | compile check of the length |

An array does not turn into a list on its own, and a list does not turn into an array on its own. `@arrCopy` on an array gives a list. Turning a list into an array needs a length check, so it is left to [Q-conversion].

#### Part 5: Run time

<!-- [Q5-runtime]: #part-5-run-time -->

The compiler already emits `OP_ARRAY` with the item count in one byte, and an array literal with more than 255 items is a compile error (`Too many elements in array literal.`). Reading and writing use `OP_INDEX_GET` and `OP_INDEX_SET`. An array can use all three unchanged. How it is stored is [Q-representation].

## Impacts

### Existing Syntax Or Behavior

- Every existing `[T]` program keeps its meaning. No existing program is a `[T; n]` program, because `;` inside `[]` in a type is a parse error today.
- Error messages and docs that say "array" for a growable value say "list" afterwards. `docs/CHANGELOG.md` has entries that use the word array for `[T]` (for example "Array types are written `[T]`"). Those entries are history and stay as they are. A new entry records the rename.
- The `@arr*` natives keep their names for now ([Q-native-names]), even though the mutating ones work on lists only.
- The `arrays` test folder (`tests/run/arrays`) holds tests for the growable type.

### Related Proposals

- [Generic Types Proposal] — says `[T]` becomes a list later and `[T; n]` is a fixed size array. This proposal is that plan. Size parameters (`[T; N]`) are left out here and would need that proposal to accept them.
- [Tuples Proposal] — says `[T; n]` is planned, and uses it to explain why a tuple type is written `(A, B)`. The two types are alike in having a fixed size and unalike in having one item type versus many.
- [Collection Methods Proposal] — the "third shape" in [Q-category] is the array. Methods on a list can change its length, and methods on an array cannot. That proposal's mechanism has to name `[T]` and `[T; n]` as different receivers.
- [Destructuring Proposal] — says an array's length is not part of its type. After this proposal, destructuring an array `[T; n]` can check the pattern length at compile time, and destructuring a list still checks at run time.
- [For-Each Proposal] — a loop over either type should work through one rule.
- [Pattern Matching Proposal] — array patterns have a fixed length. Matching an `[T; n]` against a pattern of the same length is always a fit, but a list always needs a catch-all.
- [Embedded Library Proposal] — `KRB_LIB_ARRAY` and `krbArray*` name the growable type. They would be renamed or left alone with the natives ([Q-native-names]).
- [Map Proposal] and [Strings Proposal] — use `[string]`, `[f64]` and `[u8]` as growable lists. They are unaffected.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases. Each test is a `.krb` file with an `.argv` file, as described in `tests/README.md` in kirbylang.

##### CHANGED: tests/run/arrays/array_pop_empty.krb (and the other growable type tests)

Part 1 renames the word in messages. Only the text of those snapshots changes.

###### Expected Outcome

The same exit code as before. The message says list instead of array.

##### NEW: tests/run/arrays/fixed_array.krb

```kirby
let a: [i64; 3] = [1, 2, 3];
a[0] = 9;
@println(a[0]);
@println(@arrLen(a));
```

###### Expected Outcome

Prints `9` and then `3`.

##### NEW: tests/run/arrays/fixed_array_wrong_count.krb

```kirby
let a: [i64; 3] = [1, 2];
```

###### Expected Outcome

A compile error that says the array has 2 items and the type needs 3.

##### NEW: tests/run/arrays/fixed_array_push.krb

```kirby
let a: [i64; 2] = [1, 2];
@arrPush(a, 3);
```

###### Expected Outcome

A compile error that `@arrPush` needs a list and got `[i64; 2]`.

##### NEW: tests/run/arrays/fixed_array_literal_index_out_of_range.krb

```kirby
let a: [i64; 2] = [1, 2];
@println(a[2]);
```

###### Expected Outcome

A compile error, not a run time error.

##### NEW: tests/run/arrays/fixed_array_not_a_list.krb

```kirby
fun total(xs: [i64]): i64 { return 0; }
let a: [i64; 2] = [1, 2];
total(a);
```

###### Expected Outcome

A compile error that expected `[i64]` and got `[i64; 2]`.

##### NEW: tests/run/arrays/array_type_missing_size.krb

```kirby
let a: [i64;] = [1];
```

###### Expected Outcome

A parse error that a size is expected after `;`.

## Questions

### **Q-native-names:** Should the `@arr*` natives be renamed?

**Status:** Open

Options: (a) Leave them. `@arrLen` and `@arrContains` read as short for "array" and still work on both types. The mutating ones, such as `@arrPush`, read oddly on a type called list. (b) Rename to `@list*`, with the read only ones also kept for arrays under their old names. This touches the [Prefixed Native Functions Proposal] naming and the VS Code hover data. (c) Wait for the [Collection Methods Proposal], so that `list.push(x)` replaces them and the question goes away.

### **Q-natives-both:** How does a native say it takes a list or an array?

**Status:** Open

A native signature is a string such as `"fun ([T], T) => unit"` (`src/native_signatures.c`). A read only native such as `@arrLen` should take both `[T]` and `[T; n]`. Options: (a) A signature form for either type, for example `[T; _]` meaning any length. (b) Let an array be used wherever a list is expected, but only for natives that do not change length. (c) Two signatures per read only native.

### **Q-conversion:** How do values move between the two types?

**Status:** Open

An array to a list is a copy (`@arrCopy`). A list to an array needs a length check, so it can fail. Options: (a) A native that returns an `Option[[T; n]]` once [Enums Proposal] lands. (b) A cast, `list as [i64; 3]`, that stops the program when the length is wrong, in the way `1 / 0` stops it. (c) Destructuring only.

### **Q-literal-default:** Should a literal with no annotation be a list?

**Status:** Open

`let a = [1, 2, 3];` is a list in this sketch, since it keeps every current program working. The other choice is to make it an array when its items are fixed and let a program ask for a list. That is a break in every program that calls `@arrPush` on an unannotated literal.

### **Q-repeat:** Should there be a repeat literal such as `[0; 100]`?

**Status:** Open

It matches the type syntax and is the natural way to make a large zeroed array. The literal limit of 255 items stops writing one out by hand. It is left out of the first version.

### **Q-representation:** Is an array a different heap object from a list?

**Status:** Open

Options: (a) Reuse `ObjArray` for both, with the length fixed by the type checker. Least new code. At run time the two look alike, so `@typeof` cannot tell them apart. (b) A second object kind with no `capacity` field, which saves memory and gives `@typeof` something to say. The garbage collector (`src/gc.c`) and printing each get a new case. The [Tuples Proposal] asks the same question for tuples.

### **Q-size-source:** What can `n` be?

**Status:** Open

In this version, a whole number literal. Later, any value known when the program is compiled, such as a top level `let` ([Top-Level Declarations Proposal]).

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document.
- **List**: An ordered group of values of one type whose length can change. Written `[T]`. This is what Kirby calls an array today.
- **Array**: An ordered group of values of one type whose length is part of the type and cannot change. Written `[T; n]`.
- **Length**: How many items a list or array holds.
- **Compile error**: A mistake found before the program runs.
- **Run time error**: A mistake found while the program runs.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Questions -->

[Q-native-names]: #q-native-names-should-the-arr-natives-be-renamed
[Q-natives-both]: #q-natives-both-how-does-a-native-say-it-takes-a-list-or-an-array
[Q-conversion]: #q-conversion-how-do-values-move-between-the-two-types
[Q-literal-default]: #q-literal-default-should-a-literal-with-no-annotation-be-a-list
[Q-repeat]: #q-repeat-should-there-be-a-repeat-literal-such-as-0-100
[Q-representation]: #q-representation-is-an-array-a-different-heap-object-from-a-list
[Q-size-source]: #q-size-source-what-can-n-be

<!-- Related proposals -->

[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Q-category]: ../collection-methods/PROPOSAL.md#q-is-collection-type-one-category-or-two
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Map Proposal]: ../map/PROPOSAL.md
[Strings Proposal]: ../strings/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Prefixed Native Functions Proposal]: ../prefixed-native-functions/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
