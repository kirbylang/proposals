---
status: Draft
category: tooling
created: 2026-10-25
from_commit: b41332d
---

# Proposal: Formatter (`krb format`)

A formatter rewrites a program's layout (indentation, spaces, line breaks)
into one standard style, without changing what the program means. It ends
arguments about style, keeps diffs about code instead of spacing, and lets an
editor tidy a file on save.

This proposal adds `krb format`, written in C inside `krb`. It parses the file
with Kirby's real parser, prints the tree back out in the standard style, and
puts the comments and blank lines back where they were. Before it writes
anything, it parses its own output again and checks that the tree is the same
as the original and that no comment was lost. If that check fails, or the file
does not parse, the file is left alone.

The style has no settings in the first version. The style choices that are
still open (indent width, line width, trailing commas, and when a block stays
on one line) are listed as [Questions], with what the existing Kirby code does
today.

The title and the folder name are working titles.

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

- **No formatter.** `docs/CHANGELOG.md` lists "Formatter `krb format`" under
  Future State, next to `krb lint` and `krb test`. `just fmt` formats the C
  code with `clang-format`, not Kirby code.
- **Comments are thrown away by the scanner.** Kirby has line comments (`//`
  to the end of the line) and a first line starting `#!`. There are no block
  comments. `skipWhitespace` in `src/scanner.c` skips both kinds the same way
  it skips spaces, so no token, and nothing in the tree, records them:

  ```c
  case '/':
    if (peekNext(scanner) == '/') {
      while (peek(scanner) != '\n' && !isAtEnd(scanner))
        advance(scanner);
    } else {
      return;
    }
    break;
  ```

- **The tree does not keep how a literal was written.** A number keeps its
  value and kind (`LiteralNode` in `src/ast.h`), so `1.50` and `1.5` are the
  same node, and so are `12u8` and `12` given `u8`. A string keeps its text
  after escapes are decoded (`stringLiteral` in `src/parser.c`), so `"a\tb"`
  holds a real tab. Printing a literal from the tree would change how it is
  spelled.
- **The tree keeps groupings.** `(1 + 2)` is a `NODE_GROUPING`, so parentheses
  the programmer wrote can be printed back.
- **The tree has one line per node and no end.** `AstNode.line` is where a
  node starts for most kinds. A few kinds keep `endLine`, `declEndLine`, or
  `bodyEndLine`. Most do not, so the tree cannot say where one statement ends
  and so whether a blank line follows it.
- **`print_ast` is not the same tree.** `krb parse` prints the tree with
  `print_ast` in `src/ast.c`. It leaves out the bodies of methods in an `impl`,
  whether a field is `var` or `let`, and the kind of a number: `pub var value:
T;` prints as `(pub-field value : T)`, and `1u8` prints as `1`. Two files
  with the same `print_ast` output can be different programs.
- **`krb parse` exits with 0 after a parse error.** The parser reports the
  error, recovers at the next statement, and prints what it could build.
- **Kirby code does not agree on a style.** Across the 1,293 `.krb` files in
  `tests/`, `stdlib/`, `examples/` and `example_project/` (6,667 lines):

  | What                                              | Count                                   |
  | ------------------------------------------------- | --------------------------------------- |
  | Files whose first indented line uses 2 spaces     | 133 (of which `tests/run/generics`: 70) |
  | Files whose first indented line uses 4 spaces     | 184 (of which `tests/run/structs`: 81)  |
  | Files using 1 or 5 spaces                         | 2                                       |
  | Lines longer than 80 / longer than 100 characters | 33 / 14                                 |
  | Lines ending in spaces                            | 23                                      |
  | `{` alone on a line                               | 13                                      |
  | Blocks written on one line, such as `{ a + b }`   | 190                                     |
  | Own-line comments / comments after code           | 56 / 6                                  |
  | Tabs                                              | 0                                       |

  Some things everyone already does the same way: one space after `if`,
  `while` and `for` before `(` (94 of 94), a space in `name: Type` (148 of
  148), a space between `fun` and `(` in a lambda (48 of 48), and spaces
  inside the braces of a struct literal, `Box { value: value }` (127 of 127).

### What is missing

1. **No standard layout.** Two people writing the same code write it two ways,
   and a test file can use 2 spaces in one place and 4 in another.
2. **No way for an editor to tidy a file.** The VS Code extension in `vsc/`
   has no formatting provider.
3. **Nothing in the toolchain keeps comments.** Any tool that prints Kirby code
   (a formatter, a rename, a macro expansion shown to the user) would lose them.

## Proposed Changes

### Goals and Non Goals

What this proposal covers:

- `krb format PATH...`, which rewrites files in place, plus `--check`,
  `--stdout` and `--stdin`.
- One style for every construct Kirby has today, with no settings.
- Keeping every comment, the `#!` line, and single blank lines between
  statements.
- Keeping every literal exactly as it was written.
- A check, every time, that the output parses to the same tree and holds the
  same comments, so the formatter can never change a program.
- A formatting provider in the VS Code extension.
- Tests in the existing snapshot runner, and a script that runs the formatter
  over every `.krb` file in the repo.

The following is intentionally left out of scope for this proposal:

- **Settings.** No indent, width, or brace options ([Q-options]).
- **Breaking long expressions.** Only lists (arguments, parameters, array items,
  struct literal fields) are broken over lines when too long. A long `a + b +
c` or method chain stays on one line ([Q-long-lines]).
- **Sorting or moving code.** Nothing is reordered, added or removed, apart
  from spaces, line breaks and extra blank lines. Parentheses are never added
  or removed, even when they are not needed.
- **Formatting part of a file.** An editor's "format selection" formats the
  whole file. `textDocument/rangeFormatting` can come later.
- **Files that do not parse.** They are reported and left alone.
- **Folders and projects.** `krb format` takes file paths ([Q-dirs]).
- **A linter.** `krb lint` is a separate tool in the changelog's plan.
- **Reformatting the repo's own `.krb` files.** That is [Q-repo].

### Implementation Plan

#### Part 1: The scanner can keep comments

<!-- [Part 1]: #part-1-the-scanner-can-keep-comments -->

The scanner gets an optional list. When it is set, `skipWhitespace` adds each
comment it skips to it. When it is `NULL`, which is every case today, nothing
changes and nothing is recorded. Two new types in `src/scanner.h`:

```c
/**
 * A `//` comment, or the `#!` line. Recorded only when a tool asks.
 */
typedef struct {
  int startOffset; // Position of the first `/` or `#`
  int endOffset;   // Position just after the last byte, before the newline
  int line;
  bool ownLine; // Nothing but spaces before it on its line
} Comment;

typedef struct {
  Comment *items;
  int count;
  int capacity;
} CommentList;
```

and one field on `Scanner`:

```diff
   int interpBraces[MAX_INTERP_DEPTH];
+  /**
+   * Comments skipped so far, or NULL when nobody asked for them.
+   */
+  CommentList *comments;
 } Scanner;
```

`initScanner` sets `comments` to `NULL`, so every caller today is unchanged.
The formatter collects comments with its own pass of the real scanner over the
source, before it parses. It does not change `lex()` or `parse()`, and the
parser never sees a comment. Using the real scanner, and not a search for `//`,
matters: `"http://a"` holds `//` inside a string, and the scanner already knows
it is a string.

The text of a comment is never changed, apart from spaces at its end, which
are removed.

#### Part 2: Spans for blank lines and literal text

<!-- [Part 2]: #part-2-spans-for-blank-lines-and-literal-text -->

The formatter needs to know, for every node, where in the file it starts and
ends. Parts 1 and 2 of the [Tooling Data Proposal] give every node a span with
byte offsets and line and column for both ends ([td-1], [td-2]). The formatter
uses them for three things:

- **Literal text.** A literal is printed as the source text between its span's
  offsets, so `1.50`, `12u8`, `"a\tb"` and `$"x is {a}"` come out exactly as
  they went in. An interpolated string is printed whole, placeholders
  included. The expressions inside its `{` `}` are not formatted.
- **Blank lines.** There is a blank line between two statements when the next
  one starts more than one line after the previous one ends, and no comment
  fills the gap.
- **Placing comments.** A comment belongs to the gap between the two nodes
  whose offsets are on each side of it ([Part 4]).

The formatter reads only the tree side of the tooling data, never the compiled
output. It does not need Parts 3 to 7 of that proposal.

The source text has to live until formatting ends. `krb format` reads it and
keeps it, unlike `runFile`, which frees it after compiling.

#### Part 3: The style

<!-- [Part 3]: #part-3-the-style -->

A new file, `src/format.c`, walks the tree and writes into a `StrBuf`
(`src/strbuf.h`). It has one `switch` on `NodeKind` with no `default`. The
build uses `-Wall`, which includes `-Wswitch`, so a new kind of node that the
formatter does not handle is a warning the day it is added.

The rules, using an indent of 2 spaces ([Q-indent]) and a width of 100
([Q-width]):

**Lines and indentation**

- One statement or declaration per line.
- Each block, struct, `impl` and `trait` body is indented one level.
- No spaces at the end of a line. The file ends with exactly one newline.
- The `#!` line, if any, stays the first line, as written.

**Braces**

- `{` goes at the end of the line that opens it, after one space: `fun f() {`,
  `if (a) {`, `struct Point {`.
- `}` goes on its own line, at the indent of the line that opened it. An `else`
  goes on the same line: `} else {`, `} else if (b) {`.
- An empty body is `{}`: `struct Struct {}`, `Self {}`.
- A block with no statements and only a value, such as `{ a + b }`, stays on one
  line if it was written on one line and fits. Otherwise it is broken over lines
  ([Q-one-line]).

**Spaces**

| Where                                                         | Written as                          |
| ------------------------------------------------------------- | ----------------------------------- |
| Around a binary operator, `=`, `=>`, `as`, `&&`, `\|\|`, `??` | `a + b`, `x = 1`, `1 as u8`         |
| After `,` and after `:`; none before                          | `f(a, b)`, `a: f64`                 |
| After `if`, `while`, `for` and a lambda's `fun`               | `if (a)`, `fun (a: f64): f64 { a }` |
| None after a function's name or a call's callee               | `fun add(`, `add(1, 2)`             |
| None inside `( )` and `[ ]`                                   | `(a + b)`, `[1, 2]`, `Box[T]`       |
| One inside the braces of a struct literal                     | `Point { x: 1, y: 2 }`              |
| None after a unary operator                                   | `-a`, `!done`                       |
| In a `for` head, after each `;`                               | `for (var i = 0; i < 3; i = i + 1)` |
| In an array type                                              | `[T; 2]`                            |

**Blank lines**

- A run of blank lines between two statements becomes one blank line.
- A blank line where none was written is never added.
- Blank lines at the start or end of a block are removed.

**Lists**

A list is the arguments of a call, the parameters of a function, the items of
an array, the fields of a struct literal, or the type arguments in `[ ]`.

- A list goes on one line if the whole line fits in the width.
- Otherwise each item goes on its own line, one level deeper, and the closing
  bracket goes on its own line at the opening line's indent. There is no comma
  after the last item, because the parser does not accept one ([Q-trailing]).
- The outermost list is broken first. A list inside it is broken only if its
  own line is still too long.
- A list with a comment inside it is always broken.

```kirby
@println(
  someFunctionWithALongName(firstArgument, secondArgument),
  anotherFunctionWithALongName(third, fourth)
);
```

Anything that is still too long after its lists are broken stays too long
([Q-long-lines]).

#### Part 4: Comments

<!-- [Part 4]: #part-4-comments -->

Every comment from [Part 1] is placed in one of two ways:

- **A comment after code** (`ownLine` is false) stays at the end of the line
  that holds the code before it, after one space. If that code is the end of a
  list item, the item's line ends with the comment, which is why such a list is
  always broken.
- **An own-line comment** goes on its own line before the next statement,
  declaration, field, method, or list item, at that item's indent. A comment
  before a `}` is placed at the indent of the block's contents. Blank lines
  around it follow the rules for statements: one blank line before or after it
  stays one blank line.

**A comment the formatter cannot place is never dropped.** If a comment falls
somewhere that the rules above do not cover, such as between the `-` and the
number in `- // why\n 1`, the file is left alone and `krb format` reports the
line of the comment. The check in [Part 5] makes sure this is the only
outcome: a missing comment is a failed check.

#### Part 5: The check

<!-- [Part 5]: #part-5-the-check -->

After printing, the formatter parses its own output and compares:

1. **The trees.** A new function, `astEqual` in `src/ast.c`, walks both trees
   together and compares every field that came from the source: node kinds,
   names and operators by their text, literals by kind and value, and the flags
   such as `isPublic`, `isMutable` and `isLambda`. It skips spans and the fields
   the type checker fills in later. It is not built on `print_ast`, which leaves
   out method bodies, field mutability and number kinds.
2. **The comments.** Both lists hold the same comment texts in the same order.

If either differs, the output is a bug in the formatter. The file is not
written, and `krb format` says so, with the path. The cost is a second parse
and a second scan, which for a file of a few KB is well under a millisecond
([Language Server Proposal], Appendix B: about 0.55 ms for parse, check and
compile together).

The tests also check that formatting is **stable**: formatting the output a
second time changes nothing ([Part 8]). That is not checked on every run.

#### Part 6: The command

<!-- [Part 6]: #part-6-the-command -->

A new `krb` command:

```diff
 static const Command commands[] = {
     {"init", cmdInit},     {"run", cmdRun},   {"example", cmdExample},
     {"repl", cmdRepl},     {"exec", cmdExec}, {"compile", cmdCompile},
     {"build", cmdBuild},   {"lex", cmdLex},   {"parse", cmdParse},
-    {"config", cmdConfig}, {"docs", cmdDocs}};
+    {"config", cmdConfig}, {"docs", cmdDocs}, {"format", cmdFormat}};
```

The [Language Server Proposal] adds `{"lsp", cmdLsp}` to the same table.
Either can land first.

| Command                      | What it does                                                                |
| ---------------------------- | --------------------------------------------------------------------------- |
| `krb format PATH...`         | Formats each file in place. A file that is already formatted is not written |
| `krb format --check PATH...` | Writes nothing. Prints the path of each file that would change              |
| `krb format --stdout PATH`   | Writes nothing. Prints the formatted text of one file                       |
| `krb format --stdin`         | Reads source from stdin and prints the formatted text. For editors          |

- **Writing in place** writes to `PATH.tmp` next to the file and then renames it
  over the file, with `rename` from `<stdio.h>`, so a crash never leaves half a
  file.
- **Several paths** are formatted one by one. A problem with one file does not
  stop the others.
- **Errors** go to stderr. A parse error is printed as `krb compile` prints it
  today.

Exit codes, from the codes `src/common.h` already uses:

| Exit | When                                                                          |
| ---- | ----------------------------------------------------------------------------- |
| 0    | Every file was formatted, or with `--check`, every file was formatted already |
| 1    | `--check` found a file that would change                                      |
| 64   | Wrong arguments, the same as every other command's usage error                |
| 65   | A file did not parse (`EXIT_CODE_COMPILER_ERR`)                               |
| 70   | The check in [Part 5] failed, or a comment could not be placed                |
| 71   | A file could not be read or written (`EXIT_CODE_OS_ERR`)                      |

When several files fail, the highest code wins.

The usage text gains:

```diff
   lex <path>             Print the tokens of a file\n"
   parse <path>           Print the AST of a file\n"
+  format <path>...       Format files in place. --check lists the files\n"
+                         that would change. --stdout prints one file.\n"
+                         --stdin formats stdin to stdout\n"
```

The C entry point is shared with the language server:

```c
typedef enum {
  FORMAT_OK,
  FORMAT_PARSE_ERROR,   // Errors were reported. `out` is empty
  FORMAT_CHECK_FAILED,  // A formatter bug, or a comment with no place
} FormatStatus;

// Formats `source` into `out`. Nothing is written anywhere else.
FormatStatus formatSource(const char *source, StrBuf *out);
```

#### Part 7: Editors

<!-- [Part 7]: #part-7-editors -->

**VS Code.** `vsc/src/extension.ts` registers a document formatting provider
for `kirby`. It runs `<kirby.executablePath> format --stdin`, the setting "Run
Kirby File" already uses, sends the document's text, and replaces the whole
document with stdout if the exit code is 0. On any other code it changes
nothing and shows stderr in the output panel. "Format Document" and format on
save then work.

**The language server.** Once the [Language Server Proposal]'s `krb lsp`
exists, it can answer `textDocument/formatting` by calling `formatSource` on the
open document's text, with one edit that replaces it all. The VS Code provider
is then removed, so there is one path. Other editors use the server, or run
`krb format --stdin` themselves.

#### Part 8: Testing

<!-- [Part 8]: #part-8-testing -->

The work follows the repo's TDD loop: one failing test, see it fail, make it
pass.

- **E2E snapshots** in a new folder, `tests/format/`. Each test is a `.krb`
  file, badly formatted on purpose, with an `.argv` of `format`, `--stdout`,
  `$file`. The `.out` snapshot is the formatted file. See the [Testing Plan].
- **Stable output.** `scripts/test-format.sh` formats each `.out` in
  `tests/format/` again and fails if anything changes.
- **The whole repo.** The same script runs `krb format --stdout` on every
  `.krb` file under `tests/`, `stdlib/`, `examples/` and `example_project/`. A
  file must either be refused with a parse error (many tests in
  `tests/run/errors/` do not parse on purpose) or pass the check and be
  stable. It writes nothing. With 1,293 files it covers every kind of node
  the tests use, and a new kind of node is covered as soon as it has a test.
- **Unit tests** in `unit/`, in the style of `unit/lexer.c` and
  `unit/parser.c`: comment collection (own-line, after code, `#!`, `//` inside
  a string, a comment on the last line with no newline), and `astEqual` (equal
  trees, and one test for each field that must make two trees differ).

#### Part 9: Suggested order

<!-- [Part 9]: #part-9-suggested-order -->

1. Comment collection in the scanner ([Part 1]).
2. `astEqual` ([Part 5]).
3. `krb format --stdout` for every kind of node, with the check, without
   comments, blank lines or line breaking. Literals use their span text.
4. Blank lines and comments ([Part 4]).
5. Breaking lists ([Part 3]).
6. In place, `--check` and `--stdin` ([Part 6]), and `scripts/test-format.sh`.
7. The VS Code provider ([Part 7]).
8. `textDocument/formatting` in `krb lsp`, once that exists.

Steps 1 and 2 need nothing that is not built yet. Step 3 needs the [Tooling
Data Proposal]'s Parts 1 and 2, which are Ready.

## Impacts

### Existing Syntax Or Behavior

- **No language change.** No syntax, opcode, bytecode or behavior changes.
- **`Scanner`** gains one pointer, `NULL` for every caller today. `Comment` and
  `CommentList` are new types in `src/scanner.h`.
- **`src/ast.c`** gains `astEqual`. `print_ast` does not change, so the parser
  tests keep passing as they are.
- **`src/main.c`** gains `cmdFormat`, a row in `commands`, and lines in the
  usage text. The usage text is in snapshots (such as the `tests/build` tests
  of wrong arguments), and those snapshots change.
- **New files:** `src/format.c`, `src/format.h`, `tests/format/`,
  `scripts/test-format.sh`, and unit tests.
- **`vsc/`** gains a formatting provider.
- **Every later proposal that adds syntax** adds a case to the formatter's
  `switch`, and a test in `tests/format/`. The `-Wswitch` warning and the
  repo-wide script catch a missing case. Following the [Roadmap]'s rule that
  work a later proposal causes belongs to the later proposal, that case is
  part of the proposal that adds the syntax, not a reopening of this one.

### Related Proposals

- [Tooling Data Proposal] — this proposal depends on its Parts 1 and 2 (a span
  on every node) for blank lines, literal text, and placing comments. It left
  comments and blank lines out, as "which a formatter would need". That
  proposal is updated to point here.
- [Language Server Proposal] — left formatting out of its first version. Its
  server can answer `textDocument/formatting` with `formatSource` ([Part 7]).
  Both proposals add a row to the `commands` table in `src/main.c`. That
  proposal is updated to say so.
- [Enums Proposal] and [Pattern Matching Proposal] — each allows a trailing
  comma (after the last variant, after the last arm), which no list in Kirby
  allows today. Whether every list should allow one is [Q-trailing]. Those
  proposals are updated to say so.
- [Tuples Proposal] — a tuple with one item needs a trailing comma, `(1,)`,
  to tell it apart from a grouping. The formatter must always keep that comma,
  whatever [Q-trailing] decides. That proposal is updated to say so.
- [Projects Proposal] — if the style ever gets settings ([Q-options]), or
  `krb format` with no path formats a project ([Q-dirs]), they would live in
  `kirby.project.toml`. Nothing changes there until those are answered.
- [Macros Proposal] — the formatter reads only source text, so code a macro
  generates is never formatted. A macro call is formatted like any other syntax
  once it has a kind of node.
- [Embedded Library Proposal] — its Part 4 sends compile errors through a
  hook. `krb format` reports parse errors the same way `krb compile` does, so it
  follows whatever that proposal changes.

### Testing Plan

How do we know the implemented proposal works?

#### E2E Tests

A test is an `.argv` file: the arguments for `krb`, one per line. The program it
runs is the file of the same name without `.argv`, written `$file` in the
arguments. Language tests go in `tests/run/` and run with `run` and `$file`;
tests of other commands have their own folders, such as `tests/exec/`. A
program without an `.argv` file is not a test. See `tests/README.md` in
kirbylang.

Unless a test says otherwise, its `.argv` is:

```
format
--stdout
$file
```

##### NEW: tests/format/spacing.krb

```kirby
var a=1+2*3;
fun add(a:f64,b:f64):f64{ a+b }
if(a>1){@println(a);}else{@println(0);}
var p=Point{x:1,y:2};
var f=fun(n:f64):f64{-n};
```

###### Expected Outcome

`.out` holds the file in the standard style, and the exit code is 0:

```kirby
var a = 1 + 2 * 3;
fun add(a: f64, b: f64): f64 { a + b }
if (a > 1) {
  @println(a);
} else {
  @println(0);
}
var p = Point { x: 1, y: 2 };
var f = fun (n: f64): f64 { -n };
```

The formatter does not type check, so `Point` need not be declared.

##### NEW: tests/format/comments.krb

```kirby
#!/usr/bin/env krb
// Adds two numbers


fun add(a: f64, b: f64): f64 {

    // the sum
    a + b   // trailing

}
```

###### Expected Outcome

```kirby
#!/usr/bin/env krb
// Adds two numbers

fun add(a: f64, b: f64): f64 {
  // the sum
  a + b // trailing
}
```

##### NEW: tests/format/literals.krb

```kirby
var a = 1.50;
var b = 12u8;
var s = "tab\there";
var i = $"a is {  a  }";
var g = (1 + 2);
```

###### Expected Outcome

`.out` is the same as the file. Literals keep their spelling, the interpolated
string is printed as written, and the parentheses stay.

##### NEW: tests/format/long_list.krb

```kirby
@println(someFunctionWithALongName(firstArgument, secondArgument), anotherFunctionWithALongName(third, fourth));
```

###### Expected Outcome

The line is 112 characters, so the outer list is broken and the inner ones fit:

```kirby
@println(
  someFunctionWithALongName(firstArgument, secondArgument),
  anotherFunctionWithALongName(third, fourth)
);
```

##### NEW: tests/format/declarations.krb

```kirby
struct Box[T]{pub var value:T;}
impl Box[T]{
pub fun get(self):T=self.value;
}
trait Show{fun show(self):string;}
type Pair[T]=[T;2];
```

###### Expected Outcome

```kirby
struct Box[T] {
  pub var value: T;
}
impl Box[T] {
  pub fun get(self): T = self.value;
}
trait Show {
  fun show(self): string;
}
type Pair[T] = [T; 2];
```

No blank lines are added between the declarations, because none were written.

##### NEW: tests/format/parse_error.krb

```kirby
var a = ;
```

###### Expected Outcome

`.err` holds the parse error as `krb compile` prints it, `.out` is empty, and
the exit code is 65.

##### NEW: tests/format/check_changed.krb

`.argv` is `format`, `--check`, `$file`.

```kirby
var a=1;
```

###### Expected Outcome

`.out` holds the file's path, the exit code is 1, and the file is not changed.

##### NEW: tests/format/check_unchanged.krb

`.argv` is `format`, `--check`, `$file`.

```kirby
var a = 1;
```

###### Expected Outcome

`.out` is empty and the exit code is 0.

##### NEW: tests/format/stdin

`stdin.argv` is `format`, `--stdin`. `stdin.in` holds `var a=1;`.

###### Expected Outcome

`.out` is `var a = 1;` and a newline, and the exit code is 0.

##### NEW: tests/format/no_paths

`no_paths.argv` is `format`.

###### Expected Outcome

The usage text on stderr, and the exit code is 64.

## Questions

### **Q:** How wide is one level of indent?

<!-- [Q-indent]: #q-how-wide-is-one-level-of-indent -->

**Status:** Open

Kirby code today is split: 133 files use 2 spaces and 184 use 4
([Problem Statement]). Options:

- **(a) 2 spaces** (used in this proposal). The newest tests use it
  (`tests/run/generics`, 70 of 72 files, added with generic types), most of the
  samples in the proposals use it, and so does the C code (`BasedOnStyle:
LLVM` in `.clang-format`).
- **(b) 4 spaces.** More files use it today, mostly the older
  `tests/run/structs` and `tests/run/types`.
- **(c) Tabs.** No Kirby file uses a tab.

Whichever is chosen, about half the files would change if [Q-repo] reformats
them.

### **Q:** How long can a line be?

<!-- [Q-width]: #q-how-long-can-a-line-be -->

**Status:** Open

Of 6,667 lines, 33 are longer than 80 characters and 14 longer than 100.
Options: **(a) 80**, the C code's limit (LLVM style). **(b) 100** (used in
this proposal), which breaks fewer of the lines people write today. Kirby has
long native names, such as `@strConcat`, and generic types, which push lines
out. **(c) 120**.

### **Q:** Can a project change the style?

<!-- [Q-options]: #q-can-a-project-change-the-style -->

**Status:** Open

- **(a) No settings** (proposed). Every Kirby file looks the same, and no
  arguments move from code to config. This is what `gofmt` and `zig fmt` do.
- **(b) A few settings** (indent, width) in `kirby.project.toml`, as `rustfmt`
  and Prettier allow. A file formatted outside its project (`--stdin`, a single
  file) would need to find the project first ([Projects Proposal]).

(a) can become (b) later. (b) cannot become (a) without reformatting everyone's
code.

### **Q:** Should lists broken over lines end with a comma?

<!-- [Q-trailing]: #q-should-lists-broken-over-lines-end-with-a-comma -->

**Status:** Open

Today the parser rejects a comma after the last item of every list: `[1, 2,
3,]` and `@println(1, )` are "Expect expression." errors, and `struct P { x:
f64, }` is "Expect field declaration." The [Enums Proposal] and the [Pattern
Matching Proposal] each allow one, and the [Tuples Proposal] needs one for a
tuple with one item.

- **(a) No trailing comma** (used in this proposal). No language change. Adding
  an item to the end of a broken list changes two lines: the new one, and the
  comma on the old last one.
- **(b) Allow a trailing comma in every list, and always print one when a list
  is broken.** One line changes when an item is added, as in Rust, Go and
  Prettier's default. It is a parser change, made before or with this
  proposal, and the enums and pattern matching rules become the general rule.
- **(c) Allow it, and keep whatever was written.** The output then depends on
  the input, so two people's formatted code can still differ.

In every case, the comma in a one-item tuple `(1,)` is kept.

### **Q:** When does a block stay on one line?

<!-- [Q-one-line]: #q-when-does-a-block-stay-on-one-line -->

**Status:** Open

190 lines in the repo hold a block on one line, such as `fun add(a: f64, b:
f64): f64 { a + b }` or a lambda `fun (n: f64): f64 { -n }`.

- **(a) Keep it on one line if it was written on one line, has no statements,
  and fits** (used in this proposal). Nobody's short functions get stretched to
  three lines. The output depends a little on the input, the way `gofmt` keeps
  a short function on one line if it was written so.
- **(b) Put it on one line whenever it has no statements and fits**, however
  it was written. One answer for every input, but a function written on three
  lines is pulled onto one.
- **(c) Always break a block with something in it.** The simplest rule, and the
  most lines.

### **Q:** What happens to a line that is still too long?

<!-- [Q-long-lines]: #q-what-happens-to-a-line-that-is-still-too-long -->

**Status:** Open

The first version breaks only lists. A long chain of `+`, `&&` or `??`, or a
long chain of method calls, stays on one line. Options for later: break before
each operator of the outermost chain, or before each `.` of a method chain, one
level deeper. Either needs a rule for where comments go inside an expression.
Leaving it out first keeps the formatter small, and only 14 lines in the repo
are over 100 characters.

### **Q:** What does the formatter do with a folder, or with no path?

<!-- [Q-dirs]: #q-what-does-the-formatter-do-with-a-folder-or-with-no-path -->

**Status:** Open

Today `krb format` needs file paths, and no path is a usage error. Options:

- **(a) A folder formats every `.krb` file in it**, recursively. Plain C has no
  way to list a folder, so this needs POSIX `opendir` and its Windows
  equivalent. `install-windows.cmd` shows Windows is a target.
- **(b) No path formats the project**, the way `krb run` with no path runs the
  project's `bin`. A project does not yet say which files belong to it
  ([Projects Proposal]).
- **(c) Neither, for now** (proposed). The shell can pass the files:
  `krb format $(find . -name '*.krb')`.

### **Q:** Should the repo's own Kirby files be formatted?

<!-- [Q-repo]: #q-should-the-repos-own-kirby-files-be-formatted -->

**Status:** Open

Every `.err` snapshot holds a bytecode listing with line numbers in it
([Tooling Data Proposal], Q-line), and many tests check a line number in an
error. Formatting a test moves its lines, so its snapshots change, and each
change has to be looked at.

- **(a) Leave them** (proposed for the first version). The repo-wide script in
  [Part 8] still runs the formatter over them without writing.
- **(b) Format them all in one commit**, with `--update`, once the style
  questions are answered. A large one-time change, and after it `--check` can
  run in CI.
- **(c) Format a file when it is next changed for another reason.**

Tests in `tests/run/errors/` that do not parse on purpose are never formatted.
Neither is a test whose layout is the point, such as one about a comment, or a
string over several lines.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Formatter**: A tool that rewrites the layout of code (spaces, line breaks,
  indentation) into one standard style, without changing what the code means.
- **Style**: The set of layout rules the formatter follows.
- **Own-line comment**: A comment with nothing but spaces before it on its line.
- **Comment after code**: A comment that follows code on the same line, such as
  `a + b // trailing`.
- **List**: Items between brackets and separated by commas: call arguments,
  function parameters, array items, struct literal fields, or type arguments.
- **Breaking a list**: Putting each item of a list on its own line.
- **Width**: The longest a line may be before the formatter breaks a list on
  it, counted in bytes.
- **Trailing comma**: A comma after the last item of a list.
- **Span**: The place in the source a node came from, as byte offsets and as
  line and column. Defined in the [Tooling Data Proposal].
- **Stable**: Formatting the formatter's output again gives the same text.
- **In place**: Writing the result over the file it was read from.
- **Snapshot**: A file holding a test's expected output, which the test runner
  compares against.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals
[Problem Statement]: #problem-statement
[Testing Plan]: #testing-plan
[Appendix A]: #appendix-a--reproducing-the-baseline-claims
[Part 1]: #part-1-the-scanner-can-keep-comments
[Part 2]: #part-2-spans-for-blank-lines-and-literal-text
[Part 3]: #part-3-the-style
[Part 4]: #part-4-comments
[Part 5]: #part-5-the-check
[Part 6]: #part-6-the-command
[Part 7]: #part-7-editors
[Part 8]: #part-8-testing
[Part 9]: #part-9-suggested-order

<!-- Questions -->

[Q-indent]: #q-how-wide-is-one-level-of-indent
[Q-width]: #q-how-long-can-a-line-be
[Q-options]: #q-can-a-project-change-the-style
[Q-trailing]: #q-should-lists-broken-over-lines-end-with-a-comma
[Q-one-line]: #q-when-does-a-block-stay-on-one-line
[Q-long-lines]: #q-what-happens-to-a-line-that-is-still-too-long
[Q-dirs]: #q-what-does-the-formatter-do-with-a-folder-or-with-no-path
[Q-repo]: #q-should-the-repos-own-kirby-files-be-formatted

<!-- Proposals -->

[Roadmap]: ../../ROADMAP.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Language Server Proposal]: ../language-server/PROPOSAL.md
[Enums Proposal]: ../enums/PROPOSAL.md
[Pattern Matching Proposal]: ../pattern-matching/PROPOSAL.md
[Tuples Proposal]: ../tuples/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Macros Proposal]: ../macros/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md

<!-- Parts of other proposals -->

[td-1]: ../tooling-support-data/PROPOSAL.md#part-1--the-span
[td-2]: ../tooling-support-data/PROPOSAL.md#part-2--a-span-on-every-ast-node

## Appendix A — Reproducing the baseline claims

The claims marked "checked" can be tried on a clean build at `from_commit`:

```shell
bash scripts/build.sh   # produces build/krb
```

**Comments are skipped by the scanner.**

```shell
grep -n -A20 'static void skipWhitespace' src/scanner.c   # '#!' and '//' cases
```

**The tree does not keep literal spelling, and `print_ast` drops fields.**

```shell
cat > /tmp/a.krb <<'EOF'
struct Box[T] {
  pub var value: T;
}
impl Box[T] {
  pub fun get(self): T = self.value;
}
var a = 1.50;
var b = 12u8;
var s = "a\tb";
var g = (1 + 2);
EOF
./build/krb parse /tmp/a.krb
# (struct Box[T] (pub-field value : T))      <- no 'var'
# (impl Box[T] (pub-method get))             <- no body
# (var a 1.5)
# (var b 12)                                 <- no 'u8'
# (var s "a	b")                              <- a real tab
# (var g (group (+ 1 2)))
```

**`krb parse` exits with 0 after a parse error.**

```shell
printf 'var a = ;\n' > /tmp/b.krb
./build/krb parse /tmp/b.krb; echo $?   # [line 1] Error at ';': ...  then 0
```

**No list allows a trailing comma.**

```shell
for s in 'var a = [1, 2, 3,];' '@println(1, );' 'struct P { x: f64, }'; do
  printf '%s\n' "$s" > /tmp/c.krb; ./build/krb parse /tmp/c.krb
done
# Expect expression. / Expect expression. / Expect field declaration.
```

**Style counts.** Over every `.krb` file in `tests`, `stdlib`, `examples` and
`example_project`:

```shell
F=$(find tests stdlib examples example_project -name '*.krb')
for f in $F; do grep -m1 -oE '^ +' $f | awk '{print length}'; done | sort | uniq -c
cat $F | awk 'length>80{a++} length>100{b++} END{print a, b}'   # 33 14
cat $F | grep -cE '[ ]+$'                                        # 23
cat $F | grep -cE '^\s*\{\s*$'                                   # 13
cat $F | grep -cE '\{ [^{}]+ \}'                                 # 190
cat $F | grep -cE '^\s*//'                                       # 56
cat $F | grep -cE '[;{}] +//'                                    # 6
cat $F | grep -oE '\b(if|while|for) ?\(' | sort | uniq -c        # all with a space
cat $F | grep -oE '(=|\(|,) ?fun ?\(' | sed -E 's/^[=(,] ?//' | sort | uniq -c  # 48 'fun ('
```
