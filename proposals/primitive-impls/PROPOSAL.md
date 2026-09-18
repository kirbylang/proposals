---
status: Draft
created: 2026-09-18
from_commit: b56a2a9
---

# Proposal: `impl` and Trait Implementations for Primitive Types

This proposal lets Kirby code write `impl` and `impl Trait for` blocks
targeting primitive types — `f64` (and, later, whatever numeric types join
it), `string`, and `bool` — the same way it already can for structs. Today
neither form is allowed on a primitive; both are rejected by the type
checker, and the restriction turns out to run deeper than the checker alone.

Following the project's convention: when this document says Kirby "does" or
"has" something, that is a checked fact about `main`, reproduced in Appendix
A. When it says Kirby "should" or "will" do something, that is what this
proposal asks for.

This is a **behavior-and-syntax-first** proposal. Part 1 establishes exactly
what stands in the way today — checked against the source, not guessed —
because the shape of that obstacle matters for reading Part 2 honestly. Part 2
is the actual ask: what programs should be legal and what they should do. Part
3 names, at a sketch level, the pieces an implementation would touch, and the
[Questions] section is where the real implementation decisions live, since
this document does not commit to them.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. Most of what is
still undecided is tracked as [Questions] rather than settled in the prose.

### Linking

[Links] in this document are defined as link references.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true
  for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is
  true after the changes presented here.

### Code & Changes

- C code from the language's implementation is shown in `c` blocks.
- Kirby code — including the desired syntax this proposal asks for, which
  does not compile on `main` — is shown in `kirby` blocks.

---

## Glossary

- **Primitive type:** one of Kirby's built-in, non-struct value types. Today
  that's `unit`, `bool`, `string`, and `f64`. This proposal is about `bool`,
  `string`, and `f64`; see [Q-unit] for why `unit` is treated separately, and the
  [Problem Statement] for why `Array` (also a "primitive" by the checker's own
  reckoning today) is out of scope entirely.
- **Primitive kind:** shorthand used throughout this document for "one of the
  `TypeKind` values a primitive uses" (`TYPE_BOOL`, `TYPE_STRING`, `TYPE_F64`).
  Used instead of naming `f64` specifically, since the design should not be
  tied to today's one numeric type — see [Q-numeric-family].
- **Inherent impl:** a plain `impl SomeType { ... }` block: methods attached
  directly to a type, not through a trait.
- **Trait impl:** an `impl SomeTrait for SomeType { ... }` block.
- **Receiver:** the value a method is called on — the `x` in `x.method()`.
- **Coherence:** the rule that a given (trait, type) pair may be implemented
  at most once. Kirby enforces this for structs today.
- **`Self`:** inside an impl block, a placeholder that stands for whatever
  type the block targets. Resolves today via `_currentImplTargetType`, a
  plain `Type *` slot in the checker's environment (see 1.6).
- **Method table:** the run-time table (a `Table` in Kirby's implementation)
  a struct's compiled methods are stored in, keyed by method name.
- **Class object:** the run-time `ObjStruct` a struct declaration produces,
  which owns that struct's method table. Every instance of the struct points
  back to the one class object; `impl` blocks attach methods to it.

---

## Problem Statement

Kirby has a working trait system — you can declare a trait, `impl` it for a
struct, and call the method (Appendix A.1). None of that extends to
primitives. Two different things are currently rejected, with two different
error messages, and the messages themselves suggest two different reasons:

```kirby
impl f64 {
    pub fun double(self): f64 = self * 2;
}
// Error: Only trait implementations are allowed on primitive types.
```

```kirby
impl Display for f64 {
    pub fun toString(self): string = "a number";
}
// Error: Primitive trait implementations aren't supported yet.
```

The first message reads as a deliberate rule ("only trait implementations are
allowed" — not "aren't supported yet"). The second reads as a TODO. That
difference is a real fact about the current source (Appendix A.2) and this
proposal does not paper over it — see [Q-inherent].

The practical gap this leaves: there is no way to give `f64`, `string`, or
`bool` a `Display`, `Eq`, `Ord`, or `Default` implementation, or any
domain-specific trait implementation, even though structs can have all four
today. Code that wants `x.equals(y)` or `x.toString()` to be callable on a
plain number or string the same way it's callable on a struct has no path to
get there.

**Explicitly out of scope:** `Array[T]` and other built-in container types.
`Array` is ephemeral today — it isn't backed by a Kirby struct, and its
future shape (once generics land) isn't settled. It happens to be caught by
the same `tokenIsPrimitiveTypeName` check as `bool`/`string`/`f64`
(Appendix A.2), but this proposal does not attempt to design impls for it.

**Also out of scope:** making Kirby's built-in operators (`+`, `==`, `<`,
etc.) *dispatch through* these implementations. That is a different, larger
question — already raised as an open option in the
[generic-types, macros, tooling proposal][generic-types], Part 4.4 and
10.3 — about whether operators stay a built-in fast path for primitives or
become uniform trait calls. This proposal is only about making `impl` and
`impl Trait for` legal on primitives and making the resulting methods
callable by name (`x.equals(y)`, `x.toString()`), independent of whether `==`
or `+` ever calls them. See [Q-sequencing] for how the two proposals relate.

---

## Part 1 — What exists today (checked baseline)

### 1.1 Traits work for structs

This builds and runs on `main`, printing `woof` (Appendix A.1):

```kirby
trait Greet { fun hello(self): string; }
struct Dog { pub var name: string; }
impl Greet for Dog {
    fun hello(self): string = "woof";
}
let d = Dog { name: "Rex" };
print d.hello();
```

### 1.2 Primitives are rejected at the type checker, for two different reasons

`src/typecheck.c` handles the two impl shapes in two different functions,
and each rejects primitives with its own wording:

```c
// typchkRegisterImplMethods -- plain `impl f64 { ... }`
if (tokenIsPrimitiveTypeName(&impl->targetName)) {
  typchkErrorAtTokenFmt(
      &impl->targetName,
      "Only trait implementations are allowed on primitive types.");
```

```c
// typchkRegisterTraitImpl -- `impl Trait for f64 { ... }`
if (tokenIsPrimitiveTypeName(&impl->targetName)) {
  typchkErrorAtTokenFmt(
      &impl->targetName,
      "Primitive trait implementations aren't supported yet.");
```

Both are reachable only because `typchkResolveImplTarget` (which resolves an
impl's target name to a struct, directly or through a type alias) returns
`NULL` for a primitive name — a primitive is simply not a struct, so it falls
into the "unknown/unsupported target" branch either way.
`tokenIsPrimitiveTypeName` recognizes exactly five names:

```c
static bool tokenIsPrimitiveTypeName(Token *token) {
  return tokenTextEquals(token, "unit") || tokenTextEquals(token, "bool") ||
         tokenTextEquals(token, "string") || tokenTextEquals(token, "f64") ||
         tokenTextEquals(token, "Array");
}
```

Checked test fixtures already exist for exactly this pair of rejections:
`tests/types/plain_impl_primitive_errors.krb` and
`tests/types/trait_impl_primitive_errors.krb` (Appendix A.2).

### 1.3 Primitives have nowhere to keep a method

`Type` (`src/types.h`) is a tagged union. Every piece of struct machinery —
fields, static methods, instance methods, trait methods, and the flat list of
implemented traits used for coherence — lives inside the `struct_` arm of
that union:

```c
struct Type {
  TypeKind kind;
  union {
    struct { // struct_
      InternedName name;
      TypeMember *fields; int fieldCount;
      TypeMember *staticMethods; int staticMethodCount; bool *staticMethodIsPublic;
      TypeMember *instanceMethods; int instanceMethodCount; bool *instanceMethodIsPublic;
      TypeMember *traitInstanceMethods; int traitInstanceMethodCount;
      TypeMember *traitStaticMethods; int traitStaticMethodCount;
      InternedName *implementedTraits; int implementedTraitCount;
      bool isGeneric;
      bool hasUnresolvedMembers;
    } struct_;
    // ... function, array, trait_ ...
  } as;
};
```

`bool`, `string`, and `f64` are singletons of kind `TYPE_BOOL`/`TYPE_STRING`/
`TYPE_F64` (`typeBool()`, `typeString()`, `typeF64()` in `src/types.c`), each
just `allocType(kind)` — a bare tagged union with nothing in `as` at all.
There is exactly one `Type *` per primitive kind, ever, so — unlike a struct,
where every declaration gets its own `Type` — whatever storage primitive
methods end up in only ever needs to hold *one* primitive kind's worth of
methods per kind, not "per declaration."

`typeStructImplementsTrait`, the function coherence and the `Eq`-for-`==`
check both call, is explicit about only working on structs:

```c
bool typeStructImplementsTrait(Type *type, InternedName traitName) {
  if (type == NULL || type->kind != TYPE_STRUCT)
    return false;
  ...
```

### 1.4 `impl` compiles by treating the target as a global variable

`compileImplDecl` (`src/compiler.c`) doesn't compile an impl block's target as
a type at all — it compiles it as an ordinary variable reference, resolves
that variable, and pushes whatever runtime value it holds onto the stack so
methods can be attached to it:

```c
static void compileImplDecl(AstNode *node) {
  ImplNode *impl = &node->as.impl;
  const Token *resolved = resolvedImplTargetsLookup(node);
  Token structName = resolved != NULL ? *resolved : impl->targetName;

  // Push the struct onto the stack so the methods can be bound to it
  VarRef ref = resolveVariable(&structName);
  emitBytes(ref.getOp, ref.arg);
  ...
  for (...) {
    compileFunction(method, type);
    emitBytes(OP_METHOD, constant);
  }
```

This works for a struct because declaring `struct Dog { ... }` compiles to
`OP_STRUCT` followed by `defineVariable`, which binds the name `Dog` to a
runtime class object (Part 1.5). `f64`, `string`, and `bool` are never bound
that way — they aren't declarations, they're names the checker recognizes by
text only inside type-annotation positions (`typchkResolveType`,
`src/typecheck.c`):

```c
if (tokenTextEquals(&t->name, "unit"))   return typeUnit();
if (tokenTextEquals(&t->name, "bool"))   return typeBool();
if (tokenTextEquals(&t->name, "string")) return typeString();
if (tokenTextEquals(&t->name, "f64"))    return typeF64();
```

Nothing registers `f64` (etc.) as a variable, so `resolveVariable(&structName)`
in `compileImplDecl` would have nothing to resolve to even if the checker let
it get that far. This is checked directly — the scanner doesn't even reserve
these as keywords, they're plain `TOKEN_IDENTIFIER`s the checker special-cases
by spelling, so trying to use one as an ordinary expression compiles but fails
at *run time*, not compile time:

```
$ echo 'print f64;' | ./build/krb -f /dev/stdin
Undefined variable 'f64'.
[line 1] in script
```

(exit code 70 — a runtime error, per the project's own convention — not 65,
a compiler error. Reproduced in Appendix A.3.)

### 1.5 The VM's method storage and dispatch assume a struct instance

Struct methods live in a `Table` on the runtime class object:

```c
typedef struct ObjStruct {
  Obj obj;
  ObjString *name;
  Table methods;
  Table fields;
  int fieldCount;
  bool fieldPublic[256];
} ObjStruct;

typedef struct {
  Obj obj;
  ObjStruct *struct_;   // every instance points back to its one class object
  Value *fields;
} ObjInstance;
```

`OP_METHOD` attaches a compiled method to whatever's on the stack, and
requires it to be a struct's class object:

```c
static bool defineMethod(ObjString *name) {
  Value target = peekStack(1);
  if (!IS_STRUCT(target)) {
    runtimeError(&vm, "Only structs can have implementations");
    return false;
  }
  ...
```

`invoke`, which runs every method call (`OP_INVOKE`), requires the receiver to
be either a struct's class object (a static-method call) or an instance:

```c
static bool invoke(ObjString *name, int argCount) {
  Value caller = peekStack(argCount);
  if (IS_STRUCT(caller)) return invokeStatic(AS_STRUCT(caller), name, argCount);
  if (!IS_INSTANCE(caller)) {
    runtimeError(&vm, "Only instances have methods.");
    return false;
  }
  ...
```

Numbers and bools are plain tagged values (`Value` in `src/value.h` — a
`ValueType` tag plus a `double`/`bool`/`Obj*` union) with **no** heap object
at all, so there is no existing place to hang a method table off of them.
Strings are heap objects (`ObjString`), but `ObjString` has no method-table
field either — it's `{ Obj obj; int length; char *chars; uint32_t hash; }`,
nothing more.

### 1.6 What already generalizes for free: `Self`

The one piece of impl-block machinery that is **not** struct-specific is
`Self`. `_currentImplTargetType` is stored as a plain `Type *`:

```c
void typchkTypeEnvSetImplTargetType(TypeEnv *env, Type *implTargetType) {
  env->_currentImplTargetType = implTargetType;
}
```

and `Self` resolves to whatever that slot holds:

```c
if (tokenTextEquals(&t->name, "Self")) {
  return env->_currentImplTargetType != NULL ? env->_currentImplTargetType
                                             : typeSelfPlaceholder();
}
```

Nothing here assumes `TYPE_STRUCT`. If the impl-target-resolution step
(1.2/1.4) is taught to accept `typeF64()` as a legitimate target, `Self`
inside `impl Eq for f64 { fun equals(self, other: Self): bool = ... }` should
resolve to `f64` without any change to this particular mechanism.

### 1.7 Property/method access on a value is gated to `TYPE_STRUCT`

`typchkInferGet`, which type-checks `x.name` expressions, falls through to:

```c
if (objectType->kind != TYPE_STRUCT) {
  typchkErrorAtTokenFmt(&get->name, "Can't access '.%.*s' on a %s.",
                        get->name.length, get->name.start,
                        typeToString(objectType));
  return NULL;
}
```

This is the check that would need to widen (or grow a parallel branch) for
`x.equals(y)` to type-check when `x: f64`.

### 1.8 `print` and `Display` are already independent — for structs, today

It's tempting to assume `impl Display for f64` would change how `print`
formats numbers. It wouldn't need to, and — importantly — that would not be a
*new* inconsistency. `print` compiles to `OP_PRINT`, which calls
`printValue` (`src/value.c`), which does built-in formatting
(`snprintf(buffer, size, "%f", ...)` for numbers, `objectToString` for heap
values) and never looks up or calls a `Display` implementation, for structs
or anything else. A struct that implements `Display` today has a callable
`.toString()` method; `print`ing that struct does not use it. This proposal
would extend the same, already-established split to primitives, not create
a new one. (The analogous, already-documented split for `Eq`/`==` is in the
[generic-types proposal][generic-types], Part 1.4.)

### 1.9 The built-in traits, for reference

Four traits are always in scope (`typchkTypeEnvDefineBuiltinTraits`,
`src/typecheck.c`), with these exact signatures:

| Trait | Method | Signature |
|---|---|---|
| `Display` | `toString` | `fun toString(self): string` |
| `Eq` | `equals` | `fun equals(self, other: Self): bool` |
| `Ord` (supertrait `Eq`) | `cmp` | `fun cmp(self, other: Self): f64` |
| `Default` | `default` (static) | `fun default(): Self` |

`Ord` requires `Eq` regardless of declaration order, and that check
(`typchkCheckTraitSupertraitSatisfied`) already resolves its target through
the same `typchkResolveImplTarget` this proposal would extend — it already
has a `NULL`-target early-out with the comment `// already reported, or a
(currently unsupported) primitive`, i.e. the supertrait check is already
written to be indifferent to *why* a target didn't resolve.

### 1.10 Operators do not check or call any trait, for primitives, at all

Unlike structs — where `==` at least *checks* for `Eq` even though it doesn't
call it (generic-types proposal, Part 1.4) — primitive operators don't
involve traits in either direction. `typchkInferBinary` hardcodes `f64` (and,
for `+`, `string`) directly:

```c
case TOKEN_EQUAL_EQUAL:
case TOKEN_BANG_EQUAL:
  if (!typesEqual(leftType, rightType)) { ... }
  if (leftType->kind == TYPE_STRUCT &&
      !typeStructImplementsTrait(leftType, internTokenName(makeTokenFromCString("Eq")))) {
    // requires Eq -- but only for TYPE_STRUCT
  }
  return typeBool();
```

So today, `impl Eq for f64` — even if it type-checked — would add a callable
`.equals()` method and nothing else; `1 == 1` neither requires it nor calls
it, exactly as `+`, `-`, `<`, etc. neither require nor call any trait for
`f64` operands. This is consistent with, and a slightly stronger version of,
the point in the [Problem Statement] about operator dispatch being a
separate, later question.

---

## Part 2 — Desired behavior and syntax

### 2.1 Trait impls on primitives

The core ask: this should be legal and should behave like the struct case in
every way that isn't specific to structs having fields or being nominal
types with multiple distinct declarations.

```kirby
impl Eq for f64 {
    fun equals(self, other: Self): bool = self == other;
}

impl Ord for f64 {
    fun cmp(self, other: Self): f64 =
        if self < other { -1 } else if self > other { 1 } else { 0 };
}

impl Display for string {
    fun toString(self): string = self;
}

impl Default for bool {
    fun default(): bool = false;
}

let a = 1;
let b = 2;
print a.cmp(b);        // -1
print a.equals(b);     // false -- and, per 1.10 and the Problem Statement,
                        // `a == b` is unaffected either way
```

Domain-specific traits work the same way — nothing about this is restricted
to the four built-ins:

```kirby
trait Squarable { fun squared(self): f64; }
impl Squarable for f64 {
    fun squared(self): f64 = self * self;
}
print 3.squared(); // 9 -- see 2.3 on call-site syntax for numeric literals
```

### 2.2 Coherence, exactly as it works for structs

A given trait may be implemented for a given primitive kind only once. A
second `impl Eq for f64` anywhere in the same program is a compile error,
with the same "already implements" wording structs get today. This falls out
of extending the existing coherence check (1.3/1.9) to primitive kinds rather
than designing a new rule.

Cross-module coherence (Rust's "orphan rule") is out of scope here exactly as
it is for the generic-types proposal (its Part 6.4) — Kirby has no module
system yet. Whatever rule that proposal settles on for "which module may
implement which trait for which type" should apply to primitive kinds the
same way it applies to struct kinds; this proposal doesn't need to say more
about it than that.

### 2.3 Calling convention

Instance methods and instance trait methods are called exactly like on a
struct — ordinary `.method(...)` syntax on any expression of that primitive
type:

```kirby
let s = "hello";
print s.toString();      // via `impl Display for string`

let x: f64 = 5;
print x.squared();       // via a user trait impl, as in 2.1
```

Static trait methods (only `Default.default()` among the built-ins, but any
user trait could declare one) are where primitives don't have an obvious
answer yet, because — per 1.4 — `f64`, `string`, and `bool` aren't
expressions today; `Dog.static_method()` works because `Dog` is a variable
holding a runtime value, and no such variable exists for a primitive kind.
This proposal takes no position on the receiver syntax for that case; see
[Q-static-receiver].

### 2.4 Which primitives, and room to grow

This should be specified in terms of **primitive kind**, not "`f64`
specifically," precisely because the user request driving this proposal is
explicit that `f64` today stands in for "the family of numeric types that
don't exist yet." A design that hardcodes `f64` would need redoing the day a
second numeric type (`i32`, `u8`, whatever) is added; a design that says "any
primitive kind may carry impls" does not. This proposal does not attempt to
design that numeric-type family — only to make sure this feature's design
doesn't foreclose it. See [Q-numeric-family].

`bool` and `string` are unambiguously in scope (they're primitive kinds
today, with no fields, exactly like `f64`). `unit` and `Array` are addressed
separately: see [Q-unit] for `unit`, and the [Problem Statement] for why `Array`
is excluded outright.

### 2.5 `Self` and supertraits

Both should work exactly as they do for structs, and per 1.6 and 1.9, both
mostly already do, mechanically, once a primitive kind is accepted as a valid
impl target at all:

```kirby
impl Eq for f64 { fun equals(self, other: Self): bool = self == other; }
impl Ord for f64 { fun cmp(self, other: Self): f64 = 0; }
// Error, same wording as for structs today:
// 'f64' also needs 'impl Eq for f64' -- 'Ord' requires it.
```

(That specific example already type-checks its *supertrait* logic correctly
against a `NULL`-resolved primitive target today, per 1.9 — it just can't get
past the initial primitive rejection to reach it.)

### 2.6 Visibility

Trait-impl methods on primitives should be implicitly public, exactly as
they are for structs — a non-`pub` method inside `impl Trait for f64` is
still callable from outside, matching the rule `hasTraitName` already
enforces for struct trait impls.

### 2.7 Inherent (non-trait) impls — the open part of the ask

The request this proposal is written against says primitive code should be
able to "`impl` or implement traits" — i.e., both forms. But 1.2 showed the
*existing* rejection for the bare form uses different wording ("only trait
implementations are allowed") than the trait form ("aren't supported yet"),
and that difference reads as intentional, not accidental — plausibly
mirroring the common design choice (Rust included) that only trait methods
may be added to a primitive type, so that a primitive's inherent surface
stays fixed and every extension to it is nameable, coherence-checked, and
(once orphan rules exist) attributable to one module. This document does not
resolve that tension; it's [Q-inherent], with a recommendation.

---

## Part 3 — Implementation shape (sketched, not committed)

Per the request driving this document, this proposal deliberately does not
commit to an implementation. What Part 1 establishes is that the restriction
is enforced independently at four layers, and each would need a
corresponding change if this feature is built:

1. **Type-level storage** (1.3): primitive kinds need *somewhere* to record
   their methods and implemented-trait list. Unlike a struct, there's exactly
   one `Type *` per primitive kind ever, which simplifies this — it's closer
   to "give three singletons some extra state" than "redesign how types
   carry methods." See [Q-storage].
2. **Impl-target resolution and compilation** (1.4): `typchkResolveImplTarget`
   needs to accept a primitive name as a target (straightforward — it's a
   `tokenIsPrimitiveTypeName` check already sitting right there, currently
   used only to pick an error message), and `compileImplDecl` needs a way to
   attach compiled methods to *something* representing "the `f64` kind" that
   doesn't depend on `resolveVariable` finding a global.
3. **Run-time storage and dispatch** (1.5): numbers and bools have no heap
   object to hang a method table on; strings have one but it has no
   method-table slot. `invoke`/`defineMethod` would need a path that doesn't
   require `IS_STRUCT`/`IS_INSTANCE`. See [Q-dispatch].
4. **The checker's property-access gate** (1.7): `typchkInferGet`'s
   `objectType->kind != TYPE_STRUCT` check needs to admit the primitive kinds
   this proposal covers.

None of these interact with monomorphization, bounds, or modules — this is
additive to the *existing* trait system, not dependent on the generic-types
proposal landing first. See [Q-sequencing].

---

## Questions

### **Q:** Should bare (non-trait) `impl` on primitives be allowed too, or should primitives stay trait-impl-only?

[Q-inherent]: #q-should-bare-non-trait-impl-on-primitives-be-allowed-too-or-should-primitives-stay-trait-impl-only

**Status:** Open

The request behind this document asks for both forms. The existing rejection
message for the bare form ("Only trait implementations are allowed on
primitive types") reads as an intentional restriction that predates this
proposal, not a "not built yet" placeholder — unlike the trait-impl
rejection, which is worded as exactly that.

**Options:**
(a) Lift only the trait-impl restriction; keep bare `impl f64 { ... }`
illegal, permanently, matching the existing wording and mirroring the common
rule that a primitive's inherent methods are closed to extension and every
addition must go through a named, coherence-checked trait. (b) Lift both
restrictions and allow bare inherent impls on primitives too.

**Why open:** (a) keeps every method added to `f64` attributable to a named
trait, which composes cleanly with coherence and (eventually) an orphan rule,
and matches what the current wording seems to already intend. (b) is closer
to the literal request and closer to how struct impls work (structs get both
forms today), but opens a namespacing question this document hasn't explored
— an inherent `impl f64 { fun double(self) ... }` from unrelated code could
silently collide with another inherent `impl f64` block's method name in a
way a *trait* impl's coherence check would have caught. Recommendation: (a),
unless a concrete use case surfaces that specifically needs an inherent
(non-trait) method on a primitive.

### **Q:** Where should a primitive kind's methods and implemented-trait list actually live?

[Q-storage]: #q-where-should-a-primitive-kinds-methods-and-implemented-trait-list-actually-live

**Status:** Open

**Options:** (a) give the three primitive `TypeKind`s their own union arm in
`Type`, shaped like a trimmed-down `struct_` (no `fields`, since primitives
have none) — since each is a singleton, this is a fixed, small amount of new
state, not "one struct_ per declaration"; (b) keep a small side table outside
`Type` entirely, keyed by `TypeKind`, that `typeStructImplementsTrait`-style
lookups consult when `type->kind != TYPE_STRUCT`; (c) something else — e.g.
generalizing `struct_`'s method-bearing fields into a shape shared by both
struct and primitive kinds, so existing lookup functions don't need a second
code path at all.

**Why open:** (a) is the most local change but adds a new union arm every
switch-on-`kind` site has to at least be aware won't be reached for it; (b)
touches `Type` not at all but means two different lookup mechanisms exist
side by side; (c) is the most invasive but the least likely to leave the
struct- and primitive- cases silently diverging later. This wasn't
investigated further because it's squarely an implementation decision, not a
behavior one.

### **Q:** How should the run-time attach and dispatch methods for kinds with no heap object?

[Q-dispatch]: #q-how-should-the-run-time-attach-and-dispatch-methods-for-kinds-with-no-heap-object

**Status:** Open

Per 1.5, `f64` and `bool` values carry no `Obj*` at all, and `ObjString` has
no method-table slot. `defineMethod` and `invoke` both hard-require
`IS_STRUCT`/`IS_INSTANCE`.

**Options:** (a) three (or, per [Q-numeric-family], eventually more) fixed global method
tables — one per primitive kind — that `OP_METHOD`/`invoke` consult directly
when the compile-time-known target or run-time value's `ValueType`/`ObjType`
says "primitive kind X," bypassing `IS_STRUCT` entirely; (b) give `ObjString`
a method-table pointer (solves strings) and find a separate answer for
unboxed numbers/bools, since they have no `Obj` to extend; (c) something that
unifies with whatever mechanism the generics proposal's operator-dispatch
work (Part 4.4) ends up needing anyway, since both need to ask "does this
value's kind have an implementation of trait X" at a call site.

**Why open:** this is the least-sketched-out corner of the whole proposal and
was deliberately left open rather than guessed at, since it's pure
implementation with no behavioral consequence either option would surface to
a Kirby programmer.

### **Q:** What is the receiver syntax for calling a static trait method on a primitive kind?

[Q-static-receiver]: #q-what-is-the-receiver-syntax-for-calling-a-static-trait-method-on-a-primitive-kind

**Status:** Open

Per 1.4 and 2.3, `f64`/`string`/`bool` are not expressions today — referencing
one as a bare identifier fails at run time with "Undefined variable," not at
parse or type-check time, because nothing ever binds them as variables the
way a struct declaration binds its name.

**Options:** (a) make primitive type names valid receiver expressions
specifically in call position (`f64.default()`), which means teaching the
parser/checker that `f64` used this one way means something, without making
it a general-purpose expression everywhere; (b) require some other explicit
syntax at the call site (a Kirby equivalent of `f64::default()` or similar);
(c) don't support static trait methods on primitives in a first version —
`Default` would simply remain unimplementable for primitives until this is
resolved, while instance methods (2.1–2.3) ship regardless.

**Why open:** (a) is the most natural reading of "the same trait system,"
but reaching it means deciding how far `f64` gets to act like a first-class
value versus staying purely a type-annotation token — a bigger decision than
this proposal's scope suggests it should make unilaterally. (c) is the
smallest change and doesn't block anything else in this document, since none
of the illustrative examples in Part 2 use a static method.

### **Q:** Should this land before, after, or independent of the generic-types proposal's operator-dispatch decision?

[Q-sequencing]: #q-should-this-land-before-after-or-independent-of-the-generic-types-proposals-operator-dispatch-decision

**Status:** Open

The [generic-types proposal][generic-types] already names "give primitives
real trait implementations" as one option (its Part 4.4, option (b), and Part
10.3) specifically so operators like `+` and `==` could dispatch to trait
methods uniformly across structs and primitives.

**Options:** (a) this proposal is independent and can land first — per 1.8
and 1.10, primitive impls have zero effect on operator behavior either way,
so shipping `.equals()`/`.toString()` as callable methods doesn't foreclose
or presuppose any operator-dispatch decision; (b) treat this proposal as
strictly subordinate to that decision and wait.

**Why open:** genuinely not much tension here — this is flagged as a
question mainly so the two documents stay in sync rather than because there's
a real disagreement. Recommendation: (a). If the generic-types proposal later
picks its option (b) (uniform operator dispatch, requiring primitive trait
impls), it should consume this proposal's design rather than re-deriving it.

### **Q:** Does `unit` get impls too?

[Q-unit]: #q-does-unit-get-impls-too

**Status:** Open

`unit` is swept into the same `tokenIsPrimitiveTypeName` check as `bool`,
`string`, `f64`, and `Array` (1.2), but the request behind this proposal
names only `f64`, `string`, and `bool`.

**Options:** (a) explicitly exclude `unit` — it has exactly one value, so a
trait method on it can only ever return a constant, and `Default for unit`
is the only built-in trait that would even type-check meaningfully; (b)
include it for uniformity's sake, since excluding it means one more special
case in every place this feature touches `tokenIsPrimitiveTypeName`.

**Why open:** low-stakes either way; listed so it's a deliberate choice
rather than an accident of whichever code path happens to touch
`tokenIsPrimitiveTypeName` first. Recommendation: (a), since there's no
motivating example for it.

### **Q:** Should the design anticipate more numeric primitive kinds now, or wait for that proposal?

[Q-numeric-family]: #q-should-the-design-anticipate-more-numeric-primitive-kinds-now-or-wait-for-that-proposal

**Status:** Open

The request driving this document explicitly frames `f64` as standing in for
"the family of numeric types that don't exist yet."

**Options:** (a) design and word this proposal (as attempted throughout) in
terms of "primitive kind" generally, so that adding `i32`/`u8`/etc. later is
"add a kind to the set this feature covers," not "redesign the feature"; (b)
scope this proposal to exactly `bool`, `string`, `f64` as they exist today
and revisit when a numeric-type-family proposal exists.

**Why open:** this document has tried to do (a) — every example and every
Question above is phrased in terms of "primitive kind" rather than hardcoding
`f64` — but no numeric-type-family proposal exists yet to check that framing
against, so it's recorded here as unverified rather than settled.

---

## Link References

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement
[generic-types]: ../generic-types/PROPOSAL.md

---

## Appendix A — Reproducing the baseline claims

Every "today" claim in Part 1 can be checked on a clean `main` build
(commit `b56a2a9`, `cmake -S . -B build && cmake --build build`, needs
`libreadline-dev`):

```shell
# traits work for structs
printf 'trait G { fun hello(self): string; }\nstruct D { pub var n: string; }\nimpl G for D { fun hello(self): string = "woof"; }\nlet d = D { n: "x" };\nprint d.hello();\n' > /tmp/a.krb
./build/krb -f /tmp/a.krb  # woof

# bare impl on a primitive: "only trait implementations are allowed"
printf 'impl f64 {\n    pub fun double(self): f64 = self * 2;\n}\n' > /tmp/b.krb
./build/krb -f /tmp/b.krb  # Error at 'f64': Only trait implementations are allowed on primitive types.

# trait impl on a primitive: "aren't supported yet"
printf 'impl Display for f64 {\n    pub fun toString(self): string = "a number";\n}\n' > /tmp/c.krb
./build/krb -f /tmp/c.krb  # Error at 'f64': Primitive trait implementations aren't supported yet.

# primitive type names aren't expressions -- fails at RUN time, not compile time
echo 'print f64;' > /tmp/d.krb
./build/krb -f /tmp/d.krb; echo "exit: $?"
# Undefined variable 'f64'.
# exit: 70   (a runtime error, per the project's own exit-code convention --
#             not 65, a compiler error)
```

The existing test suite already carries fixtures for the two primitive-impl
rejections: `tests/types/plain_impl_primitive_errors.krb` and
`tests/types/trait_impl_primitive_errors.krb`, plus their `.err`/`.exit`
snapshots.
