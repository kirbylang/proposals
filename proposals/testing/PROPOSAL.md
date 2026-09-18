# ---

status: Draft
created: 2026-09-18
from_commit: `b09db2e`

---

# Proposal: Testing

This proposal aims to outline an initial test framework for Kirby code.

---

## How to read this document

### Living Document

This proposal is a living document while in Draft status. It's an ongoing process to understand the changes being proposed and what impacts they will have. That research is largely tracked in the form of [Questions].

### Linking

[Links] in this document are defined as link references.

### Terminology

Technical terms are kept to a minimum. Where a term is unavoidable, it is defined in the [Glossary] below.

### Existing vs Proposed Behaviors

- When the proposal text says Kirby "does" or "has" something, that is true for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c` code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

Kirby doesn't have a way to write and run tests that test Kirby code.

## Proposed Changes

- New subcommand: `krb test $pattern`
  - Default pattern: `**/*.test.krb`
- New native functions
  - `@test(description, closure)`
  - `@assert(bool, string)`
  - `@panic(string)`

## Related Proposals

## Questions

### Q: Colocating test code with impplementation code

Can test code be code located with implementation code?

#### Answer

Not for this proposal. Kirby currently allows top level scripting behavior, specifically things that cause side effects. So when you try to run the file with the test code enabled, the side effects happen just interpreting the file, which is what registers the tests.

This is the same issue with importing [modules]. If modules existed in the language now, importing would cause side effects.

## Glossary

These are both technical and non-technical terms used throughout the proposal.

<!-- The glossary should be towards the bottom of the document -->

- **Changes**: Changes refer to the proposed changes in this document

## Link References

<!-- Link references are preferred for all types of links -->

[Modules]: ../modules/PROPOSAL.md

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Questions -->

<!-- Related proposals -->

<!-- Example: [Proposal Name]: path/to/PROPOSAL.md -->
