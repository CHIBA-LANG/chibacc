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