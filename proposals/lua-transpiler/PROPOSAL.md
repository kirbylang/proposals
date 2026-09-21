---
status: Draft
created: 2026-09-20
from_commit: 662d98b
---

# Proposal: Turning Kirby Into Lua

This proposal asks whether Kirby programs can be turned into Lua source code,
and what that would cost. It is a feasibility study. It gathers evidence, lists
the differences between the two languages, and sketches a plan. It is not a
commitment to build anything. The changes here have no bearing on other proposals.

The reason to ask is reach. Many programs that want a scripting language already
contain a Lua. A game engine or an editor that has Lua cannot run a Kirby
program, and would have to take on Kirby's virtual machine to do it, which is
what the [Embedded Library Proposal] describes. If the Kirby compiler could also write
Lua, a Kirby program (checked by Kirby's type checker) would run wherever Lua
does, with nothing of Kirby shipped except the program.

The short answer is in [Q-feasible]. The plan is in parts that are ordered so that
stopping after any of them leaves something that works.

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
- When the text says what Lua "does", that was checked by running Lua 5.1.5,
  LuaJIT 2.1 and Lua 5.4.6 ([Appendix A]).

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code
  blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any Lua code will be displayed in `lua` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.
- **No translator exists yet.** Every piece of Lua in this document was written
  by hand, following the rules the proposal suggests, and run against Kirby's
  real output. It shows that the rules work. It does not show that a program
  can write them.

## Problem Statement

A host that already has Lua cannot run Kirby. There are two ways to change that.
The host can take on Kirby's runtime, which is the [Embedded Library Proposal]. Or
Kirby can turn its programs into Lua before they reach the host. The second is
this proposal. It suits a host that cannot or would not add a second virtual
machine, and it keeps working with whatever tools that host has for Lua.

### What Kirby already has

Checked at `from_commit` ([Appendix A]):

- **A front end that already understands the program.** The parser and type
  checker turn source into a syntax tree of 34 kinds of node (`src/ast.h`) and
  reject programs that do not type check. `compiler.c` walks that tree and
  writes bytecode. It is 1,340 lines. A Lua writer would do the same walk and
  write text. ([A.5])
- **A language that is close to Lua in the ways that matter.** Both have closures
  that capture variables and not values ([A.1], row F). Both use byte strings
  (`@len("é")` is 2 in Kirby and `#"é"` is 2 in Lua, [A.5]). A condition in Kirby
  has to be a `bool`, so Lua's different rule for what counts as true can never
  show through in an `if` or a `while` ([A.5]). `and` and `or` accept only
  `bool` values, so they map straight onto Lua's.
- **Programs that translate exactly.** By hand, `examples/fizzbuzz.krb` and
  `examples/stringBuilder.krb` (structs, `impl`, two trait impls, methods,
  `Self`, method chaining) each produce output identical to Kirby's on Lua 5.1,
  LuaJIT and Lua 5.4. So does a loop that uses both `break` and `continue`, and
  a program that makes closures in a loop. ([A.2], [A.3])
- **A speed story that does not need help.** The benchmark from the [Debugger
  Proposal], translated by hand, ran faster than Kirby's own VM in every run: 1.1
  to 1.4 times as fast on Lua 5.1, 1.2 to 1.6 times on Lua 5.4 and 2.7 to 4.0
  times on LuaJIT, across three sets of runs ([A.4]).
  That is a pleasant side effect, but it is not the reason to do this. The reason
  is reach.

### What is missing

1. **There is no writer.** Nothing turns a syntax tree into Lua.
2. **The type checker's answers are thrown away.** Lua has no types, but the
   translation needs them. `a + b` is an addition in Lua and `a .. b` is the
   join of two strings. Kirby decides at run time which one it is (`OP_ADD`),
   and Lua cannot, so the writer must know the operand types. `obj.f(x)` is a
   call of a function stored in a field in Kirby, and Lua wants that written
   `obj.f(x)` when `f` is a field and `obj:f(x)` when it is a method. The
   checker knows. But an `AstNode` has no type on it, and the only thing the
   checker leaves behind is a small side table that says which struct each
   `impl` is for (`resolved_impl_targets.c`). ([A.5])
3. **The two languages disagree in small, sharp places.** All of these were run:

   | Kirby                                                      | Lua, written the obvious way                                            |
   | ---------------------------------------------------------- | ----------------------------------------------------------------------- |
   | `-5 % 3` is `-2`                                           | `-5 % 3` is `1`; `math.fmod(-5, 3)` is `-2`                             |
   | `1 / 0` stops the program                                  | `1 / 0` is `inf`                                                        |
   | `print 3` shows `3.000000`                                 | `print(3.0)` shows `3` (5.1, LuaJIT) or `3.0` (5.4)                     |
   | `@numberToString(1/3)` has 15 digits                       | `tostring(1/3)` has 14 digits                                           |
   | an array can hold `nil`, and `@len` counts it              | `#` of a table with holes is 0, 1 or 2, depending on version and layout |
   | closures made in a `for` loop share one variable (`3 3 3`) | a numeric `for` gives each its own (`0 1 2`)                            |
   | `a + b` joins two strings                                  | `"a" + "b"` is an error                                                 |
   | `floor(2^62) * 4` is `1.8e19`                              | on Lua 5.4 the same is `0`, because whole numbers wrap around           |
   | `pub` is checked when the program runs                     | a plain table has no privacy                                            |

   Each one has a rule that makes Lua behave like Kirby, and the rules are in
   [Proposed Changes]. The point of listing them here is that the last column is
   what a careless translation would get wrong without any error.

4. **The Lua versions disagree with each other.** `goto`, which is the neat way to
   write `continue`, is a syntax error on Lua 5.1 and works on LuaJIT and 5.4.
   `unpack` is `table.unpack` on 5.4. `math.round` and `math.trunc` do not exist on
   any of the three. Lua 5.4 has a second kind of number, the integer, that wraps
   around instead of growing. ([A.1], [A.3], [A.7])
5. **Lua has limits that Kirby does not.** A Lua function may have at most 200
   local variables on all three versions, and a closure may use at most 60
   variables from outside itself on Lua 5.1 and LuaJIT (255 on 5.4). A Kirby
   file with a few hundred top-level names cannot be written as a file full of
   `local` lines. ([A.7])
6. **There is no support library.** Printing a number in Kirby's way, checking
   for a zero divisor, keeping an array's length, and most of the 63 natives
   need small Lua functions that ship with the output.
7. **Errors and line numbers are different.** A Kirby runtime error prints its
   message and `[line 3] in script`, and the process exits with 70. A Lua error
   has Lua's line numbers, which point into the generated file.
8. **Order of evaluation.** Kirby evaluates operands left to right. The Lua
   manual does not say what order Lua uses. One of its authors wrote in 2001
   that code is in general generated left to right, and added that a program
   should not depend on it. Another said in 2016 that it is safe to assume the
   order is unspecified ([A.9]). So the writer cannot rely on it. When two
   operands both have side effects, it has to put them in named temporaries,
   which fixes the order.

## Proposed Changes

### What it would look like

Kirby's `examples/stringBuilder.krb`, and the Lua that the rules below produce
for it. The Lua was written by hand and prints `Hello World` on Lua 5.1, LuaJIT
and Lua 5.4, the same as Kirby does ([A.2]).

```kirby
struct StringBuilder {
    var value: Array;
}

impl StringBuilder {
    pub fun add(self, add: string): Self {
        @arrPush(self.value, add);

        self
    }
}

impl Default for StringBuilder {
    fun default(): Self = Self { value: [] };
}

impl Display for StringBuilder {
    fun toString(self): string = @arrJoin(self.value, "");
}

let builder = StringBuilder.default()
    .add("Hello")
    .add(" ")
    .add("World");

print builder.toString();
```

```lua
local krt = require("krt")

local StringBuilder = { __name = "StringBuilder" }
StringBuilder.__index = StringBuilder

function StringBuilder.add(self, add)
  krt.arrPush(self.value, add)
  return self
end

function StringBuilder.default()
  return setmetatable({ value = krt.array({}) }, StringBuilder)
end

function StringBuilder.toString(self)
  return krt.arrJoin(self.value, "")
end

local builder = StringBuilder.default():add("Hello"):add(" "):add("World")
krt.print(builder:toString())
```

Things to notice: a struct is a table, and its methods live in a second table
that Lua finds through the struct's _metatable_ (a table that tells Lua where to
look when a name is not found). A method that takes `self` is called with a
colon. An array is a table with a count kept next to its items. `print` goes
through a support function, because Kirby shows numbers differently from Lua.

### Rules

Five rules for the Lua that is written:

1. **Same output.** For a program that does not fail, the text written to stdout
   is byte for byte what Kirby writes. The 46 test programs that use a native
   Lua cannot offer everywhere, or that check a private field, are the known
   exceptions ([A.8]).
2. **Failure is a Lua error.** A program that would stop with a runtime error in
   Kirby raises a Lua error with the same message. A small runner turns it into
   Kirby's report and exit code 70 ([Part 8]).
3. **Nothing but Lua's standard library.** The output is one Lua file and one
   support file, `kirby_rt.lua`, that can also be pasted into the output.
4. **The order of evaluation is written down, not hoped for.** Where two operands
   could both change something, the writer names the first result before it
   evaluates the second.
5. **Lua 5.1 is the floor.** The output runs on Lua 5.1, LuaJIT and Lua 5.4.
   Faster or neater forms for one target are options ([Part 7], [Q-target]).

### The rules, one construct at a time

Every row was tried by hand ([A.1] to [A.3]) except the ones marked _design_.

| Kirby                                              | Lua                                                                                                                                    | Needs from the checker            |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| a number, `3`                                      | written as a float, `3.0`, so that Lua 5.4 stays out of its integer arithmetic                                                         | —                                 |
| `a % b`                                            | `math.fmod(a, b)`                                                                                                                      | —                                 |
| `a / b`                                            | `krt.div(a, b)`, which stops on a zero divisor. Plain `a / b` when `b` is a non-zero number written in the source (_design_)           | —                                 |
| `a + b`                                            | `a + b` for numbers, `a .. b` for strings                                                                                              | operand types                     |
| `a == b` on numbers, strings, bools, nil, arrays   | `a == b` (an array is equal only to itself, as in Kirby)                                                                               | operand types                     |
| `a == b` on structs                                | `Type.equals(a, b)`, from the `impl Eq` that Kirby already requires                                                                    | operand types                     |
| `a and b`, `a or b`, `!a`                          | `a and b`, `a or b`, `not a`                                                                                                           | —                                 |
| `a ?? b`                                           | a temporary: `local t = a; if t == nil then t = b end`. Lua's `or` would also replace `false`                                          | —                                 |
| `print v`                                          | `krt.print(v)`: numbers with six decimals, arrays as `[1.000000, 2.000000]`, an instance as `Name instance`                            | operand type (or a run-time test) |
| `if` and blocks used as values                     | statements that store into a temporary. `return if ...` becomes a `return` in each branch                                              | —                                 |
| `for (init; cond; step) body`                      | `do init; while cond do body; step end end`, so all closures share one variable, like Kirby                                            | —                                 |
| `break`, `continue`                                | a flag and a `repeat ... until true` block on every target, or `goto` on the targets that have it ([Part 7])                           | —                                 |
| `fun` and lambdas, closures                        | Lua functions and closures                                                                                                             | —                                 |
| top-level `fun`, `let`, `var`, `struct`            | fields of one table for the file (`M.tick`), not `local` names, so that the 200-locals and 60-upvalues limits cannot be hit (_design_) | —                                 |
| `struct S { ... }`, `S { a: 1 }`                   | a table `S` with `S.__index = S`; `setmetatable({ a = 1 }, S)`, fields in the order written                                            | —                                 |
| `impl S { fun m(self) }`, trait impls              | `function S.m(self)`. A trait is a name for the checker and needs no Lua                                                               | —                                 |
| `obj.m(x)` where `m` is a method                   | `obj:m(x)`                                                                                                                             | receiver type                     |
| `obj.f(x)` where `f` is a field holding a function | `obj.f(x)`                                                                                                                             | receiver type                     |
| `S.make(x)`, a static method                       | `S.make(x)`                                                                                                                            | —                                 |
| `[a, b]`, `a[i]`, `a[i] = v`, `@len(a)`            | `krt.array({a, b}, 2)`, `krt.get(a, i)`, `krt.set(a, i, v)`, `a.n`. Item 0 is stored at `[1]`, and the count is kept in `n`            | —                                 |
| natives                                            | a Lua function, a short support function, or a compile error where the target has no such thing ([Part 5])                             | —                                 |
| a runtime error                                    | `error(message)` with Kirby's message ([Part 8])                                                                                       | —                                 |

Two rows deserve a word. The **file-as-a-table** rule costs a table lookup for each
use of a top-level name, and it is what keeps a large file from breaking Lua's
limits ([A.7]). A file's table is also what the host calls into: the file ends
with `return M`, and `M.onUpdate(dt)` is the Lua form of the Embedded Library Proposal's
`krbCallGlobal` ([Part 9]). **Private fields** are not enforced by any of this. Kirby
checks `pub` only when the program runs, and the checker does not
([A.6]). [Q-privacy] asks what to do.

### Goals and Non Goals

What this proposal covers:

- Turning a Kirby program that type checks into Lua source for a chosen Lua.
- A support library and a runner that make the output behave like Kirby's.
- Running the whole existing test suite through the output, to know how close it
  is.
- Calling into the output from a host, and host functions the output can call.

The following is intentionally left out of scope for this proposal:

- **Turning Lua into Kirby,** or any other direction.
- **Running Kirby's virtual machine inside Lua.**
- **Sandboxing.** A host that runs Lua decides what the Lua may touch.
- **Beautiful Lua.** Readable output is welcome, but the rules above win when they
  disagree with it.
- **Matching the sequence of `@rand`.** Kirby's numbers and Lua's cannot match.
- **Matching Kirby's 63-call limit.** Kirby stops at 63 nested calls. Lua runs to
  more than 10,000 on 5.1 and LuaJIT and more than 190,000 on 5.4 ([A.7]). A program
  that depends on the limit is not translated faithfully, and none of the test
  programs does ([A.8]).
- **Debugging the output as Kirby.** A line map is a question ([Q-source-map]),
  not a promise.

### Implementation Plan

Each part follows the project's test-first rule. Every part but the first adds
a new option to `krb` or a new file, and none changes what `krb` does when the
option is not used. The parts are ordered so that stopping after any of them
leaves something that runs.

How big is this? `compiler.c`, which walks the same tree and writes bytecode, is
1,340 lines, and the type checker is 2,261. A writer for the same 34 kinds of
node is likely to be of the same order as `compiler.c`, plus a support library
and a test runner. That is a guess, not a measurement.

#### Part 1: Let the checker say what an expression's type is

The writer needs the type of an operand to choose between `+` and `..`, and the
type of a receiver to choose between `obj:m()` and `obj.f()`
([Q-types]). The checker computes these types and forgets them. Proposed: record
them in a side table keyed by node, in the way `resolved_impl_targets.c` already
records impl targets, filled at the point the checker settles each
expression's type and cleared with the tree. Nothing about checking changes.

The other way is a `Type *type` field on `AstNode`. The [Tooling Data Proposal]
measured a node at 160 bytes and a pointer field as growing it to 168.

Test first: a unit test checks `let a = 1 + 2; let b = "x" + "y";` and asks the
table for the type of each `+`. It should answer `f64` for the first and `string`
for the second.

#### Part 2: The smallest writer

A new file, `src/lua_emit.c`, and a long option, `krb --lua file.krb`, that
writes Lua to stdout. It handles what a program needs without structs or
arrays: numbers, strings, booleans and `nil`; variables and assignment; the
operators; functions, lambdas and closures; `if`, `while` and `for`; `break` and
`continue`; `return`; and `print`. It also does the two lowerings that are not
line for line: `if` and blocks used as values, and evaluation order
([Q-order]).

Test first: `tests/lua/fib.krb` and `tests/lua/fizzbuzz.krb`, and for each a
snapshot of the Lua that was written (`.lua`), next to the `.out` and `.exit`
the project already keeps. This is the stop point after which simple programs
run.

#### Part 3: The support library

`kirby_rt.lua` holds what the output calls: the `%` and `/` helpers, number
formatting for `print` and for `@numberToString` (`%f` and `%.15g`), the array
functions that keep a count, and the error function. It ships as a file, and the
binary carries a copy, so that `krb --lua --with-runtime` can paste it into the
output for a single-file result. The helpers have to give the same answers on
every target, and there is one place for that to be tested.

Test first: a Lua file, `lua/test_rt.lua`, run under each Lua, that checks the
helpers against answers Kirby gives: `-5 % 3`, `1/3` as text, an array holding
`nil`, and the six-decimal `print`.

#### Part 4: Structs, impls and traits

`struct`, `impl`, `Self`, static methods, methods called with `self`, function
fields, and `==` on structs through `impl Eq`. This needs the receiver types of
[Part 1]. A call whose receiver's type is not known (an item taken out of an
`Array`, for one) goes through a support function that does what Kirby's VM does:
look for a field first and a method second.

Test first: `examples/stringBuilder.krb`, whose expected Lua is in [Proposed
Changes], and a test for each of the two call forms.

#### Part 5: Arrays, strings and natives

Arrays and strings, and the 63 natives, group by group. Whether an array index is
checked is [Q-array-checks]. Each native ends up as
one of three things: a Lua function that does the same (`@sqrt` is `math.sqrt`),
a short support function (`@round` and `@trunc`, since Lua has neither; the array
natives, which have to keep the count; and string natives that must search for
plain text and not treat the text as a Lua pattern), or an error at translation
time saying the target has no such thing.

The groups are the ones the [Embedded Library Proposal] proposes for its `libs`
setting (`BASE`, `MATH`, `STRING`, `ARRAY`, `IO`, `OS`). The same names can
choose what a translation may use: `--libs base,math,string,array` refuses a
program that calls `@readFileToString`, at translation time. `IO` and `OS` are
the natives a host's Lua often removes, so they are the ones that can be
unavailable ([Q-natives]).

Test first: one test per group that calls each native and prints the result,
run under all three Luas and compared with Kirby.

#### Part 6: Run the whole suite through Lua

The guard for everything above. A script, `scripts/tests-lua.sh`, takes every
`tests/**/*.krb`, writes its Lua, runs it under each Lua that is installed, and
compares stdout and the exit code with the snapshot. A test whose expected
output is a compile error needs no Lua at all: it never reaches the writer,
because the same front end stops it. For the rest, a skip file,
`tests/lua-skip.txt`, lists each test that cannot match, with a reason.

The length of that file is the honest measure of how faithful the translation is.
46 of the 703 programs are known to belong on it ([A.8]): 20 use file or stdin
natives, 21 use environment, process or clock natives, and 6 expect a
private-field error. Some overlap, which is why the parts do not add up to 46.

Test first: the script itself, run against `fib` and `fizzbuzz` before any other
part exists.

#### Part 7: More than one Lua

`--lua-target 5.1|jit|5.4`, with 5.1 as the default and the floor ([Q-target]).
What changes between them:

| Difference                                          | 5.1 (floor)                        | LuaJIT, 5.4                                                                |
| --------------------------------------------------- | ---------------------------------- | -------------------------------------------------------------------------- |
| `continue`                                          | a flag and `repeat ... until true` | `goto` ([A.3])                                                             |
| the upvalue limit for a closure                     | 60                                 | 60 (LuaJIT), 255 (5.4)                                                     |
| whole numbers from `math.floor`, `#` and `tonumber` | plain numbers                      | on 5.4, converted back to floats so they cannot wrap around ([A.1], row G) |
| `unpack`                                            | not used (Kirby has no varargs)    | not used                                                                   |

Luau, the Lua variant Roblox uses, is a possible further target. It has not been
looked at for this proposal.

Test first: the `continue` and `break` test of [A.3] under both forms.

#### Part 8: Errors, exit codes and line numbers

**Errors.** A failure in Kirby's rules (`1 / 0`, an index out of range, a failed
native) is `error(message)` in Lua, with Kirby's message. A small runner,
`krt.run(main)`, calls the program under `pcall`, prints the message and a
trace to stderr, and exits with 70, so that a run from the command line looks
the way it does in Kirby. A host that calls into the output does not use the
runner. It catches the error itself.

**Lines.** One way to make Lua's own line numbers mean something is to put each
statement on the same line as in the Kirby source. Lua does not mind where the
statements are laid out. It was tried by hand: a script whose error is on
line 4 reports `aligned.lua:4:` and the same message on all three Luas
([A.10]). Other ways are a table that maps Lua lines to Kirby lines, or
rewriting Lua's traceback the way the TypeScriptToLua project does with its
`sourceMapTraceback` option. See [Q-source-map].

Test first: a test whose Kirby error is on a known line, which must report that
line from the Lua.

#### Part 9: Calling into the output, and host functions

**Calling in.** A translated file ends with `return M`, the table that holds its
top-level names. A host in Lua does `local game = require("enemies")` and calls
`game.onUpdate(dt)`. This is the Lua form of the [Embedded Library Proposal]'s
`krbCallGlobal`. Which names appear in `M` is [Q-exports].

**Calling out.** A host function such as `@spawn` is written as a call to a
table the host provides: `host.spawn(...)`. For the checker to know its type,
the transpiler needs the same signature the [Embedded Library Proposal] proposes for a C
host ([Embedded Lbrary Part 6]). If those become declaration files ([Q-signatures]),
one file describes the host's functions to both, and to an editor.

Test first: a Lua test that `require`s a translated file, calls a function in it
with a stand-in `host` table, and checks the result.

## Impacts

### Existing Syntax Or Behavior

- **No change to the language,** and no change to `krb` unless `--lua` is used.
- **The type checker records types** ([Part 1]). That costs memory while a program
  is being checked, about a table entry per expression. It changes no answer.
- **New files:** `src/lua_emit.c`, `lua/kirby_rt.lua`, `scripts/tests-lua.sh`,
  `tests/lua-skip.txt`, and a `tests/lua/` folder of snapshots.
- **A new way to run tests,** which needs Lua installed. The script skips a Lua
  that is not there and says so, and the existing suite does not need it.
- **Docs.** A new `docs/LUA.md`, an entry in `docs/CLI.md`, and entries in
  `docs/CHANGELOG.md`.

### Related Proposals

None of these proposals is changed by this one. Each row says how it touches the
translation.

- [Embedded Library Proposal] — the other route to the same hosts. They share the list of
  library groups ([Part 5]) and the host function signatures ([Part 9]). If
  signatures become declaration files ([Q-signatures]), one file serves
  both.
- [Modules Proposal] — a translated module is a Lua table, and an import is a
  `require`. What a module exports decides what `M` holds ([Q-exports]).
- [Top-Level Declarations Proposal] — a file made only of declarations is what
  fits the file-as-a-table rule. A top-level statement would have to run when
  the file is loaded, so this proposal would be easier if that one lands. `main`
  is a call at the end of the runner.
- [Tooling Data Proposal] — its spans are what a line map would be built from
  ([Q-source-map]).
- [Debugger Proposal] — debugging the Lua is the Lua host's business. A line map
  would let a Lua debugger show Kirby lines.
- [Testing Proposal] — `krb test` and the run of the suite through Lua are
  separate, and they can share a runner.
- [Sized Number Types Proposal] — if `u8`, `i32` and `i64` stay plain doubles at
  run time and differ only for the checker, nothing changes here. If a narrowing
  needs the value to wrap, that is arithmetic in Lua, and `i64` cannot be held
  exactly by Lua 5.1 or LuaJIT ([Q-numbers]).
- [Enums Proposal], [Tuples Proposal], [Tuple Structs Proposal], [Destructuring
  Proposal], [Pattern Matching Proposal] — each has an open question about its
  run-time form. Whatever it settles on, the Lua form is a table and, for
  `match`, a chain of `if`s. The writer should work from the checked and lowered
  tree, so each of these adds a lowering and not a new kind of output.
- [Generic Types Proposal] — specialization makes each use its own compiled code,
  so the writer sees ordinary functions and structs and never a type parameter.
- [Macros Proposal] — macros are expanded before the writer runs.
- [String Interpolation Proposal] — its lowered code calls `StringBuilder` in the
  stdlib, which is Kirby code. A program that uses interpolation carries a
  translated copy of it.
- [Primitive Impls Proposal] and [Collection Methods Proposal] — a method on a
  number, string or array is a call to a support function.
- [Prefixed Native Functions Proposal] (Closed) and [Additional Native Functions
  Proposal] (Closed) — the natives that [Part 5] maps.

### Testing Plan

How do we know the implemented proposal works?

**The existing suite is the measure.** [Part 6] runs every test through Lua. A
test passes when its stdout and exit code equal the snapshot's. The size of the
skip file, and the reason on each line of it, says how close the translation is.
The 280 tests that end in a compile error need no Lua, because they never reach
the writer. Of the rest, 46 programs are already known not to translate
faithfully ([A.8]).

#### E2E Tests

##### NEW: tests/lua/continue_and_break.krb

```kirby
var out = 0;
var log = "";
for (var i = 0; i < 12; i = i + 1) {
  if (i % 2 == 0) { continue; }
  if (i > 7) { break; }
  out = out + i;
  log = log + @numberToString(i) + ",";
}
print out;
print log;
var fs = [];
var j = 0;
while (j < 3) { @arrPush(fs, fun (): f64 { j }); j = j + 1; }
print fs[0]();
print fs[2]();
```

###### Expected Outcome

Kirby prints `16.000000`, `1,3,5,7,`, `3.000000` and `3.000000`. So must the Lua,
on every target, with both forms of `continue` ([A.3]).

##### NEW: tests/lua/array_holding_nil.krb

```kirby
let a: Array = [nil, 1, nil];
print @len(a);
```

###### Expected Outcome

`3.000000`. A Lua table written the plain way gives 0 on Lua 5.1 and 2 on the
others for the same three items ([A.1], row E).

##### NEW: tests/lua/remainder_and_division.krb

```kirby
print -5 % 3;
print 1 / 0;
```

###### Expected Outcome

`-2.000000` on stdout, then a runtime error with the message
`function / expects argument 1 to be a non-zero number but got 0.` and exit code
70 ([A.1], rows A and B).

#### Unit Tests

| File                | Part | Checks                                                                                                      |
| ------------------- | ---- | ----------------------------------------------------------------------------------------------------------- |
| `unit/expr_types.c` | 1    | The recorded type of `+`, of a receiver, and of an `impl Eq` comparison.                                    |
| `lua/test_rt.lua`   | 3    | The support library against answers Kirby gives, on each Lua.                                               |
| `unit/lua_emit.c`   | 2, 4 | The Lua text for small programs, including both forms of `continue`, `if` as a value, and evaluation order. |

## Questions

### **Q:** Is turning Kirby into Lua feasible?

<!-- [Q-feasible]: #q-is-turning-kirby-into-lua-feasible -->

**Status:** Answered

That is the question this proposal was written to answer.

#### Answer

Yes, for the language as it is today. What was checked:

- Two example programs, a loop with `break` and `continue`, and a program that
  makes closures in a loop, were translated by hand and matched Kirby's output on
  three versions of Lua ([A.2], [A.3]).
- Every difference found between the languages has a rule that removes it
  ([A.1]), and the two that Lua's own limits cause (200 locals, 60 upvalues) are
  avoided by putting a file's names in a table ([A.7]).
- The tree the writer would walk has 34 kinds of node, and a walk of the same
  size already exists ([A.5]).

What is not answered, and would be the first things to learn by building it:
how well the rules hold on the whole test suite ([Part 6] measures this), what
the extra table lookups cost on real programs, and how much the proposals in
flight (enums, patterns, generics, macros) add. Whether it is worth doing
is for the project to decide, and this document is meant to inform that.

### **Q:** Which Lua first?

<!-- [Q-target]: #q-which-lua-first -->

**Status:** Open

- **(a) A floor of Lua 5.1 that also runs on LuaJIT and Lua 5.4** (proposed).
  It costs a slower `continue` (a flag, [A.3]), and it works on every Lua that
  was tried. Faster forms are options for a target ([Part 7]). Others have made
  the same choice. TypeScriptToLua's default target, `universal`, is code that
  works on every Lua target it supports. The OpenMW game engine runs Lua 5.1 with
  a few 5.2 extensions and says it will not move to a newer Lua because LuaJIT
  does not support one ([A.9]).
- **(b) Lua 5.4 first.** It has `goto`, and a larger upvalue limit (255). It
  leaves out the engines that embed Lua 5.1 or LuaJIT.
- **(c) LuaJIT only.** It runs fastest ([A.4]) and it has `goto`.
- **(d) Luau as well.** A candidate, not tried here.

### **Q:** How does the writer learn types?

<!-- [Q-types]: #q-how-does-the-writer-learn-types -->

**Status:** Open

- **(a) A side table that the checker fills** (proposed). It is how impl targets
  are kept today. It changes no node.
- **(b) A `Type *` on every node.** It is simpler to read, and it makes every node
  8 bytes bigger (160 to 168 in the [Tooling Data Proposal]'s measurement).
- **(c) The writer works out types again.** It duplicates the checker.

### **Q:** What happens to `pub`?

<!-- [Q-privacy]: #q-what-happens-to-pub -->

**Status:** Open

Kirby checks that a field is `pub` when the program runs, and not when it is
compiled ([A.6]). A plain Lua table cannot hold a private field.

- **(a) Move the check into the type checker.** It finds the mistake earlier for
  every user of Kirby, and then there is nothing left for Lua to check. It is a
  change to Kirby in its own right, and a program that today fails at run time
  would fail at compile time.
- **(b) Emit a check** on each access to a field that is not `pub`. It keeps
  Kirby's behavior, and it is slower and it is bigger.
- **(c) Do not enforce it in Lua.** A program that is correct behaves the same. A
  program that reads a private field works in Lua and stops in Kirby.

### **Q:** How are arrays written, and are they checked?

<!-- [Q-array-checks]: #q-how-are-arrays-written-and-are-they-checked -->

**Status:** Open

The proposal keeps the count in `n` and stores item 0 at `[1]`, so that `nil`
items survive. Whether the index is checked is open.

- **(a) Always check** (proposed for the first version). It matches Kirby, whose
  `a[5]` on a one-item array is a runtime error ([A.10]). It costs a call.
- **(b) A release mode without checks.** Faster, and different from Kirby for a
  program that has this bug.
- **(c) Store item 0 at `[0]`.** No `+ 1`, but Lua keeps index 0 in the slower
  part of a table.

### **Q:** How are Lua errors tied back to Kirby lines?

<!-- [Q-source-map]: #q-how-are-lua-errors-tied-back-to-kirby-lines -->

**Status:** Open

- **(a) Keep each statement on its Kirby line** ([A.10]). There is nothing to
  ship and nothing to look up. A statement that the writer turns into several
  (an `if` used as a value, or a hoisted temporary) has to sit on one line, and a
  Kirby statement over several lines is reported at one of them.
- **(b) Ship a table** from Lua lines to Kirby lines, and a function that
  translates a message. It is exact, and it is a second file that has to stay in
  step.
- **(c) Rewrite the traceback,** as the TypeScriptToLua project does with its
  `sourceMapTraceback` option, which overrides `debug.traceback` so that error
  messages point at the original source ([A.9]). It reuses (b), and it changes
  `debug.traceback`, which a host may not want.

The [Tooling Data Proposal]'s spans are the source for (b) and (c).

### **Q:** Which natives are available?

<!-- [Q-natives]: #q-which-natives-are-available -->

**Status:** Open

The natives in `IO` and `OS` read files, the environment and stdin. A host's Lua
often has no `io` or `os`.

- **(a) Refuse at translation time** what the target library groups do not have
  (proposed). It uses the same groups as the [Embedded Library Proposal].
- **(b) Emit a call that fails when it runs.** Simpler, and it finds the mistake
  too late.
- **`@rand` and `@clock`.** A sequence of random numbers cannot match Kirby's.
  `@clock` is CPU time in Kirby, and Lua has `os.clock` where `os` exists.

### **Q:** What does a translated file export?

<!-- [Q-exports]: #q-what-does-a-translated-file-export -->

**Status:** Open

- **(a) Every top-level function and struct** (proposed for now). It is what a host
  like the one in the Embedded Library Proposal calls into.
- **(b) Only what is marked** `pub`, once top-level `pub` exists in the [Modules
  Proposal].
- **(c) Only `main`.** Right for a command line program, and wrong for a game.

### **Q:** What if numbers stop being plain doubles?

<!-- [Q-numbers]: #q-what-if-numbers-stop-being-plain-doubles -->

**Status:** Open

Today every Kirby number is a `double`, which is what a Lua 5.1 or LuaJIT number
is. If the [Sized Number Types Proposal] keeps that, nothing changes. If `i64`
must wrap around or hold more than 2^53, then Lua 5.1 and LuaJIT cannot do it
exactly, and Lua 5.4's integers can. That would make a translation of `i64` code
depend on the target, or refuse it on the floor.

### **Q:** When does the writer name a temporary?

<!-- [Q-order]: #q-when-does-the-writer-name-a-temporary -->

**Status:** Open

Kirby evaluates left to right, and Lua's manual does not say ([A.9]).

- **(a) Only when both operands could change something** (proposed). The writer
  looks for calls and assignments. Most expressions need nothing.
- **(b) Always.** Safe, and the output is full of temporaries.
- **(c) Trust Lua.** It is left to right in practice and unpromised.

## Appendix A: Checking the claims

Everything here was run at `from_commit` (`662d98b`) on Linux x86-64. Timings come
from one machine, so they show an order of magnitude and not a benchmark.

### Setup

```shell
# Kirby: build krb as in the Setup of the Embedded Library Proposal's Appendix A.
# krb must be run from a checkout, because it opens stdlib/stdlib.krb.
K=/path/to/kirbylang
KRB=/path/to/krb

# Lua, three implementations
sudo apt-get install lua5.1 luajit lua5.4

mkdir -p /tmp/lua-appx && cd /tmp/lua-appx     # the files below are saved here
```

Versions used:

```text
Lua 5.1.5  Copyright (C) 1994-2012 Lua.org, PUC-Rio
LuaJIT 2.1.1703358377 -- Copyright (C) 2005-2023 Mike Pall. https://luajit.org/
Lua 5.4.6  Copyright (C) 1994-2023 Lua.org, PUC-Rio
```

The support library that the Lua below uses, `krt.lua`, is a hand-written stand-in for the `kirby_rt.lua` of [Part 3]:

```lua
-- Runtime support that generated code would rely on (a hand-written stand-in for kirby_rt.lua).
local krt = {}
local fmt, concat = string.format, table.concat

-- Kirby's `%` is C's fmod.
krt.fmod = math.fmod

-- Kirby's `/` stops the program on a zero divisor. Lua returns inf/nan.
function krt.div(a, b)
  if b == 0 then error("function / expects argument 1 to be a non-zero number but got 0.", 2) end
  return a / b
end

-- @numberToString uses %.15g. Lua's tostring is %.14g (5.1, LuaJIT) or adds ".0" (5.3+).
function krt.numstr(n) return fmt("%.15g", n) end

-- Arrays keep an explicit length so that nil elements survive; index 0 is stored at [1].
function krt.array(items, n) items.n = n or #items; return items end
function krt.arrPush(a, v) a.n = a.n + 1; a[a.n] = v end
function krt.arrJoin(a, sep)
  for i = 1, a.n do
    if type(a[i]) ~= "string" then error("function arrJoin expects every element of argument 1 to be a string but element " .. (i - 1) .. " is a " .. type(a[i]) .. ".") end
  end
  return concat(a, sep, 1, a.n)
end
function krt.get(a, i)
  if i < 0 or i >= a.n or i % 1 ~= 0 then error("Array index out of bounds.", 2) end
  return a[i + 1]
end

-- print: numbers use %f, arrays print their items, instances print "Name instance".
local function show(v)
  local t = type(v)
  if t == "number" then return fmt("%f", v)
  elseif t == "table" and v.n and getmetatable(v) == nil then
    local parts = {}
    for i = 1, v.n do parts[i] = show(v[i]) end
    return "[" .. concat(parts, ", ") .. "]"
  elseif t == "table" then return getmetatable(v).__name .. " instance"
  else return tostring(v) end
end
krt.show = show
function krt.print(v) io.write(show(v), "\n") end
return krt
```

### A.1 Where Kirby and Lua differ

Claims: the table in [Problem Statement], and the rule that fixes each row.

Each case is a Kirby program, the output Kirby gives, and then Lua written two ways: the obvious way and the way the rules of [Proposed Changes] give. `5.1`, `JIT` and `5.4` are the three Luas. `krt` is `krt.lua` above. The script that ran them is `cases.py`, below.

```python
import subprocess

K = "/path/to/kirbylang"
KRB = "/path/to/krb"
LUAS = ["lua5.1", "luajit", "lua5.4"]

def kirby(source):
    open("c.krb", "w").write(source + "\n")
    r = subprocess.run([KRB, "-f", "/tmp/lua-appx/c.krb"], cwd=K, capture_output=True, text=True)
    return (r.stdout + r.stderr).strip().replace("\n", " | "), r.returncode

def lua(binary, source):
    open("c.lua", "w").write("local krt = require('krt')\n" + source + "\n")
    r = subprocess.run([binary, "c.lua"], capture_output=True, text=True)
    return (r.stdout + r.stderr).strip().replace("\n", " | ").replace(binary + ": ", "")

CASES = [
    ("A: remainder", "print -5 % 3;",
     [("obvious: -5 % 3", "print(-5 % 3)"),
      ("rule: math.fmod(-5, 3)", "print(math.fmod(-5, 3))")]),
    ("B: divide by zero", "print 1 / 0;",
     [("obvious: 1 / 0", "print(1 / 0)"),
      ("rule: krt.div(1, 0)", "print(pcall(krt.div, 1, 0))")]),
    ("C: printing a number", "print 3;",
     [("obvious: print(3.0)", "print(3.0)"),
      ("rule: krt.print(3.0)", "krt.print(3.0)")]),
    ("D: a number as text", "print @numberToString(1 / 3);",
     [("obvious: tostring(1/3)", "print(tostring(1/3))"),
      ("rule: krt.numstr(1/3)", "print(krt.numstr(1/3))")]),
    ("E: an array holding nil", "let a: Array = [nil, 1, nil]; print @len(a);",
     [("obvious: #{nil, 1, nil}", "print(#{nil, 1, nil})"),
      ("obvious: #{1, nil, nil}", "print(#{1, nil, nil})"),
      ("rule: {nil, 1, nil, n = 3}.n", "print(({nil, 1, nil, n = 3}).n)")]),
    ("F: closures made in a for loop",
     "let fs = []; for (var i = 0; i < 3; i = i + 1) { @arrPush(fs, fun (): f64 { i }); } print fs[0](); print fs[1](); print fs[2]();",
     [("obvious: numeric for",
       "local fs = {}; for i = 0, 2 do fs[#fs+1] = function() return i end end print(fs[1](), fs[2](), fs[3]())"),
      ("rule: one variable, while",
       "local fs = {}; local i = 0; while i < 3 do fs[#fs+1] = function() return i end; i = i + 1 end print(fs[1](), fs[2](), fs[3]())")]),
    ("G: whole numbers past 2^62", "print @floor(@pow(2, 62)) * 4;",
     [("obvious: math.floor(2^62) * 4", "print(math.floor(2^62) * 4)"),
      ("rule: ... * 4.0, printed with %f", "print(string.format('%f', math.floor(2^62) * 4.0))")]),
    ("H: continue (compares two forms of Lua only)", None,
     [("goto continue (5.2 and later)",
       "for i = 1, 2 do if i == 1 then goto continue end print(i) ::continue:: end"),
      ("flag form", "for i = 1, 2 do repeat if i == 1 then break end print(i) until true end")]),
    ("I: calling a function stored in a field",
     "struct S { pub var f: fun (f64) => f64; } let s = S { f: fun (x: f64): f64 { x + 1 } }; print s.f(1);",
     [("obvious, method style: s:f(1)",
       "local s = {f = function(x) return x + 1 end} print(pcall(function() return s:f(1) end))"),
      ("rule, field style: s.f(1)", "local s = {f = function(x) return x + 1 end} print(s.f(1))")]),
    ("J: joining strings", 'print "a" + "b";',
     [('obvious: "a" + "b"', 'print(pcall(function() return "a" + "b" end))'),
      ('rule: "a" .. "b"', 'print("a" .. "b")')]),
]

for name, kirby_source, variants in CASES:
    print("=== " + name)
    if kirby_source is not None:
        output, code = kirby(kirby_source)
        print("Kirby program: " + kirby_source)
        print("Kirby says:    " + output + (" [exit %d]" % code if code else ""))
    for label, lua_source in variants:
        results = [lua(binary, lua_source)[:70] for binary in LUAS]
        print("  %-38s 5.1: %s" % (label, results[0]))
        print("  %-38s JIT: %s" % ("", results[1]))
        print("  %-38s 5.4: %s" % ("", results[2]))
```

Saved as `cases.py`, with `K` and `KRB` set to the checkout and the `krb` binary, and run from `/tmp/lua-appx`. Output:

```text
=== A: remainder
Kirby program: print -5 % 3;
Kirby says:    -2.000000
  obvious: -5 % 3                        5.1: 1
                                         JIT: 1
                                         5.4: 1
  rule: math.fmod(-5, 3)                 5.1: -2
                                         JIT: -2
                                         5.4: -2
=== B: divide by zero
Kirby program: print 1 / 0;
Kirby says:    function / expects argument 1 to be a non-zero number but got 0. | [line 1] in script [exit 70]
  obvious: 1 / 0                         5.1: inf
                                         JIT: inf
                                         5.4: inf
  rule: krt.div(1, 0)                    5.1: false	function / expects argument 1 to be a non-zero number but got 0.
                                         JIT: false	function / expects argument 1 to be a non-zero number but got 0.
                                         5.4: false	function / expects argument 1 to be a non-zero number but got 0.
=== C: printing a number
Kirby program: print 3;
Kirby says:    3.000000
  obvious: print(3.0)                    5.1: 3
                                         JIT: 3
                                         5.4: 3.0
  rule: krt.print(3.0)                   5.1: 3.000000
                                         JIT: 3.000000
                                         5.4: 3.000000
=== D: a number as text
Kirby program: print @numberToString(1 / 3);
Kirby says:    0.333333333333333
  obvious: tostring(1/3)                 5.1: 0.33333333333333
                                         JIT: 0.33333333333333
                                         5.4: 0.33333333333333
  rule: krt.numstr(1/3)                  5.1: 0.333333333333333
                                         JIT: 0.333333333333333
                                         5.4: 0.333333333333333
=== E: an array holding nil
Kirby program: let a: Array = [nil, 1, nil]; print @len(a);
Kirby says:    3.000000
  obvious: #{nil, 1, nil}                5.1: 0
                                         JIT: 2
                                         5.4: 2
  obvious: #{1, nil, nil}                5.1: 1
                                         JIT: 1
                                         5.4: 1
  rule: {nil, 1, nil, n = 3}.n           5.1: 3
                                         JIT: 3
                                         5.4: 3
=== F: closures made in a for loop
Kirby program: let fs = []; for (var i = 0; i < 3; i = i + 1) { @arrPush(fs, fun (): f64 { i }); } print fs[0](); print fs[1](); print fs[2]();
Kirby says:    3.000000 | 3.000000 | 3.000000
  obvious: numeric for                   5.1: 0	1	2
                                         JIT: 0	1	2
                                         5.4: 0	1	2
  rule: one variable, while              5.1: 3	3	3
                                         JIT: 3	3	3
                                         5.4: 3	3	3
=== G: whole numbers past 2^62
Kirby program: print @floor(@pow(2, 62)) * 4;
Kirby says:    18446744073709551616.000000
  obvious: math.floor(2^62) * 4          5.1: 1.844674407371e+19
                                         JIT: 1.844674407371e+19
                                         5.4: 0
  rule: ... * 4.0, printed with %f       5.1: 18446744073709551616.000000
                                         JIT: 18446744073709551616.000000
                                         5.4: 18446744073709551616.000000
=== H: continue (compares two forms of Lua only)
  goto continue (5.2 and later)          5.1: c.lua:2: '=' expected near 'continue'
                                         JIT: 2
                                         5.4: 2
  flag form                              5.1: 2
                                         JIT: 2
                                         5.4: 2
=== I: calling a function stored in a field
Kirby program: struct S { pub var f: fun (f64) => f64; } let s = S { f: fun (x: f64): f64 { x + 1 } }; print s.f(1);
Kirby says:    2.000000
  obvious, method style: s:f(1)          5.1: false	c.lua:2: attempt to perform arithmetic on local 'x' (a table val
                                         JIT: false	c.lua:2: attempt to perform arithmetic on local 'x' (a table val
                                         5.4: false	c.lua:2: attempt to perform arithmetic on a table value (local '
  rule, field style: s.f(1)              5.1: 2
                                         JIT: 2
                                         5.4: 2
=== J: joining strings
Kirby program: print "a" + "b";
Kirby says:    ab
  obvious: "a" + "b"                     5.1: false	c.lua:2: attempt to perform arithmetic on a string value
                                         JIT: false	c.lua:2: attempt to perform arithmetic on a string value
                                         5.4: false	c.lua:2: attempt to add a 'string' with a 'string'
  rule: "a" .. "b"                       5.1: ab
                                         JIT: ab
                                         5.4: ab
```

Notes on the rows. In row E the array literal `[nil, 1, nil]` without a type is a
type error in Kirby (`Expected unit, got f64.`), so the program gives its array the
type `Array`. In row I the method-style call fails because `s:f(1)` passes `s` as
the first argument, so `x` is the table. Row G matters on Lua 5.4 only:
`math.floor` returns a whole-number type there, and the multiplication wraps
around.

### A.2 Two programs translated by hand

Claims: `stringBuilder.krb` and `fizzbuzz.krb` translate with the rules and match Kirby on all three Luas.

`sb.lua` is the Lua shown in [Proposed Changes]. `fb.lua` is `fizzbuzz.krb` with the `limit` fixed at 15. `sb.krb` is `examples/stringBuilder.krb`, and `fb.krb` is `examples/fizzbuzz.krb` with `var limit = 15;` in place of the prompt.

```lua
local krt = require("krt")

-- struct StringBuilder { var value: Array; }
local StringBuilder = { __name = "StringBuilder" }
StringBuilder.__index = StringBuilder

-- impl StringBuilder { pub fun add(self, add: string): Self }
function StringBuilder.add(self, add)
  krt.arrPush(self.value, add)
  return self
end

-- impl Default for StringBuilder { fun default(): Self }
function StringBuilder.default()
  return setmetatable({ value = krt.array({}) }, StringBuilder)
end

-- impl Display for StringBuilder { fun toString(self): string }
function StringBuilder.toString(self)
  return krt.arrJoin(self.value, "")
end

local builder = StringBuilder.default():add("Hello"):add(" "):add("World")
krt.print(builder:toString())
```

```lua
local krt = require("krt")
local fmod = krt.fmod

-- fun fizzbuzz(n: f64): string = if (...) ... else ...;   (an `if` expression in tail position becomes returns)
local function fizzbuzz(n)
  if fmod(n, 5) == 0 and fmod(n, 3) == 0 then return "FizzBuzz"
  elseif fmod(n, 3) == 0 then return "Fizz"
  elseif fmod(n, 5) == 0 then return "Buzz"
  else return krt.numstr(n) end
end

local limit = 15
-- for (var i = 1; i <= limit; i = i + 1) { ... }   (one shared variable, like Kirby)
do
  local i = 1
  while i <= limit do
    krt.print(fizzbuzz(i))
    i = i + 1
  end
end
```

```text
sb.krb  Kirby: Hello World
sb.lua  lua5.1: Hello World
sb.lua  luajit: Hello World
sb.lua  lua5.4: Hello World
fb.krb  Kirby: 1 | 2 | Fizz | 4 | Buzz | Fizz | 7 | 8 | Fizz | Buzz | 11 | Fizz | 13 | 14 | FizzBuzz
fb.lua  lua5.1: 1 | 2 | Fizz | 4 | Buzz | Fizz | 7 | 8 | Fizz | Buzz | 11 | Fizz | 13 | 14 | FizzBuzz
fb.lua  luajit: 1 | 2 | Fizz | 4 | Buzz | Fizz | 7 | 8 | Fizz | Buzz | 11 | Fizz | 13 | 14 | FizzBuzz
fb.lua  lua5.4: 1 | 2 | Fizz | 4 | Buzz | Fizz | 7 | 8 | Fizz | Buzz | 11 | Fizz | 13 | 14 | FizzBuzz
```

### A.3 `continue` and `break`

Claims: the flag form works on all three Luas; `goto` does not work on Lua 5.1; closures made in a loop share one variable.

```kirby
var out = 0;
var log = "";
for (var i = 0; i < 12; i = i + 1) {
  if (i % 2 == 0) { continue; }
  if (i > 7) { break; }
  out = out + i;
  log = log + @numberToString(i) + ",";
}
print out;
print log;
var fs = [];
var j = 0;
while (j < 3) { @arrPush(fs, fun (): f64 { j }); j = j + 1; }
print fs[0]();
print fs[2]();
```

```lua
local krt = require("krt")
local fmod = krt.fmod
local out, log = 0, ""
do
  local i = 0
  while i < 12 do
    do  -- the body gets its own block so `goto` never jumps into a local's scope
      if fmod(i, 2) == 0 then goto continue end
      if i > 7 then goto break_ end
      out = out + i
      log = log .. krt.numstr(i) .. ","
    end
    ::continue::
    i = i + 1
  end
  ::break_::
end
krt.print(out); krt.print(log)
local fs = krt.array({})
local j = 0
while j < 3 do krt.arrPush(fs, function() return j end); j = j + 1 end
krt.print(krt.get(fs, 0)()); krt.print(krt.get(fs, 2)())
```

```lua
local krt = require("krt")
local fmod = krt.fmod
local out, log = 0, ""
do
  local i = 0
  while i < 12 do
    local broke = false           -- set when the body executes `break`
    repeat                        -- `break` inside this repeat means "continue"
      if fmod(i, 2) == 0 then break end
      if i > 7 then broke = true; break end
      out = out + i
      log = log .. krt.numstr(i) .. ","
    until true
    if broke then break end
    i = i + 1
  end
end
krt.print(out); krt.print(log)
local fs = krt.array({})
local j = 0
while j < 3 do krt.arrPush(fs, function() return j end); j = j + 1 end
krt.print(krt.get(fs, 0)()); krt.print(krt.get(fs, 2)())
```

```text
Kirby: 16.000000 | 1,3,5,7, | 3.000000 | 3.000000
lua5.1 goto: cb_goto.lua:8: '=' expected near 'continue'
lua5.1 flag: 16.000000 | 1,3,5,7, | 3.000000 | 3.000000
luajit goto: 16.000000 | 1,3,5,7, | 3.000000 | 3.000000
luajit flag: 16.000000 | 1,3,5,7, | 3.000000 | 3.000000
lua5.4 goto: 16.000000 | 1,3,5,7, | 3.000000 | 3.000000
lua5.4 flag: 16.000000 | 1,3,5,7, | 3.000000 | 3.000000
```

Both Lua files put the body of the loop in its own block, so that `goto` never jumps into the scope of a local variable.

### A.4 Speed

Claim: the translated benchmark ran faster than Kirby's VM in every run: 1.1 to 1.4 times as fast on Lua 5.1, 1.2 to 1.6 times on Lua 5.4 and 2.7 to 4.0 times on LuaJIT, across three sets of runs.

The program is the one from the [Debugger Proposal]'s Appendix A (`bench.krb`: a 3-million-trip `while` loop and `fib(30)`). `bench.lua` is that program written by hand with the rules of [Proposed Changes]: whole numbers written as floats, `%` as `math.fmod`.

```lua
local krt = require("krt")
local fmod = krt.fmod
-- Numbers are written as floats (2.0, not 2) so that Lua 5.4 stays in float arithmetic like Kirby.
local function fib(n)
  if n < 2.0 then return n end
  return fib(n - 2.0) + fib(n - 1.0)
end
local i, sum = 0.0, 0.0
while i < 3000000.0 do
  sum = sum + fmod(i, 7.0)
  i = i + 1.0
end
krt.print(sum)
krt.print(fib(30.0))
```

Median of 15 runs after 2 warm-up runs, including start-up:

```text
output   Kirby: 8999994.000000 832040.000000  | lua5.1: 8999994.000000 832040.000000   luajit: 8999994.000000 832040.000000   lua5.4: 8999994.000000 832040.000000

Kirby VM (Release, -O3 -DNDEBUG):  0.207 s
lua5.1   hand-translated:          0.143 s   (1.4x faster than Kirby's VM)
lua5.4   hand-translated:          0.131 s   (1.6x faster than Kirby's VM)
luajit   hand-translated:          0.052 s   (4.0x faster than Kirby's VM)
```

The set above was the third. The first two, made on the same machine, gave:

| Set | Kirby's VM | Lua 5.1        | Lua 5.4        | LuaJIT         |
| --- | ---------- | -------------- | -------------- | -------------- |
| 1   | 0.253 s    | 0.208 s (1.2x) | 0.164 s (1.5x) | 0.073 s (3.5x) |
| 2   | 0.168 s    | 0.149 s (1.1x) | 0.140 s (1.2x) | 0.062 s (2.7x) |

Timings on this machine drift by that much between sets of runs, so the claim is
the range they span. Only the ordering is firm: LuaJIT is fastest, and Kirby's VM
is slowest.

The output of all four is identical. The Kirby binary is the `-O3 -DNDEBUG` build described in A.7 of the Embedded Library Proposal's appendix, run from a folder with `stdlib/`. These are hand-written translations, not the output of a program.

### A.5 What the front end records, and how Kirby behaves

Claims: 34 kinds of node; nodes carry no types; a side table records impl targets; `+` is decided at run time; a method call looks for a field first; the size of the compiler; strings are bytes; conditions must be `bool`.

```shell
cd $K/src
sed -n '/typedef enum {/,/} NodeType;/p' ast.h | grep -o 'NODE_[A-Z_]*' | sort -u | grep -v NODE_COUNT | wc -l
grep -n "Type \*" ast.h                       # a type on a node? (no output means none)
cat resolved_impl_targets.h | grep -v '^#\|^$'
sed -n '/case OP_ADD: {/,/} else {/p' vm.c
sed -n '/^static bool invoke(/,/vm.stackTop\[-argCount - 1\]/p' vm.c
wc -l compiler.c typecheck.c parser.c
```

```text
node kinds: 34
types on AstNode: 0

void resolvedImplTargetsReset(void);
void resolvedImplTargetsRecord(AstNode *implNode, InternedName structName);
const Token *resolvedImplTargetsLookup(AstNode *implNode);

    case OP_ADD: {
      Value operand_b = peekStack(0);
      Value operand_a = peekStack(1);

      if (IS_STRING(operand_b) && IS_STRING(operand_a)) {
        concatenate();
      } else if (IS_NUMBER(operand_b) && IS_NUMBER(operand_a)) {
        double b = AS_NUMBER(popFromStack());
        double a = AS_NUMBER(popFromStack());

        Value result = NUMBER_VAL(a + b);

        pushOnStack(result);
      } else {

static bool invoke(ObjString *name, int argCount) {
  Value caller = peekStack(argCount);

  if (IS_STRUCT(caller)) {
    return invokeStatic(AS_STRUCT(caller), name, argCount);
  }

  if (!IS_INSTANCE(caller)) {
    runtimeError(&vm, "Only instances have methods.");
    return false;
  }

  ObjInstance *instance = AS_INSTANCE(caller);

  int slot;

  if (structFieldSlot(instance->struct_, name, &slot)) {

  1340 compiler.c
  2261 typecheck.c
  1615 parser.c
  5216 total
```

`invoke` finds the receiver, then looks for a field of that name first and a method
second, so the same `obj.name(x)` means a field or a method depending on what the
receiver has at run time. In a typed program the checker knows which.

The last three claims, run through `krb`:

```text
$ krb -f  print @len("é");
2.000000

$ krb -f  if (0) { print 1; } else { print 2; }
[line 1] Error: Expected bool, got f64. [exit 65]

$ krb -f  print !true;
false

$ lua5.1 -e 'print(#"é")'
2
```

### A.6 Field privacy is checked when the program runs

Claim: a private field read from outside the struct compiles, and stops the program at run time.

```text
$ krb -f  struct P { var secret: f64; pub var open: f64; } let p = P { secret: 1, open: 2 }; print p.open;
Field 'secret' is private to 'P'. | [line 1] in script [exit 70]

$ krb -f  struct P { pub var open: f64; var secret: f64; } impl P { pub fun make(): P = P { open: 1, secret: 2 }; } let p = P.make(); print p.secret;
Field 'secret' is private to 'P'. | [line 1] in script [exit 70]
```

Both programs pass the type checker. The failure is `Field 'secret' is private to 'P'.` from the VM. The first program fails even though it only reads `open`, because the struct literal names `secret` from outside.

### A.7 Lua's limits

Claims: 200 locals per function on all three; 60 upvalues on Lua 5.1 and LuaJIT and 255 on 5.4; recursion depth.

```python
import subprocess

def lua(binary, source):
    open("limits_case.lua", "w").write(source)
    r = subprocess.run([binary, "limits_case.lua"], capture_output=True, text=True)
    return (r.stdout + r.stderr).strip().replace("\n", " | ")[:100]

def locals_source(n):
    return "\n".join("local v%d = %d" % (i, i) for i in range(n)) + "\nprint('ok')"

def upvalues_source(n):
    # an outer function with up to 190 locals, an inner closure that uses n of them
    # (for n > 190 a middle function holds the rest, so no function has more than 200 locals)
    head = min(n, 190)
    src = "local function outer()\n" + "\n".join("  local u%d = %d" % (i, i) for i in range(head)) + "\n"
    if n > 190:
        rest = n - 190
        src += "  local function mid()\n" + "\n".join("    local w%d = %d" % (i, i) for i in range(rest)) + "\n"
        names = ["u%d" % i for i in range(190)] + ["w%d" % i for i in range(rest)]
        src += "    return function() return " + " + ".join(names) + " end\n  end\n  return mid()\nend\n"
    else:
        src += "  return function() return " + " + ".join("u%d" % i for i in range(n)) + " end\nend\n"
    return src + "print('ok', outer()())"

def recursion_source(depth):
    return "local function f(n) if n == 0 then return 0 end return 1 + f(n - 1) end\nprint(pcall(f, %d))" % depth

for binary in ["lua5.1", "luajit", "lua5.4"]:
    print("==", binary)
    for n in (200, 201):
        print("  %3d locals in one function:      %s" % (n, lua(binary, locals_source(n))))
    for n in (60, 61, 255, 256):
        print("  %3d upvalues used by a closure:  %s" % (n, lua(binary, upvalues_source(n))))
    for d in (10000, 100000, 190000, 1000000):
        print("  recursion %7d deep:          %s" % (d, lua(binary, recursion_source(d))))
```

```text
== lua5.1
  200 locals in one function:      ok
  201 locals in one function:      lua5.1: limits_case.lua:201: main function has more than 200 local variables
   60 upvalues used by a closure:  ok	1770
   61 upvalues used by a closure:  lua5.1: limits_case.lua:63: function at line 63 has more than 60 upvalues
  255 upvalues used by a closure:  lua5.1: limits_case.lua:258: function at line 192 has more than 60 upvalues
  256 upvalues used by a closure:  lua5.1: limits_case.lua:259: function at line 192 has more than 60 upvalues
  recursion   10000 deep:          true	10000
  recursion  100000 deep:          false	limits_case.lua:1: stack overflow
  recursion  190000 deep:          false	limits_case.lua:1: stack overflow
  recursion 1000000 deep:          false	limits_case.lua:1: stack overflow
== luajit
  200 locals in one function:      ok
  201 locals in one function:      luajit: limits_case.lua:201: main function has more than 200 local variables
   60 upvalues used by a closure:  ok	1770
   61 upvalues used by a closure:  luajit: limits_case.lua:63: function at line 63 has more than 60 upvalues
  255 upvalues used by a closure:  luajit: limits_case.lua:258: function at line 192 has more than 60 upvalues
  256 upvalues used by a closure:  luajit: limits_case.lua:259: function at line 192 has more than 60 upvalues
  recursion   10000 deep:          true	10000
  recursion  100000 deep:          false	limits_case.lua:1: stack overflow
  recursion  190000 deep:          false	limits_case.lua:1: stack overflow
  recursion 1000000 deep:          false	limits_case.lua:1: stack overflow
== lua5.4
  200 locals in one function:      ok
  201 locals in one function:      lua5.4: limits_case.lua:201: too many local variables (limit is 200) in main function near '='
   60 upvalues used by a closure:  ok	1770
   61 upvalues used by a closure:  ok	1830
  255 upvalues used by a closure:  ok	20035
  256 upvalues used by a closure:  lua5.4: limits_case.lua:259: too many upvalues (limit is 255) in function at line 259 near 'end'
  recursion   10000 deep:          true	10000
  recursion  100000 deep:          true	100000
  recursion  190000 deep:          true	190000
  recursion 1000000 deep:          false	limits_case.lua:1: stack overflow
```

A closure's upvalues are the variables it uses from outside itself. The upvalue
test uses a 190-variable outer function and, past that, a middle function, so no
function has more than 200 locals. Kirby stops at 63 nested calls (A.8 of the
Embedded Library Proposal's appendix shows where).

### A.8 Which test programs would not translate

Claim: 46 of the 703 programs use a native that a host's Lua may not have or expect a private-field error; none expects a stack overflow.

Run from the root of the Kirby checkout:

```python
import glob, os, re
root = "tests"
programs = glob.glob(root + "/**/*.krb", recursive=True)
IO = r"@(readFileToString|writeStringToFile|fileExists|prompt|stdin)\b"
OS = r"@(getenv|setenv|exit|argv|argc|clock)\b"
n_io = n_os = n_stack = n_private = n_any = 0
exits = {}
for path in programs:
    source = open(path).read()
    err = open(path + ".err").read() if os.path.exists(path + ".err") else ""
    code = open(path + ".exit").read().strip() if os.path.exists(path + ".exit") else "0"
    exits[code] = exits.get(code, 0) + 1
    io, os_ = bool(re.search(IO, source)), bool(re.search(OS, source))
    stack, private = "Stack overflow" in err, "is private to" in err
    n_io += io; n_os += os_; n_stack += stack; n_private += private
    n_any += io or os_ or stack or private
print("test programs:", len(programs))
print("exit codes:", dict(sorted(exits.items())))
print("use an IO native:", n_io, "  use an OS native:", n_os)
print("expect a stack overflow:", n_stack, "  expect a private-field error:", n_private)
print("at least one of those:", n_any)
```

```text
test programs: 703
exit codes: {'0': 272, '65': 280, '7': 1, '70': 150}
use an IO native: 20   use an OS native: 21
expect a stack overflow: 0   expect a private-field error: 6
at least one of those: 46
```

The programs that expect exit code 65 end in a compile error and never reach a
translator. The counts of natives are by name in the source, so a program that
uses two groups is counted in both.

### A.9 Other people's work

These come from pages read for this document, and are not commands to run.

- **Lua's order of evaluation.** In 2001, a Lua author replied on the lua-l mailing
  list that code in Lua is in general generated from left to right, that the
  manual does not say anything about the order of evaluation, and that a program
  should not depend on it ([lua-l, 2001]). In 2016 another replied that it is
  safe to assume the order is unspecified even where the manual does not say so
  ([lua-l, 2016]).
- **TypeScriptToLua** turns TypeScript into Lua. Its options include `luaTarget`,
  with the values `"JIT"`, `"5.3"`, `"5.2"`, `"5.1"` and `"universal"` (in the
  versions of the page that were read, and `"5.0"` in one), where `universal` is
  described as generating code compatible with all supported Lua targets, and
  `sourceMapTraceback`, described as overriding Lua's `debug.traceback` to apply
  source maps to stack traces so that error messages point at the original source
  ([TypeScriptToLua docs]).
- **Teal** is a typed dialect of Lua that compiles to Lua and works with Lua 5.1
  to 5.4 and LuaJIT ([Teal]). It describes existing Lua libraries with
  declaration files, `.d.tl` ([Teal types]).
- **OpenMW** is a game engine. Its scripting documentation says it supports Lua
  5.1 with some 5.2 extensions and has no plans to move to a newer Lua, because
  newer versions are not supported by LuaJIT. It documents a Teal workflow that
  needs the declaration files for the OpenMW API ([OpenMW's Teal page]).

### A.10 Keeping Lua's line numbers equal to Kirby's

Claim: if each statement is written on the line it has in the Kirby source, Lua reports the same line as Kirby.

```kirby
var a = [1];
var b = 2;

print a[5];
```

```lua
local krt = require('krt'); local a = krt.array({1.0}, 1)
local b = 2.0

krt.print(krt.get(a, 5.0))
```

```text
Kirby: Array index out of bounds. | [line 4] in script
lua5.1: aligned.lua:4: Array index out of bounds.
luajit: aligned.lua:4: Array index out of bounds.
lua5.4: aligned.lua:4: Array index out of bounds.
```

The array has one item, so `a[5]` is out of range. The first Lua line holds both the `require` and the first statement, because Lua does not mind. The blank line 3 is a blank line in the Kirby source. All three Luas report `aligned.lua:4:` with the message that Kirby gives.

## Glossary

These are both technical and non technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Writer**: The part of `krb` that would turn a checked syntax tree into Lua
  source. (Other documents call this a transpiler or an emitter.)
- **Target**: The version of Lua that the output is meant to run on
- **Floor**: A target that the other targets can also run. The output written
  for the floor works everywhere
- **Support library**: The Lua file, `kirby_rt.lua`, that the output calls for
  things Lua does not do the way Kirby does, such as printing a number
- **Metatable**: A Lua table that tells Lua how another table behaves, for
  example where to look for a method when the table does not have one
- **Upvalue**: A variable that a Lua function uses from outside itself. Lua
  limits how many one function may use
- **LuaJIT**: A fast implementation of Lua 5.1 with some additions
- **Integer (Lua 5.4)**: A second kind of number in Lua 5.3 and later. It wraps
  around when it gets too big, where a float grows
- **Lowering**: Rewriting a construct into simpler ones that the target has, such
  as writing an `if` that is used as a value as statements that store into a
  variable
- **Temporary**: A named variable the writer adds to hold a result, so that the
  order of evaluation is fixed
- **Snapshot**: The saved expected output of a test, kept next to the test and
  compared with on each run

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Problem Statement]: #problem-statement
[Proposed Changes]: #proposed-changes
[Appendix A]: #appendix-a-checking-the-claims

<!-- Parts of the implementation plan -->

[Part 1]: #part-1-let-the-checker-say-what-an-expressions-type-is
[Part 2]: #part-2-the-smallest-writer
[Part 3]: #part-3-the-support-library
[Part 4]: #part-4-structs-impls-and-traits
[Part 5]: #part-5-arrays-strings-and-natives
[Part 6]: #part-6-run-the-whole-suite-through-lua
[Part 7]: #part-7-more-than-one-lua
[Part 8]: #part-8-errors-exit-codes-and-line-numbers
[Part 9]: #part-9-calling-into-the-output-and-host-functions

<!-- Appendix A -->

[A.1]: #a1-where-kirby-and-lua-differ
[A.2]: #a2-two-programs-translated-by-hand
[A.3]: #a3-continue-and-break
[A.4]: #a4-speed
[A.5]: #a5-what-the-front-end-records-and-how-kirby-behaves
[A.6]: #a6-field-privacy-is-checked-when-the-program-runs
[A.7]: #a7-luas-limits
[A.8]: #a8-which-test-programs-would-not-translate
[A.9]: #a9-other-peoples-work
[A.10]: #a10-keeping-luas-line-numbers-equal-to-kirbys

<!-- Questions -->

[Q-feasible]: #q-is-turning-kirby-into-lua-feasible
[Q-target]: #q-which-lua-first
[Q-types]: #q-how-does-the-writer-learn-types
[Q-privacy]: #q-what-happens-to-pub
[Q-array-checks]: #q-how-are-arrays-written-and-are-they-checked
[Q-source-map]: #q-how-are-lua-errors-tied-back-to-kirby-lines
[Q-natives]: #q-which-natives-are-available
[Q-exports]: #q-what-does-a-translated-file-export
[Q-numbers]: #q-what-if-numbers-stop-being-plain-doubles
[Q-order]: #q-when-does-the-writer-name-a-temporary

<!-- Proposals -->

[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Testing Proposal]: ../testing/PROPOSAL.md
[Sized Number Types Proposal]: ../sized-number-types/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Tuple Structs Proposal]: ../tuple-structs/PROPOSAL.md
[Destructuring Proposal]: ../destructuring/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Prefixed Native Functions Proposal]: ../prefixed-native-functions/PROPOSAL.md
[Additional Native Functions Proposal]: ../additional-native-functions/PROPOSAL.md

<!-- Other proposals' parts and questions -->

[Embedded Lbrary Part 6]: ../embedded-library/PROPOSAL.md#part-6-host-functions-the-checker-knows-about
[Q-signatures]: ../embedded-library/PROPOSAL.md#q-how-are-host-function-types-written

<!-- External pages -->

[lua-l, 2001]: https://lua-users.org/lists/lua-l/2001-02/msg00139.html
[lua-l, 2016]: https://lua-users.org/lists/lua-l/2016-07/msg00528.html
[TypeScriptToLua docs]: https://github.com/TypeScriptToLua/TypeScriptToLua.github.io
[Teal]: https://github.com/teal-language/tl
[Teal types]: https://github.com/teal-language/teal-types
[OpenMW's Teal page]: https://openmw.readthedocs.io/en/latest/reference/lua-scripting/teal.html
