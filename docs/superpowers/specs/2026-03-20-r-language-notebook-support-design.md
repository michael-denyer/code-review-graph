# R Language + Jupyter Notebook Support

**Date**: 2026-03-20
**Status**: Proposed

## Overview

Add R language parsing and Jupyter notebook (.ipynb) support to code-review-graph. R support includes functions, S4/R5 classes, imports, and calls. Notebook support extracts code cells from .ipynb files and parses them with the existing Python parser.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| R class support level | S4/R5 via `setClass`/`setRefClass` | Covers formal OO; S3 too heuristic-dependent |
| R import handling | `_IMPORT_TYPES["r"] = ["call"]` with filtering | Follows Ruby precedent in the codebase |
| Notebook parsing strategy | Concatenate code cells, parse as single source | Reuses existing Python parser; offset tracking maps back to cells |
| Notebook line reporting | Concatenated line numbers + `cell_index` in `extra` | Keeps schema uniform; cell-aware consumers use `extra` |
| Notebook language scope | Python only (phase 1) | Databricks multi-language support deferred to phase 2 |
| Databricks multi-language | Phase 2 follow-up | Multi-language cell routing and SQL table detection are non-trivial scope expansions |

## R Language Support

### File Extensions

`.r`, `.R` map to language `"r"` in `EXTENSION_TO_LANGUAGE`.

### Type Mappings

```python
_CLASS_TYPES["r"] = []  # Classes detected via call pattern-matching, not AST node types
_FUNCTION_TYPES["r"] = ["function_definition"]
_IMPORT_TYPES["r"] = ["call"]  # library(), require(), source() — filtered downstream
_CALL_TYPES["r"] = ["call"]
```

### Function Extraction

R functions are defined via assignment: `name <- function(x, y) { ... }`. Tree-sitter parses this as:

```
binary_operator
  identifier ("name")
  <-
  function_definition
    parameters (...)
    braced_expression { ... }
```

The `function_definition` node is listed in `_FUNCTION_TYPES["r"]`, but it has no `identifier` child — the name lives on the left side of the enclosing `binary_operator`. Special handling in `_extract_from_tree`:

1. When visiting a `binary_operator` node in R, check if the right child is a `function_definition`
2. If so, extract the name from the left child (identifier)
3. Emit a `Function` node using the `function_definition` child's span for line numbers
4. Recurse into the function body for nested calls

Also handle `=` assignment (`name = function(...)`) — tree-sitter parses this identically as `binary_operator` with `=` instead of `<-`.

### S4/R5 Class Detection

Pattern-match on `call` nodes where the function name is `setClass`, `setRefClass`, or `setGeneric`:

- Extract the class/generic name from the first string argument
- Emit a `Class` kind node
- For `setRefClass`, the `methods` argument contains `function_definition` nodes — recurse into these as methods with `enclosing_class` set

Example:
```r
MyClass <- setRefClass("MyClass",
  fields = list(name = "character"),
  methods = list(
    greet = function() { cat(paste("Hello", name)) }
  )
)
```

Produces:
- `Class` node: `MyClass`
- `Function` node: `greet` with `parent_name = "MyClass"`

### Import Extraction

`_IMPORT_TYPES["r"] = ["call"]` means all `call` nodes are candidates. Filter in `_extract_import` and `_collect_import_names`:

| R Call | Import Target |
|--------|--------------|
| `library(dplyr)` | `"dplyr"` |
| `require(ggplot2)` | `"ggplot2"` |
| `source("utils.R")` | `"utils.R"` |

Extract the first argument. For `library`/`require`, the argument is typically an unquoted identifier. For `source`, it's a string.

Non-import `call` nodes are ignored by the import extraction path (filtered by function name).

### Call Extraction

`_CALL_TYPES["r"] = ["call"]`. The existing `_get_call_name` handles simple `identifier` first-child calls (covers `func(args)`).

R namespace-qualified calls use `::` operator: `dplyr::filter(data, x > 5)`. Tree-sitter parses this with a `namespace_operator` node. Add a branch in `_get_call_name`:

```
call
  namespace_operator
    identifier ("dplyr")
    ::
    identifier ("filter")
  arguments (...)
```

Extract as `"dplyr::filter"` — take the full text of the `namespace_operator` node.

### Test Detection

R's testthat framework uses:
- Files named `test-*.R` or `test_*.R` in a `tests/testthat/` directory
- `test_that("description", { ... })` calls

Add to `_TEST_FILE_PATTERNS`:
```python
re.compile(r"test[_-].*\.[rR]$"),
re.compile(r"tests/testthat/"),
```

Add to `_TEST_PATTERNS` or handle in `_is_test_function`: detect functions defined inside `test_that()` calls, or flag the `test_that` call itself.

## Jupyter Notebook Support (.ipynb)

### Detection

Add `".ipynb": "notebook"` to `EXTENSION_TO_LANGUAGE`.

### Parsing Flow

New `_parse_notebook` method called from `parse_bytes` when language is `"notebook"`:

1. **Parse JSON**: `json.loads(source)` to get the notebook structure
2. **Check kernel language**: Read `metadata.kernelspec.language` (or `metadata.language_info.name`). If not `"python"`, return empty (phase 1 — Python only)
3. **Extract code cells**: Iterate `cells` array, collect cells where `cell_type == "code"`
4. **Filter magic commands**: Skip lines starting with `%` or `!` (IPython magics, shell commands)
5. **Concatenate with offset tracking**: Join cell sources with single blank line separators. Build a mapping: `[(cell_index, start_line, end_line), ...]`
6. **Parse concatenated source**: Pass to `_get_parser("python")` and run `_extract_from_tree` as normal
7. **Post-process nodes**: For each `NodeInfo`, look up which cell it belongs to via the offset mapping. Set `extra["cell_index"]` to the 0-based cell index

### Line Offset Mapping

```python
# Example: 3 code cells with 5, 3, 8 lines
# Cell 0: lines 1-5    (concat lines 1-5)
# Cell 1: lines 1-3    (concat lines 7-9, after 1 blank separator)
# Cell 2: lines 1-8    (concat lines 11-18)

cell_offsets = [
    (0, 1, 5),    # (cell_index, concat_start, concat_end)
    (1, 7, 9),
    (2, 11, 18),
]
```

### Edge Cases

| Case | Handling |
|------|----------|
| Magic commands (`%pip`, `%matplotlib`, `!ls`) | Skip lines starting with `%` or `!` |
| Empty notebooks / no code cells | Return just the File node |
| Malformed JSON | Return empty `([], [])` |
| Missing kernel metadata | Default to Python |
| Markdown cells | Skipped (code cells only) |
| Raw cells | Skipped |

### File Node

The notebook gets a single `File` node with:
- `kind = "File"`
- `name = "<path>.ipynb"`
- `language = "python"` (the actual parsing language, not `"notebook"`)

## Test Plan

### R Language Tests (`tests/test_multilang.py`)

New `TestRParsing` class following the existing pattern:

| Test | Assertion |
|------|-----------|
| `test_detects_language` | `.r` and `.R` map to `"r"` |
| `test_finds_functions` | Detects `add`, `multiply` (both `<-` and `=` assignment) |
| `test_finds_s4_classes` | Detects `MyClass` from `setRefClass` |
| `test_finds_class_methods` | Detects `greet` with `parent_name = "MyClass"` |
| `test_finds_imports` | Detects `library(dplyr)`, `require(ggplot2)`, `source("utils.R")` |
| `test_finds_calls` | Detects function calls including `dplyr::filter` |
| `test_finds_params` | Parameters extracted from function definitions |
| `test_detects_test_functions` | testthat patterns detected |

### Notebook Tests (`tests/test_notebook.py`)

| Test | Assertion |
|------|-----------|
| `test_detects_notebook` | `.ipynb` maps to `"notebook"` |
| `test_parses_python_notebook` | Extracts functions/classes from code cells |
| `test_cell_index_tracking` | `extra["cell_index"]` correctly maps to source cell |
| `test_cross_cell_calls` | CALLS edges connect functions across cells |
| `test_imports_from_cells` | Import statements in cells produce IMPORTS_FROM edges |
| `test_skips_magic_commands` | `%pip`, `!ls` lines don't break parsing |
| `test_empty_notebook` | Returns only File node |
| `test_non_python_kernel` | Non-Python notebooks return empty |

### Test Fixtures

- `tests/fixtures/sample.R` — R code exercising all detected constructs
- `tests/fixtures/sample_notebook.ipynb` — Python notebook with multiple code cells, imports, functions, cross-cell calls, and magic commands

## Files Modified

| File | Changes |
|------|---------|
| `code_review_graph/parser.py` | Extension mapping, type dicts, R-specific extraction in `_extract_from_tree`, `_extract_import`, `_collect_import_names`, `_get_call_name`; new `_parse_notebook` method; notebook handling in `parse_bytes` |
| `tests/test_multilang.py` | New `TestRParsing` class |
| `tests/test_notebook.py` (new) | Notebook parsing tests |
| `tests/fixtures/sample.R` (new) | R test fixture |
| `tests/fixtures/sample_notebook.ipynb` (new) | Notebook test fixture |

## Future Work (Phase 2)

- Databricks multi-language notebooks: route cells to per-language parsers based on `%sql`, `%r`, `%python` magic prefixes
- SQL cell support: extract table references as `DEPENDS_ON` edges
- R notebook support: use R parser for IRkernel notebooks
- S3 class heuristics (if demand warrants)
