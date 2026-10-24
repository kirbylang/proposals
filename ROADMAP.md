# Roadmap

This is a high level view of the proposals and how they relate to each other.

## Epics

| Stage | Proposal                                                                                   | Needs                                                             | Status      | Notes                                                                                                            |
| ----- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------- |
| 1     | [Top-Level Declarations]                                                                   |                                                                   | Ready       | Files hold declarations; `main` is called                                                                        |
| 1     | [Tooling Data]: [Part 1: the span][td-1], [Part 2: a span on every node][td-2]             |                                                                   | Ready       |                                                                                                                  |
| 1     | [Tooling Data]: [Part 4: instruction origins][td-4] to [Part 7: what reaches the VM][td-7] |                                                                   | Ready       | Source names, lines, local variable names                                                                        |
| 1     | [Generic Types]: [Part 3: type parameters][gt-3]                                           |                                                                   | Ready       |                                                                                                                  |
| 1     | [Projects]                                                                                 |                                                                   | Ready       | Design with modules ([module naming][Q-naming]). Multi-file projects wait for modules                            |
| 2     | [Generic Types]: [Part 4: trait bounds][gt-4]                                              | generic-types: type parameters                                    | Not started |                                                                                                                  |
| 2     | [Macros], with [Tooling Data]: [Part 3: spans that survive generation][td-3]               | tooling-data: spans                                               | Not started | Built together: macros need origins on spans. Design with top-level-declarations ([comptime values][Q-comptime]) |
| 3     | [Generic Types]: [Part 5: monomorphization][gt-5]                                          | generic-types: type parameters, trait bounds; tooling-data: spans | Not started | Each specialized copy keeps its source spans                                                                     |
| 4     | [Modules]                                                                                  | generic-types: monomorphization; top-level-declarations           | Not started | Must ship generic bodies ([body form][Q-bodyform])                                                               |

## Features

| Stage | Proposal                                                                  | Needs                                                                                | Status      | Notes                                                     |
| ----- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ----------- | --------------------------------------------------------- |
| 1     | [Tuples]                                                                  |                                                                                      | Ready       |                                                           |
| 1     | [Enums]: [Part A: plain variants][en-a]                                   |                                                                                      | Ready       |                                                           |
| 1     | [Destructuring]: [struct patterns][de-struct], [array patterns][de-array] |                                                                                      | Ready       |                                                           |
| 1     | [For-Each]: [Part A: arrays][fe-a], [Part B: ranges][fe-b]                |                                                                                      | Ready       |                                                           |
| 1     | [String Encoding]                                                         |                                                                                      | Ready       |                                                           |
| 1     | [Primitive Impls]                                                         |                                                                                      | Ready       | [Order against generics][Q-sequencing] is open            |
| 1     | [Trait Types]                                                             |                                                                                      | Ready       |                                                           |
| 1     | [Unimplemented]                                                           |                                                                                      | Ready       |                                                           |
| 2     | [Tuple Structs]                                                           | tuples                                                                               | Not started |                                                           |
| 2     | [Pattern Matching]                                                        | destructuring: struct and array patterns; enums: plain variants                      | Not started | Tuple patterns in `match` follow destructuring's          |
| 2     | [Strings]                                                                 | string-encoding                                                                      | Not started |                                                           |
| 2     | [Collection Methods]                                                      | primitive-impls                                                                      | Not started | `Map`/`Set` methods [wait for generics][Q-generics-order] |
| 3     | [Destructuring]: [tuple and tuple struct patterns][de-tuple]              | tuples, tuple-structs                                                                | Not started |                                                           |
| 3     | [Enums]: [Part B: variants that carry values][en-b]                       | pattern-matching                                                                     | Not started | Unusable without `match`                                  |
| 4     | [Enums]: [Part C: type parameters][en-c]                                  | enums: variants that carry values; **generic-types: type parameters**                | Not started | `Option[T]`, `Result[T, E]`                               |
| 5     | [Result]                                                                  | enums: type parameters; pattern-matching                                             | Not started |                                                           |
| 5     | [Map]                                                                     | enums: type parameters                                                               | Not started | `@mapGet` returns `Option[V]`                             |
| 5     | [For-Each]: [Part C: one rule for everything else][fe-c]                  | for-each: arrays, ranges; enums: type parameters; **generic-types: type parameters** | Not started | `Iterator[T]`                                             |

**Bold** marks an epic.

## Tooling

| Proposal                                                                                                      | Needs                                                        | Status      | Notes                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | ----------- | --------------------------------------------------------------------------------------------------------------------- |
| [Embedded Library]: [Part 9: compiled scripts][el-9]                                                          |                                                              | In progress | The `KRBC` format and `krb build` are built (kirbylang #118). `.krbc` files, `krb compile -o`, `krbLoadBytes` are not |
| [Embedded Library]: [Part 1: runtime without the compiler][el-1] to [Part 8: public header and library][el-8] |                                                              | Ready       |                                                                                                                       |
| [Lua Transpiler]                                                                                              |                                                              | Ready       | Easier after top-level-declarations                                                                                   |
| [Debugger]                                                                                                    | **tooling-data: instruction origins to what reaches the VM** | Not started | Easier after top-level-declarations                                                                                   |
| [Testing]                                                                                                     | **top-level-declarations**                                   | Not started | Easier after embedded-library                                                                                         |

## Experimental

None.

## Progress

| Proposal                 | Category | Status      |
| ------------------------ | -------- | ----------- |
| [Generic Types]          | epic     | Not started |
| [Macros]                 | epic     | Not started |
| [Modules]                | epic     | Not started |
| [Projects]               | epic     | Not started |
| [Tooling Data]           | epic     | Not started |
| [Top-Level Declarations] | epic     | Not started |
| [Collection Methods]     | feature  | Not started |
| [Destructuring]          | feature  | Not started |
| [Enums]                  | feature  | Not started |
| [For-Each]               | feature  | Not started |
| [Map]                    | feature  | Not started |
| [Pattern Matching]       | feature  | Not started |
| [Primitive Impls]        | feature  | Not started |
| [Result]                 | feature  | Not started |
| [String Encoding]        | feature  | Not started |
| [Strings]                | feature  | Not started |
| [Trait Types]            | feature  | Not started |
| [Tuple Structs]          | feature  | Not started |
| [Tuples]                 | feature  | Not started |
| [Unimplemented]          | feature  | Not started |
| [Debugger]               | tooling  | Not started |
| [Embedded Library]       | tooling  | In progress |
| [Lua Transpiler]         | tooling  | Not started |
| [Testing]                | tooling  | Not started |

## Critical paths

- **Epics:** generic-types: type parameters → trait bounds → monomorphization
  → modules. Four stages.
- **Features:** destructuring: struct and array patterns → pattern-matching →
  enums: variants that carry values → enums: type parameters → result, map,
  for-each: everything else. Five stages. Joins the epics at generic-types:
  type parameters. Everything that uses `Option` or `Result` waits on it.

## Unsettled order

Answering these can move a row between stages.

- [Collection Methods: wait for generics?][Q-generics-order]
- [Collection Methods: lists alone, or after a `Map`/`Set` sketch?][Q-scope-order]
- [Primitive Impls: before, after, or independent of generics?][Q-sequencing]
  It recommends independent.
- [Generic Types 10.3: operators for primitives][gt-10.3]: if every operator
  becomes a trait method, generic-types needs primitive-impls first.
- [Top-Level Declarations: when samples drop top-level statements][Q-samples]
- Debugger and testing: `krb test` support goes in whichever lands second.

## Closed

Closed before this roadmap existed: [Additional Native Functions],
[Prefixed Native Functions], [String Interpolation], [Sized Number Types],
[Lists].

## Categories

Each proposal has a `category` in its frontmatter.

| Category       | Meaning                                              | May need                  | Needed by                       |
| -------------- | ---------------------------------------------------- | ------------------------- | ------------------------------- |
| `epic`         | Must do. Large changes to the language or toolchain. | epics                     | anything                        |
| `feature`      | Nice to have. Syntax and built-in types.             | epics, features           | features, tooling, experimental |
| `tooling`      | Standalone tools around the language.                | anything but experimental | nothing                         |
| `experimental` | Exploring an idea. Its effects stay inside it.       | anything but experimental | nothing                         |

A proposal that something else needs can't be `experimental`. If one becomes
needed, it moves to another category first.

## Status

Rows in the stage tables:

| Status      | Meaning                          |
| ----------- | -------------------------------- |
| Not started | Something it needs isn't Done    |
| Ready       | Everything it needs is Done      |
| In progress | Being built                      |
| Done        | Built. Links to the PR or commit |

Proposals, in [Progress]:

| Status      | Meaning                                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------------------------- |
| Not started | No row is In progress or Done                                                                                          |
| In progress | A row is In progress, none is Done                                                                                     |
| Partial     | Some rows are Done, not all                                                                                            |
| Done        | The proposal is `Closed`. This can come long after its last row, since proposals keep changing as other proposals need |

Rows are never removed. A Done row stays where it is.

Work that a later proposal causes in a Done row belongs to the later
proposal. For example, if generic types is Done before tuples, teaching the
[substitution helper][gt-3.6] about tuple types is part of the tuples
proposal, and generic types doesn't reopen.

## Link References

<!-- Sections -->

[Progress]: #progress

<!-- Proposals -->

[Additional Native Functions]: ./proposals/additional-native-functions/PROPOSAL.md
[Collection Methods]: ./proposals/collection-methods/PROPOSAL.md
[Debugger]: ./proposals/debugger/PROPOSAL.md
[Destructuring]: ./proposals/destructuring/PROPOSAL.md
[Embedded Library]: ./proposals/embedded-library/PROPOSAL.md
[Enums]: ./proposals/enums/PROPOSAL.md
[For-Each]: ./proposals/for-each/PROPOSAL.md
[Generic Types]: ./proposals/generic-types/PROPOSAL.md
[Lists]: ./proposals/lists/PROPOSAL.md
[Lua Transpiler]: ./proposals/lua-transpiler/PROPOSAL.md
[Macros]: ./proposals/macros/PROPOSAL.md
[Map]: ./proposals/map/PROPOSAL.md
[Modules]: ./proposals/modules/PROPOSAL.md
[Pattern Matching]: ./proposals/pattern-matching/PROPOSAL.md
[Prefixed Native Functions]: ./proposals/prefixed-native-functions/PROPOSAL.md
[Primitive Impls]: ./proposals/primitive-impls/PROPOSAL.md
[Projects]: ./proposals/projects/PROPOSAL.md
[Result]: ./proposals/result/PROPOSAL.md
[Sized Number Types]: ./proposals/sized-number-types/PROPOSAL.md
[String Encoding]: ./proposals/string-encoding/PROPOSAL.md
[String Interpolation]: ./proposals/string-interpolation/PROPOSAL.md
[Strings]: ./proposals/strings/PROPOSAL.md
[Testing]: ./proposals/testing/PROPOSAL.md
[Tooling Data]: ./proposals/tooling-support-data/PROPOSAL.md
[Top-Level Declarations]: ./proposals/top-level-declarations/PROPOSAL.md
[Trait Types]: ./proposals/trait-types/PROPOSAL.md
[Tuple Structs]: ./proposals/tuple-structs/PROPOSAL.md
[Tuples]: ./proposals/tuples/PROPOSAL.md
[Unimplemented]: ./proposals/unimplemented/PROPOSAL.md

<!-- Parts of proposals -->

[gt-3]: ./proposals/generic-types/PROPOSAL.md#part-3--generic-type-parameters
[gt-3.6]: ./proposals/generic-types/PROPOSAL.md#36-a-single-substitution-helper-used-everywhere
[gt-4]: ./proposals/generic-types/PROPOSAL.md#part-4--trait-bounds
[gt-5]: ./proposals/generic-types/PROPOSAL.md#part-5--monomorphization
[gt-10.3]: ./proposals/generic-types/PROPOSAL.md#103-operators-for-primitives-built-in-fast-path-or-uniform-trait-dispatch
[td-1]: ./proposals/tooling-support-data/PROPOSAL.md#part-1--the-span
[td-2]: ./proposals/tooling-support-data/PROPOSAL.md#part-2--a-span-on-every-ast-node
[td-3]: ./proposals/tooling-support-data/PROPOSAL.md#part-3--spans-that-survive-generation
[td-4]: ./proposals/tooling-support-data/PROPOSAL.md#part-4--where-an-instruction-came-from
[td-7]: ./proposals/tooling-support-data/PROPOSAL.md#part-7--what-reaches-the-running-vm
[en-a]: ./proposals/enums/PROPOSAL.md#part-a-enums-with-plain-variants
[en-b]: ./proposals/enums/PROPOSAL.md#part-b-variants-that-carry-values
[en-c]: ./proposals/enums/PROPOSAL.md#part-c-enums-with-type-parameters
[fe-a]: ./proposals/for-each/PROPOSAL.md#part-a-arrays
[fe-b]: ./proposals/for-each/PROPOSAL.md#part-b-ranges
[fe-c]: ./proposals/for-each/PROPOSAL.md#part-c-one-rule-for-everything-else
[de-struct]: ./proposals/destructuring/PROPOSAL.md#struct-patterns
[de-array]: ./proposals/destructuring/PROPOSAL.md#array-patterns
[de-tuple]: ./proposals/destructuring/PROPOSAL.md#tuple-and-tuple-struct-patterns
[el-1]: ./proposals/embedded-library/PROPOSAL.md#part-1-the-runtime-links-without-the-compiler
[el-8]: ./proposals/embedded-library/PROPOSAL.md#part-8-a-public-header-and-a-library-that-is-easy-to-link
[el-9]: ./proposals/embedded-library/PROPOSAL.md#part-9-compiled-scripts-and-a-runtime-only-build

<!-- Questions in proposals -->

[Q-generics-order]: ./proposals/collection-methods/PROPOSAL.md#q-does-this-wait-on-generics-or-ship-an-untyped-version-first
[Q-scope-order]: ./proposals/collection-methods/PROPOSAL.md#q-should-this-start-from-arrays-alone-or-wait-until-tuplessetmap-are-at-least-sketched
[Q-sequencing]: ./proposals/primitive-impls/PROPOSAL.md#q-should-this-land-before-after-or-independent-of-the-generic-types-proposals-operator-dispatch-decision
[Q-bodyform]: ./proposals/modules/PROPOSAL.md#q-in-what-form-does-a-module-ship-its-generic-bodies
[Q-naming]: ./proposals/modules/PROPOSAL.md#q-is-a-module-named-by-its-file-path-or-by-a-namespace
[Q-samples]: ./proposals/top-level-declarations/PROPOSAL.md#q-when-do-code-samples-in-other-proposals-change
[Q-comptime]: ./proposals/top-level-declarations/PROPOSAL.md#q-how-much-is-a-comptime-value
