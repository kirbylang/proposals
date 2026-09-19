---
status: Draft
created: 2026-07-09
from_commit: 61b7cc5
---

# Proposal: Scala-style `???` (unimplemented placeholder)

Add a Scala-style `???` expression for marking declared-but-unimplemented
functions. It type-checks anywhere an expression is expected to have a
particular type, and panics with `NotImplemented` when executed, per
[kirbylang#48](https://github.com/kirbylang/kirbylang/issues/48):

```kirby
fun sum(a: f64, b: f64): f64 {
  ???
}

sum(1, 2); // Panics: NotImplemented
```

(The `def` in the issue is a typo for `fun`, Kirby's function keyword.)

## Problem Statement

Kirby has no idiom for "this function is declared but not written yet." The
only current workaround is `@panic`, and it has two problems:

1. **It doesn't produce a value.** `@panic` has the native signature
   `(string) -> unit` (`src/native.c`), so it cannot stand in for the
   declared return type. The body must still produce a value of the declared
   type on every path (`typchkCheckIfBodyProducesDeclaredValue` in
   `src/typecheck.c`), so the workaround requires a dead placeholder value:

   ```kirby
   fun sum(a: f64, b: f64): f64 {
     @panic("TODO: implement sum");
     0.0; // never reached, exists only to satisfy the checker
   }
   ```

2. **The message is ad hoc.** There is no standard name for the
   "not implemented yet" failure, so every stub invents its own string.

Scala's `???` (a placeholder expression of the bottom type `Nothing` that
throws `NotImplementedError`) and Rust's `todo!()` both solve this with a
single, recognizable token. This proposal adds Kirby's equivalent.

## Proposed Changes

### 1. Scanner: new token `TOKEN_QUESTION_QUESTION_QUESTION`

Today `?` is only valid as the two-character nullish-coalescing operator
`??` (`src/scanner.c`); anything else yields a `TOKEN_ERROR` token that the
parser rejects:

```c
case '?':
  return makeToken(scanner, match(scanner, '?') ? TOKEN_QUESTION_QUESTION
                                                : TOKEN_ERROR);
```

So `???` lexes today as `TOKEN_QUESTION_QUESTION` followed by
`TOKEN_ERROR` — a compile error in every program.

Extend with maximal munch:

```c
case '?':
  if (match(scanner, '?')) {
    if (match(scanner, '?'))
      return makeToken(scanner, TOKEN_QUESTION_QUESTION_QUESTION);
    return makeToken(scanner, TOKEN_QUESTION_QUESTION);
  }
  return makeToken(scanner, TOKEN_ERROR);
```

- New enum value `TOKEN_QUESTION_QUESTION_QUESTION` in `src/token.h`, next
  to `TOKEN_QUESTION_QUESTION`.
- `tokenTypeToString()` in `src/token.c` gains the new name; `unit/token.c`
  gains the corresponding assertion.

### 2. Parser: primary expression

Add a prefix-only rule to the parse table in `src/parser.c` (same shape as
the `literal` rules for `true`/`false`/`nil`):

```c
[TOKEN_QUESTION_QUESTION_QUESTION] = {unimplemented_, NULL, PREC_NONE},
```

```c
static AstNode *unimplemented_(Parser *parser, bool canAssign) {
  (void)canAssign;
  return astAlloc(NODE_UNIMPLEMENTED, parser->previous.line);
}
```

(`??` itself is unchanged: `[TOKEN_QUESTION_QUESTION] = {NULL, nullish_,
PREC_OR}`.)

`???` is an atom: it parses anywhere an expression is allowed and composes
with infix operators on its right (`??? + 1.0` parses; the type checker
rejects it, see §4). No postfix behavior is needed.

### 3. AST: `NODE_UNIMPLEMENTED`

- New `NodeKind` value in `src/ast.h` (before the `NODE_COUNT` sentinel).
  No payload — the node carries no data.
- `print_ast()` in `src/ast.c` prints `unimplemented`.
- The definite-assignment pass (`src/definite_assignment.c`) needs no change:
  its switches fall through to `default: break;` for unknown expression
  kinds, and `???` assigns nothing.

### 4. Type checker: check-position only

Kirby's checker is two-moded: `typchkInfer(env, node)` (no context) and
`typchkCheck(env, node, expected)` (with an expected type). `???` has no
type of its own; it type-checks **only where an expected type exists**. This
is the conservative version of Scala's `Nothing` — instead of adding a
bottom type to the type system, `???` is special-cased in the check entry
point:

`typchkCheck()` in `src/typecheck.c` (at the top, before the lambda/array
special cases):

```c
if (node->kind == NODE_UNIMPLEMENTED) {
  if (expected == NULL) {
    typchkErrorAtNode(node,
                      "'???' needs an expected type here. Add a type "
                      "annotation (e.g. a return type) so its type is "
                      "known.");
    return false;
  }
  return true;
}
```

`typchkInfer()` gains a matching case that reports the same error and
returns `NULL`.

**Positions that type-check** (an expected type is available):

- Body of a function or lambda with a declared or expected return type, via
  `typchkCheckBlockContents(env, block, expectedValueType)` — the issue's
  example: `fun sum(a: f64, b: f64): f64 { ??? }` ✓
- `return ???;` in a function with a return type ✓
- Annotated variable initializer: `var x: f64 = ???;` (then `x` is `f64`) ✓
- Assignment to an annotated variable: `x = ???;` ✓
- Trailing items of an array literal whose element type is already known
  (`[1.0, ???]`) ✓
- Lambda whose annotated function type supplies the return type:
  `var f: fun (f64) => f64 = fun (x: f64) { ??? };` ✓

**Positions that are compile errors** (no expected type):

- `var x = ???;` (unannotated initializer)
- `print ???;`
- `fun f() { ??? }` (function without a return type annotation)
- `??? + 1.0` (binary operands are inferred without context)
- `if/else` branches (branch types are inferred, not checked against an
  expectation)

The error message directs the user to the fix (add the annotation).

**"Must return on every path":** `typchkCheckIfBodyProducesDeclaredValue()`
already treats a block with a trailing value as returning on every path, so
`{ ??? }` satisfies the declared return type with no extra change.

**Statement position:** a bare `???;` should be accepted as a TODO marker
(see Open Question 3). Implementation: in `typchkCheckStmt()`'s
`NODE_EXPR_STMT` case, skip inference when the expression is
`NODE_UNIMPLEMENTED`.

### 5. Compilation: new opcode `OP_NOT_IMPLEMENTED`

- New opcode in `src/opcode.h` at index 42 (after
  `OP_CLOSE_BLOCK_EXPR`), no operands, no stack effect — execution never
  continues, so nothing is pushed and no `OP_POP` is needed.
- `compileExpr()` in `src/compiler.c`:

  ```c
  case NODE_UNIMPLEMENTED:
    emitByte(OP_NOT_IMPLEMENTED);
    break;
  ```

  When `???` is a block's value expression, `compileBlockContents()` already
  compiles it through `compileExpr()`; the function's (unreachable) implicit
  `OP_RETURN` still follows, so the bytecode stays well-formed for the
  disassembler.

### 6. VM: panic with `NotImplemented`

`src/vm.c`:

```c
case OP_NOT_IMPLEMENTED:
  runtimeError(&vm, "NotImplemented");
  exit(EXIT_CODE_RUNTIME_ERR);
```

This matches `@panic`'s behavior exactly (`raiseScriptMessage()` does the
same `runtimeError()` + `exit(EXIT_CODE_RUNTIME_ERR)`), so the issue's
example produces:

```
NotImplemented
[line 2] in sum()
[line 5] in script
```

with exit code `70` (`EXIT_CODE_RUNTIME_ERR`). Note that `@panic` kills the
process directly, even from the REPL, because the `exit()` happens inside
the native before `interpret()` can return. The alternative is the VM's
standard runtime-error convention (`runtimeError()` +
`return INTERPRET_RUNTIME_ERROR;`, as `OP_ADD` and friends use): file mode
still exits `70`, but the REPL would print `Runtime Error!` and stay alive.
This proposal uses the `@panic`-style exit; see Open Question 1.

### 7. Tooling

- **Disassembler** (`src/debug.c`): add `OP_NOT_IMPLEMENTED` to the
  instruction printer (no operands, like `OP_NIL`).
- **VS Code extension** (`vsc/syntaxes/kirby.tmLanguage.json`): the nullish
  pattern `\?\?` currently matches the first two characters of `???`. Add a
  longer pattern **before** it (TextMate patterns are ordered):

  ```json
  { "name": "keyword.unimplemented.krb", "match": "\\?\\?\\?" },
  { "name": "keyword.operator.nullish.krb", "match": "\\?\\?" }
  ```

### 8. Tests

**E2E** (file-based, per `tests/README.md`), new `tests/unimplemented/`
directory:

| Test                                 | Source                                        | Expected                                                                                                         |
| ------------------------------------ | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `unimplemented_function.krb`         | the issue's example                           | `.err`: `NotImplemented` + stack trace; `.exit`: `70`; `.out`: empty                                             |
| `unimplemented_message.krb`          | call a stub; compare the runtime message      | message is exactly `NotImplemented` (modeled on `native_functions/native_fn_panic_call_message_is_verbatim.krb`) |
| `unimplemented_annotated_var.krb`    | `var x: f64 = ???;` compiles; using it panics | exit `70`                                                                                                        |
| `unimplemented_missing_type.krb`     | `var x = ???;`                                | compile error at the `???`, exit `65`                                                                            |
| `unimplemented_statement.krb`        | bare `???;` marker compiles                   | compiles; panics if executed                                                                                     |
| `unimplemented_nullish_coexists.krb` | `a ?? b` still lexes/parses as nullish        | guards the maximal-munch boundary                                                                                |

**Unit** (`unit/`):

- Scanner: `???` → one `TOKEN_QUESTION_QUESTION_QUESTION`; `??` unchanged;
  a lone `?` still errors.
- Parser: `???` parses as a primary; `??? + 1.0` parses as a binary
  expression.
- Type checker: accepted with an expected type; error without one.

## Impacts

### Compatibility

- **No breakage.** Any run of three or more `?` is already a compile error
  in every program today (an odd run ends in a `TOKEN_ERROR` token; an even
  run lexes as `??` `??` and fails to parse), so no valid source changes
  meaning. The only behavior change is that previously rejected source now
  compiles.
- Purely additive: new token enum value, new AST node kind, new opcode
  index. No existing token, opcode, or AST index is renumbered, and the
  `??` nullish operator is untouched (maximal munch only changes runs of
  three or more `?`).

### Documentation

- `docs/CHANGELOG.md`: entry under the next release.
- `docs/OPCODES.md`: new row for `OP_NOT_IMPLEMENTED` (index 42).
- VS Code extension syntax file (§7).

### Other

- `krb -l` (token dump) prints the new token name.
- No impact on the standard library, the loader, or the GC (the node and
  opcode allocate nothing).

## Alternatives Considered

1. **A native function `@notImplemented()`.** No new token required, but
   verbose, needs a global binding and an argument-count story, and loses
   the Scala-familiar syntax that is the point of the request. Note that
   `@panic("...")` already exists as a manual workaround (with the
   placeholder-value problem above).
2. **Give `???` type `unit`.** Then `fun sum(a: f64, b: f64): f64 { ??? }`
   fails type checking — not what the issue asks for.
3. **A full bottom type `Nothing`.** Add a `TYPE_NOTHING` to the type
   system, have `typchkInfer()` return it for `???`, and treat it as
   compatible with every type in comparisons. More powerful (`??? + 1.0`
   type-checks, `var x = ???` yields `x: Nothing`), but Kirby's checker
   compares types by exact equality with no subtyping lattice, so this
   touches `typesEqual()`, type printing, and every type switch. A viable
   follow-up if the conservative design proves limiting (Open Question 2).
4. **A macro.** Kirby has no macros yet (they are in `docs/CHANGELOG.md`
   Future State); a built-in is simpler and available sooner.

## Open Questions

1. **Runtime message and behavior.** Bare `NotImplemented` (as in the issue)
   or `panic: NotImplemented` (consistent with `@panic`'s prefix)? And
   should the opcode `exit(70)` immediately (exactly like `@panic`, killing
   even the REPL) or `return INTERPRET_RUNTIME_ERROR` (file mode still
   exits `70`; the REPL prints `Runtime Error!` and stays alive)? The
   proposal uses bare `NotImplemented` + immediate exit, matching `@panic`;
   the stack trace identifies the location either way.
2. **Inference positions.** The proposal makes `var x = ???;`,
   `print ???;`, etc. compile errors, forcing an annotation. Alternatives:
   treat `???` as `unit` there (simple but misleading) or as a bottom type
   (Alternative 3).
3. **Bare `???;` statement.** Should a standalone `???;` be allowed as a
   TODO marker (proposed: yes, §4) or restricted to expression positions
   that have an expected type?
4. **Naming.** `NODE_UNIMPLEMENTED` with `OP_NOT_IMPLEMENTED` mixes
   prefixes; unify on `NODE_UNIMPLEMENTED`/`OP_UNIMPLEMENTED` or
   `NODE_NOT_IMPLEMENTED`/`OP_NOT_IMPLEMENTED`?
5. **Future work.** A generic `OP_PANIC` with a message-constant operand
   could host both `???` and `@panic`/`@assert`, unifying the
   runtime-error path. Out of scope here, but worth keeping in mind when
   adding the opcode.
