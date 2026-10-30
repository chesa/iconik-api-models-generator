<!-- generated-by: gsd-doc-writer -->

# Iconik API Models Generator

A CLI tool that downloads the OpenAPI v3 specifications for the
[iconik](https://www.iconik.io/) MAM/DAM API and generates a Pydantic
v2 models package from their `components.schemas`.

## Installation

``` bash
git clone git@github.com:chesa/iconik-api-models-generator.git
cd iconik-api-models-generator

# Core dependency installation
pip install -e .
```

For development (linting, formatting, and test tools):

``` bash
pip install -e ".[dev]"        # pylint, ruff, yapf, isort, pycln, mypy, pytest, black, flake8, toml-sort, uv
pip install -e ".[formatters]" # just pycln, isort, yapf
```

Requires Python 3.9 or later.

## Quick Start

``` bash
# Generate models into ./models (default output directory)
generate-iconik-models

# Custom output directory, with automatic code formatting
generate-iconik-models -o my_models -F
```

The tool downloads the OpenAPI specs for each iconik API, extracts
their component schemas, resolves cross-spec `$ref`s, and writes one
Pydantic v2 module per spec into the output directory alongside a
generated `__init__.py`.

### Command-line options

| Option             | Syntax                                   | Purpose                                                             |
| ------------------ | ----------------------------------------- | -------------------------------------------------------------------- |
| Help               | `-h, --help`                              | Displays usage information and available options                     |
| Version            | `--version`                               | Shows the program's version number and exits                         |
| Output Directory   | `-o OUTPUT_DIR, --output-dir OUTPUT_DIR`  | Specifies target location for generated models (default: `models`)   |
| Code Formatting    | `-F, --format-code`                       | Applies automated formatting using `pycln`, `isort`, and `yapf`      |
| Preserve Downloads | `--keep-downloads`                        | Retains downloaded specification files (`_specs/*.json`) after processing |
| Debug Mode         | `--debug`                                 | Enables verbose diagnostic logging for troubleshooting               |

## Generated Package Structure

By default, one module is generated per iconik API specification:

``` 
models/
├── __init__.py
├── acls.py
├── assets.py
├── auth.py
├── automations.py
├── files.py
├── jobs.py
├── metadata.py
├── notifications.py
├── search.py
├── settings.py
├── stats.py
├── transcode.py
├── users_notifications.py
└── users.py
```

When run with `--keep-downloads`, the raw OpenAPI JSON specs used to
generate each module are preserved under `models/_specs/` for
reference (useful in IDEs that can render OpenAPI JSON).

A sample of previously generated output is checked in under
`examples/models/` for reference — it is not regenerated automatically
and may not reflect the current iconik API.

## Usage

Once you've generated the models package, import and use it like any
other Python package:

``` python
from models import assets, files

# Create an asset
asset = assets.AssetCreateSchema(
    title="My Asset",
    external_id="ext-123",
    type="ASSET"
)

# Create a file
file = files.FileCreateSchema(
    directory_path="/path/to/dir",
    original_name="my_file.mp4",
    type="FILE"
)
```

## Formatting Generated Code

Passing `-F/--format-code` runs the generated package through:

1.  Import optimization with `pycln`
2.  Import organization with `isort`
3.  Trailing-whitespace and blank-line removal (via `sed`)
4.  Code style normalization with `yapf`

If you skip `-F` at generation time, you can run pycln/isort/yapf
manually against the output directory, though this skips step 3 above
(there's no standalone command for it here — `sed -i` syntax differs
between BSD/macOS and GNU/Linux, so check `platform.system()` in
`get_empty_line_removal_command()` if you need to reproduce it exactly):

``` bash
pycln --all models/
isort models/
yapf -i models/*.py
```

## License

MIT — see [LICENSE](LICENSE).
