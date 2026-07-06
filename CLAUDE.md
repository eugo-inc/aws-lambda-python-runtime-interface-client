# CLAUDE.md

Eugo fork (`eugo-inc/aws-lambda-python-runtime-interface-client`) of
`aws/aws-lambda-python-runtime-interface-client` -- the `awslambdaric` PyPI
package (Lambda Runtime Interface Client for Python). The fork swaps
upstream's setup.py build (which compiles vendored `deps/` tarballs) for
meson + `mesonpy` against SYSTEM libcurl and aws-lambda-runtime from the eugo
container. `setup.py` is deleted; `deps/` tarballs are kept for merge hygiene
but UNUSED. Sole build consumer: protomolecule
`dependencies/python/wave_4/awslambdaric/`, pinned by git commit -- landing on
the default branch changes nothing downstream until its `meta.json` pin bumps.

Deep content lives in `.claude/skills/`. Route by what you are about to do:

- About to build or smoke-test, or hit `Dependency "CURL" not found` /
  `runtime_client` import failure -> skill `eugo-build-and-test`.
- Diff in hand, deciding what to rebuild or whether protomolecule needs a
  pin bump -> skill `eugo-rebuild`.
- Touching `meson.build` or `pyproject.toml` -> run the skill
  `eugo-meson-build-review` checklist before committing, then the configure
  gate must pass (exit 0, eugo container):
  `meson setup eugo_build "-Dcmake_prefix_path=${EUGO_CMAKE_PREFIX_PATH}"`.
- About to merge upstream -> skill `eugo-upstream-merge`. MERGE, never
  rebase; land with a merge commit, never squash -> squashing destroys the
  upstream history future syncs merge against.

## Ground rules

- Default branch is `eugo-main` (`origin/HEAD`); protomolecule's meta.json
  pins this repo by commit on it. When any doc disagrees with git -> trust
  `git branch --show-current`.
- NO `@EUGO_CHANGE` annotations here -> `meson.build` is wholly eugo-authored
  and organized by `# === @begin/@end: <section> ===` blocks; keep new logic
  inside a paired block.
- We do not patch Python sources (`awslambdaric/**.py`, `tests/`,
  `requirements/`) -> in merge conflicts take upstream verbatim.
- Version is dynamic: `project()` sed-extracts `__version__` from
  `awslambdaric/__init__.py`. NEVER hardcode a version in `meson.build` ->
  keep `pyproject.toml` `dynamic = ["version"]` and the sed extraction.
- `runtime_client` installs INSIDE `awslambdaric/` (`subdir:`), not
  site-packages root -> upstream's root layout is collision-prone; keep the
  subdir install.

## Test / smoke

- Unit tests: `python3 -m pytest --cov awslambdaric --cov-fail-under 90 tests`
  (exit 0). NEVER run `make test` / `make format` / `make build` -> they
  invoke black / `python3 setup.py` on the deleted `setup.py` (upstream
  Makefile leftovers); call pytest/black on existing paths directly.
- Wheel smoke, from /tmp so the repo dir doesn't shadow the install:
  `python3 -c "import awslambdaric, awslambdaric.runtime_client"` -- exit 0
  proves the libcurl + libaws-lambda-runtime DT_NEEDED chain resolves.
- Build/test tooling lives only in the eugo container (`.devcontainer/`
  builds the arm64 eugo toolchain image). Tool missing on the host -> stop
  and ask for the container; don't pip-install substitutes.
