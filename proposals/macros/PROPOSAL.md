---
status: Draft
created: 2026-09-18
from_commit: bc3ae67
---

# Proposal: Macros — Compile-Time Code That Produces Code

This proposal adds macros to Kirby: ordinary Kirby functions that run during
compilation and transform syntax. A macro takes a piece of the program's own
syntax and returns a piece of syntax, and the compiler replaces each macro call
with what the macro returns. Macros are a separate system from generics. Their
hygiene is automatic and composes through nested macros. Macro expansion is its
own pass that runs before type checking.

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

Kirby has no way to generate code at compile time. Every pattern a programmer
writes must be written out by hand, every time. The clearest example is
auto-generating a trait implementation from a struct's shape — the equivalent
of "derive `Eq` for this struct" — which today has to be written by hand for
each struct even though the code is mechanical.

A Macro fills this gap: code that runs while the program is being compiled and
produces more code. The design question is not _whether_ to have macros but
_what shape_ they take, because that shape determines how powerful they are, how
surprising they are, and how well tooling can see through them.

This proposal takes the position that macros should be **ordinary Kirby code**
operating on Syntax as data — not a separate pattern-matching mini-language,
and not fused with the generics system.

### Relationship to other proposals

- This proposal depends on macros being able to carry origin information on
  every piece of syntax they produce, so error messages and tools can point
  back to the source the programmer actually wrote. That machinery is described
  in the [Tooling Data Proposal]; this proposal assumes it exists and uses it
  for Hygiene as well.
- This proposal is independent of the type-system proposal in the sense that
  expansion runs entirely before type checking (see [The Pipeline]). It does
  not touch generics, bounds, or monomorphization.
- The [Debugger Proposal] needs to tell code the programmer wrote from code a
  macro generated. The origin information on spans is what lets a debugger step
  over generated code, or show the macro call instead. A variable a macro
  introduces has a name the programmer never wrote, so the local variable
  records in the [Tooling Data Proposal] (its Part 6) should be able to mark it
  as generated. Macros run on the VM while compiling
  (Part 3), and debugging that is outside the scope of the debugger proposal.

## The Changes

### Part 1 — What a macro is

A macro is an ordinary Kirby function with one difference: it runs during
compilation, and instead of ordinary values it receives and returns Syntax —
pieces of the program's own AST. The compiler replaces each macro call with
the syntax the macro returns, then carries on.

This is deliberately "macros are just Kirby code." A macro is written in Kirby,
using Kirby's ordinary control flow and data structures, operating on syntax as
data. There is no separate macro mini-language and no pattern-template
sub-language to learn.

### Part 2 — Separate from generics

Some languages make one compile-time-execution mechanism serve as _both_ the
macro system and the generics system. This design keeps them separate. Generics
are declared, checked, bounded, and monomorphized in their own proposal. Macros
are a different tool for a different job: generating and transforming syntax.

Keeping them apart means generics stay statically checkable from signatures
alone (which the module system relies on) and are not turned into "run some code
and see what comes out." The power of macros comes from being real code over
syntax, not from merging them with the type system.

### Part 3 — Running macros: reuse the VM

A macro is Kirby code, so the compiler can run it the same way Kirby runs any
code: compile the macro to bytecode and execute it on an instance of the
existing virtual machine, at compile time, handing it the syntax of its call and
taking back the syntax it returns. No separate interpreter for macros needs to
be built; the VM Kirby already has is the macro evaluator.

### Part 4 — Hygiene

Hygiene is the guarantee that names a macro introduces cannot accidentally
collide with names in the code that called it, and that names the caller passes
in resolve in the caller's world, not the macro's. A macro that introduces a
helper variable `tmp` must not clobber a `tmp` the caller already has.

The requirement for Kirby: hygiene is **automatic** and **composes** — it keeps
working when a macro expands into a call to another macro, without the macro
author placing manual "escape" annotations to get ordinary cases right.
Automatic, composable hygiene is a solved problem in the Lisp/Scheme tradition:
attach, to each piece of syntax, information about the scope it came from, and
honor that information during name resolution rather than comparing names as
bare text.

The practical implication: a syntax value is not just a name and a shape; it
also carries where it came from. That is the same per-syntax origin information
the [Tooling Data Proposal] introduces for error reporting, so the two are one
design, not two.

### Part 5 — Where expansion sits in the pipeline

Macro expansion is its own pass, and it runs **before** type checking:

```
parse  ->  expand macros  ->  type check  ->  (generics) monomorphize  ->  compile
```

The type checker then only ever sees fully expanded code and needs no awareness
of macros. This ordering fits Kirby's existing checker, which already runs as an
ordered sequence of whole-program passes. Whether any macro will ever need
name-resolution information _during_ expansion (which would force interleaving
expansion with resolution) is [Q-resolve].

### Part 6 — Declaration-producing macros

A useful macro often needs to produce a whole new **declaration**, not just an
expression — for example, generating an `impl` block. This is a larger
capability than "a macro that expands to an expression," and the macro system
should support it explicitly. It is the foundation for derive-style macros.

### Part 7 — Derive-style macros

Auto-generating a trait implementation from a struct's shape is expressible with
the pieces above, without a separate feature: a derive is a
declaration-producing macro that reads a struct's fields (via Reflection) and
returns an `impl` block.

A sensible build order, smallest useful piece first:

1. **Reflection over a concrete type.** Give compile-time code a way to ask, of
   a struct, "what are your fields and their types?"
2. **One hard-coded derive, end to end.** Pick a single trait (`Eq` is a natural
   first target since it already exists and compares field by field) and
   generate its `impl` as a special case, to prove a generated `impl` can be
   injected and behaves exactly like a hand-written one.
3. **Generalize.** Turn the special case into an ordinary declaration-producing
   macro any programmer could write.

This is a capability the macro and reflection design should _enable_, not
immediate work. The exact form Reflection takes, and whether it is shared with
the module system's interface data, is [Q-reflect].

## Questions

### **Q:** Will macros ever need resolved names during expansion?

<!-- [Q-resolve]: #q-will-macros-ever-need-resolved-names-during-expansion -->

**Status:** Open

Is "expand fully, then type-check" always sufficient, or will some macros need
name-resolution information mid-expansion?

Options: (a) keep the clean phase split (far simpler, and the proposal);
(b) interleave expansion with name resolution if a real need appears, as some
languages do. Whether any intended macro genuinely needs resolved names before
expansion finishes is not yet known; if one does, the pipeline would need the
more complex interleaving.

### **Q:** What does compile-time reflection look like, and is it shared with modules?

<!-- [Q-reflect]: #q-what-does-compile-time-reflection-look-like-and-is-it-shared-with-modules -->

**Status:** Open

Derive-style macros need a way to ask a struct "what are your fields and their
types?" The same information a module's public interface records.

Options: (a) design reflection as its own compile-time API, independent of the
module interface; (b) treat the two as one body of data with two readers — the
module interface serializes it, reflection exposes it to compile-time code.
(b) avoids duplicating the same information in two formats but couples the two
designs. This interacts directly with the module proposal.

### **Q:** What form of syntax value do macros receive and return?

<!-- [Q-syntax]: #q-what-form-of-syntax-value-do-macros-receive-and-return -->

**Status:** Open

A macro operates on Syntax, but the exact representation handed to macro code
is undecided.

Options: (a) the raw `AstNode` tree directly; (b) a dedicated syntax-object type
that wraps AST nodes together with their scope/origin information (needed for
hygiene) and presents a stable surface to macro authors. (a) is simplest and
reuses what exists; (b) is more work but decouples macro-facing code from the
compiler's internal AST shape and is the natural home for the per-syntax scope
information hygiene requires.

## Glossary

These are both technical and non technical terms used throughout the proposal.

- **Changes**: Changes refer the proposed changes in this document
- **Macro**: Code that runs at compile time and produces more code. A macro
  takes a piece of syntax and returns a piece of syntax.
- **Syntax**: Pieces of the program's own structure (AST nodes) treated as data
  that a macro can read and build.
- **AST**: The tree-shaped data structure the parser produces from source text.
  In Kirby this is the `AstNode` type in `src/ast.h`.
- **Hygiene**: The property that names a macro introduces cannot accidentally
  clash with names in the code that used the macro, and vice versa.
- **The Pipeline**: The ordered stages a program passes through: parse, expand
  macros, type check, monomorphize, compile.
- **Reflection**: Compile-time access to a concrete type's shape — for a struct,
  its fields and their types.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions

<!-- Proposals -->

[Debugger Proposal]: ../debugger/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md

<!-- Questions -->

[Q-resolve]: #q-will-macros-ever-need-resolved-names-during-expansion
[Q-reflect]: #q-what-does-compile-time-reflection-look-like-and-is-it-shared-with-modules
[Q-syntax]: #q-what-form-of-syntax-value-do-macros-receive-and-return
[The Pipeline]: #part-5--where-expansion-sits-in-the-pipeline
