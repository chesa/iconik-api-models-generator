<!-- generated-by: gsd-doc-writer -->
# Configuration

`iconik-api-models-generator` is a command-line tool. It has no runtime
`.env` file, no server configuration, and no environment variables
required to run it locally. Configuration happens through:

1. Command-line options passed to the `generate-iconik-models` entry
   point.
2. Hardcoded constants in `src/generate_iconik_models.py` that define
   which Iconik API specifications are downloaded and where.
3. `[tool.*]` sections in `pyproject.toml` that configure the optional
   linting/formatting tools used during development and CI.
4. A small set of deployment-only environment variables consumed by
   `bitbucket-pipelines.yml` when publishing a release to PyPI.

## Environment Variables

No environment variables are read by `src/generate_iconik_models.py` —
there are no `os.environ` or `os.getenv` calls in the source. The only
environment variables in the project are consumed by the Bitbucket
Pipelines release step, not by the tool itself:

| Variable         | Required                  | Description                                                                                 |
| ---------------- | -------------------------- | --------------------------------------------------------------------------------------------- |
| `PYPI_USERNAME`  | Required (tag pipeline only) | PyPI account username used by `twine upload` in the `build-push-pypi` step. <!-- VERIFY: value configured in Bitbucket repository/deployment variables --> |
| `PYPI_PASSWORD`  | Required (tag pipeline only) | PyPI account password/token used by `twine upload` in the `build-push-pypi` step. <!-- VERIFY: value configured in Bitbucket repository/deployment variables --> |

These are set as Bitbucket Pipelines repository variables (see
`bitbucket-pipelines.yml`, line 2: "Prerequisites: $PYPI_USERNAME and
$PYPI_PASSWORD setup in the Deployment variables.") and are not present
anywhere in the repository itself.

## Command-Line Options

The primary way to configure a run of the generator is through
arguments to the `generate-iconik-models` command (defined in
`main()` in `src/generate_iconik_models.py`):

| Option               | Syntax                                   | Required | Default   | Description                                                    |
| -------------------- | ----------------------------------------- | -------- | --------- | ---------------------------------------------------------------- |
| Help                  | `-h, --help`                               | No       | —         | Displays usage information and exits.                            |
| Version               | `--version`                                | No       | —         | Prints the program version and exits.                            |
| Output directory      | `-o OUTPUT_DIR, --output-dir OUTPUT_DIR`   | No       | `models`  | Target directory for the generated Python package.               |
| Format generated code | `-F, --format-code`                        | No       | disabled  | Runs `pycln`, `isort`, `sed`, and `yapf` over the generated code. |
| Keep downloads        | `--keep-downloads`                         | No       | disabled  | Preserves the downloaded `_specs/*.json` files instead of deleting them after generation. |
| Debug logging         | `--debug`                                  | No       | disabled  | Switches logging to `DEBUG` level with a more detailed log format. |

All options are optional — running `generate-iconik-models` with no
arguments uses the defaults above.

## Config File Format

There is no application-specific config file (no `config.json`,
`config.yaml`, or similar). `pyproject.toml` does contain `[tool.*]`
sections that configure the development tooling used to lint, type
check, and format the codebase:

| Section                    | Tool               | Purpose                                                    |
| --------------------------- | ------------------ | ------------------------------------------------------------ |
| `[tool.pylint.*]`           | Pylint 3.3.x        | Static analysis; `fail-under = 9`, `max-line-length = 80`.    |
| `[tool.ruff]` / `[tool.ruff.lint]` / `[tool.ruff.format]` | Ruff 0.11.x | Linting and formatting; `line-length = 80`, `target-version = "py39"`. |
| `[tool.isort]`              | isort 6.0.x         | Import sorting; `profile = "pycharm"`, `line_length = 80`.    |
| `[tool.yapf]`               | yapf 0.43.x         | Code formatting; `based_on_style = "pep8"`, `column_limit = 80`. |
| `[tool.black]`              | Black 25.1.x        | Alternate formatter; `line-length = 88` (used manually, not in the default CI pipeline). |
| `[tool.mypy]`                | mypy 1.15.x         | Type checking; `python_version = "3.9"`.                      |
| `[tool.pytest.ini_options]`  | pytest 8.3.x        | Test discovery (`testpaths = ["tests"]`, `pythonpath = ["src"]`) and asyncio mode. |
| `[tool.prettier]`            | Prettier            | Markdown formatting (`print-width = 72`, `tab-width = 4`).    |

These sections are read automatically by each tool when invoked from
the project root (e.g. `pylint --rcfile pyproject.toml --recursive yes
src/`, `ruff check --config pyproject.toml src/`).

## Required vs. Optional Settings

The generator itself has no settings that cause a startup failure if
omitted — every CLI flag has a default and the tool runs with zero
arguments. The only hardcoded, non-configurable value that affects
whether the tool can produce output at all is network reachability to
`https://app.iconik.io`; if a specification download fails for a given
spec name, `download_specs()` logs the error and continues with the
remaining specs, and `main()` exits with status `1` only if **no**
specifications were downloaded at all (see `src/generate_iconik_models.py`,
`main()`).

## Defaults

Defaults are defined as module-level constants near the top of
`src/generate_iconik_models.py`:

| Constant     | Value                                                                                                                                                          | Used for                                                          |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `BASE_URL`   | `https://app.iconik.io/docs/{}/spec/`                                                                                                                          | URL template for downloading each OpenAPI specification.             |
| `OUTPUT_DIR` | `models`                                                                                                                                                        | Default value for `-o/--output-dir` when not supplied.               |
| `SPEC_NAMES` | `files`, `acls`, `assets`, `auth`, `automations`, `jobs`, `metadata`, `notifications`, `search`, `settings`, `stats`, `transcode`, `users`, `users-notifications` | Fixed list of Iconik API specifications downloaded on every run.     |

These constants are not exposed as CLI flags or environment variables;
changing which specs are fetched or the base URL requires editing the
source file directly.

## Per-Environment Overrides

The generator does not distinguish between development, staging, and
production — it always produces the same output for a given set of CLI
flags, and there are no `.env.development` / `.env.production` files
in the repository.

The only place environment-specific configuration exists is in CI:
`bitbucket-pipelines.yml` defines a `Production` deployment environment
(under `pipelines.branches.release`) that runs the lint, test, and
security-scan steps, and a separate `tags` pipeline that additionally
runs the `build-push-pypi` step using the `PYPI_USERNAME` /
`PYPI_PASSWORD` deployment variables described above.
<!-- VERIFY: exact Bitbucket deployment environment configuration (approvers, restrictions) is managed in the Bitbucket repository settings, not in this repo -->
