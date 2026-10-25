---
status: Draft
category: epic
created: 2026-10-25
from_commit: b41332d
---

# Proposal: Type And Definition Information For Tools

The type checker works out the type of every expression and which declaration
every name refers to. It uses those facts to find errors and then throws them
away. An editor needs exactly those facts: hover shows a type, go to definition
follows a name to its declaration, and find references lists every name that
points at one declaration.

This proposal has the checker write those facts into tables that a caller can
ask for. The tables are kept next to the AST, not inside it, and they record
places as [spans][Span] from the [Tooling Data Proposal], not as pointers to
nodes. A caller that does not ask for the tables gets exactly what it gets
today. The first reader is the [Language Server Proposal].

The title and the folder name are working titles. The choices that are still
open are listed as [Questions].

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

- When the proposal text says Kirby "does" or "has" something, that is true for
  the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is
  true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code
  blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

### What Kirby has today

Checked at `from_commit` (see [Appendix A]):

- **Types are worked out and returned, never stored.** Every expression's type
  comes back from one function, `typchkInfer` in `src/typecheck.c`, as a
  `Type *`. The caller compares it with what it expected and drops it. An
  `AstNode` (`src/ast.h`) has a `kind`, a `line`, and its payload. It has no
  place for a type.
- **Five kinds of expression are typed without `typchkInfer`.**
  `typchkCheckInto` handles a lambda, an array literal with an expected type, a
  generic function used as a value, and a generic struct initializer with an
  expected type itself, and only falls through to `typchkInfer` for the rest:

  ```c
  if (node->kind == NODE_FUNCTION && node->as.function.isLambda)
    return typchkCheckOrInferLambda(env, node, expected) != NULL;

  if (node->kind == NODE_ARRAY && expected != NULL) {
  ```

  The fifth is the callee of a call by name, such as the `id` in `id(1)`.
  `typchkInferCall` looks the name up itself:

  ```c
  if (c->callee->kind == NODE_VARIABLE) {
    Token *name = &c->callee->as.variable.name;
    Type *calleeType = typchkTypeEnvLookup(env, *name);
  ```

- **`@print` and `@println` have no type.** They are not in the
  `nativeSignatures` table (`src/native_signatures.c`), because they take any
  value. `typchkInferCall` finds nothing for them, checks the arguments, and
  returns `NULL`, so a call to them has no type. `let v = @println(1);` compiles,
  and `v` has no type either.

- **A name is looked up by its text, and the binding forgets where it was
  declared.** The checker's environment holds names and types only:

  ```c
  typedef struct {
    InternedName name;
    Type *type;
  } TypeEnvBinding;
  ```

  Struct fields and methods are kept the same way, as a `TypeMember` with a name
  and a type (`src/types.h`). Nothing records which declaration a use of a name
  found.

- **The checker's memory outlives each AST, not the other way round.** `krb run`
  starts one checker session (`typchkSessionBegin`), compiles the stdlib, then
  compiles the program, each with `compileSource` in `src/main.c`. Every
  `compileSource` ends with `astFreeAll()`. The environment and every `Type`
  stay until `typchkSessionEnd()` frees them with `typesFreeAll()`. So a `Type *`
  is safe to keep for the whole session, and a pointer to a stdlib `AstNode` is
  not safe once the program starts compiling.
- **A generic body is checked once.** `fun id[T](x: T): T` is checked one time
  with `T` as an unknown type, however many times it is called ([Generic Types
  Proposal] 3.3, and [Appendix A]). Monomorphization, when it lands, runs after
  checking ([Generic Types Proposal] 5.3), so it does not change what the checker
  sees.
- **The checker keeps going after an error.** A file with three type errors in
  two functions reports all three. Any parse error stops the checker from
  running at all, because `compileSource` only calls `typchkCheckProgram` when
  parsing had no error.
- **A type can already be printed.** `typeToString` (`src/types.c`) returns the
  type as Kirby writes it, such as `Box[i64]` or `fun (i64) => Date`. Error
  messages use it.
- **Positions are lines only.** That is the [Tooling Data Proposal]'s subject.
  Its Parts 1 and 2 give every token a column and every node a span. Its first
  version leaves out "a symbol index for go to definition and find references",
  which is what this proposal adds.

### What is missing

1. **No record of types.** A tool cannot ask "what is the type of the
   expression at line 4, column 9?"
2. **No record of definitions.** A tool cannot ask "which declaration does this
   `x` refer to?" or "where is everything that refers to this `let x`?"
3. **No safe way to keep either across units.** A pointer into the stdlib's AST
   is freed before the program is checked, and the stdlib is where `Date` lives.

### How other languages do it

- **Go** (`go/types`) leaves the AST alone. The caller passes an `Info` struct
  to the checker. `Types` "maps expressions to their types", `Defs` "maps
  identifiers to the objects they define" and `Uses` "maps identifiers to the
  objects they denote". `Check` "populates each of the non-nil maps", so a
  caller that asks for nothing pays nothing. In `Types`, "Invalid expressions
  are omitted" ([go/types api.go], [go/types package]).
- **TypeScript** answers on request: `getTypeAtLocation` and
  `getSymbolAtLocation` take a node from a program's checker
  ([TypeScript checker API]).
- **rust-analyzer** chose not to store meaning in the tree: "syntax tree is a
  value type", and it "doesn't store semantic info" because "using the tree as a
  store for semantic info is convenient in traditional compilers, but doesn't
  work nicely in the IDE" ([rust-analyzer architecture]).

Kirby's checker is closest to Go's. It checks a whole program in one call and
then stops, and its data is plain C structs. This proposal follows Go.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- A table of expression types: for each expression the checker accepts, its
  span and its type.
- A table of declarations: for each declared name, its span, what kind of
  thing it declares, and its type.
- A table of uses: for each name that refers to a declaration, its span and
  which declaration it found.
- Tables that cover every unit compiled in one checker session, so a use in the
  program can point at a declaration in the stdlib.
- No change for any caller that does not ask: no extra memory, no change in
  errors, no snapshot updates.
- A `krb` command that prints the tables, so they can be tested with the
  existing snapshot tests.

The following is intentionally left out of scope for this proposal:

- **The language server itself.** That is the [Language Server Proposal].
- **Completion.** Completion after `p.` needs the type of `p` while the code
  around it does not parse, and a parse error stops the checker today.
- **Types in the compiled output.** `CompiledUnit` stores no types, and that
  stays with the [Modules Proposal]'s interface.
- **The compiler reading the tables.** `src/compiler.c` does not include
  `types.h` and works out nothing from types today. Nothing here changes that.
- **Files other than the stdlib and one program.** Declarations in other files
  wait for the [Modules Proposal] ([Q-units]).
- **Macros.** A span in generated syntax carries an origin ([Tooling Data
  Proposal] Part 3). A tool reading these tables follows it. Nothing here needs
  macros to exist.

### Implementation Plan

#### Part 1: The tables

<!-- [Part 1]: #part-1-the-tables -->

A new header, `src/type_info.h`:

```c
typedef enum {
  DEF_VARIABLE,   // let and var
  DEF_PARAMETER,
  DEF_FUNCTION,
  DEF_STRUCT,
  DEF_FIELD,
  DEF_METHOD,
  DEF_TRAIT,
  DEF_TYPE_ALIAS,
  DEF_GENERIC_PARAM,
  DEF_NATIVE,     // An @name function. It has no span.
  DEF_BUILTIN,    // A builtin type or trait. It has no span.
} DefKind;

typedef struct {
  int unit;       // Which unit of the session, in the order compiled
  Span span;      // Tooling Data Proposal, Part 1
} InfoPlace;

typedef struct {
  InfoPlace place; // The whole expression
  Type *type;
} InfoExpr;

typedef struct {
  InfoPlace place; // The declared name only, such as the `x` in `let x = 1;`
  bool hasPlace;   // False for DEF_NATIVE and DEF_BUILTIN
  DefKind kind;
  InternedName name;
  Type *type;
  int owner;       // For a field or method: its struct or trait. Else -1
} InfoDef;

typedef struct {
  InfoPlace place; // The name as written at the use
  int def;         // Index into defs
} InfoUse;

typedef struct {
  InfoExpr *exprs;
  int exprCount;
  int exprCapacity;

  InfoDef *defs;
  int defCount;
  int defCapacity;

  InfoUse *uses;
  int useCount;
  int useCapacity;
} TypeInfo;

void typeInfoInit(TypeInfo *info);
void typeInfoFree(TypeInfo *info);
```

Some rules that follow from what the checker does today:

- **A place is a unit and a span.** A `fileId` belongs to its unit ([Tooling
  Data Proposal] Part 5), and both the stdlib and the program have a file 0.
  `unit` says which of them is meant: 0 for the stdlib, 1 for the program, and
  counting up for each REPL line. See [Q-units].
- **A use points at a declaration by index.** The declarations of every unit in
  the session are in one table, so a use in the program can point at the
  stdlib's `Date` without a pointer into a freed AST.
- **An expression with no type has no entry.** When `typchkInfer` returns
  `NULL` nothing is recorded, as Go leaves out invalid expressions. That covers
  an expression with an error, and a call to `@print` or `@println`. A
  declaration whose type is not known still gets an `InfoDef`, with a `NULL`
  type.
- **A generic body records its own parameters.** Inside
  `fun id[T](x: T): T { x }` the `x` has type `T`. At the call `id(1)` the call
  expression has type `i64`, and the callee `id` has the signature the call
  filled in, `fun (i64) => i64`. The checker already writes that type when a
  generic function is used where a function type is expected:
  `let c: fun (i64) => bool = id;` reports
  `Expected fun (i64) => bool, got fun (i64) => i64.`
- **A member of a generic instance points at the declaration.** For
  `Box[i64]`, a use of `value` points at the field declared in `struct Box[T]`.
  This is how the checker already finds members: "An instance holds no members
  of its own" (`src/types.h`).
- **Types are borrowed.** `InfoExpr.type` and `InfoDef.type` point into the
  checker's memory, and are valid until `typchkSessionEnd()`. A reader that wants
  a type later keeps its `typeToString` text, which is also in that memory, or
  copies it.

**Cost.** Only when asked. Worked out, not measured: an `InfoExpr` is about 40
bytes (a 28 byte span, a unit, and a pointer, rounded). The tests average one
node for every 8 bytes of source ([Tooling Data Proposal] Appendix B), and at
most every node is an expression, so a 100 KB program would use at most about
500 KB for this table while the session lasts. A caller that passes no
`TypeInfo` uses none.

#### Part 2: Asking the checker for them

<!-- [Part 2]: #part-2-asking-the-checker-for-them -->

`typchkCheckProgram` keeps its form. A second function takes the tables:

```diff
 bool typchkCheckProgram(AstNode **program, int count);
+// Like typchkCheckProgram. Also records types, declarations and uses in
+// `info`, which must stay alive until typchkSessionEnd().
+bool typchkCheckProgramWithInfo(AstNode **program, int count, TypeInfo *info);
```

`typchkCheckProgram` calls it with `NULL`. The `TypeInfo *` sits on the
`TypeEnv` while checking, and every recording helper does nothing when it is
`NULL`. The one call in `src/main.c` and the four in `unit/` do not change.

The checker counts units itself: each call in a session is the next unit.
`typchkSessionBegin` sets the count to 0.

When the [Embedded Library Proposal]'s `FrontEnd` bundle lands (its Part 2),
the unit count and the `TypeInfo *` are two more of its fields.

#### Part 3: Recording expression types

<!-- [Part 3]: #part-3-recording-expression-types -->

`typchkInfer` becomes a wrapper. The current switch moves into a function of its
own, and the wrapper records what comes back:

```diff
-Type *typchkInfer(TypeEnv *env, AstNode *node) {
+static Type *typchkInferInner(TypeEnv *env, AstNode *node);
+
+Type *typchkInfer(TypeEnv *env, AstNode *node) {
+  Type *type = typchkInferInner(env, node);
+  typchkRecordExpr(env, node, type);
+  return type;
+}
+
+static Type *typchkInferInner(TypeEnv *env, AstNode *node) {
   switch (node->kind) {
```

The four paths in `typchkCheckInto` that do not reach `typchkInfer` record the
type they checked against: the lambda's type from `typchkCheckOrInferLambda`,
and the expected type for the array literal, the generic function value, and
the generic struct initializer, when they succeed. `typchkInferCall` records
the callee of a call by name: the type it looked up, or for a generic function
the signature with the call's type arguments filled in.

A unit test makes sure nothing is missed. It checks every program under
`tests/run/` that has no error and walks its AST. Every expression node must
have exactly one entry with its span, apart from a call to `@print` or
`@println` and any expression whose value comes from one ([Part 6]).

#### Part 4: Recording declarations and uses

<!-- [Part 4]: #part-4-recording-declarations-and-uses -->

**Declarations.** Every place that adds a name to the environment adds an
`InfoDef` first, and keeps its index:

```diff
 typedef struct {
   InternedName name;
   Type *type;
+  int def; // Index into TypeInfo.defs, or -1 when no TypeInfo was asked for
 } TypeEnvBinding;
```

`TypeMember` gets the same field, so a struct's fields and methods, and a
trait's methods, remember their declarations. The struct and trait parts of a
`Type` get one too, for the struct or trait itself. A generic parameter's
binding (`_genericParams`) is a `TypeEnvBinding` already.

The natives in the `nativeSignatures` table are declared through
`typchkTypeEnvRegisterFunction` (`src/native_signatures.c`). They get
`DEF_NATIVE` declarations with no place. `@print` and `@println` are not in the
table, so they have none, and a use of them records nothing. Builtin traits and
types get `DEF_BUILTIN`.

**Uses.** Every place that finds a name records a use with the declaration it
found:

| Use                                | Found by                                     |
| ---------------------------------- | -------------------------------------------- |
| A variable, parameter, or function | `typchkLookupVariableType`                   |
| A function called by name          | `typchkTypeEnvLookup` and `typchkTypeEnvLookupFunction` in `typchkInferCall` |
| A struct name in an initializer    | `typchkStructInitDeclaration`                |
| A field or method after `.`        | the member lookup in `typchkInferGet` and friends |
| A type name in an annotation       | `typchkResolveType` and the `typchkTypeEnvLookup*` functions |
| A trait in `impl Trait for X`      | `typchkTypeEnvLookupTrait`                   |

A declaration is not also recorded as a use of itself. A name that is not found
records nothing, because the error already says so.

#### Part 5: Printing the tables

<!-- [Part 5]: #part-5-printing-the-tables -->

`krb` has `lex` and `parse` to print the tokens and the AST. A third command
prints the tables for one file, after the stdlib ([Q-command]):

```kirby
fun id[T](x: T): T { x }

let n = id(1);
@println(n);
```

```
def 1:5-1:7 function id: fun (T) => T
def 1:8-1:9 generic-param T: T
def 1:11-1:12 parameter x: T
use 1:14-1:15 T -> 1:8
use 1:18-1:19 T -> 1:8
expr 1:22-1:23 T
use 1:22-1:23 x -> 1:11
def 3:5-3:6 variable n: i64
expr 3:9-3:14 i64
expr 3:9-3:11 fun (i64) => i64
use 3:9-3:11 id -> 1:5
expr 3:12-3:13 i64
expr 4:10-4:11 i64
use 4:10-4:11 n -> 3:5
```

Line 4 has entries only for `n`, because `@println` has no type and no
declaration ([Part 4]). The exact text is [Q-command]. Positions are
line:column, start and end, as spans store them. A use of a stdlib declaration
says `stdlib` and the position.
Only the program's own entries are printed. Entries are sorted by where they
start, with a longer range before a shorter one that starts at the same place,
so the order does not depend on the order the checker visits things in.

#### Part 6: Testing

<!-- [Part 6]: #part-6-testing -->

The same TDD loop as the rest of the repository: one failing test, then the
change.

- **Snapshot tests** of the command in [Part 5], in a folder of their own
  (`tests/types/`), one per kind of declaration and use ([Testing Plan]).
- **Every expression has an entry.** The unit test described in [Part 3].
- **Every use points at a declaration with the same name.** The same unit test
  checks it for every use.
- **Nothing changes without the tables.** `just test` passes with no snapshot
  updates at every step.

#### Part 7: Suggested order

<!-- [Part 7]: #part-7-suggested-order -->

1. The `TypeInfo` type and `typchkCheckProgramWithInfo`, recording nothing.
2. Expression types ([Part 3]), and the print command with only `expr` lines.
3. Local variables and parameters as declarations and uses.
4. Functions, structs, fields, and methods.
5. Traits, type aliases, generic parameters, natives, and builtins.

Steps 2 to 5 need spans, so they come after the [Tooling Data Proposal]'s Parts
1 and 2. Step 1 does not.

## Impacts

### Existing Syntax Or Behavior

- **No language change.** No syntax, errors, opcodes, or output change.
- **`typchkInfer`** becomes a wrapper around the current switch.
- **`TypeEnvBinding` and `TypeMember`** gain a declaration index. The struct and
  trait parts of `Type` gain one too. Each of the first two is 16 bytes today
  (an 8 byte `InternedName` and a pointer), and padding makes it 24. Worked out,
  not measured.
- **`krb`** gains one command ([Q-command]).
- **Snapshots.** None change.

### Related Proposals

- [Tooling Data Proposal] — depends on it. Every place in the tables is a span
  from its Parts 1 and 2. Its first version leaves out "a symbol index for go to
  definition and find references", and this proposal is that index. A file id
  only means something inside its unit, so the tables add which unit
  ([Q-units]), which interacts with [Q-files] there. That proposal is updated to
  point here.
- [Language Server Proposal] — the first reader. Hover reads `exprs` and
  `defs`, go to definition and find references read `uses`.
- [Generic Types Proposal] — a generic body is checked once with its parameters
  unknown (3.3), so its entries show `T`. Monomorphization runs after checking
  (5.3) and does not add entries. That proposal's Part 8 is updated to point
  here.
- [Trait Types Proposal] — its Part 4 asks for hover and completion data that
  list a trait's methods. Hover is this proposal. Completion is left out here.
  That proposal is updated to point here.
- [Modules Proposal] — a declaration in another module has no unit in the
  session until modules decide how files are named ([Q-naming] there). The
  module interface would need a span for each public declaration for go to
  definition to reach into a module. That proposal is updated to say so.
- [Embedded Library Proposal] — its `FrontEnd` bundle (Part 2) is where the
  unit count and the `TypeInfo *` would live. That proposal is updated to say so.
- [Macros Proposal] — a span in generated syntax carries an origin. The tables
  store spans as they are, and tools follow the origin.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime error cases.

A test is an `.argv` file: the arguments for `krb`, one per line. The program it
runs is the file of the same name without `.argv`, written `$file` in the
arguments. Language tests go in `tests/run/` and run with `run` and `$file`;
tests of other commands have their own folders, such as `tests/exec/`. A
program without an `.argv` file is not a test. See `tests/README.md` in
kirbylang.

The command name below is the working name from [Q-command].

##### NEW: tests/types/local_variable.krb

```kirby
let a = 1;
let b = a + 2;
```

###### Expected Outcome

Two `def` lines, for `a` and `b`, both `i64`. A `use` of `a` on line 2 that
points at line 1, column 5. `expr` lines for `1`, `a`, `2`, and `a + 2`.

##### NEW: tests/types/struct_field_and_method.krb

```kirby
struct Point {
  pub var x: i64;
}

impl Point {
  pub fun getX(self): i64 = self.x;
}

let p = Point { x: 1 };
@println(p.getX());
```

###### Expected Outcome

`def` lines for `Point`, its field `x`, and its method `getX`. On line 9, `Point`
is a use of the struct and `x` is a use of the field. On line 10, `getX` is a
use of the method. On line 6, `x` after `self.` is a use of the field.

##### NEW: tests/types/generic_function.krb

The program from [Part 5].

###### Expected Outcome

The output in [Part 5].

##### NEW: tests/types/generic_struct_member.krb

```kirby
struct Box[T] {
  pub var value: T;
}

let b: Box[i64] = Box { value: 1 };
@println(b.value);
```

###### Expected Outcome

`Box[i64]` on line 5 is a use of the struct. The `value` on line 6 is a use of
the field declared on line 2, and its `expr` line says `i64`.

##### NEW: tests/types/stdlib_struct.krb

```kirby
let d: Date = Date.new(0);
@println(d.get_year());
```

###### Expected Outcome

Both `Date`s are uses that point at `stdlib` and the position of `struct Date`
in `stdlib/stdlib.krb`. `new` and `get_year` are uses of the methods declared
there. No `def` line is printed for any of them, because they are not in the
program.

##### NEW: tests/types/type_error_has_no_entry.krb

```kirby
let a = 1 + "x";
let b = 2;
```

###### Expected Outcome

The error `'+' needs two numbers of the same type or two strings, got i64 and
string.` is printed as it is today, and the exit code is 65. `1` and `"x"` have
`expr` lines. `1 + "x"` has none. `a` has a `def` line with no type. The `def`
and `expr` lines for line 2 are there, because the checker keeps going after an
error.

## Questions

### **Q:** Are the tables separate from the AST, or fields on each node?

<!-- [Q-shape]: #q-are-the-tables-separate-from-the-ast-or-fields-on-each-node -->

**Status:** Open

- **(a) Tables next to the AST** (proposed). Places are spans and indexes. They
  survive `astFreeAll()`, cover every unit in a session, and cost nothing when
  no one asks.
- **(b) Fields on `AstNode`**: a `Type *type` and a `AstNode *decl`. The
  simplest to write, and a reader gets the type straight from the node. A node
  grows by 16 bytes for every caller. A `decl` that points into the stdlib is
  freed before the program is checked, so a use of `Date` would hold a bad
  pointer. It also makes nodes hold meaning, which rust-analyzer chose against.
- **(c) Answer on request, as TypeScript does.** The checker would need its
  scopes again after it finished, and Kirby's checker pops a scope when it
  leaves one. It would need a second way to walk the program.

### **Q:** How does a place say which unit it is in?

<!-- [Q-units]: #q-how-does-a-place-say-which-unit-it-is-in -->

**Status:** Open

A span's `fileId` is an index into its unit's own list of files ([Tooling Data
Proposal] Part 5), so file 0 of the stdlib and file 0 of the program are not the
same file.

- **(a) The order the unit was checked in the session** (proposed). 0 is the
  stdlib, 1 is the program, and each REPL line is the next one. It needs nothing
  from other proposals. A reader that wants a path asks the session which source
  name each unit had.
- **(b) A file id unique in the whole session.** This is option (b) of [Q-files]
  in the [Tooling Data Proposal]. If that is chosen, `unit` goes away and the
  span alone is enough.
- **(c) The source name on each place.** An interned string, so 8 bytes each.
  The same answer [Q-files] gives for (c): it saves nothing once ids exist.

(a) can change to (b) later without changing any reader except the one that
turns a place into a path.

### **Q:** What is the print command called, and what does it print?

<!-- [Q-command]: #q-what-is-the-print-command-called-and-what-does-it-print -->

**Status:** Open

`krb lex FILE` and `krb parse FILE` exist. Options for the name: `krb types
FILE`, `krb check --info FILE`, or `krb parse --types FILE`. The output in
[Part 5] is one line per entry, sorted by position. Should it print the stdlib's
entries too, behind an option?

### **Q:** Does the checker keep going past a parse error?

<!-- [Q-partial]: #q-does-the-checker-keep-going-past-a-parse-error -->

**Status:** Open

Today any parse error means there are no tables at all. While someone types,
the code is often not valid for a moment, and an editor that loses every type
for that moment is less useful.

- **(a) No** (proposed for the first version). A reader keeps the tables from
  the last check that worked. The [Language Server Proposal] asks the same
  question from the other side ([Q-parse-errors] there).
- **(b) Check the statements that parsed.** The parser already recovers at the
  next statement (`synchronize` in `src/parser.c`). The checker could run on the
  rest. Errors that only exist because a declaration is missing would need to be
  kept out, or a user sees errors that are not real.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **AST**: The tree the parser builds from source text (`AstNode` in
  `src/ast.h`).
- **Span**: Where a piece of syntax is in the source, as defined by the
  [Tooling Data Proposal].
- **Declaration**: The place where a name is introduced, such as `let x`,
  `fun f`, a parameter, or a struct field.
- **Use**: A place where a name refers to a declaration.
- **Unit**: What one call to `compileSource` compiles: the stdlib, the program,
  or one REPL line.
- **Session**: Everything between `typchkSessionBegin()` and
  `typchkSessionEnd()`. Its units share one checker environment.
- **Hover**: An editor feature that shows information, such as a type, when the
  pointer rests on some code.
- **Go to definition**: An editor feature that moves to the declaration of the
  name under the cursor.
- **Find references**: An editor feature that lists every use of a declaration.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals
[Testing Plan]: #testing-plan
[Appendix A]: #appendix-a--reproducing-the-baseline-claims
[Part 1]: #part-1-the-tables
[Part 2]: #part-2-asking-the-checker-for-them
[Part 3]: #part-3-recording-expression-types
[Part 4]: #part-4-recording-declarations-and-uses
[Part 5]: #part-5-printing-the-tables
[Part 6]: #part-6-testing
[Part 7]: #part-7-suggested-order

<!-- Questions -->

[Q-shape]: #q-are-the-tables-separate-from-the-ast-or-fields-on-each-node
[Q-units]: #q-how-does-a-place-say-which-unit-it-is-in
[Q-command]: #q-what-is-the-print-command-called-and-what-does-it-print
[Q-partial]: #q-does-the-checker-keep-going-past-a-parse-error

<!-- Related proposals -->

[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Span]: ../tooling-support-data/PROPOSAL.md#part-1--the-span
[Language Server Proposal]: ../language-server/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-files]: ../tooling-support-data/PROPOSAL.md#q-how-are-multiple-source-files-identified
[Q-naming]: ../modules/PROPOSAL.md#q-is-a-module-named-by-its-file-path-or-by-a-namespace
[Q-parse-errors]: ../language-server/PROPOSAL.md#q-what-does-the-server-show-while-the-file-does-not-parse

<!-- Outside references -->

[go/types api.go]: https://go.googlesource.com/go/+/refs/tags/go1.5.1/src/go/types/api.go
[go/types package]: https://pkg.go.dev/go/types@go1.26.6
[TypeScript checker API]: https://tessl.io/registry/tessl/npm-typescript/5.9.0/files/docs/type-checker.md
[rust-analyzer architecture]: https://git.peppe.rs/languages/rust-analyzer/plain/docs/dev/architecture.md

## Appendix A — Reproducing the baseline claims

On a clean build at `from_commit` (`just build`):

**Types are not stored on nodes.**

```shell
sed -n '/^struct AstNode {/,/^};/p' src/ast.h   # kind, line, and the union
grep -n "^Type \*typchkInfer(" src/typecheck.c  # the one place types come from
```

**Five kinds of expression skip `typchkInfer`.** Read `typchkCheckInto` in
`src/typecheck.c`. The lambda, array, generic function value, and generic struct
initializer cases return before the `typchkInfer` call at its end. In
`typchkInferCall`, a callee that is a name is looked up with
`typchkTypeEnvLookup`.

**`@println` has no type.**

```shell
grep -c '"@println"' src/native_signatures.c       # 0
./build/krb exec 'let v = @println(1); let w: bool = v;'   # prints 1, exit 0
```

**The checker's spelling of types.**

```shell
./build/krb exec 'fun id[T](x: T): T { x } let c: fun (i64) => bool = id;'
# Expected fun (i64) => bool, got fun (i64) => i64.
./build/krb exec 'fun id[T](x: T): T { let q: bool = x; x }'
# Expected bool, got T.
```

**The compiler does not use types.**

```shell
grep -n '#include "types.h"' src/compiler.c    # nothing
```

**A generic body is checked once, and the checker keeps going.**

```shell
cat > /tmp/g.krb <<'EOF'
fun id[T](x: T): T {
  let y: i64 = "s";
  x
}
@println(id(1));
@println(id("a"));
@println(id(true));
EOF
./build/krb run /tmp/g.krb   # one error, on line 2. Exit 65.

cat > /tmp/t.krb <<'EOF'
fun a(): i64 {
  let x: i64 = "one";
  let y: string = 2;
  x
}

fun b(): string {
  let z: bool = 3;
  "b"
}
EOF
./build/krb run /tmp/t.krb   # three errors: lines 2, 3 and 8
```

**A parse error stops the checker.**

```shell
printf 'let x: i64 = "one";\nlet y = ;\nlet z: bool = 3;\n' > /tmp/p.krb
./build/krb run /tmp/p.krb   # only "[line 2] Error at ';': Expect expression."
```

**The session outlives each AST.** Read `sessionBegin` and `compileSource` in
`src/main.c`: one `typchkSessionBegin()`, then `compileSource` for the stdlib and
for the program, each ending in `astFreeAll()`, then `typchkSessionEnd()` in
`sessionEnd`.
