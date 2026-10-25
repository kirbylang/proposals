---
status: Draft
category: epic
created: 2026-09-18
from_commit: b41332d
---

# Proposal: Generic Types, Macros, Tooling Needs

This document describes a complete type system for Kirby: generics, trait
bounds, a macro system, and the module and tooling support they need. It is
written to stand on its own. Everything it says about how Kirby works today has
been checked against the `main` branch and can be reproduced by building that
branch and running the examples shown. Everything it proposes is described in
enough detail to implement without any other document.

The plain-English rule for reading this: when the text says Kirby "does" or
"has" something, that is a checked fact about `main`. When it says Kirby
"should" or "will" do something, that is a proposal.

**Part 3 (type parameters) is built.** It was merged in [PR #169] and
[PR #170]. Parts 1.5 and 3 describe it as it is on `main`. Parts 4 onward are
still proposals.

---

## How to read this document

Technical terms are kept to a minimum. Where a term is unavoidable, it is
defined in the glossary below the first time it matters. C code is shown in
`c` blocks. Proposed changes to existing C code are shown as `diff` blocks,
where lines starting with `-` are removed and lines starting with `+` are
added.

---

## Glossary

- **AST (Abstract Syntax Tree):** the tree-shaped data structure the parser
  produces from source text. Every later stage reads or rewrites this tree.
  In Kirby this is the `AstNode` type in `src/ast.h`.
- **Bytecode:** the list of small numbered instructions the compiler produces
  and the virtual machine runs. Kirby's instructions are the `OP_*` values in
  `src/opcode.h`.
- **Virtual machine (VM):** the loop in `src/vm.c` that reads bytecode
  instructions one at a time and carries them out.
- **Type checker:** the pass in `src/typecheck.c` that runs after parsing and
  before compiling. It works out the type of every expression and reports an
  error if the program uses a value in a way its type does not allow.
- **Generic:** code written once that works for many types. `struct Box[T]`
  is a generic struct; `T` stands in for whatever type is used later.
- **Type parameter:** the placeholder in a generic declaration. The `T` in
  `struct Box[T]` is a type parameter.
- **Type argument:** the concrete type supplied where a generic is used.
  In `Box[f64]`, `f64` is the type argument.
- **Instantiate:** to produce a concrete version of a generic by filling its
  type parameters with type arguments. `Box[f64]` is an instantiation of
  `Box[T]`.
- **Monomorphization:** one way to implement generics. The compiler makes a
  separate, fully concrete copy of the generic code for each distinct set of
  type arguments actually used. `Box[f64]` and `Box[string]` become two
  separate compiled structs. The name means "turning one shape into many
  single shapes."
- **Trait:** a named set of method signatures a type can promise to provide.
  Other languages call this an interface or a typeclass. Kirby already has
  traits (`src/typecheck.c`, `src/types.h`).
- **Bound:** a requirement written on a type parameter that it must implement
  a given trait, e.g. `T: Display`. Kirby does not have bounds yet.
- **Dispatch:** choosing which function to actually run when a method is
  called. "Static dispatch" means the choice is fixed at compile time.
  "Dynamic dispatch" means the choice is made at run time by looking at the
  value in hand.
- **Interned name:** a piece of text (like a field or method name) stored once
  in a shared table and referred to afterward by a small handle, so that
  comparing two names is a cheap number comparison instead of a
  character-by-character one. Kirby's is `InternedName` in `src/types.h`.
- **Macro:** code that runs at compile time and produces more code. A macro
  takes a piece of syntax and returns a piece of syntax.
- **Hygiene:** the property that names a macro introduces cannot accidentally
  clash with names in the code that used the macro, and vice versa.
- **Module:** a unit of Kirby code that can be compiled on its own and used by
  other code, potentially without that other code having its source.
- **Span:** a record of where a piece of syntax came from in the original
  source (which file, which line, which columns).

---

## Part 1 — What Kirby's type system is today

This part is the checked baseline. Every claim here was verified against a
clean build of `main`. Section 1.5 covers generics, which Part 3 now builds.

### 1.1 The kinds of types

Kirby represents every type as a `Type` struct (`src/types.h`). A `Type` has a
`kind` field that is one of ten values:

```c
typedef enum {
  TYPE_UNIT,
  TYPE_BOOL,
  TYPE_STRING,
  TYPE_NUMBER,
  TYPE_STRUCT,
  TYPE_FN,
  TYPE_ARRAY,
  TYPE_TRAIT,
  TYPE_SELF,          // `Self`
  TYPE_GENERIC_PARAM, // A placeholder such as the `T` in `fun ([T]) => T`
} TypeKind;
```

Numbers are one kind, `TYPE_NUMBER`, with ten number kinds (`i8` to `i64`,
`u8` to `u64`, `f32`, `f64`) held in `as.number.kind`. `TYPE_SELF` is a
placeholder used inside a trait or `impl` block that gets replaced with the
concrete type once it is known. Arrays are written `[T]`: `[i64]` is an array
of `i64`.

`TYPE_GENERIC_PARAM` exists, but only native function signatures can use it.
Signatures are Kirby type text (`src/native_signatures.c`), such as `@listPop`'s
`fun ([T]) => T`, and inside them the names `T`, `U`, `V` and `W` are generic
parameters. A parameter is numbered (0 is `T`, 1 is `U`, and so on) rather than
named, and has no bound. Native signatures can also
write `N` and `M` for the length of an array, as in `fun ([T; N]) => [T; N]`.
They are filled in from the arguments the same way, in the slots after `W`.
They are for native signatures only, and user code still can't write them:

```c
struct {
  // 0 is T, 1 is U, 2 is V, and so on
  int index;
  // The name the program gave it. Native signatures have none.
  InternedName name;
  bool hasName;
  // A declared parameter as seen inside its own declaration: one unknown
  // type that is never filled in and is only equal to itself.
  bool isOpaque;
} genericParam;
```

A native signature makes one with only an index. A program makes one with a
name when it declares `fun id[T]` or `struct Box[T]` (Part 3.1).

### 1.2 How types are compared

Type equality (`typesEqual` in `src/types.c`) is a mix of two rules, depending
on the kind:

- **Structs and traits are nominal.** Two struct types are equal only if they
  are the same named declaration. Two different structs with identical fields
  are still different types. Two instances of a generic struct are equal only
  if they come from the same declaration and all their type arguments are
  equal, so `Box[f64]` and `Box[string]` differ (Part 3.5).
- **Functions and arrays are structural.** Two function types are equal if
  their parameter types and return type are equal, regardless of where they
  were declared. Two array types are equal if their element types are equal.
- Numbers are equal when they are the same number kind. `unit`, `bool`,
  `string` and `Self` are equal to their own kind.
- **Generic parameters are equal when their numbers are equal**, except an
  opaque parameter, which is equal only to itself (Part 3.1). So the two `T`s in
  a native's `fun ([T], T) => unit` are the same type, and the `T` inside
  `fun id[T]` is not equal to any other parameter.

A second function, `typeAccepts` (also `src/types.c`), decides whether a value
can be given where a type is expected. It is `typesEqual` with three
additions: an unknown type accepts and is accepted by anything, the native-only
names `number` and `integer` accept any number or any integer kind, and arrays
and function types are compared part by part with the same rules.

### 1.3 Traits work; bounds do not

Kirby has a working trait system. You can declare a trait, implement it for a
struct, and call the method. This program builds and runs on `main`, printing
`woof`:

```kirby
trait Greet { fun hello(self): string; }
struct Dog { pub var name: string; }
impl Greet for Dog {
    fun hello(self): string = "woof";
}
let d = Dog { name: "Rex" };
@println(d.hello());
```

Facts about the trait system, each checked:

- **Coherence is enforced.** A given trait can be implemented for a given
  struct only once. A second `impl` of the same trait for the same struct is a
  compile error. (`typeStructImplementsTrait`, checked in `typchkRegisterTraitImpl`,
  `src/typecheck.c`.)
- **Methods in a trait-impl block are forced public.** In an `impl Trait for
Type` block, the parser sets every method's visibility to public regardless
  of whether `pub` is written (`src/parser.c`, `isPublic = true` when
  `hasTraitName`). A method in a plain `impl Type` block without `pub` is
  private, as usual. Checked: a non-`pub` trait-impl method is callable from
  outside; a non-`pub` plain-impl method reports `Method 'secret' is private to 'D'.`
- **There are four built-in traits**, always in scope: `Display`, `Eq`, `Ord`,
  and `Default` (`typchkTypeEnvDefineBuiltinTraits`, `src/typecheck.c`).
  `Ord` has `Eq` as a supertrait, and its `cmp` returns an `i64`.
- **Supertraits are enforced regardless of declaration order.** Implementing
  `Ord` for a type that does not also implement `Eq` is an error. On `main`:

  ```
  'P' also needs 'impl Eq for P' -- 'Ord' requires it.
  ```

- **You cannot implement a trait for a primitive.** `impl Eq for f64` reports
  "Primitive trait implementations aren't supported yet." Lifting this
  restriction — independent of generics, bounds, or monomorphization — is its
  own proposal (`/proposals/primitive-impls/PROPOSAL.md`).
  There is **no bound syntax.** You cannot write `fun f[T: Display](x: T)`. The
  grammar has no place for it and the checker has no notion of it.

### 1.4 The `==` operator and `Eq`: a subtle split

This detail matters for the proposal, so it is stated precisely.

The type checker **requires** a struct to implement `Eq` before `==` may be
used on it. Without `impl Eq for P`, the expression `a == b` for two `P`
values is a compile error (in `typchkInferBinary`, `src/typecheck.c`):

```
P needs 'impl Eq for P' to support '=='.
```

But at run time, `==` does **not** call the `Eq` trait's `equals` method. It
compiles to a single `OP_EQUAL` instruction that does built-in value/identity
equality (`valuesEqual`, invoked by the `OP_EQUAL` case in `src/vm.c`). The consequence, checked on `main`:

```kirby
struct P { pub var x: f64; }
impl Eq for P { fun equals(self, other: Self): bool = self.x == other.x; }
let a = P { x: 1 };
let b = P { x: 1 };
@println(a == b);        // prints false -- OP_EQUAL, not equals()
@println(a.equals(b));   // prints true  -- the method really was called
```

So on `main` the built-in traits act as **compile-time gates** — the checker
insists the trait is implemented — but the operator does not actually dispatch
to the trait method at run time. Any design that makes operators call trait
methods (as this proposal does) is changing runtime behavior, not just adding
a check.

### 1.5 Generics: type parameters work, bounds and a few edges do not

Generic syntax is accepted by the parser (`genericParams` on struct, function,
impl, and type-alias nodes, and `genericArgs` on type nodes, in `src/ast.h`),
and since [PR #169] and [PR #170] the type checker handles nearly all of it.
This was checked on a build of `main`:

- A generic struct is declared and used. `struct Box[T] { pub var value: T; }`
  works, `Box[f64]` and `Box[string]` are different types, and
  `let c: Box[string] = a;` with `a: Box[f64]` reports
  `Expected Box[string], got Box[f64].`
- A generic function is declared and called. `fun id[T](x: T): T = x;` works,
  `id(5)` is an `i64`, and `let b: string = id(5);` reports
  `Expected string, got i64.`
- Inside a generic declaration, a parameter is opaque. `fun bad[A, B](a: A): B = a;`
  reports `Expected B, got A.`, and `a + b` on two `T`s reports that `+` needs
  two numbers.
- A generic type alias is declared and used. `type Wrapped[T] = Box[T];` makes
  `Wrapped[i64]` the same type as `Box[i64]`, and the argument count is checked
  (`'Wrapped' needs 1 type argument(s).`).
- A parameter only the return type mentions is filled in from the type the call
  is expected to return. `let s: Stack[i64] = Stack.new();` and
  `let xs: [i64] = empty();` work.
- The number of type arguments is checked. `Box[f64, i64]` reports
  `'Box' takes 1 type argument(s), got 2.` and a bare `Box` as a type reports
  `'Box' needs 1 type argument(s).`

What does not work yet, each checked on `main`:

- **Trait impls on a generic struct.** `impl D for Box { ... }` where `Box` is
  generic reports `Trait implementations on generic structs aren't supported
  yet.`
- **Type arguments on a call.** `make[f64](1)` is read as indexing `make` with a
  variable named `f64`, and fails at run time with `Undefined variable 'f64'.`
- **A parameter only the return type mentions, with no expected type.**
  `let xs = empty();` reports `Can't tell what 'T' is from the arguments.`
- **A generic function with no expected type.** `let f = id;` gives `f` no
  type, and calling `f` is not checked.
- **Bounds.** A value of a type parameter can be passed around, returned and
  stored, and nothing else (Part 3.3).

### 1.6 Generic natives

The one place generics work is calls to native functions. When a native's
signature has generic parameters, `typchkCheckGenericCall` (`src/typecheck.c`)
checks the call. It goes through the arguments once, left to right. A
parameter whose generic parameters are already filled is checked like any
other parameter, so `@listPush(numbers, 2.5)` with `numbers: [i64]` reports
`Expected i64, got f64.` Otherwise the argument's type fills them, as `[i64]`
fills `T` in `[T]`. A lambda argument is checked against the partly filled
type, so its parameters get their types from earlier arguments. A generic
parameter that is never filled is unknown, not an error, so `@listPop([])`
gives an unknown type. A generic native used as a value (`let pop = @listPop;`)
is also unknown.

The helpers are in `src/types.c`: `typeBindGenericParams` fills the slots,
`typeSubstituteGenericParams` puts them in, `typeEraseGenericParams` turns any
left over into unknown, and `typeGenericParamCount` says whether a type still
has any.

Native signatures can also name `number` and `integer`, which accept a number
of any kind or an integer of any kind. They stand in for bounds, but the type
they give back is fixed: `@abs` is `fun (f64) => f64`, because
`fun (number) => number` would lose the argument's kind.

**Summary of the baseline:** Kirby has primitives, structs (nominal), functions
and arrays (structural), a real and enforced trait system with four built-in
traits, and generic structs and functions without bounds (Part 3). The call
check that natives use, `typchkCheckGenericCall`, is the same one user generic
functions use.

### 1.7 How method calls are dispatched today

When Kirby runs `value.method(args)`, the VM looks the method up **by name at
run time** in the receiver's method table. The relevant code is `invoke` and
`invokeFromStruct` in `src/vm.c` (around lines 219–300). `invokeFromStruct`
does a `tableGet` on the struct's `methods` hash table, keyed by the method
name, every time the call runs.

This is dynamic dispatch, and it is the same for every method call — trait
method or not. Nothing about the checker's static type decisions is used at
run time to speed this up; there is no per-call-site cache and no vtable. This
is a checked fact and it shapes the proposal's performance discussion.

### 1.8 The compiler pipeline and passes

The type checker runs as an ordered series of whole-program passes over the
full array of top-level nodes (`typchkCheckProgram`, `src/typecheck.c`).
The order, checked from the source, is:

1. Register struct placeholders (names only).
2. Register traits, detecting duplicates.
3. Register non-generic type aliases (order-independent, cycle-checked).
4. Resolve trait method signatures and supertraits.
5. Detect supertrait cycles.
6. Resolve struct fields.
7. Register impl methods.
8. Check supertrait satisfaction across all impls.
9. Register top-level function signatures.
10. Definite-assignment analysis (`src/definite_assignment.c`).
11. Check function and method bodies.
    The important property for later: the checker already works in multiple
    passes and already sees the whole program at once. Adding a new early pass
    (macro expansion) or threading extra information between passes fits this
    shape.

### 1.9 The compiled-unit format and the runtime

Compiled programs are a `CompiledUnit` (`src/compiled_unit.h`), which
`krb build` writes out as bytes (`src/unit_bytes.c`). Checked contents: it stores compiled functions, their bytecode chunks,
constants (numbers, booleans, nil, strings, function indices), upvalue
records, and a string table. It stores **no type information at all** — no
struct shapes, no trait signatures, nothing a separate compiler or tool could
read to understand a compiled unit's interface. This is the starting point for
the module work in Part 6.

Memory is managed by a mark-and-sweep garbage collector (`src/gc.c`:
`markRoots`, `sweep`, `collectGarbage`). This proposal does not change the
memory model.

---

## Part 2 — Design overview

This part states the shape of the whole design in plain terms before the
detailed parts. Each decision is expanded, with its trade-offs, in the part
named.

1. **Structs, traits, and declared trait bounds** are the core. Bounds are
   _written down_ by the programmer (`T: Display`), never guessed from how a
   type parameter is used. (Parts 3 and 4.)
2. **Generics are implemented by monomorphization.** For each distinct set of
   type arguments actually used, the compiler produces a separate concrete
   copy. There is no runtime type-parameter machinery in the generated code.
   (Part 5.)
3. **Method dispatch stays as it is** — by name, at run time — for now. The
   proposal does not require vtables. It does note where a cache could be
   added later if measurement shows it is needed. (Part 5.7.)
4. **Modules can be distributed as compiled artifacts.** A module ships two
   things: its compiled bytecode, and a separate, readable _interface_ that
   lists its public types, traits, function signatures, and the un-specialized
   bodies of its generic functions. That interface is what lets a separate,
   later compilation instantiate this module's generics with new types, with
   no access to the original source. (Part 6.)
5. **A coherence rule governs impls across modules.** A trait may be
   implemented for a type only in the module that defines the trait or the
   module that defines the type. This is Rust's "orphan rule." (Part 6.4.)
6. **Macros are ordinary Kirby code that runs at compile time** and transforms
   syntax. They are a separate system from generics. Hygiene is automatic and
   composes correctly through nested macros. Macro expansion is its own pass,
   before type checking. (Part 7.)
7. **Every piece of syntax carries a span**, including syntax produced by
   macros, so that tools and error messages can always point back to the
   source the programmer actually wrote. (Part 8.)
   A note on what is deliberately _not_ here: there is no just-in-time compiler
   and no native-code backend in this design. Everything runs on the existing
   bytecode VM. Part 9 records how an ahead-of-time native backend could be added
   later and why nothing here blocks it, but it is out of scope.

### Examples

#### Box

[box.krb]

```kirby
struct Box[T] {
  pub var value: T;
}

impl Box[T] {
  pub fun new(value: T): Self[T] = Self {
    value: value
  };

  pub fun get(self): T = self.value;

  pub fun map[U](self, map: fun (T) => U): Self[U] =
    Self.new(map(self.value));
}

let box_a = Box.new(5);
@println(box_a.get()); // 5

let box_b = box_a.map(double);
@println(box_b.get()); // 10

let box_c = box_b.map(square);
@println(box_c.get()); // 100

let box_d = box_c.map(gt(1000));
@println(box_d.get()); // false

fun square(value: i64): i64 = value * value;
fun double(value: i64): i64 = value * 2;
fun gt(threshold: i64): fun (i64) => bool = fun (value) { value > threshold };
```

#### Point

[point.krb]

```kirby
struct Point {
    pub var x: f64;
    pub var y: f64;
}

impl Point {
    pub fun new(x: f64, y: f64): Self {
        Self {
            x: x,
            y: y,
        }
    }
}

impl Display for Point {
    fun toString(self): string {
        "(" + @numberToString(self.x) + "," + @numberToString(self.y) + ")"
    }
}

impl Add for Point {
    fun add(self, other: Self): Self {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

impl Sub for Point {
    fun sub(self, other: Self): Self {
        Self {
            x: self.x - other.x,
            y: self.y - other.y,
        }
    }
}

impl Div for Point {
    fun div(self, other: Self): Self {
        Self {
            x: self.x / other.x,
            y: self.y / other.y,
        }
    }
}

impl Mul for Point {
    fun mul(self, other: Self): Self {
        Self {
            x: self.x * other.x,
            y: self.y * other.y,
        }
    }
}

let point_a = Point.new(10, 20);
let point_b = Point.new(30, 40);
let point_c = point_a + point_b;
let point_d = point_c - Point.new(5, 5);
let point_e = point_d / Point.new(5, 5);
let point_f = point_e * Point.new(3, 3);

@println(point_a.toString());
@println(point_b.toString());
@println(point_c.toString());
@println(point_d.toString());
@println(point_e.toString());
@println(point_f.toString());

@println((point_a * point_b - point_c / point_d + point_e).toString());
```

---

## Part 3 — Generic type parameters

User code can declare "a type to be chosen later," on functions, structs and
impls, without bounds. It was implemented and merged in [PR #169], with generic
type aliases and expected-type inference added in [PR #170]. The
sections below describe what was built. The earlier plan changed in four ways:
a parameter has two forms instead of one, bounds are left to Part 4, a lookup
for a parameter comes before struct and alias lookups, and a struct instance
holds its arguments instead of a copy of its members.

### 3.1 Named type parameters, in two forms

`TYPE_GENERIC_PARAM` holds an index and, for a parameter a program declared, its
name:

```c
struct {
  // 0 is T, 1 is U, 2 is V, and so on
  int index;
  // The name the program gave it. Native signatures have none.
  InternedName name;
  bool hasName;
  // A declared parameter as seen inside its own declaration: one unknown
  // type that is never filled in and is only equal to itself.
  bool isOpaque;
} genericParam;
```

A declared parameter has two forms, made by `typeDeclaredGenericParam`:

- A **blank** (`isOpaque` false) is the form a signature is stored with. A caller
  fills it in. `typeBindGenericParams` binds it, `typeSubstituteGenericParams`
  replaces it, and `typeGenericParamCount` counts it.
- An **opaque** parameter (`isOpaque` true) is the form the declaration's own
  body sees. Nothing fills it in, so the three helpers above skip it.

```c
case TYPE_GENERIC_PARAM:
  // Opaque parameters are equal only to themselves, which the check at the
  // top already caught
  if (a->as.genericParam.isOpaque || b->as.genericParam.isOpaque)
    return false;
  return a->as.genericParam.index == b->as.genericParam.index;
```

The signature of `fun id[T](x: T): T` is resolved once, with blanks, and stored.
To check the body, the checker runs the stored signature through
`typeSubstituteGenericParams` with opaque parameters as the bindings
(`typchkCheckGenericFunctionBody`). Native signatures keep their blanks without
names, so they work as before.

Indexes number the parameters of one declaration. An `impl Box[T]` takes the
struct's parameters in order, and a method's own parameters are numbered after
the impl's (`env->_implGenericCount`).

### 3.2 Making a parameter name resolve to its type

`TypeEnv` holds a stack of name-to-type bindings for the generic parameters of
the declarations being checked. `typchkEnterGenericParams` pushes one binding per
declared parameter, `typchkLeaveGenericParams` pops them, and a duplicate name
reports `Generic parameter 'T' is already declared.` The stack is pushed before a
declaration's signature, struct fields or body are resolved.

`typchkResolveType` looks in this stack after the built-in names and `Self`, and
**before** structs and aliases, so the innermost name wins:

```c
Type *genericParam =
    bindingArrayLookup(env->_genericParams, env->_genericParamCount, t->name);
if (genericParam != NULL)
  return genericParam;

Type *structType = typchkTypeEnvLookupStruct(env, t->name);
```

A parameter named `T` hides a `struct T` inside its declaration. Outside a generic
declaration the stack is empty, so a bare `T` in ordinary code still reports
`Unknown type.`

### 3.3 Checking a generic function body

With the opaque form in place, a generic body type-checks like any other:

- `fun id[T](x: T): T = x;` is accepted.
- `fun id[T](x: T): T = true;` reports `Expected T, got bool.`
- `fun bad[A, B](a: A): B = a;` reports `Expected B, got A.`

For an unbounded parameter the only things allowed on a value are the ones
allowed for every type: pass it around, return it, store it. Adding it, indexing
it or calling a method on it is an error in the generic definition, not at the
call: `fun add[T](a: T, b: T): T = a + b;` reports that `+` needs two numbers,
got `T` and `T`. Bounds (Part 4) are what widens this.

### 3.4 Checking a call to a generic function

`typchkCheckGenericCall` (Part 1.6), the check natives already used, handles user
functions too. Arguments are checked once, left to right, by `typchkUnifyValue`:

1. Every parameter starts unbound.
2. A parameter type with no blanks left is checked like any other.
3. Otherwise the argument is checked against the type with unfilled blanks
   erased to unknown, then `typeBindGenericParams` fills the first blank each
   argument lines up with. A later argument that disagrees is an ordinary
   mismatch: `Expected i64, got string.` A lambda argument is checked against the
   partly filled type, so its own parameters get types from earlier arguments.
4. A parameter the arguments left empty is then filled from the type the call is
   expected to return, when there is one (`env->_callExpected`). So
   `let s: Stack[i64] = Stack.new();` makes `T` an `i64`.
5. A parameter that appears in the signature but is still unfilled is an error
   for a program's parameter: `Can't tell what 'T' is from the arguments.` For
   a native it stays unknown.
6. The bindings are substituted into the return type.

For `id(5)`, `5` is an `i64`, so `T` is `i64`, and `let a: string = id(5);`
fails with `Expected string, got i64.`

**A generic function used as a value** where a function type is expected is
filled in from that type. `let f: fun (i64) => i64 = id;` makes `T` an `i64`, and
`let g: fun (i64) => string = id;` reports
`Expected fun (i64) => string, got fun (i64) => i64.` A generic method works the
same way, such as `.flat_map(Id.new)` for `fun (T) => Self[U]`
(`typchkFillGenericFunctionValue`). A lambda whose body's type fills a blank, such as
`.flat_map(fun (n) { Id.new(n * n) })`, uses the body's own type
(`typchkCheckInto`, `typeIsFullyKnown`).

A generic function passed into a parameter that is itself generic, as in
`apply(3, id)` for `fun apply[T, U](x: T, f: fun (T) => U): U`, fills `T` and
`U` from the argument. One case is still unchecked: `let f = id;` with no
expected type gives `f` no type, so the check is skipped instead of failing.

### 3.5 Generic structs

A generic struct declaration has `genericParamCount` on its `Type`. Its fields and
methods are resolved with the parameters in scope, so they may mention `T`.

A use such as `Box[f64]` is an **instance**: a separate `Type` of the struct kind
with `typeConstructor` pointing at the declaration and the arguments in
`typeArgs`:

```c
// How many type parameters `struct Box[T]` declares. 0 if none.
int genericParamCount;

// The generic struct this is an instance of: `Box` for `Box[f64]`.
// NULL if this type is a type constructor, or a struct that isn't generic
Type *typeConstructor;
Type **typeArgs;
int typeArgCount;
```

An instance holds no members. Looking up a field or method (`instantiateMember`)
finds it on the declaration and substitutes the arguments, so a method added to
the declaration later is seen by every instance. Two instances are equal when
their declaration and all their arguments are equal, so `Box[f64]` and
`Box[string]` differ. `typeAccepts` is lenient where an argument is unknown.

`typchkResolveGenericInstance` resolves the annotation. It finds the declaration
(or `Self`), checks the count (`'Box' takes 1 type argument(s), got 2.`), and
resolves each argument. A bare `Box` as a type reports
`'Box' needs 1 type argument(s).`

**Struct literals.** `Box { value: 1 }` works out its arguments from its fields
(`typchkInferGenericStructInit`, which shares `typchkUnifyValue` with calls). With a
type expected, the fields are checked against it, so `let a: Box[f64] = Box { value: 1 };`
makes the `1` an `f64` (`typchkCheckGenericStructInit`). A literal that cannot work out
an argument reports `Can't tell the type arguments of 'Box' from its fields.`

**Impls and `Self`.** `impl Box[T]` must declare as many parameters as the struct,
which stand for the struct's in order. In a type, `Self[T]` is the instance with
arguments and a plain `Self` is `Box[T]`. In an expression, `Self` and `Box` are
the declaration, and a call or literal works out the arguments. `impl Trait for` a
generic struct is not supported yet.

### 3.6 A single substitution helper, used everywhere

Replacing type parameters happens in many places: call return types, struct
members, `Self`, function values. It is written once in `src/types.c`:

```c
// Replaces each generic parameter inside `type` with its slot in `bindings`.
// Parameters with an empty slot are kept.
Type *typeSubstituteGenericParams(Type *type, Type **bindings, int count);
```

It recurses into lists, arrays, function parameter and return types, and the type
arguments of struct instances. `typeBindGenericParams`, `typeFindGenericParam` and
`typeGenericParamCount` recurse the same way, and `typeEraseGenericParams` turns
what is left into unknown. Only opaque parameters are skipped.

### 3.7 A caution that still holds

After substitution, a function type that has no parameters left must no longer be
treated as generic, or later call checking would try to infer parameters that are
gone and skip checks that depend on them. The answer to "is anything left to
infer?" comes from the contents of the type. `typeGenericParamCount` walks the
type and counts what is there, and no flag is kept that substitution could
forget to update. Part 5 should keep this rule.

### 3.8 Generic type aliases

An alias can declare type parameters: `type Wrapped[T] = Box[T];`. Aliases are
resolved with the other aliases, in any order, with the parameters in scope as
blanks. A generic alias is stored as `GenericAlias`, with its parameter count and
its target:

```c
typedef struct {
  InternedName name;
  int paramCount;
  Type *target;
} GenericAlias;
```

Using one, as in `Wrapped[i64]`, goes through `typchkInstantiateGenericAlias`: it
checks the argument count, resolves each argument, and runs the target through
`typeSubstituteGenericParams`. So `Wrapped[i64]` is exactly `Box[i64]`, not a
new type. An alias can use other aliases and generic structs, appear in
fields and signatures, and take several parameters. Errors are
`'Wrapped' needs 1 type argument(s).`, `'Wrapped' takes 1 type argument(s), got 2.`
and, for an alias with no parameters, `'Number' doesn't take type arguments.`

---

## Part 4 — Trait bounds

Part 3 gives unbounded type parameters, which can only be passed around. Bounds
let a generic promise that a type parameter supports a trait's methods, so the
body may call them.

### 4.1 Syntax

A bound is written after a type parameter with a colon:

```kirby
fun show[T: Display](x: T): string = x.toString();

struct Wrapper[T: Display] {
    pub var inner: T;
}
```

One bound per parameter to start. (Multiple bounds — `T: Display + Eq` — is an
open question, Part 10.) The grammar change is local: where the parser reads a
generic parameter list (`parseGenericParamList`, `src/parser.c`), allow an
optional `: TraitName` after each parameter name and store it on the AST node
alongside the parameter name.

Native signatures are a good first user of bounds, since they need no syntax
for declaring the parameter. The math natives would gain the most: with a
bounded parameter, `@abs` could be `fun abs[N: number](n: N): N` and keep the
argument's kind, where today it takes and returns `f64` (Part 1.6). `number`,
`integer` and `float` are not traits, so this needs either a new kind of bound
or built-in traits for the number families.

### 4.2 What a bound means to the checker

When a parameter `T` is bound to trait `Tr`:

1. At the **definition**, the checker records `Tr` on the `TYPE_GENERIC_PARAM`
   (a new `boundTrait` field and a `hasBound` flag next to `name` and `isOpaque`
   from Part 3.1). Inside the body, a value of type `T`
   is now allowed to call exactly the methods `Tr` declares — nothing more. The
   method's result type is whatever `Tr`'s signature says (with `Self` read as
   `T`). Calling a method `Tr` does not declare remains an error.
2. At a **call site**, once unification (Part 3.4) has bound `T` to a concrete
   type, the checker verifies that concrete type implements `Tr`, using the
   existing `typeStructImplementsTrait` (declared in `src/types.h`). If it does not,
   that is the error — reported at the call site, against the specific type
   supplied:
   ```
   NoDisplay doesn't implement Display, required by T.
   ```

This is the whole point of declared bounds: the requirement is a fact written
in the signature. The checker reads it directly. It never inspects the body to
discover what the parameter needs. That keeps the requirement checkable from
the signature alone — which Part 6 relies on, because a distributed module
ships signatures but may not ship bodies.

### 4.3 Bounds replace any usage-based inference

A tempting shortcut is to _infer_ a bound by scanning a generic body for
operators or method calls and requiring whatever they imply. This design does
**not** do that, for concrete reasons:

- The requirement would only be discoverable by reading the body. A module
  that ships compiled code without source could not be checked against.
- Bugs hide easily. A path that checks some uses but not others (for example,
  a value reached through the receiver of a method versus through an ordinary
  argument) silently accepts wrong programs.
- It does not generalize. Each new operator or trait needs new special-case
  scanning code.
  A declared bound has none of these problems: it is one uniform mechanism, it
  lives in the signature, and it is checked by a single trait-implements lookup.

### 4.4 Operators as trait methods

Given bounds, arithmetic and comparison operators become ordinary trait method
calls. Define built-in traits for them — for example an `Add` trait with a
method `add(self, other: Self): Self` — and read `a + b` as `a.add(b)` once
both sides' type is known to implement `Add`.

This is where the runtime split from Part 1.4 must be resolved. Today `==`
type-checks against `Eq` but runs as a built-in `OP_EQUAL` that ignores the
trait method. If operators are to work for user types through their trait
impls, the operator must actually dispatch to the trait method for those types.
Two coherent options:

- **Keep the fast path for primitives, dispatch for the rest.** For number
  operands, `+` stays the built-in numeric instruction. For struct operands,
  `+` compiles to a call to the type's `add` method. The operator opcode
  checks operand kind and either does the built-in thing or performs a method
  call. This matches how `OP_EQUAL` already special-cases built-in equality;
  it extends the same shape to route struct operands through the trait method.
- **Make every operator a trait method uniformly**, including for numbers, by
  giving primitives real trait implementations (which requires lifting the
  current "primitive trait implementations aren't supported yet" restriction;
  the primitive-impls proposal (`/proposals/primitive-impls/PROPOSAL.md`)
  designs that lift on its own, independent of this proposal, and should be
  consumed here rather than re-derived if this option is chosen).
  Cleaner in principle, more work, and pays a dispatch cost on the hottest
  path (plain number arithmetic) unless specially optimized.
  The first option is the smaller, safer step and is recommended. Whichever is
  chosen, note clearly that this **changes runtime behavior**: after this, `==`
  (and the arithmetic operators) on a struct will call the trait method, where
  today `==` does built-in identity/value equality regardless of the `Eq` impl.
  The equality change in particular needs its own tests and a changelog note,
  because existing programs that rely on today's `==`-is-identity behavior would
  observe a difference.

### 4.5 Built-in trait dispatch table

The operators, their trait, and the trait's method belong in **one** table
that the three places needing them all read from: trait registration, the
checker's operator handling, and the compiler/VM's operator lowering. A single
source such as:

```c
// operator token, trait name, method name
{ TOKEN_PLUS,  "Add", "add" },
{ TOKEN_MINUS, "Sub", "sub" },
{ TOKEN_STAR,  "Mul", "mul" },
{ TOKEN_SLASH, "Div", "div" },
```

Driving all three sites from this one table prevents them drifting apart — for
instance, the checker requiring a trait the compiler does not actually call, or
two operators' error messages ending up worded inconsistently. Hand-writing
each operator separately in each place is exactly how such drift happens.

---

### 4.6 Bounds and trait types

A bound is the static way to use a trait: the concrete type is known at each
call, and the compiler can specialize (Part 5). The [Trait Types Proposal] adds
the dynamic way: a trait name where a type is expected, so one array can hold
values of different structs that implement the trait. Both use the same
"does this type implement the trait" lookup (`typeStructImplementsTrait`). A
trait that mentions `Self` outside its receiver, like `Eq`, can be a bound but
can not be a trait type, which is one reason a bound is not a replacement for
it.

## Part 5 — Monomorphization

Parts 3 and 4 let generics be _written_ and _checked_. This part says how they
are _compiled and run_: by producing a separate concrete copy of a generic for
each distinct set of type arguments actually used.

### 5.1 Why monomorphization

The goal is that generic code runs as fast as hand-written concrete code, with
no runtime cost for having been generic. After monomorphization there are no
type parameters left in the compiled program — `Box[f64]` is just a struct with
an `f64` field, and `sum[f64]` is just a function that adds two `f64`s. The VM
never sees a `T`.

The costs, stated honestly:

- **Code size.** Each distinct instantiation is its own compiled code. Ten
  different `Box[...]` types mean ten compiled structs. For a small embeddable
  scripting language this is usually acceptable, but it is a real cost.
- **Compile-time work.** Specializing happens at compile time, so heavily
  generic programs do more work to compile.
  Neither cost falls on the running program, which is the point.

### 5.2 What gets specialized

Specialization is driven by **use**, not by declaration. A generic that is
never instantiated produces no code. When the compiler encounters a concrete
instantiation — `Box[f64]`, `sum[string]` — it:

1. Checks whether a specialization for that exact set of type arguments already
   exists. If so, reuse it.
2. Otherwise, take the generic's checked body, substitute the type arguments
   for the type parameters throughout (the Part 3.6 helper), and compile the
   result as an ordinary concrete function or struct.
   Because two instantiations with the same arguments share one specialization,
   using `Box[f64]` in a hundred places produces one compiled `Box[f64]`, not a
   hundred.

### 5.3 Where specialization sits in the pipeline

Monomorphization is a compile-time step that runs after type checking (so the
generic is known to be correct) and feeds the existing compiler (so each
specialization is compiled by the code that already compiles concrete
declarations). It does not touch the VM. This keeps the change contained: the
VM, the bytecode format, and the garbage collector are all unchanged.

### 5.4 Nested and recursive generics

`Box[Pair[f64]]` requires `Pair[f64]` to be specialized first, then `Box[...]`
around it. The substitution helper (Part 3.6) already recurses, so producing
the concrete inner type is automatic; the specializer just needs to specialize
inner instantiations before the outer one that contains them.

Generic code that instantiates _itself_ with an ever-changing type argument
could in principle ask for infinitely many specializations. This is a known
situation for monomorphizing languages. A simple, safe answer for a first
version is to bound the depth of specialization and report an error if it is
exceeded, rather than loop forever. (A more complete answer is an open
question, Part 10.)

### 5.5 The interaction with modules

Monomorphization needs the generic's **body** at the point of instantiation.
When the generic and the instantiation are in the same compilation, the body is
right there. When the generic lives in a separately compiled module, the body
must travel with the module. This is exactly why Part 6's module interface
includes generic bodies, not just signatures. The two parts are designed
together.

Crucially, this does **not** require a just-in-time compiler. Specialization
happens when the _consuming_ code is compiled — ahead of time — using the body
the module shipped. Rust works this way: it has no JIT, and it monomorphizes
across separately compiled crates by carrying each generic's body in the
crate's metadata. Kirby follows the same approach.

### 5.6 What the running program sees

Nothing generic. Every function is concrete, every struct is concrete, every
method call is a call to a concrete method. The type parameters existed only at
compile time. This is why monomorphization is a good match for a possible
future native backend (Part 9): concrete code is exactly what compiles well to
fast machine code.

### 5.7 Dispatch cost and a possible future cache

Ordinary method calls still dispatch by name at run time (Part 1.7), even after
monomorphization, because monomorphization removes type _parameters_, not
dynamic dispatch. For a call like `x.foo()` where the checker already knows
`x`'s exact type, the run-time lookup repeats work the compiler could have
settled.

This design does not change dispatch. It records, for whoever measures a
bottleneck later, the standard cheap improvement: cache, at each call site, the
method found last time, and skip the table lookup when the next receiver has
the same type. This is a self-contained optimization that needs no change to
the type system or the module format. It is explicitly out of scope here and
listed only so the option is on record.

One concrete, low-risk improvement is worth noting because it is free of any
design trade-off: the method-table lookup uses `%` (integer division) to reduce
a hash to a slot (`findEntry`, `src/hashtable.c`), while the table's capacity is
always a power of two (`GROW_CAPACITY`, `src/gc.h`, doubles from 8). A
power-of-two modulo can be replaced by a bit-mask (`hash & (capacity - 1)`),
which is faster and always gives the same result. This is a safe change on the
hottest path and does not depend on anything else in this document.

---

## Part 6 — Modules

A module is a unit of Kirby code that can be compiled on its own and used by
other code that may not have its source. This part describes what a module must
carry and the rules that make cross-module generics and impls well-defined.

### 6.1 The problem modules introduce

Today the checker always has the whole program's source at once
(`typchkCheckProgram`, Part 1.8). A module breaks that assumption: code that
uses a module may be compiled long after the module, on a different machine,
with only the module's compiled form on hand. Two capabilities then need
information the current compiled form does not carry:

- **Type checking against the module.** To check `box.value` where `box` came
  from another module, the consumer's checker needs that module's struct
  shapes, trait definitions, and function signatures.
- **Monomorphizing the module's generics.** To instantiate the module's
  `Box[T]` with the consumer's own type, the consumer's compiler needs the
  _body_ of `Box`'s generic code (Part 5.5).
  The current `CompiledUnit` carries neither — it stores only bytecode,
  constants, and strings, with no type information at all (Part 1.9).

### 6.2 The module interface

Alongside its compiled bytecode, a module ships a second artifact: its
**interface**. Think of it as the module's published summary — the equivalent
of a C header file, but richer. It lists, for everything the module makes
public:

- struct names, their fields, and field types;
- trait definitions (their method signatures and supertraits);
- which structs implement which traits;
- function and method signatures, including any generic parameters and their
  bounds;
- for every generic function and struct, the **un-specialized body**, in a
  form the compiler can specialize later (the checked AST or a compact
  intermediate form derived from it).
  The first several items are the "header": enough for a consumer's _checker_ to
  type-check code that uses the module. The last item is the "recipe": enough for
  a consumer's _compiler_ to monomorphize the module's generics against new
  types (Part 5.5). Both are needed; they serve different stages.

This same interface is what external tooling (Part 8) reads to answer "what
does this module contain" without running the whole compiler — so it is worth
designing to be complete and stable, not minimal.

### 6.3 What "public" means

A module needs a boundary between what it exposes and what is internal. Kirby
already has a `pub` marker on struct fields and methods (`src/parser.c`).
Extending that idea, a module's interface contains exactly its `pub` items and
the types they mention. The exact surface (whether whole structs, functions,
and traits also carry `pub`, and defaults) is a detail to settle during
implementation; the principle is that the interface is the public surface and
nothing more.

### 6.4 Coherence across modules (the orphan rule)

Within one program the rule is simple: a trait may be implemented for a struct
only once (Part 1.3). Across modules a new hazard appears. Two unrelated
modules could each write `impl SomeTrait for SomeType` for a trait and a type
that neither of them defines. A third module using both now faces two
conflicting implementations with no principled way to choose.

The rule that prevents this — **the orphan rule**, as in Rust:

> An `impl Trait for Type` is allowed only in the module that defines `Trait`
> or the module that defines `Type`.

You may always implement your own trait for anyone's type, and anyone's trait
for your own type. You may not implement someone else's trait for someone
else's type. Because at least one side is always "yours," two different modules
can never both be allowed to write the same conflicting impl, so conflicts
cannot arise.

When a programmer genuinely needs to implement a foreign trait for a foreign
type, the standard workaround is the **newtype pattern**: wrap the foreign type
in a thin struct you own, and implement the trait for your wrapper. You own the
wrapper, so the impl is allowed.

This rule is checkable by each module on its own, using only its own source and
the interfaces of the modules it depends on — no global view of all modules at
once is required. That is what makes it suitable for separate compilation.

An alternative considered and set aside: allow any impl anywhere and detect
conflicts only when a whole program is finally assembled. It needs no ownership
rule, but it lets a conflict lie hidden until two particular modules are
combined, producing a late and confusing error. The orphan rule's up-front
restriction is preferred precisely because it turns a possible late failure
into an immediate, local one.

### 6.5 Whole-program assembly

Even with separately compiled modules, the final program is assembled from a
module and everything it depends on. At that point the specializer (Part 5) has
every generic body it needs (from each module's interface) and can produce all
required specializations. The result is a single runnable program in which
nothing generic remains.

---

## Part 7 — Macros

Macros are code that runs at compile time and produces code. This part
describes a macro system that is separate from generics, powerful, hygienic,
and friendly to tooling.

### 7.1 What a macro is

A macro is an ordinary Kirby function with one difference: it runs during
compilation, and instead of ordinary values it receives and returns **syntax**
— pieces of the program's own AST. A macro takes syntax in and gives syntax
back. The compiler replaces each macro call with the syntax the macro returns,
then carries on.

This is deliberately "macros are just Kirby code." A macro is written in Kirby,
using Kirby's ordinary control flow and data structures, operating on syntax as
data. There is no separate macro mini-language and no pattern-template
sub-language to learn.

### 7.2 Powerful, but not by fusing with generics

Some languages make one compile-time-execution mechanism serve as _both_ the
macro system and the generics system. This design keeps them separate.
Generics are declared, checked, bounded (Parts 3–4), and monomorphized
(Part 5). Macros are a different tool for a different job: generating and
transforming syntax. Keeping them apart means generics stay statically
checkable from signatures (which Part 6 needs) and are not turned into
"run some code and see what comes out."

Making macros "just Kirby code" does not require the two to share machinery.
The power comes from macros being real code over syntax, not from merging them
with the type system.

### 7.3 Running macros: reuse the VM

A macro is Kirby code, so the compiler can run it the same way Kirby runs any
code: compile the macro to bytecode and execute it on an instance of the
existing VM, at compile time, handing it the syntax of its call and taking back
the syntax it returns. No separate interpreter for macros needs to be built;
the VM Kirby already has is the macro evaluator.

### 7.4 Hygiene

Hygiene is the guarantee that names introduced by a macro cannot accidentally
collide with names in the code that called it, and that names the caller passes
in are resolved in the caller's world, not the macro's. A macro that
introduces a helper variable `tmp` must not clobber a `tmp` the caller already
has.

The requirement for Kirby: hygiene is **automatic** and **composes** — it keeps
working when a macro expands into a call to another macro, without the macro
author having to place manual "escape" annotations to get ordinary cases right.
Automatic, composable hygiene is achievable — it is a solved problem in the
Lisp/Scheme tradition — by attaching, to each piece of syntax, information
about the scope it came from, and honoring that information during name
resolution rather than comparing names as bare text.

The practical implication for Kirby's data structures: a syntax value is not
just a name and a shape; it also carries where it came from. That is the same
information spans carry for error reporting (Part 8), so the two are designed
together.

### 7.5 Where expansion sits in the pipeline

Macro expansion is its own pass, and it runs **before** type checking:

```
parse  ->  expand macros  ->  type check  ->  monomorphize  ->  compile
```

The type checker (Part 1.8) then only ever sees fully expanded code and needs
no awareness of macros. This ordering — expand, then check — is the
straightforward one and fits Kirby's existing multi-pass checker, which already
runs as an ordered sequence of whole-program passes.

For completeness, other languages order things differently: some interleave
expansion with name resolution because a macro can introduce names a later
macro call needs resolved; some check a generic template's body per
instantiation rather than once; some interleave compile-time execution directly
with type checking. Those are more invasive to Kirby's current pipeline than a
separate expansion pass and are not proposed. (The interleaving question, if
macros ever need to see resolved names, is noted in Part 10.)

### 7.6 Declaration-producing macros

A useful macro often needs to produce a whole new **declaration**, not just an
expression — for example, generating an `impl` block. This is a larger
capability than "a macro that expands to an expression," and the macro system
should support it explicitly. It is the foundation for derive-style macros
(next).

### 7.7 Derive-style macros

A common convenience is to auto-generate a trait implementation from a struct's
shape — the equivalent of "derive `Eq` for this struct." With the pieces above,
this is expressible without a separate feature: a derive is a
declaration-producing macro (7.6) that reads a struct's fields (via the
reflection capability below) and returns an `impl` block.

A sensible build order, smallest useful piece first:

1. **Reflection over a concrete type.** Give compile-time code a way to ask,
   of a struct, "what are your fields and their types?" This is the same
   information the module interface already records (Part 6.2), so it is one
   body of data with two readers.
2. **One hard-coded derive, end to end.** Pick a single trait (`Eq` is a
   natural first target since it already exists and compares field by field)
   and generate its `impl` as a special case, to prove a generated `impl` can
   be injected and behaves exactly like a hand-written one.
3. **Generalize.** Turn the special case into an ordinary
   declaration-producing macro any programmer could write.
   This is listed as a capability the macro and reflection design should _enable_,
   not as immediate work.

---

## Part 8 — Tooling and span tracking

Strong tooling — precise error messages, go-to-definition, find-references,
autocomplete — is a first-class goal, and macros make it harder unless planned
for. This part states the one piece of infrastructure that must be present from
the start. The design of that data (what a span holds, how it is stored, and
what reaches the compiled output) is in the [Tooling Data Proposal]. This part
records what the type-system work needs from it. The types and declarations the
checker finds are recorded for tools by the [Type Info Proposal]. A generic body
is checked once (3.3), so its entries there show its parameters, such as `T`.
Monomorphization runs after checking (5.3) and adds none.

### 8.1 The problem macros create for tooling

Once code can be generated (Part 7), the code that runs is not always the code
the programmer typed. A method may exist only because a derive macro produced
it. A type error may occur inside generated syntax. If tools and error messages
describe the _generated_ code by its own location, the programmer sees errors
pointing at code they never wrote — the single most common complaint about
macro systems in practice.

### 8.2 Spans, carried through expansion

The fix is that every piece of syntax carries a **span**: a record of where in
the original source it came from — which file, which line and columns. Kirby's
tokens already carry line information (`Token` in `src/token.h`); this extends
that idea to every AST node and every piece of syntax a macro produces.

The essential requirement: when a macro generates syntax, the generated syntax
carries a span that points back to the macro call the programmer wrote (and,
through nested macros, back through each layer). Then any tool or error message
can always translate "this happened in generated code" into "here is the source
line responsible."

This is the same per-syntax origin information hygiene needs (Part 7.4), so the
two are one design, not two.

### 8.3 Build it in from the start

Span tracking must be part of the AST/syntax data structures from the
beginning. Retrofitting "where did this come from" onto data structures that
were not built to carry it is a well-known source of pain: every place that
constructs or rewrites syntax has to be revisited. Building it in from the
start — every node has a span, every syntax-producing operation sets a
meaningful one — avoids that. This applies to the generics and macro work in
this document: as those passes create and rewrite syntax and types, they should
set spans, not leave them to be added later.

### 8.4 Debugging

The [Debugger Proposal] sets breakpoints by source file and line. After
monomorphization one generic function is compiled once for each set of type
arguments (Part 5.2), so a breakpoint on a line inside it has to stop in every
copy. That needs no extra design as long as each specialization keeps the
generic's source file and line numbers when its checked body is substituted and
compiled (Part 5.2, step 2). This is the same rule 8.3 sets for spans. A
specialization should also have a readable name in the call stack, such as
`sum[f64]`, rather than an internal one.

---

## Part 9 — Ahead-of-time native compilation (deferred)

This is out of scope but recorded so that nothing above accidentally blocks it.

Ahead-of-time (AOT) native compilation means turning a program all the way into
machine code, with no bytecode interpreter running it. This design does **not**
propose that. Everything above runs on the existing bytecode VM.

Two facts are worth stating so the option stays open:

- **Bytecode can remain the central form.** An AOT path can be added later as an
  additional consumer of the same bytecode — one that emits machine code
  instead of interpreting — without changing the front end (parsing, macro
  expansion, type checking) or the bytecode format. Deferring AOT does not risk
  a rewrite; it defers building a new back end.
- **This design's choices help, not hinder, a future AOT path.**
  Monomorphization (Part 5) produces fully concrete code with no type
  parameters left, which is exactly what compiles well to fast machine code.
  Declared bounds (Part 4) and the module interface (Part 6) provide the static
  facts such a back end would also want.
  One honest note for whoever picks this up: a straightforward translation of
  bytecode to machine code removes interpreter overhead and is a real but modest
  win. Getting substantially more requires an optimizing back end (inlining,
  register allocation, and so on), which is a large project in its own right —
  larger than anything proposed here. That is a reason to defer it, not a reason
  it is precluded.

---

## Part 10 — Open questions

Each entry states the question, the options, and why it is open. These are
genuine decisions left for implementation, not hidden assumptions.

### 10.1 Multiple bounds per type parameter

**Question:** should a type parameter be allowed more than one bound, e.g.
`T: Display + Eq`?
**Options:** (a) one bound per parameter to start, add more later; (b) support
a set of bounds from the beginning.
**Why open:** one bound is simpler and covers many cases; a set is more
expressive but complicates the parser, the `TYPE_GENERIC_PARAM` representation
(one trait name versus a list), and the call-site check (verify all bounds).
Part 4 assumes (a); moving to (b) is additive.

### 10.2 Type parameters that appear only in the return type

**Question:** how should a call be checked when a type parameter cannot be
inferred from the arguments because it appears only in the return type (e.g. a
`parse[T](s: string): T`, or `Stack.new()` for `fun new(): Self[T]`)?
**Options:** (a) forbid it — every parameter must be inferable from arguments;
(b) allow the call site to supply type arguments explicitly, e.g.
`parse[f64](s)`; (c) infer from the surrounding expected type where one exists.
**Status:** (c) is built ([PR #170]). A parameter the arguments left empty is
filled from the expected return type, so `let s: Stack[i64] = Stack.new();` and
`return empty();` work. With no expected type the call still reports
`Can't tell what 'T' is from the arguments.`
**Why still open:** Explicit type arguments at a call do not parse: `make[f64](1)`
is read as indexing `make` with a variable named `f64`, and fails at run time with
`Undefined variable 'f64'.` So (b) needs the parser to tell type arguments from an
index, and the checker and monomorphizer must then accept and thread explicit
arguments. (b) is still needed where no type is expected, such as
`let n = parse[f64](s);` with no annotation, or as an argument to a call.

### 10.3 Operators for primitives: built-in fast path or uniform trait dispatch

**Question:** should arithmetic and comparison operators on numbers remain
built-in instructions, or go through trait methods uniformly like struct
operands?
**Options:** (a) keep the built-in fast path for primitives, dispatch only
struct operands to trait methods (Part 4.4, recommended); (b) make every
operator a trait method for all types, requiring primitive trait impls and
accepting dispatch cost on numeric arithmetic unless optimized.
**Why open:** (a) is faster and smaller but means primitives and structs take
different paths for the "same" operator; (b) is more uniform but slower on the
hottest path and needs the "primitive trait implementations aren't supported
yet" restriction lifted first (the primitive-impls proposal proposes exactly
that lift, independent of this document).

### 10.4 Changing `==` runtime semantics

**Question:** when `==` starts dispatching to `Eq` for structs (Part 4.4), how
is the behavior change from today managed?
**Options:** (a) change it and document the break; (b) keep `OP_EQUAL`
identity/value equality and require a differently named method or operator for
trait-based equality; (c) make `==` dispatch to `Eq` only for types that
implement it and keep built-in equality otherwise.
**Why open:** today `==` type-checks against `Eq` but runs as built-in
equality (Part 1.4). Any of these resolves the split, but they differ in how
much existing behavior changes and how surprising the result is. Needs tests
and a changelog note whichever way it goes. The [Tuples Proposal]
([Q-equality-tuples]) and the [Enums Proposal] ([Q-equality-enums]) each ask
what `==` means for their own values, so the answer here affects both.

### 10.5 Recursive/infinite monomorphization

**Question:** how should the specializer handle generic code that would request
infinitely many specializations of itself?
**Options:** (a) bound the specialization depth and error when exceeded
(simple, safe first version, Part 5.4); (b) a more complete detection of
genuinely unbounded instantiation.
**Why open:** (a) is easy and prevents a hang but can reject some legitimate
deep-but-finite cases; (b) is more precise and more work.

### 10.6 Module public-surface details

**Question:** exactly which declarations carry `pub`, and what the defaults are,
for the module boundary (Part 6.3).
**Options:** extend `pub` to whole structs / functions / traits with
private-by-default, or public-by-default with an explicit private marker, or
some mix.
**Why open:** Kirby has `pub` on fields and methods today; the module-level
surface is not yet designed. The choice affects what lands in the module
interface and how much is exposed by accident.

### 10.7 Whether macros ever need resolved names during expansion

**Question:** is "expand fully, then type-check" (Part 7.5) always sufficient,
or will some macros need name-resolution information mid-expansion?
**Options:** (a) keep the clean phase split; (b) interleave expansion with name
resolution if a real need appears (as some languages do).
**Why open:** the clean split is far simpler and is the proposal. Whether any
intended macro genuinely needs resolved names before expansion finishes is not
yet known; if one does, the pipeline would need the more complex interleaving.

### 10.8 Intermediate form for shipped generic bodies

**Question:** in what exact form does a module ship its generic bodies
(Part 6.2) — the checked AST directly, or a more compact intermediate
representation derived from it?
**Options:** (a) ship the AST; (b) design a smaller, stable intermediate form.
**Why open:** the AST is simplest and already exists but is larger and ties the
on-disk format to the AST's shape; a dedicated intermediate form is more work
but more stable across compiler versions. This interacts with how the
`CompiledUnit` format (which today stores no types at all) is extended.

### 10.9 Tuple and enum types in substitution and monomorphization

**Question:** how do tuple types and enum types fit into the substitution
helper (3.6) and the specializer (Part 5)?
**Options:** (a) treat both as ordinary kinds of type from the start. The
helper recurses into a tuple's item types and into the type arguments of an
enum, and an enum declared with type parameters is specialized the way a
generic struct is; (b) leave them out of the first version, so
`[(f64, f64)]` and `Option[T]` are rejected until a later change.
**Why open:** the [Tuples Proposal] adds a type compared by its parts (like
function and array types) whose parts can contain type parameters, and the
[Enums Proposal] adds a named type (like a struct) that can have type
parameters itself: `Option[T]` and `Result[T, E]`. `@argv`, `@prompt` and
`@stdin` have no signature until `Option[T]` exists (`docs/TYPES.md`). Both need the same "write it once, call it
everywhere" treatment that 3.6 gives the existing kinds, or one path will
miss a case. (a) costs more up front. (b) is smaller, but leaves `Option[T]`
unavailable, and that is what the natives pending it need. The [Map Proposal]
(`Map[K, V]`) and the [Result Proposal] (natives that return `Result[T, E]`)
need the same treatment, and the second also needs native signatures to be able
to write a generic enum type.

---

## Appendix A — Summary of C-level changes

A checklist of the concrete code touched, by area. This is a map, not a
substitute for the parts above.

- **`src/types.h` / `src/types.c`**
  - done ([PR #169]): a name and an opaque form on `TYPE_GENERIC_PARAM`,
    struct instances with `typeConstructor` and `typeArgs`, and the one
    substitution helper that reaches into them — Parts 3.1, 3.5, 3.6;
  - add an optional bound to `TYPE_GENERIC_PARAM`'s `as.genericParam` payload
    — Part 4.1.
- **`src/typecheck.c` / `src/typecheck.h`**
  - done ([PR #169]): a generic-parameter scope on `TypeEnv`, bound while a
    declaration is checked; call-site unification; generic structs and
    impls — Parts 3.2 to 3.5;
  - done ([PR #170]): fill a parameter that only the return type mentions from
    the expected type, and generic type aliases — Parts 3.4, 3.8;
  - explicit type arguments at a call — Part 10.2;
  - record and enforce bounds; at call sites, check the bound trait is
    implemented via `typeStructImplementsTrait` — Part 4.2;
  - the single operator/trait/method table and operator handling read from it
    — Part 4.5.
- **`src/parser.c` / `src/ast.h`**
  - accept `: TraitName` after a generic parameter and store it — Part 4.1
    (the `genericParams` arrays already exist);
  - spans on every AST node, and a syntax representation macros can produce
    — Parts 8, 7.1.
- **Compiler / monomorphizer (new work, after the checker)**
  - a specialization step that, driven by use, substitutes type arguments into
    a checked generic body and compiles the concrete result, memoized per
    argument set — Part 5;
  - operator lowering that routes struct operands to trait methods while
    keeping the primitive fast path — Part 4.4.
- **`src/vm.c`**
  - unchanged for generics (monomorphized code is ordinary concrete code);
  - only if operator-on-struct dispatch is added: the operator opcodes route
    struct operands to a method call — Part 4.4;
  - optional, out of scope: call-site method cache, and the power-of-two
    mask-instead-of-modulo change in `src/hashtable.c` — Part 5.7.
- **`src/compiled_unit.c` / `src/compiled_unit.h`** and a new interface artifact
  - a module interface carrying public type/trait/signature information and
    un-specialized generic bodies — Part 6.2 (today's `CompiledUnit` stores no
    type information at all).
- **Macro subsystem (new work)**
  - an expansion pass before type checking that runs macros on the VM and
    substitutes their returned syntax; hygiene via per-syntax scope
    information; support for declaration-producing macros — Part 7.
- **Reflection (new work, shared with modules)**
  - compile-time access to a concrete struct's fields and types, reading the
    same data the module interface records — Part 7.7.

## Appendix B — Reproducing the baseline claims

Every "today" claim in Part 1 can be checked on a clean `main` build at
`from_commit`:

```shell
# build
just build                 # produces build/krb

# generic structs and functions work
./build/krb exec 'struct Box[T] { pub var value: T; } let a: Box[f64] = Box { value: 1 }; let c: Box[string] = a;'
# "Expected Box[string], got Box[f64]."

./build/krb exec 'fun id[T](x: T): T = x; let b: string = id(5);'
# "Expected string, got i64."

./build/krb exec 'fun bad[A, B](a: A): B = a;'
# "Expected B, got A."

# the edges that are not built
./build/krb exec 'type W[T] = [T]; let a: W = [1];'
# "'W' needs 1 type argument(s)."

./build/krb exec 'fun empty[T](): [T] = []; let xs = empty();'
# "Can't tell what 'T' is from the arguments."

./build/krb exec 'fun make[T](x: T): T = x; let n = make[f64](1);'
# "Undefined variable 'f64'." (at run time)

# array types and generic natives work
./build/krb exec 'let xs: [f64] = [1, 2.5]; @println(xs);'
# [1, 2.5]
./build/krb exec 'let numbers: [i64] = [1]; @listPush(numbers, 2.5);'
# "Expected i64, got f64."

# traits work
./build/krb exec 'trait G { fun hello(self): string; } struct D { pub var n: string; } impl G for D { fun hello(self): string = "woof"; } let d = D { n: "x" }; @println(d.hello());'
# woof

# == checks Eq but runs as identity equality
./build/krb exec 'struct P { pub var x: f64; } impl Eq for P { fun equals(self, other: Self): bool = self.x == other.x; } let a = P { x: 1 }; let b = P { x: 1 }; @println(a == b); @println(a.equals(b));'
# false, then true

# supertraits enforced regardless of order
./build/krb exec 'struct P { pub var x: f64; } impl Ord for P { fun cmp(self, other: Self): i64 = 0; }'
# "'P' also needs 'impl Eq for P' -- 'Ord' requires it."

# primitive impls rejected
./build/krb exec 'impl Eq for f64 { fun equals(self, other: Self): bool = true; }'
# "Primitive trait implementations aren't supported yet."
```

## Link References

[PR #169]: https://github.com/kirbylang/kirbylang/pull/169
[PR #170]: https://github.com/kirbylang/kirbylang/pull/170
[box.krb]: ./examples/box.krb
[point.krb]: ./examples/point.krb
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Type Info Proposal]: ../type-info/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[Map Proposal]: ../map/PROPOSAL.md
[Result Proposal]: ../result/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md
[Q-equality-tuples]: ../tuples/PROPOSAL.md#q-how-does-tuple-equality-compare-items
[Q-equality-enums]: ../enums/PROPOSAL.md#q-how-does-equality-work-on-enums
