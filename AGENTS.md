# Repository Guidelines

## Project Structure & Module Organization

CKAN core lives in `ckan/`: Flask routes are under `ckan/views/`, business
actions under `ckan/logic/`, SQLAlchemy models under `ckan/model/`, and Jinja
templates/static assets under `ckan/templates/` and `ckan/public/`. Keep the
request flow `view -> logic action -> model`; extensions should use the public
toolkit rather than importing models directly. Bundled plugins live in
`ckanext/<extension>/`. Documentation is in `doc/`, Cypress specifications in
`cypress/e2e/`, and release-note fragments in `changes/`. The
`templates-midnight-blue/` and `public-midnight-blue/` trees mirror the default
theme; assess both when changing UI behavior.

## Build, Test, and Development Commands

- `pip install -r requirements.txt -r dev-requirements.txt -e .` installs CKAN
  and Python development dependencies.
- `npm ci` installs the Node 24 toolchain and refreshes vendored frontend files.
- `ckan generate config ckan.ini && ckan -c ckan.ini run` creates local config
  and starts the development server.
- `pytest` runs Python tests using `test-core.ini`; target a test with
  `pytest ckan/tests/lib/test_helpers.py::test_get_translated`.
- `ruff check .` and `npx pyright` run linting and type checks.
- `npm run build` and `npm run build-midnight-blue` compile theme assets;
  `npm test` runs Cypress against a server on port 5000.
- `sphinx-build doc build/sphinx` builds the documentation.

Tests require PostgreSQL, Solr, and Redis. For a CI-like Docker environment,
run `cd test-infrastructure && ./setup.sh && ./execute.sh`, then `./teardown.sh`.

## Coding Style & Naming Conventions

Follow `.editorconfig`: UTF-8, LF endings, spaces, two-space defaults, and
four-space Python indentation. Ruff enforces an 88-character Python line
length. Use `snake_case` for modules/functions, `PascalCase` for classes, and
private prefixes for action-module helpers so they are not exposed through the
API. Preserve established import aliases such as `ckan.plugins.toolkit as tk`
and `sqlalchemy as sa`.

## Testing Guidelines

Pytest files are named `test_*.py`. Core tests mirror source paths under
`ckan/tests/`; extension tests belong in `ckanext/<extension>/tests/`. Add or
update focused tests for every behavior change and do not reduce coverage.
Cypress files use `*.cy.js`. Use only markers registered in `pyproject.toml`.

## Commit & Pull Request Guidelines

Recent history uses short imperative subjects, often with `fix:`, `feat:`, or
`chore:` prefixes; older commits commonly include `[#123]`. Keep each branch
and commit to one logical change. Target `master`, link the issue (`Fixes #...`),
describe the proposed fix, and check applicable PR-template items for tests,
docs, API, user-visible, or backport changes. Add
`changes/<issue>.<feature|bugfix|migration|removal|misc>` unless the PR is
explicitly exempt from changelog requirements.
