<!-- generated-by: gsd-doc-writer -->

# Contributing

Thanks for your interest in improving `generate-iconik-models`. This
document covers coding standards, the pull request process, and how to
report issues. For environment setup, see the Installation and
Development sections of [README.md](README.md); for how the tool is
structured, see [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Development Setup

```bash
git clone git@github.com:chesa/iconik-api-models-generator.git
cd iconik-api-models-generator

pip install -e ".[dev]"        # pylint, ruff, yapf, isort, pycln, mypy, pytest, black, flake8, toml-sort, uv
```

Requires Python 3.9 or later. See [README.md](README.md) for the
default and development install commands, and
[docs/CONFIGURATION.md](docs/CONFIGURATION.md) for how the generator's
command-line options and `pyproject.toml` tool sections are set up.

## Coding Standards

All lint/format tool configuration lives in `[tool.*]` sections of
`pyproject.toml`. The `lint` step in `bitbucket-pipelines.yml` runs
these against `src/` on every push and pull request:

```bash
ruff --config pyproject.toml check src/
pylint --rcfile pyproject.toml --recursive yes src/
yapf --diff --recursive src/
```

- **Ruff** — linting; `line-length = 80`, `target-version = "py39"`.
- **Pylint** — static analysis; `fail-under = 9`, `max-line-length = 80`.
- **yapf** — code formatting; `based_on_style = "pep8"`, `column_limit = 80`,
  `indent_width = 4`.
- **isort** and **pycln** are also available (`pip install -e ".[formatters]"`)
  for import sorting and unused-import removal, but are only run
  directly against the generator's *output* (`models/`) via
  `format_generated_code()` / the `-F` flag — not part of the `lint`
  CI step above.

Run `yapf -i --recursive src/` to auto-format your changes before
committing, and fix any `ruff`/`pylint` findings the CI step above
would report.

## Testing

```bash
pytest -v tests/*
```

See [README.md](README.md) and `pyproject.toml`'s
`[tool.pytest.ini_options]` for test discovery configuration
(`testpaths = ["tests"]`, `pythonpath = ["src"]`). Note that the
current test suite (`tests/test_dummy_module.py`) exercises the
unrelated `src/dummy_module/` scaffold used to give CI something to
run coverage against — it does not cover
`src/generate_iconik_models.py`. If you add tests for the generator
itself, place them under `tests/` following the existing
`test_*.py` / `Test*` / `test_*` naming convention.

The `test` step in `bitbucket-pipelines.yml` runs:

```bash
pytest -v tests/* --cov=src/dummy_module --junitxml=test-reports/report.xml
```

## Branch Conventions

No branch naming convention is documented in this repository. Base new
work on the default branch and open a pull request when ready — CI
(`bitbucket-pipelines.yml`, `pull-requests: '**'`) automatically runs
the `lint` and `test` steps against every pull request.

## Pull Request Process

- Keep pull requests focused on a single change; avoid bundling
  unrelated fixes.
- Ensure `ruff`, `pylint`, and `yapf --diff` (see Coding Standards
  above) pass locally before opening the PR — the same checks run in
  CI and must pass before merge.
- Ensure `pytest -v tests/*` passes locally.
- Describe what changed and why in the PR description; reference any
  related issue.
- CI runs the `lint` and `test` pipeline steps automatically on every
  pull request (`bitbucket-pipelines.yml`, `pipelines.pull-requests`).
<!-- VERIFY: whether merge checks / required approvals are configured at the repository-host level, since this is not defined in the repository files -->

## Reporting Issues

No issue template is defined in this repository. When reporting a bug,
please include:

- Steps to reproduce, including the exact `generate-iconik-models`
  command-line invocation used (see the Command-line options table in
  [README.md](README.md)).
- Expected behavior vs. actual behavior.
- Your Python version and operating system.
- Relevant log output — re-run with `--debug` for verbose diagnostic
  logging.

## License

By contributing, you agree that your contributions will be licensed
under the project's [MIT License](LICENSE).
