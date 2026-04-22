# ChibaCC Test Workflow

This file records the validated build and execution commands for the current `chibacc` stage.

## 1. Build `chibacc`

Build the project with the stable compiler:

```text
./main.o --project chibacc
```

This writes the current `chibacc` binary to:

```text
./chibacc/target/debug/main.o
```

## 2. Ordinary Rule Debug Run

Minimal ordinary-rule example:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/simple_rule.chibacc \
  chibacc/examples/simple_rule.source \
  -o chibacc/examples/simple_rule.out
```

Broader ordinary-rule example:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/ordinary_rule.chibacc \
  chibacc/examples/ordinary_rule.source \
  -o chibacc/examples/ordinary_rule.out
```

Expected result: the output file contains:

```text
== LabeledAST ==
GeneratedParser.OK
```

## 3. Extract Generated Parser Source

Extract generated parser code from a debug output file:

```text
awk 'f{print} /^== Generated Parser Preview ==$/{f=1; next}' \
  chibacc/examples/sample.out > chibacc/src/generated/sample_parser.chiba
```

Rebuild `chibacc` to confirm the extracted parser compiles:

```text
./main.o --project chibacc
```

The main tool now compiles and executes generated parsers through the internal project:

```text
chibacc/generated_runner
```

## 4. Pratt Debug Run

Focused Pratt example:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/pratt_rule.chibacc \
  chibacc/examples/pratt_rule.source \
  -o chibacc/examples/pratt_rule.out
```

Richer sample Pratt example:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/sample.chibacc \
  chibacc/examples/source.sample \
  -o chibacc/examples/sample.out
```

Expected result: the output file contains:

```text
== LabeledAST ==
GeneratedParser.OK
```

## 5. Execute Generated Pratt Parser

Extract the generated parser used by the runner:

```text
awk 'f{print} /^== Generated Parser Preview ==$/{f=1; next}' \
  chibacc/examples/sample.out > chibacc/src/generated/sample_parser.chiba
cp chibacc/src/generated/sample_parser.chiba \
  working/sample_parser_runner/src/sample_parser.chiba
```

Build and run the isolated runner project:

```text
./main.o --project working/sample_parser_runner
./working/sample_parser_runner/target/debug/main.o
```

Expected result:

```text
[sample-parser-runner] OK
```

## 6. Predicate Parser Workflow

Generate debug output for the standard predicate syntax example:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/predicate_rule.chibacc \
  chibacc/examples/predicate_rule.source \
  -o chibacc/examples/predicate_rule.out
```

The AST / IR should show the predicate as:

```text
AtomPredicate: ?pred_ok(name)
IrAtomPredicate: ?pred_ok(name)
```

The `LabeledAST` section should now come from generated parser execution:

```text
== LabeledAST ==
GeneratedParser.OK
ConsumedTokens: Ident
```

Extract the generated parser and sync it into the isolated runner:

```text
awk 'f{print} /^== Generated Parser Preview ==$/{f=1; next}' \
  chibacc/examples/predicate_rule.out > chibacc/src/generated/predicate_parser.chiba
cp chibacc/src/generated/predicate_parser.chiba \
  working/predicate_parser_runner/src/predicate_parser.chiba
```

Build and run the predicate runner:

```text
./main.o --project working/predicate_parser_runner
./working/predicate_parser_runner/target/debug/main.o
```

Expected result:

```text
[predicate-parser-runner] OK
```

## 7. Current Notes

- `source.xxx` is currently interpreted as whitespace-separated token tags by both the debug runtime and the generated-parser runner bridge.
- `main.chiba` now prefers generated parser execution for the `LabeledAST` section and falls back to the debug runtime only when the generated bridge cannot build or execute.
- The generated-parser bridge no longer depends on `working/`; it uses the internal project at `chibacc/generated_runner`.
- The generated runner accepts token-tag input from either `stdin` or an optional source-file path passed as `argv[1]`.
- The debug runtime fallback now has a file-aware entrypoint that can execute ordinary token-bound predicates via the same internal runner project.
- Generated parser execution is also validated through isolated runner projects under `working/`.
- Pratt runtime and Pratt codegen are already wired through the sample chains above.
- Standard predicate syntax `name:Ident ?pred_ok(name)` is now parsed and validated through generated parser execution.

## 8. Direct Runtime Predicate Check

The runtime predicate fallback can be validated directly with the dedicated harness:

```text
./main.o --project working/runtime_predicate_runner
./working/runtime_predicate_runner/target/debug/main.o
```

Expected result:

```text
[runtime-predicate-runner] OK
```

Current scope:

```text
run_start_rule_with_file(...) currently executes ordinary token-bound predicates.
```

## 9. Recovery Check

The generated Pratt parser should now preserve the deepest consumed token prefix on malformed input instead of collapsing to an empty failure.

Use the existing sample grammar with a missing closing parenthesis:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/sample.chibacc \
  chibacc/examples/sample_missing_rparen.source \
  -o chibacc/examples/sample_missing_rparen.out
```

Expected `LabeledAST` section:

```text
== LabeledAST ==
GeneratedParser.Err
HasPartialAst: 0
ConsumedCount: 2
ConsumedTokens: LParen Ident
```

After that command regenerates and rebuilds the internal runner, the generated parser can be checked directly in either mode:

```text
cat chibacc/examples/sample_missing_rparen.source \
  | ./chibacc/generated_runner/target/debug/main.o

./chibacc/generated_runner/target/debug/main.o \
  chibacc/examples/sample_missing_rparen.source
```

Expected direct runner result in both cases:

```text
GeneratedParser.Err
HasPartialAst: 0
ConsumedCount: 2
ConsumedTokens: LParen Ident
```

Nested-rule recovery is also validated with a deeper malformed token stream:

```text
printf 'Ident LParen Ident Comma RParen Dot Ident' \
  > /tmp/generated_nested_recovery.source

./chibacc/generated_runner/target/debug/main.o \
  /tmp/generated_nested_recovery.source
```

Expected result:

```text
GeneratedParser.Err
HasPartialAst: 1
ConsumedCount: 7
ConsumedTokens: Ident LParen Ident Comma RParen Dot Ident
```

Stmt-level recovery is also validated with a minimal statement-list grammar where the first statement is missing an expression before `Semi`, but the parser still resynchronizes at the statement boundary and continues into the next statement:

```text
./chibacc/target/debug/main.o \
  chibacc/examples/stmt_recovery.chibacc \
  chibacc/examples/stmt_recovery_missing_expr.source \
  -o /tmp/stmt_recovery.out

./main.o --project chibacc/generated_runner

./chibacc/generated_runner/target/debug/main.o \
  chibacc/examples/stmt_recovery_missing_expr.source
```

Expected result:

```text
GeneratedParser.Err
HasPartialAst: 1
ConsumedCount: 9
ConsumedTokens: Let Ident Assign Semi Let Ident Assign Ident Semi
```

The current recovery slice now covers malformed Pratt-prefix input, nested per-rule recovery, and stmt-level recovery in generated execution, while preserving the deepest consumed prefix instead of collapsing to an empty failure.