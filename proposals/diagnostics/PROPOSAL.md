---
status: Draft
created: 2026-09-20
from_commit: 662d98b
---

# Proposal: Diagnostics

When the scanner, the parser, the compiler, or the type checker finds a problem,
it prints a line to stderr and forgets it. Nothing keeps the problem: not where
it was, not what kind it was, not a list of everything that was found. An editor
that wants to underline the mistakes in a file has nothing to ask, and a program
that embeds Kirby has no way to get the mistakes as data.

This proposal adds one small module for that. A _diagnostic_ is a record of one
problem: how serious it is, which phase found it, where it is, and what it says.
Every phase that reads a program (lexing, parsing, compiling, type checking, and
definite assignment) reports through the same few functions, and each record
goes to a _sink_. The default sink prints exactly the text Kirby prints today.
Another sink collects the records into a list, which is what an editor needs.
There is also one answer to "was there an error?", where today there are three.

The same change ends a loop between the type checker and definite assignment
analysis, and it shrinks what `src/typecheck.h` has to expose. It also gives
lexical errors a proper representation: today an error token's text is sometimes
the message and sometimes the source, and three lexical errors print the rest of
the file as their message. Nothing about what a correct program does changes,
and every error message keeps its text, with one exception: those three.

A record needs a position better than a line, and the [Tooling Data Proposal] is
where that comes from. Whether this proposal lands before it or after it is the
first open question, [Q-order]. A later step, readable errors on stderr in the
style of Rust's `codespan-reporting`, is not built here, but the design leaves
room for it ([Looking ahead]).

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

Kirby reports a problem well enough for a person reading a terminal. It does not
report one in a form a program can use, and the way it reports is spread over
five functions that each do the same job on their own.

### What Kirby has today

Checked at `from_commit` (see [Appendix A]):

- **Five functions write every compile error.** They are `parse_error_at` in
  `src/parser.c`, `compilerErrorAt` and `compilerErrorAtToken` in
  `src/compiler.c`, and `typchkErrorAtToken` and `typchkErrorAtNode` in
  `src/typecheck.c`. `src/definite_assignment.c` reports through the type
  checker's `typchkErrorAtTokenFmt`. Each writes to stderr with a few `fprintf`
  calls: first `[line 3] Error`, then ` at 'x'` or ` at end`, then the message.
  The three that take a token (the parser's, the compiler's, and the type
  checker's) are the same code copied.
- **The scanner does not report errors.** It returns a `TOKEN_ERROR` token, and
  the parser's `advance` reports it, using the token's text as the message.
  There are five cases. `errorToken` makes two of them, `Unexpected character.`
  and `Unterminated string.`, and the token's text is that message: a string in
  the binary, not a place in the source. A lone `?`, `&`, or `|` makes the other
  three, with `makeToken`, so the token's text is the source. The parser prints
  that as if it were the message, and it prints the rest of the file: `var x = 1
& 2;` gives `[line 1] Error: & 2;` followed by every line after it. No
  snapshot covers those three, or `Unexpected character.`. The one lexical
  snapshot is `Unterminated string.` for a bare `"Hello;`.
- **An unterminated string in a `var` crashes.** `var s = "abc;` exits with a
  segmentation fault (139). A bare `"abc;` and `var s = #;` exit with 65 as they
  should. The cause is not known.
- **Three shapes of text.** 280 of the 705 `.err` snapshots contain a compile
  error line, 321 lines in all: 247 say ` at 'x'`, 1 says ` at end`, and 73 have
  no `at`.
- **108 places report an error.** 23 are in the parser, 21 in the compiler, and
  64 in the type checker and definite assignment. 80 point at a token, 24 at an
  AST node, and 4 at "the line the compiler is on now".
- **Failure is signalled three ways.** `parse` hands back a flag through an
  out-parameter (2 callers in `main.c`, 16 calls in 5 unit test files).
  `typchkCheckProgram` returns a bool, and `compile` returns `NULL`. Each is fed
  by its own flag: the parser keeps `hadError` in its `Parser`, and the compiler
  and the type checker each keep a file-scope `hadError`. The type checker's is
  read and cleared through `typchkHadError` and `typchkResetError`, both
  declared in `src/typecheck.h`. Unit tests call the first 54 times and the
  second 154 times. `compileSource` in `src/main.c` combines the three.
- **Nothing remembers an error.** Once the line is printed, all that is left is
  a flag.
- **A position is a line.** A `Token` has a `line` and no column, and an
  `AstNode` has a `line`. The [Tooling Data Proposal] describes this in full. An
  error token has less than that: its text is a message, or the rest of the
  file, so it cannot say where in the source it is.
- **Some things on stderr are not diagnostics.** Out-of-memory messages such as
  `realloc failed in tsWrite`, the REPL's `Compiler Error!` and `Runtime Error!`
  lines in `main.c`, and the debug listings in `debug.c` and `object.c`. Runtime
  errors are written by `runtimeError` in `src/vm.c` ([Q-runtime]).

### What is missing

1. **Errors as data.** An editor wants, for each problem, a range, a severity, a
   code, and a message. That is the shape of a diagnostic in the Language Server
   Protocol (LSP). Today the only way to get one is to read stderr.
2. **A list.** After a check, there is no way to ask "what did you find?"
3. **A choice of where the text goes.** It goes to stderr. The [Embedded Library
   Proposal] wants it to go to a hook the host sets, and with five writers that
   is five edits, and five more for a record instead of text.
4. **A way for a test to see what was reported.** A test can see that a flag is
   set. A test that expects an error passes for any error, including a wrong one
   that came from a mistake in the test's own source.
5. **Two files that do not need each other.** The type checker calls definite
   assignment (`daaCheckFn`, `daaCheckAssignmentStmt`), and definite assignment
   calls back into the type checker to report (`typchkErrorAtTokenFmt` in
   `src/definite_assignment.c`). `src/typecheck.h` exposes the writer and the
   flag so that it can.
6. **Lexical errors that say where they are.** An error token says what went
   wrong in its text, in two of the five cases, and nothing of where.
7. **One answer to "was there an error?"** Today it takes three signals,
   combined by hand in `compileSource`.
8. **Room for more than errors.** LSP has four severities. Kirby has one kind of
   problem, and nothing needs the others yet, but a record with no severity
   would have to change the day one is added.

### Who needs what

| Who                                          | Needs                                                                     |
| -------------------------------------------- | ------------------------------------------------------------------------- |
| The command line                             | The same text as today                                                    |
| Editors (LSP)                                | A list of records: range, severity, code, message                         |
| Hosts ([Embedded Library Proposal])          | The text, or the record, handed to the host instead of stderr             |
| Tests                                        | To see what was reported, not only that something was                     |
| [Macros Proposal]                            | A problem in generated syntax reported at the macro call that produced it |
| [Tooling Data Proposal] (compile errors row) | A way to hand a span over as data and not only as text                    |

## Proposed Changes

One new module, `src/diagnostics.c` and `src/diagnostics.h`, builds every
compile-time diagnostic. The five writer functions keep their names and their
callers, and their bodies shrink to one call each. The scanner's error tokens
are fixed so that they can be reported properly, and the three flags become one.

This is every place a problem in a program is reported, and what happens to it:

| Phase                               | Reports today through                        | After this proposal                                               |
| ----------------------------------- | -------------------------------------------- | ----------------------------------------------------------------- |
| Lexing                              | `TOKEN_ERROR` tokens, reported by the parser | Diagnostics with the code `lexical`, and an exact span ([Part 5]) |
| Parsing                             | `parse_error_at`                             | `diagErrorAtToken`, code `syntax` ([Part 3])                      |
| Compiling                           | `compilerErrorAt`, `compilerErrorAtToken`    | `diagErrorAt*`, code `compile` ([Part 3])                         |
| Type checking                       | `typchkErrorAtToken`, `typchkErrorAtNode`    | `diagErrorAt*`, code `type` ([Part 2])                            |
| Definite assignment                 | The type checker's writer                    | `diagErrorAtToken`, code `flow` ([Part 2])                        |
| Whether there was an error          | Three flags and three signals                | One flag ([Part 6])                                               |
| Runtime errors                      | `runtimeError` in `src/vm.c`                 | Not a diagnostic ([Q-runtime])                                    |
| Out of memory, REPL lines, listings | `fprintf(stderr, ...)`                       | Not diagnostics. They are not about a program                     |

### Goals and Non Goals

What this proposal covers:

- One module builds every compile-time diagnostic. The scanner's errors (which
  the parser reports), the parser, the compiler, the type checker, and definite
  assignment all report through it.
- A diagnostic is data: a severity, a code, a span, and a message.
- A sink receives each diagnostic. The default sink prints exactly today's text.
  A second sink collects diagnostics into a list.
- A lexical error says what it is and where: it has its own code, and its span
  is the source text it is about ([Part 5]).
- One flag. `diagHadError()` is the only answer to "was there an error?". The
  parser's and the compiler's flags, and the out-parameter of `parse`, go ([Part
  6]).
- The text, its order, and the exit codes do not change, except for three
  lexical errors that print the rest of the file today, and one crash ([Part
  5]). `just test` passes with no snapshot updated, after every part, apart from
  the new tests that part adds.
- The type checker and definite assignment stop needing each other.
  `definite_assignment.c` no longer includes `typecheck.h`, and `typecheck.h`
  loses the writer and the flag functions.
- The design works whether it lands before or after the [Tooling Data Proposal]
  ([Q-order]).
- The record has room for readable errors on stderr later ([Looking ahead]).

The following is intentionally left out of scope for this proposal:

- **Changing any message**, except the three above. The wording of the other
  messages stays.
- **Better lexical messages.** The three lone characters get `Unexpected
character.`, like any other character Kirby does not know. A message such as
  "Expect '&&'." can come later.
- **Runtime errors.** They carry a stack trace and end the run. See [Q-runtime].
- **Warnings and hints.** The record has a severity so they can be added.
  Nothing emits one.
- **A language server.** This proposal produces the records. The server, the
  protocol, and its JSON are separate work. Turning a span into what an editor
  counts in (LSP numbers lines from 0 and characters in UTF-16 units) belongs to
  an editor adapter, which has the line's text. The [Tooling Data Proposal] says
  the same.
- **Readable errors on stderr.** Not built here ([Looking ahead]).
- **Related locations and notes**, such as "first declared here". A record has
  one span and one message ([Q-labels]).
- **Error recovery.** The parser's panic mode, which decides which errors are
  reported at all, stays where it is.
- **Out-of-memory messages, the REPL's status lines, and the debug listings.**
  They are not about a program.
- **Passing a context through every function.** See [Q-state].

### Implementation Plan

The parts go in order. [Part 1] builds the module. [Part 2] moves the type
checker and definite assignment over, and [Part 3] moves the parser and the
compiler. [Part 4] gives a record its position. [Part 5] gives lexical errors a
source range and a code of their own, and [Part 6] makes the module the only
place that knows whether there was an error. [Part 7] says how an editor or a
host uses the records. Where a position comes from depends on [Q-order], so Part
4 is written for each answer, and under one of them it comes first.

Nothing is left half done. After Part 6 every problem the front end finds goes
through the module, and the module is the only thing that answers whether there
was one.

#### Part 1: The record, the sinks, and the text form

<!-- [Part 1]: #part-1-the-record-the-sinks-and-the-text-form -->

A new `src/diagnostics.h`:

```c
typedef enum {
  DIAG_ERROR = 1, // These numbers are the ones the LSP specification uses
  DIAG_WARNING = 2,
  DIAG_INFORMATION = 3,
  DIAG_HINT = 4,
} DiagSeverity;

typedef struct {
  DiagSeverity severity;
  const char *code;   // The phase: "lexical", "syntax", "compile", "type", "flow"
  Span span;          // Where. See Part 4
  const char *atText; // What the text form says the error is "at", or NULL
  int atLength;
  bool atEnd;         // The text form says "at end"
  char *message;
} Diagnostic;
```

`Span` is the type from the [Tooling Data Proposal] ([Span]): byte offsets, and
a line and column for the start and the end.

A record is what an editor gets. `atText` and `atEnd` are there only so the text
form can be rebuilt exactly. `atText` points into the source text and is valid
only while a sink runs, so a sink that keeps a record copies what it needs and
leaves `atText` out.

Three functions build a record from what a phase has, format the message, and
give the record to the sink. They are variadic, as the type checker's `Fmt`
functions are today:

```c
void diagErrorAtToken(const char *code, const Token *token, const char *fmt,
                      ...);
void diagErrorAtNode(const char *code, const AstNode *node, const char *fmt,
                     ...);
void diagErrorAtLine(const char *code, int line, const char *fmt, ...);
```

The three match the three things a phase points at today: a token (80 of the
108 places), a node (24), and a bare line (4). Reporting an error also sets the
module's flag.

A sink is a function that is called once for each diagnostic:

```c
typedef void (*DiagSink)(void *user, const Diagnostic *diagnostic);

void diagSetSink(DiagSink sink, void *user); // NULL restores the default

bool diagHadError(void);                     // An error since the last reset
void diagReset(void);
```

There are two sinks to start with:

- **The default sink** turns the record into text and writes it to stderr in one
  call. The text is built by one function that any test or host can also call,
  so the text is tested without capturing stderr:

  ```c
  void diagFormat(const Diagnostic *diagnostic, StrBuf *out);
  ```

  It writes `[line N] Error` and then ` at 'x'`, ` at end`, or nothing, then
  `: message` and a newline. The three shapes are the ones in the snapshots.
  Building the whole line before writing it is also what the [Embedded Library
  Proposal] asks for in its [Embedded Library Part 4].

- **The list sink** copies each record into a list:

  ```c
  typedef struct {
    Diagnostic *items;
    int count;
    int capacity;
  } DiagList;

  void diagListInit(DiagList *list);
  void diagListFree(DiagList *list);
  // user is a DiagList *
  void diagListSink(void *user, const Diagnostic *diagnostic);
  ```

The module keeps three things in file-scope variables: the sink, its `user`
pointer, and the flag ([Q-state]).

Test first: `unit/diagnostics.c` builds one record of each shape and checks the
text `diagFormat` gives against the text the writers give today, and checks that
a list sink receives records in order.

#### Part 2: The type checker and definite assignment

<!-- [Part 2]: #part-2-the-type-checker-and-definite-assignment -->

The type checker has 64 of the 108 places, and the loop, and the flag that unit
tests read, so it goes first. Its four functions (the two writers and their
`Fmt` forms) keep their names and their 63 callers. They become `static`, and
each writer's body becomes one call:

```diff
-void typchkErrorAtToken(Token *token, const char *message) {
-  hadError = true;
-  fprintf(stderr, "[line %d] Error", token->line);
-  if (token->type == TOKEN_EOF) {
-    fprintf(stderr, " at end");
-  } else if (token->type != TOKEN_ERROR) {
-    fprintf(stderr, " at '%.*s'", token->length, token->start);
-  }
-  fprintf(stderr, ": %s\n", message);
-}
+static void typchkErrorAtToken(Token *token, const char *message) {
+  diagErrorAtToken("type", token, "%s", message);
+}
```

`typchkErrorAtNode` changes the same way, to `diagErrorAtNode`. The `Fmt`
functions still format into their 256 byte buffer and call the writers, as they
do today.
The type checker's own flag goes. `typchkCheckProgram` calls `diagReset()` where
it calls `typchkResetError()`, and asks `diagHadError()` where it asks
`typchkHadError()`.

Definite assignment stops including `typecheck.h` and reports directly:

```diff
-#include "typecheck.h"
+#include "diagnostics.h"
 ...
-      typchkErrorAtTokenFmt(name, "'%.*s' might not be assigned yet.",
-                            name->length, name->start);
+      diagErrorAtToken("flow", name, "'%.*s' might not be assigned yet.",
+                       name->length, name->start);
```

`src/typecheck.h` loses `typchkErrorAtToken`, `typchkErrorAtTokenFmt`,
`typchkHadError`, and `typchkResetError`. The type checker still calls definite
assignment, and definite assignment no longer calls the type checker.

The unit tests that read the flag change to read what was reported. A test
installs a list sink and checks the code, the line, and the message, so a test
that expects one error no longer passes on a different one:

```diff
-  typchkResetError();
-  bool ok = typecheckSource("var x: f64 = \"hi\";");
-  assert(!ok);
+  DiagList found;
+  diagListInit(&found);
+  diagSetSink(diagListSink, &found);
+  bool ok = typecheckSource("var x: f64 = \"hi\";");
+  assert(!ok);
+  assert(found.count == 1);
+  assert(found.items[0].span.startLine == 1);
```

Test first: change one existing unit test as above and see it fail to compile,
then move the writers.

#### Part 3: The parser and the compiler

<!-- [Part 3]: #part-3-the-parser-and-the-compiler -->

The parser has one writer, `parse_error_at`. It keeps its panic mode and, until
[Part 6], sets `p->hadError` as it does today. The `fprintf` calls become one
call, and its 23 callers do not change:

```diff
   p->panicMode = true;
   p->hadError = true;

-  fprintf(stderr, "[line %d] Error", token->line);
-  if (token->type == TOKEN_EOF) {
-    fprintf(stderr, " at end");
-  } else if (token->type != TOKEN_ERROR) {
-    fprintf(stderr, " at '%.*s'", token->length, token->start);
-  }
-  fprintf(stderr, ": %s\n", message);
+  diagErrorAtToken("syntax", token, "%s", message);
```

The compiler has two writers, and its 21 callers do not change. Each of the
three functions the callers use still sets the compiler's own `hadError`, until
[Part 6]. `compilerErrorAtToken` calls `diagErrorAtToken`, `compilerErrorAtNode`
calls `diagErrorAtNode`, and `compilerError` calls `diagErrorAtLine` with
`currentLine`. `compilerErrorAt` goes: its two callers are those two wrappers,
and both pass no lexeme, so the branch that prints one never runs.

The parser also reports error tokens here, with the code `syntax`, until [Part
5]. After this part no phase writes a diagnostic to stderr itself. The three
copies of the token-printing code are one, in `diagFormat`.

Test first: `unit/diagnostics.c` parses a snippet with a syntax error and
compiles one with a compile error, with a list sink installed, and checks the
codes and lines.

#### Part 4: Positions

<!-- [Part 4]: #part-4-positions -->

A record's `span` is the [Span] type from the [Tooling Data Proposal]: 1-based
lines and columns that count bytes, and byte offsets. `fileId` is 0, because
every unit holds one source today. The three report functions build it:

- `diagErrorAtToken` builds it from the token's position and text. An error
  token is the exception until [Part 5]: its text is a message, or the rest of
  the file, and not the source it is about.
- `diagErrorAtNode` copies the node's span.
- `diagErrorAtLine` has only a line. Its span has `startLine` and `endLine` set
  to that line, its columns set to 0 (a column starts at 1), and its offsets set
  to -1. An editor underlines the whole line. The four places in the compiler
  that report at `currentLine` are these until the compiler stops using it
  ([Tooling Data Proposal], Part 4).

The options for [Q-order] differ in who builds the spans.

**A: before the tooling data.** This proposal does the two steps of the
[Tooling Data Proposal]'s suggested order that a diagnostic needs ([Part 9]
there, steps 1 and 2), in slices:

- **A1: tokens.** The scanner gives every token a start line and a column, and
  `Token` gets a `column`. The `Span` type is added, as that proposal defines
  it. `diagErrorAtToken` builds a span from a token. This covers the 80 places
  that point at a token, which is the whole parser, most of the type checker,
  and part of the compiler. Reports on error tokens get their range in [Part 5].
- **A2: nodes.** `AstNode` gets a span, one kind at a time, in the order the 24
  node-anchored reports need. Each kind has a test that checks the range of the
  diagnostic reported on it, and the test doubles as the span test that proposal
  asks for. Kinds no report points at are left to that proposal.

Once A2 has covered every kind, steps 1 and 2 of the tooling proposal are done,
and that proposal continues from step 3. Nothing else of it is done here: not
the origin, not the instruction spans, not the file names, and not the variable
records. `Token` and `AstNode` change size as that proposal says ([Q-cost]
there).

**B: after the tooling data.** Steps 1 and 2 are already done. `Token` has a
`column` and every node has a span. This part is only the three functions above.

**C: structure first, ranges later.** Parts 1 to 3 do not need a real span, so
they could land first with a record that holds only a `line`, and the field
would become a `Span` when the tooling data lands. That changes the record
twice, but nothing outside this repository reads it yet, so the cost is one
struct, three functions, and `diagFormat`.

#### Part 5: Lexing errors

<!-- [Part 5]: #part-5-lexing-errors -->

The scanner does not report errors. It returns an error token, and the parser
reports it ([What Kirby has today]). The parser is the right place to keep
reporting them: it holds a lexical error back while it is in panic mode, and it
reports one when it reaches it, in the order of the source. What has to change
is what an error token carries. It has to say what went wrong and where, and
today it says neither reliably.

An error token's `start` and `length` will always be the source text it is
about, and its type will say what went wrong. Its message comes from its type,
not from its text:

| Case                                  | Token today                                       | Token after                                                                              | Message                                                   |
| ------------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| A character the scanner does not know | `errorToken`. Its text is `Unexpected character.` | `TOKEN_ERROR`. Its text is the character                                                 | `Unexpected character.`                                   |
| A lone `?`, `&`, or `\|`              | `makeToken`. Its text is the source               | `TOKEN_ERROR`. No change to the token                                                    | `Unexpected character.`. Today it is the rest of the file |
| An unterminated string                | `errorToken`. Its text is `Unterminated string.`  | `TOKEN_UNTERMINATED_STRING`. Its text runs from the opening quote to the end of the file | `Unterminated string.`                                    |

Two functions join `src/token.h`, and the parser's checks for `TOKEN_ERROR`
become one call:

```c
bool tokenIsError(TokenType type);
const char *tokenErrorMessage(TokenType type);
```

```diff
-    if (parser->current.type != TOKEN_ERROR)
+    if (!tokenIsError(parser->current.type))
       break;

-    error_at_current(parser, parser->current.start);
+    error_at_current(parser, tokenErrorMessage(parser->current.type));
```

`errorToken` goes. Its two callers use `makeToken`:

```diff
-  return errorToken(scanner, "Unexpected character.");
+  return makeToken(scanner, TOKEN_ERROR);
```

```diff
-    return errorToken(scanner, "Unterminated string.");
+    return makeToken(scanner, TOKEN_UNTERMINATED_STRING);
```

The parser's writer gives a lexical error its own code, and the diagnostic
leaves out the "at" text for an error token, as the three copies of `token->type
!= TOKEN_ERROR` do today:

```diff
-  diagErrorAtToken("syntax", token, "%s", message);
+  diagErrorAtToken(tokenIsError(token->type) ? "lexical" : "syntax", token,
+                   "%s", message);
```

Now that an error token's text is always the source, [Part 4] builds its span
like any other token's. For an unterminated string that is from the opening
quote to the end of the file, over as many lines as that is. Its `line` stays
where the scanner stopped, as it is today, until the tooling data changes a
token's line to where the token starts.

The three lone characters print `Unexpected character.` and not the rest of the
file. That is a change of text, and it is the only one in this proposal. How the
scanner hands a lexical error over is [Q-lexing].

**The crash.** `var s = "abc;` ends in a segmentation fault today. The first
test of this part is that program, and it fails before it reaches the message.
Finding out why is the first step of the part. The cause is not known to this
proposal.

Test first: the new E2E tests in the [Testing Plan], and a case in
`unit/lexer.c` for each of the five, checking the token's type and its text.

#### Part 6: One flag

<!-- [Part 6]: #part-6-one-flag -->

After Parts 2 and 3, every phase reports through the module, so `diagHadError()`
already knows whether anything went wrong. This part makes it the only place
that knows, and removes the three signals that `compileSource` combines today:

- `compileSource` calls `diagReset()` first. After the parser, and after each
  phase that follows, it asks `diagHadError()`. This replaces the parser's
  out-parameter, the type checker's return value, and the `NULL` from `compile`.
- `Parser.hadError` and the `hadError` out-parameter of `parse` go, once nothing
  reads them. `parse` returns the AST. Its 18 callers change: 2 in `main.c`, and
  16 calls in 5 unit test files ([Q-flag]). The parser's panic mode stays. It is
  not a flag for callers. It decides which errors are reported.
- The compiler's file-scope `hadError` goes. `compile` returns `NULL` when
  `diagHadError()` is true as it finishes.
- `typchkCheckProgram` still returns a bool. The answer is read from the module.

The REPL calls `compileSource` for every line, and `diagReset()` runs at the
start of each, so the errors of one line never reach the next.

Test first: a unit test runs a snippet with a lexical error, one with a syntax
error, one with a compile error, and one with a type error, and checks that
`diagHadError()` is true after each, and false after a good snippet.

#### Part 7: What an editor or a host does with it

<!-- [Part 7]: #part-7-what-an-editor-or-a-host-does-with-it -->

Nothing new is built in this part. It says how the pieces are used.

An editor adapter installs the list sink, runs the phases on a buffer, and reads
the list:

```c
DiagList found;
diagListInit(&found);
diagSetSink(diagListSink, &found);

AstNode **ast = parse(text, &count, &hadError, &endLine);
if (!hadError) {
  typchkCheckProgram(ast, count);
}

for (int i = 0; i < found.count; i++) {
  const Diagnostic *d = &found.items[i];
  // Turn d->span into the editor's range, then send
  // d->severity, d->code and d->message.
}

diagListFree(&found);
diagSetSink(NULL, NULL);
```

A host that embeds Kirby has two choices. Its output hook ([Embedded Library
Part 4]) is where the default sink writes, so compile errors reach the host as
text with no more work. A host that wants records sets its own sink.

### Looking ahead: readable errors on stderr

<!-- [Looking ahead]: #looking-ahead-readable-errors-on-stderr -->

Not built here. The aim is output like Rust's compiler and the
`codespan-reporting` crate print: the line of source, an underline beneath the
exact range, the message beside it, and notes below. The design should not need
a second redesign to get there. The [reqlang-expr errors] file already makes
this conversion in Rust: it builds a `codespan_reporting` diagnostic from a
code, a message, and one primary label whose range is a range of bytes.

What such a renderer needs, and where each thing comes from:

| Needs                                                                | Comes from                                                                                                                                                                                                                               |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The exact range, in bytes                                            | `Diagnostic.span`. Its offsets count bytes, which is what the ranges of `codespan-reporting` count                                                                                                                                       |
| Severity, code, and message                                          | The record                                                                                                                                                                                                                               |
| The source text                                                      | The renderer is given it as its `user` data. A sink runs while the phase runs, and the text is alive until compiling ends (`runFile` frees it after that), so a renderer can print as each diagnostic arrives, or when the phases finish |
| The file name                                                        | `fileId`, and the source name of the [Tooling Data Proposal]'s Part 5                                                                                                                                                                    |
| A range over several lines                                           | The span's start and end line and column                                                                                                                                                                                                 |
| More than one label ("first declared here"), and notes ("help: ...") | Not in the record ([Q-labels])                                                                                                                                                                                                           |
| Tab width, wide characters, and colour                               | The renderer                                                                                                                                                                                                                             |

**Switching it on.** The default sink keeps printing today's text when stderr is
not a terminal, and when a flag such as `--error-format=short` asks for it. Then
the `.err` snapshots, and any script that reads Kirby's stderr, stay as they
are. A renderer is a third sink beside the default and the list sink. Choosing
it is one `diagSetSink` call in `main.c`, made when stderr is a terminal. No
phase changes. Whether the terminal check and the flag are the right switch is
decided when the renderer is proposed.

## Impacts

### Existing Syntax Or Behavior

- **No language change.** No syntax, opcode, or bytecode changes.
- **Text, order, and exit codes stay, with two exceptions.** The 280 snapshots
  with a compile error line pass unchanged. A lone `?`, `&`, or `|` prints
  `Unexpected character.` and not the rest of the file. An unterminated string
  in a `var` exits with 65 and its message, where it crashes today. No snapshot
  covers either. One more difference is the one the [Tooling Data Proposal]
  describes: a token that is a string over several lines reports its start line
  and not its end line.
- **Five writers become calls.** Their 108 callers do not change. The three
  copies of the token-printing code become one, in `diagFormat`.
- **The scanner and `token.h`.** An error token's text is always source text.
  `TOKEN_UNTERMINATED_STRING` is new, `tokenIsError` and `tokenErrorMessage`
  join `token.h`, and `errorToken` goes ([Q-lexing]). `TOKEN_ERROR` is checked
  in the parser's `advance` and in each of the three token-printing writers,
  which the module replaces. `unit/lexer.c` expects a `TOKEN_ERROR` token today
  and changes with the scanner.
- **The flags go** ([Q-flag]): `Parser.hadError`, the compiler's `hadError`, and
  the out-parameter of `parse`, which has 18 callers. `compileSource` in
  `main.c` asks the module instead.
- **`src/typecheck.h`** loses four functions: `typchkErrorAtToken`,
  `typchkErrorAtTokenFmt`, `typchkHadError`, and `typchkResetError`.
- **`src/definite_assignment.c`** includes `diagnostics.h` and not
  `typecheck.h`.
- **Unit tests** that call `typchkHadError` (54 times) and `typchkResetError`
  (154 times) change to a list sink.
- **Two new files** join the library and the build: `src/diagnostics.c` and
  `src/diagnostics.h`. `unit/diagnostics.c` is a new test.
- **New shared state.** The module's sink, `user` pointer, and flag are
  file-scope variables. They replace the flags of the compiler and the type
  checker, and they join the bundle in [Embedded Library Part 2].
- **`main.c`** changes in `compileSource` and in its calls to `parse` ([Part
  6]). Its default sink is what it uses today.
- **Code in other proposals** that adds calls to `typchkErrorAtToken` or
  `typchkErrorAtTokenFmt` inside `typecheck.c` (the [Generic Types Proposal],
  the [Primitive Impls Proposal], and the [Unimplemented Proposal]) still works
  as it is written. The functions stay in that file, now `static`.

### Related Proposals

- [Tooling Data Proposal] — defines the `Span` a record holds. Steps 1 and 2 of
  its suggested order (a column on every token, and a span on every node) are
  what a diagnostic needs, and either proposal can land first ([Q-order]).
  Whoever lands second updates the other. That proposal says a token's position
  can be worked out from its text. That holds for every token but an error token
  ([Part 5]), so it is updated to say so. Both proposals change `parse`: it
  gains a source name there (its Part 5) and loses its out-parameter here
  ([Q-flag]), so they should be done in one pass. That proposal is updated to
  say so.
- [Embedded Library Proposal] — its Part 4 names the five functions that write
  compile errors and sends their text to a hook. With this proposal there is one
  module and its default sink writes to that hook, and a structured record is
  one more sink. Its Part 2 bundles the compiler's and the type checker's
  `hadError`, and this proposal replaces both with the module's own state, which
  joins the bundle. If [Q-flag] is answered (a), `parse` loses the out-parameter
  that the harness in its Appendix A.1 uses. That proposal is updated to say so.
- [Top-Level Declarations Proposal] — its `checkTopLevel` reports errors that
  point at a statement, which has a line and no token. It would report with
  `diagErrorAtNode`, and its text stays as that proposal writes it. Its change
  to `compileSource` reads `hadError`, and with [Q-flag] (a) it reads
  `diagHadError()`. That proposal is updated to say so.
- [Macros Proposal] — a diagnostic on generated syntax should be reported at the
  macro call that produced it. The report functions are the one place that can
  follow a span's origin. That waits on [Q-origin]. That proposal is updated to
  say so.
- [Generic Types Proposal], [Primitive Impls Proposal], [Unimplemented Proposal]
  — the code they add to `typecheck.c` calls the writers by name, and it keeps
  working ([Existing Syntax Or Behavior]). They are not changed.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

The E2E tests should cover all valid and invalid parser/compiler/runtime error
cases. Nothing here changes the text of an error, except the three lone
characters, and five lexical cases have no snapshot today (or crash), so they
are new.

##### NEW: tests/errors/lexical_lone_ampersand.krb

```kirby
var x = 1 & 2;
```

###### Expected Outcome

`[line 1] Error: Unexpected character.` on stderr, and exit code 65. Today it
prints the rest of the file.

##### NEW: tests/errors/lexical_lone_pipe.krb

```kirby
var x = 1 | 2;
```

###### Expected Outcome

The same as the lone `&`.

##### NEW: tests/errors/lexical_lone_question.krb

```kirby
var x = 1 ? 2;
```

###### Expected Outcome

The same as the lone `&`.

##### NEW: tests/errors/lexical_unexpected_character.krb

```kirby
var x = 1 # 2;
```

###### Expected Outcome

`[line 1] Error: Unexpected character.` and exit code 65. This is what it prints
today, and no snapshot has it.

##### NEW: tests/errors/lexical_unterminated_string_in_var.krb

```kirby
var s = "abc;
```

###### Expected Outcome

`Unterminated string.` and exit code 65, and no crash. The line is where the
scanner stops, as it is today, and it becomes where the string starts once the
tooling data lands.

##### Existing: the 705 `.err` snapshots

The 280 with a compile error line cover the three shapes: 247 with `at 'x'`, 1
with `at end`, and 73 with no `at`.

###### Expected Outcome

`just test` passes with no snapshot updated, after every part of the plan, apart
from the five new ones. A snapshot that has to change means the text changed,
and the part is wrong.

#### Unit Tests

- **`unit/diagnostics.c`, new.** `diagFormat` gives the text of each shape, for
  a token, a token at the end, a token with no text (an error token), a node,
  and a bare line. A list sink keeps records in order and copies the message.
  `diagHadError` is false until an error is reported and false again after
  `diagReset`. `diagSetSink(NULL, NULL)` restores the default.
- **Each phase.** With a list sink installed, a snippet with a lexical error,
  one with a syntax error, one with a compile error, one with a type error, and
  one that is read before it is assigned each give one record with the right
  `code`, line, and message.
- **`unit/lexer.c`.** Each of the five lexical cases gives a token of the right
  type, and its text is the character, or the string from its opening quote to
  the end. `tokenErrorMessage` and `tokenIsError` are checked for each token
  type.
- **One flag.** After each kind of error `diagHadError()` is true, and after a
  good snippet it is false ([Part 6]).
- **The existing type checker and definite assignment tests** read the list and
  not the flag ([Part 2]).
- **Positions.** They depend on [Q-order]. Under A, each slice adds tests that
  check the range of a diagnostic against the source text. Under B, one test per
  report function checks that the span is the token's or the node's.

## Questions

### **Q:** Does this land before or after the tooling data?

<!-- [Q-order]: #q-does-this-land-before-or-after-the-tooling-data -->

**Status:** Open

The two proposals share two steps. The [Tooling Data Proposal] would give every
token a column and every node a span ([Part 9] there, steps 1 and 2), and a
diagnostic needs both to point at code and not at a line. Whichever proposal
lands first does those steps.

The reports split like this ([Appendix A]):

| Points at                   | Places | Needs                         |
| --------------------------- | ------ | ----------------------------- |
| A token                     | 80     | A column on the token         |
| An AST node                 | 24     | A span on the node            |
| The compiler's current line | 4      | The compiler to stop using it |

|                                    | A: before the tooling data                                              | B: after the tooling data                                      |
| ---------------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------- |
| Position work                      | Does steps 1 and 2 of the tooling proposal                              | None. It uses what that proposal built                         |
| When ranges appear                 | 80 places after slice A1. The 24 node reports as their kinds get a span | All at once, when that proposal's steps 1 and 2 are done       |
| The record, the sink, the loop fix | First                                                                   | Wait for steps 1 and 2                                         |
| `Token` and `AstNode`              | Change before the debugger needs them                                   | Change with the rest of that proposal                          |
| [Embedded Library Part 4]          | Edits one module and not five writers                                   | Edits five writers, then this proposal replaces them, or waits |
| The tooling proposal               | Starts at its step 3, and is updated                                    | Unchanged                                                      |

- **A: Before** ([Part 4]). The structure and the ranges come together,
  and 80 of the 108 places have exact ranges after the first slice. A larger
  first step, and a change to `Token` and `AstNode` ahead of the proposal that
  designs them. Each node kind's test checks a diagnostic's range, so it also
  serves as that proposal's span test for the kind.
- **B: After.** Smaller steps, and nothing in `Token` or `AstNode` changes here.
  The record, the sinks, the hook for the [Embedded Library Proposal], and the
  end of the loop between the type checker and definite assignment all wait for
  steps 1 and 2 of the tooling proposal.
- **C: Structure first, ranges later.** Parts 1 to 3 land with a record that
  holds only a line, and the `Span` arrives with the tooling data. It changes
  the record twice, at a small cost ([Part 4]).

### **Q:** Where does the sink live?

<!-- [Q-state]: #q-where-does-the-sink-live -->

**Status:** Open

The sink, its `user` pointer, and the flag have to live somewhere the five
writers can reach.

- **(a) In file-scope variables in `diagnostics.c`** (proposed). It changes the
  fewest lines, and it is the same choice as [Q-frontend-context] in the
  [Embedded Library Proposal]: the variables join the bundle that is swapped by
  one pointer. Two instances can be compiled one after another. Two compiles at
  the same instant have to take turns.
- **(b) In a context passed to every function.** The parser has a `Parser`, and
  the type checker a `TypeEnv`, but the compiler keeps its state in file-scope
  variables, so it has no context to hang it on. It is a large mechanical
  change, and it makes compiles on several threads safe.

Do (a) now, and (b) if a host needs several compiles at once. The functions
keep their names either way, so the change to (b) is mechanical.

### **Q:** How fine is a diagnostic's code?

<!-- [Q-codes]: #q-how-fine-is-a-diagnostics-code -->

**Status:** Open

A record has a `code`. LSP shows it next to the message, and editors can use it
to offer a fix or link to help.

- **(a) One per phase** (proposed): `lexical`, `syntax`, `compile`, `type`,
  `flow`. It costs nothing to add, and it is what the author's own [Prior art]
  uses. It cannot tell one type error from another.
- **(b) One per message.** Each of the messages gets a name. A test can then
  check the kind of error and not its wording, and an editor can attach a fix to
  one. It is a list of names to write and keep.
- **(c) Both.** A phase and a message name.

Starting with (a) closes nothing off: (b) adds a field or changes the string,
and the reporters already know which message they are sending.

### **Q:** Do runtime errors go through diagnostics?

<!-- [Q-runtime]: #q-do-runtime-errors-go-through-diagnostics -->

**Status:** Open

- **(a) No** (proposed). A runtime error happens while the program runs. It
  carries a stack trace and ends the run with exit code 70. An editor does not
  underline it. The [Embedded Library Proposal] gives it its own hook ([Embedded
  Library Part 4]).
- **(b) Yes.** A runtime error becomes a record, with the trace as its message
  or as related locations. It puts every message in one format, and it needs a
  way to say "related location" that this proposal leaves out.

### **Q:** How does the scanner hand a lexical error to the parser?

<!-- [Q-lexing]: #q-how-does-the-scanner-hand-a-lexical-error-to-the-parser -->

**Status:** Open

An error token has to say what went wrong and where. Today it says the first in
its text in two cases, and neither reliably ([What Kirby has today]). Options:

- **(a) A type for each kind of error** (proposed). `TOKEN_ERROR` keeps meaning
  a character the scanner does not know, and `TOKEN_UNTERMINATED_STRING` is new.
  The text is always the source, and `tokenErrorMessage` gives the message from
  the type. `Token` does not grow, so the layout in the [Tooling Data Proposal]
  holds. A new kind of lexical error is one more token type. There are five
  cases today, and two messages.
- **(b) A message field on `Token`.** No new types. `Token` grows from 24 bytes
  by a pointer, and every node that keeps a token grows with it.
- **(c) The scanner reports the error itself**, through the module. It knows the
  exact range, and `Token` does not change. But the scanner runs over the whole
  file into a token stream before the parser starts, so every lexical error
  would print before any syntax error, and the parser's panic mode would no
  longer hold one back. The order and the number of messages change for some
  inputs.

(a) keeps the order and the suppression the parser has today, which is why it is
proposed.

### **Q:** Does `parse` keep its out-parameter?

<!-- [Q-flag]: #q-does-parse-keep-its-out-parameter -->

**Status:** Open

After [Part 6] the parser's flag is a second copy of what the module knows.
`parse` has 18 callers: 2 in `main.c`, and 16 calls in 5 unit test files.

- **(a) Remove it** (proposed). `parse` returns the AST, and callers ask
  `diagHadError()`. There is one answer to "was there an error?", and nothing to
  keep in step. It is 18 mechanical edits. The [Tooling Data Proposal] also
  changes `parse`, giving it a source name (its Part 5), so the two edits to
  `parse` should be done together.
- **(b) Keep it**, and fill it in from the module. No caller changes. But there
  are then two ways to ask the same question, which is the half-finished state
  this proposal is meant to avoid.

### **Q:** Does the record carry more than one label, and notes?

<!-- [Q-labels]: #q-does-the-record-carry-more-than-one-label-and-notes -->

**Status:** Open

A renderer like `codespan-reporting` draws a primary label, and it can draw
others ("first declared here", "expected f64 because of this") and notes ("help:
..."). A record has one span and one message ([Looking ahead]).

- **(a) Not now** (proposed). The record stays as it is. Adding `labels` and
  `notes` later is one struct, the list sink that copies it, and the reporters
  that want them. The 108 existing callers do not change, because a reporter
  that wants a second label would use a new function that builds one.
- **(b) Now.** `Diagnostic` gets a list of labels and a list of notes, empty for
  every reporter. It costs fields nobody fills, and the list sink has to copy
  them.

Starting with (a) closes nothing off, because nothing outside this repository
reads the record yet.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Diagnostic**: A record of one problem found in a program: how serious it is,
  which phase found it, where it is, and what it says.
- **Sink**: A function that is called once for each diagnostic. The default one
  prints it. Another one keeps it in a list.
- **Writer**: One of the five functions that print a compile error today.
- **Lexical error**: A problem found while the source is split into tokens, such
  as a character Kirby does not know, or a string that is never closed.
- **Error token**: The token the scanner returns for a lexical error,
  `TOKEN_ERROR` today.
- **Flag**: A true or false variable that a phase sets when it has reported an
  error.
- **Phase**: One of the steps that read a program: the parser, the compiler, the
  type checker, or definite assignment.
- **Span**: A record of where a piece of syntax is in the source: which file,
  and where it starts and ends, as byte offsets and as line and column. It is
  defined in the [Tooling Data Proposal].
- **Token**: One piece of source text as the scanner sees it, such as a name, a
  number, or `+`. In Kirby this is `Token` in `src/token.h`.
- **AST**: The tree-shaped data the parser produces from source text. In Kirby
  its nodes are `AstNode` in `src/ast.h`.
- **LSP**: The Language Server Protocol, which editors use to ask a program for
  errors, completions, and go to definition. A diagnostic in LSP has a range, a
  severity, a code, and a message.
- **Snapshot**: A saved copy of what a test printed. `.err` is what a test wrote
  to stderr.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Existing Syntax Or Behavior]: #existing-syntax-or-behavior
[Appendix A]: #appendix-a--reproducing-the-baseline-claims
[Prior art]: #appendix-b--prior-art-reqlang-expr
[What Kirby has today]: #what-kirby-has-today
[Looking ahead]: #looking-ahead-readable-errors-on-stderr
[Testing Plan]: #testing-plan

<!-- Parts -->

[Part 1]: #part-1-the-record-the-sinks-and-the-text-form
[Part 2]: #part-2-the-type-checker-and-definite-assignment
[Part 3]: #part-3-the-parser-and-the-compiler
[Part 4]: #part-4-positions
[Part 5]: #part-5-lexing-errors
[Part 6]: #part-6-one-flag
[Part 7]: #part-7-what-an-editor-or-a-host-does-with-it

<!-- Proposals -->

[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Generic Types Proposal]: ../generic-types/PROPOSAL.md
[Primitive Impls Proposal]: ../primitive-impls/PROPOSAL.md
[Unimplemented Proposal]: ../unimplemented/PROPOSAL.md

<!-- Other proposals' parts and questions -->

[Span]: ../tooling-support-data/PROPOSAL.md#part-1--the-span
[Part 9]: ../tooling-support-data/PROPOSAL.md#part-9--suggested-order
[Q-cost]: ../tooling-support-data/PROPOSAL.md#q-how-much-does-per-node-span-storage-cost-and-does-it-matter
[Q-origin]: ../tooling-support-data/PROPOSAL.md#q-how-is-origin-chained-through-nested-macro-expansion
[Embedded Library Part 2]: ../embedded-library/PROPOSAL.md#part-2-one-instance-no-hidden-shared-state
[Embedded Library Part 4]: ../embedded-library/PROPOSAL.md#part-4-everything-kirby-says-goes-through-a-hook
[Q-frontend-context]: ../embedded-library/PROPOSAL.md#q-how-should-the-compilers-shared-state-be-handled

<!-- Questions -->

[Q-order]: #q-does-this-land-before-or-after-the-tooling-data
[Q-state]: #q-where-does-the-sink-live
[Q-codes]: #q-how-fine-is-a-diagnostics-code
[Q-runtime]: #q-do-runtime-errors-go-through-diagnostics
[Q-lexing]: #q-how-does-the-scanner-hand-a-lexical-error-to-the-parser
[Q-flag]: #q-does-parse-keep-its-out-parameter
[Q-labels]: #q-does-the-record-carry-more-than-one-label-and-notes

<!-- Other links -->

[reqlang-expr errors]: https://github.com/testingrequired/reqlang-expr/blob/main/src/errors.rs

## Appendix A — Reproducing the baseline claims

Each claim can be checked on a clean checkout at `from_commit`, from the root of
the `kirbylang` repository.

**Five functions write every compile error.** Each prints the `[line` prefix:

```shell
grep -n 'fprintf(stderr, "\[line' src/parser.c src/compiler.c src/typecheck.c
# src/parser.c:1600, src/compiler.c:65 and :74, src/typecheck.c:45 and :65
```

The other two `fprintf(stderr` calls in `src/typecheck.c` (lines 92 and 195) say
`realloc failed`. They are not diagnostics and this proposal leaves them alone.
`compilerErrorAt` is called only from `compilerErrorAtNode` and `compilerError`,
and both pass `NULL` as the lexeme:

```shell
grep -n 'compilerErrorAt(' src/compiler.c
# 62 (the definition), 84 and 93 (the two callers, both with NULL, 0)
```

**108 places report an error.** Calls to each function, without its definition
and prototype:

```shell
count() {
  grep -nE "\b$2\(" "$1" \
    | grep -vE '^[0-9]+:(static )?(void|bool) '"$2"'\(' | wc -l
}
count src/parser.c error_at_current           # 14
count src/parser.c parse_error                # 9
count src/compiler.c compilerErrorAtToken     # 9
count src/compiler.c compilerErrorAtNode      # 8
count src/compiler.c compilerError            # 4
count src/typecheck.c typchkErrorAtToken      # 5
count src/typecheck.c typchkErrorAtTokenFmt   # 43
count src/typecheck.c typchkErrorAtNode       # 5
count src/typecheck.c typchkErrorAtNodeFmt    # 12
count src/definite_assignment.c typchkErrorAtTokenFmt  # 1
```

One of the `typchkErrorAtToken` calls is inside `typchkErrorAtTokenFmt`, and one
of the `typchkErrorAtNode` calls is inside `typchkErrorAtNodeFmt`, so those are
not places. That gives 23 in the parser, 21 in the compiler, and 63 plus 1 in
the type checker and definite assignment: 108. By what they point at, 23 + 9 +
(4 + 43) + 1 = 80 tokens, 8 + (4 + 12) = 24 nodes, and 4 at `currentLine`.

**Snapshots.**

```shell
find tests -name '*.krb.err' | wc -l                                  # 705
grep -lE '^\[line [0-9]+\] Error' $(find tests -name '*.krb.err') | wc -l  # 280
grep -hE '^\[line [0-9]+\] Error' $(find tests -name '*.krb.err') \
  | sed -E "s/^\[line [0-9]+\] Error( at (end|'.*'))?:.*/\1/" \
  | sed -E "s/ at '.*'/ at 'x'/" | sort | uniq -c
#   73          (no "at")
#  247  at 'x'
#    1  at end
```

**The flag and the loop.**

```shell
grep -nE 'hadError|typchkResetError|typchkHadError' src/typecheck.c
# 11 (the flag), 44 and 64 (set by the two writers), 77 and 78 (the two
# functions), 2071 (reset by typchkCheckProgram), 2253 (read by it)
grep -n 'typchk' src/definite_assignment.c   # the include, and the call at 103
grep -n 'daa[A-Z][A-Za-z]*(' src/typecheck.c    # the type checker calls it
grep -ho '\btypchkHadError\b' unit/*.c | wc -l   # 54, in 2 files
grep -ho '\btypchkResetError\b' unit/*.c | wc -l # 154, in 3 files
```

**No compile error is longer than the buffer that formats it.**

```shell
grep -hE '^\[line [0-9]+\] Error' $(find tests -name '*.krb.err') \
  | awk '{ if (length($0) > m) m = length($0) } END { print m }'   # 120
grep -n 'char message\[' src/typecheck.c   # 55 and 69: 256 bytes
```

**The scanner does not report errors.**

```shell
grep -nE 'errorToken\(scanner|: TOKEN_ERROR\)' src/scanner.c
# 98, 100 and 102: a lone ?, & and | (made with makeToken).
# 126 and 343: errorToken, "Unexpected character." and "Unterminated string."
sed -n 193,201p src/scanner.c   # errorToken: token.start = message
sed -n 70,80p src/parser.c      # advance: error_at_current(parser, parser->current.start)
grep -n 'lex(source)' src/parser.c   # 1565: parse scans the whole file first
grep -rn 'TOKEN_ERROR' src unit | grep -v '^src/scanner.c'
# token.h, token.c, parser.c (76, 672, 1603), compiler.c:77, typecheck.c:48,
# and unit/lexer.c:59
```

**A lone `?`, `&`, or `|` prints the rest of the file.**

```shell
printf 'var x = 1 & 2;\nprint x;\n' > /tmp/amp.krb
./build/krb -f /tmp/amp.krb; echo $?
# [line 1] Error: & 2;
# print x;
# (a blank line) and then 65
printf 'var x = 1 # 2;\n' > /tmp/hash.krb
./build/krb -f /tmp/hash.krb; echo $?   # [line 1] Error: Unexpected character.  65
grep -rlE 'Unexpected character|Unterminated string' tests --include='*.err'
# tests/errors/compiler_error_unterminated_string.krb.err, and no other
```

**An unterminated string in a `var` crashes.**

```shell
printf 'var s = "abc;\n' > /tmp/u.krb
./build/krb -f /tmp/u.krb; echo $?   # [line 2] Error: Unterminated string.  then 139
printf '"abc;\n' > /tmp/v.krb
./build/krb -f /tmp/v.krb; echo $?   # exit 65
printf 'var s = #;\n' > /tmp/w.krb
./build/krb -f /tmp/w.krb; echo $?   # exit 65
```

It also crashes with AddressSanitizer on, which prints no report for it.

**Failure is signalled three ways.**

```shell
grep -nE '\bparse\(' src/*.c | grep -v '^src/parser.c'   # main.c:98 and main.c:201
grep -hE '\bparse\(' unit/*.c | wc -l                      # 16
grep -lE '\bparse\(' unit/*.c | wc -l                      # 5
sed -n 199,210p src/main.c   # compileSource combines the flag, the bool, and NULL
```

## Appendix B — Prior art: reqlang-expr

The author's [reqlang-expr errors] file has the same job, in Rust, for a
different language. The shape carries over:

| reqlang-expr                                                               | Here                                                                                                |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| An error is a value: an enum for each phase, with data in its cases        | A `Diagnostic` with a code and a formatted message                                                  |
| `Spanned<ExprError>`: an error and a byte range                            | The `span` field, a [Span]                                                                          |
| `Vec<ExprErrorS>` is returned, and nothing prints                          | A list sink collects, and the default sink prints                                                   |
| `AsDiagnostic` and `get_range` turn a span and the source to a range       | An editor adapter does this. It has the line's text                                                 |
| Severity is 1 to 4, LSP's own numbers                                      | `DiagSeverity` with the same numbers                                                                |
| `code` is one string per phase: `lexical`, `syntax`, `compiler`, `runtime` | `code` is one string per phase to start: `lexical`, `syntax`, `compile`, `type`, `flow` ([Q-codes]) |
| A second conversion prints with `codespan_reporting`                       | `diagFormat` prints the text form, and a renderer like it is a later sink ([Looking ahead])         |

Two things differ. Rust's errors carry data in each case (an expected type and
an actual one), where a C record carries only the formatted message. And Rust
returns the list from each function, where a C phase reports as it goes, to a
sink, so no phase's signature changes ([Q-state]).
