# MachZero Open-Source Hackathon
## Biome Issue #10784 — Safe Overload Unification

**Integrated technical investigation, design decisions, implementation record, regression strategy, development environment, failures, and validation history**

> This document consolidates the team's Round 2 investigation, Round 3 implementation report, final validation summary, and the repository-state information shared during the development conversation. It distinguishes reported test results from independently verified facts and records the latest known Git history accurately.

---

## 1. Project overview

| Project detail | Information |
|---|---|
| Hackathon | MachZero Open-Source Hackathon |
| Upstream repository | [biomejs/biome](https://github.com/biomejs/biome) |
| Team fork | [NIRANDEV09/biome](https://github.com/NIRANDEV09/biome) |
| Upstream issue | [#10784 — `useUnifiedTypeSignatures` unsafe fix changes `Parameters<>` output and breaks TypeScript](https://github.com/biomejs/biome/issues/10784) |
| Rule | `lint/style/useUnifiedTypeSignatures` |
| Main Rust package | `biome_js_analyze` |
| Main implementation file | `crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs` |
| Implementation language | Rust |
| Related language/tooling | TypeScript, Biome CLI, Cargo, rustfmt, Clippy |
| Feature branch | `fix/10784-parameters-overload` |
| Team size | Four members |
| Development stages | Round 2 investigation and Round 3 implementation |
| Reported investigation dates | 7–8 November 2026 |

## 2. Executive summary

Biome's `useUnifiedTypeSignatures` lint rule identifies similar TypeScript overload signatures that may be consolidated. The reported bug occurs when an automatic fix changes a required parameter into an optional parameter. Although the resulting signature may look like a reasonable simplification, it can change observable TypeScript type information.

The central example uses TypeScript's `Parameters<T>` utility type. For an overloaded function, the utility extracts the parameters of the final overload signature. Changing that final signature from a required parameter to an optional parameter changes the extracted tuple. Existing code expecting a required tuple can then fail with TypeScript error `TS2322`.

The selected implementation strategy is deliberately narrow:

- Keep the lint diagnostic.
- Suppress the automatic code action only when the proposed merge contains `MergeParameterOperation::MakeOptional`.
- Preserve other code actions when no parameter needs to be made optional.
- Avoid disabling the rule, adding a new configuration option, or broadly rejecting all overloads with different parameter counts.

The production guard was reported as implemented in `Rule::action()` in the rule's Rust source file. Regression fixtures and a changeset were also prepared during the team's work. The validation section below carefully separates test results reported in the supplied team report from results independently confirmed in this document's preparation.

---

## 3. Understanding the bug

### 3.1 Original TypeScript example

```ts
interface Api {
    run(): void;
    run(input: string): void;
}

type RunArgs = Parameters<Api["run"]>;

const args = ["value"] as RunArgs;
const requiredArgs: [string] = args;
```

TypeScript's `Parameters<T>` utility extracts a function's parameter types as a tuple. With overloaded functions, the final overload signature is used for this inference.

In the example, the final overload is:

```ts
run(input: string): void;
```

The extracted tuple therefore has one required string element:

```ts
[input: string]
```

The assignment to `[string]` is compatible.

### 3.2 The unsafe unified form

The unsafe transformation under investigation consolidates the overloads into a signature like this:

```ts
interface Api {
    run(input?: string): void;
}
```

The extracted tuple is now equivalent to:

```ts
[input?: string | undefined]
```

An optional tuple element may be absent. It is therefore not assignable to a tuple that requires an element at index zero:

```ts
const requiredArgs: [string] = args;
```

TypeScript reports `TS2322` because the source tuple may have no first element.

The change can also affect call-site checking: a required parameter becomes optional, so calls that previously needed an argument may become permitted.

### 3.3 Why this is a semantic correctness issue

Two declarations can appear similar when considering a subset of ordinary calls but still expose different type-level behavior. TypeScript utility types, tuple assignments, overload inference, and API consumers can observe that difference.

The problem is therefore not simply formatting or style. An automatic fix must not silently change relevant type behavior.

---

## 4. Round 2 — Technical investigation and solution planning

### 4.1 Investigation objectives

The team set out to answer four questions:

1. What TypeScript behavior is changing?
2. Which part of Biome's rule creates the transformation?
3. Why does the transformation change type-level behavior?
4. How can the unsafe edit be prevented without removing useful diagnostics and safe fixes?

The team's stated priority was to understand the code path and establish a justified solution before changing production code.

### 4.2 Source-code path investigated

The primary source file was identified as:

`crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs`

The investigation examined these components:

| Component | Role in the rule |
|---|---|
| `Rule::run()` | Finds candidate overloads |
| `try_merge_overloads()` | Checks whether overloads can be considered for merging |
| `try_merge()` | Compares parameters and creates merge instructions |
| `MergeOverloadSignaturesInfo` | Carries information about the proposed merge |
| `MergeParameterOperation` | Describes parameter-level edits |
| `MergeParameterOperation::MakeOptional` | Represents making a parameter optional |
| `Rule::diagnostic()` | Reports the lint finding |
| `Rule::action()` | Builds the code action and syntax-tree mutation |
| `mutation.replace_node()` | Participates in applying syntax-tree changes |

The conceptual flow is:

```text
Rule::run()
    |
    v
try_merge_overloads()
    |
    v
try_merge()
    |
    v
Generate merge instructions
    |
    v
Rule::diagnostic()          Rule::action()
    |                            |
    v                            v
Report lint finding        Inspect/apply proposed edits
```

### 4.3 Critical investigation point: `MakeOptional`

The team identified `MergeParameterOperation::MakeOptional` as the key operation to investigate because it represents a parameter becoming optional.

The investigation discussed parameter eligibility logic resembling:

```rust
if !self_param.can_be_optional() {
    return None;
}
```

The exact path and conditions should always be checked against the source revision being tested. The key design observation was the separation between:

- determining and reporting a merge opportunity; and
- offering and applying an automatic source-code edit.

This separation makes it possible to retain the diagnostic while withholding a particular unsafe code action.

### 4.4 Reproduction workflow

The Round 2 notes referenced the external reproduction project:

[biome-use-unified-type-signatures-repro](https://github.com/jakeleventhal/biome-use-unified-type-signatures-repro)

The reported environment included Biome `2.5.1`, TypeScript `5.9.2`, and Bun. The described workflow was:

```bash
bun install
bunx tsc --noEmit
bunx biome check --write --unsafe .
bunx tsc --noEmit
```

The purpose is to type-check the original source, apply Biome's unsafe fix, and type-check the transformed source. The supplied reproduction notes describe the initial TypeScript check passing and the post-transformation check producing `TS2322`.

These are reproduction details from the team's supplied notes, not a claim that this document's author reran that external project during document preparation.

An important distinction from the investigation was:

- **TypeScript-level reproduction:** prove that required and optional parameter tuples are different.
- **Biome-level reproduction:** prove that Biome's actual unsafe fix produces the transformation.

Both are useful. The second is necessary to verify the precise rule behavior.

### 4.5 Proposed solution directions considered

The investigation considered multiple approaches.

#### Approach A — Conservative safeguard at the action stage

Inspect the generated parameter operations in `Rule::action()`. If any operation is `MakeOptional`, do not offer the code action.

Advantages:
- Keeps the lint diagnostic.
- Prevents the identified automatic edit.
- Keeps other merge actions available when they do not require `MakeOptional`.
- Does not add a configuration option.
- Makes a localized change at the point where the edit is offered.

Trade-off:
- It suppresses the whole action for a proposed merge if any parameter operation requires `MakeOptional`, even if other parts of that same proposed action might otherwise be safe. This is an intentionally conservative choice.

**Selected approach.**

#### Approach B — Reject the merge earlier in `try_merge()`

This would reject the candidate before the diagnostic/action stages.

Concern:
- It could remove the diagnostic as well as the unsafe fix.
- It changes the rule's reporting behavior rather than narrowly preventing the edit.

Not selected.

#### Approach C — Add a configuration option

This would allow users to choose whether the rule may make parameters optional.

Concern:
- Adds configuration surface for a narrowly defined safety issue.
- Requires additional option documentation and tests.
- The team did not consider a new option necessary for the selected conservative behavior.

Not selected.

#### Approach D — Strict type-equivalence or last-overload comparison

This would attempt a more sophisticated check of whether the merged signature preserves the relevant TypeScript behavior.

Potential advantage:
- Could preserve more automatic fixes if equivalence could be established robustly.

Concerns:
- TypeScript overload and type-level semantics are complex.
- A syntactic comparison alone may not establish semantic equivalence.
- It is a broader implementation than the targeted issue fix.

Not selected for this contribution.

### 4.6 Lessons from the investigation

1. Syntax-level similarity does not guarantee type-level equivalence.
2. Automatic fixes must consider observable type behavior.
3. Diagnostic generation and code-action generation are distinct responsibilities.
4. A narrow safeguard can be preferable to disabling an entire rule.
5. Reproducing a manually written transformed signature is not the same as proving Biome generated it.
6. Regression tests must cover the unsafe case and ensure legitimate transformations remain available.

---

## 5. Round 3 — Implementation and working prototype

### 5.1 Selected implementation

The production change was made in:

`crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs`

At the beginning of `Rule::action()`, the implementation checks the proposed parameter operations. If any operation is `MergeParameterOperation::MakeOptional`, the action returns `None`.

The reported guard is:

```rust
if state
    .parameters_to_merge
    .iter()
    .any(|info| matches!(info.operation, MergeParameterOperation::MakeOptional))
{
    return None;
}
```

The surrounding method has the following conceptual shape:

```rust
fn action(
    ctx: &RuleContext<Self>,
    state: &Self::State,
) -> Option<RuleAction<JsLanguage>> {
    if state
        .parameters_to_merge
        .iter()
        .any(|info| matches!(info.operation, MergeParameterOperation::MakeOptional))
    {
        return None;
    }

    let mut mutation = ctx.root().begin();

    // Existing mutation logic continues here.
}
```

This is an explanatory excerpt, not a replacement for the complete source method.

### 5.2 Behavior of the guard

1. `Rule::run()` identifies the overload pattern.
2. `Rule::diagnostic()` can still report the lint.
3. `Rule::action()` inspects the proposed parameter operations.
4. If an operation is `MakeOptional`, `Rule::action()` returns `None`.
5. If no `MakeOptional` operation exists, the existing action path continues.

The intended result is a visible diagnostic without the unsafe automatic fix for the identified transformation. Safe action paths remain available where the proposed merge does not require a parameter to become optional.

### 5.3 Why the action stage was selected

The guard is placed where the code action is created rather than in the earlier merge-discovery logic. This preserves the distinction between identifying a lint opportunity and deciding whether a specific automatic edit is safe to offer.

The implementation does not:
- disable `useUnifiedTypeSignatures`;
- remove the diagnostic for the affected overloads;
- add a new user configuration option;
- rewrite the overload-merging architecture; or
- claim general TypeScript semantic equivalence for every possible overload.

---

## 6. Scenario analysis and expected behavior

The team considered several overload patterns to ensure the safeguard did not unnecessarily suppress unrelated fixes.

| Case | Scenario | Example shape | Expected behavior under the selected design |
|---|---|---|---|
| A | Different required parameter counts | `send(msg: string)` and `send(msg: string, priority: number)` | Suppress the action if the proposed merge contains `MakeOptional` |
| B | Different parameter types | `process(value: string)` and `process(value: number)` | Preserve a merge if it combines types without `MakeOptional` |
| C | Different return types | Overloads with incompatible return types | Existing compatibility logic is expected to govern whether a merge is possible; verify against current source/tests |
| D | Identical duplicate signatures | Two identical `calc(x: number): number` declarations | Preserve safe duplicate removal if no `MakeOptional` operation is generated |
| E | Longer overload already has an optional extra parameter | `update(id: number)` and `update(id: number, name?: string)` | Preserve the action if no required parameter needs conversion to optional |

The implementation checks the generated operation, not merely whether two overloads have different parameter counts. This is intended to avoid broad suppression based only on signature length.

**Important scope note:** These rows describe the team's scenario analysis and intended behavior. They are not a formal proof of all TypeScript overload equivalence cases. Each behavior should be confirmed by the rule's fixtures and current source revision.

---

## 7. Regression tests and test coverage

### 7.1 Issue-specific fixture files

The supplied team reports identify the following paths:

- `crates/biome_js_analyze/tests/specs/style/useUnifiedTypeSignatures/issue_10784.ts`
- `crates/biome_js_analyze/tests/specs/style/useUnifiedTypeSignatures/issue_10784.ts.snap`
- `crates/biome_js_analyze/tests/specs/style/useUnifiedTypeSignatures/invalid.ts.snap` (an update was reported in a later test-integration commit)

The issue fixture is reported to cover:
1. Zero-argument versus one-required-argument overloads.
2. One-required-argument versus two-required-argument overloads.
3. Identical duplicate signatures.
4. An overload with an already-optional extra parameter.

The expected outcomes are:
- diagnostics remain for the relevant overload patterns;
- the unsafe action is absent when `MakeOptional` is required; and
- safe duplicate or redundant-signature actions remain available where no `MakeOptional` operation is generated.

### 7.2 Existing rule coverage

The supplied reports also identify existing fixture coverage for:
- standalone function overloads;
- call signatures in type literals and interfaces;
- class constructor overloads; and
- overloads with differing parameter types.

### 7.3 Snapshot validation

The final validation summary supplied by the team says the snapshots matched generated spec-test output. This document records that as a reported result, not as a fresh snapshot-generation run.

---

## 8. Development environment and system details

This section separates known workstation details from information that was not supplied. Hardware specifications must not be invented for a judge-facing report.

### 8.1 Workstation details available from the development conversation

| Item | Known information |
|---|---|
| Operating system family | Windows |
| Shell used | Windows PowerShell |
| Repository location | `C:\Users\niroo\OneDrive\Documents\ENGINEERING\hackathon\machzero\biome` |
| Repository | Local clone of the team's Biome fork |
| Git remote `origin` | `https://github.com/NIRANDEV09/biome.git` |
| Working branch | `fix/10784-parameters-overload` |
| Rust/Cargo availability at latest troubleshooting checkpoint | `rustc` and `cargo` were not recognized by PowerShell |
| Rust installation status | Not confirmed as installed; installation was suggested |
| Windows linker flag discussed | `-C link-arg=/STACK:16777216` |
| TypeScript versions mentioned in supplied reports | `5.9.2` in Round 2 reproduction notes; `7.0.2` in the later final validation report |

### 8.2 Hardware details not available

The conversation does not establish the following workstation specifications:

- Exact Windows edition/build.
- Laptop or desktop manufacturer and model.
- CPU model or core count.
- Installed RAM.
- GPU model.
- Storage type/capacity.
- Rust, Cargo, MSVC, Visual Studio Build Tools, Node.js, npm, Bun, or `just` exact versions.

These details should be filled in from the actual development machine before submission if the hackathon requires a complete hardware inventory. Do not claim a specific CPU, RAM size, or operating-system edition without checking the machine.

Useful PowerShell commands for collecting the missing details:

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber, CsSystemType
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model,
    @{Name="RAM_GB";Expression={[math]::Round($_.TotalPhysicalMemory / 1GB, 2)}}
Get-CimInstance Win32_VideoController | Select-Object Name
Get-Volume | Select-Object DriveLetter, FileSystem, Size, SizeRemaining
```

Toolchain checks:

```powershell
git --version
rustc --version
cargo --version
rustfmt --version
clippy-driver --version
node --version
npm --version
bun --version
just --version
```

A command that is not installed may return an error; record that rather than inventing a version.

### 8.3 Software and packages

| Tool/component | Purpose | Version recorded in supplied material |
|---|---|---|
| Rust compiler (`rustc`) | Compile Biome's Rust workspace | Not recorded |
| Cargo | Build and test Rust packages | Not recorded |
| `biome_js_analyze` | Analyzer package containing the rule and tests | Workspace package |
| `rustfmt` | Rust formatting validation | Not recorded |
| Clippy | Rust lint validation | Not recorded |
| `just` | Biome task runner | Not recorded |
| TypeScript (`tsc`) | Reproduce and validate type behavior | `5.9.2` in Round 2 notes; `7.0.2` in final report |
| Bun | Package/command runner in external reproduction notes | Not recorded |
| Node.js / npm | `npx tsc` workflow | Not recorded |
| Biome CLI | Run lint and unsafe-fix behavior | Workspace/local CLI; exact version not recorded for the final run |

The supplied material does not identify a new third-party dependency added by the fix. The implementation is reported to use the existing Biome Rust workspace.

---

## 9. Failures, challenges, and troubleshooting

### 9.1 Rust and Cargo unavailable in PowerShell

During later local testing, PowerShell returned errors equivalent to:

```text
rustc : The term 'rustc' is not recognized as the name of a cmdlet...
cargo : The term 'cargo' is not recognized as the name of a cmdlet...
```

This means the current PowerShell session could not locate the Rust compiler and Cargo. It does not prove that the source code is broken. Rust may be uninstalled or its Cargo bin directory may not be on `PATH`.

The suggested installation route was the official Rust installer:

https://rustup.rs/

After installation, PowerShell should be restarted and these commands checked:

```powershell
rustc --version
cargo --version
```

A PATH/install diagnostic used in the conversation was:

```powershell
Test-Path "$env:USERPROFILE\.cargo\bin\cargo.exe"
```

**Validation caveat:** Because the latest troubleshooting output showed both commands unavailable, a fresh local Rust test run on that workstation cannot be claimed based on that output. The final test outcomes elsewhere in this document remain attributed to the supplied team validation report.

### 9.2 Windows MSVC debug stack overflow

The Round 3 report describes a locally compiled debug `biome.exe` aborting with:

```text
thread 'main' has overflowed its stack
```

The report attributes this to the default thread stack size in that Windows MSVC debug setup and says increasing the linker stack resolved the issue in that environment.

The workaround recorded was:

```powershell
$env:RUSTFLAGS="-C link-arg=/STACK:16777216"
cargo build -p biome_cli --bin biome
```

This is a reported workaround for that particular setup, not a general hardware requirement or a guarantee that every Windows build needs this flag.

### 9.3 Nested Biome configuration conflict

The Round 3 report describes a configuration error when running Biome from nested directories:

```text
Found a nested root configuration, but there's already a root configuration.
```

The reported cause was Biome discovering multiple `biome.json` files in the directory hierarchy. The reported workaround was to pass an explicit configuration path for the isolated test:

```powershell
biome lint --config-path biome.json index.ts
```

Confirm the appropriate configuration path for the actual working directory before using this command.

### 9.4 Test integration and repository-history correction

The repository history contains a later test-integration commit and its revert. The team reverted the latest commit after identifying a problem with that version. This is recorded in Section 12 so that the project report does not imply the reverted changes are still present in the current branch.

### 9.5 Lessons from failures

- A missing `cargo` executable is an environment/toolchain issue until proven otherwise.
- A debug stack overflow is distinct from a Rust compilation error.
- Nested configuration discovery can affect local CLI experiments.
- A pushed commit can be safely undone with `git revert`, preserving public Git history.
- Validation claims must match the actual environment and the exact repository revision tested.

---

## 10. Validation plan and reported results

### 10.1 Commands listed by the team

Focused and broader analyzer tests:

```bash
cargo test -p biome_js_analyze use_unified_type_signatures
cargo test -p biome_js_analyze specs::style
cargo test -p biome_js_analyze
```

Biome task-runner command:

```bash
just test-lintrule useUnifiedTypeSignatures
```

Rust quality checks:

```bash
cargo clippy -p biome_js_analyze -- -D warnings
rustfmt --check crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs
```

Additional checks listed in the Round 3 report:

```bash
cargo test --doc -p biome_js_analyze
cargo test -p biome_js_analyze --test control_flow
cargo test -p biome_js_analyze --test quick_test
cargo test -p biome_rule_options
```

TypeScript checks mentioned in the reports include:

```bash
tsc --noEmit
tsc --project tsconfig.json
npx tsc --noEmit
```

The correct fixture path, working directory, and TypeScript configuration must be selected for each actual run.

### 10.2 Final validation results supplied by the team

The later final validation summary supplied to the assistant reported the following:

| Check | Command/target | Result in supplied final report |
|---|---|---|
| Focused lint-rule tests | `cargo test -p biome_js_analyze use_unified_type_signatures` | 11/11 passed |
| Style-category analyzer tests | `cargo test -p biome_js_analyze specs::style` | 795/795 passed |
| Clippy | `cargo clippy -p biome_js_analyze -- -D warnings` | Passed, 0 warnings |
| Rust formatting | `rustfmt --check` on the production file | Passed |
| TypeScript issue fixture | `tsc --noEmit` on `issue_10784.ts`, TypeScript 7.0.2 | 0 errors |
| Unsafe transformed TypeScript example | `tsc --noEmit` on the transformed example | Reproduced `TS2322` |

The reported `TS2322` reproduction is the tuple incompatibility: an optional tuple element is not assignable to a tuple requiring a string at position zero.

### 10.3 Additional results listed in the Round 3 report

The Round 3 prototype report also listed:

| Check | Command | Result reported in Round 3 material |
|---|---|---|
| Analyzer documentation tests | `cargo test --doc -p biome_js_analyze` | 10/10 passed |
| Control-flow tests | `cargo test -p biome_js_analyze --test control_flow` | 4/4 passed |
| Quick tests | `cargo test -p biome_js_analyze --test quick_test` | 2/2 passed |
| Rule options tests | `cargo test -p biome_rule_options` | 21/21 passed |
| Clippy | `cargo clippy -p biome_js_analyze -- -D warnings` | Clean, 0 warnings |
| Formatting | `rustfmt --check` | Clean, 0 diffs |
| TypeScript project check | `tsc --project tsconfig.json` | Exit code 0 |

The Round 3 report also describes a local CLI demonstration in which the diagnostic remained visible, no fix was applied with unsafe write, and TypeScript compilation succeeded after the safeguard.

### 10.4 Validation provenance and limitations

The results in Sections 10.2 and 10.3 are transcribed from the team's supplied final validation summary and Round 3 report. They were **not independently rerun while preparing this integrated README**.

The later troubleshooting output in the development conversation showed that `rustc` and `cargo` were not recognized by the user's PowerShell session. Therefore:

- Do not describe this README's preparation as a fresh successful build.
- Do not imply that the user's current Windows workstation has a working Rust toolchain.
- Before a live judge demonstration, install/verify Rust and rerun the commands against the exact current branch.
- Preserve actual terminal output or CI logs if the judges request evidence.

The source reports also contain different TypeScript versions for different activities: `5.9.2` in the Round 2 reproduction notes and `7.0.2` in the later final validation report. They are intentionally recorded separately.

---

## 11. Team structure and contributions

The team had four members, with work divided by focus area.

| Member | Responsibility | Reported contribution |
|---|---|---|
| Member 1 | Core implementation | Added the `MakeOptional` safeguard in `Rule::action()` |
| Member 2 | Investigation and documentation | Traced TypeScript behavior and the Biome code path; prepared Round 2 and Round 3 technical documentation |
| Member 3 | Regression testing | Prepared issue-specific fixtures/snapshots and worked on existing expected snapshots |
| Member 4 | Release preparation and validation documentation | Prepared changeset and validation documentation; contributed validation notes |

### Contribution and commit references in supplied reports

- Member 1 production commit: `13b6da3a6c` (reported commit message: `fix(biome): prevent unsafe unified signature fixes`).
- Member 3 regression-test work: `64691288a6` on the member branch, later integrated in merge commit `163d1ca2be`.
- Member 2 documentation work: `223d5f3769` on the member branch; its integration was later reverted because it replaced the upstream Biome README symlink with hackathon content.
- Member 4 release/validation work: `d90d1abef0` is recorded in the supplied final report as the documentation/changeset integration commit. The actual branch and file contents should be checked against the live repository before submission.

The team should present each contribution accurately and be prepared to explain the actual work and review history behind each commit. Separate commits are useful evidence of collaboration, but commit history alone does not establish the substance or authorship of work.

---

## 12. Git collaboration, integration, and latest known repository state

### 12.1 Repository and branch setup

The team used:

- Upstream: `https://github.com/biomejs/biome.git`
- Fork (`origin`): `https://github.com/NIRANDEV09/biome.git`
- Feature branch: `fix/10784-parameters-overload`

### 12.2 Important README correction

The upstream Biome repository's root `README.md` is a symbolic link to the package README:

`./packages/@biomejs/biome/README.md`

The team discovered that one hackathon documentation integration replaced the root README symlink with hackathon content. That integration was reverted to preserve the upstream repository structure.

Hackathon-specific documents should be kept as separate files, for example:
- `README-R2-12.md`
- `README-R3-12.md`
- `README-R2-12-VALIDATION.md`
- `readme-machzero.md` (this integrated report)

Do not replace the upstream Biome root `README.md` with the hackathon report.

To check the symlink in Git:

```powershell
git ls-tree HEAD README.md
```

A mode beginning with `120000 blob` indicates a symbolic link.

### 12.3 Latest test commit was reverted

The user later identified a problem with the latest test-integration version and reverted commit `91d3f466ab`.

The known sequence shared in the conversation was:

```text
91add55db5 Revert "test(useUnifiedTypeSignatures): integrate issue #10784 regression cases into invalid.ts"
91d3f466ab test(useUnifiedTypeSignatures): integrate issue #10784 regression cases into invalid.ts
d90d1abef0 docs(biome): add issue 10784 release and validation notes
cb6a4eb884 Revert "docs: integrate member 2 README contribution"
dd6f59781d docs: integrate member 2 README contribution
163d1ca2be test: integrate member 3 regression tests
```

The user reported that the revert was pushed successfully to:

`origin/fix/10784-parameters-overload`

The reported push output advanced the remote branch from `91d3f466ab` to `91add55db5`. The user also reported:

```text
On branch fix/10784-parameters-overload
Your branch is up to date with 'origin/fix/10784-parameters-overload'.

nothing to commit, working tree clean
```

### 12.4 What the revert means

The latest known revert undoes the changes introduced by `91d3f466ab`. It does not erase that commit from history. The earlier commit `163d1ca2be` remains in the displayed history and is reported to contain the original Member 3 regression-test integration.

Because the latest test commit was reverted, do not claim that every change from `91d3f466ab` is present in the current branch. Before submission, inspect the exact current files and test output.

Useful verification commands:

```powershell
git fetch origin
git switch fix/10784-parameters-overload
git status
git log -8 --oneline
git ls-tree HEAD README.md
git diff upstream/main...HEAD --stat
```

If the local `upstream` remote is not configured, inspect remotes with:

```powershell
git remote -v
```

Do not use `git reset --hard`, `git clean`, or force-push as routine synchronization steps. A normal pull is appropriate when local changes do not conflict; inspect status first if Git reports conflicts or local modifications.

---

## 13. Suggested final demonstration for judges

A clear demonstration should show the problem, the targeted safeguard, the regression test, and the honest validation evidence.

### Step 1 — Show the original TypeScript behavior

Show the overloads and the `Parameters<Api["run"]>` assignment. Explain why the final overload produces a required one-element tuple.

### Step 2 — Show the unsafe transformation

Show the unified `run(input?: string)` form and the resulting `TS2322` tuple incompatibility.

### Step 3 — Show the production guard

Open:

`crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs`

Explain the `MakeOptional` check in `Rule::action()` and why it returns `None` rather than removing the diagnostic.

### Step 4 — Show regression coverage

Open the current issue fixture and snapshot files. Verify they are actually present in the branch being demonstrated.

### Step 5 — Run tests from the current branch

After confirming Rust/Cargo are installed and the working tree/branch are correct, run:

```powershell
cargo test -p biome_js_analyze use_unified_type_signatures
cargo test -p biome_js_analyze specs::style
cargo clippy -p biome_js_analyze -- -D warnings
rustfmt --check crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs
```

Run the TypeScript checks from the correct reproduction or fixture directory with the appropriate `tsconfig.json`.

### Step 6 — Show Git history and changeset

Show the feature branch, production commit, regression-test integration, changeset, and revert history. Explain why the later test integration was reverted and identify which regression files remain in the current branch.

**Judge-facing honesty:** If the toolchain cannot be installed or a command cannot be rerun, say so. Present supplied historical test results as reported results, not as a live demonstration.

---

## 14. Final conclusion

The team's investigation identified a concrete semantic risk in overload consolidation: making a required parameter optional can change the tuple extracted by TypeScript's `Parameters<T>` utility and break code that depends on the required tuple shape.

The selected fix is a targeted action-stage safeguard. It retains the lint diagnostic but withholds the automatic fix when the proposed merge includes `MergeParameterOperation::MakeOptional`. This approach aims to prevent the unsafe edit while preserving other actions that do not require that operation.

The work covered:
- TypeScript behavior and issue reproduction analysis;
- tracing the Biome overload-merging and code-action path;
- evaluating multiple solution designs and their trade-offs;
- implementing a narrow Rust safeguard;
- preparing regression fixtures and snapshots;
- preparing changeset and validation documentation;
- handling Windows toolchain, stack-size, and nested-configuration challenges; and
- managing collaborative Git integration and a corrective revert.

The supplied final validation report records passing focused tests, broader style tests, Clippy, formatting, and TypeScript checks. Those results are clearly identified as reported historical results and should be corroborated with fresh terminal or CI output for the exact revision submitted to the judges.

**Overall outcome:** a narrowly scoped proposed safety improvement for Biome issue #10784, supported by a documented investigation, implementation rationale, regression strategy, and transparent validation history.

---

## Related links

- [Biome issue #10784](https://github.com/biomejs/biome/issues/10784)
- [Biome upstream repository](https://github.com/biomejs/biome)
- [MachZero team fork](https://github.com/NIRANDEV09/biome)
- [External reproduction repository](https://github.com/jakeleventhal/biome-use-unified-type-signatures-repro)
- [Rust installation](https://rustup.rs/)

---

*Documentation note: This integrated report was assembled from the Round 2 investigation notes, Round 3 implementation report, final validation summary, and Git outputs supplied during the conversation. It is a consolidated project record, not an independently executed build or test certification. Verify the current branch, file list, tool versions, hardware inventory, and test logs before final submission.*
