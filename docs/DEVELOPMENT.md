<!-- generated-by: gsd-doc-writer -->

# Development

This document covers local development setup, build/lint/test commands,
code style tooling, and the pull request process for
`iconik-api-models-generator`. For first-run instructions, see the
[README](../README.md). For runtime configuration (CLI options,
constants, environment variables), see
[CONFIGURATION.md](CONFIGURATION.md).

## Local Setup

The project uses a standard `src`-layout Python package
(`package-dir = {"" = "src"}` in `pyproject.toml`). To set up a
development environment:

``` bash
git clone git@github.com:chesa/iconik-api-models-generator.git
cd iconik-api-models-generator

python -m venv .venv
source .venv/bin/activate   # or .venv\Scripts\activate on Windows

pip install --upgrade pip
pip install -e ".[dev]"
```

`requires-python = ">=3.9"` (per `pyproject.toml`). The `dev` extra
installs everything needed for linting, formatting, type checking, and
testing: `pylint`, `ruff`, `yapf`, `isort`, `pycln`, `mypy`, `pytest`,
`pytest-asyncio`, `pytest-cov`, `pytest-mock`, `black`, `flake8`,
`Flake8-pyproject`, `toml-sort`, `uv`.

A lighter `formatters` extra (`pycln`, `isort`, `yapf`) is available if
you only need to reformat code the generator produces, without the
full lint/test toolchain:

``` bash
pip install -e ".[formatters]"
```

There is no `.env` file or environment-variable setup required for
local development — see [CONFIGURATION.md](CONFIGURATION.md) for the
full explanation of how the tool is configured.

Run the generator itself once installed:

``` bash
generate-iconik-models                      # writes to ./models
generate-iconik-models -o OUTPUT_DIR -F      # custom dir + auto-format output
generate-iconik-models --keep-downloads      # keep _specs/*.json alongside generated modules
generate-iconik-models --debug
```

## Build Commands

This project has no bundler/build step for the generator itself — the
package is pure Python, built via `setuptools` (`build-backend = "setuptools.build_meta"`). There are no `npm`-style `scripts` in
`pyproject.toml`; the commands below are the ones actually run in
`bitbucket-pipelines.yml` and documented in [CLAUDE.md](../CLAUDE.md).

| Command | Description |
| --- | --- |
| `pip install -e ".[dev]"` | Install the package in editable mode with all dev tooling. |
| `ruff --config pyproject.toml check src/` | Lint `src/` with Ruff. |
| `pylint --rcfile pyproject.toml --recursive yes src/` | Lint `src/` with Pylint (`fail-under = 9`). |
| `yapf --diff --recursive src/` | Show formatting diffs for `src/` without writing changes. |
| `pytest -v tests/*` | Run the test suite. |
| `python -m build` | Build sdist/wheel distribution files (used by the release pipeline). |
| `twine check dist/*` | Validate built distribution artifacts before upload. |

The commands above mirror the `lint` and `test` step definitions in
[`bitbucket-pipelines.yml`](../bitbucket-pipelines.yml).

To format code that the tool *generates* (not the generator's own
source), run the same tools against an output directory instead of
`src/` — this is only relevant when working on the code-formatting
step of the generator (`format_generated_code()`), or manually cleaning
generated output:

``` bash
pycln --all models/
isort models/
yapf -i models/*.py
```

To sanity-check generated output for quality issues, point `pylint` at
the output directory instead of `src/`:

``` bash
pylint --rcfile pyproject.toml --recursive yes models/
```

## Code Style

Code style is enforced with three tools, each configured in
`pyproject.toml`:

  - **Ruff** (`[tool.ruff]`, `[tool.ruff.lint]`, `[tool.ruff.format]`) —
    linting and formatting. `line-length = 80`, `target-version = "py39"`.
    Run with: `ruff --config pyproject.toml check src/`
  - **Pylint** (`[tool.pylint.*]`) — static analysis. `fail-under = 9`,
    `max-line-length = 80`. Run with:
    `pylint --rcfile pyproject.toml --recursive yes src/`
  - **yapf** (`[tool.yapf]`) — code formatting, `based_on_style = "pep8"`,
    `column_limit = 80`, `indent_width = 4`. Run with:
    `yapf --diff --recursive src/` to preview changes, or
    `yapf -i --recursive src/` to apply them.

Additional tools configured but not run in the default CI `lint` step:
`isort` (`[tool.isort]`, `profile = "pycharm"`), `mypy` (`[tool.mypy]`,
`python_version = "3.9"`), and `black` (`[tool.black]`, `line-length = 88`, used manually rather than in CI).

All of these are enforced by the `lint` step in `bitbucket-pipelines.yml`,
which runs on every push to `default` and on every pull request **that
changes files under `src/**`** — the step is gated by a
`condition.changesets.includePaths` check, so a PR that doesn't touch
`src/` (e.g. a docs-only change) skips it.

## Branch Conventions

No branch naming convention is documented in
[`CONTRIBUTING.md`](../CONTRIBUTING.md); it instructs contributors to
base new work on the default branch and open a pull request when
ready. The default branch is `main`. `bitbucket-pipelines.yml` treats a
`release` branch and any Git tag as special:

  - `branches.release` — runs `lint`, `test`, and `security-scan` under a
    `Production` deployment stage.
  - `tags: '*'` — runs `lint`, `test`, `security-scan`, and
    `build-push-pypi` (publishes to PyPI; tags must match the format
    `v2025.5-beta.1` or `2025.5-beta.1` to match the package version).

## PR Process

See [`CONTRIBUTING.md`](../CONTRIBUTING.md) for the full pull request
process. Summary:

  - Keep pull requests focused on a single change; avoid bundling
    unrelated fixes.
  - Ensure `ruff`, `pylint`, and `yapf --diff` (see Code Style above)
    pass locally before opening the PR — the same checks run in CI and
    must pass before merge.
  - Ensure `pytest -v tests/*` passes locally.
  - Describe what changed and why in the PR description; reference any
    related issue.
  - CI runs the `test` pipeline step on every pull request, and the
    `lint` step whenever the pull request touches `src/**`
    (`bitbucket-pipelines.yml`, `pipelines.pull-requests`).
    <!-- VERIFY: whether merge checks / required approvals are configured at the repository-host level, since this is not defined in the repository files -->
