---
name: eugo-meson-build-review
description: Pre-commit review checklist for meson.build / pyproject.toml changes in the eugo awslambdaric fork - section-marker discipline, system-deps-only rule, dynamic version, install layout, and the protomolecule meta.json mirror. Activates on "review meson change", "check meson.build", "pre-commit build review", "validate pyproject", "audit the build files".
---

# Eugo meson build review (awslambdaric, pre-commit)

Run this checklist over any diff touching `meson.build` or `pyproject.toml`.
The build surface is tiny (one extension module, one install_subdir), so review
is exhaustive, not sampled.

## 1. Section markers

`meson.build` is organized by `# === @begin: <section> ===` /
`# === @end: <section> ===` blocks (General, Meson modules imports,
Dependencies, Compilable Python modules, Pure Python modules). This repo does
NOT use `@EUGO_CHANGE` annotations - the whole file is eugo-authored. Keep
every block paired and new logic inside an appropriately named block.

## 2. System deps only

- All native deps come from the eugo container via `dependency()`:
  `CURL` (cmake modules) and `aws-lambda-runtime` (`method: 'cmake'`). Never
  reintroduce upstream's vendored `deps/` tarballs or add meson
  subprojects/wraps - the harness passes `wrap_mode=nofallback` and a fallback
  would silently vendor.
- Any NEW `dependency()` must be mirrored into protomolecule
  `dependencies/python/wave_4/awslambdaric/meta.json`
  `dependencies.buildtime.standalone` in the same change window, or the
  dependency DAG never builds it (`Dependency "X" not found`).

## 3. Flags and version that must not regress

- `default_options`: `c_std=gnu17`, `cpp_std=gnu++23`, `buildtype=release`.
  The harness passes the same via `-Csetup-args=`; keep defaults aligned.
- `project()` version stays DYNAMIC: sed over `__version__` in
  `awslambdaric/__init__.py` with `check: true`. Never hardcode a version in
  meson.build; `pyproject.toml` keeps `dynamic = ["version"]`.
- `[build-system]` stays `mesonpy` with `meson` + `meson_python` requires.

## 4. Install layout (the load-bearing eugo divergence)

- `py.extension_module('runtime_client', ..., subdir: 'awslambdaric/')`:
  installs the .so INSIDE the package, not site-packages root (upstream's
  collision-prone location). Do NOT flip this back without reading
  `awslambdaric/lambda_runtime_client.py` - its top-level `import
  runtime_client` has a silent `ImportError -> None` fallback, and the invoke
  loop requires it non-None at Lambda runtime.
- `install_subdir('awslambdaric/', install_dir: py.get_install_dir() /
  'awslambdaric', ...)`: the TRAILING SLASH means contents-only; dropping it
  nests `awslambdaric/awslambdaric/`. Keep `exclude_files: ['runtime_client.cpp']`
  so the C++ source is not shipped in the wheel.

## 5. Python dependency sync

`pyproject.toml [project] dependencies` must cover every runtime import in
`awslambdaric/` (currently `simplejson`; `snapshot_restore_py` is imported by
`lambda_runtime_hooks_runner.py` but only declared in upstream
`requirements/base.txt` and protomolecule meta.json - a known gap; do not
widen it). After an upstream sync, diff `requirements/base.txt` against
`[project] dependencies` and against meta.json `dependencies.runtime`.

## 6. Post-upstream-sync extras

- Compare upstream's setup.py churn (they still have one; we deleted ours) for
  new sources, new link libs, or new ext modules to port into meson.build.
- Confirm the `__version__` line format still matches the sed pattern.
- Gate with `meson setup eugo_build ...` then a wheel build + smoke - commands
  in `eugo-build-and-test`. Merge recipe and pin bump: `eugo-upstream-merge`.
