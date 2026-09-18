---
status: Draft
created: 2026-09-D18
from_commit: b56a2a9
---

# Proposal: Projects

This proposal adds a project structure & [conventions], a project manager CLI. Out of scope: dependency management, requiring a main function to make Kirby files runnable/executable (which requires restricting what top level items can be declared).

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

- Root project folder e.g. `kirby.toml|kdl`

## Questions

### **Q:** <!-- A very concise wording of the question -->

<!-- [Q#]: #id-of-this-question -->

**Status:** <!-- Open | Answered -->

<!-- A breakdown of the question -->

#### Answer

<!-- What answer or conclusion to the question. This section only appears after the question is answered.  -->

## Glossary

These are both technical and non technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer the proposed changes in this document
- **Project**: A folder containing a root project file, one or more Kirby source files
- **<!--Term-->**: <!-- Concise definition of the term. -->
- **<!--Term-->**: <!-- Concise definition of the term... -->

## Link References

<!-- Link references are preferred for all types of links -->

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Conventions]: #conventions

<!-- Proposals -->

[Modules]: ../modules/PROPOSAL.md
