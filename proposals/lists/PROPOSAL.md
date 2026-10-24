---
status: Closed
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: Lists and Fixed Size Arrays

This proposal splits today's "array" into two types. The type Kirby calls an array today, `[T]`, grows and shrinks, so it becomes a **list**. The name **array** moves to a new type, `[T; n]`, whose length is part of its type and never changes. It was implemented and merged in [PR #168]. The sections below describe that implementation.

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

The parts below are steps of one change. Each depends on the one before it, and none is useful alone, so they land together as a single change. Part 1 and Part 2 are mechanical renames, and the rest add the new type.

#### Part 1: Rename array to list inside the compiler

<!-- [Q1-rename]: #part-1-rename-array-to-list-inside-the-compiler -->

No behavior changes. `TYPE_ARRAY`, `NODE_TYPE_ARRAY` and `typeArray()` become `TYPE_LIST`, `NODE_TYPE_LIST` and `typeList()`. Every snapshot stays the same, so this part can land first and proves nothing else moved.

The `ObjArray` object that holds the values is renamed in Part 2, together with the natives that use it.

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

The same change is made to the other nine: `@arrInsert`, `@arrRemove`, `@arrClear`, `@arrContains`, `@arrEqual`, `@arrCopy`, `@arrReverse`, `@arrSlice` and `@arrConcat`. In `src/native.c` the C functions (`arrPushNative` and so on), their argument checks, and the messages (`Cannot pop empty array.`) say list. This is a plain rename. After it, the old name is no longer a native, so `@arrPush(xs, 1);` reports `Undefined variable '@arrPush'.` when the program runs, like any other name that is not defined ([Q-migration]).

The heap object follows the natives. `ObjArray`, `OBJ_ARRAY`, `IS_ARRAY`, `AS_ARRAY`, `newArray()` and `writeValueToArrayObj()` become `ObjList`, `OBJ_LIST`, `IS_LIST`, `AS_LIST`, `newList()` and `writeValueToListObj()`, which frees the name `ObjArray` for the fixed size object in Part 6. The opcode `OP_ARRAY` (38) is renamed `OP_LIST`, keeping its number. Messages and `@typeof` say list for a growable value: `Cannot pop empty list.`, `function @listLen expects argument 1 to be a list.`, `index 10 out of bounds. list length: 2` and `@typeof` gives `"list"`. Messages that apply to both types, such as `Array index out of bounds.`, keep saying array.

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

After the element type it accepts `;` and a whole number literal, and makes an array type node (`NODE_TYPE_ARRAY`) that holds both. The new type kind `TYPE_ARRAY` holds an element type and a length. Two array types are the same type when both parts are the same, so `[i64; 3]` and `[i64; 4]` are different types, and neither is the same as `[i64]`.

- A length of `0` is allowed. A missing length (`[i64;]`), a name (`[i64; n]`), a decimal (`[i64; 2.5]`) and a negative number report errors at parse time. A length too big for an `int` reports `Array length is too large.`
- The message for a list type that is not closed is now `Expect ']' after list element type.`
- Native signatures can name the length with a placeholder, `N` or `M` ([Q-size-placeholder]). The parser only accepts them when it reads a native signature, so `fun f(a: [i64; N])` in a program is a parse error.

#### Part 4: Literals

<!-- [Q4-literals]: #part-4-literals -->

An array literal is written the same as a list literal. The type checker decides which one it is, the way it already decides whether a number literal is an `f64` (`typchkCheck` in `src/typecheck.c`). It records the answer on the literal (`isFixedSize` on `ArrayNode`) for the compiler:

```diff
 if (node->kind == NODE_ARRAY && expected != NULL) {
-    if (expected->kind != TYPE_LIST) {
+    if (expected->kind != TYPE_LIST && expected->kind != TYPE_ARRAY) {
```

- With an expected type `[T]`, the literal is a list.
- With an expected type `[T; n]`, the literal is an array and must have exactly `n` items. A different count is a compile error: `Expected [i64; 3], got an array literal with 2 items.`
- With no expected type, a literal with items is an array whose length is its item count, so `let a = [1, 2, 3];` is a `[i64; 3]` ([Q-literal-default]).
- An empty literal has no items to learn an item type from, so it needs an expected type: `let xs: [i64] = [];` is a list and `let xs: [i64; 0] = [];` is an array. With none, `let xs = [];` is a compile error that says it can't tell the item type of an empty literal and suggests adding a type. Before this change `[]` had an unknown item type, and anything could be pushed to it. The same error covers a bare `[];` statement and a literal given to a native that takes any value, such as `@typeof([])`. If the declared type of the variable was already reported as unknown, the literal adds no second error.
- A literal given where a generic native expects a list is a list. `@listLen([1, 2, 3])`, `@listReverse([1, 2, 3])` and `@listLen([])` work, because the parameter `[T]` says list, and the item type of `[]` stays unknown. `@arrLen([])` is the error above, because an array parameter says nothing about the item type. A literal held in a variable has already been given its type, so `let xs = [1, 2, 3]; @listPush(xs, 4);` is a type error. The fix is to say which type is wanted: `let xs: [i64] = [1, 2, 3];`.
- A generic native used as a value (`let pop = @listPop;`) has no type to check against, so a literal passed to it is an array at run time.

Nested literals follow the same rule: `[[1, 2], [3, 4]]` is a `[[i64; 2]; 2]`, and `[[1, 2], [3]]` is an error because the inner arrays have different lengths.

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

A list does not turn into an array on its own, and an array does not turn into a list on its own. A list native given an array reports a type error, and so does an array native given a list. The messages show the parameter with the placeholders filled in as far as the arguments allow, such as `Expected [T; N], got [i64].` for `@arrLen` given a list, and `Expected [i64; 3], got [i64; 2].` for the second argument of `@arrEqual`.

An index that is a number literal past the end of an array is a compile error, `Index 2 is out of range for [i64; 2].` Any other index, such as a variable, is checked when the program runs, as it is for a list.

#### Part 6: Run time

<!-- [Q6-runtime]: #part-6-run-time -->

An array is its own heap object, `ObjArray`, with a `count` and the values but no `capacity` ([Q-representation]):

```c
typedef struct {
  Obj obj;
  int count;
  Value *values;
} ObjArray;
```

`newArray(gc, count)` allocates the values first and the object second, so a collection while it allocates cannot free an object nothing holds yet. The garbage collector (`src/gc.c`) marks and frees it, printing shows `[1, 2, 3]` like a list, and `@typeof` gives `"array"`.

The compiler emits a new opcode, `OP_ARRAY` (43), for an array literal, with the item count in one byte. A list literal emits `OP_LIST`, which is the old `OP_ARRAY` (38) under its new name. An array literal with more than 255 items is a compile error, as before (`Too many elements in array literal.`). `OP_GET_INDEX` and `OP_SET_INDEX` accept either object.

## Impacts

### Existing Syntax Or Behavior

- **Breaking: every call to an `@arr*` native on a list must be renamed to `@list*`.** The type syntax `[T]` and everything else about lists stays the same. After Part 2 the old name reports `Undefined variable '@arrPush'.` The fix is a find and replace. [Q-migration] asks whether to soften this.
- **Breaking: an unannotated literal with items is an array.** `let xs = [1, 2, 3]; @listPush(xs, 4);` reports `Expected [T], got [i64; 3].` The fix is an annotation, `let xs: [i64] = [1, 2, 3];`. A literal given directly to a list parameter is still a list. `var xs = [];` is also an error now, since its item type is unknown, and needs `var xs: [i64] = [];`. In the repository's own tests about 55 programs needed an annotation for the first reason and 15 for the second.
- The `@typeof` of a list is `"list"` and of an array literal is `"array"`.
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

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases. Each test is a `.krb` file with an `.argv` file, as described in `tests/README.md` in kirbylang. These are the cases that exist, grouped by what they show. Every file name below is in `tests/run/` and ends in `.krb`.

##### CHANGED: native_functions/native_fn_list_pop_call_empty_array

Renamed from `native_fn_arr_pop_call_empty_array`. It calls `@listPop` on an empty list. The exit code (70) is the same, the bytecode listing shows `'@listPop'`, and the message is `Cannot pop empty list.` All the `native_fn_arr_*` tests are renamed `native_fn_list_*` and `tests/run/arrays` is renamed `tests/run/lists`.

##### CHANGED: tests that held a literal in a variable

About 55 tests assigned a literal to a variable and then called a list native. They now annotate the variable (`var array: [i64] = [1, 2, 3];`). Their expected output is unchanged.

##### NEW: lists/list_old_native_name, lists/list_natives

`@arrPush(xs, 2)` on a list exits with 70 and `Undefined variable '@arrPush'.` `@listPush`, `@listLen` and `@listPop` print `2` and `2`.

##### NEW: arrays/array_type_*

The type syntax: `array_type_param`, `array_type_zero_length`, `array_type_nested` and `array_type_same_type` run. `array_type_missing_size`, `array_type_size_not_a_number`, `array_type_size_not_whole`, `array_type_size_negative` and `array_type_length_placeholder_in_user_code` are parse errors. `array_type_not_a_list`, `array_type_list_not_an_array`, `array_type_different_length` and `array_type_different_element` are type errors that show `[i64; 2]`, `[i64; 3]` and `[i64]` are different types.

##### NEW: arrays/fixed_array_*

Literals and indexing: `fixed_array` (read, write and print), `fixed_array_inferred` (`@typeof` is `array`), `fixed_array_nested`, `fixed_array_nested_annotation`, `fixed_array_struct_field`, `fixed_array_in_for`, `fixed_array_of_strings`, `fixed_array_zero_length` and `fixed_array_gc`. Errors: `fixed_array_wrong_count`, `fixed_array_wrong_count_too_many`, `fixed_array_wrong_element_type`, `fixed_array_jagged`, `fixed_array_set_wrong_type` and `fixed_array_list_native` (compile errors), `fixed_array_literal_index_out_of_range` and `fixed_array_literal_index_set_out_of_range` (compile errors), and `fixed_array_index_out_of_range_runtime` and `fixed_array_index_set_out_of_range_runtime` (run time errors, exit 70).

##### NEW: lists/list_*

`list_annotation_literal`, `list_literal_return`, `list_literal_argument` and `list_empty_literal_is_list` show a list from an annotation, a return type, a parameter and `[]` with a type. `list_empty_literal_without_type`, `list_empty_literal_statement` and `list_empty_literal_to_unknown_type` are the errors for an empty literal with no type, and `list_empty_literal_to_generic_native` shows `@listLen([])` working. `fixed_array_empty_literal_to_array_native` shows `@arrLen([])` as an error. `list_unannotated_literal_to_list_native` and `list_unannotated_literal_to_list_param` are the breaking case: both report that a list was expected and `[i64; 2]` was found.

##### NEW: native_functions/native_fn_arr_*

Each array native has a normal call and the cases that apply to it: no arguments, too many arguments, a wrong argument type, a list given instead of an array (`Expected [T; N], got [i64].`), a different item type, a different length for `@arrEqual`, and `@arrCopy` and `@arrReverse` keeping the length (indexing the result past the end is a compile error). `@arrSlice`, `@arrConcat` and `@arrToList` return lists that can be pushed to. `native_fn_arr_is_empty_is_not_defined` and `native_fn_arr_push_is_not_defined` show there is no `@arrIsEmpty` or `@arrPush`, and `native_fn_list_len_call_array_arg` shows a list native refusing an array. `arrays/fixed_array_natives` is the example from Part 5.

Unit tests in `unit/types.c` and `unit/typecheck.c` cover array equality, how a type prints, filling `N` and `M` from arguments, and how a literal gets its type.

## Outcome

**Status:** Accepted

Implemented and delivered in commit `3ee3a32` ([PR #168]). All of the parts above landed together as one change. The questions below were all answered, and the answers are what was built.

## Questions

### **Q-native-names:** Should the `@arr*` natives be renamed?

<!-- [Q-native-names]: #q-native-names-should-the-arr-natives-be-renamed -->

**Status:** Answered

Options: (a) Leave them. `@arrLen` and `@arrContains` read as short for "array" and still work on both types. The mutating ones, such as `@arrPush`, read oddly on a type called list. (b) Rename to `@list*`, with the read only ones also kept for arrays under their old names. (c) Wait for the [Collection Methods Proposal], so that `list.push(x)` replaces them and the question goes away.

#### Answer

Separate sets, as in (b). Every current `@arr*` native becomes the matching `@list*` native (Part 2). The `@arr*` names are then used for a smaller set that works on fixed size arrays (Part 5). A native name says which type it takes.

### **Q-size-placeholder:** How does a native signature say "any length"?

<!-- [Q-size-placeholder]: #q-size-placeholder-how-does-a-native-signature-say-any-length -->

**Status:** Answered

A native signature is a string such as `"fun ([T], T) => unit"` (`src/native_signatures.c`). `T` stands for any item type, and is filled in from the arguments. An array native needs a second kind of placeholder for the length, written `N` in Part 5, so that `@arrEqual` can say both arrays have the same length and `@arrCopy` can say the result has the same length as the argument. Options: (a) `N` and `M`, filled in from the arguments the way `T` is. (b) A single `_` that means "any length" and is not tracked, so `@arrCopy` gives `[T; _]` and the length is lost. (a) keeps the length through a copy. (b) needs no new rules for filling in a placeholder.

#### Answer

`N` and `M`, option (a). Each is filled in from the arguments the way `T` is, using binding slots after `T` to `W`. The first argument to fill one wins, and a later parameter that uses it is checked against the filled in length, so `@arrEqual` on arrays of different lengths is a compile error and `@arrCopy` keeps the length. A placeholder in an expected type accepts any length. Only native signatures can use them.

### **Q-migration:** Should the old names keep working for a while?

<!-- [Q-migration]: #q-migration-should-the-old-names-keep-working-for-a-while -->

**Status:** Answered

Part 2 breaks every call to an `@arr*` native on a list. Options: (a) Break at once. The change is a find and replace. The changelog has precedent for changing a signature in place. (b) Keep `@arrPush` and the others for one release, accepting lists, with a warning. Kirby has no warning system, so this needs one. (c) A migration note only, with the rename done by a script in `scripts/`.

#### Answer

Break at once, option (a). The old names are not defined, and the changelog entry says so. There is no warning system and no migration script.

### **Q-slice-concat:** What do `@arrSlice` and `@arrConcat` return?

<!-- [Q-slice-concat]: #q-slice-concat-what-do-arrslice-and-arrconcat-return -->

**Status:** Answered

The length of a slice depends on numbers that are known only when the program runs, and the length of `@arrConcat(a, b)` would be `N + M`, which the type system cannot write. Part 5 returns a list from both. Options: (a) A list, as in Part 5. (b) Leave both out for arrays, so a program calls `@arrToList` first. (c) Return an array for `@arrConcat` once sizes can be added in a type.

#### Answer

A list from both, option (a), as in Part 5.

### **Q-conversion:** How does a list become an array?

<!-- [Q-conversion]: #q-conversion-how-does-a-list-become-an-array -->

**Status:** Answered

An array becomes a list with `@arrToList`, which copies. A list becomes an array only if its length matches, so it can fail. Options: (a) A native that returns an `Option[[T; n]]` once [Enums Proposal] lands. (b) A cast, `list as [i64; 3]`, that stops the program when the length is wrong, in the way `1 / 0` stops it. (c) Destructuring only: `let [a, b, c] = list;` checks the length when the program runs.

#### Answer

Deferred. Only `@arrToList` exists, which copies an array to a list. A way to go from a list to an array waits for the [Enums Proposal] (an `Option`) or a cast.

### **Q-literal-default:** Should a literal with no annotation be a list?

<!-- [Q-literal-default]: #q-literal-default-should-a-literal-with-no-annotation-be-a-list -->

**Status:** Answered

`let a = [1, 2, 3];` is a list in this sketch, since it keeps every list program working. The other choice is to make it an array when its items are fixed and let a program ask for a list. That is a break in every program that calls a `@list*` native on an unannotated literal.

#### Answer

No. A literal with items and no expected type is an array whose length is its item count, so `let a = [1, 2, 3];` is a `[i64; 3]`. An empty literal is an error unless a type is expected, because there are no items to learn the item type from. A literal where a list type is expected, including a generic native's `[T]` parameter, is a list. See Part 4 for the rules and the breaking change in Impacts.

### **Q-repeat:** Should there be a repeat literal such as `[0; 100]`?

<!-- [Q-repeat]: #q-repeat-should-there-be-a-repeat-literal-such-as-0-100 -->

**Status:** Answered

It matches the type syntax and is the natural way to make a large zeroed array. The literal limit of 255 items stops writing one out by hand. It is left out of the first version.

#### Answer

Left out of this version. It can be its own change.

### **Q-representation:** Is an array a different heap object from a list?

<!-- [Q-representation]: #q-representation-is-an-array-a-different-heap-object-from-a-list -->

**Status:** Answered

Options: (a) Reuse `ObjArray` for both, with the length fixed by the type checker. Least new code. At run time the two look alike, so `@typeof` cannot tell them apart. (b) A second object kind with no `capacity` field, which saves memory and gives `@typeof` something to say. The garbage collector (`src/gc.c`) and printing each get a new case. The [Tuples Proposal] asks the same question for tuples.

#### Answer

A second object kind, option (b). `ObjArray` has no `capacity`, so it is smaller, and `@typeof` gives `"array"` for it and `"list"` for the growable object, which was renamed `ObjList`. The garbage collector and printing each got a case. See Part 6.

### **Q-size-source:** What can `n` be?

<!-- [Q-size-source]: #q-size-source-what-can-n-be -->

**Status:** Answered

In this version, a whole number literal. Later, any value known when the program is compiled, such as a top level `let` ([Top-Level Declarations Proposal]).

#### Answer

A whole number literal, as in this version. A named constant waits for the [Top-Level Declarations Proposal].

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

<!-- External -->

[PR #168]: https://github.com/kirbylang/kirbylang/pull/168

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
