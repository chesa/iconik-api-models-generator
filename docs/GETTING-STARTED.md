<!-- generated-by: gsd-doc-writer -->

# Getting Started

This guide walks through installing `generate-iconik-models`, running it
for the first time, and where to go next. See [ARCHITECTURE.md](ARCHITECTURE.md)
for how the tool works internally and [CONFIGURATION.md](CONFIGURATION.md)
for the full list of command-line options and defaults.

## Prerequisites

  - **Python `>=3.9`** — set by `requires-python` in `pyproject.toml`. The
    project is tested against Python 3.9–3.13 (see the `classifiers` list in
    `pyproject.toml`), and CI (`bitbucket-pipelines.yml`) runs on the
    `python:3.11` Docker image.
  - **`pip`** — used for all installation steps below. No other package
    manager (Poetry, uv as a resolver, etc.) is required.
  - **`git`** — to clone the repository.
  - **Network access to `https://app.iconik.io`** — the tool downloads the
    OpenAPI specifications it converts into models from this host at run
    time (see `BASE_URL` in `src/generate_iconik_models.py`). No API key or
    authentication is required for these downloads.

There is no `.env` file, `.nvmrc`, or Docker requirement for local use —
the tool is a plain Python CLI script with no runtime configuration beyond
its command-line flags (see [CONFIGURATION.md](CONFIGURATION.md)).

## Installation

1.  Clone the repository:
    
    ``` bash
    git clone git@github.com:chesa/iconik-api-models-generator.git
    cd iconik-api-models-generator
    ```

2.  Install the package. For normal use, install the core dependencies only:
    
    ``` bash
    pip install -e .
    ```
    
    For development (linting, formatting, and running the test suite),
    install the `dev` extras instead:
    
    ``` bash
    pip install -e ".[dev]"
    ```
    
    This pulls in `pylint`, `ruff`, `yapf`, `isort`, `pycln`, `mypy`,
    `pytest`, `black`, `flake8`, `toml-sort`, and `uv` as defined under
    `[project.optional-dependencies]` in `pyproject.toml`. A smaller
    `formatters` extra (`pycln`, `isort`, `yapf` only) is also available if
    you only need to reformat generated code manually.

## First Run

Once installed, the `generate-iconik-models` console script is on your
`PATH`. Run it with no arguments to generate the models package into the
default `./models` directory:

``` bash
generate-iconik-models
```

This downloads the OpenAPI specs for each iconik API (`SPEC_NAMES` in
`src/generate_iconik_models.py`), extracts their `components.schemas`,
and writes one Pydantic v2 module per spec into `models/`, alongside a
generated `__init__.py`.

To generate into a custom directory and automatically format the output:

``` bash
generate-iconik-models -o my_models -F
```

See [README.md](../README.md) for the full command-line option reference.

## Common Setup Issues

  - **`generate-iconik-models: command not found`** — the console script is
    registered by `pip install -e .` (see `[project.scripts]` in
    `pyproject.toml`). If it's missing, confirm the install completed
    without errors and that your virtual environment's `bin`/`Scripts`
    directory is on `PATH`.
  - **All spec downloads fail / "no specifications were downloaded"** —
    `download_specs()` requires outbound network access to
    `https://app.iconik.io`. A single failed spec download is logged and
    skipped, but `main()` exits with an error if none succeed. Check your
    network connection or any outbound proxy/firewall rules if this happens
    in a CI or sandboxed environment.
  - **`ModuleNotFoundError` when running `pytest`** — the dev/test tools
    (`pytest`, `pylint`, `ruff`, etc.) are only installed via the `dev`
    extra. Run `pip install -e ".[dev]"` rather than the plain
    `pip install -e .` before running tests or linters.
  - **Unsupported Python version** — the project declares
    `requires-python = ">=3.9"` in `pyproject.toml`. If `pip install`
    fails during dependency resolution, confirm your interpreter is 3.9 or
    later with `python3 --version`.

## Next Steps

  - [ARCHITECTURE.md](ARCHITECTURE.md) — how the generation pipeline works,
    from spec download through code rendering.
  - [CONFIGURATION.md](CONFIGURATION.md) — full reference for command-line
    options, defaults, and the (deployment-only) environment variables used
    by the release pipeline.
  - Run the test suite with `pytest -v tests/*` (requires the `dev` extra)
    to confirm your environment is set up correctly.
