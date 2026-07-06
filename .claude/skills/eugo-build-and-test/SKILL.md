---
name: eugo-build-and-test
description: Use when building, installing, or smoke-testing the eugo fork of awslambdaric (aws-lambda-python-runtime-interface-client) - the meson/mesonpy build against system libcurl + libaws-lambda-runtime, the protomolecule consumer flow, and what a healthy wheel looks like. Activates on "build awslambdaric", "build the lambda RIC", "test awslambdaric", "runtime_client won't import", "Dependency CURL not found", "awslambdaric wheel", "smoke test lambda runtime client".
---

# Build and test the eugo awslambdaric fork

## What this fork builds

Upstream builds the `runtime_client` C++ extension via `setup.py`, compiling
vendored tarballs from `deps/` (curl-7.83.1, aws-lambda-cpp-0.2.6). The eugo
fork DELETED setup.py and builds with meson + the `mesonpy` backend
(`pyproject.toml [build-system]`) against SYSTEM libraries:

- `dependency('CURL', modules: ['CURL::libcurl'])` -> protomolecule `native/curl`
- `dependency('aws-lambda-runtime', method: 'cmake', modules:
  ['AWS::aws-lambda-runtime'])` -> protomolecule `native/aws_lambda_cpp`
  (sister fork `eugo-inc/aws-lambda-cpp`, built with `-DBUILD_SHARED_LIBS=ON`,
  so `runtime_client.so` carries DT_NEEDED on it and on libcurl at runtime)

`deps/` tarballs are kept for merge hygiene but UNUSED. Project version is
sed-extracted from `__version__` in `awslambdaric/__init__.py` (see meson.build).

## How protomolecule builds it (ground truth)

`protomolecule/dependencies/python/wave_4/awslambdaric/` - `meta.json` pins this
repo by git commit on branch `eugo-main`; `setup` is a single pip install:

```bash
pip3 install "${EUGO_PACKAGE_NAME} @ ${EUGO_PACKAGE_VERSION}" \
  ${EUGO_PIP_COMPILABLE_PACKAGE_OPTIONS} ${EUGO_PIP_TARGET_FLAG} ${EUGO_MESONPY_COMMON_OPTIONS}
```

The env comes from `protomolecule/scripts/ben_life_easy.sh`:
- `EUGO_PIP_COMPILABLE_PACKAGE_OPTIONS`: `-v -v -v --compile --no-deps
  --no-cache-dir --no-build-isolation --no-binary :all:`
- `EUGO_MESONPY_COMMON_OPTIONS`: `-Csetup-args=` for `c_std=gnu17`,
  `cpp_std=gnu++23`, `buildtype=release`, `cmake_prefix_path`,
  `pkg_config_path`, `python.bytecompile=2`, `wrap_mode=nofallback`, plus
  `-Cbuild-dir=eugo_build`
- `EUGO_PIP_TARGET_FLAG`: `-t ${EUGO_INSTALL_PREFIX_PATH}` (set by the harness)

Buildtime deps MUST be declared in `meta.json` (`native/curl`,
`native/aws_lambda_cpp`); undeclared, the DAG never builds them and meson dies
with `Dependency "CURL" not found (tried pkg-config)`. Runtime deps:
`python/simplejson`, `python/snapshot_restore_py` (imported by
`lambda_runtime_hooks_runner.py`).

## Local build (eugo container)

```bash
# Cheap configure gate (catches missing system deps, bad meson syntax):
meson setup eugo_build -Dcpp_std=gnu++23 -Dbuildtype=release \
  "-Dcmake_prefix_path=${EUGO_CMAKE_PREFIX_PATH}"
# Full wheel (or source ben_life_easy.sh and use $EUGO_MESONPY_COMMON_OPTIONS):
pip3 wheel . --no-build-isolation --no-deps --no-binary :all: \
  -Cbuild-dir=eugo_build -Csetup-args=-Dcpp_std=gnu++23 \
  -Csetup-args=-Dbuildtype=release \
  "-Csetup-args=-Dcmake_prefix_path=${EUGO_CMAKE_PREFIX_PATH}"
```

## What healthy looks like

- Wheel `awslambdaric-<ver>-...-linux_<arch>.whl`; `<ver>` matches
  `__version__` in `awslambdaric/__init__.py` (dynamic; never hardcoded).
- Wheel contains `awslambdaric/*.py` AND `awslambdaric/runtime_client.cpython-*.so`
  (SUBDIR, not site-packages root - eugo divergence from upstream layout).
- Smoke (run from /tmp so the repo dir does not shadow the install):
  `python3 -c "import awslambdaric, awslambdaric.runtime_client"` - loading the
  .so proves the libcurl + libaws-lambda-runtime DT_NEEDED chain resolves.
- Upstream pure-Python unit tests still pass:
  `python3 -m pytest --cov awslambdaric --cov-fail-under 90 tests`
  (does not exercise the meson extension). NEVER `make test` /
  `make format` / `make build` - the Makefile targets still reference
  the deleted `setup.py` and fail; call pytest/black directly.

## Known traps

- `awslambdaric/lambda_runtime_client.py` does a TOP-LEVEL `import
  runtime_client` with a silent `except ImportError: runtime_client = None`
  fallback. With the eugo subdir install, a bare top-level import does NOT
  resolve; the invoke loop needs it non-None at Lambda runtime. When validating
  a rebuilt lambda base image, assert in-image that
  `awslambdaric.lambda_runtime_client.runtime_client is not None`.
- protomolecule's `tree.txt` snapshot is STALE (3.0.0, .so at site-packages
  root); the pinned commit builds 3.1.1 with the subdir layout.
- Runtime consumer: epstein-drive lambda base images (`eugo_lambda_entrypoint.sh`
  -> `exec python -m awslambdaric ...`; `aws-lambda-rie`-wrapped outside Lambda).

Related skills: `eugo-rebuild` (what a given diff requires),
`eugo-meson-build-review` (pre-commit meson.build checklist),
`eugo-upstream-merge` (sync recipe + protomolecule pin bump).
