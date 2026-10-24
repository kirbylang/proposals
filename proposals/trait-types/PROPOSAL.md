---
status: Draft
created: 2026-10-24
from_commit: d9279e4
---

# Proposal: Traits as Types

This proposal lets a trait name be used as a type. A value of type `Display` is a value of any type that implements `Display`, and the only thing a program can do with it is call the trait's methods. It is what lets one array, one field or one parameter hold values of different structs. The trait name is written bare, with no extra keyword: `[Shape]`, not `[impl Shape]`.

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

A trait says what a type can do. A type is checked against a trait only where a struct is declared (`impl Trait for Struct`). A trait can not be named where a type is expected. Each of these was verified against a build of `from_commit`:

```kirby
trait T { fun a(self): string; }

fun f(x: T): string = x.a();
// [line 3] Error at 'T': Unknown type.
```

```kirby
fun f(x: Display): string = x.toString();
// [line 1] Error at 'Display': Unknown type.
```

The type resolver (`typchkResolveType` in `src/typecheck.c`) resolves function types and array types from their parts. For a plain name it knows only `unit`, `bool`, `string`, the number kinds, `Self`, a struct, and a type alias (and, while it reads a native's signature, the placeholder names those signatures use). Anything else ends in `typchkErrorAtToken(&t->name, "Unknown type.")`.

The effect is that a group of values that share behavior but not a type can not be held together:

```kirby
trait Shape { fun area(self): f64; }

struct Circle { pub var radius: f64; }
struct Square { pub var side: f64; }

impl Shape for Circle { fun area(self): f64 = 3.14 * self.radius * self.radius; }
impl Shape for Square { fun area(self): f64 = self.side * self.side; }

let shapes = [Circle { radius: 1 }, Square { side: 2 }];
// [line 9] Error: Expected Circle, got Square.
```

An array holds one type (`[Circle]`), and there is no type that both `Circle` and `Square` are. The workaround is a struct with a tag and every field any case needs, which is what the JSON example in the [Enums Proposal] Problem Statement describes. Enums are the right answer when the set of cases is closed and known in one place. A trait type is the answer when the set is open: any struct that implements the trait, including one written later or in another file.

How a call finds its method is already settled. The VM looks the method up by name in the receiver's method table every time (`invoke` and `invokeFromStruct` in `src/vm.c`, described in section 1.7 of the [Generic Types Proposal]). A call through a trait type needs no new run time mechanism. The type checker just has to allow it.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- A trait name where a type is expected: variables, parameters, return types, fields and array items
- Converting a value to a trait type when its type implements the trait
- Calling the trait's methods through the trait type
- Which traits can not be used this way, and the error for them

The following is intentionally left out of scope for this proposal:

- Any new keyword or marker on the trait name (`impl Trait`, `dyn Trait`). The trait name is written bare.
- Generic bounds, `fun show[T: Display](x: T)`. That is static and stays in the [Generic Types Proposal] (Part 4).
- Trait impls on primitives. See [Q-primitives].
- Reading fields, or getting the original struct back. See [Q-downcast].
- Traits that extend other traits beyond what already works. A trait type also offers the methods of its super traits.

### Part A: A trait name is a type

```kirby
trait Shape { fun area(self): f64; }

fun describe(shape: Shape): string =
  $"area is {@numberToString(shape.area())}";

let shapes: [Shape] = [Circle { radius: 1 }, Square { side: 2 }];

for (var i = 0; i < @listLen(shapes); i = i + 1) {
  @println(describe(shapes[i]));
}
```

- A trait name resolves to its existing `TYPE_TRAIT` type, compared by name like a struct.
- A value whose type implements the trait can be used where the trait type is expected: passed as an argument, returned, stored in a field, or put in an array. Nothing is copied or wrapped, and the value is still the same struct instance at run time.
- `shapes[i].area()` is checked against the methods the trait declares. A method the trait does not declare is a compile error, even if the struct has it.
- The check that a struct implements the trait uses what the type already records: the list of traits a struct implements, filled in by `impl Trait for`.

### Part B: Which traits can be used as types

A trait method that mentions `Self` outside the receiver can not be called through a trait type, because the compiler no longer knows what `Self` is. The built-in traits show it:

```kirby
trait Eq { fun equals(self, other: Self): bool; }
trait Default { fun default(): Self; }
```

`a.equals(b)` where `a` and `b` are `Eq` values could be asked of a `Circle` and a `Square`. `default()` has no receiver at all. So a trait that uses `Self` anywhere except as the `self` receiver should not be used as a type. Using one is a compile error that names the method:

```kirby
let x: Eq = ...;
// Error at 'Eq': Eq can't be used as a type because 'equals' uses Self.
```

By that rule `Display` can be used as a type and `Eq`, `Ord` and `Default` can not. A trait the user writes can be used as a type whenever it has no such method. See [Q-self-traits] for the options.

### Part C: Arrays of trait types

An array is typed by its items (`[Shape]`). Whether `[Circle]` can be passed where `[Shape]` is expected is [Q-array-variance].

### Implementation Plan

This plan is preliminary. It names the parts of the code that are expected to change so the size of the work is visible.

#### Part 1: Type kind

- The type kind already exists. `TYPE_TRAIT` is in `src/types.h`, `typeTrait` allocates one with the trait's required methods, and `typchkTypeEnvLookupTrait` finds a trait by name. Today they serve only `trait` declarations and the `impl Trait for` check. No new kind is needed.
- `typchkResolveType` gets a call to `typchkTypeEnvLookupTrait` after the struct and alias lookups, in place of the `Unknown type.` error.
- A struct and a trait can share a name today. This compiles with no error (verified): `trait Foo { fun a(self): string; } struct Foo { pub var x: i64; }`. Once a name can mean either one in a type position, the lookup order would decide silently, so declaring both under one name should become an error ([Q-name-clash]).

#### Part 2: Assignability

- The function that decides whether a value of one type can be used as another gains one rule: a struct type is assignable to a trait type when the struct implements the trait.
- Passing an argument, returning, assigning to a field, and putting an item in an array literal all go through that function, so they get the rule together.
- The literal `[Circle {...}, Square {...}]` takes its item type from the expected type when there is one (`[Shape]`), the way a whole number literal takes the kind from its context. Without an expected type it stays an error, as above. See [Q-literal-inference].

#### Part 3: Calls

- A method call on a trait type is looked up in the trait's declared methods, and its signature is used for the argument and result checks.
- The compiler emits the same invoke instruction it emits today. No VM change is expected. The test plan has a case to prove it.

#### Part 4: Docs and tooling

- `docs/TYPES.md` Traits section, and the changelog.
- Hover and completion data ([Tooling Data Proposal]) list the trait's methods for a trait typed value.

## Impacts

### Existing Syntax Or Behavior

- `Unknown type.` for a trait name changes to a resolved type. Programs that compiled before are unaffected.
- A struct and a trait with the same name stop compiling ([Q-name-clash]). Nothing in this repository does this.
- `@typeof` and `@instanceOf` on a trait typed value report the struct, because the value is unchanged.

### Related Proposals

- [Generic Types Proposal] — trait bounds (`T: Display`) are the static counterpart. They resolve at compile time and can use `Self`, so `T: Eq` works where `Eq` as a type does not. Section 1.7 there describes the dispatch this proposal relies on.
- [Enums Proposal] — enums can implement traits (Part 2 there), so an enum value converts to a trait type by the same rule. For a closed set of cases an enum is the better tool.
- [Tuple Structs Proposal] — a tuple struct is a struct, so it converts the same way.
- [Primitive Impls Proposal] — a primitive has no object to hold a method table, and trait impls on primitives are rejected today (`Primitive trait implementations aren't supported yet.`). See [Q-primitives].
- [Pattern Matching Proposal] — a trait typed value can not be taken apart with a pattern unless type patterns exist ([Q-downcast]).
- [Result Proposal] — an error type that any struct can implement (`Result[T, Error]`) is a candidate use ([Q-error-trait] there).
- [Collection Methods Proposal] — methods on collection types dispatch through the receiver; a `[Shape]` array is an ordinary array.
- [Macros Proposal] — derive-style macros would generate the impls that make a struct convertible.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it runs is the file of the same name without `.argv`, written `$file` in the arguments. Language tests go in `tests/run/` and run with `run` and `$file`. See `tests/README.md` in kirbylang.

##### NEW: tests/run/traits/trait_type_array.krb

```kirby
trait Shape { fun area(self): f64; }

struct Circle { pub var radius: f64; }
struct Square { pub var side: f64; }

impl Shape for Circle { fun area(self): f64 = 3 * self.radius * self.radius; }
impl Shape for Square { fun area(self): f64 = self.side * self.side; }

let shapes: [Shape] = [Circle { radius: 1 }, Square { side: 2 }];

@println(shapes[0].area());
@println(shapes[1].area());
```

###### Expected Outcome

Prints `3` and `4`. The compiled bytecode for the two calls is the same invoke instruction used for struct receivers.

##### NEW: tests/run/traits/trait_type_parameter.krb

A function takes `Display` and is called with a struct and an interpolated string.

###### Expected Outcome

Prints the `toString()` of each argument.

##### NEW: tests/run/errors/trait_type_struct_does_not_implement.krb

```kirby
trait Shape { fun area(self): f64; }
struct Rock { pub var weight: f64; }

let s: Shape = Rock { weight: 1 };
```

###### Expected Outcome

Does not compile. The error says `Rock` does not implement `Shape`.

##### NEW: tests/run/errors/trait_type_uses_self.krb

```kirby
fun same(a: Eq, b: Eq): bool = a.equals(b);
```

###### Expected Outcome

Does not compile. The error names `Eq` and the method `equals` that uses `Self`.

##### NEW: tests/run/errors/trait_type_method_not_in_trait.krb

A struct has a method its trait does not declare, and the program calls it on a trait typed value.

###### Expected Outcome

Does not compile. The error says the trait has no such method.

## Questions

### **Q:** What is done about traits that use `Self`?

<!-- [Q-self-traits]: #q-what-is-done-about-traits-that-use-self -->

**Status:** Open

Options: (a) Reject any trait that uses `Self` outside the receiver, as Part B says. A simple rule, and it removes `Eq`, `Ord` and `Default` as types. (b) Allow the trait but hide the methods that use `Self`, so only the safe ones can be called. More permissive, and a trait quietly offers fewer methods than it declares. (c) Allow it and check at run time that both values are the same struct. A call can then fail at run time, which the rest of Kirby avoids.

(a) is the recommendation. It gives the error before the program runs, and a program that needs `==` on mixed values can use an enum.

### **Q:** What happens when a struct and a trait have the same name?

<!-- [Q-name-clash]: #q-what-happens-when-a-struct-and-a-trait-have-the-same-name -->

**Status:** Open

Both are accepted today. Options: (a) A compile error at the second declaration. Simple, and keeps a type name meaning one thing. (b) Allow it, and prefer the struct in type positions. A trait type can then not be reached for that name, which is a surprise.

(a) is the recommendation.

### **Q:** Can `[Circle]` be used as `[Shape]`?

<!-- [Q-array-variance]: #q-can-circle-be-used-as-shape -->

**Status:** Open

Lists can be changed (`@listPush`). If `[Circle]` were accepted as `[Shape]`, a function could push a `Square` into it, and the original array would hold a `Square` under a `[Circle]` type.

Options: (a) No. `[Circle]` and `[Shape]` are different types, and the conversion happens per item. Safe, and means a caller builds a `[Shape]` explicitly. (b) Yes, for reading only. It needs a notion of a read-only view that Kirby does not have. (c) Yes, and accept that a push can break the array's type. Not recommended.

(a) is the recommendation.

### **Q:** Does an array literal take its item type from the expected type?

<!-- [Q-literal-inference]: #q-does-an-array-literal-take-its-item-type-from-the-expected-type -->

**Status:** Open

`let shapes: [Shape] = [Circle {...}, Square {...}]` needs each item to be checked against `Shape`, not against the first item. Number literals already take the expected kind from their context (`docs/TYPES.md`, "Number Literals"), including through array literals. Options: (a) Do the same for trait types: with an expected type of `[Shape]`, each item is checked against `Shape`. (b) Require the items to be converted one by one, which needs a conversion syntax that does not exist.

(a) is the recommendation.

### **Q:** Can a number, string or bool convert to a trait type?

<!-- [Q-primitives]: #q-can-a-number-string-or-bool-convert-to-a-trait-type -->

**Status:** Open

A struct instance is an object, and the VM finds its methods through the object. A number is a bare value with no object (`src/value.h`), and `string` and `bool` have no method table either. The [Primitive Impls Proposal] resolves primitive method calls at compile time from the static type, which a trait typed value has lost.

Options: (a) Only structs, enums and tuple structs convert. Primitives are left out until they can be answered. This costs nothing now, because trait impls on primitives are rejected today. (b) Wrap a primitive in an object when it converts, so the method table can be found. A real cost, and a new kind of hidden allocation.

(a) is the recommendation. (b) can be added when primitive impls land.

### **Q:** Can the original type be recovered from a trait typed value?

<!-- [Q-downcast]: #q-can-the-original-type-be-recovered-from-a-trait-typed-value -->

**Status:** Open

`@instanceOf(value, Struct)` already exists for structs. A trait typed value hides the struct, so a program that has a `Shape` and needs the `Circle` has no checked way to get it.

Options: (a) None in this proposal. The trait's methods are the interface. (b) A type test and narrowing, such as a type pattern in `match`. It belongs with the [Pattern Matching Proposal]. (c) A cast with a run time check, like `shape as Circle`. Kirby's `as` today only changes between number types.

(a) is the smallest. (b) is the likely home.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Trait**: A list of methods a type promises to have.
- **Trait type**: A type written as a trait name. A value of it is a value of any type that implements the trait.
- **Dispatch**: Choosing which function to run for a method call. Here, the VM looks the method up by name on the value each time.
- **Bound**: A promise on a generic type parameter that it implements a trait, written `T: Trait`. It is checked at compile time.
- **Receiver**: The value a method is called on, the `self` of the method.
- **Self**: The type an `impl` block is for. In a trait, it stands for whichever type implements it.
- **Variance**: Whether a container of one type can be used as a container of a related type.

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
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Result Proposal]: ../result/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-error-trait]: ../result/PROPOSAL.md#q-what-is-the-error-type

<!-- Questions -->

[Q-self-traits]: #q-what-is-done-about-traits-that-use-self
[Q-name-clash]: #q-what-happens-when-a-struct-and-a-trait-have-the-same-name
[Q-array-variance]: #q-can-circle-be-used-as-shape
[Q-literal-inference]: #q-does-an-array-literal-take-its-item-type-from-the-expected-type
[Q-primitives]: #q-can-a-number-string-or-bool-convert-to-a-trait-type
[Q-downcast]: #q-can-the-original-type-be-recovered-from-a-trait-typed-value
