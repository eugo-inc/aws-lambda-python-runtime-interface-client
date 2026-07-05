---
name: eugo-rebuild
description: Decide what level of rebuild a change to the eugo awslambdaric fork actually requires, and when the protomolecule consumer package must be rebuilt / pin-bumped. Activates on "rebuild awslambdaric", "do I need to rebuild", "pin bump awslambdaric", "does this change need a wheel build", "bump meta.json commit".
---

# eugo-rebuild (awslambdaric)

This package is small: a full wheel build is well under a minute. The decision
that actually matters is not local rebuild cost but whether the protomolecule
consumer needs a pin bump. Walk the diff, pick the first matching row.

## In-repo: what to run for a given diff

| Files changed | Action |
|---------------|--------|
| Docs, `.devcontainer/`, `.mcp.json`, `.claude/skills/`, `.github/` | Nothing. |
| `tests/`, `requirements/dev.txt`, `Makefile` | Nothing to build; `make test` if tests changed. |
| `deps/` tarballs or `scripts/update_deps.sh` | Nothing - unused by the eugo meson build (upstream setup.py leftovers). |
| `awslambdaric/*.py` only | No compile step needed, but run `make test` and rebuild the wheel anyway before pinning (mesonpy copies .py into the wheel). |
| `awslambdaric/__init__.py` touching the `__version__` line | Verify meson.build's sed still extracts it: `meson setup` must not fail in `project()`. |
| `meson.build`, `pyproject.toml` | Configure gate first (`meson setup eugo_build ...`), then full wheel + smoke. |
| `awslambdaric/runtime_client.cpp` | Full wheel + smoke (`import awslambdaric.runtime_client`). |

Build and smoke commands live in the `eugo-build-and-test` skill.

## When protomolecule must rebuild / pin-bump

`protomolecule/dependencies/python/wave_4/awslambdaric/meta.json` pins this repo
by `version.commit` on branch `main`. Consequences:

- Landing ANYTHING on `main` changes nothing downstream until that commit sha
  is bumped. Merge/adopt are decoupled - use this deliberately.
- Every adoption is a pin bump; there is no "partial" rebuild downstream. The
  harness reruns the whole `setup` pip install at the new commit.
- Docs-only commits (like this skills suite) do NOT need a pin bump.
- New runtime imports in the Python sources (pattern: `snapshot_restore_py`)
  need `meta.json dependencies.runtime` AND `pyproject.toml [project]
  dependencies` updated with the pin bump.
- New native link deps in meson.build need `meta.json
  dependencies.buildtime.standalone` updated, or the DAG never builds them.
- Only pin shas already pushed to `eugo-inc/...-interface-client` `main`;
  never force-push `main` (breaks every consumer fetch).
- If entrypoint-relevant behavior changed (`__main__.py`, `bootstrap.py`),
  re-test an epstein-drive lambda base image boot after the bump.
