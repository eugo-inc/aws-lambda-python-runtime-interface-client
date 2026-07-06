---
name: eugo-upstream-merge
description: Merge upstream aws/aws-lambda-python-runtime-interface-client into the eugo-inc fork (branch eugo-main). Covers the meson/mesonpy divergence, the merge recipe, and the protomolecule commit-pin bump. Activates on "merge upstream", "upstream sync", "awslambdaric sync", "catch up to aws main".
---

# Merging upstream aws-lambda-python-runtime-interface-client

## What this fork is / who consumes it

Eugo fork of `aws/aws-lambda-python-runtime-interface-client` (the `awslambdaric`
PyPI package). Canonical branch: `eugo-main`. Sole build consumer:
`protomolecule/dependencies/python/wave_4/awslambdaric/` - its `meta.json` pins
this repo by `git_commit` on branch `eugo-main` (currently `6075662`, one merge behind
head because the devcontainer merge has no build impact). Because the pin is a
commit, merging upstream and adopting the merge are DECOUPLED: landing the merge
on `eugo-main` changes nothing until the meta.json commit is bumped.
Runtime consumer: epstein-drive `base_docker_images` lambda images
(`exec ... awslambdaric ...` entrypoint) - they get whatever protomolecule built.

## Eugo divergence inventory (vs upstream)

Total delta is small (7 files) and there are NO `@EUGO_CHANGE` annotations here;
`meson.build` uses `# === @begin/@end: <section> ===` markers instead.

- `setup.py` DELETED. Upstream builds the `runtime_client` C++ extension via
  setup.py, compiling vendored tarballs in `deps/` (curl-7.83.1,
  aws-lambda-cpp-0.2.6).
- `meson.build` ADDED. Builds `awslambdaric/runtime_client.cpp` against SYSTEM
  libs: `dependency('CURL', modules: ['CURL::libcurl'])` and
  `dependency('aws-lambda-runtime', method: 'cmake', modules:
  ['AWS::aws-lambda-runtime'])`. Those come from protomolecule `native/curl` and
  `native/aws_lambda_cpp` (itself the sister fork `eugo-inc/aws-lambda-cpp`,
  built shared). Version is sed-extracted from `awslambdaric/__init__.py`.
- The extension installs into the `awslambdaric/` subdir, NOT site-packages root
  (upstream's location; collision-prone name). Upstream code does a top-level
  `import runtime_client` with a graceful ImportError fallback in
  `awslambdaric/lambda_runtime_client.py` - if upstream ever changes that import
  or drops the fallback, revisit the subdir install.
- `pyproject.toml` REWRITTEN for the `mesonpy` build backend, `dynamic =
  ["version"]`, runtime dep `simplejson`.
- `deps/` vendored tarballs are kept but UNUSED - accept upstream's version of
  anything under `deps/` without thought.
- `.devcontainer/`, `.mcp.json`: eugo dev tooling, no upstream counterpart.

## Merge recipe

1. Branch off eugo-main: `<user>/feat/MM-DD-YY-merge-upstream`.
2. `git remote add upstream https://github.com/aws/aws-lambda-python-runtime-interface-client.git`
   then `git fetch upstream`.
3. `git merge upstream/main` - MERGE, never rebase (rebase rewrites fork history
   and breaks the commit pin and future merge bases).
4. Conflict rules:
   - `setup.py`: keep deleted (ours). Port anything build-relevant upstream
     added there (new sources, new ext modules) into `meson.build`.
   - `pyproject.toml`: ours is authoritative for `[build-system]` (mesonpy) and
     dynamic version; take upstream changes to runtime `dependencies` (e.g. new
     imports like snapshot_restore_py) and classifiers.
   - `awslambdaric/**.py`, `tests/`, `requirements/`: take upstream; we do not
     patch Python sources.
   - After resolving: check upstream did not add new C++ sources or link deps
     for `runtime_client` (compare their setup.py/Makefile churn) and that the
     `__version__` line format still matches meson.build's sed.
5. Smoke: `pip wheel . --no-build-isolation` in an env with meson, meson-python,
   libcurl, and aws-lambda-runtime discoverable (in practice: the eugo container).
6. PR to `eugo-main`. Land with a MERGE COMMIT only - never squash; squashing
   destroys the upstream history that future syncs merge against.

## Post-merge adoption (protomolecule)

Bump `protomolecule/dependencies/python/wave_4/awslambdaric/meta.json`
`version.commit` to the new `eugo-main` head. If upstream added runtime deps, mirror
them in `meta.json` `dependencies.runtime` (it already lists `python/simplejson`
and `python/snapshot_restore_py`). Rebuild the package via its `setup` script
flow and re-test a lambda base image boot if entrypoint behavior changed.

## Push before pin

protomolecule pins this repo by commit sha. The sha you pin MUST already be
pushed to `github.com/eugo-inc/aws-lambda-python-runtime-interface-client`
`eugo-main` - never pin a local-only sha, and never force-push/rewrite `eugo-main`, or
every consumer build breaks on fetch.

## Related skills

`eugo-build-and-test` (container build + wheel smoke), `eugo-rebuild` (what a
diff requires + pin-bump rules), `eugo-meson-build-review` (pre-commit
meson.build / pyproject.toml checklist).
