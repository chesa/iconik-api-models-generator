# Technology Stack

**Analysis Date:** 2026-07-08

## Languages

**Primary:**

  - Python 3.9+ (targets 3.9–3.13 per `pyproject.toml` classifiers) - all source
    code in `src/generate_iconik_models.py` and `src/dummy_module/`
  - Jinja2 template string (embedded in Python) - `MODEL_TEMPLATE` constant in
    `src/generate_iconik_models.py:60-80`, used to render generated Pydantic model
    source

**Generated output:**

  - Python (Pydantic v2 model modules) - written by the tool at runtime into an
    output directory (default `models/`), not present in this repo except as
    samples in `examples/models/`

## Runtime

**Environment:**

  - CPython 3.9+ (`requires-python = ">=3.9"` in `pyproject.toml` is an open
    floor, not a pin; `.venv/` is gitignored local state so its exact version
    varies per developer/CI run; CI uses `python:3.11` Docker image per
    `bitbucket-pipelines.yml`)

**Package Manager:**

  - `pip` (setuptools build backend, `pyproject.toml` driven)
  - `uv~=0.7.2` listed as an optional dev dependency (not the primary installer)
  - Lockfile: none present (no `requirements.txt`/`uv.lock`/`poetry.lock`)

## Frameworks

**Core:**

  - Pydantic v2 (`pydantic~=2.11.4`) - target framework for all generated models;
    `BaseModel`, `Field`, `HttpUrl` used in generated code
  - Jinja2 (`Jinja2~=3.1.6`) - renders `MODEL_TEMPLATE` into Python source in
    `generate_model_code()` (`src/generate_iconik_models.py:823-836`)
  - requests (`requests~=2.32.3`) - HTTP client for downloading OpenAPI specs in
    `download_specs()` (`src/generate_iconik_models.py:83-122`)

**Testing:**

  - pytest (`pytest~=8.3.5`) + `pytest-asyncio`, `pytest-cov`, `pytest-mock` -
    configured in `pyproject.toml` `[tool.pytest.ini_options]`;
    `pythonpath = ["src"]`, `testpaths = ["tests"]`

**Build/Dev (linting/formatting, all invoked via `pyproject.toml` config
sections):**

  - ruff (`ruff~=0.11.7`) - primary linter, `[tool.ruff]` / `[tool.ruff.lint]`
  - pylint (`pylint~=3.3.6`) - secondary linter, `[tool.pylint.*]`
    (fail-under=9.0, max-line-length=80)
  - yapf (`yapf~=0.43.0`) - code formatter, `[tool.yapf]` (PEP8-based, 80-col)
  - isort (`isort~=6.0.1`) - import sorter, `[tool.isort]`
  - pycln (`pycln~=2.5.0`) - unused import remover
  - mypy (`mypy~=1.15.0`) - type checker, `[tool.mypy]`
  - black (`black~=25.1.0`) and flake8 (`flake8~=7.2.0`) also listed as dev deps
    but not invoked in CI pipeline (only ruff/pylint/yapf run in
    `bitbucket-pipelines.yml`)
  - toml-sort (`toml-sort~=0.24.2`) - keeps `pyproject.toml` sorted

## Key Dependencies

**Critical (runtime, non-dev):**

  - `requests~=2.32.3` - all network I/O (downloading specs)
  - `Jinja2~=3.1.6` - code generation templating
  - `pydantic~=2.11.4` - the target model framework the tool generates code for
    (not used by the generator's own logic at runtime, only referenced as the
    output's dependency)

**Infrastructure:**

  - Standard library only otherwise: `argparse`, `json`, `logging`, `subprocess`,
    `shutil`, `glob`, `platform`, `collections.OrderedDict`, `datetime`

## Configuration

**Environment:**

  - No `.env` files or environment-variable-driven config detected
  - CI secrets (`PYPI_USERNAME`, `PYPI_PASSWORD`) configured as Bitbucket
    Deployment variables, referenced only in `bitbucket-pipelines.yml`

**Build:**

  - `pyproject.toml` - single source of truth for project metadata, dependencies,
    and all tool configs (setuptools, pylint, ruff, yapf, isort, mypy, pytest)
  - Dynamic versioning: `version = {attr = "generate_iconik_models.__version__"}`
    reads `__version__` from `src/generate_iconik_models.py:26` (currently
    `"2025.5-beta.3"`)
  - CLI entry point: `generate-iconik-models = "generate_iconik_models:main"`
    (`pyproject.toml` `[project.scripts]`)

## Platform Requirements

**Development:**

  - Python 3.9+ virtual environment (`.venv/` is gitignored local state; no
    `.python-version` file pins an exact version)
  - macOS/Linux for full formatter chain (`get_empty_line_removal_command()` in
    `src/generate_iconik_models.py:1159-1182` uses BSD `sed` on Darwin, GNU `sed`
    on Linux, and explicitly skips sed-based empty-line removal on Windows)

**Production:**

  - No deployment target — this is a CLI tool distributed as a PyPI package
    (`generate-iconik-models`), not a hosted service
  - Published to PyPI via tagged commits in Bitbucket Pipelines (`build-push-pypi`
    step in `bitbucket-pipelines.yml`)

## Notable Structural Note

`src/dummy_module/` (a toy `Calculator` class in `src/dummy_module/__main__.py`)
is unrelated to the real generator. It exists purely as CI-pipeline scaffolding
to give `pytest`/`pylint`/`ruff`/`coverage` something concrete to exercise (see
`tests/test_dummy_module.py` and the `--cov=src/dummy_module` flag in
`bitbucket-pipelines.yml`). It is not part of the iconik model-generation
pipeline and has no import relationship with `src/generate_iconik_models.py`.

-----

*Stack analysis: 2026-07-08*
