---
status: Draft
created: 2026-09-19
from_commit: 662d98b
---

# Proposal: Debugger (VS Code)

This proposal adds debugger support to Kirby. In VS Code, a programmer will be
able to click next to a line of a `.krb` file to set a breakpoint, press F5,
and when the program stops there, look at the call stack and variables and
step through the program one line at a time.

It has four parts: extra information the compiler saves (described in the
[Tooling Data Proposal] and summarized in Part 1), a way for the VM to stop and
be looked at, a small text protocol between `krb` and VS Code, and a VS Code
adapter. The design leans on things Kirby already has (line numbers on every
instruction, one plain call stack). The choices that are still open are listed
as [Questions].

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

- When the proposal text says Kirby "does" or "has" something, that is true
  for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is
  true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c`
  code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Today there are two ways to see what a running Kirby program is doing: `print`,
and the trace Kirby prints when a runtime error happens:

```
panic: boom
[line 3] in area()
[line 10] in script
```

There is no way to stop a program while it runs, look at its variables, or
step through it. The [Change Log] already lists "Debugger (compatible with VS
Code)" under Future State.

### What Kirby has today

Checked at `from_commit` (see [Appendix A]):

- **Line numbers.** Every byte of bytecode has a line number (`Chunk.lines` in
  `src/chunk.h`, copied from `CompiledFn.codeLines` by the loader). The error
  trace above uses it. This is enough for line breakpoints and line stepping.
  There are no columns.
- **Function names.** `ObjFunction.name` holds the name. Lambdas get a
  generated name such as `lambda0x1` (`initCompiler` in `src/compiler.c`). The
  top-level script has no name.
- **Global variable names.** `vm.globals` is keyed by name.
- **All VM state in one place.** The call stack is the array `vm.frames`, and
  each `CallFrame` keeps its own `ip` (`src/vm.h`). `run()` in `src/vm.c` reads
  and writes `frame->ip` directly and keeps no private copy. Everything a
  debugger needs to read can be reached from `vm`.
- **Values print into a buffer.** `valueToString` (`src/value.h`) writes into a
  buffer the caller supplies, so it needs no garbage-collected memory.

### What is missing

1. **No file name.** A `CompiledUnit` does not say which file it came from.
   This matters because `krb -f` runs two units one after the other:
   `stdlib/stdlib.krb`, then the program.
2. **No local variable names.** The compiler knows them, but each one is a
   `Local` holding a `Token`, which points into the source text. `runFile` in
   `src/main.c` frees the source text as soon as compiling ends. What is left
   in the bytecode is only slot numbers, such as `OP_GET_LOCAL 1`.
3. **No way to stop.** The main loop in `run()` does nothing between
   instructions. (`DEBUG_TRACE_EXECUTION` prints a trace there, but it is a
   compile-time switch and cannot pause.)
4. **No way to talk to another program.** A Kirby program uses stdin and
   stdout itself: `print` writes to stdout and `@stdin` reads stdin until the
   end of input. A debugger cannot share them. `krb` also has no debug option.
5. **Errors throw the stack away.** `runtimeError` ends by calling
   `resetStack()`, so by the time anyone could look, the frames are gone.

Gaps 1 and 2 are gaps in the data the compiler saves. The [Tooling Data
Proposal] closes them (its Parts 5 and 6), and this proposal builds on that.

The VS Code extension (`vsc/`) has only a hover provider and a "Run Kirby File"
command.

## Proposed Changes

### Goals and non-goals

The first version should support:

- Line breakpoints. A breakpoint on a line with no code moves to the next line
  that has code.
- Continue, step over, step into, and step out.
- The call stack, with function names and lines.
- Variables: locals, captured variables, and globals. Arrays and struct
  instances can be expanded.
- Stopping on a runtime error, so the stack and variables at the moment of the
  error can be inspected.
- Starting from VS Code with F5, with the program running in the integrated
  terminal so `@stdin` and `@prompt` keep working.

Left out of the first version. Each can be added later without redoing the
parts below:

- **Evaluating expressions** (Watch, the Debug Console, values on hover). This
  needs an expression to be compiled and run against a paused frame. It is the
  hardest missing piece, and conditional breakpoints and logpoints depend on it.
- **Pausing a running program**, and changing breakpoints while it runs. The
  VM would need to check the channel while running. Until then, breakpoint
  changes made while running take effect at the next stop.
- **Attaching** to a `krb` that is already running.
- **Changing variable values.**
- **The REPL (`-r`) and `-c`.** Only `-f` is debugged.
- **Column-precise breakpoints**, and choosing which call on a line to step
  into. These need columns in the compiled output. The [Tooling Data Proposal]
  leaves that open ([Q-compiled]).
- **Debugging code that runs at compile time.** The [Macros Proposal] runs
  macros on the VM while compiling.

### Part 1 — Debug information in the compiled unit

The debugger reads data that the compiler saves in the compiled unit. That data
is described in the [Tooling Data Proposal], so that it is defined in one place.
What the debugger needs from it:

| The debugger needs                                                        | Where it is defined             |
| ------------------------------------------------------------------------- | ------------------------------- |
| The path of the file each function came from (`sourcePath`)               | [Tooling Data Proposal], Part 5 |
| A line for every instruction                                              | Part 4                          |
| The name of each local variable, and where in the bytecode it has a value | Part 6                          |
| The names of a closure's captured variables                               | Part 6                          |
| Copying all of it onto function objects only when debugging is on         | Part 7                          |

`krb --debug` turns on the copying in Part 7 before anything is loaded, so the
stdlib's functions have their data too.

Two decisions in that proposal change how the debugger behaves:

- **Which line an instruction has** ([Q-line]). An instruction is on the line
  where the code that produced it starts. Stepping through a call written over
  several lines goes to its first line, then its argument lines, then its first
  line again, as it does in Python. The line events, breakpoints, and stepping
  below only use "the line of an instruction".
- **What form the path takes** ([Q-paths]). The recorded path is the one `krb`
  was given. VS Code sends absolute paths, so the adapter starts `krb` with an
  absolute path (Part 5), and the debugger compares the two as they are.

### Part 2 — Pausing the VM

**A `Debugger` object.** A new file, `src/debugger.c`, holds everything the
debugger needs: the channel (Part 4), the breakpoints, what kind of step is in
progress, and a table of handles for the variables view. `VM` gets a
`Debugger *debugger` field, `NULL` when debugging is off.

**A check before each instruction.** At the top of the loop in `run()`, before
the next instruction is read, the VM asks the debugger whether to stop. How
this check is compiled in, so that it costs nothing when nobody is debugging,
is [Q-check]. The shape of the change:

```diff
   for (;;) {
+    if (debugging && debuggerShouldStop(&vm))
+      debuggerPause(&vm);
+
 #ifdef DEBUG_TRACE_EXECUTION
```

**Pausing.** `debuggerPause` sends a `stopped` message, then reads commands
from the channel until one of them lets the program run again. While paused,
the VM does not run, so nothing changes and the garbage collector cannot run.
The debugger must keep it that way: it builds its messages with plain string
buffers (`src/strbuf.c`) and `valueToString`, never with garbage-collected
strings, so pausing can never start a collection. The `kirby-test` binary is
built with `DEBUG_STRESS_GC`, so the tests will catch a mistake here.

**Runtime errors.** Every error, including those raised by natives such as
`@panic` and `@assert`, goes through `runtimeError`. The debugger hooks in
after the message and trace are printed and before the stack is thrown away:

```diff
+  if (vm->debugger != NULL)
+    debuggerStopOnError(vm);
+
   resetStack();
 }
```

The debugger keeps a copy of the message to send. When the programmer lets the
program go on, the error continues exactly as today, with the same output and
exit code 70. Compile errors happen before anything runs and do not involve the
debugger; the process exits with 65 and the adapter sees the connection close.

**Start-up.** With debugging on, `krb` connects to the channel first, receives
the breakpoints, and waits for `go` before running anything. That includes the
stdlib, which `-f` loads before the program. The program's own unit does not
exist yet at that point, so its breakpoints wait as _pending_. The loader tells
the debugger whenever a unit is loaded (`debuggerUnitLoaded`), which happens
before that unit's first instruction runs. The debugger then matches pending
breakpoints to the unit's source path and reports them as verified.

A `stopOnEntry` setting stops at the first line of the program's own code.

### Part 3 — Breakpoints and stepping

**Line events.** The debugger stops only at _line events_. A line event
happens when the next instruction is on a different line, in a different
function, or at a different call depth than the one before it, or right after a
jump backwards (`OP_LOOP`). This means a line with many instructions stops
once, and a loop stops on every trip around.

**Breakpoints.** A breakpoint is a file and a line. At each line event, the
debugger checks whether the current function's `sourcePath` and line match one.
Two consequences:

- A line can have code in several functions, for example a lambda written on
  the same line as the call that receives it. The breakpoint applies to all of
  them.
- After [monomorphization][Generic Types Proposal], one source line is
  compiled many times. The breakpoint applies to every copy, with no extra
  design, as long as each copy keeps the generic's source path and lines.

When a breakpoint is set, the debugger looks for the first line at or after the
requested one that has code in a function from that file. If there is one, the
breakpoint uses that line and reports it back. If not, or if the file's unit is
not loaded yet, the breakpoint stays unverified until one is.

**Stepping.** Each step remembers the call depth (`vm.frameCount`), function,
and line where it began:

| Step     | Stops at the next line event where...                                                                        |
| -------- | ------------------------------------------------------------------------------------------------------------ |
| Into     | ...there is any line event (except in library code, [Q-library])                                             |
| Over     | ...the depth is smaller, or the depth is the same and the function or line differs from where the step began |
| Out      | ...the depth is smaller                                                                                      |
| Continue | ...there is a breakpoint (or an error, or the program ends)                                                  |

Native functions have no frame, so stepping into a call to a native steps over
it. When a function returns, the caller's line counts as a new line event even
if it is the line of the call, so stepping out lands back on the calling line,
as it does in most debuggers.

**Known limits.** Two statements on one line are one step. A loop header
(`while`, `for`) is one line, so the debugger stops on it once for every trip
around the loop.

### Part 4 — The channel between VS Code and `krb`

**The option.** `krb` gets a new option, `--debug PORT`. The `-f` case in
`src/main.c` runs the file the moment it is seen, so `--debug` must come
_before_ `-f`:

```diff
     {"code", no_argument, 0, 'c'},  {"lex", no_argument, 0, 'l'},
-    {"parse", no_argument, 0, 'p'}, {0, 0, 0, 0}};
+    {"parse", no_argument, 0, 'p'},
+    {"debug", required_argument, 0, 'd'},
+    {"debug-script", required_argument, 0, 'D'}, {0, 0, 0, 0}};
```

An older `krb` that does not know the option prints "unrecognized option" and
then runs the program normally (checked, see [Appendix A]). So the adapter must
not assume the debugger is there. If `krb` exits without connecting, it should
tell the programmer that this `krb` has no debugger support.

**Who connects.** The adapter listens on a free port that only the same machine
can reach, and passes the port number to `krb`, which connects. The adapter
accepts one connection. See [Q-channel] for the alternatives.

**The messages.** A message is one line of text: words separated by single
spaces. Text that could contain spaces or newlines, such as paths and string
values, is written in quotes with the same escapes as a Kirby string literal.
Plain lines can be read and written with ordinary C input and output, and
pasted into a test file. An example of a session:

```
> break "/work/area.krb" 2
< bp 2 pending
> go
< verified "/work/area.krb" 2 2
< stopped breakpoint
> stack
< frame 0 "area" "/work/area.krb" 2
< frame 1 "script" "/work/area.krb" 10
< end
> scopes 0
< scope 1 "Locals"
< scope 2 "Captured"
< scope 3 "Globals"
< end
> vars 1
< var "w" number "5" 0
< var "h" number "2" 0
< end
> over
< stopped step
```

`>` is sent to `krb`, `<` comes back. The commands are `break`, `go`, `in`,
`over`, `out`, `stack`, `scopes`, `vars`, `option`, and `quit`. The events are
`stopped` (with a reason: `breakpoint`, `step`, `entry`, or `error`),
`verified`, and `exited`. The exact wording is settled when it is built and
written down in `docs/`.

**Handles.** A frame is numbered by its depth, top first. `scopes` and `vars`
hand out numbers that stand for a scope, an array, or a struct instance.
Every handle is dropped when the program runs again.

**What can be expanded.** Arrays show their elements, and struct instances show
their fields by name (`structFieldName` in `src/object.h`), including private
ones. Privacy is a rule for programs, not for someone inspecting one.

**One channel, two ends.** The channel is two small functions: read a line, and
write a line. The network connection is one implementation. A script file is
the other (Part 6), and it is how the C side gets tested.

### Part 5 — The VS Code side

**Extension manifest.** `vsc/package.json` gains a `breakpoints` entry for
`kirby` files (so the red dot works in the gutter), a `debuggers` entry, and a
"Debug Kirby File" command next to "Run Kirby File". The command finds `krb`
the same way `kirby.run` does, using the `kirby.executablePath` setting.

```diff
     "commands": [
       {
         "command": "kirby.run",
         "title": "Run Kirby File"
-      }
+      },
+      {
+        "command": "kirby.debug",
+        "title": "Debug Kirby File"
+      }
     ],
+    "breakpoints": [{ "language": "kirby" }],
+    "debuggers": [
+      {
+        "type": "kirby",
+        "label": "Kirby",
+        "languages": ["kirby"],
+        "configurationAttributes": { "launch": { "properties": {
+          // program, cwd, args, stopOnEntry, stopOnError
+        } } },
+        "initialConfigurations": [
+          { "type": "kirby", "request": "launch",
+            "name": "Debug Kirby File", "program": "${file}" }
+        ]
+      }
+    ],
```

**The adapter.** VS Code talks to debuggers using the [Debug Adapter Protocol],
so something has to translate. This proposal puts the translation in
TypeScript inside the extension, and keeps the C side as the small text
protocol above ([Q-adapter]). The adapter needs no new runtime dependency: an
extension can give VS Code an adapter object directly
(`DebugAdapterInlineImplementation`).

| VS Code asks for                        | The adapter                                                                                                                          |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `launch`                                | listens on a free port, asks VS Code to run `krb --debug PORT -f PROGRAM` in the integrated terminal, and waits for `krb` to connect |
| `setBreakpoints`                        | sends `break`, and shows the answers                                                                                                 |
| `configurationDone`                     | sends `go`                                                                                                                           |
| `threads`                               | answers with one thread, since the VM has one                                                                                        |
| `stackTrace`                            | sends `stack`                                                                                                                        |
| `scopes` / `variables`                  | send `scopes` / `vars`                                                                                                               |
| `continue`, `next`, `stepIn`, `stepOut` | send `go`, `over`, `in`, `out`                                                                                                       |
| `disconnect`                            | sends `quit` and closes the connection                                                                                               |

Running in the integrated terminal means `print` output and `@stdin` input
behave as they do with "Run Kirby File". Frames from library code are marked as
less important, so VS Code shows them dimmed.

**Working folder.** `krb -f` opens `stdlib/stdlib.krb` relative to the current
folder. Run from a folder without it, `krb` crashes (checked, see [Appendix
A]). The debug launch therefore starts in the same folder "Run Kirby File" does.
This is an existing limit and this proposal does not change it.

**Paths.** The adapter starts `krb`, so it passes the absolute path of the
program: the `program` from the launch configuration, made absolute against the
workspace folder if it is relative. `krb` records the path it was given
([Q-paths]), and VS Code sends breakpoints with the path it has for the same
file, so the two are compared as they are. Once there are projects, the recorded
path is expected to be relative to the project root. The adapter would then
turn the path from VS Code into that form before it sends `break`, and nothing
else changes.

### Part 6 — Testing

The repo uses snapshot tests, and this work follows the same TDD loop: write
one failing test, see it fail, make it pass.

- **The C side** is tested through the script channel. `krb --debug-script
FILE -f PROGRAM` reads its commands from `FILE` instead of a network
  connection, and writes the replies to stdout. It uses the same code as the
  network path; only the channel differs. When the script runs out while the
  program is paused, the program runs to the end. The replies and the program's
  own `print` output share one stdout, in a fixed order, because the VM has one
  thread.
- **Test files.** A test is `tests/debug/name.krb` plus `name.krb.dbg` (the
  commands) and the usual `.out`, `.err`, and `.exit`. `scripts/tests.sh` builds
  the command `$BIN -f FILE -- ARGS`, and `--debug-script` has to come before
  `-f`, so the runner needs a small addition: if a `.dbg` file exists, use it.
- **Debug information** is tested through the transcript (the `vars` output
  shows the names), not by printing it in the disassembler. `kirby-test` prints
  the bytecode listing to stderr and the `.err` snapshots contain it. This
  proposal does not change the bytecode, so no existing snapshot should change.
  The data itself is tested by unit tests in the [Tooling Data Proposal] (its
  Part 8). It also changes which line some instructions have ([Q-line]), and
  those snapshots are updated there.
- **Nothing changes without `--debug`.** The existing suite must pass with no
  snapshot updates.
- **The adapter** keeps its translation logic apart from the `vscode` API, so
  it can be tested with a fake channel and Node's built-in test runner. The
  extension has no tests today. Checking the whole thing in VS Code is manual at
  first.

### Part 7 — Suggested order

Each step is small, starts with a failing test, and leaves behavior without
`--debug` unchanged:

1. The channel and the script mode, with `go` and `exited` only.
2. Breakpoints on lines, `stopped`, and `stack` (this needs the source path,
   from Part 5 of the [Tooling Data Proposal]).
3. `scopes` and `vars` for locals (this needs the local variable records, from
   Part 6 of the [Tooling Data Proposal]).
4. Step over, into, and out.
5. Captured variables, globals, and expanding arrays and instances.
6. Stopping on runtime errors.
7. The network channel and `--debug PORT`.
8. The VS Code adapter, manifest, and command.

## Impacts

### Existing Syntax Or Behavior

- **No language change.** No syntax, opcode, or bytecode changes. A program run
  without `--debug` behaves exactly as before.
- **Bigger structures.** The [Tooling Data Proposal] describes the new fields on
  `CompiledUnit`, `CompiledFn`, `CompiledUpvalue`, and `ObjFunction`, and what
  it costs to record them ([Q-strip] there). `VM` gets a `Debugger *debugger`
  field.
- **Speed.** Depends on [Q-check]. A check on every instruction was measured at
  9–15% slower.
- **`runtimeError`** calls the debugger before `resetStack()`.
- **`src/main.c`** gets the new options. They must come before `-f`. Today an
  unknown option is reported and then ignored.
- **Snapshots contain bytecode.** `.err` files include the listing that
  `kirby-test` prints. Nothing here changes the bytecode. Printing local names in
  the disassembler would change many snapshots, and is deliberately not part of
  this proposal. Which line an instruction has changes under the [Tooling
  Data Proposal] ([Q-line]), and those snapshots are updated there.
- **Windows.** `scripts/install-windows.cmd` implies Windows users. Network
  connections need start-up code there that Linux and macOS do not.
- **The extension** gains a debugger contribution, a command, and TypeScript
  for the adapter.

### Related Proposals

- [Tooling Data Proposal] — defines the data the debugger reads: the file each
  function came from, a line for every instruction, local variable names with
  ranges, and captured variable names. [Q-paths] and [Q-strip] moved there from
  this proposal, and it holds [Q-line], [Q-files], and [Q-compiled]. Columns are
  not needed for the first version.
- [Modules Proposal] — a module may be shipped as compiled code without its
  source, and the debug information lives in that compiled code. It has to be
  decided whether it ships, and how a recorded path makes sense on another
  machine ([Q-paths] and [Q-strip], both now in the [Tooling Data Proposal]).
  That proposal is updated to say so.
- [Macros Proposal] — stepping through generated code, and locals a macro adds
  that the programmer never wrote (hygiene renames them), should not confuse
  the variables view. The origin information from spans is what lets the
  debugger point at the macro call. That proposal is updated to say so.
- [Generic Types Proposal] — each specialization of a generic is a separate
  compiled function. For breakpoints to work on all of them, each copy must keep
  the generic's source path and lines, and its name should be readable in the
  call stack (for example `sum[f64]`). That proposal is updated to say so.
- [String Interpolation Proposal] — the lowered code calls into the stdlib
  `StringBuilder`, so step into on such a line would enter the stdlib unless
  library code is skipped ([Q-library]). The lowered instructions should carry
  the line of the string. That proposal is updated to say so.
- [Testing Proposal] — once `krb test` exists, debugging a single test is a
  matter of launching the same way with a test file. That proposal is updated to
  mention this.
- [Projects Proposal] — a project root is the long-term base for the recorded
  source paths ([Q-paths]), and a project could name a default file to debug.
  How the root is found is not decided there yet, and is now a question in that
  proposal. That proposal is updated to say so.
- [Top-Level Declarations Proposal] — a run becomes "load the files, then call
  `main`". A file's top level then only defines names, so `stopOnEntry` would
  stop at the first line of `main`, not at the first line of the file, and the
  `script` frame in the session example would be `main` ([Q-call-main]).
  Nothing here changes until that proposal is accepted.

## Questions

### **Q:** Where does the Debug Adapter Protocol live?

<!-- [Q-adapter]: #q-where-does-the-debug-adapter-protocol-live -->

**Status:** Open

VS Code speaks the Debug Adapter Protocol, which is JSON. Options:

- **(a) In TypeScript, inside the extension** (proposed). The C side speaks the
  small text protocol from Part 4. `krb` needs no JSON code, and Kirby has none
  today.
- **(b) In C, inside `krb`** (for example `krb --dap`). Any editor that speaks
  the protocol could use `krb` directly, with no extension code. The cost is a
  JSON reader and writer in C, and a bigger `krb`.

The debugger core (Parts 2 and 3) does not depend on the protocol, so (b) could
be added later on top of it.

### **Q:** What carries the debugger messages?

<!-- [Q-channel]: #q-what-carries-the-debugger-messages -->

**Status:** Open

Options:

- **(a) A network connection to the same machine** (proposed), with the adapter
  listening. It works on every operating system. The port can be reached by
  other programs on the machine, so the adapter accepts one connection only, and
  a one-time token in the first message could close the gap further.
- **(b) An extra inherited file or named pipe.** No port, but it is done
  differently on Windows and Linux/macOS.
- **(c) stdin and stdout.** Ruled out: Kirby programs use both.

### **Q:** How does the VM check whether to stop?

<!-- [Q-check]: #q-how-does-the-vm-check-whether-to-stop -->

**Status:** Open

The check in Part 2 runs before every instruction. Measured on a scratch copy
of `from_commit` (a Release build; a program with a 3-million-trip `while` loop
and `fib(30)`; 25 runs alternating between builds; medians; one machine, so
treat the numbers as rough; [Appendix A] describes it):

| Version                                            | Time    | Difference |
| -------------------------------------------------- | ------- | ---------- |
| Today                                              | 0.278 s | —          |
| (a) One check per instruction                      | 0.305 s | +9.9%      |
| (b) Two copies of the loop, one chosen at start-up | 0.281 s | +1.1%      |

Two other runs of (a) measured +9.4% and +15.2%. The loop was made of cheap
instructions, so a program that spends its time in natives or the collector
would see less.

- **(a) One check per instruction.** Simplest. Costs everyone, debugging or not.
- **(b) Two copies of the loop** (proposed). `run()` becomes an inlined function
  taking a `debugging` flag, and is called twice: once with `false` (no check
  at all) and once with `true`. `interpretFunction` picks one, once. The
  measured cost when not debugging is within noise. The binary grew by about
  8.5 KB (218,624 to 227,112 bytes). It needs a way to force inlining
  (`__attribute__((always_inline))` on GCC and Clang, `__forceinline` on MSVC).
  The project builds with `-Wall -Wextra -Wpedantic`, and the experiment
  produced no new warnings.
- **(c) Patch the bytecode.** Overwrite the first instruction of a breakpoint
  line with a special "break" opcode, and put the original back when the
  breakpoint goes. No cost when not debugging, but stepping and error stops
  still need a check, and the patching has to be undone correctly.

### **Q:** How does step into treat code the programmer did not write?

<!-- [Q-library]: #q-how-does-step-into-treat-code-the-programmer-did-not-write -->

**Status:** Open

The stdlib is loaded with every run, and the [String Interpolation
Proposal] lowers `$"..."` into calls into it. Stepping into such a line would
land in stdlib code. Options:

- **(a) Skip library files** (proposed for the first version). `main.c` already
  loads the stdlib through its own call, so it can mark that unit as library
  code. Step into passes over functions from it.
- **(b) Also skip generated code**, once spans record where code came from (the
  [Tooling Data Proposal], Part 3), for example macro output.
- **(c) Never skip.** Simplest, but stepping through a program would keep
  dropping into the stdlib.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Debugger**: A tool that lets a programmer stop a running program, look at
  its state, and run it one step at a time.
- **Breakpoint**: A file and line where the program should stop.
- **Pending breakpoint**: A breakpoint whose file has not been loaded yet, so
  it cannot be matched to code.
- **Step over / into / out**: Run to the next line without entering calls / to
  the next line including inside calls / until the current function returns.
- **Line event**: A moment in the run when the next instruction is on a
  different line, function, or call depth than the one before, or follows a
  jump backwards. The debugger stops only at these.
- **Call depth**: How many functions are running at once, that is
  `vm.frameCount`.
- **Frame**: The record of one running function call (`CallFrame`), with its
  position in the bytecode and where its variables start on the stack.
- **Slot**: A place in a frame's part of the stack where a local variable lives.
  Bytecode names variables by slot number.
- **Bytecode position**: How far into a function's bytecode an instruction is.
  It is the first number on each line of the disassembler's output.
- **Compiled unit**: What the compiler produces for one file (`CompiledUnit`).
- **Library code**: Code that comes with the language rather than from the
  programmer, today `stdlib/stdlib.krb`.
- **Debug Adapter Protocol**: The standard set of JSON messages VS Code uses to
  talk to debuggers.
- **Debug adapter**: The small program that translates between VS Code's
  messages and a particular debugger's. Here it is TypeScript in the extension.
- **Channel**: The connection that carries debugger messages between `krb` and
  the adapter.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Appendix A]: #appendix-a--reproducing-the-baseline-claims

<!-- Proposals -->

[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md
[Testing Proposal]: ../testing/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-line]: ../tooling-support-data/PROPOSAL.md#q-which-line-does-an-instruction-get
[Q-files]: ../tooling-support-data/PROPOSAL.md#q-how-are-multiple-source-files-identified
[Q-paths]: ../tooling-support-data/PROPOSAL.md#q-what-form-does-the-recorded-source-path-take
[Q-compiled]: ../tooling-support-data/PROPOSAL.md#q-which-span-information-reaches-the-compiled-unit
[Q-strip]: ../tooling-support-data/PROPOSAL.md#q-is-tooling-data-always-recorded
[Q-call-main]: ../top-level-declarations/PROPOSAL.md#q-how-does-kirby-call-main

<!-- External -->

[Change Log]: https://github.com/kirbylang/kirbylang/blob/61b7cc5/docs/CHANGELOG.md
[Debug Adapter Protocol]: https://microsoft.github.io/debug-adapter-protocol/

<!-- Questions -->

[Q-adapter]: #q-where-does-the-debug-adapter-protocol-live
[Q-channel]: #q-what-carries-the-debugger-messages
[Q-check]: #q-how-does-the-vm-check-whether-to-stop
[Q-library]: #q-how-does-step-into-treat-code-the-programmer-did-not-write

## Appendix A — Reproducing the baseline claims

The claims marked "checked" can be tried on a clean build at `from_commit`:

```shell
bash scripts/build.sh   # produces build/krb and build/kirby-test

cat > /tmp/a.krb <<'EOF'
fun area(w: f64, h: f64): f64 {
  var result = w * h;
  @panic("boom");
  result
}

var total = 0;
{
  var inner = 5;
  total = area(inner, 2);
}
print total;
EOF

# The error trace, and exit code 70
./build/krb -f /tmp/a.krb

# Bytecode listing on stderr: locals are slot numbers only (OP_GET_LOCAL 1),
# and the lines are there
./build/kirby-test -f /tmp/a.krb

# An unknown option is reported, then ignored, and the program still runs
./build/krb --debug 4711 -f /tmp/a.krb

# stdlib/stdlib.krb is opened relative to the current folder: this crashes
(cd /tmp && /path/to/build/krb -f /tmp/a.krb)
```

### The check's cost ([Q-check])

Two throwaway copies of the source tree were built with CMake's Release
configuration, next to an unchanged build.

- **(a)** A global `int` that is always 0, tested at the top of the loop in
  `run()`, calling a function that is never inlined when it is not 0.
- **(b)** The body of `run()` turned into an `always_inline` function taking a
  `bool debugging`. `run()` calls it with `false` and a second function calls
  it with `true`; `interpretFunction` picks between the two on the same global.
  Both copies are reachable, so the compiler cannot drop the second.

The program:

```kirby
fun fib(n: f64): f64 {
  if (n < 2) return n;

  fib(n - 2) + fib(n - 1)
}

var i = 0;
var sum = 0;
while (i < 3000000) {
  sum = sum + i % 7;
  i = i + 1;
}
print sum;
print fib(30);
```

It was run 25 times per build, alternating between builds, and the medians were
compared. This is one machine and a program of very cheap instructions, so it
shows the order of magnitude, not a benchmark.
