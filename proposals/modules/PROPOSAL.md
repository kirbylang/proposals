---
status: Draft
created: 2026-09-18
from_commit: bc3ae67
---

# Proposal: Modules — Distributable Units of Compiled Kirby

This is a **skeleton** proposal. It states the problem modules solve, sketches
the pieces a solution needs, and records the open questions that must be
answered before it can become a full design. It is deliberately light on
committed decisions: most of the substance lives in the [Questions] section.

A Module is a unit of Kirby code that can be compiled on its own and used by
other code that may not have its source. Distributing modules as compiled
artifacts — code compiled once and consumed later, possibly on a different
machine, without the original source — is the target this proposal works toward.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing
process to understand the changes being proposed and what impacts they will
have. That research is largely tracked in the form of [Questions]. This proposal
in particular is early: expect most of it to be questions rather than answers.

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

Today Kirby's type checker always has the whole program's source at once. The
checker runs as a series of whole-program passes over the full array of
top-level nodes (`typchkCheckProgram`, `src/typecheck.c`), and it sees every
declaration's source every time.

A module breaks that assumption. Code that uses a module may be compiled long
after the module, on a different machine, with only the module's compiled form
on hand. Two capabilities then need information the current compiled form does
not carry:

- **Type checking against the module.** To check `box.value` where `box` came
  from another module, the consumer's checker needs that module's struct shapes,
  trait definitions, and function signatures.
- **Monomorphizing the module's generics.** To instantiate the module's
  `Box[T]` with the consumer's own type, the consumer's compiler needs the
  _body_ of the generic code, not just its signature. (Monomorphization is
  described in the type-system proposal.)

What Kirby can serialize today carries neither. The `CompiledUnit`
(`src/compiled_unit.h`) stores compiled functions, their bytecode chunks,
constants, upvalue records, and a string table — and **no type information at
all**. There is nothing a separate compiler or tool could read to understand a
compiled unit's interface.

This proposal exists to close that gap, but most of _how_ is still open.

### Relationship to other proposals

- **Type system.** Cross-module generics are the hardest part of modules, and
  they depend entirely on how monomorphization works. A module must ship the
  un-specialized bodies of its generic functions and structs so a consumer can
  specialize them. That requirement is the main new thing modules add on top of
  the type-system proposal.
- **Tooling data.** Spans must identify which file they point into; modules and
  multi-file programs are what make that necessary. How files are named and
  identified is shared with the [Tooling Data Proposal] ([Q-files] there), and
  turns on whether a module is a file or a name ([Q-naming]).
- **Macros.** Compile-time reflection over a struct's shape reads the same
  information a module's public interface records. Whether they share one body
  of data is an open question in both proposals.
- **Debugger and other tools.** A debugger needs source paths and local variable
  names, and those live in the compiled unit. A module shipped without its
  source raises two questions: does its compiled form carry that information at
  all, and what does a source path recorded on one machine mean on another? See
  [Q-paths] and [Q-strip] in the [Tooling Data Proposal]. The [Debugger
  Proposal] is the first tool to read it. The direction chosen for [Q-paths] is
  a path relative to a project root, which does not depend on the machine.
- **Top-level declarations.** The [Top-Level Declarations Proposal] limits a
  file to declarations, with comptime values for its `let` and `var`. That is
  what makes importing safe: loading a file has no side effects, and a
  module's compiled form does nothing observable when it loads. It also means
  only the entry file's `main` runs. The rules apply to each file, so they hold
  whether a module is a file or a namespace ([Q-naming]). It adds no import
  syntax and no namespaces; those stay in [Q-assembly].
- **embedded library.** The [Embedded Library Proposal] proposes a versioned file for a
  compiled unit: a magic number, a format version and the Kirby version, then the
  unit. The interface of [Q-interface] can extend that container instead of
  starting one ([Embedded Lbrary Part 9]). Whether a host function belongs to a module,
  so that `engine.spawn` replaces `@spawn`, is [Q-host-names] there, and waits
  for [Q-assembly] here.

## The Changes (Sketch)

This section is intentionally a sketch. Each piece names what is needed and
points to the question that must resolve it.

### Part 1 — The module interface

Alongside its compiled bytecode, a module should ship a second artifact: its
**interface**. Think of it as the module's published summary — like a C header
file, but richer. It would list, for everything the module makes public: struct
names and their fields; trait definitions; which structs implement which traits;
function and method signatures including generic parameters and bounds; and, for
every generic function and struct, the **un-specialized body** in a form the
consumer's compiler can specialize.

The first several items are the "header" — enough for a consumer's _checker_.
The last is the "recipe" — enough for a consumer's _compiler_ to monomorphize.
Both are needed; they serve different stages. Today's `CompiledUnit` carries no
type information at all, so this is entirely new. The exact contents and format
are [Q-interface] and [Q-bodyform].

### Part 2 — What "public" means

A module needs a boundary between what it exposes and what is internal. Kirby
already has a `pub` marker on struct fields and methods (`src/parser.c`). A
module's interface would contain exactly its public items and the types they
mention. Exactly which declarations carry `pub`, and what the defaults are, is
[Q-public].

### Part 3 — Coherence across modules

Within one program, a trait may be implemented for a struct only once, and this
is enforced today. Across modules a new hazard appears: two unrelated modules
could each implement the same trait for the same type, and a third module using
both would face a conflict with no principled way to choose. Which rule prevents
this — and how strict it should be — is [Q-coherence].

### Part 4 — Whole-program assembly

Even with separately compiled modules, the final program is assembled from a
module and everything it depends on. At that point a specializer has every
generic body it needs (from each module's interface) and can produce all
required specializations, yielding a single runnable program in which nothing
generic remains. How assembly and dependency resolution actually work is
[Q-assembly].

## Questions

### **Q:** What exactly does a module interface contain, and in what format?

<!-- [Q-interface]: #q-what-exactly-does-a-module-interface-contain-and-in-what-format -->

**Status:** Open

The interface must carry enough for both a consumer's checker (type shapes and
signatures) and a consumer's compiler (generic bodies).

Options and sub-questions: what serialization format (an extension of
`CompiledUnit`, a separate side-file, or both); how it is versioned so a
consumer can detect an incompatible producer; whether the interface is a
separate artifact from the bytecode or embedded alongside it. This is the
central open question of the whole proposal. The [Embedded Library Proposal] proposes the container for a bare
unit ([Embedded Lbrary Part 9]).

### **Q:** In what form does a module ship its generic bodies?

<!-- [Q-bodyform]: #q-in-what-form-does-a-module-ship-its-generic-bodies -->

**Status:** Open

Monomorphizing a module's generic against a new type needs the generic's body
to travel with the module.

Options: (a) ship the checked AST directly; (b) design a smaller, stable
intermediate representation derived from it. The AST is simplest and already
exists but is larger and ties the on-disk format to the AST's shape; a dedicated
intermediate form is more work but more stable across compiler versions. This
interacts with how `CompiledUnit` (which stores no types today) is extended, and
with the [Tooling Data Proposal] (generated/shipped syntax still needs origins).

### **Q:** What is the coherence rule across modules?

<!-- [Q-coherence]: #q-what-is-the-coherence-rule-across-modules -->

**Status:** Open

Two unrelated modules could implement the same trait for the same type,
producing a conflict for anyone using both.

Options:

- (a) An **ownership rule**: an `impl Trait for Type` is allowed only in the
  module that defines the trait or the module that defines the type. This is
  checkable by each module on its own, using only its own source and its
  dependencies' interfaces — no global view required. The standard workaround
  when you own neither side is the newtype pattern (wrap the foreign type in a
  thin struct you own). This turns a possible late failure into an immediate,
  local one.
- (b) **Whole-program detection**: allow any impl anywhere, detect conflicts
  only when a whole program is finally assembled. No ownership rule, but a
  conflict can lie hidden until two particular modules are combined, producing a
  late and confusing error.

Note: option (a) must not accidentally forbid what Kirby allows today — any
struct may currently implement any of the four built-in traits (`Display`, `Eq`,
`Ord`, `Default`). An ownership rule needs the built-in traits to remain
implementable by user code, so the rule's exact statement must account for
traits the language itself defines.

### **Q:** Which declarations carry `pub`, and what are the defaults?

<!-- [Q-public]: #q-which-declarations-carry-pub-and-what-are-the-defaults -->

**Status:** Open

Kirby has `pub` on fields and methods today; the module-level surface is not yet
designed.

Options: extend `pub` to whole structs / functions / traits with
private-by-default; or public-by-default with an explicit private marker; or
some mix. The choice affects what lands in the module interface and how much is
exposed by accident.

### **Q:** Is a module named by its file path or by a namespace?

<!-- [Q-naming]: #q-is-a-module-named-by-its-file-path-or-by-a-namespace -->

**Status:** Open

A program has to refer to a module somehow. Two ways:

- **(a) By file path.** A module is a file, and the program names it by where
  the file is.
- **(b) By namespace.** A module has a name, and the toolchain finds the files
  that make it up. A module could be more than one file, and a file could hold
  more than one module.

This is narrower than [Q-assembly], which covers importing, versions, and
dependencies. Several things in other proposals wait on it:

- **What a span points into.** A span must say which file it is in ([Q-files]).
  With (a), a file and a module are the same thing, and the list of files in a
  unit is a list of paths. With (b), a module can be several files, or none (as
  with `<repl>` and `<code>`), so a span needs a file, and the module's name is
  a separate thing.
- **What a recorded path means.** With (a), the path is also the module's
  identity, so how it is written ([Q-paths]) matters for more than tools. With
  (b), it is only where the source was.
- **How a project is laid out.** The [Projects Proposal] has a `src/**/*.krb`
  convention. It does not yet say whether the folders are part of a module's
  name.

### **Q:** How are modules named, resolved, and assembled?

<!-- [Q-assembly]: #q-how-are-modules-named-resolved-and-assembled -->

**Status:** Open

Nothing about how a program refers to a module, how a module's dependencies are
located, or how the final program is assembled has been decided.

Sub-questions: the syntax for importing/using a module; how a module's identity
and version are expressed (path or namespace is [Q-naming]); how the toolchain
finds a module's compiled artifact and interface; how a dependency graph is
resolved and in what order specialization runs across it. It also includes
whether a module's compiled artifact carries debug information (source paths,
local variable names) and how a recorded source path is meant to be read on
another machine; the [Tooling Data Proposal] raises this as [Q-paths] and
[Q-strip]. This is a large area on its own and may warrant its own proposal once
the interface format is settled.

### **Q:** How do modules interact with compile-time reflection?

<!-- [Q-reflect-shared]: #q-how-do-modules-interact-with-compile-time-reflection -->

**Status:** Open

A module's public interface records a struct's fields and types; compile-time
reflection (from the macro proposal) needs the same information.

Options: (a) one body of data with two readers — the interface serializes it,
reflection exposes it; (b) two separate mechanisms. (a) avoids duplication but
couples the module and macro designs. This question is shared with the macro
proposal.

## Glossary

These are both technical and non technical terms used throughout the proposal.

- **Changes**: Changes refer the proposed changes in this document
- **Module**: A unit of Kirby code that can be compiled on its own and used by
  other code, potentially without that other code having its source.
- **Module Interface**: A module's published summary of its public types,
  traits, and signatures, plus the un-specialized bodies of its generics —
  enough for a separate compilation to type-check and specialize against the
  module without its source.
- **CompiledUnit**: Kirby's current serialized form of a compiled program
  (`src/compiled_unit.h`): bytecode, constants, upvalues, and strings, with no
  type information.
- **Namespace**: A name that groups related code, such as `shapes`, used to
  refer to it instead of the path of the file it is in.
- **Monomorphization**: Compiling a generic by making a separate concrete copy
  for each distinct set of type arguments used. Defined fully in the type-system
  proposal.
- **Coherence**: The rule that a given trait may be implemented for a given type
  only once.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions

<!-- Proposals -->

[Debugger Proposal]: ../debugger/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Projects Proposal]: ../projects/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-files]: ../tooling-support-data/PROPOSAL.md#q-how-are-multiple-source-files-identified
[Q-paths]: ../tooling-support-data/PROPOSAL.md#q-what-form-does-the-recorded-source-path-take
[Q-strip]: ../tooling-support-data/PROPOSAL.md#q-is-tooling-data-always-recorded
[Q-host-names]: ../embedded-library/PROPOSAL.md#q-what-names-may-host-functions-have
[Embedded Lbrary Part 9]: ../embedded-library/PROPOSAL.md#part-9-compiled-scripts-and-a-runtime-only-build

<!-- Questions -->

[Q-interface]: #q-what-exactly-does-a-module-interface-contain-and-in-what-format
[Q-bodyform]: #q-in-what-form-does-a-module-ship-its-generic-bodies
[Q-naming]: #q-is-a-module-named-by-its-file-path-or-by-a-namespace
[Q-coherence]: #q-what-is-the-coherence-rule-across-modules
[Q-public]: #q-which-declarations-carry-pub-and-what-are-the-defaults
[Q-assembly]: #q-how-are-modules-named-resolved-and-assembled
[Q-reflect-shared]: #q-how-do-modules-interact-with-compile-time-reflection
