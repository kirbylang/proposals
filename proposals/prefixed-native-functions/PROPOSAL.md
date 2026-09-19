---
status: Closed
created: 2026-09-18
from_commit: b56a2a9
---

# Proposal: Prefix Native Function Names

This proposal documents an idea rather than a design: mark every native
function's name with a leading character, so `len(x)` is written `@len(x)`
(or whatever character [Questions] settles on). It stays deliberately
shallow — the point right now is the reasoning and the open questions, not
a lexer/parser implementation.

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

- When the proposal text says Kirby "does" or "has" something, that is true
  for the current implementation.
- When the proposal text says Kirby "should" or "will" do something, that is
  true after changes presented in this proposal.

### Code & Changes

- Any C code from the language's implementation will be displayed in `c`
  code blocks.
- Any Kirby code will be displayed in `kirby` code blocks.
- Any code changes (C or Kirby) will be displayed as `diff` blocks.

## Problem Statement

A native function's name is just another entry in the same global table
user code writes to. `defineNative` (`src/native.c`) puts each of the
~40 names in `nativeDefinitions[]` straight into `vm->globals`:

```c
void defineNative(VM *vm, const char *name, NativeFn function) {
  pushOnStack(OBJ_VAL(copyString(vm->gc, name, (int)strlen(name))));
  pushOnStack(OBJ_VAL(newNative(vm->gc, function)));
  tableSet(vm->gc, &vm->globals, AS_STRING(vm->stack[0]), vm->stack[1]);
  popFromStack();
  popFromStack();
}
```

and `OP_DEFINE_GLOBAL` (`src/vm.c`) overwrites whatever is already sitting
at a given key with no check at all:

```c
case OP_DEFINE_GLOBAL: {
  ObjString *name = READ_STRING();
  Value value = peekStack(0);
  tableSet(vm.gc, &vm.globals, name, value);
  popFromStack();
  break;
}
```

So `fun len(list) { return 0; }` at the top of a script doesn't error — it
just replaces the native `len` in `vm->globals` for the rest of the run,
silently, in whichever direction the definitions happen to run. There is no
scanner, parser, or type-checker pass anywhere that treats the names in
`nativeDefinitions[]` as anything other than ordinary identifiers a user
happened not to pick this time.

Because nothing at the language level tells them apart, editor tooling
fakes it with hand-maintained lists instead. Native calls are only
highlighted differently from user calls because of one long, manually kept
alternation in the syntax file:

```json
"match": "\\b(clock|len|argv|argc|typeof|rand|rand01|randBetween|ceil|readFileToString|writeStringToFile|fileExists|getenv|setenv|__version__|exit|instanceOf|prompt|stdin|parseNumber|numberToString|arrPush|arrPop|arrInsert|arrRemove|arrClear|arrContains|arrCopy|arrEqual|arrIsEmpty|arrSlice|arrConcat|arrReverse|arrJoin)\\b"
```

and hover text comes from a second hand-maintained list, one file per
native under `vsc/hovers/`, keyed by the bare name (`vsc/hovers/len.md`,
`vsc/hovers/arrPush.md`, and so on). AGENTS.md's own checklist for adding a
native function spells out editing both of these by hand, alongside the
signature table, every single time — and nothing enforces that any of them
stay in sync. Forgetting the regex update just means the new native
quietly highlights like a plain identifier.

## Proposed Changes

Every native function name would carry a fixed prefix character at every
call site: `len(x)` becomes `@len(x)`, `setenv("K", "V")` becomes
`@setenv("K", "V")`, and so on for all ~40 entries in `nativeDefinitions[]`,
plus anything added later (e.g. by the [Additional Native Functions
Proposal]).

`@` is the current leading candidate — see [Q-prefix-char] for why, and for
the other characters considered.

That's the entire design for now, on purpose. How the scanner would
tokenize the prefix, how the parser would then recognize a native
reference, and a migration path for existing call sites are all left as
follow-up work once the prefix question itself is settled.

The prefix character would not be valid in user code defined identifiers so there is no possibility of overwriting native functions.

## Related Proposals

- [Additional Native Functions Proposal] — adds roughly 18 more native
  functions. If this proposal is accepted, those new names would need the
  same prefix; the two should agree on the final list before either lands.
- [Collection Methods Proposal] — proposes turning array natives like
  `arrPush`/`arrContains` into methods (`arr.push(x)`) instead of global
  functions. Any native that proposal moves to method syntax stops being a
  user-facing global and drops out of this proposal's scope.
- [Modules Proposal] — still a skeleton, but [Q-modules] below touches the
  same undecided ground as that proposal's own naming and assembly
  questions.

## Outcome

This was implemented and delivered in commit `acf2f6`.

## Questions

### **Q:** What should the prefix character be?

**Status:** Answered

#### Answers

`@` is the prefix.

### **Q:** Does this change with modules?

**Status:** Answered

Every native lives in one flat, always-present table (`vm->globals`)
today, which is what makes it global in the first place. The [Modules
Proposal] doesn't yet say whether natives would keep living there once
modules exist, or move into some built-in module's namespace instead
(`std.len(x)`-style), reached the same way any other module's contents
would be.

If natives moved into a module namespace, that namespace already solves
the problem this prefix is meant to solve — telling `len` apart from a
same-named function of the user's — through `std.` rather than punctuation.
The two would then either need to compose (`std.@len`) or one would make
the other redundant. The [Modules Proposal] has no naming design yet
of its own ([Q-assembly] there), so this can't really be answered until
that one is further along; whichever answer comes first, the two documents
should stay in agreement.

#### Answer

Deferring this. It will need to be addressed on the modules proposal.

### **Q:** Whether documentation stays keyed by the bare name.

**Status:** Answered

`vsc/hovers/len.md` is filename-keyed to the bare name today; deciding
now whether that key becomes `@len.md` avoids a second rename later.

#### Answer

If it works, `@len.md` is ideal for simplity.

## Glossary

These are both technical and non technical terms used throughout the
proposal.

- **Changes**: Changes refer to the proposed changes in this document
- **Native function**: A function implemented in C and exposed to Kirby
  code under a global name, as opposed to one written in Kirby itself.
  Defined in `src/native.c`.
- **Global table**: `vm->globals` (`src/vm.c`), the single hash table
  backing every top-level variable and function in a running program,
  native or user-defined alike.

## Link References

<!-- Sections -->

[Links]: #link-references
[Glossary]: #glossary
[Questions]: #questions
[Related Proposals]: #related-proposals

<!-- Questions -->

[Q-prefix-char]: #q-what-should-the-prefix-character-be
[Q-modules]: #q-does-this-change-with-modules
[Q-other]: #q-what-else-should-this-account-for

<!-- Related proposals -->

[Additional Native Functions Proposal]: ../additional-native-functions/PROPOSAL.md
[Collection Methods Proposal]: ../collection-methods/PROPOSAL.md
[Modules Proposal]: ../modules/PROPOSAL.md
