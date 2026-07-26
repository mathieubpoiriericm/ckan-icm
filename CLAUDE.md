# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

CKAN core (`__version__` in [ckan/__init__.py](ckan/__init__.py), currently `2.13.0a0`) — a Flask + SQLAlchemy + Solr data portal platform, in a fork of `ckan/ckan`. Python >= 3.10 (CI runs 3.10), Node >= 24 for the frontend toolchain. Reference docs live in [doc/](doc/) and are published at docs.ckan.org.

Two Python packages ship from this repo: `ckan` (core) and `ckanext` (a namespace package holding bundled extensions). Runtime dependencies are Postgres, Solr (`ckan/ckan-solr` image), and Redis.

## Commands

### Setup (local venv)

```bash
pip install -r requirements.txt -r dev-requirements.txt -e .
npm ci                     # also runs `gulp updateVendorLibs` via postinstall
ckan generate config ckan.ini   # ckan.ini is gitignored; most CLI needs -c ckan.ini
ckan -c ckan.ini db init
ckan -c ckan.ini run       # dev server on :5000
```

`.devcontainer/setup.sh` is the shortest end-to-end reference for a working instance (config-tool calls, sysadmin user, datastore permissions).

### Tests

`pyproject.toml` already sets `addopts = "... --ckan-ini=test-core.ini"` and `testpaths = ["ckan", "ckanext"]`, so a bare `pytest` picks up the test config. Tests need live Postgres/Solr/Redis as configured in [test-core.ini](test-core.ini).

```bash
pytest                                              # everything
pytest ckan/tests/lib/test_helpers.py               # one module
pytest ckan/tests/lib/test_helpers.py::test_get_translated   # one test
pytest ckanext/datastore/tests/ -k search           # filter
```

Dockerized run that mirrors CI (needs docker compose):

```bash
cd test-infrastructure && ./setup.sh && ./execute.sh   # ./teardown.sh when done
docker compose exec ckan pytest --ckan-ini=test-core-ci.ini ckan/tests/lib/test_helpers.py::test_get_translated
```

`test-core-ci.ini` is identical to `test-core.ini` except for service hostnames — use it only inside the container/CI.

### Lint, types, frontend, docs

```bash
ruff check .            # config in pyproject.toml [tool.ruff]
npx pyright             # pyright is an npm devDependency, version-pinned there
npm run build           # compile ckan/public/base/scss -> css (gulp + dart-sass)
npm run watch
npm run build-midnight-blue      # the second bundled theme
npx cypress run         # needs a server on :5000 (`ckan -c ckan.ini run`)
sphinx-build doc build/sphinx    # what the docs CI job runs
```

`ckan -c ckan.ini sass` shells out to `npm run build`; `ckan -c ckan.ini asset build` compiles webassets bundles.

### Changelog fragments (CI-enforced)

`towncrier check` runs on every PR unless it carries the `No changelog` label. Add a file `changes/<issue-or-PR-number>.<type>` where type is one of `migration`, `feature`, `bugfix`, `removal`, `misc`.

### Database migrations

Alembic, versions under [ckan/migration/versions/](ckan/migration/versions/). Generate with `ckan -c ckan.ini generate migration --autogenerate -m "..."`, then hand-edit and rename with a numeric prefix one higher than the previous file. Model changes and their migration must land in the same commit. If merging master gives `MultipleHeads`, repoint your migration's `down_revision` at master's new head (see [doc/contributing/database-migrations.rst](doc/contributing/database-migrations.rst)).

## Architecture

Request flow: **blueprint/view → action function (`logic`) → model**. Each layer has a hard rule attached to it.

- **Views** ([ckan/views/](ckan/views/)) are Flask blueprints; they render Jinja2 templates from [ckan/templates/](ckan/templates/). Extensions add routes via `IBlueprint`.
- **Logic** ([ckan/logic/](ckan/logic/)) is the only sanctioned door to data. Never touch `ckan.model` from views, `ckan.lib`, or extensions — call an action function. And never call `ckan.logic.action.get.foo()` directly: go through `logic.get_action('foo')` so `IActions` overrides apply. Same for auth: use `logic.check_access` (aliased `_check_access` inside action modules), not the `ckan.logic.auth` function.
- **Every non-underscore function defined in `ckan.logic.action.{get,create,update,delete,patch,file}` is automatically exposed at `/api/action/<name>`** (discovery loop in [ckan/logic/__init__.py](ckan/logic/__init__.py); imported names are filtered out, locally-defined helpers are not). So: prefix module-local helpers with `_`, and bring convenience imports in privately (`_get_or_bust = logic.get_or_bust`). Functions in `action/get.py` also get `side_effect_free = True` set on them automatically.
- Action functions take `(context, data_dict)`, read required keys via `get_or_bust` (raises `ValidationError`, not `KeyError`), validate against a schema from [ckan/logic/schema/](ckan/logic/schema/) (`context.get('schema')` first, then the default), and raise only `NotFound` / `NotAuthorized` / `ValidationError`.
- **Models** ([ckan/model/](ckan/model/)) own all SQLAlchemy. Add classmethods like `User.get()` rather than writing queries in `logic`. Don't pass ORM objects into templates — it produces detached-instance 500s.

Supporting subsystems:

- **Plugins** ([ckan/plugins/](ckan/plugins/)): ~32 `I*` interfaces in [interfaces.py](ckan/plugins/interfaces.py); `blanket.py` provides decorators that auto-wire conventionally-named modules. Plugins are discovered through the `ckan.plugins` entry-point group in [setup.cfg](setup.cfg) — a new bundled extension must be registered there (test-only plugins go under `ckan.test_plugins`). [ckan/plugins/toolkit.py](ckan/plugins/toolkit.py) is the stable extension-facing API surface and is explicitly *not* for internal core use.
- **Config declaration** ([ckan/config/config_declaration.yaml](ckan/config/config_declaration.yaml) + [ckan/config/declaration/](ckan/config/declaration/)): every config option is declared with type/default/description; that file generates the config reference docs and validation. Adding a new `ckan.*` option means adding it here, or `IConfigDeclaration` from an extension. Undeclared options log a warning at startup.
- **Theming**: two parallel bundles selected by config — `ckan.base_public_folder` (`public` | `public-midnight-blue`) and `ckan.base_templates_folder` (`templates` | `templates-midnight-blue`). A template or SCSS change usually needs the mirror-image change in the `-midnight-blue` tree. Static assets are declared in `webassets.yml` files under `ckan/public/base/*/`; JS uses the `ckan.module()` pattern in [ckan/public/base/javascript/](ckan/public/base/javascript/).
- **Bundled extensions** ([ckanext/](ckanext/)): real features (`datastore`, `activity`, `datapusher`, `tabledesigner`, view plugins…) plus ~35 `example_*` extensions that are executable documentation for each interface, referenced by `literalinclude` from the docs. Changing an interface means checking its `example_*` extension too.
- **Search** ([ckan/lib/search/](ckan/lib/search/)): Solr indexing/query layer; schema in [ckan/config/solr/](ckan/config/solr/).
- **Files** ([ckan/lib/files/](ckan/lib/files/)): storage abstraction built on the external `file_keeper` library (aliased `fk`), configured via `ckan.files.storage.*`, extended through `IFiles`.
- **Background jobs**: Redis + RQ via [ckan/lib/jobs.py](ckan/lib/jobs.py) (`toolkit.enqueue_job`).
- **CLI** ([ckan/cli/](ckan/cli/)): `click` groups registered in [cli.py](ckan/cli/cli.py); extensions add commands via `IClick`.

## Code conventions

Beyond PEP 8 (`ruff`, line length 88):

- Single quotes for identifiers-as-strings, double quotes for natural-language strings; `'''triple single quotes'''` for docstrings.
- `.format()` for string building, never `%`-formatting (breaks i18n). Mark user-facing strings with `_()`.
- Import aliases enforced by ruff: `ckan.plugins as p`, `ckan.plugins.toolkit as tk`, `sqlalchemy as sa`, `file_keeper as fk`. Imports grouped stdlib / third-party / `ckan` / `ckanext`. No `from module import *`.
- Docstrings on public API use Sphinx field lists (`:param x:` / `:type x:` / `:returns:` / `:raises:`) and `:py:func:`-style cross-references — they are rendered into the API docs by autodoc. Public/extension-facing functions document their exceptions; purely internal ones don't.
- Deprecate with `@ckan.lib.maintain.deprecated("short message")`, keep backward compatibility for anything reachable from extensions, themes, or the API, and add a changelog fragment.
- `pyproject.toml` carries a long `[tool.ruff] extend-exclude` list and per-file `C901`/`E501` ignores for legacy modules. Files on those lists are grandfathered, not blessed — don't add new entries, and don't "fix" an excluded file wholesale as a side effect of an unrelated change.

## Test conventions

- `ckan/tests/` mirrors `ckan/`'s module layout (`ckan/tests/logic/action/test_update.py` ↔ `ckan/logic/action/update.py`); extension tests live in `ckanext/<ext>/tests/`.
- Long, explicit test names, grouped in `TestSomeAction` classes. One assertion target per test; no shared setup on `self` — use fixtures.
- Key fixtures (from [ckan/tests/pytest_ckan/fixtures.py](ckan/tests/pytest_ckan/fixtures.py), auto-registered as a pytest plugin): `app`, `cli`, `clean_db` (slow, truly empty DB) vs `non_clean_db` (fast, just initialized — prefer it unless the test needs emptiness), `clean_index`, `clean_redis`, `clean_queues`, `with_request_context`, `mail_server`, `with_test_worker`, `with_extended_cli`, `create_with_upload`, `migrate_db_for`.
- **All plugins are force-unloaded before the test loop runs.** A test that exercises plugin-registered actions/helpers must use `with_plugins`, either as `@pytest.mark.ckan_config("ckan.plugins", "xxx")` + `@pytest.mark.usefixtures("with_plugins")`, or `@pytest.mark.with_plugins("xxx")` to add to whatever `ckan.plugins` already lists.
- Registered markers (`pyproject.toml`): `ckan_config`, `with_plugins`, `provide_plugin`. `--strict-markers` is on, and the `ckan_config` / `with_plugins` / `provide_plugin` marks auto-inject their fixtures.
- Build objects with [ckan/tests/factories.py](ckan/tests/factories.py) (`factories.Dataset()`, `Sysadmin()`, `Organization()`, `UserWithToken()`, …) and call logic through `helpers.call_action` / `helpers.call_auth`.
- Mock sparingly: fine in `logic/auth` and validator unit tests, avoid in `logic/action` and view tests.
- New or changed code needs new or changed tests; CI splits the suite 12 ways with `pytest-split` using `.test_durations`.

## Contribution flow

PRs target `master` (release-branch backports are handled by maintainers). One logical change per branch, branch named `<issue-number>-short-synopsis`. Commit messages: imperative present tense, `[#123]` issue prefix on the first line.
