---
status: Draft
created: 2026-09-19
from_commit: 662d98b
---

# Proposal: Additional Data For Tooling Support

Tools need to know two things about a program that Kirby does not fully record
today: where each piece of code came from, and what its variables are called. An
editor needs the first to underline an error or jump to a definition. The
debugger needs both to show a file, a line, and a variable name. Macros will
need the first to point generated code back at the source that caused it.

This proposal adds that data. Every piece of syntax gets a [Span]: its exact
place in the source, stored as byte positions and as line and column. Generated
syntax can point back at what produced it. The compiled output gains a file
name, a way to get from any instruction back to its place in the source, and the
names of local and captured variables. Nothing about what a program does
changes.

It replaces the earlier Span Tracking proposal and takes over the "debug
information" part of the [Debugger Proposal], so that the data is described in
one place. The title and the folder name are working titles. The choices that
are still open are listed as [Questions].

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

Good tooling needs to answer, for any piece of a program, "where did this come
from?" and "what is this called?" Kirby answers both only in part, and what it
does answer is not always dependable.

### What Kirby has today

Checked at `from_commit` (see [Appendix A]):

- **Tokens.** A `Token` (`src/token.h`) holds a type, a pointer into the source
  text, a length, and a line. Because it points into the text, its byte position
  in the file can be worked out. It has no column. For a string that runs over
  several lines, `line` is the line where the string _ends_.
- **AST nodes.** Every `AstNode` (`src/ast.h`) has one `line`. Some kinds of
  node also keep `endLine`, `declEndLine`, or `bodyEndLine`. Nodes that hold a
  name or an operator keep its `Token`, which points into the text.
- **Lines on instructions.** The compiler keeps one global, `currentLine`.
  `compileExpr` sets it from `node->line`, other places set it by hand from the
  `endLine` fields, and every byte emitted takes whatever value it has at that
  moment (`emitByte` in `src/compiler.c`).
- **Lines in the compiled output.** A `CompiledFn` keeps one `int` for every
  byte of bytecode (`codeLines` in `src/compiled_unit.h`). The loader copies it
  into `Chunk.lines`, and runtime error traces read it.
- **Names.** `ObjFunction.name` holds a function's name (a lambda gets a
  generated one, such as `lambda0x1`). `vm.globals` is keyed by global name.

### What is missing

1. **A position is a line and nothing else.** There is no column and no end, so
   an error about an expression can point at a line, not at the code itself.
2. **The line is not always the line you would expect.** An instruction takes
   whichever line the compiler looked at last. For a call written over several
   lines:

   ```kirby
   var v = boom(   // line 6
     1,            // line 7
     2             // line 8
   );              // line 9
   ```

   the error trace says `[line 8] in script`, which is the line of the last
   argument. A string that runs over lines 1 to 3 puts its first instruction on
   line 3, so a breakpoint on line 1 can never be hit as written. Python, Node,
   Lua, Java, and C all put such a call on its first line ([Appendix B]).

3. **No file.** A `CompiledUnit` does not say which file it came from. `krb -f`
   runs two units in one process: `stdlib/stdlib.krb`, then the program.
4. **No variable names in the compiled output.** The compiler knows each local's
   name, but holds it as a `Token` that points into the source text, and
   `runFile` in `src/main.c` frees that text as soon as compiling ends. What is
   left in the bytecode is slot numbers, such as `OP_GET_LOCAL 1`. The names of
   captured variables are lost the same way. The array meant to describe a
   function's captured variables (`CompiledFn.upvalues`) is never filled in
   today: `cuAddUpvalue` exists, but nothing calls it.
5. **No source text once the program runs.** Anything a tool wants to show has
   to be worked out while compiling, or the text has to be kept. This is why
   this proposal stores line and column up front instead of working them out
   later ([Q-repr]).
6. **No idea of generated code.** This is not a problem today because Kirby
   cannot generate code, but the [Macros Proposal] changes that. Once code can be
   generated, the code that runs is not always the code the programmer typed. A
   method may exist only because a derive macro produced it, and a type error
   may occur inside generated syntax. If tools describe generated code by its
   own location, the programmer sees errors pointing at code they never wrote.
   This is the most common complaint about macro systems in practice.

### Who needs what

| Who                                                   | Needs                                                                                                                                              |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Compile errors                                        | A span on every node, so an error can underline the exact code                                                                                     |
| Runtime error traces                                  | The file, the function name, and where the running instruction came from                                                                           |
| [Debugger Proposal]                                   | A file for each function, a line for every instruction, local variable names with the range of code each is alive for, and captured variable names |
| Editors (go to definition, find references, complete) | A span on every node. These tools read source, not compiled output, so they need only the AST side                                                 |
| [Macros Proposal]                                     | Generated syntax that points back at the macro call, and a way to mark a variable the macro added                                                  |
| [Generic Types Proposal]                              | Each specialization of a generic keeps the generic's file and positions                                                                            |

## Proposed Changes

### Goals and non-goals

The first version should:

- Give every AST node a span with byte offsets, line, and column for both its
  start and its end.
- Let an instruction be traced back to a span, and a function to a file.
- Save what the debugger needs: local variable names with the range of code each
  is alive for, and captured variable names.
- Change nothing about what a program does, and nothing about its bytecode.

Left out of the first version. Each can be added later without redoing the parts
below:

- **Types of variables** in the compiled output. `CompiledUnit` stores no types
  today, and that belongs with the [Modules Proposal].
- **A symbol index** for go to definition and find references. It needs names
  resolved across files, and would be built on the spans here.
- **Comments and blank lines**, which a formatter would need.
- **Where a variable was declared.** A local variable record could hold the span
  of its declaration. The debugger does not need it, so it is not included.
- **Changing the text of errors or traces.** Their format stays. The only change
  they can see is which line is shown ([Q-line]).

### Part 1 — The span

A span records a place in the source precisely enough for tooling:

```c
typedef struct {
  int fileId;
  int startOffset; // Position of the first byte
  int endOffset;   // Position just after the last byte
  int startLine;
  int startColumn;
  int endLine;
  int endColumn;
} Span;
```

- **Offsets** count bytes from the start of the file, exactly as `readFile`
  reads it. The end is one past the last byte, so `endOffset - startOffset` is
  the length. For the REPL and `-c`, the text is the line or argument given.
- **Line and column** both start at 1, as `line` does today. A column counts
  bytes from the start of the line, so a tab is one and a character outside
  ASCII is several. An editor adapter converts to whatever unit its editor uses.
  It has the line's text; the VM does not.
- **The end** is the position just after the last byte, in both forms. That is,
  `endLine` and `endColumn` say where `endOffset` falls.
- **Both forms are stored**, even though one can be worked out from the other.
  The reasons are in [Q-repr]. A unit test checks that they agree with the
  source text.
- **`fileId`** says which file the span is in. What it indexes is [Part 5] and
  [Q-files].

**Where the numbers come from.** The scanner already counts lines. It will also
remember where the current line started, so a column is the token's start minus
the line's start, plus one. `Token` gets a `column`, and its `line`
becomes the line where the token _starts_. Today `line` is where it ends. The
two differ only for a string that runs over several lines, and no test has one.
A token's end is worked out from its text when a node's span is built: only a
string can contain a newline, so for every other token the end is on the same
line, one length further along. `Token` stays 24 bytes: the new field fits once
its fields are reordered ([Appendix B]).

### Part 2 — A span on every AST node

`AstNode` swaps its `line` for a `Span`. The parser sets it as it builds the
node. The start comes from the node's first token, and the end from the last
token the parser consumed. An infix node such as `a + b` takes its start from
its left operand. There are 34 kinds of node and 41 places in `src/parser.c`
that create one, so this is done one kind at a time, each with a test
([Part 8]).

The tokens a node already keeps (an operator, a name) stay. A tool can still get
the position of just the operator, or just the name in a call.

The scattered `endLine`, `declEndLine`, and `bodyEndLine` fields say what the
span's end now says. They can be removed once the compiler reads the span, but
nothing needs them removed first.

**Cost.** `AstNode` goes from 136 to 160 bytes: 24 more, or 18%. The AST lives
in an arena that `compileSource` frees after every compile (`astFreeAll`), so
this is memory used only while compiling. The tests average one node for every 8
bytes of source, so a 100 KB program would use about 300 KB more while it
compiles. See [Q-cost].

### Part 3 — Spans that survive generation

The essential requirement: when a macro (or any future step that produces
syntax) generates syntax, the generated syntax carries a span that points back
to the source the programmer actually wrote, which is the macro call. Through
nested macros it points back through each layer.

This means a span is not only "a range in a file". It can also carry an
**origin**: a link to the span of the call or construct that produced it. A tool
or error message can then always turn "this happened in generated code" into
"here is the source line responsible." How origin is chained through nested
expansion is [Q-origin]. The 160 bytes above do not include an origin. A field
on every node adds 8 more, whether it is a 4-byte index or a pointer
([Appendix B]).

**Build it in from the start.** Span tracking has to be part of the AST from the
beginning. Adding "where did this come from" to structures that were not built
to carry it is a well-known source of pain, because every place that creates or
rewrites syntax has to be revisited. So the generics and macro work should set
spans as they create and rewrite syntax, and not leave that to be added later.

### Part 4 — Where an instruction came from

Part 2 puts a span on every node. This part is about what survives into the
compiled output, and how each instruction gets its location.

**Each instruction gets the span of the code that produced it.** The compiler
stops using one global `currentLine`. Each emit is handed the span of the node
being compiled. An instruction that closes a construct (the pops at the end of a
block, a function's implicit return) gets the span of the closing token, so it
stays on the `}` line, as it does today. The line of an instruction is its
span's start line ([Q-line]).

**Readers ask one function.** Today the error trace reads `chunk.lines` directly
in `runtimeError`, and the debugger will need the same answer. Instead, a single
function answers "where did the instruction at this bytecode position come
from?", and everything that wants a location calls it. How the answer is stored
can then change without touching readers. That is [Q-compiled].

**The loader always fills in `Chunk.lines`**, as it does today, because error
traces need a line on every run. Anything beyond a line is Part 7.

### Part 5 — Source names

**The path of the file.** `CompiledUnit` gets a list of the files its code came
from, stored in the unit's string blob like function names are. Today the list
has one entry. `parse()`, `compile()`, and `compileSource()` take a source name;
the REPL and `-c` pass `<repl>` and `<code>`. For a file, the name is the path
`krb` was given ([Q-paths]): `runFile` already has it, and passes it on. The
loader puts an interned copy of the name on each `ObjFunction` as `sourcePath`,
and the garbage collector marks it next to `name`.

**A `fileId` belongs to its unit.** It is an index into the list of the unit it
sits in, and has no meaning outside it. This matters because `krb -f` loads two
units into one VM, and both have a file 0. The VM does not depend on the
compiler (see `interpret()` in `src/vm.h`), so the list has to travel inside the
unit and cannot stay in the compiler. What the list holds is [Q-paths]. How
files are told apart once programs have several is [Q-files].

`parse` has about twenty callers in `unit/`. Giving it a source name means
updating them, or adding a second function that takes the name.

### Part 6 — Local and captured variable names

For each function, one record per named local:

```diff
 typedef struct {
   bool isLocal;
   uint8_t index;
+  int nameOffset; // Offset in to the compiled unit's string blob
+  int nameLength;
 } CompiledUpvalue;
+
+/**
+ * A named local variable of a compiled function. Tooling data only.
+ */
+typedef struct {
+  int nameOffset; // Offset in to the compiled unit's string blob
+  int nameLength;
+  uint8_t slot;   // The operand OP_GET_LOCAL uses for this variable
+  int startPc;    // First bytecode position where the variable has a value
+  int endPc;      // First bytecode position after the variable is gone
+} CompiledLocal;
```

and on `CompiledFn`:

```diff
   CompiledUpvalue *upvalues;
   int upvalueDescCount;
   int upvalueDescCapacity;
+
+  // Function Local Variables (tooling data)
+
+  CompiledLocal *locals;
+  int localCount;
+  int localCapacity;
```

The compiler already knows each of these facts. It only throws them away:

- **The name** is copied into the string blob when the local is declared
  (`addLocal` in `src/compiler.c`), before the source text is freed.
- **The slot** is the local's index in `FnCompiler.locals`, the same number
  `resolveLocal` puts in `OP_GET_LOCAL`. The record must always hold the number
  the compiler put in the instruction. See the limit below.
- **`startPc`** is the first position where the slot really holds the variable's
  value. For a `var`, that is the position after its initializer, where
  `defineVariable` runs. For a parameter, it is 0. For a local `fun` it is
  _after_ the `OP_CLOSURE` that builds it, including its captured-variable
  bytes. Today `markInitialized` runs for a local `fun` before its `OP_CLOSURE`
  is emitted, so it cannot be used as it is. In the example in [Appendix A],
  `helper` is marked at position 2 but only has a value from position 4.
- **`endPc`** is the position after the instruction that removes the variable:
  after its `OP_POP` or `OP_CLOSE_UPVALUE` in `captureOrCleanLocalsGoingOutOfScope`,
  or after the `OP_CLOSE_BLOCK_EXPR` in `compileBlockExprClose`. Each local ends
  after its own pop, so at a block's closing `}` the inner locals are already
  gone. `emitPopsToDepth` (used by `break` and `continue`) does not stop
  tracking anything. Parameters and other locals of a function's outermost scope
  last until the end of the function, and `endCompiler` closes them.
- **Slot 0** has an empty name in a plain function and `self` in a method. Empty
  names are not recorded.

A record has a range because slots are reused: two blocks can put different
variables in the same slot at different times.

**Captured variables.** `CompiledFn.upvalues` exists but is never filled in.
The compiler will start filling it, calling `cuAddUpvalue` from `addUpvalue` in
step with its own list, so that entry `i` describes closure upvalue `i`.
`resolveUpvalue` has the name at hand, so the name can be stored as it goes in.
With it, a closure's captured variables can be listed by name.

**Generated variables.** A local that a macro adds has a name the programmer
never wrote. The record is expected to gain a way to say so, either a flag or
the origin from Part 3. The [Macros Proposal] asks for it. Nothing needs it
until macros exist, and its form waits on [Q-origin].

**A known limit.** A block used as an expression gets the wrong slot numbers
when values are already on the stack under it. `var r = 1 + 2 * { var b = 5; b +
a };` gives 25 instead of 31, because the compiler numbers `b` as if nothing
were on the stack ([Appendix A]). That is a bug in the compiler and is not part
of this proposal. Until it is fixed the recorded slot matches the wrong number
the instruction uses, so a debugger would show the wrong value for such a
variable. Once the compiler numbers slots correctly, the records follow.

### Part 7 — What reaches the running VM

A normal run needs a line for each instruction and a function name. It does not
need the rest. The loader copies the extra data onto the function objects only
when a tool asked for it: the file name, the variable records, the captured
variable names, and anything in the location data beyond a line. Debugging is
the first such tool, and `krb --debug` turns it on before anything is loaded
(see the [Debugger Proposal]).

Without it, a normal run keeps nothing extra once the unit is freed after
loading. It still pays for producing the data while compiling. See [Q-strip].

### Part 8 — Testing

The repo uses snapshot tests, and this work follows the same TDD loop: write one
failing test, see it fail, make it pass. Nothing here is visible to a running
program, so most of the tests are C unit tests in `unit/`, in the style of
`unit/parser.c`. One `unit/` test already calls `compile()`.

- **Spans on nodes.** Parse a snippet, find a node, and check that the text
  between its start and end offsets is what it should be, and that its line and
  column match. The line and column are checked against a count made by the test
  itself, from the source text, so they cannot silently drift from the offsets.
  Each of the 34 node kinds gets its own test.
- **Tokens.** A lexer test for start line and column, including a string over
  several lines.
- **Compiled data.** Compile a snippet and check the local variable records, the
  captured variable names, and the source name in the unit. The local function,
  block expression, and reused-slot cases above each get a test.
- **`print_ast` does not change.** Its output is what the existing parser tests
  compare against. A test that needs to see spans reads the node, not the
  printed form.
- **End-to-end snapshots.** `just test` must pass with no snapshot updates until
  the step that changes which line an instruction has ([Q-line], Part 9 step 6).
  That step has its own snapshot updates, each looked at before it is committed.

### Part 9 — Suggested order

Each step is small, starts with a failing test, and leaves what a program does
unchanged:

1. The scanner gives every token a start line and a column.
2. The `Span` type, and spans on nodes one kind at a time, starting with
   literals, variables, and binary expressions.
3. Source names through `parse`, `compile`, and the loader, ending in
   `sourcePath`. The debugger needs this for its step 2.
4. Local variable records and captured variable names. The debugger needs this
   for its step 3.
5. The single function that answers "where did this instruction come from?", and
   then the storage chosen for [Q-compiled].
6. The change to which line an instruction has ([Q-line]). It is a step of
   its own.
7. The loader flag from Part 7.

Origin (Part 3) waits for macros.

## Impacts

### Existing Syntax Or Behavior

- **No language change.** No syntax, opcode, or bytecode changes. A program does
  the same thing.
- **`Token`.** Gets a `column`. Its `line` becomes the line where the token
  starts. Fields are reordered, and the size stays 24 bytes. No test has a
  string over several lines, so no snapshot changes because of this.
- **`AstNode`.** `line` becomes `span`, 136 to 160 bytes. `astAlloc` takes a
  span in place of a line, and 41 call sites change. 77 places outside the
  scanner read a `line`, and they read the span's start line instead
  ([Q-line]). See [Q-cost] for the memory.
- **The compiler** stops using `currentLine` ([Q-line]).
  `addLocal`, `markInitialized`, the scope-closing functions, `endCompiler`, and
  `addUpvalue` record the data in Part 6.
- **`parse`, `compile`, and `compileSource`** take a source name. About twenty
  callers in `unit/` change, unless `parse` keeps its form and a second function
  is added.
- **Bigger structures.** `CompiledUnit`, `CompiledFn`, `CompiledUpvalue`, and
  `ObjFunction` gain fields. The garbage collector must mark `sourcePath` the way
  it marks `name` (`src/gc.c`). See [Q-strip] for the cost of recording the data
  at compile time.
- **Snapshots.** Every `.err` snapshot (705 of 705) contains a bytecode listing
  with lines in it. Some of them change ([Q-line]).
  `print_ast` output is deliberately not changed, so the parser tests keep
  passing as they are.
- **Errors and traces** keep their text.

### Related Proposals

- [Debugger Proposal] — reads this data: the file for each function, a line for
  every instruction, local variable names with ranges, and captured variable
  names. Its Part 1 now points here, and [Q-paths] and [Q-strip] moved here from
  it. That proposal is updated to say so.
- [Macros Proposal] — needs the origin in Part 3, which is the same per-syntax
  information as Hygiene, and a way to mark generated variables (Part 6). That
  proposal is updated to say so.
- [Generic Types Proposal] — its Part 8 states the requirement that every piece
  of syntax carries a span. Each specialization must keep the generic's file and
  positions. That proposal is updated to point here.
- [Modules Proposal] — shares how files are named and identified ([Q-files]),
  which turns on whether a module is a file or a name ([Q-naming] there). It
  also has to decide whether shipped compiled code carries this data and what a
  recorded path means on another machine ([Q-paths], [Q-strip]). That proposal is
  updated to say so.
- [String Interpolation Proposal] — its lowered code is generated, so it should
  carry the span of the interpolated string. A string over several lines now has
  a start line, which matters there. That proposal is updated to say so.
- [Projects Proposal] — a project root is the base the recorded source path is
  meant to be relative to in the long run ([Q-paths]). That proposal does not
  say yet how a root is found, which is now a question there ([Q-root]). That
  proposal is updated to say so.
- [Top-Level Declarations Proposal] — also adds a parameter to `compileSource`:
  the kind of source (entry file, library file, or snippet). It is separate from
  the source name added in Part 5, and either proposal can land first. It also
  splits `runFile` into `loadFile` and `runFile`, and both pass the path on.
  That proposal is updated to say so.

## Questions

### **Q:** What is the exact representation of a span?

<!-- [Q-repr]: #q-what-is-the-exact-representation-of-a-span -->

**Status:** Answered

How precisely should a span record a location? Options were (a) line and column
pairs for start and end; (b) byte offsets into the source, from which line and
column are worked out on demand; (c) a hybrid. Offsets are compact and cheap to
carry, but need the source text to become a line and column. Line and column
pairs can be read directly, but are larger.

The source text is freed as soon as compiling ends (`runFile` in `src/main.c`),
so anything the VM or a tool shows at run time has to be worked out before that.
Offsets alone would mean keeping the text, or a table of where each line starts.

The measured size of an `AstNode` for each choice ([Appendix B]):

| Stored in each node                       | Size      | More than today |
| ----------------------------------------- | --------- | --------------- |
| Today: a line                             | 136 bytes | —               |
| File and offsets                          | 144 bytes | +8 (6%)         |
| Offsets and start line and column         | 152 bytes | +16 (12%)       |
| Offsets and start and end line and column | 160 bytes | +24 (18%)       |

#### Answer

Store both, for the start and the end of every span: byte offsets, and line and
column (the last row). The scanner works out the line and column while it scans
(Part 1). Line and column both start at 1, and a column counts bytes.

Offsets are what a tool can rely on, and line and column are what an error
message or a range in an editor needs. Both ends are stored because an editor
draws a range, and Python stores the same for every instruction. The cost is a
little memory while compiling and nothing afterwards ([Q-cost]).

### **Q:** How much does per-node span storage cost, and does it matter?

<!-- [Q-cost]: #q-how-much-does-per-node-span-storage-cost-and-does-it-matter -->

**Status:** Answered

Putting a span on every AST node adds memory to every node. Options were (a)
store a full span inline on every node; (b) store a compact handle on each node
that indexes a side table of spans. (a) is simplest, and (b) is smaller per node
but adds an indirection.

#### Answer

(a), a full span inline. It adds 24 bytes to a 136 byte node. The AST is in an
arena that `compileSource` frees after every compile, so the memory is used only
while compiling. The tests average one node for every 8 bytes of source, which
puts a 100 KB program at about 300 KB extra for the time it takes to compile.
The programs in the tests are small, so the ratio is what matters and not the
totals. It leaves out an origin link, which is [Q-origin].

### **Q:** Which line does an instruction get?

<!-- [Q-line]: #q-which-line-does-an-instruction-get -->

**Status:** Answered

Today an instruction takes the line the compiler looked at last (see the Problem
Statement). The same program was run in five other languages: a call whose `(`
is on the first line, with its arguments and its `)` on the lines after, and a
callee that fails ([Appendix B]).

| Language | The call is put on                                                  |
| -------- | ------------------------------------------------------------------- |
| Python   | The first line. The call instruction spans first line to last       |
| Node     | The first line, at the callee's name                                |
| Lua      | The first line, though the last argument is loaded on the last line |
| Java     | The first line                                                      |
| C (gcc)  | The first line, in the debug line table                             |
| Kirby    | The last argument's line                                            |

For a method call spread over lines (`o` / `.a()` / `.b()`), Python and Node put
the failing call on the line of that call's method name. Lua puts it on the
first line of the whole chain.

- **(a) The span of the code that produced it.** The compiler hands
  each instruction the span of the node it is compiling, and the line is the
  span's start line. An instruction that closes a construct gets the span of the
  closing token, so it stays on the `}` line. A method call uses the position of
  the method name, as in Python and Node. Stepping through a call written over
  several lines visits its first line, its argument lines, and then its first
  line again, which is what Python does. The cost is snapshots. All 705 `.err`
  snapshots contain a listing, and up to 95 contain an `OP_CLOSURE`, which today
  sits on the closing-brace line of a function declaration and would move to the
  line of the declaration. Multi-line calls, variable declarations, and
  expressions would move too. The number is not measured.
- **(b) Keep today's rules.** No snapshot changes. `AstNode.line` stays next to
  the span (Part 2), and the spans only add columns and files. The debugger's
  line events still work, but a breakpoint on the first line of a string that
  runs over several lines can never be hit as written, and a call written over
  several lines is shown on its last argument's line.

#### Answer

(a): an instruction gets the line where the code that produced it starts. A call
written over several lines is on its first line, and so is a string that runs
over several lines. The rules in (a) for closing tokens and for method calls
stay with it: an instruction that closes a construct is on the line of the
closing token, and a method call is on the line of the method's name.

`AstNode.line` does not stay next to the span. The compiler reads the span's
start line, and `currentLine` goes away (Part 4). The snapshots that change are
their own step (Part 9, step 6), and how many move will be counted there.

### **Q:** How is origin chained through nested macro expansion?

<!-- [Q-origin]: #q-how-is-origin-chained-through-nested-macro-expansion -->

**Status:** Open

When a macro expands into a call to another macro, generated syntax has more
than one layer of "where did this come from."

Options: (a) each generated span points only at its immediate producer, and
tools walk the chain; (b) spans carry a full chain of origins. (a) keeps one
link on each node and leaves the walking to consumers; (b) keeps the whole chain
ready to read. This is the same information Hygiene in the [Macros Proposal]
needs, so the two should be decided together. The same answer decides how a
generated local variable is marked (Part 6).

**How much bigger is (b)?** Not much in memory. More in code.

| Cost                 | (a) A link to the immediate producer                                            | (b) The whole chain                                                                                      |
| -------------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Each node            | +8 bytes: one index or pointer                                                  | +8 bytes if the nodes of one expansion share one chain, +16 with a count next to the pointer             |
| Each expansion       | Nothing new: the link points at the macro call's span, which exists already     | A list of every layer: about 28 bytes a layer if spans are copied, so 84 for three deep                  |
| In the compiled unit | An index in each span row (+4 bytes)                                            | A second table for the chains, and a reference to it in each row                                         |
| Code to write        | One field, one assignment where syntax is made, one loop that follows the links | The same, and a chain type, and a step at each expansion that copies the parent's chain and adds a layer |
| Reading a chain      | Follow the links                                                                | Read the list                                                                                            |

In the AST, (a) can point straight at the span of the macro call. The AST is
freed after compiling (`astFreeAll`), and links inside a `CompiledUnit` are
indexes and string offsets, never pointers, so in the unit the link has to be an
index.

A node grows by the same amount either way, and a chain is small next to the
tree: a 100 KB program has about 12,500 nodes, and 8 bytes on each is 100 KB
while it compiles. The difference is in the code and in the unit: (b) is more to
write, more to get wrong, and needs a second table. The sizes are measured
([Appendix B]). The per-expansion figure is worked out, and the code row is an
estimate from the design, because none of it is built.

A chain of single links holds every fact that a full chain does, so (b) adds no
new information. It only saves the walk. An error message or a debugger stop
walks once. Hygiene might look at the chain for every name it resolves, which is
why this waits on the Macros proposal ([Q-syntax] there). Starting with (a)
closes nothing off: going to (b) later would add storage for a ready-made chain,
but nothing new that has to be recorded.

**Testing before macros exist.** Nothing in Kirby makes generated syntax today.
`src/parser.c` is the only code that creates a node, and nothing else writes
into one (checked at `from_commit`, [Appendix A]). String interpolation, as
proposed, is lowered straight to bytecode, so it makes none either. Until macros
or generics arrive, chains can only be tested with a stand-in, in a C unit test
in `unit/` in the style of `unit/parser.c`:

- **A stand-in expander, written in the test file.** It takes a call node from a
  parsed snippet and creates a few new nodes (a variable, a binary expression, a
  call) whose origin is that call's span. It runs no macro. Handing one of the
  new nodes to a second expansion as its call builds a chain of two, and a third
  builds a chain of three.
- **What the test checks.** A node from the parser has no origin. Following the
  origin from the innermost node ends at the call the test wrote, with the right
  offsets, line, and column, after as many steps as the depth. A node the
  expander was handed and only moved into its output keeps its own origin, since
  it came from where the programmer wrote it. Only the nodes the expander
  creates are marked. If (b) is chosen, the same test also covers the step that
  copies the parent's chain.
- **The size.** The test asserts `sizeof(AstNode)`, so a field added by accident
  is noticed.
- **Afterwards.** The checks stay when the stand-in is replaced by a real macro
  or a generic's specialization. Only the thing that creates the nodes changes.

### **Q:** How are multiple source files identified?

<!-- [Q-files]: #q-how-are-multiple-source-files-identified -->

**Status:** Open

A span must say _which_ file it points into. Today Kirby compiles from a single
source at a time, but `krb -f` already loads two units, and once modules and
multi-file programs exist, spans must tell files apart.

Options:

- **(a) An integer id into a list of source names that travels in each unit**,
  so an id only means something together with its unit (proposed in Part 5).
- **(b) An id that is unique across every unit in the run**, with one shared
  table.
- **(c) The path on every span.** As text, each span would hold its own copy,
  and every node in a file has the same path, so this is the largest by far. As
  a pointer to one shared string nothing is copied, but a pointer is 8 bytes
  where an id is 4, and a node grows from 160 to 168 bytes ([Appendix B]). A
  pointer also cannot go into a compiled unit, where links between records are
  indexes and offsets, so the spans kept in the unit would still need an id.
  That id is what (a) and (b) use, so (c) saves nothing.

The VM does not depend on the compiler, so a table held only by the compiler
cannot be an option. How a name is written is [Q-paths].

Which of (a) and (b) fits depends on whether a module is a file or a name, which
is [Q-naming] in the [Modules Proposal]:

- **A module is a file.** A file and a module are the same thing, so the list
  holds paths. For (b), ids stay unique only if every path is written one way,
  which brings back [Q-paths].
- **A module is a name (a namespace).** A module could be several files, or
  none, as with `<repl>` and `<code>` today. A span still needs the file,
  because editors and the debugger open files. So an entry in the list is a
  source name that may not be a path, and the module's own name is kept
  separately.

Nothing in the first version waits on this. Every unit holds one source, so
every `fileId` is 0.

### **Q:** What form does the recorded source path take?

<!-- [Q-paths]: #q-what-form-does-the-recorded-source-path-take -->

**Status:** Answered

VS Code sends absolute paths, and the debugger compares them with the path saved
in the unit. Options:

- **(a) An absolute path, resolved when `krb` starts the file.** Works today,
  because compiling and running happen in one process. A folder reached through
  a symbolic link can show up as a different path in VS Code, so the adapter
  should make both sides the same before comparing.
- **(b) The path as given on the command line.** Simple, but often relative, so
  the adapter would need to know the folder.
- **(c) Relative to a project root**, once there are [projects][Projects Proposal]. Portable, but needs the root to be known.

This interacts with [modules][Modules Proposal]: a unit compiled on one machine
and run on another has a path that means nothing there.

#### Answer

**First version: (b), the path as given.** It is the least work. `runFile` in
`src/main.c` already has the path, and Part 5 only needs it passed on as the
source name. (a) would add a call to resolve the path (`realpath`, which is
POSIX and not standard C), and a path with its symbolic links resolved can
differ from the one VS Code has.

The trouble with (b) is a relative path, and the debugger does not have it. The
adapter starts `krb`, so it passes an absolute path, and VS Code sends
breakpoints with the path it has for that file, so the two are compared as they
are ([Debugger Proposal], Part 5). Running `krb --debug` by hand with a relative
path is not covered, and attaching is left out of the debugger's first version.

Other sources record what they are given. The stdlib is recorded as
`stdlib/stdlib.krb`, the string `main.c` opens it with, and the debugger does
not need that path to know it is library code ([Q-library] there). `<repl>` and
`<code>` are names, not paths.

**Long term: (c), relative to the project root.** Such a path reads the same on
every machine, which (a) and (b) do not, so it is also the answer for a module
shipped as compiled code. It waits for the [Projects Proposal], which does not
yet say how a root is found ([Q-root] there). Moving to it should not mean
redoing anything else. The path is written in one place, where `runFile` passes
the source name. The adapter is the only reader that has to change, by turning
the path from VS Code into the recorded form before it sends a breakpoint. The
file list in the unit, `sourcePath`, and the debugger's comparison stay as they
are.

### **Q:** Which span information reaches the compiled unit?

<!-- [Q-compiled]: #q-which-span-information-reaches-the-compiled-unit -->

**Status:** Open

Part 2 puts a span on every AST node. Part 4 asks what survives into the
compiled output. Today a `CompiledFn` keeps one line for every byte of bytecode
(`codeLines`), and nothing else. Runtime error traces and the [Debugger
Proposal]'s breakpoints and stepping are built on it. Whatever is chosen, the VM
has to be able to read it without the compiler.

Measured on the 423 test programs that compile (12,135 bytes of bytecode, 7,260
instructions; [Appendix B]). The line changes 2,355 times.

| Option                                                    | Size                                                                            |
| --------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Today: one line per byte of bytecode                      | 48.5 KB                                                                         |
| (a) Keep that, and add a file for each function           | 48.5 KB                                                                         |
| (b) A full span (28 bytes) for every byte of bytecode     | about 340 KB                                                                    |
| (c) A table with 32 bytes per entry, one entry per change | 75 KB with an entry per line change, up to 232 KB with an entry per instruction |

- **(a) Keep one line per byte, and add a file per function.** The smallest
  change, and all the debugger's first version needs. Columns and offsets are
  lost, so run-time errors and the debugger stay at line level.
- **(b) A full span for every byte.** Allows precise run-time errors and
  column-level debugging. It is the largest by far.
- **(c) A table of entries, each covering a range of bytecode and holding a
  span** (proposed). Spans change more often than lines do, so the real size is
  between the two figures. It should be measured once spans are on nodes (Part 9,
  step 2). It carries offsets as well as line and column, which is what the
  spans in Part 1 are for.

The size is only a cost while compiling, and, when a tool asked for the data, for
as long as the function objects live (Part 7). It interacts with [Q-strip].

### **Q:** Is tooling data always recorded?

<!-- [Q-strip]: #q-is-tooling-data-always-recorded -->

**Status:** Open

Parts 5 and 6 record names and ranges in every compiled unit, and Part 4 may
record more than a line. The cost of the local variable records is not measured.
Since the loader copies the extra data only when a tool asked for it (Part 7),
and the unit is freed right after loading, a normal run pays only the
compile-time work. Options:

- **(a) Always record it** (proposed for the first version).
- **(b) Record it only with a flag**, for example when debugging or building for
  the editor.
- **(c) Keep it in a separate file** next to the compiled code, so shipped code
  stays small. This only matters once compiled units are saved and shipped,
  which is the [Modules Proposal]'s territory.

An embeddable language may care about size, so this should be measured before
being settled.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Span**: A record of where a piece of syntax came from in the original source:
  which file, and where it starts and ends, as byte offsets and as line and
  column. For generated syntax, a span can also carry an origin.
- **Offset**: A position in a file counted in bytes from its start. The first
  byte is at offset 0.
- **Column**: A position in a line counted in bytes from the start of the line.
  The first byte is column 1.
- **Origin**: A link from generated syntax to the span of the source that
  produced it.
- **Token**: One piece of source text as the scanner sees it, such as a name, a
  number, or `+`. In Kirby this is `Token` in `src/token.h`.
- **AST**: The tree-shaped data structure the parser produces from source text.
  Every later stage reads or rewrites this tree. In Kirby this is the `AstNode`
  type in `src/ast.h`.
- **Arena**: A block of memory that things are placed in one after another and
  that is freed all at once. The AST lives in one (`src/ast.c`).
- **Compiled unit**: What the compiler produces for one file (`CompiledUnit`).
- **String blob**: The block of text in a compiled unit that holds its names, so
  each name is stored as an offset and a length.
- **Bytecode position**: How far into a function's bytecode an instruction is.
  It is the first number on each line of the disassembler's output.
- **Slot**: A place in a frame's part of the stack where a local variable lives.
  Bytecode names variables by slot number.
- **Namespace**: A name that groups related code, such as `shapes`, used to
  refer to it instead of the path of the file it is in.
- **Hygiene**: The property that names a macro introduces cannot accidentally
  clash with names in the code that used the macro. It relies on the same
  per-syntax origin information spans carry.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Span]: #part-1--the-span
[Part 5]: #part-5--source-names
[Part 8]: #part-8--testing
[Appendix A]: #appendix-a--reproducing-the-baseline-claims
[Appendix B]: #appendix-b--measurements

<!-- Proposals -->

[Debugger Proposal]: ../debugger/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-naming]: ../modules/PROPOSAL.md#q-is-a-module-named-by-its-file-path-or-by-a-namespace
[Q-root]: ../projects/PROPOSAL.md#q-how-is-the-project-root-found
[Q-syntax]: ../macros/PROPOSAL.md#q-what-form-of-syntax-value-do-macros-receive-and-return
[Q-library]: ../debugger/PROPOSAL.md#q-how-does-step-into-treat-code-the-programmer-did-not-write

<!-- Questions -->

[Q-repr]: #q-what-is-the-exact-representation-of-a-span
[Q-cost]: #q-how-much-does-per-node-span-storage-cost-and-does-it-matter
[Q-line]: #q-which-line-does-an-instruction-get
[Q-origin]: #q-how-is-origin-chained-through-nested-macro-expansion
[Q-files]: #q-how-are-multiple-source-files-identified
[Q-paths]: #q-what-form-does-the-recorded-source-path-take
[Q-compiled]: #q-which-span-information-reaches-the-compiled-unit
[Q-strip]: #q-is-tooling-data-always-recorded

## Appendix A — Reproducing the baseline claims

The claims marked "checked" can be tried on a clean build at `from_commit`:

```shell
bash scripts/build.sh   # produces build/krb and build/kirby-test
```

**A call over several lines is reported on its last argument's line.**

```shell
cat > /tmp/a.krb <<'EOF'
fun boom(a: f64, b: f64): f64 {
  @panic("boom");
  a
}

var v = boom(
  1,
  2
);
EOF
./build/krb -f /tmp/a.krb   # [line 2] in boom() then [line 8] in script; exit 70
```

**A string over several lines puts its instruction on its last line.**

```shell
printf 'var s = "first\nsecond\nthird";\nprint s;\n' > /tmp/b.krb
./build/kirby-test -f /tmp/b.krb 2>&1 >/dev/null   # OP_CONSTANT is on line 3
```

**A local function is marked initialized before its closure exists.**

```shell
cat > /tmp/c.krb <<'EOF'
fun outer(): f64 {
  var before = 1;
  fun helper(): f64 {
    2
  }
  var after = helper();
  before + after
}
print outer();
EOF
./build/kirby-test -f /tmp/c.krb 2>&1 >/dev/null
# In outer: OP_CLOSURE is at position 0002, on line 5. helper is slot 2 and only
# has a value from position 0004.
```

**A block used as an expression gets the wrong slot when values are under it.**

```shell
cat > /tmp/d.krb <<'EOF'
fun f(): f64 {
  var a = 10;
  var r = 1 + 2 * { var b = 5; b + a };
  r
}
print f();
EOF
./build/krb -f /tmp/d.krb   # prints 25, and 31 is right
# In the listing, b is read with OP_GET_LOCAL 3, but two values are on the stack
# under it, so b is really in slot 4.
```

Making the block an argument of a call with two values before it, such as
`add3(1, 2, { var m = 3; m + k - 7 })`, fails with a runtime error.

**`CompiledFn.upvalues` is never filled in.**

```shell
grep -n "cuAddUpvalue" src/*.c   # only its definition in src/compiled_unit.c
```

**Only the parser creates AST nodes, and nothing else writes into one.**

```shell
grep -n "astAlloc(" src/*.c | grep -v src/parser.c
# only its definition in src/ast.c
grep -n -E -- "->(as\.[]A-Za-z_.[0-9]+|line|kind)[[:space:]]*(=|\+=)[^=]" src/*.c \
  | grep -v src/parser.c | grep -v -E "^src/(types|scanner)\.c"
# only the two lines in astAlloc. src/types.c writes into a Type, and
# src/scanner.c into a Scanner. Neither is a node.
```

**Snapshots.**

```shell
find tests -name '*.krb.err' | wc -l                        # 705
grep -l OP_CLOSURE $(find tests -name '*.krb.err') | wc -l  # 95
```

## Appendix B — Measurements

One machine, and programs from the test suite, so this shows the order of
magnitude and not a benchmark.

### The same call in other languages

Each program has a call whose `(` is on the first line, with its arguments on the
next two lines and its `)` on the last, and a callee that fails.

```python
def boom(a, b):
    raise RuntimeError("x")

v = boom(
    1,
    2
)
```

The Node, Lua, and Java programs are the same shape (`var v = boom(`,
`local v = boom(`, and `int v = boom(`). For C:

```c
int boom(int a, int b) { return a + b; }

int main(void) {
  int v =
    boom(
      1,
      2
    );
  return v;
}
```

Results, with the version tried in brackets:

- **Python (3.12).** The traceback says line 4, the first line of the call.
  `dis.get_instructions` gives the `CALL` instruction the range from line 4,
  column 4 to line 7, column 1. Its `sys.settrace` line events for the program,
  with the call on lines 4 to 7, were 4, 5, 6, 4 and then the callee.
- **Node (22).** The stack says `3:9`: the first line, at the callee's name.
- **Lua (5.4).** The traceback says line 5, the first line. `luac -l` shows the
  `CALL` on line 5 and the load of the last argument on line 8.
- **Java (21).** The stack trace says line 4, the first line.
- **C (gcc 13, `-g -O0`).** `objdump --dwarf=decodedline` has a row for the
  first line of the call and none for the argument lines.

For a method chain, Python and Node reported the line of the failing call's
method name, and Lua the first line of the chain.

### The size of a node

A throwaway C program that includes `src/ast.h` and prints `sizeof(AstNode)`
gave 136 bytes. `sizeof` of the same struct with the header changed in turn gave
the other rows of the table in [Q-repr]. It is built from a `NodeKind`, the
fields listed on that row, and a copy of the union of node payloads, which is
128 bytes. `Token` was 24 bytes both today and with a `column` added and its
fields reordered.

The same program was run with each of these changes made to the 160 byte node,
for [Q-origin] and [Q-files]. C rounds the size of a struct up to a multiple of
8, so a 4-byte field costs 8:

| Change to the 160 byte node                       | Node size | More than 160 |
| ------------------------------------------------- | --------- | ------------- |
| A 4-byte origin index, on the node or in the span | 168 bytes | +8            |
| An 8-byte origin pointer on the node              | 168 bytes | +8            |
| An 8-byte origin pointer inside the span          | 176 bytes | +16           |
| A chain pointer and a count on the node           | 176 bytes | +16           |
| A path pointer in place of `fileId`               | 168 bytes | +8            |

Inside the span, the origin pointer makes the span 40 bytes, and the path
pointer makes it 32. The 84 bytes for a chain three layers deep in [Q-origin] is
28 times 3, worked out and not measured.

### Nodes and bytecode in the tests

A second throwaway program was linked against `build/libkirby_core.a` with
`-Wl,--wrap=astAlloc` to count nodes as `parse()` ran over each `tests/**/*.krb`.
Across the 703 test programs that gave 7,800 nodes for 61,187 bytes of source,
or about one node for every 8 bytes. `stdlib/stdlib.krb` is empty today.

A third program ran `parse`, the type check, and `compile` on each program, and
for the 423 that compile it added up `codeCount` (12,135 bytes of bytecode) and
counted the places where `codeLines[b] != codeLines[b - 1]` (2,355). The
instruction count (7,260) is the number of listing lines that `kirby-test` prints
for those programs, apart from the two lines of the empty stdlib script at the
top of each. The sizes in [Q-compiled] are those counts times 4 bytes for today's
array, 28 for a span, and 32 for a table entry (4 for the position and 28 for the
span).
