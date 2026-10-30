# External Integrations

**Analysis Date:** 2026-07-08

## APIs & External Services

**iconik MAM/DAM Platform (OpenAPI specs):**

  - Service: iconik API documentation site,
    `https://app.iconik.io/docs/{spec}/spec/` (`BASE_URL` in
    `src/generate_iconik_models.py:54`)
  - Purpose: source of truth for OpenAPI v3 JSON specs, one per API area
  - Fixed spec list fetched (`SPEC_NAMES`, `src/generate_iconik_models.py:36-51`):
    `files`, `acls`, `assets`, `auth`, `automations`, `jobs`, `metadata`,
    `notifications`, `search`, `settings`, `stats`, `transcode`, `users`,
    `users-notifications`
  - SDK/Client: plain `requests.get()` call, 60s timeout, no retry logic
    (`download_specs()`, `src/generate_iconik_models.py:83-122`)
  - Auth: none — these are public documentation endpoints; no API key/token
    required or referenced anywhere in the codebase

## Data Storage

**Databases:**

  - None. No database client, ORM, or connection string exists in the codebase.

**File Storage:**

  - Local filesystem only:
      - Downloaded specs cached to `<output_dir>/_specs/*.json`
        (`download_specs()`, `src/generate_iconik_models.py:83-122`), deleted
        after generation unless `--keep-downloads` is passed (`delete_directory()`
        call in `main()`, `src/generate_iconik_models.py:1283-1284`)
      - Generated Python modules written to `<output_dir>/*.py` and
        `<output_dir>/__init__.py` (`create_module_file()`,
        `create_package_files()`)
      - Example pre-generated output committed under `examples/models/` for
        reference only (not a runtime dependency)

**Caching:**

  - None (CI pip caching in `bitbucket-pipelines.yml` is a build concern, not
    application-level caching)

## Authentication & Identity

**Auth Provider:**

  - None. The tool downloads publicly available OpenAPI spec JSON; it does not
    authenticate against the iconik API itself and contains no auth/session logic.

## Monitoring & Observability

**Error Tracking:**

  - None (no Sentry/Rollbar/etc.)

**Logs:**

  - Python standard `logging` module, module-level logger
    (`logger = logging.getLogger(__name__)`, `src/generate_iconik_models.py:33`)
  - Default format: `"%(asctime)s - %(name)s - %(levelname)s - %(message)s"`,
    level `INFO`
  - `--debug` CLI flag switches to a more verbose format including
    filename/lineno/funcName and `DEBUG` level (`main()`,
    `src/generate_iconik_models.py:1233-1241`)

## CI/CD & Deployment

**Hosting:**

  - PyPI (package `generate-iconik-models`) — this is a CLI tool, not a hosted
    service

**CI Pipeline:**

  - Bitbucket Pipelines (`bitbucket-pipelines.yml`), NOT GitHub Actions
      - `lint` step (default pipeline + PRs): `ruff check`, `pylint --recursive`,
        `yapf --diff` against `src/`
      - `test` step: `pytest tests/* --cov=src/dummy_module` (coverage scoped only
        to the unrelated dummy module, not the real generator script)
      - `security-scan` step: Bitbucket pipe `atlassian/git-secrets-scan:3.1.0`
      - `build-push-pypi` step: runs only on tagged commits; validates the git tag
        matches `dummy_module.__version__` (note: this appears to check the wrong
        module's version — see CLAUDE.md's "Known repo inconsistencies"), builds
        via `python -m build`, checks with `twine check`, installs and smoke-tests
        the wheel, then `twine upload` to PyPI using
        `PYPI_USERNAME`/`PYPI_PASSWORD` deployment variables
      - Pipelines defined: `default` (lint only), `branches.release`
        (lint+test+security-scan, deployed as "Production"), `tags.*` (full
        lint+test+security-scan+publish),
        `custom.build-and-push`/`custom.lint`/`custom.security-scan`/`custom.test`,
        `pull-requests.**` (lint+test)

## Environment Configuration

**Required env vars:**

  - `PYPI_USERNAME`, `PYPI_PASSWORD` — Bitbucket Deployment variables, used only
    in the `build-push-pypi` CI step for `twine upload`

**Secrets location:**

  - Bitbucket repository/deployment-level variables (not stored in repo). No
    `.env` files present in the repository.

## Webhooks & Callbacks

**Incoming:**

  - None

**Outgoing:**

  - None (aside from the outbound `GET` requests to
    `https://app.iconik.io/docs/{spec}/spec/` described above)

-----

*Integration audit: 2026-07-08*
