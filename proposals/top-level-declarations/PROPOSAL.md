---
status: Draft
category: epic
created: 2026-09-20
from_commit: 2f96a2e
---

# Proposal: Top-Level Declarations Only

This proposal changes what a `.krb` file may contain at its top level. A file
would hold only _declarations_: functions, structs, impls, traits, type aliases,
and `let` / `var` bindings whose values are known at compile time. Loading a
file would do nothing else. To run a file, it needs a `fun main(): unit`, which
Kirby calls after the file has loaded.

The point is that loading a file, for example importing it as a module, never
has side effects.

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
  the current implementation. Each such claim was checked against a build at
  `from_commit`, and [Appendix A] shows how to check it again.
- When the proposal text says Kirby "should" or "will" do something, that is
  true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code
  blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Today, loading a file means running it. The parser accepts any statement at the
top level, and the compiler turns the whole file into one function (shown as
`<script>` in bytecode listings) that runs from top to bottom. Checked at
`from_commit` ([Appendix A]):

- A file can hold calls (including `@println`), `if`, `while`, `for`, blocks
  and assignments at its top level. Two top-level rules exist already: `return` is refused, and
  `struct`, `impl`, `trait` and `type` are refused _inside_ blocks.
- Every top-level `let` and `var` runs its initializer when the file loads.
  That includes calls to natives such as `@prompt`, `@argv` and `@clock`, and
  calls to functions the file defines.
- `krb run` loads the stdlib by running it through `runCode` (`sessionBegin` in
  `src/main.c`), the function `krb exec` uses for its code. The stdlib is built
  into the binary as the text `KIRBY_STDLIB`, and is not read from
  `stdlib/stdlib.krb` at run time. That is the only "import" Kirby has today.
  It holds only declarations: the `Date` struct and its impls.
- The REPL (`krb repl`) and `krb exec` compile their code through the same path.
- There is no entry point. The file is the program.

While a program is a single file, none of that hurts. It hurts as soon as a
file is loaded for a reason other than running it. This file prints and reads
the clock when it is loaded:

```kirby
var start = @clock();
@println("counter loaded");

fun next(n: i64): i64 = n + 1;
```

There is no way to load this file just to get `next`. Three other proposals run
into the same wall:

- [Modules Proposal] — importing a module would run its top-level code. Someone
  importing it cannot tell what will happen, or in what order, without reading
  it.
- [Testing Proposal] — test code cannot sit beside the code it tests, because
  loading the file runs everything, and running everything is what registers
  the tests.
- [Projects Proposal] — lists "requiring a main function" as out of scope until
  top-level items are restricted. This proposal is that restriction.

## Proposed Changes

After this proposal, the top level of a file is a list of declarations, and a
program starts at `main`. This is `examples/fizzbuzz.krb` before and after:

```diff
 #!/usr/bin/env krb run

 fun fizzbuzz(n: i64): string =
     if (n % 5 == 0 and n % 3 == 0) "FizzBuzz"
     else if (n % 3 == 0) "Fizz"
     else if (n % 5 == 0) "Buzz"
     else @numberToString(n);

-var limit = @parseInt(@prompt("limit: "));
-
-for (var i = 1; i <= limit; i = i + 1) {
-    @println(fizzbuzz(i));
-}
+fun main(): unit {
+    var limit = @parseInt(@prompt("limit: "));
+
+    for (var i = 1; i <= limit; i = i + 1) {
+        @println(fizzbuzz(i));
+    }
+}
```

The new file prints the same output as the old one for the same input
([Appendix A], A.9).

### Files and snippets

Kirby reads source in two ways, and only one of them changes:

- A **file** is read from disk: the program named after `krb run`. The stdlib
  that is loaded before it is built into the binary and goes through `runCode`
  today, but this proposal treats it as a file (Part 2). The new rules apply to
  files.
- A **snippet** is code typed into the REPL (`krb repl`) or given to
  `krb exec`. Snippets keep today's rules: statements are allowed, no `main` is
  needed, and initializers are unrestricted.
  `krb exec '@println("Hello World");'` is the example in `docs/CLI.md`, and
  the REPL runs one statement at a time. See [Q-snippets].

Files play one of two roles:

- The **entry file** is the one named after `krb run` (or the project's `bin`
  file when no path is given). Kirby loads it and then calls its `main`.
- A **library file** is loaded for what it declares. The stdlib is one today,
  and imported modules will be others. A library file is loaded and never run,
  so it needs no `main`.

Both roles follow the same top-level rules. Only the entry file needs `main`.

### What can appear at the top level

A file may hold these declarations, and nothing else:

- `fun`, `struct`, `impl`, `trait`, `type`
- `let` and `var`, with a comptime value (next section)

Anything else at the top level is an error: a call or other expression
statement (including `@println(...)`), an assignment, `if`, `while`, `for`, a
`{ }` block, `break` or `continue`. `return` at the top level is already an error and stays one.

```kirby
fun greet(name: string): unit { @println("Hello " + name); }

greet("Kirby");         // error: a call is a statement
@println("loaded");     // error: a call is a statement
```

Nothing inside a function changes. Function, method and lambda bodies hold
statements as they do today. Functions, structs and trait impls are still
hoisted by the compiler, so a function can be used above the place it is
declared.

### Top-level `let` and `var` take comptime values

A **comptime value** is an expression whose result is fixed by the source text
alone. Evaluating it cannot call a function, read a `var`, or reach outside the
program. Kirby has no compile-time evaluation today ([Appendix A], A.1), so this
proposal has to say what counts. The first version accepts:

| Form                      | Accepted when                                                   | Example                             |
| ------------------------- | --------------------------------------------------------------- | ----------------------------------- |
| Literal                   | always                                                          | `1`, `"kirby"`, `true`, `nil`, `()` |
| Operators and `( )`       | every operand is comptime                                       | `-1`, `24 * 60 * 60`, `"a" + "b"`   |
| Array literal             | every element is comptime                                       | `[1, 2, 3]`, `[]`                   |
| Struct literal            | every field value is comptime                                   | `Point { x: 1, y: 2 }`              |
| Lambda                    | always: only creating it happens on load, its body does not run | `fun (n: f64): f64 { n * n }`       |
| Name of a top-level `fun` | always                                                          | `helper`                            |
| Name of a top-level `let` | that `let` is earlier in the file and is itself comptime        | `alias` in `let alias = names;`     |

Nothing else is accepted. That rules out calls of any kind (natives, functions,
methods, constructors), field access, indexing, `if` and block expressions,
names of `var`s, and assignments.

Other proposals add forms that only build data: tuple literals, tuple struct
values, and enum variants. They would join this table on the same grounds as
struct literals, once each proposal settles how its form is written. A `let` or
`var` with a pattern is a declaration like any other, and its initializer
follows the same rule. See [Related Proposals].

```kirby
struct Point { pub var x: f64; pub var y: f64; }

fun helper(): unit {}

let secondsPerDay = 24 * 60 * 60;
let origin = Point { x: 0, y: 0 };
let names = ["a", "b"];
let square = fun (n: f64): f64 { n * n };
let callback = helper;
let alias = names;
var count = 0;

var start = @clock();            // error: a call
let limit = @parseInt("5");      // error: a call, even to a pure native
var zoo = Zoo.init();            // error: a call
var first = names[0];            // error: indexing
var copy = count;                // error: count is a var
var pending: f64;                // error: no value
```

The first block (up to `var count = 0;`) runs today as it is written
([Appendix A], A.9). Some details:

- **A `var` is fine.** It starts from a comptime value, and functions can
  assign to it later. What cannot happen is code running at load time to set
  it up.
- **A value is required.** `let` already requires an initializer today
  ([Appendix A], A.6). A top-level `var` would too. See [Q-no-value].
- **Load order is not observable.** Initializers still run in source order,
  as they do today. The rule that a name must refer to an _earlier_ `let` keeps
  it that way, and rules out cycles. Today a `var` that names a later `let`
  compiles and then fails at load with `Undefined variable`
  ([Appendix A], A.8). Under this rule it is a compile error.
- **Loading can still fail.** The value is still computed by the VM when the
  file loads, as it is today, and an operator can fail. `let a = 1 / 0;`
  compiles and stops at load with a runtime error ([Appendix A], A.4). So the
  guarantee is precise: loading a file cannot call code, read the environment,
  or write anything, and behaves the same every time. It can still fail, with
  the same error the operator gives anywhere else. Whether the compiler should
  compute these values itself, making `1 / 0` a compile error, is
  [Q-comptime].

Measured over the 773 top-level `let` / `var` declarations in `tests/`, this
rule accepts 606, refuses 156 (calls and other forms), and 11 have no
initializer. Accepting only literals and operators would accept 405. The test
files check language features, so they are not a sample of real programs. See
[Q-comptime].

### `main`

A file that is run needs this declaration at its top level:

```kirby
fun main(): unit {
  @println("Hello, World!");
}
```

- It is a `fun` declaration with no parameters, no generic parameters, and the
  return type `unit`. The return type is spelled out because every function
  needs one today: `fun main() {}` is refused with "needs a return type"
  ([Appendix A], A.3). See [Q-main-signature].
- Running `krb run app.krb` does this, in order: load the stdlib, load
  `app.krb` (declarations only, so this defines names and does nothing else),
  call `main`. When `main` returns, the program ends with exit code 0.
  `@exit(code)` ends it earlier with another code. A program made with
  `krb build` has to do the same. `krb-runtime` (`src/runtime.c`) interprets
  the units attached to it in order, the stdlib and then the program, and
  stops, so the call to `main` and the error for a missing one are added there
  as well.
- Arguments are read with `@argv` and `@argc`, as today.
- A `main` in a library file is never called. Until modules have their own
  namespaces, two files that both define `main` collide the way any two files
  defining the same name do.
- A runtime error inside `main` exits with 70, as before. The last line of the
  trace changes from `in script` to `in main()` ([Q-call-main]).

### Errors

All of these are compile errors and exit with 65, the code the other
compile errors use. Messages follow the two forms that exist today:
`[line 2] Error: Can't return from top-level code.` and
`[line 1] Error at 'struct': ... can only appear at the top level.`

| Mistake                              | Message                                                                             |
| ------------------------------------ | ----------------------------------------------------------------------------------- |
| Statement at the top level           | `[line 3] Error: Only declarations can appear at the top level of a file.`          |
| Top-level `var` / `let` not comptime | `[line 5] Error at 'zoo': A top-level 'var' can only be assigned a comptime value.` |
| Top-level `var` without a value      | `[line 6] Error at 'pending': A top-level 'var' must be assigned a comptime value.` |
| Entry file has no `main`             | `Error: No 'main' function. Running a file needs 'fun main(): unit'.`               |
| `main` has the wrong shape           | `[line 2] Error at 'main': 'main' must be declared as 'fun main(): unit'.`          |

The exact wording is settled by the E2E snapshots when this is implemented. All
statement and initializer errors in a file are reported together, before any
type errors, so one run shows everything to fix.

### Goals and Non Goals

What this proposal covers:

- Loading a file cannot call code, read the environment, or write anything.
- A file states its entry point.
- Keep the REPL and `krb exec` working as they do today.
- Give the [Modules Proposal], [Testing Proposal] and [Projects Proposal] a
  base to build on.

The following is intentionally left out of scope for this proposal:

- Import syntax and namespaces. Those are in the [Modules Proposal].
- Which file is a project's entry file. That is for the [Projects Proposal].
- Running code at compile time, so that calls are allowed in initializers
  ([Q-comptime], option c). That is the mechanism described in the
  [Macros Proposal].
- Parameters or a return value for `main` ([Q-main-signature]).
- A way to check a file without running it ([Q-check-only]).
- Any tool that rewrites existing programs.

### Implementation Plan

Each part follows the project's test-first rule: write one test, see it fail,
make the change, see it pass. Parts 3 to 5 make existing tests fail on purpose,
which Part 6 deals with. Parts 3 to 6 land together, on one branch.

#### Part 1: Files made of declarations must run correctly

This bug exists today, but it has to be fixed first, because after Part 3 a
file with no statements is the normal case, and the bug is most likely in such
a file. This file fails on a plain build, a debug build and an
AddressSanitizer build ([Appendix A], A.10):

```kirby
let a = 1;
let b = 2;
let c = 3;
let d = 4;
let e = 5;
let f = e;
let g = 7;
```

It should run and print nothing. A plain build stops with `Expected 0
arguments but got 1.` and exit code 70, a debug build with `Operands must be
two numbers or two strings.`, and AddressSanitizer reports the VM reading past
the end of the script's bytecode (`run` in `src/vm.c`). The messages are
whatever the bytes past the end happen to say.

The cause is in `endCompiler` (`src/compiler.c`). It writes the script's final
`OP_NIL` and `OP_RETURN` only when the code does not already end in a return:

```c
static uint8_t previousOpCode(void) {
  CompiledFn *fn = currentFn();

  if (fn->codeCount == 0) {
    return 0;
  }

  return fn->code[fn->codeCount - 1];
}
```

```c
  if (previousOpCode() != OP_RETURN) {
    emitImplicitReturn();
  }
```

`previousOpCode` returns the last byte, which is an operand whenever the last
instruction has one. Here the last instruction is `OP_DEFINE_GLOBAL 11 'g'`,
and 11 is the number of `OP_RETURN`, so no return is written. Any function or
script whose last instruction ends in the byte 11 has the same problem.
`emitByte` writes opcodes and operands alike, so the fix has to know where the
last instruction starts: by keeping the offset of the last opcode written, or
by walking the code with the operand sizes in `opInfo` (`src/opcode.c`).

Test first: add the file above as a new E2E test, in its current form without
`main`. It fails now.

#### Part 2: Tell files and snippets apart

In `src/main.c`, `compileSource` learns which kind of source it has, and
`runFile` gets a partner, `loadFile`, for library files. The stdlib is source
text (`KIRBY_STDLIB`), not a file, so `loadFile` takes text.

```diff
+typedef enum {
+  SOURCE_ENTRY_FILE,
+  SOURCE_LIBRARY_FILE,
+  SOURCE_SNIPPET
+} SourceKind;
+
-static CompiledUnit *compileSource(const char *source, bool typecheck);
+static CompiledUnit *compileSource(const char *source, SourceKind kind,
+                                   bool typecheck);
```

```diff
 static void sessionBegin(int argc, char *argv[]) {
   initVM(argc, argv);
   typchkSessionBegin();
-  runCode(KIRBY_STDLIB, /*disassembleCode=*/false);
+  loadFile(KIRBY_STDLIB);
 }
```

`sessionBegin` is shared by `krb run`, `krb example`, `krb repl` and
`krb exec`, so that one line covers all four. `runFile` (used by `krb run` and
`krb example`) passes `SOURCE_ENTRY_FILE`. `repl` and `runCode` (`krb exec`)
pass `SOURCE_SNIPPET`. `krb compile` (`cmdCompile`) and `krb build`
(`cmdBuild`) compile the stdlib and a file without `sessionBegin`, and pass
`SOURCE_LIBRARY_FILE` and `SOURCE_ENTRY_FILE`. This part changes no behavior,
so it has no new test of its own: the existing suite is the test.

The [Tooling Data Proposal] also adds a parameter to `compileSource`, a source
name (its Part 5). The two parameters are separate, and either proposal can
land first.

#### Part 3: Reject top-level statements in files

A new pass in `src/top_level.c`, in the same style as `src/definite_assignment.c`
(its own file and header). It walks the top-level nodes and reports every one
that is not a declaration.

```c
static bool isDeclaration(const AstNode *node) {
  switch (node->kind) {
  case NODE_FUNCTION:
  case NODE_STRUCT:
  case NODE_IMPL:
  case NODE_TRAIT:
  case NODE_TYPE_ALIAS:
  case NODE_VAR_DECL:
    return true;
  default:
    return false;
  }
}
```

The list grows as declarations are added. `enum` ([Enums Proposal]) and a `let`
or `var` with a pattern ([Destructuring Proposal]) each need a case. A pattern
has no single name token, so its error points at the start of the pattern.

It runs from `compileSource` after `parse`, for files only, and it does not stop
the type checker, so both kinds of error appear in one run ([Q-check-where]):

```diff
   AstNode **ast = parse(source, &count, &hadError, &endLine);

+  bool topLevelOk = true;
+  if (!hadError && kind != SOURCE_SNIPPET) {
+    topLevelOk = checkTopLevel(ast, count, kind == SOURCE_ENTRY_FILE);
+  }
+
   if (!hadError && typecheck) {
     if (!typchkCheckProgram(ast, count)) {
       hadError = true;
     }
   }

+  if (!topLevelOk) {
+    hadError = true;
+  }
+
   CompiledUnit *unit = hadError ? NULL : compile(ast, count, endLine);
```

Statement nodes carry a line number but no token, so the message has the form
`[line N] Error: ...`, like "Can't return from top-level code." Test first:
`tests/run/top_level/statements.krb`.

#### Part 4: Check that top-level initializers are comptime

The same pass checks every `NODE_VAR_DECL`. This is a check on the shape of the
tree. Nothing is evaluated, so Kirby needs no new interpreter for it.

| Node kind                                    | Comptime when                                            |
| -------------------------------------------- | -------------------------------------------------------- |
| `NODE_LITERAL` (includes the unit literal)   | always                                                   |
| `NODE_UNARY`, `NODE_BINARY`, `NODE_GROUPING` | every operand is                                         |
| `NODE_AND`, `NODE_OR`, `NODE_NULLISH`        | every operand is                                         |
| `NODE_ARRAY`                                 | every element is                                         |
| `NODE_STRUCT_INIT`                           | every field value is                                     |
| `NODE_FUNCTION` with `isLambda`              | always                                                   |
| `NODE_VARIABLE`                              | it names a top-level `fun`, or an earlier comptime `let` |
| anything else                                | never                                                    |

The pass first collects the names of all top-level functions, then walks the
declarations in order and adds each comptime `let` to a set of names
(`src/stringset.c` already exists). A `var` with no `initializer` in its
`VarDeclNode` is an error. The error is reported at the declaration's `name`
token.

Test first: `tests/run/top_level/comptime_allowed.krb`, then one test per way to be
refused.

#### Part 5: Require and call `main`

For the entry file, the same pass looks for a top-level `NODE_FUNCTION` named
`main` that is not a lambda or a method, with `arity` 0, no generic parameters,
and return type `unit`. A missing `main` and a wrong `main` are separate
errors. A top-level `let` or `var` named `main` is a wrong `main`, not a
missing one.

Kirby calls `main` after the unit has loaded. The VM gets one new entry point
([Q-call-main]):

```c
/**
 * Call the global function `main` with no arguments and run it.
 */
InterpretResult interpretMain(void);
```

```diff
   InterpretResult result = interpret(unit, true);

+  if (result == INTERPRET_OK)
+    result = interpretMain();
+
   if (result == INTERPRET_RUNTIME_ERROR)
     exit(EXIT_CODE_RUNTIME_ERR);
```

Test first: `tests/run/top_level/main_missing.krb`, then one test per case in the
[Testing Plan].

#### Part 6: Migrate the tests, examples and docs

This is most of the work. In `tests/`, 933 of 1110 files have a top-level
statement ([Appendix A], A.8). The `.err` snapshot of each of the 608 tests that
get past compiling holds a bytecode listing of that test's own `<script>`, so
all of those change. The migration follows these rules:

- Statements move into `main`. Declarations stay at the top level.
- A test about global variables keeps its globals. The `var` stays at the top
  level with a comptime value, and only the statements move (see the
  [Testing Plan]).
- For a test whose point survives, the `.out` and `.exit` snapshots must not
  change. Only `.err` may change: the `<script>` listing gets shorter, a `main`
  listing appears, and `in script` lines in traces become `in main()`. Anything
  else is reviewed by hand, as `tests/README.md` step 5 already asks.
- A test whose point is a top-level rule is replaced, not migrated. For example,
  `tests/run/closures/upvalue_global.krb` checks that Kirby refuses a top-level call
  to a global that a function assigns later ("might not be assigned yet",
  exit 65). With no top-level statements and no top-level `var` without a
  value, that program cannot be written.
- Files that only check that something compiles need an empty
  `fun main(): unit {}` if they succeed ([Q-check-only]).

The 13 programs in `examples/` all have top-level statements and are migrated
the same way. The docs to update are `docs/CLI.md` and `book/src/cli.md`
(`krb run` needs `main`, `krb exec` does not), the "Writing A Test" steps in
`tests/README.md`, the samples in `README.md`, `docs/TYPES.md` and
`book/src/`, and `docs/CHANGELOG.md` under the next version.

## Impacts

### Existing Syntax Or Behavior

- **Programs with top-level statements stop compiling.** That is 933 of the 1110
  test files and 13 of 13 example programs. The fix is to move the statements into
  `main`.
- **Top-level initializers that are not comptime stop compiling.** In `tests/`
  that is 156 of 773, plus 11 with no initializer.
- **`krb run FILE` needs a `main`.** A file made only of declarations used to run
  and do nothing. Now it is an error.
- **`main` becomes a special name in the entry file.** It must be
  `fun main(): unit`. `tests/run/closures/upvalue_global.krb` uses `main` as a
  helper name today.
- **Traces end at `main()`** instead of `script`, if [Q-call-main] goes as
  proposed. 180 tests expect a runtime error.
- **Bytecode snapshots change.** The `.err` snapshot of each of the 608 tests
  that compile has a listing of the test's own `<script>`. The 499 compile-error
  tests only change if their code has top-level statements.
- **The "might not be assigned yet" error can no longer come from a top-level
  `var`**, because a top-level `var` always has a value.
- **Unchanged:** the REPL and `krb exec` ([Q-snippets]), `krb parse` and
  `krb lex` (the pass runs after parsing, so `krb parse` still prints the tree
  of an old-style script),
  hoisting, forward references between functions, and where `struct`, `impl`,
  `trait` and `type` may appear.

### Related Proposals

- [Modules Proposal] — this makes importing safe: a file holds declarations, so
  loading it has no side effects, and a module's compiled form does nothing you
  can observe when it loads. It also settles which `main` runs: only the entry
  file's. The rules apply to each file, so they hold whether a module is a
  file or a namespace ([Q-naming]). It adds no import syntax or namespaces,
  which stay in [Q-assembly]. That proposal is updated to say so.
- [Testing Proposal] — `@test(description, closure)` was meant to register a
  test by being called at the top level, and a top-level call is now an error.
  That proposal gets a new question, [Q-register], and its answer about
  colocating tests is updated.
- [Projects Proposal] — its "requiring a main function" is this proposal. Which
  file is a project's entry file stays there. That proposal is updated to link
  here.
- [Debugger Proposal] — a run becomes "load, then call `main`". `stopOnEntry`
  would stop at the first line of `main`, and the `script` frame in the session
  example would be `main`. That proposal is updated to say so.
- [Macros Proposal] — a macro call at the top level that expands into
  declarations (its Part 6) is not a side effect, so the check should run on the
  expanded program ([Q-check-where]). Option (c) of [Q-comptime] would use the
  mechanism in its Part 3. That proposal is updated to say so.
- [String Interpolation Proposal] — it compiles to a native and adds nothing to
  the stdlib, so the stdlib holding only declarations doesn't affect it. It
  rejects placeholders whose type isn't known, which includes globals used in
  a function before they're declared; making globals known first removes that
  case. That proposal is updated to say so.
- [Tooling Data Proposal] — also adds a parameter to `compileSource`, a source
  name (its Part 5). The two parameters are separate, and either proposal can
  land first. It also splits `runFile` into `loadFile` and `runFile`, and both
  pass the path on. That proposal is updated to say so.
- [Enums Proposal] — `enum` would join the declarations a top level may hold.
  A variant value only builds data, so it would be a comptime value when its
  contents are, once [Q-variant-access] and [Q-variant-fields] settle how it is
  written. That proposal is updated to say so.
- [Tuples Proposal], [Tuple Structs Proposal] — a tuple literal and a tuple
  struct value only build data, so they would be comptime values when their
  contents are ([Q-constructor] for the second). A tuple struct declaration is
  a declaration. Those proposals are updated to say so.
- [Destructuring Proposal] — a `let` or `var` with a pattern is still a
  declaration with a comptime initializer. The names a `let` pattern binds
  count as `let` names for the "earlier `let`" rule, and a `var` pattern's do
  not. An array pattern that does not fit fails while the file loads, the same
  kind of failure as `1 / 0` ([Q-array-mismatch]). That proposal is updated to
  say so.
- [Pattern Matching Proposal], [Generic Types Proposal],
  [Primitive Impls Proposal], [Unimplemented Proposal] — many of their code
  samples use top-level statements. `match` is neither a declaration nor a
  comptime form, so it is used inside functions. See [Q-samples].
- [Embedded Library Proposal] — an embedded host is a second kind of host. It loads
  library files, then calls what it needs by name, and it never needs `main`. The
  `interpretMain` of Part 5 is the case of that general call with no arguments,
  and becomes `krbCallGlobal(k, "main", 0, NULL, &result)` there, so it needs no
  function of its own ([Q-call-main]). The interface follows the rules of
  [Q-snippets]: `krbLoad` takes a file and `krbEval` takes a snippet. Its
  compile-only mode (`krb compile`, which that proposal would give an output
  file) overlaps with [Q-check-only]. That proposal is updated to say so.
- [Map Proposal] — a map literal only builds data, so it would be a comptime
  value when its keys and values are. That proposal is updated to say so.
- [Result Proposal] — `?` needs a function that returns a `Result`, so it is
  not valid in a top-level `let` or `var` initializer. Whether it is valid in a
  script's loose statements is [Q-try-top-level]. That proposal is updated to
  say so.
- [Strings Proposal], [String Encoding Proposal], [Trait Types Proposal],
  [For-Each Proposal] — many of their code samples use top-level statements,
  like the other proposals' samples. See [Q-samples].

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E syntax tests should cover all valid and invalid parser/compiler/runtime
error cases. The harness (`scripts/tests.sh`) runs the command in each test's
`.argv` file. Language tests run `krb run`, `tests/exec/` runs `krb exec` and
`tests/repl/` runs `krb repl`, so snippets have tests today ([Q-snippets]).

##### NEW: tests/run/top_level/declarations_only_scripts.krb

The file from Part 1, exactly as shown there.

###### Expected Outcome

No output, exit code 0. (In Part 6 it gets an empty `main`.) Today it fails
with exit code 70.

##### NEW: tests/run/top_level/declarations_and_main.krb

```kirby
struct Point { pub var x: f64; pub var y: f64; }

impl Point {
  pub fun sum(self): f64 = self.x + self.y;
}

let origin = Point { x: 1, y: 2 };

fun main(): unit {
  @println(origin.sum());
}
```

###### Expected Outcome

`3` on stdout, exit code 0.

##### NEW: tests/run/top_level/statements.krb

```kirby
fun greet(): unit {}

@println("hello");
greet();
if (true) { greet(); }
while (false) {}
for (var i = 0; i < 1; i = i + 1) {}
{ greet(); }
var count = 0;
count = 1;

fun main(): unit {}
```

###### Expected Outcome

Seven errors on stderr, one each for lines 3, 4, 5, 6, 7, 8 and 10, each
`Error: Only declarations can appear at the top level of a file.` Line 9 is a
valid declaration. Exit code 65.

##### NEW: tests/run/top_level/comptime_allowed.krb

```kirby
struct Point { pub var x: f64; pub var y: f64; }

fun helper(): unit {}

let secondsPerDay = 24 * 60 * 60;
let origin = Point { x: 0, y: 0 };
let names = ["a", "b"];
let square = fun (n: f64): f64 { n * n };
let callback = helper;
let alias = names;
let nothing = ();
var count = 0;

fun main(): unit {
  count = count + 1;
  @println(secondsPerDay);
  @println(origin.x);
  @println(names[1]);
  @println(square(4));
  callback();
  @println(count);
}
```

###### Expected Outcome

```
86400
0
b
16
1
```

Exit code 0.

##### NEW: tests/run/top_level/comptime_not_allowed.krb

```kirby
struct Zoo { pub var animals: f64; }

impl Zoo {
  pub fun init(): Zoo = Zoo { animals: 1 };
}

var count = 0;
let names = ["a", "b"];

var start = @clock();
let limit = @parseInt("5");
var zoo = Zoo.init();
var first = names[0];
var copy = count;
var later = tail;
let tail = 1;

fun main(): unit {}
```

###### Expected Outcome

Six errors on stderr, at `start` (line 10), `limit` (11), `zoo` (12), `first`
(13), `copy` (14) and `later` (15), each saying the declaration "can only be
assigned a comptime value". Exit code 65.

##### NEW: tests/run/top_level/comptime_no_value.krb

```kirby
var pending: f64;

fun main(): unit {}
```

###### Expected Outcome

`[line 1] Error at 'pending': A top-level 'var' must be assigned a comptime
value.` Exit code 65.

##### NEW: tests/run/top_level/comptime_division_by_zero.krb

```kirby
let broken = 1 / 0;

fun main(): unit {}
```

###### Expected Outcome

The file loads and fails before `main` runs. This records the limit described in
[Top-level `let` and `var` take comptime values]. stderr:

```
function / expects argument 1 to be a non-zero number but got 0.
[line 1] in script
```

Exit code 70.

##### NEW: tests/run/top_level/main_missing.krb

```kirby
fun helper(): unit { @println("never runs"); }
```

###### Expected Outcome

`Error: No 'main' function. Running a file needs 'fun main(): unit'.` Nothing on
stdout. Exit code 65.

##### NEW: tests/run/top_level/main_with_parameter.krb, main_returns_number.krb, main_not_a_function.krb

```kirby
fun main(args: f64): unit {}
```

```kirby
fun main(): f64 = 0;
```

```kirby
let main = fun (): unit {};
```

###### Expected Outcome

Each file gives `Error at 'main': 'main' must be declared as 'fun main(): unit'.`
with its own line number. Exit code 65.

##### NEW: tests/run/top_level/main_exit_code.krb

```kirby
fun main(): unit {
  @println("before");
  @exit(3);
  @println("after");
}
```

###### Expected Outcome

`before` on stdout, exit code 3.

##### NEW: tests/run/top_level/main_runtime_error.krb

```kirby
fun divide(a: f64, b: f64): f64 = a / b;

fun main(): unit {
  @println(divide(1, 0));
}
```

###### Expected Outcome

stderr, with `main()` as the last frame (no `script` line, if [Q-call-main]
goes as proposed). Exit code 70.

```
function / expects argument 1 to be a non-zero number but got 0.
[line 1] in divide()
[line 4] in main()
```

##### CHANGED: tests/run/variable_reference.krb

A test about globals keeps its globals. Only the statement moves.

```diff
 var greeting = "Hello";
 var name = "World";

-@println(greeting + " " + name);
+fun main(): unit {
+  @println(greeting + " " + name);
+}
```

###### Expected Outcome

`.out` (`Hello World`) and `.exit` (`0`) are unchanged. `.err` changes: the
`<script>` listing no longer has the `@println` call, and a `main` listing appears. This
is the shape of most of the 933 files that have a top-level statement. A change to any `.out` or
`.exit` of a migrated test means the migration changed what the test checks, and
is reviewed by hand.

## Questions

### **Q:** How much is a comptime value?

<!-- [Q-comptime]: #q-how-much-is-a-comptime-value -->

**Status:** Open

Kirby has no compile-time evaluation today ([Appendix A], A.1), so this
proposal has to say what "comptime" means. Options:

- **(a) Literals and operators.** `1`, `-1`, `"a" + "b"`, `24 * 60 * 60`. It is
  the smallest version, but it rules out arrays, structs and lambdas.
- **(b) (a), plus arrays, struct literals, lambdas, and the names of top-level
  `fun`s and earlier comptime `let`s** (proposed). None of these can call code
  or read anything, and a struct literal does not run a `Default` impl
  ([Appendix A], A.7).
- **(c) A real compile-time evaluator, so calls are allowed**, for example
  `let side = @sqrt(16);`. The compiler would have to run code, which is the
  mechanism the [Macros Proposal] describes in its Part 3. It could also make
  `1 / 0` a compile error.

Over the 773 top-level `let` / `var` declarations in `tests/`, (a) accepts 405
and (b) accepts 606. 156 use calls or other forms that neither accepts, and 11
have no initializer. The test files are not a sample of real programs, but they
show that (a) alone leaves most aggregates behind.

Under (a) and (b) the VM still computes the value when the file loads, as it
does today. The rule only limits what the expression may contain. That is why
`let a = 1 / 0;` still fails at load, and it is fine as long as the guarantee is
stated as "cannot call code, read the environment, or write anything".

Two smaller choices inside (b):

- A name may refer to an _earlier_ `let` only (proposed). The alternative is any
  `let`, in any order, which needs the compiler to sort declarations by what
  they depend on.
- `if` and block expressions are left out to keep the list short. They could be
  added later without breaking anything.

### **Q:** Should `main` take parameters or return a value?

<!-- [Q-main-signature]: #q-should-main-take-parameters-or-return-a-value -->

**Status:** Open

Proposed: `fun main(): unit`, with no parameters and no return value.

- Arguments already have `@argv` and `@argc`.
- A non-zero exit code already has `@exit(code)`.
- Every function needs a declared return type today, so `unit` is the natural
  spelling.

Alternatives are `main(args: [string])` and
a `main` that returns the exit code (which gives two ways to exit). Accepting
more signatures later does not break a file that uses `fun main(): unit`.

### **Q:** Are top-level declarations without a value allowed?

<!-- [Q-no-value]: #q-are-top-level-declarations-without-a-value-allowed -->

**Status:** Open

Proposed: no. A top-level `var` needs a comptime value. (`let` already needs an
initializer today.)

Nothing at the top level can assign to it, so it could only be set by a function
later. That leaves a gap: today the checker accepts a read of such a variable
inside a function, and only refuses one at the top level ([Appendix A], A.6).
With modules, that gap would exist in every file. 11 of the 773 declarations in
`tests/` have no initializer.

To migrate one, give it a starting value, for example
`var handler: fun () => unit = fun () {};`. For a type with no natural starting
value, create the value in `main` and pass it down.

### **Q:** What happens when a file has no `main`?

<!-- [Q-missing-main]: #q-what-happens-when-a-file-has-no-main -->

**Status:** Open

Proposed: a compile error with exit code 65, and only when running an entry
file. A library file needs no `main`.

The problem is found before anything runs, and it is a problem with the file,
like the other errors that exit with 65. The alternative is exit code 64, the
code `krb` uses for a usage error, on the view that the person asked Kirby to run
a file that cannot be run.

### **Q:** Are the REPL and code snippets exempt?

<!-- [Q-snippets]: #q-are-the-repl-and-code-snippets-exempt -->

**Status:** Open

Proposed: yes. The REPL and `krb exec` keep today's rules: statements, no `main`, no
limit on initializers. The stdlib that is loaded before a snippet is a library
file, so it follows the new rules.

The harness runs the command in each test's `.argv` file, so a snippet test
is an `.argv` of `exec` and `$(cat $file)`, which runs the contents of the
test's program as a snippet. `tests/exec/` and `tests/repl/` hold such tests.
What is left is the tests that show snippets still allow statements once
every other test is migrated.

### **Q:** How does Kirby call `main`?

<!-- [Q-call-main]: #q-how-does-kirby-call-main -->

**Status:** Open

- **(a) The host calls it** (proposed). After the file has loaded, `runFile`
  asks the VM to call the global `main` (Part 5). The compiled unit only defines
  things and never contains a call to `main`, so the same unit works as a library
  file or an entry file, which suits the [Modules Proposal]. Traces end at
  `main()`. If the [Embedded Library Proposal]'s call by name (its Part 5) is
  built first, `interpretMain` is a use of it and not a new function.
- **(b) The compiler adds the call.** When compiling an entry file, it adds a call
  to `main` at the end of the script. Nothing new is needed in the VM. But an
  entry file and a library file compile to different units, and traces end with an
  extra `in script` line.

### **Q:** Where does the top-level check run?

<!-- [Q-check-where]: #q-where-does-the-top-level-check-run -->

**Status:** Open

Proposed: a separate pass after parsing and before type checking, for files
only (Part 3).

- **(a) A separate pass** (proposed). `krb parse` keeps printing the tree of an old-style
  script, because it only calls `parse`. Every offending statement is reported
  in one run. The cost is that statement nodes have a line but no token, so
  messages have no `at 'x'` part.
- **(b) In the parser**, next to the existing rule that keeps `struct` and
  friends out of blocks. Messages can name the exact token. But `parse` is
  shared by every mode, so it would need to be told the kind of source, and the
  parser stops after the first error in a statement.

Once macros exist, the [Macros Proposal] expands them before type checking, and a
top-level macro call that expands into declarations is not a side effect. The
pass should run on the expanded program, right after expansion.

### **Q:** How are tests that only check compilation written?

<!-- [Q-check-only]: #q-how-are-tests-that-only-check-compilation-written -->

**Status:** Open

174 test files contain only declarations. Most are compile-error tests. The 12
that succeed today do nothing when run, and under this proposal they need a
`main`.

- **(a) An empty `fun main(): unit {}`** (proposed for now). It adds nothing to
  the CLI, but it adds noise to every such test.
- **(b) A check-only mode** that compiles and type checks without running and
  without needing `main`. `krb compile FILE` is that mode today: it type checks
  and compiles the file and the stdlib, runs nothing, and prints `Compiled!`.
  A test uses it through its own `.argv` file
  (`compile`, then `$file`), so the harness needs no per-test option either.
  This is useful for tools too. Nothing new is needed in the CLI or the harness;
  what is left to decide is whether tests like these use it.

### **Q:** When do code samples in other proposals change?

<!-- [Q-samples]: #q-when-do-code-samples-in-other-proposals-change -->

**Status:** Open

79 `kirby` code blocks in thirteen other proposals use top-level statements:
the [Tuple Structs Proposal] (12), the [Tuples Proposal] (12), the
[Enums Proposal] (12), the [Pattern Matching Proposal] (12), the
[Destructuring Proposal] (6), the [String Interpolation Proposal] (6), the
[Lua Transpiler Proposal] (6), the [Generic Types Proposal] (4), the
[Primitive Impls Proposal] (4), the [Embedded Library Proposal] (2), the
[Unimplemented Proposal] (1), the [Debugger Proposal] (1) and the
[Tooling Data Proposal] (1). A block counts when a line outside any braces is
not a declaration, a comment or a closing brace ([Appendix A], A.11).

Proposed: leave them. A sample that shows today's behavior is a checked fact
about `from_commit` and stays true. A sample that shows proposed behavior is
rewritten with a `main` the next time that proposal is edited after this one is
accepted. Only the design points that this proposal changes are edited now (see
[Related Proposals]).

## Appendix A: Checking the claims

Build with `just build`. The commands below use the `build/krb` it produces.
`just test` uses `build/kirby-test`, which is the same program built with
`DEBUG_PRINT_CODE` and `DEBUG_STRESS_GC`. The results in this proposal come
from a build at `from_commit`, on Ubuntu 24.

**A.1 No compile-time evaluation exists.** This prints only a comment about
`break` (`src/compiler.c:602`):

```shell
grep -rniE "comptime|constant.?fold|fold(Constant|Const)|isConst|constExpr|compile.?time" src/*.c src/*.h
```

**A.2 What is refused at the top level today.**

```
$ printf '@println(1);\nreturn;\n@println(2);\n' > a.krb && krb run a.krb
[line 2] Error: Can't return from top-level code.          (exit 65)

$ printf '{ struct S { var a: f64; } }\n' > b.krb && krb run b.krb
[line 1] Error at 'struct': 'struct', 'impl', 'trait', and 'type' are declarations and can only appear at the top level.   (exit 65)
```

The file from the Problem Statement (`var start = @clock();`,
`@println("counter loaded");`, and a `fun`) prints `counter loaded` and exits 0.

**A.3 `main` is an ordinary name today.**

```
$ printf 'fun main() { @println("in main"); }\n@println("top");\n' > c.krb && krb run c.krb
[line 1] Error at 'main': 'main' needs a return type.       (exit 65)
```

With `fun main(): unit { @println("in main"); }` the file compiles, prints
`top`, and `main` runs only if the file calls it.

**A.4 An operator can fail while a file loads.**

```
$ printf 'let a = 1 / 0;\n@println(a);\n' > d.krb && krb run d.krb
function / expects argument 1 to be a non-zero number but got 0.
[line 1] in script                                          (exit 70)
```

**A.5 The stdlib is loaded by running it, and snippets share the path.**
`stdlib/stdlib.krb` holds only declarations. In `src/main.c`, `sessionBegin`
runs it with `runCode(KIRBY_STDLIB, ...)`, and `sessionBegin` is called by
`krb run` and `krb example` (through `runProgram`), `krb repl` and `krb exec`.
`krb compile` and `krb build` compile it with the file instead of running it.
`compileSource` is called by `repl`, `runFile`, `runCode`, `compileCode` and
`cmdBuild`. `krb parse` calls `parse` directly. `compile` in `src/compiler.c`
emits the top-level functions, then the structs, then the trait impls ahead of
everything else, which is why a function can be used above its declaration.

**A.6 A variable with no value.** `let` already needs an initializer. A read of
an unassigned `var` is refused at the top level but not inside a function:

```
$ printf "let name: string;\n@println(1);\n" > e.krb && krb run e.krb
[line 1] Error at 'string': 'let' binding requires an initializer.   (exit 65)

$ printf 'var x: f64;\n@println(x);\n' > f.krb && krb run f.krb
[line 2] Error at 'x': 'x' might not be assigned yet.       (exit 65)

$ cat g.krb
var counter: i64;
fun bump(): unit { counter = counter + 1; }
fun get(): i64 = counter;
$ krb run g.krb                                             (exit 0)
```

**A.7 A struct literal does not run user code.** With
`impl Default for P { fun default(): P = { @println("default() ran"); P { x: 0 } }; }`
and `let p = P { x: 1 };`, printing `p.x` prints `1` and never
`default() ran`.

**A.8 How much existing code is affected.** `krb parse FILE` prints one
S-expression per top-level node. This script reads them:

```python
#!/usr/bin/env python3
# Usage: python3 survey.py $(find tests -name '*.krb' | sort)
# Reads `build/krb parse FILE`, which prints one S-expression per top-level node.
import collections, re, subprocess, sys

DECLS = {'fun', 'struct', 'impl', 'trait', 'type', 'var', 'let'}
LITERAL = re.compile(r'-?\d+(\.\d+)?|"(?:\\.|[^"\\])*"|true|false|nil')
OPS = {'+', '-', '*', '/', '%', '==', '!=', '<', '>', '<=', '>=', '!',
       'and', 'or', '??', 'group'}

def parse(s):
    toks = re.findall(r'"(?:\\.|[^"\\])*"|[()]|[^\s()]+', s)
    def go(i):
        if toks[i] != '(':
            return toks[i], i + 1
        out, i = [], i + 1
        while toks[i] != ')':
            node, i = go(i)
            out.append(node)
        return out, i + 1
    return go(0)[0]

def initializer(decl):            # (var NAME [: TYPE] [INIT])
    i = 4 if len(decl) > 2 and decl[2] == ':' else 2
    return decl[i] if len(decl) > i else None

def comptime(n, tier, funs, lets):
    if n == []:
        return True                                   # unit: ()
    if isinstance(n, str):
        return bool(LITERAL.fullmatch(n)) or (tier == 'B' and (n in funs or n in lets))
    head, args = n[0], n[1:]
    if head in OPS:
        return all(comptime(a, tier, funs, lets) for a in args)
    if tier == 'B':
        if head == 'array':
            return all(comptime(a, tier, funs, lets) for a in args)
        if head == 'struct-init':                     # (struct-init P (field x 1) ...)
            return all(comptime(f[2], tier, funs, lets) for f in args[1:])
        if head == 'lambda':
            return True
    return False

files = statement_files = 0
result = collections.Counter()
for path in sys.argv[1:]:
    out = subprocess.run(['build/krb', 'parse', path], capture_output=True, text=True).stdout
    files += 1
    heads = re.findall(r'^\(([^ )]+)', out, re.M)
    statement_files += any(h not in DECLS for h in heads)
    funs = set(re.findall(r'^\(fun (\S+)', out, re.M))
    lets = {'A': set(), 'B': set()}
    for line in out.splitlines():
        if not re.match(r'\((var|let) ', line):
            continue
        try:
            decl = parse(line)
        except IndexError:                            # node printed over several lines
            continue
        init = initializer(decl)
        for tier in 'AB':
            if init is None:
                result[tier, 'no initializer'] += 1
            elif comptime(init, tier, funs, lets[tier]):
                result[tier, 'comptime'] += 1
                if decl[0] == 'let':
                    lets[tier].add(decl[1])
            else:
                result[tier, 'not comptime'] += 1

print(f"{statement_files} of {files} files have a top-level statement")
total = sum(v for (t, _), v in result.items() if t == 'A')
print(f"{total} top-level var/let declarations")
for tier, name in (('A', 'literals and operators'),
                   ('B', 'A, plus arrays, structs, lambdas, references')):
    print(f"tier {tier} ({name}): "
          f"{result[tier, 'comptime']} comptime, "
          f"{result[tier, 'not comptime']} not, "
          f"{result[tier, 'no initializer']} no initializer")
```

Run it from the repo root as `python3 survey.py $(find tests -name '*.krb' | sort)`.
It prints:

```
933 of 1110 files have a top-level statement
773 top-level var/let declarations
tier A (literals and operators): 405 comptime, 357 not, 11 no initializer
tier B (A, plus arrays, structs, lambdas, references): 606 comptime, 156 not, 11 no initializer
```

Run on `examples/*.krb` its first line says 13 of 13. Of the 1110 files, 174
have only declarations, and 12 of those are tests expected to succeed. The
expected exit codes come from the `.exit` files next to each `.krb`: 499 are
65 (compile error), 426 are 0, 180 are 70 (runtime error), one is 7 and one is
71, and three files have no snapshot. The 608 tests that get past compiling
are the 426 + 180 + 1 + 1. A `var` that names a later `let` compiles today and
fails at load:

```
$ printf 'var later = tail;\nlet tail = 1;\n@println(later);\n' > h.krb && krb run h.krb
Undefined variable 'tail'.
[line 1] in script                                          (exit 70)
```

**A.9 The samples in this proposal.** The migrated `fizzbuzz` and the
allowed-forms block, each with `main();` added at the end (which stands in for
the call Kirby would make), run today. The migrated `fizzbuzz` prints the same
15 lines as `examples/fizzbuzz.krb` for the input `15`. The allowed-forms block
prints the five lines shown in the Testing Plan. The block under "Top-level
`let` and `var` take comptime values" (up to `var count = 0;`) also runs on its
own today, on a plain build and a debug build, and prints nothing.

**A.10 The Part 1 bug.** Save the file from Part 1 as `full.krb`.

- A plain build: `krb run full.krb` prints `Expected 0 arguments but got 1.`
  and `[line 332044] in script`, and exits with 70.
- A debug build (`build/kirby-test`): `Operands must be two numbers or two
strings.` and a nonsense line number, exit 70. The bytecode listing of
  `<script>` ends with `OP_DEFINE_GLOBAL 11 'g'`, with no `OP_NIL` or
  `OP_RETURN` after it.
- An AddressSanitizer build (`just test-with-memcheck` builds one):
  `heap-buffer-overflow` in `run`.
- Adding any statement at the end, such as `@println(g);`, makes it pass. So
  does removing `let a = 1;`, which moves `g` to another constant number.

**A.11 Samples in other proposals.** This script counts the `kirby` blocks in
each proposal that have a top-level statement:

````python
#!/usr/bin/env python3
# Usage: python3 samples.py proposals/*/PROPOSAL.md
import re, sys

DECL = re.compile(r'^(fun |struct |impl |trait |type |let |var |pub |enum |}|//|#|$)')

for path in sys.argv[1:]:
    blocks = 0
    for m in re.finditer(r'```kirby\n(.*?)```', open(path).read(), re.S):
        depth, statement = 0, False
        for line in m.group(1).splitlines():
            s = line.strip()
            if depth == 0 and s and not DECL.match(s) and not s.startswith('...'):
                statement = True
            depth = max(0, depth + line.count('{') - line.count('}'))
        blocks += statement
    if blocks:
        print(path, blocks)
````

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Declaration**: A top-level item that introduces a name or adds to one: `fun`,
  `struct`, `impl`, `trait`, `type`, `let` and `var`.
- **Statement**: Anything else that can appear in a function body:
  `if`, `while`, `for`, a `{ }` block, a call or assignment used as a statement,
  `break`, `continue` and `return`.
- **Load**: Compile a file and run its top level once, so that the names it
  declares exist. With this proposal that run only defines things.
- **Run**: Load an entry file, then call its `main`.
- **Entry file**: The file named after `krb run`.
- **Library file**: A file loaded for what it declares and never run: the stdlib
  today, and imported modules later.
- **Snippet**: Code given to the REPL or to `krb exec`.
- **Comptime value**: An expression whose result is fixed by the source text
  alone. It cannot call a function, read a `var`, or reach outside the program.
  The exact list for the first version is in [Top-level `let` and `var` take
  comptime values].
- **Side effect**: Anything code does besides work out its own value: printing,
  reading input, reading the clock or environment, calling code that does, or
  changing something that outlives it.
- **Script**: The compiler's name for the one function that holds a file's top
  level. It is listed as `<script>` in bytecode output and shown as `script` in
  traces.
- **Hoisting**: The compiler emits functions, structs and trait impls before
  the rest of the file, so they can be used above the place they are declared.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals
[Testing Plan]: #testing-plan
[Appendix A]: #appendix-a-checking-the-claims
[Top-level `let` and `var` take comptime values]: #top-level-let-and-var-take-comptime-values

<!-- Proposals -->

[Modules Proposal]: ../modules/PROPOSAL.md
[Testing Proposal]: ../testing/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Unimplemented Proposal]: ../unimplemented/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Lua Transpiler Proposal]: ../lua-transpiler/PROPOSAL.md
[Map Proposal]: ../map/PROPOSAL.md
[Result Proposal]: ../result/PROPOSAL.md
[Strings Proposal]: ../strings/PROPOSAL.md
[String Encoding Proposal]: ../string-encoding/PROPOSAL.md
[Trait Types Proposal]: ../trait-types/PROPOSAL.md
[For-Each Proposal]: ../for-each/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-assembly]: ../modules/PROPOSAL.md#q-how-are-modules-named-resolved-and-assembled
[Q-naming]: ../modules/PROPOSAL.md#q-is-a-module-named-by-its-file-path-or-by-a-namespace
[Q-variant-access]: ../enums/PROPOSAL.md#q-how-is-a-variant-written-and-what-else-shares-its-name
[Q-variant-fields]: ../enums/PROPOSAL.md#q-how-are-the-values-inside-a-variant-declared-built-and-read
[Q-constructor]: ../tuple-structs/PROPOSAL.md#q-how-is-a-tuple-struct-constructed
[Q-array-mismatch]: ../destructuring/PROPOSAL.md#q-what-happens-when-an-array-pattern-does-not-fit
[Q-register]: ../testing/PROPOSAL.md#q-how-are-tests-registered-when-the-top-level-cannot-make-calls
[Q-try-top-level]: ../result/PROPOSAL.md#q-what-does-the-early-return-operator-do-at-the-top-level-of-a-script

<!-- Questions -->

[Q-comptime]: #q-how-much-is-a-comptime-value
[Q-main-signature]: #q-should-main-take-parameters-or-return-a-value
[Q-no-value]: #q-are-top-level-declarations-without-a-value-allowed
[Q-missing-main]: #q-what-happens-when-a-file-has-no-main
[Q-snippets]: #q-are-the-repl-and-code-snippets-exempt
[Q-call-main]: #q-how-does-kirby-call-main
[Q-check-where]: #q-where-does-the-top-level-check-run
[Q-check-only]: #q-how-are-tests-that-only-check-compilation-written
[Q-samples]: #q-when-do-code-samples-in-other-proposals-change
