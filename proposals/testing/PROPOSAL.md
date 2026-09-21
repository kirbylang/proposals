---
status: Draft
created: 2026-09-18
from_commit: b09db2e
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

## Related Proposals

- [Debugger Proposal] — debugging a single test would be a natural use of the
  debugger. The debugger's first version only supports `krb -f`, so supporting
  `krb test` is left to whichever of the two proposals lands second.
- [Top-Level Declarations Proposal] — a call at the top level becomes an error, so
  `@test(description, closure)` can no longer register a test by being called
  there ([Q-register]). It also removes the reason test code cannot sit beside
  the code it tests; the answer to the colocating question below is updated.
- [Embedded Library Proposal] — `krb test` could run every test in one process through
  the interface that proposal adds: load a file, call each test by name, and
  collect the failures without a test being able to end the process
  ([Embedded Lbrary Part 3]). Option (c) of [Q-register], finding tests by name, needs
  a way to list the functions of a loaded file, which would be a small addition
  to its [Embedded Lbrary Part 5]. That proposal is updated to say so.

## Questions

### Q: Colocating test code with impplementation code

Can test code be code located with implementation code?

#### Answer

Not for this proposal. Kirby currently allows top level scripting behavior, specifically things that cause side effects. So when you try to run the file with the test code enabled, the side effects happen just interpreting the file, which is what registers the tests.

This is the same issue with importing [modules]. If modules existed in the language now, importing would cause side effects.

If the [Top-Level Declarations Proposal] is accepted, loading a file no longer has side effects, and this answer changes: test code can sit beside the code it tests, once registering a test is not a top-level call ([Q-register]).

### **Q:** How are tests registered when the top level cannot make calls?

<!-- [Q-register]: #q-how-are-tests-registered-when-the-top-level-cannot-make-calls -->

**Status:** Open

`@test(description, closure)` registers a test when it is called, which would be at the top level of a `*.test.krb` file. The [Top-Level Declarations Proposal] makes a call at the top level an error. Options:

- **(a) The file's `main` registers the tests.** `fun main(): unit { @test("adds", fun () { ... }); }`, and `krb test` runs each matching file the way `krb -f` does. Nothing new in the language, but a test file cannot share a file with code that has its own `main`.
- **(b) A test is a declaration**, such as `test "adds" { ... }`, and `krb test` finds tests among the declarations of a loaded file. It needs new syntax, but tests can sit next to the code they test, and loading the file still does nothing.
- **(c) Tests are functions found by name**, for example `fun test_adds(): unit`. No new syntax, but a naming rule that is easy to get wrong. Finding them needs a way to list a file's functions, which the [Embedded Library Proposal] could provide.

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

[Q-register]: #q-how-are-tests-registered-when-the-top-level-cannot-make-calls

<!-- Related proposals -->

[Debugger Proposal]: ../debugger/PROPOSAL.md
[Top-Level Declarations Proposal]: ../top-level-declarations/PROPOSAL.md
[Embedded Library Proposal]: ../embedded-library/PROPOSAL.md
[Embedded Lbrary Part 3]: ../embedded-library/PROPOSAL.md#part-3-a-scripts-mistake-stops-the-script
[Embedded Lbrary Part 5]: ../embedded-library/PROPOSAL.md#part-5-call-kirby-from-the-host-and-get-values-back

<!-- Example: [Proposal Name]: path/to/PROPOSAL.md -->
