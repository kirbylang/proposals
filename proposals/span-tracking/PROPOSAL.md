---
status: Draft
created: 2026-09-18
from_commit: bc3ae67
---

# Proposal: Span Tracking — Foundations for Tooling

This proposal makes every piece of Kirby syntax carry a [Span]: a record of
where in the original source it came from. Kirby's tokens already carry a line
number; this extends that idea to every AST node, and — crucially — to syntax
that is _generated_ rather than typed by the programmer, so that a generated
piece of code can point back to the source responsible for it. Strong tooling
(precise error messages, go-to-definition, find-references, autocomplete)
depends on this information existing everywhere, and it is far cheaper to build
in from the start than to retrofit.

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
  after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code
  blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Good tooling needs to answer, for any piece of a program, "where did this come
from?" Today Kirby can answer that only partially, and only for code the
programmer typed directly.

What exists today, checked against a clean build: Kirby's `Token`
(`src/token.h`) carries a pointer into the source, a length, and a line number:

```c
typedef struct {
  TokenType type;
  const char *start;
  int length;
  int line;
} Token;
```

Two gaps follow from this:

1. **Location is line-only and lives on tokens, not on the tree.** There is no
   column information, and AST nodes do not systematically carry a location of
   their own. An error about an expression can point at a line, not at the exact
   span of that expression.

2. **There is no notion of _generated_ origin at all.** This is not a problem
   today because Kirby cannot generate code — but the [Macros Proposal] changes
   that. Once code can be generated, the code that runs is not always the code
   the programmer typed. A method may exist only because a derive macro produced
   it; a type error may occur inside generated syntax. If tools describe
   generated code by its own (synthetic) location, the programmer sees errors
   pointing at code they never wrote. This is the single most common complaint
   about macro systems in practice.

The fix for both is the same: a Span on every piece of syntax, carried
faithfully through every stage that creates or rewrites syntax — including macro
expansion.

### Relationship to other proposals

- The [Macros Proposal] needs generated syntax to carry a span pointing back to the
  macro call responsible. It also needs per-syntax scope information for
  Hygiene, which is the same shape of per-syntax origin data. This proposal
  provides the foundation both rely on.
- The generics work in the type-system proposal creates and rewrites types and
  syntax (substitution, instantiation); as it does, it should set spans rather
  than leave them to be added later.

## The Changes

### Part 1 — A span type

Introduce a Span that records a source location precisely enough for tooling:
which file, and a start and end position (line and column, or byte offsets from
which line/column can be derived). The exact representation is [Q-repr].

Conceptually:

```c
typedef struct {
  int fileId;      // which source file
  int startOffset; // start position in that file
  int endOffset;   // end position in that file
  // origin: for generated syntax, where this came from. See Part 3.
} Span;
```

### Part 2 — A span on every AST node

Every AST node carries a span covering the source it was parsed from. The
parser, which already has the tokens' positions, sets each node's span as it
builds the node. This replaces "the error is on line N" (derived from a token)
with "the error covers this exact span," which is what go-to-definition,
find-references, and precise underlines need.

### Part 3 — Spans that survive generation

The essential requirement: when a macro (or any future syntax-producing step)
generates syntax, the generated syntax carries a span that points back to the
source the programmer actually wrote — the macro call — and, through nested
macros, back through each layer.

This means a span is not only "a range in a file" but can also carry an
**origin**: a link to the span of the call or construct that produced it. A tool
or error message can then always translate "this happened in generated code"
into "here is the source line responsible." The exact way origin is chained
through nested expansion is [Q-origin].

### Part 4 — Build it in from the start

Span tracking must be part of the AST/syntax data structures from the beginning.
Retrofitting "where did this come from" onto data structures that were not built
to carry it is a well-known source of pain: every place that constructs or
rewrites syntax has to be revisited. Building it in from the start — every node
has a span, every syntax-producing operation sets a meaningful one — avoids
that.

This has an ordering consequence for the other proposals: the generics and macro
work should set spans as they create and rewrite syntax and types, not leave
spans to be added afterward.

## Questions

### **Q:** What is the exact representation of a span?

<!-- [Q-repr]: #q-what-is-the-exact-representation-of-a-span -->

**Status:** Open

How precisely should a span record a location?

Options: (a) line and column pairs for start and end; (b) byte offsets into the
source, from which line/column are computed on demand; (c) a hybrid. Offsets are
compact and cheap to carry but need the source text to render as line/column;
line/column pairs are directly human-readable but larger. This also interacts
with how multiple source files are identified (see [Q-files]).

### **Q:** How is origin chained through nested macro expansion?

<!-- [Q-origin]: #q-how-is-origin-chained-through-nested-macro-expansion -->

**Status:** Open

When a macro expands into a call to another macro, generated syntax has more
than one layer of "where did this come from."

Options: (a) each generated span points only at its immediate producer, and
tools walk the chain; (b) spans carry a full chain of origins. (a) is smaller
per node but pushes work to consumers; (b) is richer but larger. This is the
same information Hygiene in the macro proposal needs, so the two should be
decided together.

### **Q:** How are multiple source files identified?

<!-- [Q-files]: #q-how-are-multiple-source-files-identified -->

**Status:** Open

A span must say _which_ file it points into. Today Kirby compiles from a single
source at a time; once modules and multi-file programs exist, spans must
distinguish files.

Options: (a) an integer file id into a table of file paths held by the compile
session; (b) storing paths directly on spans (larger). This interacts with how
the module system names and loads files.

### **Q:** How much does per-node span storage cost, and does it matter?

<!-- [Q-cost]: #q-how-much-does-per-node-span-storage-cost-and-does-it-matter -->

**Status:** Open

Putting a span on every AST node adds memory to every node.

Options: (a) store a full span inline on every node; (b) store a compact handle
on each node that indexes a side table of spans. (a) is simplest; (b) is smaller
per node but adds an indirection. Whether this matters depends on how large ASTs
get in practice, which is not yet measured.

## Glossary

These are both technical and non technical terms used throughout the proposal.

- **Changes**: Changes refer the proposed changes in this document
- **Span**: A record of where a piece of syntax came from in the original source
  — which file, and which positions. For generated syntax, a span can also carry
  an origin pointing back to the source that produced it.
- **AST**: The tree-shaped data structure the parser produces from source text.
  Every later stage reads or rewrites this tree. In Kirby this is the `AstNode`
  type in `src/ast.h`.
- **Hygiene**: The property that names a macro introduces cannot accidentally
  clash with names in the code that used the macro. It relies on the same
  per-syntax origin information spans carry.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions

<!-- Proposals -->

[Macros Proposal]: ../macros/PROPOSAL.md

<!-- Questions -->

[Q-repr]: #q-what-is-the-exact-representation-of-a-span
[Q-origin]: #q-how-is-origin-chained-through-nested-macro-expansion
[Q-files]: #q-how-are-multiple-source-files-identified
[Q-cost]: #q-how-much-does-per-node-span-storage-cost-and-does-it-matter
