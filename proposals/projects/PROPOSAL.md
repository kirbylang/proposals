---
status: Draft
created: 2026-09-18
from_commit: b56a2a9
---

# Proposal: Projects

This proposal adds a project structure & [conventions], a project manager CLI. Out of scope: dependency management, and requiring a main function to make Kirby files runnable/executable, which is the [Top-Level Declarations Proposal].

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing process to understand the changes being proposed and what impacts they will have. That research is largely tracked in the form of [Questions].

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it' is defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Kirby has no concept of a project and without [Modules], Kirby is limited to executing single files.

## Conventions

- Root project folder [kirby.toml]
- Source folder: `src/**/*.krb`

## Questions

### **Q:** How is the project root found?

<!-- [Q-root]: #q-how-is-the-project-root-found -->

**Status:** Open

The [Tooling Data Proposal] plans for the source path recorded in compiled code
to be relative to the project root in the long run ([Q-paths]), so that it reads
the same on every machine. That needs answers this proposal does not give yet:

- **Finding the root.** Given a single file, how does `krb` know where the root
  is: an option, or looking for `kirby.toml` in the file's folder and the
  folders above it?
- **A file outside a project.** `krb -f file.krb` with no project around has no
  root. What path is recorded then?
- **What the path looks like.** Does it include the source folder
  (`src/main.krb` or `main.krb`)? Is it written with `/` on every system?
- **Files or names.** Whether a module is a file or a name is [Q-naming] in the
  [Modules] proposal, and decides how much the path matters.

## Glossary

These are both technical and non technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer the proposed changes in this document
- **Project**: A folder containing a root project file, one or more Kirby source files

## Link References

<!-- Link references are preferred for all types of links -->

[kirby.toml]: ./kirby.toml

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Conventions]: #conventions

<!-- Questions -->

[Q-root]: #q-how-is-the-project-root-found

<!-- Proposals -->

[Modules]: ../modules/PROPOSAL.md
[Tooling Data Proposal]: ../tooling-support-data/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md

<!-- Other proposals' questions -->

[Q-paths]: ../tooling-support-data/PROPOSAL.md#q-what-form-does-the-recorded-source-path-take
[Q-naming]: ../modules/PROPOSAL.md#q-is-a-module-named-by-its-file-path-or-by-a-namespace
