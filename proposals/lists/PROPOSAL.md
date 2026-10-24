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

It can grow, because the natives `@arrPush`, `@arrInsert`, `@arrPop`, `@arrRemove` and `@arrClear` all change its length. All 14 natives that work on it start with `@arr`. In C it is an `ObjArray` with a `count`, a `capacity` and a pointer to the values (`src/object.h`). The type checker only knows the type of the items. It does not know how many there are:

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

- Renaming today's array type to list. The type syntax `[T]` stays the same.
- Renaming today's `@arr*` natives to `@list*`, since they work on lists.
- A new fixed size array type, `[T; n]`, with its own `@arr*` natives.
- Which operations each type has.
- How a literal gets one type or the other.

The following is intentionally left out of scope for this proposal:

- Methods such as `list.push(1)`. That belongs to the [Collection Methods Proposal].
- Multi-dimensional arrays, slices that borrow, and arrays with a size taken from a constant. A size is a number literal in this version ([Q-size-source]).
- Generic sizes in user code, such as `fun sum[N](a: [i64; N])`. That needs the [Generic Types Proposal] to accept values as parameters. Native signatures get a size placeholder ([Q-size-placeholder]).

### Implementation Plan

#### Part 1: Rename array to list inside the compiler

<!-- [Q1-rename]: #part-1-rename-array-to-list-inside-the-compiler -->

No behavior changes. `TYPE_ARRAY`, `NODE_TYPE_ARRAY` and `typeArray()` become `TYPE_LIST`, `NODE_TYPE_LIST` and `typeList()`. Every snapshot stays the same, so this part can land first and proves nothing else moved.

The `ObjArray` object that holds the values keeps its name until [Q-representation] is settled.

#### Part 2: Rename the array natives to list natives

<!-- [Q2-rename-natives]: #part-2-rename-the-array-natives-to-list-natives -->

Every native in `src/native_signatures.c` that starts with `@arr` gets the same name with `@list` in its place, with the same signature and the same behavior:

```diff
-    {"@arrJoin", "fun ([string], string) => string"},
-    {"@arrLen", "fun ([T]) => i64"},
-    {"@arrIsEmpty", "fun ([T]) => bool"},
-    {"@arrPush", "fun ([T], T) => unit"},
-    {"@arrPop", "fun ([T]) => T"},
+    {"@listJoin", "fun ([string], string) => string"},
+    {"@listLen", "fun ([T]) => i64"},
+    {"@listIsEmpty", "fun ([T]) => bool"},
+    {"@listPush", "fun ([T], T) => unit"},
+    {"@listPop", "fun ([T]) => T"},
```

The same change is made to the other nine: `@arrInsert`, `@arrRemove`, `@arrClear`, `@arrContains`, `@arrEqual`, `@arrCopy`, `@arrReverse`, `@arrSlice` and `@arrConcat`. In `src/native.c` the C functions (`arrPushNative` and so on), their argument checks, and the messages (`Cannot pop empty array.`) say list. This is a plain rename. After it, the old name is no longer a native, so `@arrPush(xs, 1);` reports `Undefined variable '@arrPush'.` ([Q-migration]).

The rename also covers the places that repeat the names:

- the test folder `tests/run/native_functions` (`native_fn_arr_*` becomes `native_fn_list_*`) and `tests/run/arrays` (becomes `tests/run/lists`), where every `.err` file shows the native's name in its bytecode listing;
- `unit/native_signatures.c` and `unit/typecheck.c`;
- `stdlib/stdlib.krb` and `examples/stringBuilder.krb`;
- the hover files `vsc/hovers/@arr*.md` and `vsc/snippets.json`;
- `README.md`, `docs/TYPES.md` and `book/src/reference/builtin-functions.md`.

A count of `@arr` over the repository, tests included, finds about 360 uses in 143 files, so this part is large but mechanical. `docs/CHANGELOG.md` keeps its old entries as history and gets a new one.

#### Part 3: Parse and check `[T; n]`

<!-- [Q3-array-type]: #part-3-parse-and-check-t-n -->

The type parser reads `[`, a type, and then `]`:

```c
if (match(p, TOKEN_LEFT_BRACKET)) {
    int line = p->previous.line;
    AstNode *elementType = parseType(p);
    consume(p, TOKEN_RIGHT_BRACKET, "Expect ']' after array element type.");
```

After the element type it should also accept `;` and a whole number literal, and make an array type node that holds both. The new type kind `TYPE_ARRAY` holds an element type and a length. Two array types are the same type when both parts are the same, so `[i64; 3]` and `[i64; 4]` are different types, and neither is the same as `[i64]`.

#### Part 4: Literals

<!-- [Q4-literals]: #part-4-literals -->

An array literal is written the same as a list literal. The type checker decides which one it is from the type it expects, the way it already decides whether a number literal is an `f64` (`typchkCheck` in `src/typecheck.c`):

```diff
 if (node->kind == NODE_ARRAY && expected != NULL) {
-    if (expected->kind != TYPE_ARRAY) {
+    if (expected->kind != TYPE_LIST && expected->kind != TYPE_ARRAY) {
```

- With no expected type, or with an expected type `[T]`, the literal is a list. Every list program that works today keeps working once its natives are renamed.
- With an expected type `[T; n]`, the literal must have exactly `n` items, and a different count is a compile error.

#### Part 5: What each type can do, and the array natives

<!-- [Q5-operations]: #part-5-what-each-type-can-do-and-the-array-natives -->

| Operation | List `[T]` | Array `[T; n]` |
| --- | --- | --- |
| Read `a[i]` | yes | yes |
| Write `a[i] = x` | yes | yes |
| Index out of range | run time error | compile error when `i` is a literal, run time error otherwise |
| Length | `@listLen` | `@arrLen` |
| Change length | `@listPush`, `@listPop`, `@listInsert`, `@listRemove`, `@listClear` | none |
| Search and compare | `@listContains`, `@listEqual`, `@listIsEmpty` | `@arrContains`, `@arrEqual` |
| Copy, reverse, join | `@listCopy`, `@listReverse`, `@listJoin` | `@arrCopy`, `@arrReverse`, `@arrJoin` |
| Cut and join pieces | `@listSlice`, `@listConcat` | `@arrSlice`, `@arrConcat` give a list ([Q-slice-concat]) |
| Make the other type | none (see [Q-conversion]) | `@arrToList` |
| Destructuring `let [a, b] = x;` | run time check of the length | compile check of the length |

The `@arr*` names come back as natives for the new type. Their signatures need a placeholder for the length, written here as `N` ([Q-size-placeholder]):

```kirby
fun ([T; N]) => i64                   // @arrLen
fun ([T; N], T) => bool               // @arrContains
fun ([T; N], [T; N]) => bool          // @arrEqual
fun ([T; N]) => [T; N]                // @arrCopy and @arrReverse
fun ([string; N], string) => string   // @arrJoin
fun ([T; N], integer, integer) => [T] // @arrSlice
fun ([T; N], [T; M]) => [T]           // @arrConcat
fun ([T; N]) => [T]                   // @arrToList
```

There is no `@arrIsEmpty`: only `[T; 0]` is empty, and the type already says so. `@arrLen` is known when the program is compiled, so a later change can turn a call into the number itself.

A list does not turn into an array on its own, and an array does not turn into a list on its own. A list native given an array reports a type error, and so does an array native given a list.

#### Part 6: Run time

<!-- [Q6-runtime]: #part-6-run-time -->

The compiler already emits `OP_ARRAY` with the item count in one byte, and an array literal with more than 255 items is a compile error (`Too many elements in array literal.`). Reading and writing use `OP_INDEX_GET` and `OP_INDEX_SET`. An array can use all three unchanged. How it is stored is [Q-representation].

## Impacts

### Existing Syntax Or Behavior

- **Breaking: every call to an `@arr*` native on a list must be renamed to `@list*`.** The type syntax `[T]` and everything else about lists stays the same. After Part 2 the old name reports `Undefined variable '@arrPush'.` The fix is a find and replace. [Q-migration] asks whether to soften this.
- No existing program is a `[T; n]` program, because `;` inside `[]` in a type is a parse error today. The names `@arrLen`, `@arrContains` and the others are reused for the new type, so a program that still uses them on a list gets a type error rather than an undefined name.
- Error messages and docs that say "array" for a growable value say "list" afterwards. `docs/CHANGELOG.md` has entries that use the word array for `[T]` (for example "Array types are written `[T]`"). Those entries are history and stay as they are. A new entry records the rename.
- The embedded library groups natives by name. The `ARRAY` group in the [Embedded Library Proposal] becomes a `LIST` group, and a new `ARRAY` group holds the fixed size natives.

### Related Proposals

- [Generic Types Proposal] — says `[T]` becomes a list later and `[T; n]` is a fixed size array. This proposal is that plan. Size parameters in user code (`[T; N]`) are left out here and would need that proposal to accept them. The native size placeholder is a smaller version of the same idea.
- [Tuples Proposal] — says `[T; n]` is planned, and uses it to explain why a tuple type is written `(A, B)`. The two types are alike in having a fixed size and unalike in having one item type versus many. Its [Q-len] asks for a tuple length native, and `@arrLen` is the nearest example.
- [Collection Methods Proposal] — the "third shape" in [Q-category] is the array. Methods on a list can change its length, and methods on an array cannot. That proposal's mechanism has to name `[T]` and `[T; n]` as different receivers, and its examples of typed natives (`@arrMap`) are `@listMap` after this proposal.
- [Destructuring Proposal] — says an array's length is not part of its type. After this proposal, destructuring an array `[T; n]` can check the pattern length at compile time, and destructuring a list still checks at run time with `@listLen`.
- [For-Each Proposal] — a loop over either type should work through one rule. The natives that change a list while it is read are `@list*` after this proposal, and an array cannot change length at all.
- [Pattern Matching Proposal] — array patterns have a fixed length. Matching an `[T; n]` against a pattern of the same length is always a fit, but a list always needs a catch-all.
- [Embedded Library Proposal] — `KRB_LIB_ARRAY` and the `krbArray*` functions name the growable type, and its table of native groups lists the `@arr*` names. They become `LIST` and `ARRAY` groups.
- [Map Proposal] — its natives are described "in the style of the array natives (`@arrPush`, `@arrLen`, ...)". Those are `@listPush` and `@listLen` after this proposal. The `@map*` names do not change.
- [Prefixed Native Functions Proposal] — Closed. It set the `@` prefix. Adding the `@list` and `@arr` prefixes follows its rules, and the hover files and snippets it describes are renamed with them.
- [Strings Proposal] — `@strToBytes` and `@strSplit` return lists, and `@arrJoin` on a list of strings is `@listJoin`. Nothing else changes.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases. Each test is a `.krb` file with an `.argv` file, as described in `tests/README.md` in kirbylang.

##### CHANGED: tests/run/native_functions/native_fn_arr_pop_call_empty_array.krb

The test is renamed to `native_fn_list_pop_call_empty_array.krb`.

```diff
 var array = [];

-@arrPop(array);
+@listPop(array);
```

###### Expected Outcome

The same exit code (70). The bytecode listing in the `.err` file shows `'@listPop'`, and the message says `Cannot pop empty list.`

##### NEW: tests/run/lists/list_old_native_name.krb

```kirby
let xs: [i64] = [1];
@arrPush(xs, 2);
```

###### Expected Outcome

A compile error: `Undefined variable '@arrPush'.`

##### NEW: tests/run/lists/list_natives.krb

```kirby
let xs: [i64] = [1];
@listPush(xs, 2);
@println(@listLen(xs));
@println(@listPop(xs));
```

###### Expected Outcome

Prints `2` and then `2`.

##### NEW: tests/run/arrays/fixed_array.krb

```kirby
let a: [i64; 3] = [1, 2, 3];
a[0] = 9;
@println(a[0]);
@println(@arrLen(a));
```

###### Expected Outcome

Prints `9` and then `3`.

##### NEW: tests/run/arrays/fixed_array_natives.krb

```kirby
let a: [i64; 3] = [1, 2, 3];
let b = @arrReverse(a);
@println(b[0]);
@println(@arrContains(a, 2));
@println(@arrEqual(a, b));
let xs = @arrToList(a);
@listPush(xs, 4);
@println(@listLen(xs));
```

###### Expected Outcome

Prints `3`, `true`, `false`, and `4`.

##### NEW: tests/run/arrays/fixed_array_wrong_count.krb

```kirby
let a: [i64; 3] = [1, 2];
```

###### Expected Outcome

A compile error that the array has 2 items and the type needs 3.

##### NEW: tests/run/arrays/fixed_array_list_native.krb

```kirby
let a: [i64; 2] = [1, 2];
@listPush(a, 3);
```

###### Expected Outcome

A compile error that `@listPush` needs a list and got `[i64; 2]`. There is no array version of `@listPush`.

##### NEW: tests/run/arrays/list_array_native.krb

```kirby
let xs: [i64] = [1, 2];
@println(@arrLen(xs));
```

###### Expected Outcome

A compile error that `@arrLen` needs an array and got `[i64]`.

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

<!-- [Q-native-names]: #q-native-names-should-the-arr-natives-be-renamed -->

**Status:** Answered

Options: (a) Leave them. `@arrLen` and `@arrContains` read as short for "array" and still work on both types. The mutating ones, such as `@arrPush`, read oddly on a type called list. (b) Rename to `@list*`, with the read only ones also kept for arrays under their old names. (c) Wait for the [Collection Methods Proposal], so that `list.push(x)` replaces them and the question goes away.

#### Answer

Separate sets, as in (b). Every current `@arr*` native becomes the matching `@list*` native (Part 2). The `@arr*` names are then used for a smaller set that works on fixed size arrays (Part 5). A native name says which type it takes.

### **Q-size-placeholder:** How does a native signature say "any length"?

<!-- [Q-size-placeholder]: #q-size-placeholder-how-does-a-native-signature-say-any-length -->

**Status:** Open

A native signature is a string such as `"fun ([T], T) => unit"` (`src/native_signatures.c`). `T` stands for any item type, and is filled in from the arguments. An array native needs a second kind of placeholder for the length, written `N` in Part 5, so that `@arrEqual` can say both arrays have the same length and `@arrCopy` can say the result has the same length as the argument. Options: (a) `N` and `M`, filled in from the arguments the way `T` is. (b) A single `_` that means "any length" and is not tracked, so `@arrCopy` gives `[T; _]` and the length is lost. (a) keeps the length through a copy. (b) needs no new rules for filling in a placeholder.

### **Q-migration:** Should the old names keep working for a while?

<!-- [Q-migration]: #q-migration-should-the-old-names-keep-working-for-a-while -->

**Status:** Open

Part 2 breaks every call to an `@arr*` native on a list. Options: (a) Break at once. The change is a find and replace. The changelog has precedent for changing a signature in place. (b) Keep `@arrPush` and the others for one release, accepting lists, with a warning. Kirby has no warning system, so this needs one. (c) A migration note only, with the rename done by a script in `scripts/`.

### **Q-slice-concat:** What do `@arrSlice` and `@arrConcat` return?

<!-- [Q-slice-concat]: #q-slice-concat-what-do-arrslice-and-arrconcat-return -->

**Status:** Open

The length of a slice depends on numbers that are known only when the program runs, and the length of `@arrConcat(a, b)` would be `N + M`, which the type system cannot write. Part 5 returns a list from both. Options: (a) A list, as in Part 5. (b) Leave both out for arrays, so a program calls `@arrToList` first. (c) Return an array for `@arrConcat` once sizes can be added in a type.

### **Q-conversion:** How does a list become an array?

<!-- [Q-conversion]: #q-conversion-how-does-a-list-become-an-array -->

**Status:** Open

An array becomes a list with `@arrToList`, which copies. A list becomes an array only if its length matches, so it can fail. Options: (a) A native that returns an `Option[[T; n]]` once [Enums Proposal] lands. (b) A cast, `list as [i64; 3]`, that stops the program when the length is wrong, in the way `1 / 0` stops it. (c) Destructuring only: `let [a, b, c] = list;` checks the length when the program runs.

### **Q-literal-default:** Should a literal with no annotation be a list?

<!-- [Q-literal-default]: #q-literal-default-should-a-literal-with-no-annotation-be-a-list -->

**Status:** Open

`let a = [1, 2, 3];` is a list in this sketch, since it keeps every list program working. The other choice is to make it an array when its items are fixed and let a program ask for a list. That is a break in every program that calls a `@list*` native on an unannotated literal.

### **Q-repeat:** Should there be a repeat literal such as `[0; 100]`?

<!-- [Q-repeat]: #q-repeat-should-there-be-a-repeat-literal-such-as-0-100 -->

**Status:** Open

It matches the type syntax and is the natural way to make a large zeroed array. The literal limit of 255 items stops writing one out by hand. It is left out of the first version.

### **Q-representation:** Is an array a different heap object from a list?

<!-- [Q-representation]: #q-representation-is-an-array-a-different-heap-object-from-a-list -->

**Status:** Open

Options: (a) Reuse `ObjArray` for both, with the length fixed by the type checker. Least new code. At run time the two look alike, so `@typeof` cannot tell them apart. (b) A second object kind with no `capacity` field, which saves memory and gives `@typeof` something to say. The garbage collector (`src/gc.c`) and printing each get a new case. The [Tuples Proposal] asks the same question for tuples.

### **Q-size-source:** What can `n` be?

<!-- [Q-size-source]: #q-size-source-what-can-n-be -->

**Status:** Open

In this version, a whole number literal. Later, any value known when the program is compiled, such as a top level `let` ([Top-Level Declarations Proposal]).

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document.
- **List**: An ordered group of values of one type whose length can change. Written `[T]`. This is what Kirby calls an array today. Its natives start with `@list`.
- **Array**: An ordered group of values of one type whose length is part of the type and cannot change. Written `[T; n]`. Its natives start with `@arr`.
- **Placeholder**: A letter such as `T` or `N` in a native's signature that stands for a type or a length and is filled in from the arguments.
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
[Q-size-placeholder]: #q-size-placeholder-how-does-a-native-signature-say-any-length
[Q-migration]: #q-migration-should-the-old-names-keep-working-for-a-while
[Q-slice-concat]: #q-slice-concat-what-do-arrslice-and-arrconcat-return
[Q-conversion]: #q-conversion-how-does-a-list-become-an-array
[Q-literal-default]: #q-literal-default-should-a-literal-with-no-annotation-be-a-list
[Q-repeat]: #q-repeat-should-there-be-a-repeat-literal-such-as-0-100
[Q-representation]: #q-representation-is-an-array-a-different-heap-object-from-a-list
[Q-size-source]: #q-size-source-what-can-n-be

<!-- Related proposals -->

[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Q-category]: ../collection-methods/PROPOSAL.md#q-is-collection-type-one-category-or-two
[Q-len]: ../tuples/PROPOSAL.md#q-should-a-tuple-have-a-length-native
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Map Proposal]: ../map/PROPOSAL.md
[Strings Proposal]: ../strings/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Prefixed Native Functions Proposal]: ../prefixed-native-functions/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
