# ty nondeterminism investigation

Goal: find distinct sources of nondeterministic behavior in `ty`, fix each confirmed issue, and open one focused PR in `jelle-openai/ruff` per issue.

## Working rules

- Prefer a small reproducer before changing code.
- Treat output ordering, inference results, diagnostics, snapshots, and incremental behavior as possible nondeterminism surfaces.
- Keep each PR focused on one root cause with a regression test.
- Update this file with confirmed patterns, useful commands, and ruled-out leads as the investigation progresses.

## Initial leads

- Recent history contains ty fixes for deterministic type, constraint, overload, and synthesized-field ordering.
- Search `crates/ty*` for iteration over hash-based collections that can influence user-visible output or semantic tie-breaking.
- Check GitHub PR discussion around prior nondeterminism fixes for nearby unfinished cases.

## Investigation notes

- Prepared the current `scikit-build-core` primer project under `/private/tmp/scikit-build-core-py311-nondet` while following upstream issue `astral-sh/ty#1389`.
- Reproduced a current `scikit-build-core` flake with the primer's Python 3.11 environment: `TY_MAX_PARALLELISM=8 ty check ... src tests noxfile.py` alternated between 75 and 77 diagnostics, while `TY_MAX_PARALLELISM=1` was stable at 77 diagnostics.
- Existing inference tests use a useful perturbation pattern: type-check an unrelated file first, then inspect the target file after Salsa has cached and interned a different path.
- Confirmed a current `dd-trace-py` primer flake after `95eec58af2`: twelve `TY_MAX_PARALLELISM=8` full-project runs on unpatched `main` produced two concise diagnostic hashes, and the differing diagnostic came from `SubprocessCmdLine.arguments`.
- Minimized that flake to a full-scope `new_args = []` collection inside an implicit instance attribute cycle. Priming `new_args` before reading `SubprocessCmdLine("").arguments` changed the inferred `arguments` type, so query entrypoint was affecting the fixed point.
- Root cause: a later collection use could feed a typevar owned by an inner generic call such as `collections.deque(...)` back into full-scope collection inference. That out-of-scope typevar sometimes escaped as `_T@deque` and sometimes became `Unknown`, depending on which query headed the shared cycle.
- Fix direction for the first PR: replace typevars not bound by an enclosing generic context with `Unknown` before using later-use constraints for full-scope collection literals; keep in-scope generic typevars intact.
- Published the first confirmed fix as `jelle-openai/ruff#3`.
- Removed `dd-trace-py` from the flaky primer list in that PR after eight final parallel concise checks produced the same diagnostic hash.
- The `scikit-build-core` flake was not semantic inference: missing runs indexed 173 files while present runs indexed 185, omitting tracked `src/scikit_build_core/build/*.py` files before diagnostics could be checked.
- The same omission reproduced below ty with parallel multi-root `rg --files src tests noxfile.py`; a minimal `.gitignore` containing `tests/**/build/` sometimes hid a source `build` package during one combined `ignore` walk.
- Fix direction for the second PR: walk OS directory roots independently so ignore matching cannot depend on worker scheduling across roots.
- Published the second confirmed fix as `jelle-openai/ruff#4`.
- Removed `scikit-build-core` from the flaky primer list in that PR after 100 final parallel concise checks produced the same diagnostic hash.
