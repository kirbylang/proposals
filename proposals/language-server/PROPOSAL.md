---
status: Draft
category: tooling
created: 2026-10-25
from_commit: b41332d
---

# Proposal: Language Server (`krb lsp`)

An editor can show errors as you type, show the type of a name on hover, and
jump to where a name was declared, if a program called a language server tells
it these things. Editors and language servers talk with the Language Server
Protocol (LSP): JSON messages over the server's stdin and stdout.

This proposal adds `krb lsp`, a small language server written in C inside
`krb`. It does not have a parser or checker of its own. On every change to a
file it runs Kirby's real parser, type checker, and compiler on the text, and
turns what they report into LSP messages. The first version gives diagnostics,
hover, and go to definition for one file at a time. Once it exists, any editor
with an LSP client can use it: VS Code through a small change to the extension
in `vsc/`, and Neovim, Helix, and Zed by pointing them at the command.

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

- **The VS Code extension** (`vsc/src/extension.ts`, 78 lines) has a hover
  provider and a "Run Kirby File" command. The hover provider takes the word
  under the pointer and shows `vsc/hovers/<word>.md` if that file exists. There
  are 86 of them, one per native. Colors come from a TextMate grammar
  (`vsc/syntaxes/kirby.tmLanguage.json`). Nothing in the editor knows about
  errors or types.
- **Errors are text on stderr.** Every message from the parser, the type
  checker, and the compiler is written by one of five functions:
  `parse_error_at` in `src/parser.c`, `compilerErrorAt` and
  `compilerErrorAtToken` in `src/compiler.c`, and `typchkErrorAtToken` and
  `typchkErrorAtNode` in `src/typecheck.c`. The format is
  `[line N] Error at 'x': message`. An error written from a token knows the
  token. An error written from a node knows only its line:

  ```c
  static void typchkErrorAtNode(AstNode *node, const char *message) {
    hadError = true;
    fprintf(stderr, "[line %d] Error: %s\n", node->line, message);
  }
  ```

- **The front end never writes to stdout.** No `printf`, `puts`, or
  `fprintf(stdout` in the scanner, parser, checker, compiler, or their helpers.
  Every other `fprintf(stderr` there reports running out of memory, followed by
  `exit()`.
- **Checking can be started again and again.** A session is
  `typchkSessionBegin()`, `compileSource` for the stdlib and for the program,
  then `compilerSessionEnd()` and `typchkSessionEnd()`. Running that 2,000 times
  in one process, on the same file, gave the same result every time and the
  memory in use did not grow after the first 100 runs. One run, for a 3 KB test
  file and the stdlib, took about 0.55 ms ([Appendix B]).
- **A parse error hides every other error.** `compileSource` only calls the
  type checker when parsing had no error. The checker itself keeps going after
  an error and reports every one it finds.
- **The parser recovers at the next statement.** `synchronize` in
  `src/parser.c` skips to a statement boundary after an error, so two broken
  statements give two errors.
- **There is no JSON code.** The one outside library is `lib/toml-c`, a single
  MIT licensed header that `src/main.c` includes to read `kirby.project.toml`.

### What is missing

1. **No errors in the editor.** A mistake shows up only when the program is run.
2. **No types or definitions in the editor.** The type checker knows them and
   no tool can ask. The [Type Info Proposal] adds tables for that.
3. **Errors are text, not data.** A tool has to read `[line N]` back out of
   stderr to know where an error is.
4. **Support for one editor only.** The hover provider is VS Code code. An
   editor that speaks LSP gets nothing.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- `krb lsp`: a language server on stdin and stdout.
- Opening, changing, and closing documents, with the whole text sent on each
  change.
- Diagnostics: every parse, type, and compile error in the document, published
  after every change.
- Hover: the type of the expression or declaration under the cursor.
- Go to definition: the declaration of the name under the cursor, in the same
  document.
- The VS Code extension starting `krb lsp`.
- Tests in the existing snapshot runner.

The following is intentionally left out of scope for this proposal:

- **Completion.** It needs types while the code around the cursor does not
  parse ([Q-parse-errors]).
- **Find references, document symbols, rename, and formatting.** Find
  references and document symbols are cheap once the [Type Info Proposal]'s
  tables exist, and can come straight after this proposal. Rename needs more.
  Formatting is the [Formatter Proposal]: once it exists, the server answers
  `textDocument/formatting` by calling its `formatSource` on the open
  document's text.
- **Several files.** Each open document is checked on its own, after the
  stdlib, as `krb run FILE` does. Programs made of several files wait for the
  [Modules Proposal] and the [Projects Proposal].
- **Changes sent as edits instead of whole text.** The whole text is a few KB
  and is checked in under a millisecond ([Appendix B]).
- **Checking only what changed.** Same reason.
- **Syntax colors.** The TextMate grammar stays. A Tree-sitter grammar is a
  separate idea.
- **A server in another language.** If Kirby is ever rewritten in another
  language, its server would be written there, against the same tests.

### Implementation Plan

#### Part 1: Messages in and out

<!-- [Part 1]: #part-1-messages-in-and-out -->

A new file, `src/lsp.c`, and a new `krb` command:

```diff
 static const Command commands[] = {
     {"init", cmdInit},     {"run", cmdRun},   {"example", cmdExample},
     {"repl", cmdRepl},     {"exec", cmdExec}, {"compile", cmdCompile},
     {"build", cmdBuild},   {"lex", cmdLex},   {"parse", cmdParse},
-    {"config", cmdConfig}, {"docs", cmdDocs}};
+    {"config", cmdConfig}, {"docs", cmdDocs}, {"lsp", cmdLsp}};
```

`cmdLsp` reads messages from stdin until stdin closes or the client sends
`exit`. Each message has a header and a body. The header holds
`Content-Length`, "the length of the content part in bytes", and the "header
and content part are separated by a '\r\n'" ([LSP 3.17]). Answers go to stdout
the same way. Nothing else is ever written to stdout. On Windows, stdin and
stdout are switched to binary mode first, so `\r\n` is not changed and the byte
counts stay right.

JSON is read and written with a small library in `lib/`, the way `lib/toml-c`
is used today. Which library is [Q-json].

The server answers:

| Message                   | What the server does                                                                        |
| ------------------------- | ------------------------------------------------------------------------------------------- |
| `initialize`              | Answers with what it supports ([Part 2] to [Part 5]) and the position encoding ([Q-encoding]) |
| `initialized`             | Nothing                                                                                     |
| `shutdown`                | Answers `null`, and refuses any later request                                               |
| `exit`                    | Ends `krb`                                                                                  |
| Any other request         | An error saying the method is not supported                                                 |
| Any other notification    | Nothing                                                                                     |

Before `initialize`, a request gets the error the specification asks for,
"`code: -32002`" ([LSP 3.17]).

#### Part 2: Documents

<!-- [Part 2]: #part-2-documents -->

The server keeps the text of every open document, keyed by its URI.
`textDocument/didOpen` adds one, `didChange` replaces its text (the server asks
for the whole text on each change in `initialize`), and `didClose` removes it
and clears its diagnostics. The server never reads open documents from disk.

#### Part 3: Diagnostics

<!-- [Part 3]: #part-3-diagnostics -->

**Errors as records.** The five error functions build a record and hand it to
one function, as well as writing the line they write today:

```c
typedef struct {
  int line;             // 1-based, as Token.line
  const char *at;       // The token's text in the source, or NULL
  int atLength;
  bool atEnd;           // The error is at the end of the input
  const char *message;
} KirbyError;

typedef void (*KirbyErrorSink)(void *user, const KirbyError *error);

void setErrorSink(KirbyErrorSink sink, void *user); // NULL: stderr only
```

This is the same change as Part 4 of the [Embedded Library Proposal], which
already plans to send these five functions' output through a hook. Whichever
proposal lands first makes the change, and the other uses it. When the
[Tooling Data Proposal] gives nodes spans, the record carries the span in place
of `line`, `at`, and `atLength`.

`krb lsp` sets a sink and turns off the stderr text. Stderr is not part of the
protocol, and the errors already reach the editor as diagnostics.

**One check per change.** After `didOpen` or `didChange` the server runs one
session on the document:

```c
typchkSessionBegin();
compileSource(KIRBY_STDLIB, /*typecheck=*/true); // shared with main.c
compileSource(documentText, /*typecheck=*/true);
// records arrive in the sink
```

It runs the compiler too, and throws the compiled unit away, because some
errors only the compiler finds, such as `Already declared in this scope.` and
`Too many constants in one chunk.` `compileSource` is in `src/main.c` today, and
moves where both can call it.

**Where an error is.** An error with `at` points into the document text the
server passed in, so its start is `at - documentText` bytes into the document,
and its end is `atLength` bytes further. The server turns those into a line and
a character with the text it holds ([Q-encoding]). An error with only a line
covers that whole line. An error at the end covers the last character.

**Publishing.** The server sends `textDocument/publishDiagnostics` with every
record for the document, all with severity Error and source `kirby`. A check
with no errors sends an empty list, which clears the old ones.

**What the server shows while the file does not parse** is [Q-parse-errors].

#### Part 4: Hover and go to definition

<!-- [Part 4]: #part-4-hover-and-go-to-definition -->

These need the [Type Info Proposal]. The check in [Part 3] passes a `TypeInfo`
to `typchkCheckProgramWithInfo`, and the server keeps the session open, so the
types in the tables stay valid until the next check.

**Hover** (`textDocument/hover`) turns the cursor into a byte offset and looks
for, in order:

1. A declaration whose name contains the offset. It shows `name: type`, for
   example `x: i64`, or the signature for a function.
2. The smallest expression that contains the offset. It shows its type.
3. Nothing.

The text is a `kirby` code block in markdown. A native's type comes from its
declaration in the tables ([Q-native-hover]).

**Go to definition** (`textDocument/definition`) finds the use that contains
the offset and answers with its declaration's place. A declaration in the same
document is answered with its range. A native or builtin has no place, and the
answer is empty. A declaration in the stdlib is [Q-stdlib-definition].

**One session at a time.** The checker keeps its state in file-level variables,
so only one session can exist at once. The server keeps the session of the
document it checked last. A request about another document checks that one
first, which [Appendix B] says is well under a millisecond for files of the
size in the tests. When the [Embedded Library Proposal]'s `FrontEnd` bundle
lands (its Part 2), the server can keep one bundle per document instead.

#### Part 5: The VS Code extension

<!-- [Part 5]: #part-5-the-vs-code-extension -->

`vsc/` gains the `vscode-languageclient` package and starts
`<kirby.executablePath> lsp`, using the setting that "Run Kirby File" already
uses. The hover provider for `vsc/hovers/*.md` stays until [Q-native-hover] is
settled, and runs next to the server's hover.

Other editors are configured by their users to run `krb lsp` for `.krb` files.
The book's tooling page shows how for Neovim, Helix, and Zed.

#### Part 6: Testing

<!-- [Part 6]: #part-6-testing -->

The snapshot runner already sends a test's `.in` file to stdin and compares
stdout with `.out` (`tests/README.md`). A server test is an `.argv` file with
one line, `lsp`, an `.in` file holding the messages the client sends, and an
`.out` file holding what the server sent back. The tests live in `tests/lsp/`.

Header lengths must match the body byte for byte, which is easy to get wrong by
hand. A script, `scripts/new-lsp-test.sh`, writes the `.in` file from a plain
file with one JSON message on each line. The tests use made up URIs such as
`file:///test.krb`, and the server never reads them from disk, so the output
does not depend on the machine.

Every test ends with `shutdown` and `exit`, so the `.exit` snapshot checks that
the server stops cleanly.

#### Part 7: Suggested order

<!-- [Part 7]: #part-7-suggested-order -->

1. `krb lsp` with `initialize`, `shutdown`, and `exit`, and the test script.
2. Documents ([Part 2]).
3. The error record and sink, unless the [Embedded Library Proposal] made it
   first.
4. Diagnostics ([Part 3]), with ranges from tokens and lines.
5. The VS Code client ([Part 5]). At this point the server is useful.
6. Exact ranges once the [Tooling Data Proposal]'s Parts 1 and 2 are done.
7. Hover and go to definition once the [Type Info Proposal] is done.

Steps 1 to 5 need nothing that is not built yet.

## Impacts

### Existing Syntax Or Behavior

- **No language change.**
- **`krb`** gains the `lsp` command.
- **The five error functions** also hand a record to a sink. With no sink set
  they behave as today, so no snapshot changes.
- **`compileSource`** moves out of `src/main.c` so `src/lsp.c` can call it.
- **A JSON library** is added under `lib/` ([Q-json]).
- **`vsc/`** gains a dependency, `vscode-languageclient`.

### Related Proposals

- [Type Info Proposal] — needed for hover and go to definition ([Part 4]). Not
  needed for diagnostics.
- [Tooling Data Proposal] — its spans give every error and every entry an exact
  range. Until then, errors from a token get the token's range and errors from a
  node get the whole line ([Part 3]). Its byte columns are turned into the
  editor's position encoding by the server ([Q-encoding]).
- [Embedded Library Proposal] — its Part 4 sends the five error functions'
  output through a hook. [Part 3] needs the same change, so the two share it.
  Its `FrontEnd` bundle (Part 2) would let the server keep a session per
  document. That proposal is updated to say so.
- [Debugger Proposal] — [Q-adapter] there asks whether the Debug Adapter
  Protocol, which is also JSON, should live in C inside `krb`. The JSON library
  this proposal adds would make that option cheaper. That proposal is updated to
  say so.
- [Modules Proposal] and [Projects Proposal] — checking a program of several
  files, and knowing which files are in it, waits for them. A server would look
  for the project root the same way `krb` does ([Q-root] there). Those
  proposals are updated to say so.
- [Formatter Proposal] — adds `krb format` and `formatSource`, which this
  server can call for `textDocument/formatting`. Its VS Code provider runs
  `krb format --stdin` until then, and is removed once the server formats. Both
  proposals add a row to the `commands` table in `src/main.c`, and either can
  land first.

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

Each test below is `tests/lsp/NAME.argv` containing `lsp`, with `NAME.in`
written by the script in [Part 6] from the messages shown. Headers are left out
here.

##### NEW: tests/lsp/initialize.argv

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"capabilities":{}}}
{"jsonrpc":"2.0","method":"initialized","params":{}}
{"jsonrpc":"2.0","id":2,"method":"shutdown"}
{"jsonrpc":"2.0","method":"exit"}
```

###### Expected Outcome

An answer to request 1 with the server's capabilities: whole-text sync, hover
and definition (once [Part 4] is done), and the position encoding. A `null`
answer to request 2. Exit code 0.

##### NEW: tests/lsp/request_before_initialize.argv

A `textDocument/hover` request before `initialize`, then `exit`.

###### Expected Outcome

An error answer with code `-32002`.

##### NEW: tests/lsp/diagnostics_type_errors.argv

After `initialize`, a `didOpen` of `file:///test.krb` with this text:

```kirby
let a: i64 = "one";
let b: string = 2;
```

###### Expected Outcome

One `publishDiagnostics` for `file:///test.krb` with two diagnostics, on lines
0 and 1 (LSP lines count from 0), with the messages `Expected i64, got string.`
and `Expected string, got i64.`

##### NEW: tests/lsp/diagnostics_cleared.argv

A `didOpen` with the text above, then a `didChange` whose whole text is
`let a: i64 = 1;`.

###### Expected Outcome

Two `publishDiagnostics`: the first with two diagnostics, the second with an
empty list.

##### NEW: tests/lsp/diagnostics_parse_error_range.argv

A `didOpen` with `let y = ;`.

###### Expected Outcome

One diagnostic, `Expect expression.`, whose range covers the `;`: line 0,
characters 8 to 9.

##### NEW: tests/lsp/diagnostics_compiler_error.argv

A `didOpen` with the text of `tests/run/structs/struct_shadowed_by_struct.krb`:

```kirby
struct Struct {}
struct Struct {}
```

###### Expected Outcome

One diagnostic, `Already declared in this scope.`, on line 1, characters 7 to
13. Only the compiler reports this error (`src/compiler.c`), so it proves the
compiler runs.

##### NEW: tests/lsp/hover_variable.argv

A `didOpen` with `let a = 1;\nlet b = a + 2;`, then a hover at line 1,
character 8.

###### Expected Outcome

A hover whose markdown is a `kirby` code block holding `a: i64`.

##### NEW: tests/lsp/definition_local.argv

The same document, then a definition request at line 1, character 8.

###### Expected Outcome

A location in `file:///test.krb`, line 0, characters 4 to 5.

##### NEW: tests/lsp/two_documents.argv

Two documents opened, each with a different error, then a hover in the first.

###### Expected Outcome

Each document gets only its own diagnostic. The hover answer is right for the
first document after the second was checked ([Part 4], one session at a time).

## Questions

### **Q:** Which JSON library?

<!-- [Q-json]: #q-which-json-library -->

**Status:** Open

- **(a) A small, MIT licensed C library in `lib/`**, such as cJSON, kept the way
  `lib/toml-c` is. Well tested, and it handles escapes and Unicode, which LSP
  text is full of.
- **(b) Written for Kirby.** The server reads a few fixed shapes and writes a
  few more. Less code to carry, but string escapes and `\u` sequences are where
  hand-written JSON goes wrong.

The [Debugger Proposal] may want the same library ([Q-adapter] there), so the
choice should suit both.

### **Q:** Which position encoding does the server use?

<!-- [Q-encoding]: #q-which-position-encoding-does-the-server-use -->

**Status:** Open

LSP counts characters on a line in an encoding the client and server agree on
in `initialize`. "The only mandatory encoding is UTF-16", and since 3.17 a
server picks "one of the encodings offered by the client" ([LSP 3.17],
[lsp-types Position]). Kirby counts columns in bytes ([Tooling Data Proposal],
Part 1).

- **(a) Use `utf-8` when the client offers it, and convert to `utf-16`
  otherwise** (proposed). Bytes need no converting. The server holds the
  document text, so it can convert a byte column for any client.
- **(b) Always `utf-16`.** One way of doing it, and the conversion runs on every
  position.

Which editors offer `utf-8` should be checked before choosing. Until then the
server must be able to do `utf-16`, since every client can ask for it.

### **Q:** What does the server show while the file does not parse?

<!-- [Q-parse-errors]: #q-what-does-the-server-show-while-the-file-does-not-parse -->

**Status:** Open

While someone types, the file often does not parse for a moment. Today a parse
error means no type errors and no type tables at all.

- **(a) Parse errors only, and keep the last tables** (proposed). The parse
  errors are shown. Hover and go to definition keep answering from the last
  check that parsed, as long as the positions they answer still fall inside the
  document.
- **(b) Parse errors only, and no hover.** The simplest, and the editor goes
  quiet at the moments it would help most.
- **(c) Check the statements that parsed.** Needs the checker to run after a
  parse error, which is [Q-partial] in the [Type Info Proposal].

### **Q:** Where do hovers for natives come from?

<!-- [Q-native-hover]: #q-where-do-hovers-for-natives-come-from -->

**Status:** Open

The extension has 86 hand-written markdown files for the natives, and the
server knows each native's type from `nativeSignatures`. `@print` and
`@println` have no entry there ([Type Info Proposal]).

- **(a) Keep both for now** (proposed). The server shows the type and the
  extension shows the markdown. Whether VS Code shows the two together well
  should be tried before this is settled.
- **(b) Move the markdown into `krb`.** One place, and every editor gets it. The
  files would be built into the binary, as the stdlib is.
- **(c) The server reads `vsc/hovers/`.** It would depend on where the
  extension is installed.

### **Q:** Where does go to definition go for a stdlib declaration?

<!-- [Q-stdlib-definition]: #q-where-does-go-to-definition-go-for-a-stdlib-declaration -->

**Status:** Open

The stdlib is built into `krb` as text (`scripts/generate_stdlib_c.sh`). An
installed `krb` may have no `stdlib.krb` file next to it.

- **(a) No answer** (proposed for the first version). Hover still works.
- **(b) Write the stdlib to a file** in a cache folder and answer with that
  file.
- **(c) A URI the server serves itself**, such as `kirby:stdlib.krb`. Needs the
  client to ask the server for the text, which not every client does.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Language Server Protocol (LSP)**: The set of JSON messages editors use to
  ask a language server about code.
- **Language server**: A program that answers an editor's questions about code
  in one language.
- **Client**: The editor, or the part of an editor extension, that starts the
  language server and talks to it.
- **Request**: A message that expects an answer. It has an `id`.
- **Notification**: A message that expects no answer.
- **Diagnostic**: An error or warning shown in the editor, with the range of
  text it is about.
- **URI**: The name LSP uses for a document, such as `file:///home/a/main.krb`.
- **Position encoding**: How the characters on a line are counted: in bytes
  (`utf-8`) or in UTF-16 code units (`utf-16`).
- **Hover**: Information an editor shows when the pointer rests on code.
- **Go to definition**: Moving to the declaration of the name under the cursor.
- **Session**: Everything between `typchkSessionBegin()` and
  `typchkSessionEnd()`.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals
[Appendix A]: #appendix-a--reproducing-the-baseline-claims
[Appendix B]: #appendix-b--measurements
[Part 1]: #part-1-messages-in-and-out
[Part 2]: #part-2-documents
[Part 3]: #part-3-diagnostics
[Part 4]: #part-4-hover-and-go-to-definition
[Part 5]: #part-5-the-vs-code-extension
[Part 6]: #part-6-testing
[Part 7]: #part-7-suggested-order

<!-- Questions -->

[Q-json]: #q-which-json-library
[Q-encoding]: #q-which-position-encoding-does-the-server-use
[Q-parse-errors]: #q-what-does-the-server-show-while-the-file-does-not-parse
[Q-native-hover]: #q-where-do-hovers-for-natives-come-from
[Q-stdlib-definition]: #q-where-does-go-to-definition-go-for-a-stdlib-declaration

<!-- Related proposals -->

[Type Info Proposal]: ../type-info/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Debugger Proposal]: ../debugger/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Formatter Proposal]: ../formatter/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-partial]: ../type-info/PROPOSAL.md#q-does-the-checker-keep-going-past-a-parse-error
[Q-adapter]: ../debugger/PROPOSAL.md#q-where-does-the-debug-adapter-protocol-live
[Q-root]: ../projects/PROPOSAL.md#q-how-is-the-project-root-found

<!-- Outside references -->

[LSP 3.17]: https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/
[lsp-types Position]: https://hackage-origin.haskell.org/package/lsp-types-2.1.0.0/docs/src/Language.LSP.Protocol.Internal.Types.Position.html

## Appendix A — Reproducing the baseline claims

On a clean build at `from_commit` (`just build`):

**The five error functions, and nothing else writes errors.**

```shell
grep -n 'fprintf(stderr, "\[line' src/*.c
# parser.c (parse_error_at), compiler.c (compilerErrorAt, compilerErrorAtToken),
# typecheck.c (typchkErrorAtToken, typchkErrorAtNode), and vm.c, which is the
# runtime error trace and not a compile error
grep -n 'fprintf(stderr' src/parser.c src/typecheck.c src/compiler.c \
  src/types.c src/definite_assignment.c src/resolved_impl_targets.c
# every other line is an out of memory message
```

**The front end never writes to stdout.**

```shell
grep -n '\bprintf(\|puts(\|putchar(\|fprintf(stdout' src/parser.c \
  src/typecheck.c src/compiler.c src/scanner.c src/types.c \
  src/definite_assignment.c src/ast.c src/lexer.c src/token_stream.c \
  src/native_signatures.c src/resolved_impl_targets.c
# nothing
```

**A parse error hides type errors, and the parser recovers per statement.**

```shell
printf 'let x: i64 = "one";\nlet y = ;\nlet z: bool = 3;\n' > /tmp/p.krb
./build/krb run /tmp/p.krb   # only the parse error on line 2
printf 'let a = ;\nlet b = 1;\nlet c = ;\n' > /tmp/q.krb
./build/krb run /tmp/q.krb   # errors on lines 1 and 3
```

**The extension.**

```shell
wc -l vsc/src/extension.ts   # 78
ls vsc/hovers | wc -l        # 86
```

## Appendix B — Measurements

One machine, so this shows the order of size and is not a benchmark.

A throwaway C program, linked against `build/libkirby_core.a` and
`build/libkirby_runtime.a` with `build/generated/stdlib.c`, ran a whole session
in a loop:

```c
for (int i = 0; i < iters; i++) {
  typchkSessionBegin();
  bool okStd = check(KIRBY_STDLIB); // parse, typchkCheckProgram, compile
  bool ok = check(src);             // the same, then astFreeAll()
  compilerSessionEnd();
  typchkSessionEnd();
}
```

`check` is `compileSource` from `src/main.c` with the compiled unit freed at
the end.

| File                                                         | Size    | Runs  | Same result every run | Peak memory      | Time per run |
| ------------------------------------------------------------ | ------- | ----- | --------------------- | ---------------- | ------------ |
| `examples/fizzbuzz.krb`                                      | small   | 2,000 | yes                   | 2,388 KB from run 100 on | —     |
| `tests/run/generics/generic_function_identity.krb`           | small   | 2,000 | yes                   | 2,348 KB from run 100 on | —     |
| `tests/run/native_functions/native_fns.krb`                  | 3,191 B | 1,000 | yes                   | —                | 0.56 ms      |
| `tests/run/types/impl_target_name_survives_arena_growth.krb` | 2,791 B | 1,000 | yes                   | —                | 0.58 ms      |

Peak memory is the process's maximum resident size, read with `getrusage`
after run 1, run 100, and the last run. It rose by about 130 KB between run 1
and run 100 and not at all after that. The largest `.krb` file in the tests is
5,335 bytes, so these are the sizes Kirby programs have today. The check was
not run under AddressSanitizer.
