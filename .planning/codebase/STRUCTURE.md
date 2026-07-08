# Codebase Structure

**Analysis Date:** 2026-07-08

## Directory Layout

``` 
iconik-api-models-generator/
├── src/
│   ├── generate_iconik_models.py   # The entire real tool (~1290 lines, single module)
│   └── dummy_module/                # Unrelated CI lint/test scaffolding (toy Calculator)
│       ├── __init__.py
│       └── __main__.py
├── tests/
│   ├── __init__.py
│   └── test_dummy_module.py         # Tests only src/dummy_module, NOT the real generator
├── examples/
│   └── models/                      # Sample pre-generated output (reference only, not source)
│       ├── __init__.py
│       ├── acls.py, assets.py, auth.py, ... (one file per iconik API spec)
│       └── _specs/                  # Sample raw downloaded OpenAPI JSON specs
├── docs/                             # Project documentation (ARCHITECTURE, CONFIGURATION, DEVELOPMENT, GETTING-STARTED, TESTING)
├── bitbucket-pipelines.yml           # Bitbucket Pipelines CI/CD (NOT GitHub Actions)
├── pyproject.toml                    # Single source of truth: deps + all tool configs
├── README.md
├── CONTRIBUTING.md
├── LICENSE
└── CLAUDE.md
```

## Directory Purposes

**`src/`:**

  - Purpose: all real Python source code (setuptools `package-dir = {"" = "src"}`
    in `pyproject.toml`)
  - Contains: `generate_iconik_models.py` (the actual generator, importable as
    top-level module `generate_iconik_models` per `[tool.setuptools.dynamic]`
    version attr and `[project.scripts]` entry point) and `dummy_module/`
    (unrelated CI scaffold package)
  - Key files: `src/generate_iconik_models.py`

**`src/dummy_module/`:**

  - Purpose: exists solely to give the lint/test CI pipeline a small, working,
    well-formed Python package to check (`bitbucket-pipelines.yml` test step runs
    `--cov=src/dummy_module`). It is a toy `Calculator` class with
    add/subtract/multiply/divide and history tracking.
  - Contains: `__init__.py` (re-exports `Calculator`, `CalculationError`,
    `DEFAULT_OPERATIONS`, `MAX_HISTORY_SIZE`), `__main__.py` (actual
    implementation, not read in detail here beyond exports)
  - Disconnected from the iconik model generator — no import relationship, no
    shared logic

**`tests/`:**

  - Purpose: pytest suite
  - Contains: `test_dummy_module.py` only — there are currently no tests for
    `src/generate_iconik_models.py` itself
  - Key files: `tests/test_dummy_module.py`

**`examples/models/`:**

  - Purpose: a checked-in sample of what the tool's output looks like (one `.py`
    module per spec, plus `__init__.py` and `_specs/*.json` raw specs), for
    documentation/reference — not hand-written source and not imported by anything
    in `src/`
  - Contains: generated Pydantic model modules
  - Key files: `examples/models/__init__.py`, `examples/models/<spec>.py`

## Key File Locations

**Entry Points:**

  - `src/generate_iconik_models.py:1202-1291` (`main()`): CLI entry point,
    registered as `generate-iconik-models` console script in `pyproject.toml`
  - `src/generate_iconik_models.py:1290-1291`: `if __name__ == "__main__"` guard

**Configuration:**

  - `pyproject.toml`: all dependency and tool config (setuptools, pylint, ruff,
    yapf, isort, mypy, pytest, toml-sort)
  - `bitbucket-pipelines.yml`: CI/CD pipeline definitions

**Core Logic:**

  - `src/generate_iconik_models.py`: 100% of real logic — download, extract,
    resolve, generate, sort, render, write, format

**Testing:**

  - `tests/test_dummy_module.py`: tests the unrelated scaffold only; no tests
    exist for the actual generator

## Naming Conventions

**Files:**

  - snake\_case module names matching package/import name
    (`generate_iconik_models.py`)
  - Generated output modules are named after the spec, with dashes converted to
    underscores (e.g., spec `users-notifications` → file `users_notifications.py`,
    per `module_name = spec_name.replace("-", "_")` in `create_module_file()`,
    `src/generate_iconik_models.py:851`)

**Directories:**

  - `src/` layout (setuptools `src`-layout convention) with
    `package-dir = {"" = "src"}`
  - Test directory mirrors none of the source structure (single flat `tests/` with
    one test file)

## Where to Add New Code

**New Feature (e.g., new pipeline stage, new CLI flag):**

  - Primary code: add a new top-level function in `src/generate_iconik_models.py`
    and wire it into `main()`'s sequential pipeline
    (`src/generate_iconik_models.py:1202-1291`); follow the existing pattern of
    pure functions taking/returning plain dicts/lists
  - Tests: add to `tests/` (currently no generator tests exist — this would be the
    first; use `pytest-mock` to stub `requests.get` per existing dev dependency)

**New Component/Module:**

  - Given the single-script architecture, new substantial functionality should
    still live in `src/generate_iconik_models.py` unless it is large enough to
    warrant extraction into a new `src/<name>/` package (mirroring the
    `src/dummy_module/` layout as a structural example, though that module's
    *content* should not be treated as a pattern to follow since it is unrelated
    scaffolding)

**Utilities:**

  - Small shared helpers currently live as plain module-level functions in
    `src/generate_iconik_models.py` (e.g., `openapi_type_to_python()`,
    `get_calendar_version()`); no separate `utils.py` exists

**CLI flags:**

  - Add `parser.add_argument(...)` calls inside `main()`
    (`src/generate_iconik_models.py:1204-1230`) alongside existing
    `-o/--output-dir`, `-F/--format-code`, `--keep-downloads`, `--debug`

## Special Directories

**`examples/models/`:**

  - Purpose: reference sample of generated output, useful for understanding the
    tool's output shape without running it
  - Generated: Yes (produced by a prior run of the tool, not committed source)
  - Committed: Yes

**`.venv/`, `.idea/`, `.codegraph/`:**

  - Local dev tooling / IDE / knowledge-graph artifacts, not part of the shippable
    package (excluded from this structure analysis)

-----

*Structure analysis: 2026-07-08*
