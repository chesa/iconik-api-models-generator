<!-- generated-by: gsd-doc-writer -->

# Architecture

## System Overview

`generate-iconik-models` is a single-purpose command-line code generator, not
a service. Invoked once per run (`generate-iconik-models` or `python -m generate_iconik_models`), it downloads the OpenAPI v3 specifications
published by the iconik MAM/DAM platform, extracts their `components.schemas`
definitions, and renders each schema as a Pydantic v2 model class into a
generated Python package (`models/` by default). The entire real
implementation lives in one module, `src/generate_iconik_models.py`
(~1,290 lines), organized as a linear pipeline of top-level functions called
in sequence from `main()`. There is no persistent process, no database, and
no network listener — the tool fetches specs, generates code, writes files
to disk, and exits.

## Component Diagram

``` text
┌─────────────────────────────────────────────────────────────────┐
│                        CLI Entry Point                          │
│         main()  (argparse: -o, -F, --keep-downloads,            │
│                  --debug, --version)                             │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Acquisition — download_specs()                                │
│    GET https://app.iconik.io/docs/{spec}/spec/ for each spec in  │
│    SPEC_NAMES; writes <output_dir>/_specs/{spec}.json            │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Extraction — extract_schemas() / extract_all_schemas()        │
│    pulls components.schemas per spec, plus a flattened           │
│    global name→schema map for cross-spec references              │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Reference resolution — resolve_schema_references()             │
│    recursively inlines $ref pointers against the global map      │
│    (defined for deep resolution; the model-builder step below    │
│    performs lightweight by-name $ref lookups independently)      │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Model building — generate_models() → generate_model_field()   │
│    → handle_oneof_anyof() / openapi_type_to_python()             │
│    builds a per-model IR dict: {name, fields, config} for        │
│    classes, or {name, is_type_alias, type_hint} for oneOf/anyOf  │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Dependency ordering — sort_models_by_dependency()              │
│    collect_model_dependencies() + topological sort (DFS) so      │
│    referenced classes are defined before they're used            │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Render & write — generate_model_code() (Jinja2 MODEL_TEMPLATE) │
│    → create_module_file() per spec, create_package_files()       │
│    writes <output_dir>/{spec}.py + <output_dir>/__init__.py      │
└──────────────────────────────┬────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Optional formatting — format_generated_code()                  │
│    pycln → isort → sed (strip trailing whitespace) → yapf,       │
│    only when --format-code is passed                              │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow

A single run of `main()` (`src/generate_iconik_models.py`) moves data through
the pipeline as follows:

1.  Parse CLI arguments and create `<output_dir>` and `<output_dir>/_specs`.
2.  `download_specs()` issues a `GET` request (60s timeout, no retry) for
    each name in `SPEC_NAMES` against `BASE_URL` (`https://app.iconik.io/docs/{spec}/spec/`),
    writes the raw JSON to `_specs/`, and returns an `OrderedDict[spec_name → parsed JSON]`. A failed individual download is logged and skipped; the run
    only aborts if zero specs download successfully.
3.  `extract_all_schemas()` flattens every spec's `components.schemas` into
    one global name→schema map so schemas in one spec can reference schemas
    defined in another.
4.  `extract_schemas()` produces the per-spec schema map that determines which
    models are written into which output module.
5.  `generate_models()` iterates each spec's schemas, builds a model
    IR (intermediate representation) dict per schema via
    `generate_model_field()` and `handle_oneof_anyof()`, then orders the
    models with `sort_models_by_dependency()`.
6.  For each spec that produced at least one model, `create_module_file()`
    computes the imports the module needs (e.g. `datetime`, `date`, `UUID`,
    `HttpUrl`, `Literal`), renders the model source through the Jinja2
    `MODEL_TEMPLATE`, writes `<output_dir>/{spec}.py`, and appends
    `model_rebuild()` calls to resolve forward references between models.
7.  `create_package_files()` writes `<output_dir>/__init__.py`, importing
    every generated module and embedding a calendar version
    (`get_calendar_version()`) plus an `__info__` dict describing each spec's
    API version, title, and OpenAPI version.
8.  If `--format-code` was passed, `format_generated_code()` runs
    `pycln` → `isort` → `sed` → `yapf` over `<output_dir>`.
9.  Unless `--keep-downloads` was passed, the `_specs/` directory is deleted.

**State management:** the tool holds no state between runs. Every run
re-downloads specs from scratch and re-derives the output package; there is
no incremental or diff mode, and no state is persisted outside the files
written to `<output_dir>`.

## Key Abstractions

| Abstraction | Purpose | Location |
|---|---|---|
| Model IR dict | Intermediate representation of a generated class or type alias before templating. Classes: `{"name", "base_class", "description", "fields", "config"}`. oneOf/anyOf unions: `{"name", "is_type_alias", "type_hint", "description"}`. | Built in `generate_models()`, consumed by `MODEL_TEMPLATE` and `sort_models_by_dependency()` |
| Field info dict | Per-property Pydantic field descriptor: `{"type_hint": str, "default": Optional[str]}`. Handles `$ref`, `oneOf`/`anyOf`, arrays, objects/`additionalProperties`, and scalar constraints (pattern/min/max/length) uniformly. | `generate_model_field()` |
| `SPEC_NAMES` | Static list of iconik API spec identifiers the tool downloads (`files`, `acls`, `assets`, `auth`, `automations`, `jobs`, `metadata`, `notifications`, `search`, `settings`, `stats`, `transcode`, `users`, `users-notifications`) | Module-level constant near the top of `src/generate_iconik_models.py` |
| `MODEL_TEMPLATE` | Single shared Jinja2 template string used to render every generated model class or type alias | Module-level constant in `src/generate_iconik_models.py`, rendered by `generate_model_code()` |
| Type mapper | Converts an OpenAPI type/format pair into a Python type-hint string | `openapi_type_to_python()` |
| Union handler | Converts `oneOf`/`anyOf` schemas into `Union[...]` type hints, forward-reference aware | `handle_oneof_anyof()` |
| Dependency sorter | Topologically orders models (DFS-based) so any class referenced by another is defined earlier in the same file | `sort_models_by_dependency()`, `collect_model_dependencies()` |

## Directory Structure Rationale

``` 
iconik-api-models-generator/
├── src/
│   ├── generate_iconik_models.py   # The entire generator: download, extract,
│   │                                # resolve, generate, sort, render, write, format
│   └── dummy_module/                # Unrelated CI lint/test scaffold (toy Calculator);
│                                     # no import relationship with the generator
├── tests/
│   └── test_dummy_module.py         # Exercises dummy_module only
├── examples/
│   └── models/                      # Checked-in sample of generated output, for reference
├── docs/                            # Project documentation
├── bitbucket-pipelines.yml          # Bitbucket Pipelines CI/CD (not GitHub Actions)
├── pyproject.toml                   # Single source of truth for dependencies and tool config
└── README.md
```

  - **`src/` (src-layout):** All real source code lives under `src/`, matching
    the `package-dir = {"" = "src"}` setuptools configuration in
    `pyproject.toml`. `generate_iconik_models.py` is the entire generator;
    because the tool is a script rather than a multi-module library, there was
    no need to split it into a package.
  - **`src/dummy_module/`:** A self-contained toy `Calculator` module used only
    to give the CI lint/test pipeline something concrete to exercise
    (`bitbucket-pipelines.yml`'s `test` step runs `--cov=src/dummy_module`). It
    has no relationship to the iconik model generator and should not be
    treated as an architectural pattern to follow.
  - **`tests/`:** Currently contains only `test_dummy_module.py`; there are no
    automated tests for `generate_iconik_models.py` itself.
  - **`examples/models/`:** A committed sample of what the tool's output looks
    like — one Python module per iconik API spec, plus `__init__.py` and the
    raw `_specs/*.json` — kept for reference, not imported by anything under
    `src/`.
  - **`pyproject.toml`:** Holds project metadata, runtime/dev dependencies, and
    every tool's configuration (setuptools, pytest, ruff, pylint, yapf, isort,
    mypy, toml-sort) in one place.

## Architectural Constraints

  - **Single-threaded, synchronous:** spec downloads happen sequentially in a
    `for` loop via `requests.get()`; there is no async or concurrent I/O.
  - **Global state:** limited to module-level constants (`SPEC_NAMES`,
    `BASE_URL`, `OUTPUT_DIR`, `MODEL_TEMPLATE`, `logger`, `__version__`); no
    mutable singletons.
  - **No circular imports:** the generator is a single file; `src/dummy_module`
    is fully independent of it.
  - **Working-directory sensitivity:** `generate_model_code()` constructs a
    Jinja2 `Environment(loader=FileSystemLoader("."))`, but this loader is
    effectively unused since `env.from_string()` renders `MODEL_TEMPLATE`
    directly.

## Error Handling

  - **Log-and-continue** for per-item failures: a single spec download failing
    (`requests.RequestException`) is logged and skipped rather than aborting
    the run.
  - **Fail-fast** only when the aggregate result is unusable: `main()` exits
    with an error if zero specs were downloaded.
  - Optional post-processing formatter calls (`pycln`, `isort`, `yapf`) are
    each wrapped in `try/except subprocess.CalledProcessError`, logged, and
    skipped individually so one missing/failing tool doesn't block the others.
  - Cleanup of the `_specs/` directory (`delete_directory()`) treats
    `FileNotFoundError`/`OSError` as non-fatal, debug-level events.

## Cross-Cutting Concerns

  - **Logging:** a single module-level `logger` (stdlib `logging`); default
    format is `"%(asctime)s - %(name)s - %(levelname)s - %(message)s"` at
    `INFO` level. The `--debug` flag switches to a more verbose format
    including filename/line number/function name at `DEBUG` level.
  - **Validation:** no input validation beyond `requests.raise_for_status()`
    on spec downloads. The generated Pydantic models validate data only when
    used by a downstream consumer — this tool does not exercise that
    validation itself.
  - **Authentication:** none. The tool downloads publicly available OpenAPI
    spec documents from iconik's documentation site; no API key or token is
    required or referenced anywhere in the codebase.

## Entry Points

  - **CLI (`generate-iconik-models`):** registered via `[project.scripts]` in
    `pyproject.toml`, resolves to `main()` in `src/generate_iconik_models.py`.
  - **Direct script execution:** `if __name__ == "__main__": sys.exit(main())`
    at the bottom of `src/generate_iconik_models.py`.
