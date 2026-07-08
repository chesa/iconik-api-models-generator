<!-- generated-by: gsd-doc-writer -->
# Testing

## Test Framework and Setup

Tests use [pytest](https://docs.pytest.org/) `~8.3.5`, installed via the
`dev` extra in `pyproject.toml` along with `pytest-asyncio~=0.26.0`,
`pytest-cov~=6.1.1`, and `pytest-mock~=3.14.0`.

Install the dev dependencies before running tests:

```bash
pip install -e ".[dev]"
```

Pytest is configured under `[tool.pytest.ini_options]` in
`pyproject.toml`:

| Setting | Value | Effect |
| --- | --- | --- |
| `testpaths` | `["tests"]` | Only the `tests/` directory is collected |
| `pythonpath` | `["src"]` | `tests/` can `import` modules from `src/` directly |
| `python_files` | `test_*.py` | Test file naming convention |
| `python_classes` | `Test*` | Test class naming convention |
| `python_functions` | `test_*` | Test function naming convention |
| `asyncio_mode` | `auto` | Async test functions run without an explicit `@pytest.mark.asyncio` marker |

**Important caveat:** the only test file currently in the repository,
`tests/test_dummy_module.py`, exercises `src/dummy_module/` — a
self-contained toy `Calculator` module that exists solely to give the CI
lint/test pipeline something concrete to run against. It has no
relationship to the actual generator. **There are no automated tests for
`src/generate_iconik_models.py`**, which is the real implementation of
this tool. See `docs/ARCHITECTURE.md` for details on this distinction.

## Running Tests

Run the full suite from the project root:

```bash
pytest -v tests/
```

Run with coverage and a JUnit XML report, matching what CI runs:

```bash
pytest -v tests/* --cov=src/dummy_module --junitxml=test-reports/report.xml
```

Run a single test file:

```bash
pytest -v tests/test_dummy_module.py
```

Run a single test function:

```bash
pytest -v tests/test_dummy_module.py::test_divide_by_zero
```

There is no dedicated `scripts.test` wrapper (this is a Python project,
not an npm project) — `pytest` is invoked directly, and no watch-mode
command is configured.

## Writing New Tests

- Name new test files `test_*.py` and place them under `tests/`.
- Name test functions `test_*` and test classes `Test*` to match the
  `python_files`/`python_functions`/`python_classes` settings above.
- `tests/__init__.py` exists but is an empty file — there is no shared
  fixture/helper file (e.g. `conftest.py`) yet.
- Follow the fixture pattern already used in
  `tests/test_dummy_module.py`: a `@pytest.fixture` function returns a
  fresh instance of the class under test, and individual `test_*`
  functions take that fixture as an argument.
- Because `pythonpath = ["src"]` is set, import modules under test with an
  absolute path from `src/`, e.g. `from src.dummy_module import Calculator`
  (as `tests/test_dummy_module.py` does) or `import generate_iconik_models`
  for the real generator module.
- If you add tests for `src/generate_iconik_models.py`, you will also need
  to update the CI `--cov=` target in `bitbucket-pipelines.yml` (currently
  scoped to `src/dummy_module` only — see CI Integration below).

## Coverage Requirements

No coverage threshold is configured anywhere in the repository — there is
no `.coveragerc`, no `[tool.coverage]` section in `pyproject.toml`, and no
`fail_under` setting. The `--cov=src/dummy_module` flag used in CI
produces a coverage report but does not enforce a minimum percentage, and
it only measures the toy `dummy_module`, not the actual generator.

## CI Integration

Tests run in Bitbucket Pipelines (`bitbucket-pipelines.yml`, `image:
python:3.11`), in the `&test` step anchor:

```bash
pip install --upgrade pip
pip install -e ".[dev]"
pytest -v tests/* --cov=src/dummy_module --junitxml=test-reports/report.xml
```

The `test-reports/report.xml` JUnit report is retained as a pipeline
artifact. The `&test` step runs in these pipelines:

| Pipeline | Trigger |
| --- | --- |
| `pull-requests: '**'` | Every pull request, alongside the `&lint` step |
| `branches: release` (stage `Deploy to Production`) | Pushes to the `release` branch, after `&lint` and before `&security-scan` |
| `tags: '*'` | Any tag push, after `&lint`, before `&security-scan` and `&build-push-pypi` |
| `custom: test` | Manually triggered custom pipeline |

The default pipeline (untagged pushes to other branches) runs only the
`&lint` step and does not execute tests.
