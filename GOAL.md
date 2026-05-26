# ty nondeterminism investigation

Goal: find distinct sources of nondeterministic behavior in `ty`, fix each confirmed issue, and open one focused PR in `jelle-openai/ruff` per issue.

## Working rules

- Prefer a small reproducer before changing code.
- Treat output ordering, inference results, diagnostics, snapshots, and incremental behavior as possible nondeterminism surfaces.
- Keep each PR focused on one root cause with a regression test.
- Put the exact nondeterministic behavior and the root cause in each PR body, not just the symptom or chosen fix.
- Update this file with confirmed patterns, useful commands, and ruled-out leads as the investigation progresses.

## Upstream transfer workflow

Only run this after the user gives an explicit OK to transfer a specific PR to `astral-sh/ruff`.

1. Re-read the focused `jelle-openai/ruff` PR and nearby ty history or PR discussion for similar fixes, then adjust the change if needed so the implementation and regression test match existing ty style.
1. Start from the latest `upstream/main`, rebase or recreate the focused branch there, and verify the diff contains only the issue-specific change intended for the main repo.
1. Remove investigation-only or fork-only changes before transfer, including `GOAL.md` and any notes that belong only in `jelle-openai/ruff`.
1. Run the focused regression, the relevant crate test suite, `cargo clippy --workspace --all-targets --all-features -- -D warnings`, `uvx prek run --files ...`, and any extra validation appropriate for the touched paths.
1. Push the cleaned branch to the fork, open a draft PR against `astral-sh/ruff` with a concise root-cause description and `ty` label, then verify GitHub CI passes before calling the transfer done.

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
- Exact nondeterminism: the same `dd-trace-py` source sometimes inferred `SubprocessCmdLine.arguments` as `list[Unknown | str]` and sometimes as `list[_T@deque | str]`, changing the emitted diagnostic text. Parallel file checking changed which query first entered the shared implicit-attribute/full-scope-collection Salsa cycle.
- Root cause: a later collection use could feed a typevar owned by an inner generic call such as `collections.deque(...)` back into full-scope collection inference. That out-of-scope typevar was not meaningful in the collection literal's enclosing generic scope, but whether it escaped or collapsed to `Unknown` depended on the cycle entrypoint.
- Fix direction for the first PR: replace typevars not bound by an enclosing generic context with `Unknown` before using later-use constraints for full-scope collection literals; keep in-scope generic typevars intact.
- Published the first confirmed fix as `jelle-openai/ruff#3`.
- Removed `dd-trace-py` from the flaky primer list in that PR after eight final parallel concise checks produced the same diagnostic hash.
- The `scikit-build-core` flake was not semantic inference: missing runs indexed 173 files while present runs indexed 185, omitting tracked `src/scikit_build_core/build/*.py` files before diagnostics could be checked.
- Exact nondeterminism: `ty check ... src tests noxfile.py` sometimes included source `build` packages and emitted 77 diagnostics, and sometimes excluded those same tracked files and emitted 75 diagnostics. The omitted files contained the two disappearing diagnostics.
- Root cause: the project index passed `src`, `tests`, and `noxfile.py` into one parallel multi-root `ignore` walk. With `.gitignore` containing `tests/**/build/`, the `tests`-scoped ignore rule could sometimes be applied while visiting the source `src/scikit_build_core/build/` package, depending on worker scheduling.
- Dependency follow-up: the same omission reproduces with `rg --threads 12 --files src tests noxfile.py`, so the strongest root appears to be in ripgrep's `ignore` crate rather than ty. `Ignore::add_parents` caches compiled parent matchers only by parent directory even though each cached matcher stores an `absolute_base` derived from the current walk root; whichever parallel root populates that shared cache first can lend its base path to the other root.
- Fix direction for the second PR: keep the Ruff-side workaround focused by walking OS directory roots independently, and separately pursue the underlying `ignore` fix by scoping compiled parent matcher cache entries by both parent directory and walk root.
- Published the second confirmed fix as `jelle-openai/ruff#4`.
- Removed `scikit-build-core` from the flaky primer list in that PR after 100 final parallel concise checks produced the same diagnostic hash.
- Implemented the underlying ripgrep fix locally in `/Users/jelle/code/ripgrep` on branch `jelle/codex/ignore-multiroot-determinism`: scope compiled parent matcher cache entries by both parent directory and walk root. The focused new `ignore` regression failed before the fix and passed after it, and patched `rg --threads 12 --files src tests noxfile.py` kept the source `build` file in 100/100 runs.
- Re-ran the current primer list with eight flaky runs on unpatched current `main`; only the already-confirmed `dd-trace-py` and `scikit-build-core` cases reproduced there. Among formerly flaky projects, `bokeh`, `cloud-init`, `colour`, `ibis`, `jax`, `meson`, `pandas-stubs`, `PyGithub`, `rotki`, `sockeye`, and `vision` were stable for eight prepared-project runs, and `pydantic` was stable for 50 focused runs. `core` reproduced the diagnostic output-order root cause described below rather than a new semantic root cause.
- A generated two-file probe first exposed a deterministic panic whose raw text contains Salsa intern IDs; those IDs vary across parallel runs and need to be treated as a separate candidate instead of mistaken for semantic nondeterminism.
- After normalizing the Salsa IDs, the same generated seed found a distinct output-order flake: parallel runs emitted the same two fatal panic diagnostics for `mod.py` and `main.py`, but the two lines swapped order depending on worker completion order.
- A current Home Assistant `core` primer repro showed the same root cause without panics: repeated parallel full-project checks swapped two `invalid-argument-type` diagnostics at `tests/test_config_entries.py:5262:50` whose expected types were `Mapping[str, Any] | UndefinedType` and `MappingProxyType[str, tuple[DiscoveryKey, ...]] | UndefinedType`.
- Exact nondeterminism: diagnostics that share the same rendered source location, severity, and `DiagnosticId` could be printed in arrival order. The generated panic diagnostics had no span and tied on fatal `DiagnosticId::Panic`; the `core` diagnostics had the same primary span and tied on error `DiagnosticId::InvalidArgumentType`, so repeated parallel runs preserved worker-completion order for visibly different lines.
- Root cause: `CollectReporter` receives checked-file diagnostics from Rayon workers and sorts them with `Diagnostic::rendering_sort_key`; the existing key compared primary file/range, severity, and ID but not the visible diagnostic message, so same-location same-ID diagnostics compared equal and Rust's stable sort preserved the race-dependent collection order.
- Fix direction for the third PR: keep the existing file/range, severity, and diagnostic-ID ordering, then use the concise rendered message as the final visible tie-breaker so same-key diagnostics sort canonically.
- Published the third confirmed fix as `jelle-openai/ruff#5`.
- Exact nondeterminism for the separate Salsa payload lead: the same generated panic payload sometimes printed `[salsa id]: Id(2896)` and sometimes different IDs such as `Id(2c96)` or `Id(2496)` across equivalent parallel runs, even before considering the separate fatal-diagnostic line order.
- Root cause: `ruff_db::panic::Payload` formatted string panic payloads verbatim, while Salsa's generated `Debug` output includes intern IDs assigned according to query interning order. Those IDs are internal and race-dependent, so exposing them made user-facing panic diagnostics vary.
- Fix direction for the fourth PR: normalize Salsa IDs at the panic-output boundary by preserving the payload structure but replacing the volatile `Id(...)` contents with `_`.
- Published the fourth confirmed fix as `jelle-openai/ruff#6`.
