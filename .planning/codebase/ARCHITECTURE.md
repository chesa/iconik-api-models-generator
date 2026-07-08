<!-- refreshed: 2026-07-08 -->
# Architecture

**Analysis Date:** 2026-07-08

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│                         CLI Entry Point                      │
│              `main()` in generate_iconik_models.py            │
│                  (argparse: -o, -F, --keep-downloads,        │
│                   --debug, --version)                        │
└───────────────────────────┬───────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Fetch: download_specs()                                  │
│     GET https://app.iconik.io/docs/{spec}/spec/ per SPEC_NAMES│
│     writes <output_dir>/_specs/{spec}.json                   │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Extract: extract_schemas() / extract_all_schemas()       │
│     pulls components.schemas per spec + a flattened global   │
│     map for cross-spec $ref resolution                       │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Resolve: resolve_schema_references()                     │
│     recursively inlines $ref pointers against all_schemas    │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Model-build: generate_models() → generate_model_field()  │
│     → handle_oneof_anyof() / openapi_type_to_python()        │
│     builds per-model dict {name, fields, config} or          │
│     {name, is_type_alias, type_hint} for oneOf/anyOf          │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Order: sort_models_by_dependency()                       │
│     collect_model_dependencies() + topological sort (DFS)    │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Render/Write: generate_model_code() (Jinja2 MODEL_TEMPLATE)│
│     → create_module_file() per spec, create_package_files()  │
│     writes <output_dir>/{spec}.py + <output_dir>/__init__.py │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Optional format: format_generated_code()                 │
│     pycln → isort → sed (strip trailing ws/blank lines)      │
│     → yapf, only if --format-code passed                     │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| CLI / orchestration | Parses args, drives the pipeline end to end | `src/generate_iconik_models.py:1202-1291` (`main`) |
| Downloader | Fetches OpenAPI specs over HTTP, persists raw JSON | `src/generate_iconik_models.py:83-122` (`download_specs`) |
| Schema extraction | Pulls `components.schemas` per-spec and globally | `src/generate_iconik_models.py:125-172` |
| Reference resolver | Recursively inlines `$ref` pointers | `src/generate_iconik_models.py:175-226` |
| Type mapper | OpenAPI type/format → Python type hint string | `src/generate_iconik_models.py:229-291` |
| Union handler | oneOf/anyOf → `Union[...]` type hints, forward-ref aware | `src/generate_iconik_models.py:294-358` |
| Model builder | Schema → model dict (fields, config, or type-alias) | `src/generate_iconik_models.py:361-446` |
| Field builder | Single schema property → Pydantic field spec (aliasing, defaults, constraints) | `src/generate_iconik_models.py:449-719` |
| Dependency sorter | Topologically orders models so referenced classes are defined first | `src/generate_iconik_models.py:722-820` |
| Code renderer | Jinja2 template → Python source text | `src/generate_iconik_models.py:60-80`, `823-836` |
| Module writer | Writes one `.py` file per spec with computed imports | `src/generate_iconik_models.py:839-941` |
| Package writer | Writes `__init__.py` with calendar version + `__info__` metadata | `src/generate_iconik_models.py:944-1064` |
| Post-format pipeline | Optional pycln/isort/sed/yapf cleanup of generated output | `src/generate_iconik_models.py:1067-1182` |
| Unrelated CI scaffold | Toy `Calculator` class, exists only to give lint/test pipeline something to check | `src/dummy_module/__init__.py`, `src/dummy_module/__main__.py` |

## Pattern Overview

**Overall:** Linear batch pipeline / code generator ("script," not a service). A single Python module (`src/generate_iconik_models.py`, ~1290 lines) implements the entire flow as a sequence of pure(ish) functions called from `main()`. There is no class hierarchy, no persistent process, no web server — it runs once per invocation and exits.

**Key Characteristics:**
- Purely functional decomposition: nearly every step is a top-level function taking dicts/lists in and returning new dicts/lists out (no shared mutable global state except module-level constants like `SPEC_NAMES`, `BASE_URL`, `OUTPUT_DIR`, `MODEL_TEMPLATE`).
- Code-generation-via-templating: Jinja2 renders a single shared `MODEL_TEMPLATE` string, not per-file `.j2` template files (no `templates/` directory exists — `FileSystemLoader(".")` in `generate_model_code()` is effectively unused since `env.from_string()` is used instead).
- Best-effort/log-and-continue error handling for network calls (a failed spec download only logs a warning and is skipped, not a hard failure) but fails fast if *no* specs downloaded at all (`main()`, `src/generate_iconik_models.py:1255-1257`).
- Idempotent regeneration: each run re-downloads specs from scratch and re-derives the entire `models` package; there is no incremental/diff mode.

## Layers

**CLI/orchestration layer:**
- Purpose: parse args, sequence pipeline stages, manage output directory lifecycle
- Location: `main()`, `src/generate_iconik_models.py:1202-1291`
- Depends on: every other layer below
- Used by: the `generate-iconik-models` console-script entry point (`pyproject.toml` `[project.scripts]`)

**Acquisition layer:**
- Purpose: HTTP fetch + on-disk caching of raw OpenAPI JSON
- Location: `download_specs()`, `src/generate_iconik_models.py:83-122`
- Depends on: `requests`
- Used by: CLI/orchestration layer

**Schema-processing layer:**
- Purpose: extract, flatten, and resolve `$ref` schema references across all specs so cross-spec type references work
- Location: `extract_schemas()`, `extract_all_schemas()`, `resolve_schema_references()` (`src/generate_iconik_models.py:125-226`) — note: `resolve_schema_references()` is defined but not called from `main()`'s pipeline; `generate_model_field()`/`generate_models()` instead handle `$ref` inline by name lookup rather than full recursive resolution (see CONCERNS.md for dead-code implications)
- Depends on: raw spec dicts from acquisition layer

**Code-generation layer:**
- Purpose: transform schemas into a model-description IR (dicts), sort by dependency, render to Python source via Jinja2, and write files
- Location: `generate_models()`, `generate_model_field()`, `handle_oneof_anyof()`, `openapi_type_to_python()`, `sort_models_by_dependency()`, `collect_model_dependencies()`, `generate_model_code()`, `create_module_file()`, `create_package_files()` (`src/generate_iconik_models.py:229-1064`)
- Depends on: schema-processing layer output
- Used by: CLI/orchestration layer

**Post-processing layer:**
- Purpose: optional external formatter chain applied to already-written files
- Location: `format_generated_code()`, `get_empty_line_removal_command()` (`src/generate_iconik_models.py:1067-1182`)
- Depends on: `pycln`, `isort`, `yapf`, `sed` being present on `PATH` (each individually checked via `shutil.which`)

## Data Flow

### Primary Request Path

1. `main()` parses CLI args, creates `<output_dir>` and `<output_dir>/_specs` (`src/generate_iconik_models.py:1245-1250`)
2. `download_specs()` GETs each of `SPEC_NAMES` from `BASE_URL`, writes JSON to `_specs/`, returns an `OrderedDict[spec_name → parsed JSON]` (`src/generate_iconik_models.py:1253`)
3. `extract_all_schemas()` flattens every spec's `components.schemas` into one global name→schema map for cross-references (`src/generate_iconik_models.py:1260`)
4. `extract_schemas()` produces the per-spec schema map used to decide which models belong in which output module (`src/generate_iconik_models.py:1263`)
5. `generate_models()` iterates each spec's schemas, builds model dicts via `generate_model_field()`/`handle_oneof_anyof()`, then topologically sorts them with `sort_models_by_dependency()` (`src/generate_iconik_models.py:1266`)
6. For each spec with models, `create_module_file()` computes needed imports (datetime/date/UUID/HttpUrl/Literal), renders model source via Jinja2, writes `<output_dir>/{spec}.py`, and appends `model_rebuild()` calls for forward-ref resolution (`src/generate_iconik_models.py:1269-1271`)
7. `create_package_files()` writes `<output_dir>/__init__.py` importing every generated module, embeds a calendar version (`get_calendar_version()`) and an `__info__` dict of per-spec API version/title/openapi-version metadata (`src/generate_iconik_models.py:1274-1278`)
8. `format_generated_code()` optionally runs pycln → isort → sed → yapf over `<output_dir>` if `--format-code` was passed (`src/generate_iconik_models.py:1281`)
9. Unless `--keep-downloads`, `delete_directory()` removes `<output_dir>/_specs` (`src/generate_iconik_models.py:1283-1284`)

**State Management:**
- No persistent state between runs. All intermediate state (downloaded specs, schema maps, model IR) lives in local variables threaded through `main()`; the only durable artifacts are the files written to `<output_dir>`.

## Key Abstractions

**Model IR dict:**
- Purpose: intermediate representation of a generated class or type alias before templating
- Shape: `{"name", "base_class", "description", "fields", "config"}` for classes, or `{"name", "is_type_alias", "type_hint", "description"}` for oneOf/anyOf unions
- Examples: constructed in `generate_models()` (`src/generate_iconik_models.py:361-446`), consumed by `MODEL_TEMPLATE` (`src/generate_iconik_models.py:60-80`) and `sort_models_by_dependency()`

**Field info dict:**
- Purpose: per-property Pydantic field descriptor
- Shape: `{"type_hint": str, "default": Optional[str]}`
- Pattern: built by `generate_model_field()` (`src/generate_iconik_models.py:449-719`), handling `$ref`, `oneOf`/`anyOf`, arrays, objects/additionalProperties, and scalar constraints (pattern/min/max/length) uniformly

## Entry Points

**CLI (`generate-iconik-models`):**
- Location: `main()`, `src/generate_iconik_models.py:1202-1291`, registered via `[project.scripts]` in `pyproject.toml`
- Triggers: invoked directly from a shell after `pip install`
- Responsibilities: full pipeline orchestration described above

**Module execution (`python -m` style / direct script run):**
- Location: `if __name__ == "__main__": sys.exit(main())` (`src/generate_iconik_models.py:1290-1291`)

**Unrelated scaffold entry point:**
- `src/dummy_module/__main__.py` — a standalone toy `Calculator` CLI/module used only by `tests/test_dummy_module.py` and the CI `test` step's `--cov=src/dummy_module`; not reachable from, or related to, the iconik generator pipeline.

## Architectural Constraints

- **Threading:** Single-threaded, synchronous throughout (sequential `requests.get()` calls in a `for` loop; no async/concurrency).
- **Global state:** Module-level constants only (`SPEC_NAMES`, `BASE_URL`, `OUTPUT_DIR`, `MODEL_TEMPLATE`, `logger`, `__version__`) in `src/generate_iconik_models.py`. No mutable singletons.
- **Circular imports:** None — single-file core script; `src/dummy_module` is fully independent.
- **Working-directory sensitivity:** `generate_model_code()`'s `Environment(loader=FileSystemLoader("."))` (`src/generate_iconik_models.py:833`) depends on CWD but is functionally inert since `env.from_string()` bypasses the loader.

## Anti-Patterns

### Reference resolution defined but unused

**What happens:** `resolve_schema_references()` (`src/generate_iconik_models.py:175-226`) recursively inlines `$ref` pointers, but `main()`'s pipeline never calls it — instead `generate_model_field()` and `handle_oneof_anyof()` do lightweight by-name lookups (`ref.split("/")[-1]`) assuming the referenced class will exist in the same generated package.
**Why it's wrong:** Dead code increases maintenance surface and creates ambiguity about which resolution strategy is authoritative; a future contributor might assume `$ref`s are being fully resolved when they are not.
**Do this instead:** Either wire `resolve_schema_references()` into the pipeline where deep inlining is actually needed, or remove it and document the by-name-reference strategy as the intended design.

### Hardcoded spec list requires manual maintenance

**What happens:** `SPEC_NAMES` (`src/generate_iconik_models.py:36-51`) is a static list; new iconik API specs won't be picked up automatically.
**Why it's wrong:** Silent staleness — the tool will never notice a new spec exists at `app.iconik.io/docs/`.
**Do this instead:** If iconik exposes a spec index/catalog endpoint, fetch the list dynamically; otherwise document the manual-update requirement prominently (README already lists specs, so at minimum keep both in sync).

## Error Handling

**Strategy:** Log-and-continue for per-item failures (a single spec download failing doesn't abort the run), fail-fast only when the aggregate result is unusable (zero specs downloaded).

**Patterns:**
- `try/except requests.RequestException` around each download, logged via `logger.error` and skipped (`src/generate_iconik_models.py:117-121`)
- `try/except subprocess.CalledProcessError` around each optional formatter invocation, logged and continues to next formatter (`src/generate_iconik_models.py:1148-1154`)
- `try/except FileNotFoundError/OSError` in `delete_directory()` treated as non-fatal debug-level events (`src/generate_iconik_models.py:1192-1198`)

## Cross-Cutting Concerns

**Logging:** Single module-level `logger` via stdlib `logging`; `--debug` flag switches format/level (`src/generate_iconik_models.py:1233-1241`).
**Validation:** No schema/input validation beyond `requests.raise_for_status()`; generated Pydantic models perform validation only when *used* downstream by a consumer, not by this tool itself.
**Authentication:** Not applicable — no auth anywhere in this codebase.

---

*Architecture analysis: 2026-07-08*
