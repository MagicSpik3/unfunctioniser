# unfunctioniser

**An R debugging aid that expands package function calls into a readable, step-through script.**

> **Status:** Planning / initial TDD development. No implementation or passing tests are claimed yet.

## Why this exists

When an R pipeline produces an unexpected result for a particular input, the relevant bug may be buried several function calls below the entry point. Existing tests may not cover that input, and the developer may not know which function to inspect. Manually copying function bodies into a sequential script, matching arguments and following intermediate values is slow and error-prone.

`unfunctioniser` automates as much of that mechanical work as is practical. Given an entry function from an R package, it attempts to expand calls to functions in the same package and generates a **readable R debugging script** that can be executed line by line in RStudio (or another R environment) with the problematic input.

The objective is **diagnosis, not production-code generation**. A useful partially expanded script with clear warnings is better than a superficially complete but incorrect script.

## Intended workflow

1. Identify an entry function and a reproducible problematic input.
2. Generate a debugging script from the entry function.
3. Open the script in RStudio and execute it line by line, inspecting intermediate data.
4. Find the first transformation that introduces the unwanted result.
5. Fix the **original package function** and add or amend a regression test there.
6. Re-run the original package's tests and the reproduction case.

The generated script is **not** a replacement for the package source and should never be used as the authoritative implementation.

## Proposed API (not implemented)

```r
unfunctionise(
  package = "examplepkg",
  entry = "process_data",
  output = "debug_process_data.R"
)
```

Potential later options include `depth`, `include`, `exclude`, and `checkpoints`. These are proposals, not commitments to the initial API.

By default, expand resolvable functions belonging to the target package. Leave base R and external-package functions as calls. Unsupported or ambiguous internal calls should remain intact with a conspicuous comment explaining why they were not expanded.

## What generated code should look like

For this illustrative package code:

```r
process_data <- function(data, multiplier = 2) {
  cleaned <- clean_values(data)
  scale_values(cleaned, multiplier)
}

clean_values <- function(x) {
  x[is.na(x)] <- 0
  x
}

scale_values <- function(x, factor) {
  x * factor
}
```

A generated script should make the transformations visible, with original function boundaries and argument mappings. For example, **illustrative output, not current tool output**:

```r
# GENERATED DEBUGGING SCRIPT — DO NOT EDIT PACKAGE LOGIC HERE
# Entry: process_data(data, multiplier = 2)
# Supply your reproduction input before running.

data <- c(1, NA_real_, 3)
multiplier <- 2

# ENTER clean_values(data) | formal x <- actual data
.uf_001_x <- data
.uf_001_x[is.na(.uf_001_x)] <- 0
cleaned <- .uf_001_x
# EXIT clean_values

# ENTER scale_values(cleaned, multiplier)
# formal x <- cleaned; formal factor <- multiplier
.uf_002_x <- cleaned
.uf_002_factor <- multiplier
result <- .uf_002_x * .uf_002_factor
# EXIT scale_values

result
```

This example deliberately uses a small, straightforward subset of R. It does **not** establish a generally safe substitution strategy for lazy arguments, side effects, non-standard evaluation or complex control flow.

## Scope and safety principles

- **Readable first:** optimise for a human stepping through code, not for speed or compactness.
- **Partial expansion is acceptable:** leave unsafe calls untouched and add actionable warnings identifying the original function and reason.
- **Preserve behaviour where claimed:** no silent rewriting of semantics. If equivalence is uncertain, do not inline the call.
- **Preserve traceability:** identify original function names, call sites, source files and source lines when available.
- **Keep external functions intact:** do not recursively expand `base`, `stats`, `dplyr`, etc.
- **Avoid accidental name collisions:** distinguish locals and repeated call sites, using local scopes where needed.
- **Never silently force lazy arguments:** R promises, defaults and side effects require special care.
- **Do not blindly rename symbols inside NSE/data-mask expressions:** tidy evaluation and similar constructs need conservative handling.
- **Handle recursion and dynamic dispatch conservatively:** retain unresolved calls and report them.
- **Generated scripts must not overwrite source files.**

A completely flat script is the preferred presentation, but small `local({ ... })` blocks or retained function calls are acceptable where necessary to avoid incorrect transformations. Explicit `return()`, `...`, nested closures, S3/S4 dispatch, `do.call()`, superassignment and other difficult constructs can be unsupported initially.

## Implementation direction

Implement as an **R package**, using native R language objects (`body()`, `formals()`, `environment()`, `call`, `expression`, `substitute()`, etc.) for syntax inspection. Consider `codetools` for call discovery, but do not assume that static call resolution is complete. Use simple functions and S3 objects where they genuinely help; avoid a large compiler framework or VS Code extension at the outset. The generated script should work in RStudio even if the package is developed in VS Code.

Suggested components (adjust as tests reveal the right boundaries):

```text
R/
  unfunctionise.R       # public entry point
  resolve_functions.R   # package namespace and function resolution
  discover_calls.R      # AST traversal and call-site discovery
  bind_arguments.R      # actual/formal matching and defaults
  expand_calls.R        # conservative recursive expansion
  generate_script.R     # formatting, comments and source mapping
  diagnostics.R         # reasons for unexpanded calls

tests/testthat/
  test-end-to-end.R
  test-resolution.R
  test-arguments.R
  test-expansion.R
  test-diagnostics.R

tests/fixtures/          # small, deterministic example packages/code
```

This is a suggested structure, not a requirement to create every file immediately.

## Test-driven development (TDD) — mandatory workflow

We will develop this package using **red → green → refactor**:

1. **Red:** write a small failing `testthat` test that specifies one observable behaviour. Run it and confirm it fails for the expected reason.
2. **Green:** implement the smallest change that passes the test. Avoid adding speculative features.
3. **Refactor:** simplify the implementation while keeping all tests green.
4. Run the full relevant suite after each change, and record any outstanding limitations.

Prefer small, deterministic tests that exercise the public API. Unit tests are valuable, but **end-to-end tests are essential**: they must generate an actual `.R` script, execute it in a controlled environment with sample input, and compare its result with the original entry function.

Use `testthat` (edition 3) and standard R package checks. Keep test fixtures tiny and comprehensible; do not depend on private workplace packages, network access, credentials or large datasets. Avoid brittle tests that compare entire generated files character-for-character unless formatting itself is under test. Assert important comments/diagnostics separately from execution results.

### Initial end-to-end fixtures

Create a tiny fixture package (or equivalent isolated namespace fixture) with the following cases, adding them **one at a time under TDD**:

| Case | Minimal scenario | Required assertion |
|---|---|---|
| E2E-01 | Entry function calls one simple helper | Generated script executes and matches original result |
| E2E-02 | Two sequential helpers transform a small vector/data frame | Intermediate statements are visible; final result matches |
| E2E-03 | Formal names differ from actual argument names; named arguments are reordered | Correct argument binding and matching result |
| E2E-04 | Helper is called twice and both calls use a local variable with the same name | No variable collisions; result matches |
| E2E-05 | Helper calls another helper | Recursive expansion works to the supported depth |
| E2E-06 | An unsupported construct is encountered | Original call is retained, warning explains why, and script still runs |

Start with **E2E-01 and E2E-02** before broadening support. Use explicit test inputs containing values such as `NA` where relevant. Include tests for visible function-boundary comments and output-file creation. Execute generated scripts in fresh environments with appropriate parent environments and compare results using `expect_equal()`; also check any observable side effects relevant to the fixture.

Later regression cases should include missing/default arguments, early returns, conditional branches, loops, pipes, lazy arguments with side effects, tidy evaluation, namespace-qualified calls and recursion. A test for an unsupported feature should confirm that it is **safely retained and diagnosed**, not that it is magically inlined.

### First milestone / definition of done

A developer can point the tool at a deliberately small R package with an entry function and one or two ordinary helper calls, generate a readable debugging script, execute it with sample data, and obtain the same result as the original. The output identifies where each expanded section came from. Any unsupported calls are left intact with clear comments. All initial end-to-end and relevant unit tests pass.

## Instructions for AI coding assistants

**Read this README before modifying the repository.** Treat it as the project brief until superseded by explicit user decisions.

1. **Do not assume the project is already implemented.** Inspect the repository, current tests and `git status` first.
2. **Work incrementally with TDD.** Propose one small behaviour, add a failing test, implement it, then run tests. Do not generate a large speculative codebase in one pass.
3. **Keep the user's actual goal in mind:** a line-by-line debugging aid for unfamiliar R package pipelines, not a universal optimiser or perfect compiler.
4. **Prioritise end-to-end evidence.** Demonstrate that the generated script actually runs on sample input and agrees with the original function for supported cases.
5. **Never silently change R semantics.** Pay particular attention to promises/lazy evaluation, lexical scoping, argument matching, return values, `...`, side effects, and non-standard evaluation. When uncertain, preserve the call and emit a diagnostic.
6. **Prefer native R and readable code.** Use clear function names, documentation, comments explaining non-obvious transformations, and small modules. Avoid needless dependencies and R6 unless justified.
7. **Preserve original source context.** Emit useful comments identifying function names, call sites and original locations when available.
8. **Be precise about test status.** Distinguish tests you actually ran from tests you recommend. Never claim equivalence outside the supported, tested subset.
9. **Do not overwrite user work.** Inspect files before editing; keep changes focused; never overwrite generated outputs without a deliberate policy.
10. **After each milestone**, summarise changed files, tests run, known limitations, and the next smallest test to add.
11. **Use UK English** in documentation and user-facing messages.
12. **Ask for a real-world example only after** the tiny fixtures demonstrate the basic end-to-end workflow; do not require private package code to begin development.

## Non-goals for the first version

- Full support for every R language construct.
- Fully flattening arbitrary package dependencies.
- Automatically finding or fixing the bug.
- Replacing the original package's tests.
- Building a VS Code extension before the R core works.
- Producing production-ready rewritten source code.

## Next action

Initialise the R package skeleton with `testthat`, create the smallest deterministic fixture for **E2E-01**, and write the failing end-to-end test before implementing the public API.
