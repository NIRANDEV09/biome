# Biome issue #10784 — final validation report

## A. Integrated implementation

Validated commit: `dd6f59781d1f1120197638b349358cac8131c838` on branch `validation`, based on `origin/fix/10784-parameters-overload`.

In `crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs`, `Rule::diagnostic()` returns the lint diagnostic independently. At the start of `Rule::action()`, the code checks `parameters_to_merge` for `MergeParameterOperation::MakeOptional` and returns `None` before creating a mutation. This prevents that unsafe code action. When no `MakeOptional` operation is present, the existing action path remains available.

## B. Regression coverage

The issue fixture is `crates/biome_js_analyze/tests/specs/style/useUnifiedTypeSignatures/issue_10784.ts`; its snapshot is `issue_10784.ts.snap`.

The four fixture scenarios are:

1. Original issue: zero-argument and one-required-argument overloads; the fixture includes the `Parameters<Api["run"]>` tuple assignment.
2. One versus two required parameters.
3. Identical duplicate signatures.
4. An overload where the longer signature already has an optional parameter.

The issue snapshot shows diagnostics without a fix for scenarios 1 and 2, and fixable diagnostics with overload removal for scenarios 3 and 4. The existing `invalid.ts.snap` diff removes the prior “make parameters optional” fixes while retaining those diagnostics. I inspected the checked-in snapshots, but could not rerun the snapshot test in this environment; see section D.

## C. TypeScript verification

TypeScript version: `7.0.2` (`pnpm dlx --package typescript tsc --version`). The two reproduction files were temporary files under `work/biome-10784-validation-ts`, outside the repository.

- `pnpm dlx --package typescript tsc --noEmit original.ts` — passed with no output. The original overload tuple is assignable to `[string]`.
- `pnpm dlx --package typescript tsc --noEmit unsafe-optional.ts` — failed as expected:

  ```text
  unsafe-optional.ts(6,7): error TS2322: Type '[input?: string | undefined]' is not assignable to type '[string]'.
    Source provides no match for required element at position 0 in target.
  ```

The fork’s Biome CLI was not available, and Cargo is unavailable, so I could not independently run `biome --write --unsafe` on the reproduction. The expected diagnostic and code-action behavior is supported by source and snapshot inspection, not by a live CLI run.

## D. Biome test results

Repository guidance recommends `just test-lintrule <ruleName>` for a focused lint rule test. The following focused attempts could not start:

- `just test-lintrule useUnifiedTypeSignatures` — failed because `just` is not installed (`just` is not recognized as a command).
- `cargo test -p biome_js_analyze --test spec_tests issue_10784` — could not start because `cargo` is not installed (`cargo` is not recognized as a command).
- `cargo test -p biome_js_analyze` — same environment blocker.
- `cargo test --workspace` — same environment blocker.

No broader analyzer or style test suite was run. I did not update or regenerate snapshots.

## E. Quality checks

The repository’s `CONTRIBUTING.md` recommends `just f` and `just l` before committing. Both were attempted and could not start because `just` is not installed. No Rust source, test fixture, or snapshot was modified by Member 4. The new Markdown files follow the repository’s existing changeset frontmatter format and were reviewed manually.

Environment limitation: this Windows checkout uses `core.symlinks=false` because symlink creation is not permitted here. Symlink-dependent test behavior was not verified.

## F. Final readiness checklist

- [x] Member 1's production implementation is present; the `MakeOptional` guard is before mutation creation.
- [ ] Member 2's `README-R2-12.md` is preserved. That exact filename is absent from this branch. The branch’s root `README.md` contains the analysis write-up; the user authorized proceeding without gating on Member 2.
- [x] Member 3's issue-specific regression fixture is present.
- [ ] Member 3's snapshots are correct. Their contents were inspected, but the focused test could not be run to compare generated output.
- [x] The original unsafe code action is suppressed by the action guard and the issue snapshot shows no fix for the two MakeOptional scenarios.
- [x] The lint diagnostic remains in the separate diagnostic path and in the issue snapshot.
- [x] Existing safe code actions remain available when `MakeOptional` is absent; the snapshot shows fixable duplicate-removal cases.
- [x] The TypeScript original and unsafe transformed examples were checked with `tsc --noEmit`.
- [ ] Focused Biome tests pass. The test command could not run because `just` and `cargo` are unavailable.
- [x] The changeset is present.
- [x] No unrelated production files were modified by Member 4. The integrated branch’s production change is limited to the issue’s lint rule.

## Commands run and results

- `git fetch origin` — fetched the updated issue branch at `dd6f59781d`.
- `git switch -c validation origin/fix/10784-parameters-overload` — created `validation` from the fetched issue branch.
- `git status --short` before Member 4 edits — clean (no output).
- `git rev-parse HEAD` — `dd6f59781d1f1120197638b349358cac8131c838`.
- `git diff --name-only origin/main...HEAD` — `README.md`, the issue lint implementation, `invalid.ts.snap`, `issue_10784.ts`, and `issue_10784.ts.snap`.
- `pnpm dlx --package typescript tsc --version` — `Version 7.0.2`.
- `pnpm dlx --package typescript tsc --noEmit original.ts` — passed, no output.
- `pnpm dlx --package typescript tsc --noEmit unsafe-optional.ts` — expected TS2322 failure shown in section C.
- `just test-lintrule useUnifiedTypeSignatures` — unavailable: `just` not recognized.
- `cargo test -p biome_js_analyze --test spec_tests issue_10784` — unavailable: `cargo` not recognized.
- `just f` — unavailable: `just` not recognized.
- `just l` — unavailable: `just` not recognized.
