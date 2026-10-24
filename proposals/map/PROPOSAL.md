---
status: Draft
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: Map

This proposal adds a map to Kirby: a collection that stores values under keys and finds one by its key without looking at the rest. `Map[string, f64]` holds numbers by name. It is built from [Issue #70], which sketches a brace literal, a few natives, and a `Hash` trait for keys. This proposal turns that sketch into a design, lists what has to be decided, and orders the work so the parts that need no other proposal can ship first.

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

Kirby has one collection, the array. To find something in an array by name, a program keeps two arrays side by side and loops through the first:

```kirby
let keys: [string] = ["a", "b", "c"];
let values: [f64] = [1, 2, 3];

fun lookup(key: string): f64 {
  for (var i = 0; i < @arrLen(keys); i = i + 1) {
    if (keys[i] == key) {
      return values[i];
    }
  }

  0
}
```

Every lookup visits the keys one by one, and nothing keeps the two arrays the same length. Data keyed by name is common: the members of a JSON object, a word count, a table of settings, a cache. Without a map each of those is a hand-written search.

[Issue #70] is open, has no assignee, and is labeled `needs-proposal`. It asks for:

- A map literal in braces, with a type annotation like `Map[string, f64]`
- Natives `@mapGet`, `@mapSet`, `@mapRemove` and `@mapContains`
- Keys that are not limited to built-in types, using a new `Hash` trait with a `hash` method

It says the syntax "is not set in stone" and raises two questions: the parser can not tell `{}` is a map until it reaches a `:`, and the type may be called `Map` or `Dict`.

The [Collection Methods Proposal] mentions `Map`/`Dict` but states that it "does not design tuples, `Set`, or `Map`/`Dict` themselves." No other proposal does.

Three things about the current implementation shape the design. Each was verified against a build of `from_commit`:

- **The VM's table only has string keys.** `Table` in `src/hashtable.h` is built from entries of `ObjString *key` and a `Value`. It backs globals, struct fields and methods. It can not be reused for keys that are numbers or structs without changing.
- **`{` already has two meanings.** The parse rule table has `[TOKEN_LEFT_BRACE] = {blockExpr, struct_, PREC_CALL}`: a `{` that starts an expression is a block, and a `{` after a name starts a struct initializer. `let e = {};` gives `nil` (an empty block) and `{ 1 }` gives `1`.
- **Generic arguments are not accepted yet.** `struct Box[T]` reports `Generic structs aren't supported yet.`, so `Map[string, f64]` can not be written as a type until the [Generic Types Proposal] lands. Native signatures can already use a placeholder (`fun ([T], T) => unit` for `@arrPush`), so a native that takes a map has a signature to follow.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- A `Map[K, V]` type
- How a map is created and read, and the natives or methods that work on it
- Which types can be keys, and how a key is compared and hashed
- The order a map's entries are walked in
- Building the parts in an order where each can ship without the next

The following is intentionally left out of scope for this proposal:

- `Set`. It is a map with no values and can follow once `Map` exists.
- Maps sorted by key
- Methods like `filter` and `map` on a map. See the [Collection Methods Proposal].
- Merging, copying and comparing whole maps
- A fixed size map

### Part A: The type and its operations

```kirby
let ages: Map[string, i64] = Map.new();

ages["alice"] = 30;
ages["bob"] = 25;

@println(ages["alice"]);              // 30
@println(@mapContains(ages, "carol")); // false
```

- `Map[K, V]` is a type with two type parameters. It is compared by its parameters, like an array is compared by its item type. The name `Map` is [Q-name].
- A map is a heap object, like an array. Assigning it to another variable shares it, and changes show through both.
- `Map.new()` creates an empty map. The element types come from the annotation or the first use.
- The operations, in the style of the array natives (`@arrPush`, `@arrLen`, ...), are in the table below. Whether they are natives or methods is [Q-api-shape].

| Operation                | Native                          | Signature                              |
| ------------------------ | ------------------------------- | -------------------------------------- |
| Add or replace an entry  | `@mapSet(map, key, value)`      | `fun (Map[K, V], K, V) => unit`        |
| Read an entry            | `@mapGet(map, key)`             | `fun (Map[K, V], K) => Option[V]`      |
| Remove an entry          | `@mapRemove(map, key)`          | `fun (Map[K, V], K) => Option[V]`      |
| Is there an entry        | `@mapContains(map, key)`        | `fun (Map[K, V], K) => bool`           |
| How many entries         | `@mapLen(map)`                  | `fun (Map[K, V]) => i64`               |
| All keys, in order       | `@mapKeys(map)`                 | `fun (Map[K, V]) => [K]`               |
| All values, in order     | `@mapValues(map)`               | `fun (Map[K, V]) => [V]`               |

`@mapGet` returns an `Option[V]` because a missing key is a normal answer. `m[key]` as a shorthand is [Q-index-syntax].

### Part B: A literal

Following the issue:

```kirby
let ages: Map[string, i64] = { "alice": 30, "bob": 25 };
```

The parser sees `{`, parses an expression, and finds `:` instead of `;` or `}`. That is enough to know it is a map, since a block never has a bare `:` after its first expression. An empty `{}` is already an empty block and stays one. See [Q-literal-syntax] and [Q-empty-map].

### Part C: Keys

A key must be comparable and hashable. A key type should be one of:

- `string`, `bool` and the number kinds. The VM hashes these itself.
- A struct or enum that implements `Eq` (already built in) and a new `Hash` trait.

```kirby
trait Hash {
  fun hash(self): i64;
}
```

The issue writes `hash` as returning an `f64`. Whole numbers default to `i64` now (CHANGELOG), and a hash is a whole number, so this proposal writes `i64`. Two keys that are `equals` must give the same hash. This is a rule a program must follow, not something the checker can verify.

A float key has an edge case: `NaN` is not equal to itself. See [Q-key-types].

### Part D: Order

A map should walk its entries in the order they were first added. Replacing the value of an existing key keeps its place. Removing a key and adding it again puts it last. See [Q-order].

### Part E: Looping over a map

A `for` loop over a map is [For-Each Proposal] business. Until it lands `@mapKeys` gives an array to loop over.

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Runtime object

- A new object kind, `ObjMap`, in `src/object.h`, next to `ObjString` and the array object. It stores its entries in insertion order (see [Q-order]) with an index for finding a key.
- The index needs a hash table keyed by a `Value`, not an `ObjString *`. The existing `Table` is not changed, because globals and fields rely on it. A second table type in `src/hashtable.c` or a new file is expected.
- The garbage collector in `src/gc.c` visits keys and values. `@print`, `@println`, `@typeof` and `@instanceOf` each get a case.
- A map of `string` keys and `string` or number keys can be done entirely in C.

#### Part 2: Struct keys

- Hashing a struct key means running the key's Kirby `hash` method from inside a native. A native in C receives its arguments and returns a value, and `interpretFunction`/`interpret` in `src/vm.h` start a run from outside. Nothing in `src/native.c` calls back into Kirby code today, so a native that does it is new ground.
- There is a precedent for avoiding it. A comment in `src/native.c` says a value whose type implements `Display` "arrives already converted: the compiler calls its toString() first" (`emitDisplayToString` in `src/compiler.c`). The compiler could do the same for a map call with a struct key: call `key.hash()` itself and pass the number to the native as an extra argument. Comparing two struct keys with `Eq` has the same problem and the same answer is less clear, since the native needs to call `equals` on each candidate.
- This is the hardest part and is why [Q-key-types] suggests leaving struct keys for later.

#### Part 3: Type checker

- `Map[K, V]` as a type, and the native signatures above. Depends on the [Generic Types Proposal] (generic arguments) and the [Enums Proposal] (`Option`).
- A key type is checked against the rule in Part C.

#### Part 4: Parser

- The map literal rule from Part B, in `blockExpr` or its caller in `src/parser.c`. Parse an expression, then look at the next token.

#### Part 5: Docs and tooling

- `docs/TYPES.md`, the changelog, VS Code grammar and hover for the natives.

## Impacts

### Existing Syntax Or Behavior

- `{ expression: expression }` used to be a parse error and becomes a map. A block never has that shape.
- `Map` becomes a type name. A program with a struct called `Map` would clash (nothing in this repository has one).
- Natives starting with `@map` are new.

### Related Proposals

- [Lists Proposal] — the array natives this proposal copies the style of (`@arrPush`, `@arrLen`) are renamed `@listPush` and `@listLen`. The `@map*` names do not change.
- [Generic Types Proposal] — `Map[K, V]` needs generic arguments on a built-in type. The key rule needs a way to ask for `Eq` and `Hash` (trait bounds, Part 4 there).
- [Enums Proposal] — `@mapGet` returns an `Option[V]`.
- [Collection Methods Proposal] — whether the operations are methods ([Q-api-shape]), and its question about whether to wait for `Set` and `Map` ([Q-scope-order]). This proposal is the sketch of `Map`'s shape that its option (b) asks for.
- [For-Each Proposal] — looping over a map gives key and value pairs.
- [Tuples Proposal] and [Destructuring Proposal] — a pair of key and value is a tuple, and `for ((key, value) in map)` is a destructuring pattern.
- [Pattern Matching Proposal] — matching on the result of `@mapGet`.
- [Result Proposal] — none of the map operations fail for outside reasons, so nothing here returns a `Result`.
- [Trait Types Proposal] — a map of `string` to a trait type (`Map[string, Shape]`) is how an open set of values is stored by name.
- [Top-Level Declarations Proposal] — a map literal builds data, and would count as a comptime value when its keys and values do.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it runs is the file of the same name without `.argv`, written `$file` in the arguments. Language tests go in `tests/run/` and run with `run` and `$file`. See `tests/README.md` in kirbylang.

##### NEW: tests/run/maps/map_set_get.krb

```kirby
let ages: Map[string, i64] = Map.new();

@mapSet(ages, "alice", 30);
@mapSet(ages, "bob", 25);
@mapSet(ages, "alice", 31);

@println(@mapLen(ages));
@println(@mapGet(ages, "alice") ?? 0);
@println(@mapGet(ages, "carol") ?? 0);
@println(@mapContains(ages, "bob"));
```

###### Expected Outcome

Prints `2`, `31`, `0`, `true`.

##### NEW: tests/run/maps/map_order.krb

A map is filled with keys `c`, `a`, `b`, then `a` is removed and added again.

###### Expected Outcome

`@mapKeys` gives `["c", "b", "a"]`.

##### NEW: tests/run/maps/map_literal.krb

```kirby
let m: Map[string, i64] = { "x": 1, "y": 2 };

@println(@mapLen(m));
```

###### Expected Outcome

Prints `2`.

##### NEW: tests/run/errors/map_key_type_wrong.krb

A `Map[string, i64]` is set with a number key.

###### Expected Outcome

Does not compile. The error says the key must be a `string`.

##### NEW: tests/run/errors/map_key_not_hashable.krb

A `Map[Point, i64]` where `Point` has no `Hash` impl.

###### Expected Outcome

Does not compile. The error says `Point` needs `Hash`.

## Questions

### **Q:** Is the type called `Map` or `Dict`?

<!-- [Q-name]: #q-is-the-type-called-map-or-dict -->

**Status:** Open

Both are used in other languages. Rust, Java and Go call it a map; Python calls it a dict. Kirby's other names are Rust-like (`struct`, `impl`, `trait`, `Option`, `Result`), and the [Collection Methods Proposal] uses `Map`/`Dict` for both. This proposal writes `Map` throughout. The name is easy to change.

### **Q:** Are the operations natives or methods?

<!-- [Q-api-shape]: #q-are-the-operations-natives-or-methods -->

**Status:** Open

Options: (a) Natives, as in the issue and as arrays do (`@mapGet(m, k)`). Needs no new mechanism, and matches `@arrPush`. (b) Methods (`m.get(k)`). Reads better and chains. It needs the answer to the [Collection Methods Proposal], which is not settled. (c) Natives first, then methods when collection methods land, the way arrays are expected to move.

(c) is the recommendation. It does not wait on another proposal and keeps the migration path the arrays already have.

### **Q:** Is there an index syntax?

<!-- [Q-index-syntax]: #q-is-there-an-index-syntax -->

**Status:** Open

`m[key]` is how arrays are read and written. Options: (a) None. `@mapGet` and `@mapSet` only. (b) `m[key] = value` sets, and `m[key]` reads and returns an `Option[V]`. (c) `m[key]` returns `V` and stops the program if the key is missing, as an array does for an index out of bounds.

(b) gives a short form without a hidden stop. (c) is what an array reader would expect, and a missing key in a map is not a mistake in the way a bad array index is.

### **Q:** What does the map literal look like?

<!-- [Q-literal-syntax]: #q-what-does-the-map-literal-look-like -->

**Status:** Open

Options: (a) Braces with colons, as the issue shows: `{ "a": 1 }`. Familiar from JSON. The parser must look past the first expression to tell it from a block. (b) Prefix the braces with the type: `Map { "a": 1 }`. It matches struct initializers and needs no lookahead, and the type is spelled out each time. (c) No literal; `Map.new()` and sets.

(a) is the issue's choice and is parseable (Part B). (b) is the lowest risk.

### **Q:** How is an empty map written?

<!-- [Q-empty-map]: #q-how-is-an-empty-map-written -->

**Status:** Open

`{}` is an empty block, which evaluates to `nil`. Options: (a) `Map.new()`. Nothing is overloaded. (b) `{:}`, as some languages do. A new token pair and easy to misread. (c) Treat `{}` as a map where a map is expected by the annotation. Context dependent, and `let e = {};` would mean different things with and without an annotation.

(a) is the recommendation, and works for any literal choice.

### **Q:** Which types can be keys?

<!-- [Q-key-types]: #q-which-types-can-be-keys -->

**Status:** Open

Options: (a) `string` only. The existing `Table` could back it nearly as is. Covers JSON objects and many uses. (b) `string`, `bool` and the number kinds. The VM hashes them. (c) Also structs and enums that implement `Hash`. Needs a native to call into Kirby code (Part 2).

A float key has a trap: `NaN` is not equal to itself, so a `NaN` key could never be found again. Options within (b): reject `f64` and `f32` keys, or treat `NaN` as one key.

(a) is the earliest to build, and (b) the next. (c) is where the issue ends up, and is the largest piece.

### **Q:** What order are entries walked in?

<!-- [Q-order]: #q-what-order-are-entries-walked-in -->

**Status:** Open

Options: (a) The order they were added, as Part D says. Programs print and compare the same way every run, and a JSON round trip keeps the order. Costs an extra array of entries beside the index. (b) Unspecified. Faster and smaller, and a program can see a different order after an entry is removed. (c) Sorted by key. Needs `Ord` on keys, and is a different data structure.

(a) is the recommendation.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Map** (or dictionary): A collection that holds values by key.
- **Key**: The value used to find an entry in a map.
- **Hash**: A whole number made from a value, used to find where a key is kept without checking every entry.
- **Hash table**: The structure that stores entries by hash. `Table` in `src/hashtable.h` is the one the VM uses today.
- **Insertion order**: The order entries were first added.
- **Heap object**: A value that lives in memory managed by Kirby's garbage collector, such as a string, array, or struct instance.
- **Native function**: A function written in C and built into the language. Its name starts with `@`.

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Related proposals -->

[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Result Proposal]: ../result/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-scope-order]: ../collection-methods/PROPOSAL.md#q-should-this-start-from-arrays-alone-or-wait-until-tuplessetmap-are-at-least-sketched

<!-- External -->

[Issue #70]: https://github.com/kirbylang/kirbylang/issues/70

<!-- Questions -->

[Q-name]: #q-is-the-type-called-map-or-dict
[Q-api-shape]: #q-are-the-operations-natives-or-methods
[Q-index-syntax]: #q-is-there-an-index-syntax
[Q-literal-syntax]: #q-what-does-the-map-literal-look-like
[Q-empty-map]: #q-how-is-an-empty-map-written
[Q-key-types]: #q-which-types-can-be-keys
[Q-order]: #q-what-order-are-entries-walked-in
