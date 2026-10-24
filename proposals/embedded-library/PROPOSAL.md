---
status: Draft
created: 2026-09-20
from_commit: 2f96a2e
---

# Proposal: Embedding Kirby In Other Programs

This proposal makes Kirby usable as a library inside another program, the way
Lua is. A game engine, an editor, or any other program (the _host_) would link
Kirby in, give it a few functions of its own, load scripts, and call into them,
for example once every frame. One of the goals is game scripting.

Today Kirby is a program (`krb`) with a library inside it. The library half
can already run a script from C. But a script can end the host program, change
the host's random numbers, and write to the host's output, and the host cannot
call a script function or get a value back. This proposal lists what has to
change so that none of that happens. It is nine parts that can land one at a
time. Every part leaves `krb` behaving as it does today, except where the
[Impacts] section says otherwise.

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
defined in the [Glossary] below. C is explained in plain words where it comes
up.

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
- C code for the new public interface is a **sketch**. Names and details are
  settled by the [Questions] and by the tests that are written first.

## Problem Statement

A program that embeds a language needs five things from it. Lua is the usual
model, so they are worded the way Lua's users would expect them:

1. **Many independent copies.** A game may want one for the world, or one per
   script, each with its own variables.
2. **Safety from the script.** A mistake in a script must stop the script. It
   must not stop the game, change the game's settings, or corrupt its memory.
3. **A conversation in both directions.** The host gives the script functions
   to call (spawn an enemy). The host calls script functions (`onUpdate`) and
   gets values back.
4. **Control.** The host decides what a script may touch (files, the
   environment), how long it may run, how much memory it may use, and where its
   output goes.
5. **Small and shippable.** A shipped game should carry as little as it can,
   ideally the runtime and precompiled scripts, not the compiler.

### What Kirby already has

Checked at `from_commit` ([Appendix A]):

- **The pipeline works from C.** A program that does not use `src/main.c` can
  parse, type check, compile and run a script. `compileSource` in `main.c` is
  `static`, so today the host has to copy it, but nothing else is missing.
  ([A.1])
- **Script errors are survivable.** A runtime error or a compile error in a
  script returns a status, and the same VM runs the next script. ([A.1])
- **The runtime half stands on its own.** `vm.c` does not depend on the
  compiler (see `interpret()` in `src/vm.h`). The signature table for the type
  checker lives in `src/native_signatures.c`, apart from the natives
  themselves. `kirby_runtime`, a library of the runtime files only, is built
  from `KIRBY_RUNTIME_SOURCES` in `CMakeLists.txt`, along with an executable
  made from it, `krb-runtime`. `kirby-runtime-link-unit-test`
  (`unit/runtime_link.c`) links nothing but `kirby_runtime` and calls `initVM`
  and `freeVM`. Linking the runtime files by hand leaves no name undefined
  ([A.4]). See [Part 1].
- **The collector is already a separate object.** `GC` (`src/gc.h`) has its own
  struct and a callback that marks the VM's roots. Every object on the heap is
  allocated through one function, `reallocate` in `gc.c`. A few helper buffers
  call `malloc` directly: the collector's own list of objects still to visit, the
  loader's temporary index array, `StrBuf` (`src/strbuf.c`), three places in
  `native.c`, and the code that holds and reads compiled units
  (`compiled_unit.c`, `stringset.c`, `unit_bytes.c`, `packaged_program.c`).
- **A host can register a function.** `defineNative` puts a C function in the
  VM's globals, and a script can call it. ([A.3])

### What is missing

1. **One copy per process.** The VM is a variable that lives for the whole
   program (`VM vm;`), next to the collector (`GC gcInstance;`). The compiler,
   the type checker and their helpers add 26 more, so there are 28 in all
   (compiler 9, types 9, impl targets 4, checker 2, AST 2, VM 2). Every call to
   `pushOnStack` and its two siblings (91 call sites in five files) reaches
   `vm` through that variable, including the ones in `loader.c`, `object.c` and
   `chunk.c` that are handed a `VM *`. ([A.5])
2. **A script's mistake can end the host.** Outside `main.c` there are 51
   `exit()` calls: 17 in `asserts.c` (the argument checks), 16 in `native.c`
   (one is `@exit` itself), one in `native_signatures.c` for a built-in
   signature that does not parse, and 17 for running out of memory.
   `@arrPop` on an empty array, `@getenv("UNSET")`, `@panic`, `@assert`, a
   missing file, and `@argv("x")` all end the process with exit code 70.
   `@argv("x")` even passes the type checker, because `@argv` has no
   signature. ([A.2]) Two natives do not exit but return after reporting:
   `@stdin(1)` and `@prompt(1)` print their message and then crash (exit code
   139), because the report already emptied the stack under the caller. ([A.2])
3. **No way to call a script function or read a result.** `interpret()` returns
   a status and nothing else. A host can only compile and run a snippet such as
   `update(0.016);` every time. That is fast enough (about 3 µs per call,
   against about 0.1 µs for a direct call in the prototype, [A.6] and [A.7])
   but it cannot pass a value except by pasting it into source text. A player
   named `Bob"); @exit(9); say("` ended the test host with exit code 9. ([A.6])
4. **A script can change the host's process.** `initVM` calls
   `srand(time(NULL))`, which replaced a random sequence the host had seeded
   itself. `@print` and `@println` write straight to stdout, and errors go to
   stderr. ([A.8])
5. **Nothing stops or limits a script.** `while (true) {}` runs until the host
   is killed from outside. A script can nest at most 63 calls, because
   `FRAMES_MAX` is 64 and the script itself takes one. Both limits are fixed at
   compile time. Each VM is 394,832 bytes, almost all of it the value stack.
   ([A.8])
6. **Host functions are unchecked.** A function that is not in the signature
   table gets no type checking: `let x: i64 = @typeof(3)` compiles, and an
   unknown `@nothing(1)` is only found when it runs. Worse for a host, an
   argument keeps whatever kind the script gave it. `@hostAdd(1, 2)` passes two
   `i64` values to a host function that reads `f64`s, and it returns
   `1.5e-323`. Built-in signatures are Kirby type text, and 58 of the 73
   natives have one, but there is no way to give one to a host function.
   ([A.3])
7. **Everything ships together.** The front end (parser, type checker,
   compiler) is 142,578 bytes of code and the runtime is 73,240. A compiled
   unit has a byte form, and `krb build` attaches units to `krb-runtime`
   ([Part 9]), but a host has no call to load one. The shared library that
   CMake already builds exports 256 symbols, among them the variables `vm` and
   `gcInstance` and general names like `parse` and `compile` that can collide
   with the host's own. ([A.4], [A.9])

None of this is a flaw for a command line tool. All of it is a flaw for a
library.

## Proposed Changes

### What an embedded Kirby looks like

This is the target, written as a game would use it. The script is ordinary
Kirby, and with the three `@` functions supplied by the host it moves the
enemies as described ([A.11]). Loading it today also reports a runtime error,
because of a compiler bug that leaves out the script's final return ([A.11]).
The C code shows the interface this proposal adds.

```kirby
struct Enemy {
  pub var id: f64;
  pub var speed: f64;
}

impl Enemy {
  pub fun spawn(x: f64, y: f64): Enemy {
    let id = @spawn("goblin", x, y);
    Enemy { id: id, speed: 40 }
  }

  pub fun update(self, dt: f64): unit {
    @setX(self.id, @getX(self.id) + self.speed * dt);
  }
}

var enemies: [Enemy] = [];

fun onStart(): unit {
  @arrPush(enemies, Enemy.spawn(0, 0));
  @arrPush(enemies, Enemy.spawn(100, 50));
}

fun onUpdate(dt: f64): unit {
  var i = 0;
  while (i < @arrLen(enemies)) {
    enemies[i].update(dt);
    i = i + 1;
  }
}
```

```c
#include "kirby.h"

static KrbValue hostSpawn(Kirby *k, int argc, const KrbValue *argv) {
  const char *kind;
  double x, y;

  if (!krbToString(argv[0], &kind, NULL) || !krbToNumber(argv[1], &x) ||
      !krbToNumber(argv[2], &y))
    return krbRaise(k, "@spawn expects (string, f64, f64)");

  return krbNumber(worldSpawn(krbUserData(k), kind, x, y));
}

int main(void) {
  World world = worldNew();

  KrbConfig config;
  krbConfigInit(&config);
  config.userData = &world;
  config.libs = KRB_LIB_BASE | KRB_LIB_MATH | KRB_LIB_ARRAY;
  config.print = consolePrint;

  Kirby *k = krbNew(&config);

  krbDefineFunction(k, "@spawn", hostSpawn, "fun (string, f64, f64) => f64");
  krbDefineFunction(k, "@getX", hostGetX, "fun (f64) => f64");
  krbDefineFunction(k, "@setX", hostSetX, "fun (f64, f64) => unit");

  if (krbLoad(k, "scripts/enemies.krb", source, sourceLength) != KRB_OK) {
    logError("script: %s", krbErrorMessage(k));
    return 1;
  }

  krbCallGlobal(k, "onStart", 0, NULL, NULL);

  for (;;) {
    KrbScope scope = krbScopeBegin(k);
    KrbValue dt = krbNumber(1.0 / 60.0);

    if (krbCallGlobal(k, "onUpdate", 1, &dt, NULL) != KRB_OK)
      logError("script: %s", krbErrorMessage(k));

    krbScopeEnd(k, scope);
    worldTick(&world);
  }
}
```

What this shows, and which part of the plan provides it:

- `krbNew` makes an independent copy, and `krbFree` (not shown) ends it
  ([Part 2]).
- A script that fails returns a status and a message. The game keeps running
  ([Part 3], [Part 4]).
- The script's `@println` goes to the game's console, not to stdout
  ([Part 4]).
- The host calls `onStart` and `onUpdate` by name and can read what they return
  ([Part 5]).
- `@spawn`, `@getX` and `@setX` are checked by the type checker like any
  built-in function ([Part 6]).
- The script gets math and arrays, and nothing that reads files or the
  environment or ends the program ([Part 7]).
- The host includes one file, `kirby.h`, and links one library ([Part 8]).

### Rules

Five rules hold for an embedded Kirby. Each part of the plan exists to make one
of them true:

1. **Kirby never ends the host.** No `exit`, `abort` or crash for a mistake in a
   script. The one exception is running out of memory, which stays fatal until
   [Q-oom] is answered.
2. **Kirby touches nothing that belongs to the whole process.** Not the random
   number generator, not the environment, not stdout, unless the host allowed
   it.
3. **Everything Kirby says goes through a hook.** Program output, error
   messages and compile errors each go to a function the host can set. The
   default writes to stdout and stderr exactly as today.
4. **Every limit is a setting.** Stack depth, memory, and running time are
   chosen by the host.
5. **`krb` is the first host.** The command line tool uses the same interface,
   so the 3,366 checks in the existing E2E suite prove that nothing changed.

### Goals and Non Goals

What this proposal covers:

- Independent Kirby instances in one process.
- Errors in scripts and in native functions returned to the host, never
  `exit()`.
- Output and diagnostics through hooks.
- Calling script functions from the host, with values in and out, safe with
  the garbage collector.
- Host functions with type signatures the checker uses.
- Choosing which built-in functions a script gets, and limits on stack, memory
  and time.
- A public header and a library that is easy to link.
- Compiled scripts as a file, and a build of the runtime without the compiler.
- `krb` behaves the same, byte for byte, apart from the changes in [Impacts].

The following is intentionally left out of scope for this proposal:

- **Coroutines** (a script that pauses and continues on a later frame). Game
  scripting often wants them. Nothing here prevents adding them later
  ([Q-coroutines]).
- **Running one instance on several threads at once.** Different instances on
  different threads are in scope once compiling is taken care of
  ([Q-threads]).
- **A new heap object for host data** ("userdata"). Host objects are numbers
  (handles) at first ([Q-userdata]).
- **Reloading a changed script into a running instance** ([Q-reload]).
- **A safe way to load bytecode from an untrusted source.** Bytecode is not
  checked for safety (Part 9).
- **Wrappers for other languages, or a C++ wrapper.** A stable C interface is
  what they would be built on.
- **Speed work** on the VM, such as smaller values. Nothing here changes how
  fast a script runs.
- **Making natives portable to every platform.** `setenv` is a POSIX function
  today.

### Implementation Plan

Each part follows the project's test-first rule: write one test, see it fail,
make the change, see it pass. Parts 1 to 3 change no behavior that `krb` shows.
The existing E2E suite (3,366 checks over 1,122 tests) must pass unchanged after
every part. The exceptions are listed in [Impacts].

Suggested order: Part 1, 2 and 3 first, then 4 and 5, then 6 and 7, then 8. Part
9 needs Part 1 and can go any time after it. Part 5 needs Parts 2 and 3. Part 6
needs Part 2. Part 7 needs Parts 2 and 5.

#### Part 1: The runtime links without the compiler

This part is in place. The natives are in `src/native.c` and their type
signatures, which need the type checker, are in `src/native_signatures.c`.
`kirby_runtime` is a CMake target made of the runtime files only
(`KIRBY_RUNTIME_SOURCES`, the list is in [A.4]), and `krb-runtime` is an
executable made from it ([Part 9]). `unit/runtime_link.c` is the test:
`kirby-runtime-link-unit-test` links only `kirby_runtime` and calls `initVM`
and `freeVM`. The later parts keep that test passing.

#### Part 2: One instance, no hidden shared state

Three changes and one bundle.

**The VM is the instance.** `VM vm;` and `GC gcInstance;` at the top of `vm.c`
go away. Every function in `vm.c` that reads `vm.something` gets a `VM *vm`
parameter instead, and the small helpers that push and pop the stack take one
too:

```diff
-void pushOnStack(Value value);
-Value popFromStack(void);
-void popNFromStack(int count);
+void pushOnStack(VM *vm, Value value);
+Value popFromStack(VM *vm);
+void popNFromStack(VM *vm, int count);
```

That is 91 call sites: 63 in `vm.c`, 18 in `native.c`, 4 each in `loader.c` and
`object.c`, and 2 in `chunk.c`. Every one is a mechanical edit. Three unit tests
that `#include` `vm.c` or use `vm` directly change with them
(`vm_field_limit.c`, `vm_invoke.c`, `vm_repl_immutability.c`).

`defineNative` needs one more fix in the same pass. It puts the name and the
function on the stack, then reads them back from `vm->stack[0]` and
`vm->stack[1]`. That is only right while the stack is empty, which stops being
true once a host can register a function while a script is suspended in the
middle of a call. It reads `vm->stackTop[-2]` and `vm->stackTop[-1]` instead.

The collector becomes part of the VM's own memory: a member of the struct
instead of a separate global. `vm->gc` keeps pointing at it, so no other code
changes.

**Sizes are settings.** `FRAMES_MAX` (64) and `STACK_MAX` become the defaults
for two settings, and the two arrays are allocated when the instance is made.
The `VM` struct shrinks from 394,832 bytes to a few hundred. The stack still
never moves after it is made. The code depends on that: `CallFrame.slots` and
`ObjUpvalue.location` are plain pointers into it.

**Random numbers belong to the instance.** `srand(time(NULL))` leaves `initVM`.
Each VM holds the state of its own random number generator. It uses one small,
fixed algorithm, so the same seed gives the same numbers on every platform,
which a game replay needs. `@rand`, `@rand01` and `@randBetween` use it and keep
their current ranges (`@rand` is 0 to 2,147,483,647, which is `RAND_MAX` on the
test machine). `krb` seeds it from the clock, as before. A host passes a seed,
or lets Kirby pick one. Nothing in Kirby calls `srand` or `rand` any more.

**The compiler and type checker's shared state goes into one bundle.** The
compiler, the type checker and their helpers keep 26 file-scope variables
between them ([A.5]). Stage A gathers them into one struct, `FrontEnd`, that
lives inside the instance. The front-end code reaches it through a single
pointer, which `krbLoad` and `krbEval` set when they start and put back when
they finish. Two instances can be compiled one after the other. Two compiles at
the same instant still have to take turns. [Q-frontend-context] covers passing
the pointer explicitly through all 9,200 lines instead.

`typchkSessionBegin`, `typchkSessionEnd` and `compilerSessionEnd` become part of
making and freeing an instance. Today `main.c` calls them by hand in three
places.

**The standard library is loaded by `krbNew`.** `stdlib/stdlib.krb` is
compiled into the binary as the text `KIRBY_STDLIB` by
`scripts/generate_stdlib_c.sh`, and `src/main.c` runs that text with `runCode`
(in `sessionBegin`). `krbNew` does the same instead. The file holds one struct,
`Date`, and its `Display`, `Ord` and `Eq` impls. What that costs each new
instance is [Q-stdlib].

Test first: `unit/instances.c` makes two instances, runs `var x = 1;` in one
and `var x = 2;` in the other, and checks that each reads its own. It cannot be
written against today's code, because there is only one VM to make.

#### Part 3: A script's mistake stops the script

The rule for a native changes from "print and exit" to "raise and return".

**Today** a failing native prints its message and ends the process:

```c
static Value arrPopNative(VM *vm, int argCount, Value *args) {
  // ...
  if (array->count == 0) {
    runtimeError(vm, "Cannot pop empty array.");
    exit(EXIT_CODE_RUNTIME_ERR);
  }
  // ...
}
```

**Proposed:** a new function, `raise`, writes the message on the VM, sets a
flag, and hands back nil. The native returns at once with what it gets:

```diff
   if (array->count == 0) {
-    runtimeError(vm, "Cannot pop empty array.");
-    exit(EXIT_CODE_RUNTIME_ERR);
+    return raise(vm, "Cannot pop empty array.");
   }
```

The VM looks at the flag right after each native call, in `callValue`, with one
`if`. When it is set, the VM writes the message and the trace, unwinds, and
`run()` returns `INTERPRET_RUNTIME_ERROR`. The message and trace are what
`runtimeError` writes today, at the same moment (after the call and before the
stack is thrown away), so the output is identical. The 180 tests that expect
exit code 70 check exactly that. The [Debugger Proposal] hooks in at that same
spot, and nothing here moves it.

The 16 argument checks in `asserts.c` return a `bool` (true when the argument
is fine) so that a native can stop at the first bad one. A small macro keeps
native code short:

```c
#define REQUIRE(check)                                                         \
  do {                                                                         \
    if (!(check))                                                              \
      return NIL_VAL;                                                          \
  } while (0)
```

```diff
-  assertArgCount(vm, "@sqrt", 1, argCount);
-  assertArgIsNumber(vm, "@sqrt", args, 0);
+  REQUIRE(assertArgCount(vm, "@sqrt", 1, argCount));
+  REQUIRE(assertArgIsNumber(vm, "@sqrt", args, 0));
```

It is a macro because C has no other way to say "return from the function I am
in if this failed". It is the one place where a macro hides a `return`, and the
name says so.

In `native.c`, 15 of the 16 `exit()` calls are error exits and become `raise`.
The 16th is `@exit` itself. `@panic` and `@assert` use `raise` too. They print
`panic: ...` and exit with 70 today, and `krb` still exits with 70 because
`runFile` turns `INTERPRET_RUNTIME_ERROR` into 70.

**`@exit` becomes a status.** It stores the code on the VM and stops the script
with a new status, `INTERPRET_EXIT`, which prints nothing. `krb` turns that
into `exit(code)`. A host decides for itself. Usually it does not give scripts
`@exit` at all ([Part 7]).

**Unwinding goes back to where the call began, not to an empty stack.** Today
`runtimeError` ends with `resetStack()`, which empties the stack, the frames and
the open upvalues. It does that through the global `vm`, even when it was handed
a different `VM *`. [Part 5] lets the host call a script from inside a host
function, so an error in the inner call has to leave the outer call as it was.
`resetStack` becomes `unwindTo(vm, savedFrames, savedStackTop)`. The outermost
caller saves the empty state, which is what happens today.

**`@stdin(1)` and `@prompt(1)`** already return after `runtimeError` instead
of exiting, and crash ([A.2]). They are fixed by the same change, because
`raise` and the flag are how a native reports a problem now.

**Not in this part:** running out of memory. The 17 calls for it in `gc.c`,
`ast.c`, `loader.c` and others stay ([Q-oom]). So does the `exit` in
`native_signatures.c`, which only fires when a built-in signature in Kirby's
own source does not parse.

Test first: `unit/errors.c` runs each snippet from [A.2] and checks that the
call returns `KRB_RUNTIME_ERROR`, that the message is the one printed today,
and that the same instance then runs `@println(1);`. Also a new E2E test,
`tests/run/native_functions/native_fn_stdin_call_non_string_arg.krb`, which
crashes with 139 today and should exit with 70.

#### Part 4: Everything Kirby says goes through a hook

**Program output.** `@print` and `@println` (`src/native.c`) build the whole
text, the value and, for `@println`, its newline, in a `StrBuf`
(`src/strbuf.c`) and hand it to one function the host can set:

```c
typedef void (*KrbWriteFn)(void *user, const char *text, size_t length);
```

The default is `fwrite` to stdout. `printValue` and `printObject` print piece by
piece with `printf` today, an array being one call per element. A sibling that
appends to a `StrBuf` replaces them in the print natives. `@eprint` and
`@eprintln` go to the error hook below. The originals stay for the debug
listings.

**Error messages.** `runtimeError` writes the message and the trace lines into
a buffer on the instance, then calls an error hook (`KrbWriteFn` again, and the
default is stderr). The host can read the same text with `krbErrorMessage(k)`
until the next call.

**Compile errors.** Every message from the parser, the compiler and the type
checker is written by one of five small functions: one in `parser.c`, two in
`compiler.c`, two in `typecheck.c`. (`definite_assignment.c` uses the
checker's.) Each writes to stderr with `fprintf` a piece at a time: first
`[line 3] Error`, then ` at 'x'`, then the message. They change to build one line and give
it to the same hook and buffer. The text and its order stay the same. When the
[Tooling Data Proposal] adds spans, these five functions are also where a
structured record (file, line, column, message) can be handed over instead of
only text.

**Left alone:** the `DEBUG_*` build flags write with `fprintf` and `printf` from
`debug.c`, `gc.c` and `object.c`. They are for people working on Kirby, not for
hosts.

**Input** is not hooked. `@stdin` and `@prompt` read the process's stdin. A
host that wants script input provides its own function ([Part 6]) and leaves
`KRB_LIB_IO` out ([Part 7]).

Test first: `unit/output_hook.c` runs `@println(1);`, `@println("a");`,
`@println([1, 2]);`, a runtime error and a compile error with hooks that keep
the text in memory, and checks the text.

#### Part 5: Call Kirby from the host, and get values back

**A call that can start while another is running.** `run()` in `vm.c` runs
until the last frame returns. To call a script function from inside a host
function that a script called, `run()` takes the frame depth at which it should
stop, and a place to put the result:

```diff
-static InterpretResult run(void) {
+static InterpretResult run(VM *vm, int stopDepth, Value *out) {
 // ...
     case OP_RETURN: {
       Value result = popFromStack(vm);
       closeUpvalues(vm, frame->slots);

       vm->frameCount--;
-      if (vm->frameCount == 0) {
-        popFromStack(vm);
-        return INTERPRET_OK;
-      }
+      if (vm->frameCount == stopDepth) {
+        vm->stackTop = frame->slots;
+        if (out != NULL)
+          *out = result;
+        return INTERPRET_OK;
+      }
```

A new `vmCall(vm, callee, argc, argv, &result)` puts the callee and the
arguments on the stack, calls the existing `call()`, which already checks the
argument count, and runs. A prototype of this on the current code took about
0.1 µs per call, and left the stack and the frames empty afterwards ([A.7]).
Whether compiling a snippet for every call could do instead is
[Q-call-by-source].

**Values.** A `KrbValue` is the VM's own value. Numbers, booleans and nil are
whole on their own. Strings, arrays, instances and functions live on the heap,
and the collector frees whatever it cannot reach from its starting points (the
"roots": the stack and the globals). A value held in a C variable is not a
root, so the host needs a way to say "keep this". There are two tools:

- **Scopes.** Every heap value the interface makes or returns, such as a new
  string, the result of a call, or a global fetched by name, goes on a list that
  the collector treats as a root. `krbScopeBegin` notes how long the list is,
  and `krbScopeEnd` cuts it back to that length. Between the two, everything
  made stays alive. A game loop wraps each frame in one, as the example above
  does.
- **Pins.** `krbPin(k, value)` keeps one value alive until `krbUnpin`. It is for
  what the host stores for longer than a frame: a callback, or an object a
  script gave it.

The rule is short: **anything the interface hands you is safe until the scope
it was made in ends. Pin it to keep it longer.** A host that forgets to end a
scope leaks memory. It does not crash, which is why this is the proposed
design and not a stack of values the host indexes into ([Q-value-api]).

A few accessors read values: `krbToNumber`, `krbToBool`, `krbToString`,
`krbTypeOf`, `krbArrayLength`, `krbArrayGet` and `krbArrayPush`, plus
`krbGetField` and `krbSetField`, which only touch `pub` fields, as script code
outside the struct would. The host can make strings and arrays with `krbString`
and `krbArray`. A number carries one of ten kinds, so `krbToNumber` reads an
`f64`, `krbToInt` reads an `i64`, and both refuse a value of another kind
rather than read its bits wrongly ([A.3]). `krbNumber` and `krbInt` make
them.

**Finding and calling.** These are the calls:

```c
KrbStatus krbGetGlobal(Kirby *k, const char *name, KrbValue *out);
KrbStatus krbSetGlobal(Kirby *k, const char *name, KrbValue value);
KrbStatus krbCall(Kirby *k, KrbValue callee, int argc, const KrbValue *argv,
                  KrbValue *result);
KrbStatus krbCallGlobal(Kirby *k, const char *name, int argc,
                        const KrbValue *argv, KrbValue *result);
KrbStatus krbInvoke(Kirby *k, KrbValue receiver, const char *method, int argc,
                    const KrbValue *argv, KrbValue *result);
```

`krbInvoke` calls a method on an instance, or a static method on a struct
(`Vec2.new`). `result` may be `NULL` when the host does not want it. A global
that the host sets has no type the checker knows, so code that reads it is
unchecked, the same as a native with no signature ([A.3]).

The `interpretMain(void)` that the [Top-Level Declarations Proposal] adds
becomes `krbCallGlobal(k, "main", 0, NULL, &result)`. It needs no function of
its own.

Test first: `unit/call.c` checks each of these:

- a number in and a number out;
- a string in and a string out;
- an array argument;
- a closure stored in a global and then called;
- a method call, and a static method call;
- a struct field read;
- a wrong argument count returns an error, and the instance still works;
- a host function that calls back into the script, twice, nested;
- a pinned callback survives a collection.

The debug library is built with `DEBUG_STRESS_GC`, a collection at every
allocation, so a value that is not rooted correctly fails these tests at once.

#### Part 6: Host functions the checker knows about

```c
bool krbDefineFunction(Kirby *k, const char *name, KrbFn fn,
                       const char *signature);
```

- `name` starts with `@`, as every native's does. A name that is already taken,
  by a built-in or by an earlier host function, is refused, so a host cannot
  quietly replace `@strLen`. See [Q-host-names].
- `fn` has the same shape as a native. It reports a problem with
  `return krbRaise(k, "message")`, which is [Part 3]'s `raise`.
- `signature` is a function type in Kirby's own type syntax:
  `"fun (f64, f64) => f64"`. Built-in natives already work this way: each has a
  string such as `"fun ([T], T) => unit"` in `src/native_signatures.c`, read
  by `parseTypeText` (`src/parser.c`) and resolved by
  `typchkResolveSignatureType` (`src/typecheck.c`). A host signature goes
  through the same two calls. It can name anything a type annotation can, plus
  the names only signatures may use: `T` to `W`, `number` and `integer`.
  `NULL` means "unchecked", as an unsigned native is today. See
  [Q-signatures].

The built-in table (58 entries) is registered into each new instance by the
same code, for the libraries the instance has ([Part 7]). It stays a plain
table. A bad built-in signature stops the program at startup today, and a unit
test checks every one. A bad host signature makes `krbDefineFunction` return
`false` instead.

**Unknown `@` names become a compile error.** Today `@nothing(1)` compiles and
fails when it runs with `Undefined variable '@nothing'` ([A.3]). The instance
knows every native name it has, built-in and host, so the checker can refuse
the rest at compile time. That matters to a host: a script that calls a
function this version of the game does not have fails when it loads, not in the
middle of play. It is a change for `krb` too, exit code 65 instead of 70, for a
program that could never have worked. No test in `tests/` depends on the old
behavior. Plain unknown names such as `somethingUndefined` stay runtime errors,
as they are today ([A.3]). That is a different question.

Test first: `unit/host_functions.c` defines a function with a signature and
checks that a call with the wrong argument type is a compile error, that a
correct call works, that a duplicate name is refused, and that a function
defined in one instance is unknown in another. A new E2E test,
`tests/run/native_functions/native_fn_unknown_name.krb`, checks the compile error.

#### Part 7: What a script may use, and how much

**Libraries.** The 73 natives fall into six groups. The host chooses which
groups an instance gets. Only those are defined in the VM and known to the
checker. `krb` gets all of them.

| Group    | Natives                                                                                                                                                                                |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `BASE`   | `@print @println @eprint @eprintln @strConcat @numberToString @boolToString @typeof @is @isNumber @isFunction @isBool @isString @isNil @instanceOf @assert @panic @version` (18)       |
| `MATH`   | `@ceil @floor @round @trunc @abs @sqrt @pow @min @max @rand @rand01 @randBetween @parseInt @parseFloat` (14)                                                                           |
| `STRING` | `@strLen @strIsEmpty @strContains @strStartsWith @strEndsWith @strTrim @strToUpper @strToLower @strRepeat @strSplit @strIndexOf @strSlice @strReplace @strReplaceAll @strToBytes` (15) |
| `ARRAY`  | `@arrLen @arrPush @arrPop @arrInsert @arrRemove @arrClear @arrContains @arrCopy @arrIsEmpty @arrEqual @arrSlice @arrConcat @arrReverse @arrJoin` (14)                                  |
| `IO`     | `@readFileToString @writeStringToFile @fileExists @prompt @stdin` (5)                                                                                                                  |
| `OS`     | `@getenv @setenv @exit @argv @argc @clock @now` (7)                                                                                                                                    |

Every native is in exactly one group. `@strConcat`, `@numberToString` and
`@boolToString` are in `BASE` because the compiler calls them for string
interpolation (`$"x={x}"`), so a script needs them even if it never names
them. This is a first cut, and the default is [Q-libs].

**Limits.** These are settings on the instance:

| Setting                       | Default   | Meaning                                                                      |
| ----------------------------- | --------- | ---------------------------------------------------------------------------- |
| `maxFrames`                   | 64        | The deepest chain of calls. The value stack is sized from it ([Part 2]).     |
| `maxHeapBytes`                | 0 (none)  | Upper limit on the heap that objects use, as counted by `reallocate`.        |
| `interrupt`, `interruptEvery` | none      | A function the VM calls now and then, and how often (see "The check" below). |
| `alloc`                       | `realloc` | A function that takes the place of `realloc` and `free`.                     |

**The check.** A counter drops by one at each backward jump (`OP_LOOP`), each
call (`OP_CALL`) and each method call (`OP_INVOKE`). Nothing else is counted,
because a script cannot run for long without doing one of the three. When the
counter reaches zero the VM calls the host's `interrupt` function. The host
answers "keep going", and the counter is refilled, or "stop", and the script
ends with the runtime error `Script interrupted.`. A host that wants a time
limit reads the clock inside `interrupt`. The same place checks memory: if the
heap is still over `maxHeapBytes` after a collection, that is the runtime error
`Memory limit exceeded.`. A single native that allocates a huge block, such as
`@strRepeat`, can go over the limit before the next check ([Q-oom]).

The cost was measured ([Q-check-cost]) and is about 1% or less. The debugger's check
is a different, heavier one that runs before every instruction. The two can
share one flag test later.

**The allocator.** `KrbAllocFn(void *user, void *ptr, size_t oldSize, size_t
newSize)`, the same shape as Lua's, takes the place of `realloc` and `free`
inside `reallocate()`, the one function every heap object is allocated through.
The helper buffers that call `malloc` directly (the collector's list of objects
still to visit, the loader's index array, `StrBuf`, three places in
`native.c`, and the compiled-unit code) move to the same function, so that the host sees all of the runtime's
memory and `maxHeapBytes` counts it. The compiler and the type checker keep using
`malloc`, because a shipped game does not run them ([Part 9]).

Test first: `unit/limits.c` runs a `while (true)` loop and a method that calls
itself forever, stopped by `interrupt`. It grows an array in a loop against
`maxHeapBytes`, and it sets a small `maxFrames`. `unit/sandbox.c` compiles a
call to `@readFileToString` in an instance without `IO`, and checks that it is a
compile error, and that `@getenv` is the same without `OS`.

#### Part 8: A public header and a library that is easy to link

- **`include/kirby.h`** holds the whole interface and includes only standard
  headers. It has `extern "C"` guards, since game engines are often C++. It has
  a version macro, `KRB_API_VERSION`, and a function, `krbVersion()`. How
  stable the header is, is [Q-abi]. The comment on every function says whether it can run script code or allocate,
  because those calls can start a collection.
- **Names.** Every exported symbol starts with `krb`. The shared library is
  built with hidden visibility and exports only those. It exports 256 symbols
  today ([A.9]).
- **CMake.** `kirby` (static and shared) is built without readline.
  `configure_kirby_target` links `readline` into every target, and only `main.c`
  uses it. `src/` stops being a public include directory. `kirby_runtime` from
  [Part 1] stays as its own target.
- **`krb` uses the interface.** `main.c` creates an instance and calls
  `krbLoad` and `krbEval`. Its own commands (`lex` prints tokens, `parse` prints the
  syntax tree, `compile` and `build` compile) still use the front end directly.
- **An example and a guide.** `examples/embed/` holds the game loop above with a
  stub world, and `docs/KIRBYLIB.md` explains how to use the interface.

Test first: `unit/public_header.c` includes only `kirby.h`, builds under
`-std=c99 -Wall -Wextra -Wpedantic`, makes an instance and runs a script. A
check in `scripts/` fails if `nm -D` on the shared library lists a name that
does not start with `krb`.

#### Part 9: Compiled scripts and a runtime-only build

A shipped game does not need the parser, the type checker or the compiler. They
are 142,578 bytes of code against 73,240 for the runtime ([A.4]).

**A file for compiled code.** `CompiledUnit` is already plain data. Its records
link to each other by index and offset, never by pointer (`src/compiled_unit.h`),
and `src/unit_bytes.c` writes and reads it (`unitEncode` and `unitDecode`).
The interface gets two calls:

```c
KrbStatus krbCompileToBytes(Kirby *k, const char *name, const char *source,
                            size_t length, KrbBytes *out);
KrbStatus krbLoadBytes(Kirby *k, const char *name, const void *bytes,
                       size_t length);
```

`krb compile file.krb` gets an output, `-o file.krbc`, and `krb run file.krbc`
runs a compiled file, recognized by its first bytes. `krb compile` exists today
without the output: it type checks and compiles a file, runs nothing, and prints
`Compiled!`. This overlaps with the "check without
running" question in the [Top-Level Declarations Proposal] ([Q-check-only]) and
should be settled with it.

**The format** is the one `unitEncode` writes today, a header followed by the
unit:

- the magic bytes `KRBC`;
- a format version (`UNIT_BYTES_FORMAT_VERSION`, 3), and the Kirby version
  from `VERSION.txt`. A file made by a different version is refused, because
  bytecode is not portable between versions;
- the string blob, then the list of functions. Each function has its arity,
  its upvalue count, its flags, its name, its code, its line for each code byte,
  its constants (a kind and a value, where a function constant is an index),
  and its upvalue records.

Numbers are fixed width and little-endian, and doubles are IEEE-754 eight-byte
values. The reader checks every length and index as it reads. A truncated unit,
a wrong magic number, a wrong format version, a unit from another Kirby
version and an index or count out of range are each refused with their own
status, not a crash.

**Bytecode is not verified.** The reader checks that the file is well formed. It
does not check that the code in it is safe to run. A file made by hand can jump
outside its own code, or pop a stack that is empty, and crash the VM. So the
host turns bytecode loading on with `allowBytecode`, which is off by default,
and only loads bytes it made. A safe loader for untrusted bytecode is out of
scope.

**A runtime-only build.** `kirby_runtime` ([Part 1]), which already holds the
reader, is a complete library for running compiled scripts, about 73 KB of
code against 216 KB for everything. `krbLoad` and `krbEval` from source are not built into
it, and return an error saying so, selected by a build flag, `KRB_NO_FRONTEND`.

**What the file carries** from the [Tooling Data Proposal] (names of files,
local variables, and so on) is that proposal's [Q-strip]. The embedded library side of
that question is that a shipped game wants the least, and a game that ships mod
tools may want the most. So the host chooses when it compiles.

**What `krb build` already does with it.** `krb build FILE -o OUT` compiles
the stdlib and the file and attaches both units to a copy of `krb-runtime`, an
executable made from `kirby_runtime` ([Part 1]). Each unit is its length and
then its bytes, and a 16-byte trailer ends the file: the text `KRBAPP01` and
the length of the units (`src/packaged_program.h`). `krb-runtime` reads the
trailer from its own file, decodes each unit, and interprets them in order. It
is a file for a program, not a call for a host. The separate `kirby_runtime`
target does the job of `KRB_NO_FRONTEND` for that executable. What this part
adds is `krbCompileToBytes` and `krbLoadBytes`, `.krbc` files,
`krb compile -o`, `krb run file.krbc`, `allowBytecode` and `KRB_NO_FRONTEND`.

Test first: `unit/unit_bytes.c` already writes, reads and compares units, and
has a case for every truncation, a wrong magic number, a wrong format version
and a wrong Kirby version. `scripts/tests-packaged.sh` builds each test of a
fixed list of 19 with `krb build`, runs it, and compares it with the test's
own snapshots. This part adds a test that `krbLoadBytes` runs a unit the same
as its source, and extends the packaged run from the fixed list to every test
in `tests/`.

## Impacts

### Existing Syntax Or Behavior

- **No change to the language.** No syntax changes, and no program changes
  what it means.
- **`krb` output and exit codes stay the same,** with these exceptions:
  - A call to an `@` name that does not exist is a compile error (exit 65)
    instead of a runtime error (exit 70) ([Part 6]).
  - `@stdin(1)` and `@prompt(1)`, which crash today (exit 139), report their
    error and exit with 70 ([Part 3]).
- **`@exit` still ends `krb` with the code it was given.** It now goes through a
  status and not straight to `exit()` ([Part 3]).
- **Random numbers.** `@rand` and its two siblings come from the instance's own
  generator, seeded by `krb` from the clock. The numbers are different from run
  to run, as they are today. They are not the sequence `rand()` produced.
- **Inside the C code:**
  - `pushOnStack`, `popFromStack` and `popNFromStack` take a `VM *` (91 call
    sites).
  - `interpret()` and `interpretFunction()` are given the instance.
    (`loadUnit` already takes one.)
  - The 16 checks in `asserts.c` return `bool`, and natives use `REQUIRE`.
  - Three unit tests that use the global `vm` are updated.
- **Speed.** The check in [Part 7] costs about 1% or less ([Q-check-cost]). The
  prototype kept `vm` as a global, so it did not measure the cost of reaching
  the VM through a pointer in `run()` ([Part 2]). That has to be measured when
  Part 2 lands, with the same program.
- **Docs.** A new `docs/KIRBYLIB.md`, and entries in `docs/CHANGELOG.md`. The
  step-by-step guide for adding a native function needs two more steps (use
  `raise` and `REQUIRE`, never `exit`; choose a library group).

### Related Proposals

- [Lists Proposal] — splits the `@arr*` natives into `@list*` (growable lists) and a smaller `@arr*` set (fixed size arrays). The `ARRAY` group in its table of native groups would become a `LIST` group and a new `ARRAY` group, and `KRB_LIB_ARRAY` would follow.
- [Top-Level Declarations Proposal] — `interpretMain` becomes a call to the
  general `krbCallGlobal` ([Part 5]). A script an embedded host loads is what
  that proposal calls a library file: it declares things and is never run, so it
  has no `main`. `krbLoad` follows the file rules and `krbEval` the snippet
  rules. Until that proposal lands, both accept statements. The compile-only
  mode of [Part 9] (`krb compile` today, which would gain `-o`) and the
  check-only question ([Q-check-only]) should be settled together. That
  proposal is updated to say so.
- [Modules Proposal] — the bytecode file of [Part 9] is the container that
  proposal's compiled modules need ([Q-interface]), and its interface data would
  go beside the unit in the same file. Whether host functions become a module
  is [Q-host-names]. That proposal is updated to say so.
- [Tooling Data Proposal] — its Part 5 adds a source name to `parse`, `compile`
  and `compileSource`. For an embedded host the name is whatever the host passes
  to `krbLoad`, so it need not be a file. The five diagnostic functions of
  [Part 4] are where structured spans would be reported. The size of shipped
  scripts ([Part 9]) is evidence for its [Q-strip]. That proposal is updated to
  say so.
- [Debugger Proposal] — the debugger's check before each instruction and this
  proposal's check at loops and calls are both in the loop of `run()`
  ([Q-check]). The output hook lets a debug session send a script's
  `@println` output to its console instead of sharing stdout. The order of error message,
  trace, debugger hook and unwinding ([Part 3]) is kept as that proposal
  describes it. Its idea of marking the stdlib as library code fits the built-in
  stdlib of [Part 2], which has its own source name. That proposal is updated to
  say so.
- [Testing Proposal] — `krb test` can run tests in the same process through
  this interface. Its option (c), finding tests by name ([Q-register]), needs to
  list the globals of an instance, which is a small addition to [Part 5]. That
  proposal is updated to say so.
- [Prefixed Native Functions Proposal] (Closed) — the `@` prefix is how host
  functions are named ([Part 6]). That proposal deferred natives-versus-modules
  to the [Modules Proposal], and [Q-host-names] follows it. Not edited.
- [Additional Native Functions Proposal] (Closed) — natives added by that
  proposal are in the library groups of [Part 7]. Any native added later must be
  put in one. Not edited.
- [Collection Methods Proposal] and [Primitive Impls Proposal] — a native that
  becomes a method stops being a global function, and drops out of its library
  group. Whether methods are always available or also grouped is [Q-libs]. Not
  edited.
- [Sized Number Types Proposal] (Closed) — every number carries one of ten
  kinds at run time, and a whole number literal is an `i64`. So host
  signatures ([Part 6]) name number kinds, and `KrbValue` needs accessors that
  check the kind ([Part 5]). Not edited.
- [Generic Types Proposal] — each specialization is its own compiled code, so
  generic programs make bigger bytecode files ([Part 9]). It already names the
  size of an embeddable language as a cost. Not edited.
- [Macros Proposal] — macros run in the front end, so a runtime-only build has
  none at run time, which is what one would expect. Not edited.
- [Projects Proposal] — no change. A host decides where its scripts come from,
  and a script's source name can be a path in the game's own asset system.

### Testing Plan

How do we know the implemented proposal works?

**The existing suite is the guard.** After every part the 1,122 tests in
`tests/` must pass unchanged (3,366 checks). This matters most for [Part 2],
[Part 3] and [Part 4], which rewrite how the VM reaches its state, reports
errors and prints. The 180 tests that expect exit code 70 and the 499 that
expect 65 check that the messages are the same. The changes listed in
[Impacts] are the only ones allowed to change a snapshot.

#### E2E Tests

The E2E tests cover what `krb` shows. The embedded library interface itself cannot be
reached from a `.krb` file, so most of the new testing is C, below.

##### NEW: tests/run/native_functions/native_fn_unknown_name.krb

```kirby
@println(@doesNotExist(1));
```

###### Expected Outcome

A compile error that names `@doesNotExist`, exit code 65, and nothing on
stdout. Today this compiles and fails when it runs with exit code 70. The exact
wording is settled by the snapshot.

##### NEW: tests/run/native_functions/native_fn_stdin_call_non_string_arg.krb

```kirby
@println(@stdin(1));
```

###### Expected Outcome

`input() argument must be a string.`, then the trace line `[line 1] in script`,
and exit code 70. Today the same message is printed and the process then crashes
with exit code 139.

#### Unit Tests

Unit tests live in `unit/` and are run by `scripts/unit.sh`. The debug library
they link (`kirby_core_debug`) is built with `DEBUG_STRESS_GC`, so a value that
the collector cannot see is found at once.

| File                    | Part | Checks                                                                                                                                 |
| ----------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `unit/runtime_link.c`   | 1    | The runtime files alone link, and `initVM` and `freeVM` run.                                                                           |
| `unit/instances.c`      | 2    | Two instances keep separate globals. A host's `srand` sequence is untouched by `krbNew`. The same seed gives the same `@rand` numbers. |
| `unit/errors.c`         | 3    | Every failing native returns an error and the instance still works (sketch below).                                                     |
| `unit/output_hook.c`    | 4    | Output, runtime errors and compile errors reach the hooks with the text `krb` prints.                                                  |
| `unit/call.c`           | 5    | Values in and out, closures, methods, nested calls, pins, a wrong argument count.                                                      |
| `unit/host_functions.c` | 6    | Signatures are checked, duplicates are refused, functions belong to one instance.                                                      |
| `unit/limits.c`         | 7    | `interrupt`, `maxHeapBytes` and `maxFrames` stop a script and leave the instance usable.                                               |
| `unit/sandbox.c`        | 7    | A native from a group the instance lacks is a compile error.                                                                           |
| `unit/public_header.c`  | 8    | `kirby.h` alone is enough to build a host, under `-std=c99 -Wall -Wextra -Wpedantic`.                                                  |
| `unit/unit_bytes.c`     | 9    | Write, read, run. A truncated, foreign or wrong-version file is an error, not a crash. Exists today; Part 9 adds `krbLoadBytes`.       |

##### NEW: unit/errors.c

```c
static void test_native_failures_do_not_end_the_process(void) {
  const char *failing[] = {
      "let a: [i64] = []; @arrPop(a);",
      "@getenv(\"KIRBY_NOT_SET\");",
      "@panic(\"boom\");",
      "@assert(false, \"nope\");",
      "@readFileToString(\"/no/such/file\");",
      "@println(@argv(\"x\"));",
      "@println(@stdin(1));",
      "@println(@prompt(1));",
  };

  KrbConfig config;
  krbConfigInit(&config);
  config.libs = KRB_LIB_ALL;

  Kirby *k = krbNew(&config);

  for (size_t i = 0; i < sizeof failing / sizeof failing[0]; i++) {
    assert(krbEval(k, "<test>", failing[i], strlen(failing[i])) ==
           KRB_RUNTIME_ERROR);
    assert(strlen(krbErrorMessage(k)) > 0);
  }

  assert(krbEval(k, "<test>", "@println(1);", 12) == KRB_OK);

  krbFree(k);
}
```

###### Expected Outcome

The test cannot be built against today's code. Once it can be, it fails on
every one of the eight snippets until [Part 3] is done, because each one ends
the test process (or, for `@stdin(1)` and `@prompt(1)`, crashes it).

## Questions

### **Q:** Is it enough for a host to compile a snippet for every call?

<!-- [Q-call-by-source]: #q-is-it-enough-for-a-host-to-compile-a-snippet-for-every-call -->

**Status:** Answered

A host can call a script function today by compiling and running a snippet like
`update(0.016);`. If that were good enough, [Part 5] would not be needed.

#### Answer

No. It is not too slow: about 3 µs per call, against about 0.1 µs for a direct
call (about 30 times faster, both measured with `-O3`, [A.6] and [A.7]). The reasons
are what it cannot do:

- The host cannot get a value back: `interpret()` returns only a status.
- The host cannot pass a value except by writing it into source text. A string
  with a quote in it changes the program that is compiled. A player named
  `Bob"); @exit(9); say("` ended the test host with exit code 9.
- Every call parses, type checks and compiles again.

### **Q:** What does the check for runaway scripts cost?

<!-- [Q-check-cost]: #q-what-does-the-check-for-runaway-scripts-cost -->

**Status:** Answered

The check in [Part 7] runs at loops and calls only, not before every
instruction. It was measured the way the [Debugger Proposal] measured its own
([Q-check]): a Release build (`-O3 -DNDEBUG`), 25 runs alternating between the
two builds, medians, one machine ([A.7]).

| Program                                                   | Today   | With the check | Difference |
| --------------------------------------------------------- | ------- | -------------- | ---------- |
| `bench.krb` (3-million-trip `while` and `fib(30)`), run 1 | 1.066 s | 1.090 s        | +2.2%      |
| `bench.krb`, run 2                                        | 1.029 s | 1.007 s        | -2.2%      |
| `zoo.krb`, made of method calls (sum to 20 million)       | 3.693 s | 3.722 s        | +0.8%      |

"With the check" is the diff in [A.7]: the check at `OP_LOOP`, `OP_CALL` and
`OP_INVOKE`, together with the change to `run()` that [Part 5] needs. The two
`bench.krb` runs differ by as much in each direction, so the noise on this
machine is about the size of these differences, and the honest reading is
"about 1% or less". The prototype kept the global `vm`, so it does not cover
[Part 2].

#### Answer

About 1% or less, on a loop of cheap instructions, which is the worst case.

### **Q:** Should a native report an error by returning, or by jumping out?

<!-- [Q-error-model]: #q-should-a-native-report-an-error-by-returning-or-by-jumping-out -->

**Status:** Open

Today a native ends the process. [Part 3] replaces that. There are three ways.

- **(a) Raise and return** (proposed). `raise` sets a flag and returns nil, and
  the native returns at once. The VM checks the flag after each native call. It
  is plain C with no hidden jumps. A host function written in C++ can use it
  safely, because nothing skips a destructor. The cost is that every native must
  return early, and one that forgets carries on with a bad argument. `REQUIRE`
  makes the early return one short line.
- **(b) Jump out.** C has a pair of functions, `setjmp` and `longjmp`, that save
  a spot and later jump straight back to it from any depth, skipping everything
  in between. Lua works this way. Natives would not change, because a failed
  check would simply never return, and running out of memory could use the same
  path ([Q-oom]). But a jump skips cleanup, so a native holding a buffer leaks
  it. Worse, jumping over C++ code that has destructors is undefined, and game
  engines are often written in C++, so a host function that calls `krbRaise`
  could not be written safely.
- **(c) A status from every native.** Each native would return `bool` and put
  its result in a pointer. It is the most explicit and checkable, and every one
  of the 73 natives and every `return` in them would change.

### **Q:** How does the host hold script values safely?

<!-- [Q-value-api]: #q-how-does-the-host-hold-script-values-safely -->

**Status:** Open

The collector can free a value that only a C variable refers to. [Part 5] gives
the host a rule. There are three shapes for it.

- **(a) Scopes and pins** (proposed). Everything the interface hands out is
  safe until the scope ends, and a pin keeps a value longer. A host that forgets
  to end a scope leaks. It does not crash.
- **(b) A stack of values, as in Lua.** The host pushes values, calls with a
  count, and reads results by position (`-1` is the top). It is proven, and the
  collector always sees the stack. But a wrong position or count is a crash or a
  wrong value, and code that uses it is harder to read.
- **(c) Every value is a handle** the host must release. It is the simplest to
  get right, and every call and result needs a release, which is most of the
  code a host writes.

### **Q:** What names may host functions have?

<!-- [Q-host-names]: #q-what-names-may-host-functions-have -->

**Status:** Open

The `@` prefix marks a native, and user code cannot define such a name
([Prefixed Native Functions Proposal]). A host function is a native, so it
fits. The open part is what happens on a clash and when modules exist.

- **(a) The `@` names, and a clash is refused** (proposed for now). If a later
  Kirby adds a built-in `@spawn`, a host that already defined one gets an error
  from `krbDefineFunction` at start-up, with a clear message. It is not silent.
- **(b) A sub-namespace,** such as `@game.spawn`. It needs a rule for `.` in a
  native's name, which looks like property access.
- **(c) A host module** once the [Modules Proposal] has namespaces:
  `engine.spawn(...)`. It fits how `std.len` was imagined in the [Prefixed
  Native Functions Proposal]'s own question, and it needs modules first.

### **Q:** How are host function types written?

<!-- [Q-signatures]: #q-how-are-host-function-types-written -->

**Status:** Open

- **(a) A string in Kirby's type syntax** (proposed): `"fun (f64) => f64"`. It
  needs no new syntax, and it is how built-in natives are typed today
  (`src/native_signatures.c`, read by `parseTypeText` and checked at startup).
  A signature can only name types that exist when the function is defined. A
  struct that a script declares later cannot be named, so the first version
  would allow the built-in types, arrays such as `[f64]`, function types, and
  the signature-only names `T` to `W`, `number` and `integer`.
- **(b) A declaration file:** a file of functions with no body, such as
  `fun @spawn(kind: string, x: f64, y: f64): f64;`. TypeScript and Teal describe
  a host's API this way, and the OpenMW game engine's Teal scripting does the
  same for its game API ([OpenMW's Teal page]). An editor could read the same
  file for hovers and completion. It needs new syntax at the top level (trait
  methods already have this shape), and more work.

Strings can turn into declarations later without breaking anything, so (a) first.

### **Q:** How does a script hold something that belongs to the host?

<!-- [Q-userdata]: #q-how-does-a-script-hold-something-that-belongs-to-the-host -->

**Status:** Open

An enemy, a texture, a sound. Options:

- **(a) A number** (proposed first). The host hands out an id and keeps the real
  object. There is no new kind of value. The host controls lifetime, since the
  collector cannot see an id. A script can do arithmetic on an id, and nothing
  stops it passing a texture id where an enemy id belongs.
- **(b) A new heap object,** `OBJ_USERDATA`: a pointer with a type tag, and
  an optional function to run when the collector frees it. The type checker gets
  an opaque type name, so a script cannot mix them up. It needs collector
  support for cleanup and a new type in the checker.

### **Q:** How should the compiler's shared state be handled?

<!-- [Q-frontend-context]: #q-how-should-the-compilers-shared-state-be-handled -->

**Status:** Open

The compiler and type checker keep 26 file-scope variables ([A.5]).

- **(a) Bundle them and swap one pointer** (proposed). It changes few lines. Two
  instances can be compiled one after another. Compiling on two threads at once
  is not safe.
- **(b) Pass a context pointer to every function** in the 9,200 lines. It is a
  large mechanical diff, and it makes concurrent compiles safe.

Do (a) now. Do (b) if a host needs to compile on several threads.

### **Q:** What does a new instance get, and are the groups right?

<!-- [Q-libs]: #q-what-does-a-new-instance-get-and-are-the-groups-right -->

**Status:** Open

- **The default.** Lua's `luaL_newstate` gives no libraries until the host asks.
  The proposal leans the other way: the safe four (`BASE`, `MATH`, `STRING`,
  `ARRAY`) by default, so a host must ask for `IO` and `OS`. `krb` asks for all.
- **`@clock` and `@rand`.** `@clock` reads CPU time and `@rand` is now seeded
  per instance. A game that replays its own inputs may want neither.
- **Methods.** If the [Collection Methods Proposal] moves array and string
  natives to methods, are methods always there? A sandbox that cannot give a
  script `@arrPush` is not much of a sandbox for it.

### **Q:** What happens when memory runs out?

<!-- [Q-oom]: #q-what-happens-when-memory-runs-out -->

**Status:** Open

Today `reallocate` prints a message and calls `exit()` when `realloc` fails, and
16 other places do the same. A caller of C's allocator cannot recover, because
the code that called it does not check for the failure.

- **(a) Fatal, but tell the host first** (proposed). A host function, `onFatal`,
  is called before the process ends. It may abort, or use a jump of its own.
- **(b) A script error.** It needs every allocation site to handle failure, or
  the jump-out of [Q-error-model] (b).
- **The practical guard** is `maxHeapBytes`, which is checked at loops and
  calls. It cannot stop one huge allocation inside one native, so natives that
  take a size (`@strRepeat`) should refuse absurd ones.

### **Q:** How is a changed script loaded into a running instance?

<!-- [Q-reload]: #q-how-is-a-changed-script-loaded-into-a-running-instance -->

**Status:** Open

Game developers expect to edit a script and see it change. Today the checker
refuses the second load of a name that is already declared: `Already declared
in this scope` for both a function and a struct ([A.10]). Nothing goes stale
silently. But nothing reloads either.

- **(a) Do not reload.** Make a new instance, load the new scripts and move state
  across, through the host (proposed for now).
- **(b) Allow functions to be replaced.** A struct is harder: instances made
  from the old struct keep the old methods.
- **(c) A full swap** that migrates instances. It is a proposal of its own.

### **Q:** Can a script pause and continue on a later frame?

<!-- [Q-coroutines]: #q-can-a-script-pause-and-continue-on-a-later-frame -->

**Status:** Open

Game scripts often want `wait(2)` inside a function. It is out of scope here.
The design should not close the door. A coroutine needs its own stack and
frames, and after [Part 2] those are separate arrays that could move into a
per-coroutine struct. As in Lua, a pause could not cross a host function that
called back into the script, because the host's C code is in the way.

### **Q:** What is safe to do from several threads?

<!-- [Q-threads]: #q-what-is-safe-to-do-from-several-threads -->

**Status:** Open

The proposed promise: one instance is used by one thread at a time, and the
host takes the lock. Different instances on different threads are safe for
running scripts, because [Part 2] leaves no shared state in the VM. Compiling is
safe on one thread at a time until [Q-frontend-context] (b). `krbNew` compiles
the standard library, so it is subject to the same rule. The header should say
so.

### **Q:** How stable is the header?

<!-- [Q-abi]: #q-how-stable-is-the-header -->

**Status:** Open

Kirby is at version 0.4.0. `KrbValue` is the VM's 24-byte value: a type, a
number kind and an 8-byte payload. If it later becomes smaller, every host must
be rebuilt.

- **(a) Source stability only** (proposed while pre-1.0). A host links Kirby
  statically or rebuilds on upgrade. `KRB_API_VERSION` and a size field in
  `KrbConfig` catch a stale header.
- **(b) An opaque value** that the header hides completely. The layout can
  change without touching a host. It costs an indirection on each value.

### **Q:** What does the standard library cost each new instance?

<!-- [Q-stdlib]: #q-what-does-the-standard-library-cost-each-new-instance -->

**Status:** Open

`stdlib/stdlib.krb` holds one struct, `Date`, and its impls. It is compiled
into the binary, and [Part 2] loads it in every `krbNew`. That costs one
compile of about 210 lines per instance. The [String Interpolation Proposal]
puts nothing in it, but other features may.

- **(a) Compile and run it in every new instance** (proposed now). A game that
  makes an instance per entity pays each time.
- **(b) Ship it as bytecode** inside the binary ([Part 9]). It removes the
  compile, and the declarations still run in each instance.
- **(c) Share one loaded stdlib between instances.** It needs objects that
  belong to no single heap. That is a much larger change.

## Appendix A: Checking the claims

Every claim in this document that says Kirby "does" something was run at
`from_commit` (`2f96a2e`) on Linux x86-64 with GCC. Timings come from one
shared machine with two cores, so they show an order of magnitude and not a
benchmark. Nothing here needs CMake. `gcc`, `bash`, `python3` and
`libreadline-dev` are enough.

### Setup

```shell
K=/path/to/kirbylang      # a checkout at from_commit
B=/tmp/kirby-embed        # a scratch folder
mkdir -p $B/h && cd $B

# version.c and stdlib.c are generated at build time. CMake does the same.
bash $K/scripts/generate_version_c.sh $B/version.c $K/VERSION.txt
bash $K/scripts/generate_stdlib_c.sh $B/stdlib.c $K/stdlib/stdlib.krb

# Every source file except main.c and runtime.c, as KIRBY_SOURCES lists them
SRCS="asserts ast chunk common compiler debug hashtable lexer compiled_unit \
definite_assignment gc loader native native_signatures number_kind number_ops \
object opcode packaged_program parser resolved_impl_targets scanner strbuf \
stringset token_stream token typecheck types unit_bytes value vm"
FILES="$(for f in $SRCS; do echo $K/src/$f.c; done) $B/version.c $B/stdlib.c"

# The command line tool
gcc -std=c99 -O1 -I$K/src -I$B $FILES $K/src/main.c -lreadline -lm -o krb

# A test program that uses Kirby as a library (NAME is a file in h/)
gcc -std=c99 -O1 -I$K/src -I$B -Ih $FILES h/NAME.c -lm -o NAME
```

`krb` runs from any folder, because the stdlib is built into it.

**The existing suite passes.** After `just build`, `just test` ends with:

```text
Total: 3366 | Passed: 3366 | Failed: 0 | Skipped: 0 | Suites: 1122
```

One test, `native_fn_argc_and_argv`, prints the path of the binary, so the suite
has to be run with the default `./build/kirby-test`.

The test programs that follow play the part of a game engine. They use only what
Kirby offers today. `h/common_host.h` is the shared harness. It holds a copy of
`compileSource` from `src/main.c`, which is `static` there, and a function that
compiles and runs a piece of source and returns 0 (ok), 1 (compile error) or 2
(runtime error). It does not load the stdlib, which none of the scripts here
use.

```c
/* Test harness: plays the role of a game engine embedding Kirby using only
 * what is public today. compileSource is copied from src/main.c because it
 * is `static` there. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#include "ast.h"
#include "compiler.h"
#include "parser.h"
#include "resolved_impl_targets.h"
#include "typecheck.h"
#include "vm.h"

static CompiledUnit *compileSource(const char *source) {
  resolvedImplTargetsReset();
  int count = 0;
  bool hadError = false;
  int endLine = 0;
  AstNode **ast = parse(source, &count, &hadError, &endLine);
  if (!hadError && !typchkCheckProgram(ast, count)) hadError = true;
  CompiledUnit *unit = hadError ? NULL : compile(ast, count, endLine);
  astFreeAll();
  free(ast);
  return unit;
}

/* Returns 0 ok, 1 compile error, 2 runtime error */
static int runSnippet(const char *source) {
  CompiledUnit *unit = compileSource(source);
  if (unit == NULL) return 1;
  return interpret(unit, false) == INTERPRET_OK ? 0 : 2;
}
```

Script output goes to stdout and the harness writes to stderr. The outputs
below were captured with `stdbuf -o0`, so the two appear in the order they
happen.

### A.1 The pipeline works from C, and script errors are survivable

Claims: [Problem Statement], "What Kirby already has".

```c
#include "common_host.h"
int main(void) {
  initVM(0, NULL);
  typchkSessionBegin();
  fprintf(stderr, "host: snippet 1 -> %d\n", runSnippet("@println(1 + 2);"));
  fprintf(stderr, "host: snippet 2 (runtime error) -> %d\n",
          runSnippet("let a = [1]; @println(a[5]);"));
  fprintf(stderr, "host: snippet 3 (after the error) -> %d\n",
          runSnippet("@println(40 + 2);"));
  fprintf(stderr, "host: snippet 4 (compile error) -> %d\n",
          runSnippet("let x: f64 = \"no\";"));
  fprintf(stderr, "host: snippet 5 (after compile error) -> %d\n",
          runSnippet("@println(7);"));
  fprintf(stderr, "host: sizeof(VM) = %zu bytes\n", sizeof(VM));
  compilerSessionEnd();
  typchkSessionEnd();
  freeVM();
  return 0;
}
```

Output. The lines `3`, `42` and `7` are the scripts' own `@println` output. The `host:` lines are the test program's:

```text
3
host: snippet 1 -> 0
Array index out of bounds.
[line 1] in script
host: snippet 2 (runtime error) -> 2
42
host: snippet 3 (after the error) -> 0
[line 1] Error: Expected f64, got string.
host: snippet 4 (compile error) -> 1
7
host: snippet 5 (after compile error) -> 0
host: sizeof(VM) = 394832 bytes
```

The host survived a runtime error and a compile error, and each was followed by a
script that ran. The last line is the size of the VM struct, used in [A.8].

### A.2 A script's mistake can end the host

Claims: 51 `exit()` calls; native failures end the process; `@stdin(1)` and `@prompt(1)` crash.

```c
#include "common_host.h"
int main(int argc, char **argv) {
  const char *script = argv[1];
  initVM(0, NULL);
  typchkSessionBegin();
  fprintf(stderr, "host: running script...\n");
  int r = runSnippet(script);
  fprintf(stderr, "host: script finished with %d. Host is still alive.\n", r);
  return 0;
}
```

Each script is passed to the test program. Its last message, `host: script finished`, is never printed, and the process's own exit code is the one shown:

```text
$ ./t2_exit 'let a: [i64] = []; @arrPop(a);'
Cannot pop empty array.
[line 1] in script
[process exit code 70]

$ ./t2_exit '@getenv("KIRBY_NOT_SET");'
Environment variable not found: 'KIRBY_NOT_SET'
[line 1] in script
[process exit code 70]

$ ./t2_exit '@panic("boom");'
panic: boom
[line 1] in script
[process exit code 70]

$ ./t2_exit '@assert(false, "nope");'
assertion failed: nope
[line 1] in script
[process exit code 70]

$ ./t2_exit '@readFileToString("/no/such/file");'
File does not exist: '/no/such/file'
[line 1] in script
[process exit code 70]

$ ./t2_exit '@println(@argv("x"));'
function @argv expects argument 1 to be a number.
[line 1] in script
[process exit code 70]

$ ./t2_exit '@exit(3);'
[process exit code 3]
```

`@argv` has no signature, so `@argv("x")` is not stopped by the type checker,
which is how it reaches the VM. `@stdin(1)` and `@prompt(1)` are different.
They report and return:

```text
$ ./t2_exit '@println(@stdin(1));'
input() argument must be a string.
[line 1] in script
[process exit code 139: segmentation fault]

$ ./t2_exit '@println(@prompt(1));'
prompt() argument must be a string.
[line 1] in script
[process exit code 139: segmentation fault]
```

The `exit()` calls outside `main.c` and `runtime.c`, per file
(`grep -c "\bexit(" src/*.c`):

```text
asserts.c:17
ast.c:2
compiled_unit.c:1
compiler.c:1
definite_assignment.c:1
gc.c:2
loader.c:1
native.c:16
native_signatures.c:1
packaged_program.c:1
resolved_impl_targets.c:1
stringset.c:1
token_stream.c:1
typecheck.c:2
types.c:1
unit_bytes.c:2
total: 51
```

That is 17 in `asserts.c`, 16 in `native.c` (one of them is `@exit`), one in
`native_signatures.c` (a built-in signature that does not parse), and 17 for
running out of memory in the other files.

### A.3 A host function works today, and it is unchecked

Claims: `defineNative` works; a host function gets no type checking; its arguments keep the script's number kinds; unknown names are runtime errors.

```c
#include "common_host.h"
#include "native.h"

/* A host function the way an engine would write one today. */
static Value hostAdd(VM *vm, int argCount, Value *args) {
  (void)vm; (void)argCount;
  return NUMBER_F64_VAL(AS_NUMBER_F64(args[0]) + AS_NUMBER_F64(args[1]));
}

int main(int argc, char **argv) {
  initVM(0, NULL);
  typchkSessionBegin();
  defineNative(&vm, "@hostAdd", hostAdd);
  fprintf(stderr, "host: script calls the registered native -> %d\n",
          runSnippet(argv[1]));
  return 0;
}
```

```text
$ ./t3_hostfn '@println(@hostAdd(1.5, 2.5));'
4
host: script calls the registered native -> 0

$ ./t3_hostfn '@println(@hostAdd(1, 2));'
1.5e-323
host: script calls the registered native -> 0

$ ./t3_hostfn 'let x: i64 = @typeof(3); @println(x);'
i64
host: script calls the registered native -> 0

$ ./t3_hostfn '@println(@nothing(1, 2));'
Undefined variable '@nothing'.
[line 1] in script
host: script calls the registered native -> 2
```

The second script passes `1` and `2`, which are `i64` values, and the host
reads them as `f64`s. The third assigns a string to an `i64` and compiles,
because `@typeof` has no signature. The fourth is only found when it runs. A
plain unknown name is the same, a runtime error and not a compile error:

```text
$ ./krb exec '@println(somethingUndefined);'
Undefined variable 'somethingUndefined'.
[line 1] in script
[krb exit code 70]
```

Natives, and natives that have a signature for the type checker (the
`nativeDefinitions` table in `src/native.c` and the `nativeSignatures` table in
`src/native_signatures.c`):

```text
definitions: 73   signatures: 58
no signature: @argv @eprint @eprintln @instanceOf @is @isBool @isFunction
              @isNil @isNumber @isString @print @println @prompt @stdin @typeof
```

### A.4 The runtime half links on its own

Claims: the runtime files link with nothing else; the size of the two halves.

```c
#include "vm.h"
int main(void) { initVM(0, NULL); freeVM(); return 0; }
```

```shell
RT="asserts chunk common compiled_unit gc hashtable loader native number_kind number_ops object packaged_program stringset strbuf unit_bytes value vm"
FE="ast compiler definite_assignment lexer native_signatures opcode parser resolved_impl_targets scanner token token_stream typecheck types"

# 1. Link only the runtime files. What is missing?
gcc -std=c99 -O1 -I$K/src -I$B -Ih $(for f in $RT; do echo $K/src/$f.c; done) \
    $B/version.c h/t4_vmonly.c -lm -o t4 2>&1 | grep "undefined reference"

# 2. Code size of each file (.text, in bytes)
mkdir -p obj
for f in $RT $FE debug; do
  gcc -std=c99 -O2 -DNDEBUG -c -I$K/src -I$B $K/src/$f.c -o obj/$f.o
done
gcc -std=c99 -O2 -DNDEBUG -c -I$K/src -I$B $B/version.c -o obj/version.o
gcc -std=c99 -O2 -DNDEBUG -c -I$K/src -I$B $B/stdlib.c -o obj/stdlib.o
tot() { t=0; for n in "$@"; do s=$(size obj/$n.o | tail -1 | awk '{print $1}'); t=$((t+s)); done; echo $t; }
echo "runtime files (with version.c): $(tot $RT version)   (native.c alone: $(tot native))"
echo "front end:                      $(tot $FE)"
echo "stdlib.c:                       $(tot stdlib)"
echo "debug.c:                        $(tot debug)"
echo "everything above:               $(( $(tot $RT version) + $(tot $FE) ))"
```

Results:

```text
undefined names: (none; t4 links and runs)

runtime files (with version.c): 73240   (native.c alone: 25823)
front end:                      142578
stdlib.c:                       6431
debug.c:                        3208
everything above:               215818
```

The runtime list is `KIRBY_RUNTIME_SOURCES` from `CMakeLists.txt`. `debug.c`
is not needed. `stdlib.c` is the text of the stdlib, which only a build with
the front end can compile.

### A.5 One copy per process

Claims: 28 file-scope variables; 91 call sites of the stack helpers.

Non-constant variables that live for the whole program, per file, from `nm` on the object files of A.4 (`b`/`B` are zero-initialised, `d`/`D` initialised):

```shell
for f in $RT $FE; do
  n=$(nm --defined-only obj/$f.o | awk '$2 ~ /^[bBdD]$/ {print $3}' | grep -v '^\.' | grep -v '\.[0-9]*$' | tr '\n' ' ')
  [ -n "$n" ] && echo "$f.c: $n"
done
```

```text
native.c: nativeDefinitions
number_kind.c: infos
vm.c: gcInstance vm
ast.c: arenaHead arenaOffset
compiler.c: compilingUnit current currentImplTargetName currentLine currentLoop hadError hasCurrentImplTargetName immutableBindings lambdaCount
native_signatures.c: nativeArgConstraints nativeSignatures
parser.c: rules
resolved_impl_targets.c: entries entryCapacity entryCount
typecheck.c: hadError sessionEnv
types.c: anyIntegerSingleton arenaHead arenaOffset boolSingleton numberSingletons selfPlaceholderSingleton stringSingleton typeNames unitSingleton
```

Five of those are not state: `nativeDefinitions`, `infos`,
`nativeArgConstraints` and `nativeSignatures` are `const` tables, and
`parser.c`'s `rules` is a parse table that is never written. The command leaves
out `static` variables inside a function, whose names end in `.N`. There are
two: `printNatives` in `typecheck.c`, a table, and `resolvedImplTargetToken` in
`resolved_impl_targets.c`, which holds state between calls. So there are 28:
`vm.c` 2, `compiler.c` 9, `types.c` 9, `resolved_impl_targets.c` 4,
`typecheck.c` 2, `ast.c` 2. Of those, 26 are in the front end.

The stack helpers reach the global `vm`, including from files that were handed a
`VM *`. Calls per file, not counting the three definitions:

```text
chunk.c:2
loader.c:4
native.c:18
object.c:4
vm.c:63
total: 91

vm.c:144:void pushOnStack(Value value) {
vm.c:149:Value popFromStack(void) {
vm.c:154:void popNFromStack(int count) { vm.stackTop -= count; }
static void resetStack(void) {
  vm.stackTop = vm.stack;
  vm.frameCount = 0;
  vm.openUpvalues = NULL;
}
```

The last lines are from reading the code, not from running it. `resetStack` takes no
argument and changes `vm`, the global, even when `runtimeError` was handed another
`VM *`. `defineNative` reads `vm->stack[0]` and `vm->stack[1]` back
(`src/native.c`, line 84), which is only right when the stack is empty.

### A.6 Calling a script function by compiling source

Claims: the cost of a call made by compiling a snippet; it cannot return a value; it cannot pass one safely.

```c
#define _POSIX_C_SOURCE 200809L
#include "common_host.h"
static double now(void) { struct timespec t; clock_gettime(CLOCK_MONOTONIC, &t); return t.tv_sec + t.tv_nsec / 1e9; }
int main(void) {
  initVM(0, NULL);
  typchkSessionBegin();
  runSnippet("fun update(dt: f64): f64 = dt * 2;");

  int n = 20000;
  double t0 = now();
  for (int i = 0; i < n; i++) runSnippet("update(0.016);");
  double t1 = now();
  printf("calling a script function by compiling a snippet each time: %.1f us per call (%d calls)\n",
         (t1 - t0) / n * 1e6, n);
  return 0;
}
```

```c
#include "common_host.h"
int main(void) {
  initVM(0, NULL);
  typchkSessionBegin();
  runSnippet("var calls = 0; fun update(dt: f64): f64 { calls = calls + 1; return dt * 2; }");
  for (int i = 0; i < 20000; i++) runSnippet("update(0.016);");
  runSnippet("@println(calls);");                    /* did all 20000 calls really run? */

  /* The host wants the return value of update(). What does interpret() give back? */
  CompiledUnit *u = compileSource("update(0.5);");
  InterpretResult r = interpret(u, false);
  printf("interpret() returned status %d; there is no field or function for the value 1.0\n", r);

  /* The host wants to pass a string containing a quote. The only way is to paste it into source text. */
  runSnippet("fun say(s: string): unit { @println(s); }");
  const char *playerName = "Bob\"); @exit(9); say(\"";       /* hostile / unlucky player name */
  char src[256];
  snprintf(src, sizeof src, "say(\"%s\");", playerName);
  printf("source the host would build: %s\n", src);
  fflush(stdout);
  int rc = runSnippet(src);
  printf("host: still alive, rc=%d\n", rc);
  return 0;
}
```

With `-O3 -DNDEBUG`, so the number can be compared with A.7, the cost of a call
made by compiling a snippet, over fifteen runs, in microseconds per call,
sorted: **2.2 2.2 2.3 2.3 2.6 3.3 3.4 3.4 3.4 3.4 3.4 3.4 3.4 3.4 3.5**. The
runs fall into two groups, which is how this shared machine behaves. The median
is 3.4 µs. At `-O1` it is about the same.

```text
20000
interpret() returned status 0; there is no field or function for the value 1.0
source the host would build: say("Bob"); @exit(9); say("");
Bob
[process exit code 9]
```

`calls` reached 20,000, so every call ran. `interpret()` returns a status and has no
place for the value. The last two lines show a player's name, pasted into the
source text, changing the program and ending the host with exit code 9.

### A.7 A direct call, and the check for runaway scripts

Claims: a script function can be called directly with a small change to `run()`; the cost of a direct call; the cost of the check.

This is a throwaway prototype, not the design. It keeps the global `vm`. It changes
`run()` so that it stops at a given frame depth and gives back the result, adds
the check at `OP_LOOP`, `OP_CALL` and `OP_INVOKE` (a counter that ends the script
when it reaches zero), and adds two helper functions:

```diff
--- src/vm.c
+++ src/vm.c
@@ -21,13 +21,15 @@
 static void resetStack(void);
 void runtimeError(VM *vm, const char *format, ...);
 static Value peekStack(int distance);
-static InterpretResult run(void);
+static InterpretResult run(int stopDepth, Value *out);
 static void concatenate(void);
 static bool call(ObjClosure *function, int argCount);
-static InterpretResult run(void);
+static InterpretResult run(int stopDepth, Value *out);

 VM vm;
 GC gcInstance;
+long long krbBudget = 1000000000LL;
+#define CHECK_BUDGET() do { if (--krbBudget <= 0) { runtimeError(&vm, "Script interrupted."); return INTERPRET_RUNTIME_ERROR; } } while (0)

 /**
  * Mark the VM's roots in the garbage collector
@@ -64,7 +66,7 @@
   pushOnStack(OBJ_VAL(closure));
   call(closure, 0);

-  return run();
+  return run(0, NULL);
 }

 InterpretResult interpret(CompiledUnit *unit, bool disassembleCode) {
@@ -580,7 +582,7 @@
   return true;
 }

-static InterpretResult run(void) {
+static InterpretResult run(int stopDepth, Value *out) {
   CallFrame *frame = &vm.frames[vm.frameCount - 1];
 #define READ_BYTE() (*frame->ip++)
 #define READ_CONSTANT()                                                        \
@@ -714,8 +716,9 @@
       closeUpvalues(frame->slots);

       vm.frameCount--;
-      if (vm.frameCount == 0) {
-        popFromStack();
+      if (vm.frameCount == stopDepth) {
+        vm.stackTop = frame->slots;
+        if (out != NULL) *out = result;
         return INTERPRET_OK;
       }

@@ -809,12 +812,14 @@
     case OP_LOOP: {
       uint16_t jumpOffset = READ_SHORT();
       frame->ip -= jumpOffset;
+      CHECK_BUDGET();
       break;
     }
     case OP_CALL: {
       int argCount = READ_BYTE();
       Value callee = peekStack(argCount);

+      CHECK_BUDGET();
       if (!callValue(callee, argCount)) {
         return INTERPRET_RUNTIME_ERROR;
       }
@@ -1089,6 +1094,7 @@
     case OP_INVOKE: {
       ObjString *method = READ_STRING();
       int argCount = READ_BYTE();
+      CHECK_BUDGET();
       if (!invoke(method, argCount)) {
         return INTERPRET_RUNTIME_ERROR;
       }
@@ -1193,3 +1199,15 @@
 #undef READ_CONSTANT
 #undef READ_STRING
 }
+
+InterpretResult krbProtoCall(ObjClosure *closure, int argc, const Value *argv, Value *out) {
+  int base = vm.frameCount;
+  pushOnStack(OBJ_VAL(closure));
+  for (int i = 0; i < argc; i++) pushOnStack(argv[i]);
+  if (!call(closure, argc)) return INTERPRET_RUNTIME_ERROR;
+  return run(base, out);
+}
+bool krbProtoGlobal(const char *name, Value *out) {
+  ObjString *key = copyString(vm.gc, name, (int)strlen(name));
+  return tableGet(&vm.globals, key, out);
+}
```

It applies to a clean copy of `src/`:

```shell
mkdir pcopy && cp -r $K/src pcopy/src
(cd pcopy && patch -p0 < ../prototype-vm.diff)
```

The test program calls the function a million times, checks the results and the stack, and then runs a loop that never ends with a budget of one million checks:

```c
#define _POSIX_C_SOURCE 200809L
#include "common_host.h"
extern long long krbBudget;
InterpretResult krbProtoCall(ObjClosure *closure, int argc, const Value *argv, Value *out);
bool krbProtoGlobal(const char *name, Value *out);
static double now(void) { struct timespec t; clock_gettime(CLOCK_MONOTONIC, &t); return t.tv_sec + t.tv_nsec / 1e9; }

int main(void) {
  initVM(0, NULL);
  typchkSessionBegin();
  runSnippet("var calls = 0; fun update(dt: f64): f64 { calls = calls + 1; return dt * 2; }");

  Value fn;
  if (!krbProtoGlobal("update", &fn) || !IS_CLOSURE(fn)) { puts("no update"); return 1; }

  Value arg = NUMBER_F64_VAL(0.25), result = NIL_VAL;
  InterpretResult r = krbProtoCall(AS_CLOSURE(fn), 1, &arg, &result);
  printf("one direct call: status=%d result=%g (expect 0.5)\n", r, IS_NUMBER_F64(result) ? AS_NUMBER_F64(result) : -1);

  int n = 1000000;
  double t0 = now();
  for (int i = 0; i < n; i++) krbProtoCall(AS_CLOSURE(fn), 1, &arg, &result);
  double t1 = now();
  printf("direct call: %.3f us per call (%d calls)\n", (t1 - t0) / n * 1e6, n);
  fflush(stdout);
  runSnippet("@println(calls);");

  printf("stack depth after all calls: %ld values, frames=%d (expect 0 and 0)\n", (long)(vm.stackTop - vm.stack), vm.frameCount);

  /* runaway script */
  runSnippet("fun spin(): unit { while (true) { } }");
  Value spin; krbProtoGlobal("spin", &spin);
  krbBudget = 1000000;
  fflush(stdout);
  r = krbProtoCall(AS_CLOSURE(spin), 0, NULL, &result);
  printf("runaway loop: status=%d (2 = runtime error, host got control back)\n", r);
  return 0;
}
```

```text
one direct call: status=0 result=0.5 (expect 0.5)
direct call: 0.126 us per call (1000000 calls)
1000001
stack depth after all calls: 0 values, frames=0 (expect 0 and 0)
Script interrupted.
[line 0] in spin()
runaway loop: status=2 (2 = runtime error, host got control back)
direct call, five more runs (us per call, sorted): 0.087 0.089 0.092 0.107 0.116
```

The direct call took between 0.087 and 0.126 µs over these runs, against about
3 µs for compiling a snippet ([A.6]). The stack was empty afterwards, no frame
was left over, and the loop that never ends was stopped with a runtime error
and the host got control back. (The trace line for that error says
`[line 0]`, which is an existing quirk of the line recorded for a loop
instruction. This proposal does not depend on it.)

**Cost of the check.** Two builds with `-O3 -DNDEBUG`, one from `$K/src` and one
from the patched copy, each linked with `main.c`. The programs are the debugger
proposal's `bench.krb`, with its `print` statements written as `@println`, and
a method-call-heavy `zoo.krb`:

```shell
sed 's/sum < 100000000/sum < 20000000/' $K/examples/zoo.krb | grep -v "@clock" > zoo.krb
cp /path/to/bench.krb .     # the program from the Debugger Proposal's Appendix A
```

```python
import subprocess, statistics, sys, time

def run(binary, program):
    start = time.perf_counter()
    subprocess.run([binary, "run", program], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    return time.perf_counter() - start

runs = int(sys.argv[1])
for program in sys.argv[2:]:
    base, patched = [], []
    for _ in range(2):                      # warm up
        run("./krb_base", program); run("./krb_patched", program)
    for _ in range(runs):                   # alternate, so drift hits both builds
        base.append(run("./krb_base", program))
        patched.append(run("./krb_patched", program))
    b, p = statistics.median(base), statistics.median(patched)
    print(f"{program:10} today {b:.3f} s   with check {p:.3f} s   difference {(p - b) / b * 100:+.1f}%")
```

Run as `compare.py 25 bench.krb bench.krb` and `compare.py 15 zoo.krb`. Results:

```text
bench.krb  today 1.066 s   with check 1.090 s   difference +2.2%
bench.krb  today 1.029 s   with check 1.007 s   difference -2.2%
zoo.krb    today 3.693 s   with check 3.722 s   difference +0.8%
```

The output of both builds was identical on both programs.

### A.8 What a script can do to the host's process

Claims: `initVM` reseeds `rand()`; a loop that never ends cannot be stopped; recursion stops at 63; `@println` writes to stdout; the VM struct size.

```c
#define _POSIX_C_SOURCE 200809L
#include "common_host.h"
#include <unistd.h>
int main(int argc, char **argv) {
  const char *which = argv[1];
  if (strcmp(which, "srand") == 0) {
    /* The engine seeds C's random generator for its own use (replays, procedural levels). */
    srand(12345);
    int expectedFirst;
    { srand(12345); expectedFirst = rand(); srand(12345); }
    initVM(0, NULL);                 /* just creating the VM */
    int actualFirst = rand();
    printf("engine seeded rand() with 12345; first value should be %d, got %d after initVM\n", expectedFirst, actualFirst);
  } else if (strcmp(which, "loop") == 0) {
    initVM(0, NULL); typchkSessionBegin();
    printf("host: running a script with an infinite loop...\n"); fflush(stdout);
    runSnippet("while (true) { }");
    printf("host: script returned (never printed)\n");
  } else if (strcmp(which, "depth") == 0) {
    initVM(0, NULL); typchkSessionBegin();
    runSnippet("fun depth(n: i64): i64 { if (n == 0) { return 0; } return 1 + depth(n - 1); }");
    printf("depth(60): ");  fflush(stdout); runSnippet("@println(depth(60));");
    printf("depth(62): ");  fflush(stdout); runSnippet("@println(depth(62));");
    printf("depth(63): ");  fflush(stdout); runSnippet("@println(depth(63));");
    printf("depth(100): "); fflush(stdout); runSnippet("@println(depth(100));");
  } else if (strcmp(which, "print") == 0) {
    initVM(0, NULL); typchkSessionBegin();
    /* Is there any way to get script output into the engine's console other than OS-level tricks? */
    runSnippet("@println(\"hello from script\");");
  }
  return 0;
}
```

```text
$ ./t8_side srand
engine seeded rand() with 12345; first value should be 383100999, got 370415393 after initVM

$ timeout 3 ./t8_side loop
host: running a script with an infinite loop...
[exit code 124, 124 means killed by timeout]

$ ./t8_side depth   (the trace of the last call is cut here)
depth(60): 60
depth(62): 62
depth(63): Stack overflow.
[line 1] in depth()
[line 1] in depth()

$ ./t8_side print   (stdout and stderr captured apart)
stdout: hello from script
stderr: ''
```

The value printed after `initVM` changes from second to second, because
`initVM` seeds from the clock. The number the host expected is not.

`FRAMES_MAX` is 64 and the script itself takes one frame, so 63 nested calls fit
(`depth(62)` calls `depth` 63 times) and 64 do not (`depth(63)`). From reading
the source:

```text
src/vm.c:113:  srand(time(NULL));
src/native.c:277: static Value printlnNative(VM *vm, int argCount, Value *args) {
src/native.c:278-   assertArgCount(vm, "@println", 1, argCount);
src/native.c:279-   printValue(args[0]);
src/native.c:280-   printf("\n");
src/vm.c:90:  vfprintf(stderr, format, args);
src/vm.c:102:      fprintf(stderr, "script\n");
src/vm.h:11:#define FRAMES_MAX 64
src/vm.h:12:#define STACK_MAX (FRAMES_MAX * UINT8_COUNT)
```

`sizeof(VM)` is 394,832 bytes, printed at the end of [A.1]. Almost all of it is
the value stack: 16,384 values of 24 bytes is 393,216 bytes. A value is 24
bytes because a number holds its kind next to an 8-byte payload.

### A.9 The shared library

Claims: the shared library exports 256 symbols, among them general names.

```shell
gcc -std=c99 -O1 -fPIC -shared -I$K/src -I$B $FILES -lm -o libkirby.so
nm -D --defined-only libkirby.so | wc -l
nm -D --defined-only libkirby.so | grep -E " (vm|gcInstance|initVM|parse|compile|hashBytes|defineNative|KIRBY_STDLIB)$"
```

```text
exported symbols: 256
000000000002bbc0 R KIRBY_STDLIB
000000000000e42b T compile
0000000000014c45 T defineNative
0000000000036fe0 B gcInstance
000000000000bfc0 T hashBytes
00000000000244d9 T initVM
000000000001a20e T parse
0000000000037040 B vm
```

CMake builds `kirbylib` from the same list of files, and `nm -D` on its
`build/libkirby.so` also counts 256.

### A.10 Loading the same script twice

Claim: the second load of a declared name is refused.

```c
#include "common_host.h"
int main(void) {
  initVM(0, NULL);
  typchkSessionBegin();
  const char *v1 =
    "struct Enemy { pub var hp: f64; }\n"
    "impl Enemy { pub fun speed(self): f64 = 1; }\n"
    "fun tick(): f64 = 1;\n"
    "var e = Enemy { hp: 10 };\n";
  const char *v2 =
    "struct Enemy { pub var hp: f64; }\n"
    "impl Enemy { pub fun speed(self): f64 = 2; }\n"
    "fun tick(): f64 = 2;\n";
  printf("load v1 -> %d\n", runSnippet(v1));
  fflush(stdout);
  printf("load v2 (same names, same instance) -> %d\n", runSnippet(v2));
  printf("tick() after reload: "); fflush(stdout); runSnippet("@println(tick());");
  printf("old Enemy instance, speed() after reload: "); fflush(stdout); runSnippet("@println(e.speed());");
  return 0;
}
```

```text
load v1 -> 0
[line 3] Error at 'tick': Already declared in this scope.
[line 1] Error at 'Enemy': Already declared in this scope.
load v2 (same names, same instance) -> 1
tick() after reload: 1
old Enemy instance, speed() after reload: 1
```

### A.11 The example game script

Claim: the script in [Proposed Changes] is valid Kirby and behaves as described, when the host supplies the three functions.

`h/enemies.krb` is the script shown there. `h/t10.c` registers `@spawn`, `@getX` and `@setX` with the current `defineNative`, loads the file, runs `onStart` once and `onUpdate(1/60)` sixty times, using the compile-a-snippet workaround of [A.6]:

```c
#include "common_host.h"
#include "native.h"
static double xs[16]; static int count = 0;
static Value hSpawn(VM *vm, int argc, Value *args) { (void)vm; (void)argc; xs[count] = AS_NUMBER_F64(args[1]); return NUMBER_F64_VAL(count++); }
static Value hGetX(VM *vm, int argc, Value *args) { (void)vm; (void)argc; return NUMBER_F64_VAL(xs[(int)AS_NUMBER_F64(args[0])]); }
static Value hSetX(VM *vm, int argc, Value *args) { (void)vm; (void)argc; xs[(int)AS_NUMBER_F64(args[0])] = AS_NUMBER_F64(args[1]); return NIL_VAL; }
static char *slurp(const char *p) { FILE *f = fopen(p, "rb"); fseek(f, 0, SEEK_END); long n = ftell(f); rewind(f); char *b = malloc(n + 1); if (fread(b, 1, n, f) != (size_t)n) return NULL; b[n] = 0; fclose(f); return b; }
int main(void) {
  initVM(0, NULL); typchkSessionBegin();
  defineNative(&vm, "@spawn", hSpawn); defineNative(&vm, "@getX", hGetX); defineNative(&vm, "@setX", hSetX);
  char *src = slurp("enemies.krb");
  printf("load -> %d\n", runSnippet(src));
  printf("onStart -> %d\n", runSnippet("onStart();"));
  for (int frame = 0; frame < 60; frame++) runSnippet("onUpdate(0.016666666);");
  printf("after 60 frames: enemy0.x = %.2f, enemy1.x = %.2f (expect about 40 and 140)\n", xs[0], xs[1]);
  return 0;
}
```

Run from `h/`, because it opens `enemies.krb` by a relative path:

```text
Operands must be two numbers of the same type.
[line 17891345] in script
load -> 2
onStart -> 0
after 60 frames: enemy0.x = 40.00, enemy1.x = 140.00 (expect about 40 and 140)
```

The enemy that started at 0 moves at 40 units per second for one second, so
the script behaves as described. Loading it also reports a runtime error with
a nonsense line number. That is a compiler bug, not a problem with the script:
`endCompiler` (`src/compiler.c`) leaves out the script's final `OP_NIL` and
`OP_RETURN` when the last byte of its code is 11, the number of `OP_RETURN`,
because `previousOpCode` reads the last byte without knowing whether it is an
instruction or an operand. Here the last instruction is
`OP_DEFINE_GLOBAL 11 'enemies'`, so the VM runs past the end of the script.
The functions it defined before that point still work, which is why `onStart`
and `onUpdate` run. What happens past the end varies: `krb run enemies.krb`
stops with `Field declaration outside struct.`, and an AddressSanitizer build
reports a heap-buffer-overflow in `run`. With one more line at the end of the
file, such as `let loaded = true;`, `load` returns 0 and nothing else changes.
The [Top-Level Declarations Proposal] describes the same bug in its Part 1.

## Glossary

These are both technical and non technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document
- **Host**: The program that contains Kirby, such as a game engine
- **Instance**: One independent copy of Kirby inside a host, with its own
  variables, its own memory and its own settings
- **Native function**: A built-in function written in C, named with an `@`
  prefix, such as `@strLen`
- **Host function**: A native function that the host supplies for its own
  scripts, such as `@spawn`
- **Library group**: A named set of native functions that a host can allow or
  leave out, such as `MATH` or `IO`
- **File-scope variable**: A C variable written outside any function. It exists
  for as long as the program runs, and any code that can name it can change it.
  Often called a global variable
- **Front end**: The parser, the type checker and the compiler. It turns text
  into bytecode
- **Runtime**: The virtual machine, the garbage collector and everything else
  needed to run bytecode that has already been made
- **Garbage collector**: The part of the runtime that frees memory that nothing
  can reach any more
- **Root**: A place the garbage collector starts from when it looks for what is
  still reachable, such as the value stack and the global variables. A value
  that is not reachable from a root can be freed
- **Scope** (embedded library): A stretch of host code between `krbScopeBegin` and
  `krbScopeEnd`. Every value Kirby hands out during it stays alive until it ends
- **Pin**: A request that one value stays alive until the host lets it go
- **Hook**: A function the host gives to Kirby so that Kirby can call it later,
  for example to write output
- **Raise and return**: Reporting an error by writing it on the VM and returning
  from the current function at once, so that the caller can notice
- **`setjmp` and `longjmp`**: A pair of C functions. `setjmp` saves a spot in a
  program. `longjmp` later jumps straight back to it from any depth, skipping the
  code in between
- **Re-entrant**: Able to be started again while it is already running. The VM's
  main loop has to be, so that a host function can call back into a script
- **Unwinding**: Undoing the work of the calls in progress when an error is
  raised, so that the stack is as it was before the outermost call began
- **Bytecode**: The instructions the virtual machine runs, made by the compiler
- **Runaway script**: A script that never finishes, or takes far longer than the
  host will allow, such as `while (true) {}`
- **Signature**: The types of a function's parameters and result, such as
  `fun (f64, f64) => f64`
- **Handle**: A number a host gives a script to stand for one of the host's own
  objects. The host keeps the real object

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Impacts]: #impacts
[Problem Statement]: #problem-statement
[Proposed Changes]: #proposed-changes
[Appendix A]: #appendix-a-checking-the-claims

<!-- Parts of the implementation plan -->

[Part 1]: #part-1-the-runtime-links-without-the-compiler
[Part 2]: #part-2-one-instance-no-hidden-shared-state
[Part 3]: #part-3-a-scripts-mistake-stops-the-script
[Part 4]: #part-4-everything-kirby-says-goes-through-a-hook
[Part 5]: #part-5-call-kirby-from-the-host-and-get-values-back
[Part 6]: #part-6-host-functions-the-checker-knows-about
[Part 7]: #part-7-what-a-script-may-use-and-how-much
[Part 8]: #part-8-a-public-header-and-a-library-that-is-easy-to-link
[Part 9]: #part-9-compiled-scripts-and-a-runtime-only-build

<!-- Appendix A -->

[A.1]: #a1-the-pipeline-works-from-c-and-script-errors-are-survivable
[A.2]: #a2-a-scripts-mistake-can-end-the-host
[A.3]: #a3-a-host-function-works-today-and-it-is-unchecked
[A.4]: #a4-the-runtime-half-links-on-its-own
[A.5]: #a5-one-copy-per-process
[A.6]: #a6-calling-a-script-function-by-compiling-source
[A.7]: #a7-a-direct-call-and-the-check-for-runaway-scripts
[A.8]: #a8-what-a-script-can-do-to-the-hosts-process
[A.9]: #a9-the-shared-library
[A.10]: #a10-loading-the-same-script-twice
[A.11]: #a11-the-example-game-script

<!-- Questions -->

[Q-call-by-source]: #q-is-it-enough-for-a-host-to-compile-a-snippet-for-every-call
[Q-check-cost]: #q-what-does-the-check-for-runaway-scripts-cost
[Q-error-model]: #q-should-a-native-report-an-error-by-returning-or-by-jumping-out
[Q-value-api]: #q-how-does-the-host-hold-script-values-safely
[Q-host-names]: #q-what-names-may-host-functions-have
[Q-signatures]: #q-how-are-host-function-types-written
[Q-userdata]: #q-how-does-a-script-hold-something-that-belongs-to-the-host
[Q-frontend-context]: #q-how-should-the-compilers-shared-state-be-handled
[Q-libs]: #q-what-does-a-new-instance-get-and-are-the-groups-right
[Q-oom]: #q-what-happens-when-memory-runs-out
[Q-reload]: #q-how-is-a-changed-script-loaded-into-a-running-instance
[Q-coroutines]: #q-can-a-script-pause-and-continue-on-a-later-frame
[Q-threads]: #q-what-is-safe-to-do-from-several-threads
[Q-abi]: #q-how-stable-is-the-header
[Q-stdlib]: #q-what-does-the-standard-library-cost-each-new-instance

<!-- Proposals -->

[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Testing Proposal]: ../testing/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Prefixed Native Functions Proposal]: ../prefixed-native-functions/PROPOSAL.md
[Additional Native Functions Proposal]: ../additional-native-functions/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Lists Proposal]: ../lists/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Sized Number Types Proposal]: ../sized-number-types/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[String Interpolation Proposal]: ../string-interpolation/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-check-only]: ../top-level-declarations/PROPOSAL.md#q-how-are-tests-that-only-check-compilation-written
[Q-interface]: ../modules/PROPOSAL.md#q-what-exactly-does-a-module-interface-contain-and-in-what-format
[Q-strip]: ../tooling-support-data/PROPOSAL.md#q-is-tooling-data-always-recorded
[Q-check]: ../debugger/PROPOSAL.md#q-how-does-the-vm-check-whether-to-stop
[Q-register]: ../testing/PROPOSAL.md#q-how-are-tests-registered-when-the-top-level-cannot-make-calls

<!-- External pages -->

[OpenMW's Teal page]: https://openmw.readthedocs.io/en/latest/reference/lua-scripting/teal.html
