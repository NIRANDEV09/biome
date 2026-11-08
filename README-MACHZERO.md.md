# Biome Issue #10784 — Safe Overload Unification

> **MachZero Open-Source Hackathon · Round 3 Implementation and Validation**

**Issue:** [#10784 — `useUnifiedTypeSignatures` unsafe fix changes `Parameters<>` output and breaks TypeScript](https://github.com/biomejs/biome/issues/10784)  
**Upstream:** [`biomejs/biome`](https://github.com/biomejs/biome) · **Team fork:** [`NIRANDEV09/biome`](https://github.com/NIRANDEV09/biome)  
**Branch:** `fix/10784-parameters-overload`  
**Rule:** `lint/style/useUnifiedTypeSignatures` · **Implementation:** Rust

## 🏆 Hackathon Overview

During this hackathon, our team tackled a subtle but critical bug in Biome's TypeScript linting ecosystem. Biome's `useUnifiedTypeSignatures` rule recommends combining similar TypeScript overloads. However, its unsafe code action was automatically making required parameters optional. This edit silently changed the type returned by TypeScript's `Parameters<T>` utility, causing previously valid type assignments to fail and breaking user builds.

Our contribution successfully patches this by keeping the helpful lint diagnostic while intelligently withholding the unsafe code action whenever the proposed merge contains a `MakeOptional` operation. 

## 🚨 The Problem

Consider this API:

```ts
interface Api {
  run(): void;
  run(input: string): void;
}

type RunArgs = Parameters<Api["run"]>;

const args = ["value"] as RunArgs;
const requiredArgs: [string] = args;
```

For an overloaded function, TypeScript's `Parameters<T>` uses the last overload. With the original declarations, `RunArgs` is `[input: string]`, so the assignment to `[string]` is valid.

The unsafe unification produced by Biome previously resulted in:

```ts
interface Api {
  run(input?: string): void;
}
```

Now `RunArgs` is `[input?: string | undefined]`. The tuple may be empty, so it is no longer assignable to `[string]`. TypeScript reports `TS2322`. 

**The core insight:** Similar call signatures do not necessarily preserve type-level behavior. An automatic fix must not silently change that behavior.

## 🛠️ Our Solution & Implementation

The production change is cleanly localized to the rule's action stage in:
`crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs`

At the beginning of `Rule::action()`, we implemented a safeguard that checks the proposed parameter merge operations:

```rust
if state
    .parameters_to_merge
    .iter()
    .any(|info| matches!(info.operation, MergeParameterOperation::MakeOptional))
{
    return None;
}
```

* **Diagnostic:** Still accurately reported by `Rule::diagnostic()`.
* **Unsafe code action:** Safely withheld for merges containing `MakeOptional`.
* **Other actions:** The existing action path remains fully available when no `MakeOptional` operation is present (e.g., safe duplicate-removal cases).

## ✅ Test Results & Validation

Our team ensured rigorous testing to validate the fix. The final validation run confirmed that all checks passed successfully.

| Check | Command / Target | Result |
| :--- | :--- | :--- |
| **Focused Lint Tests** | `cargo test -p biome_js_analyze use_unified_type_signatures` | **Passed — 11/11** |
| **Full Style Tests** | `cargo test -p biome_js_analyze specs::style` | **Passed — 795/795** |
| **Rust Linting** | `cargo clippy -p biome_js_analyze -- -D warnings` | **Passed — 0 warnings** |
| **Rust Formatting** | `rustfmt --check <file>` | **Passed** |
| **TS Regression Check**| `tsc --noEmit` on `issue_10784.ts` (TypeScript 7.0.2) | **Passed — 0 errors** |
| **TS Bug Reproduction**| `tsc --noEmit` on the unsafe transformed example | **Reproduced** — `TS2322` |

### Regression Coverage
We added dedicated fixtures and snapshots for this issue:
* `crates/biome_js_analyze/tests/specs/style/useUnifiedTypeSignatures/issue_10784.ts`
* `crates/biome_js_analyze/tests/specs/style/useUnifiedTypeSignatures/issue_10784.ts.snap`

The new fixture covers four scenarios:
1. Zero-argument and one-argument overloads.
2. One-argument and two-argument overloads.
3. Duplicate signatures (verifying existing fixes still work).
4. An overload with an already-optional parameter.

## 👥 Hackathon Team & Work Split

The work was divided into focused responsibilities to maximize our hackathon velocity. **Final Impact: 6 files changed, 263 insertions(+), 79 deletions(-).**

* **Member 1 (Core Implementation):** Implemented the Rust safeguard in `Rule::action()` to intercept `MakeOptional` operations.
* **Member 2 (Investigation & Design):** Traced the TypeScript behavior, inspected the overload-merging flow, and evaluated solution approaches.
* **Member 3 (Regression Testing):** Authored the issue fixture and snapshot coverage (`commit 64691288a6`); updated existing expected diagnostics like `invalid.ts.snap`.
* **Member 4 (Validation & Release):** Ran the final validation suite, documented the test results, verified the TypeScript reproduction, and prepared the changeset/PR material (`commit d90d1abef0`).

## 🚀 Running the Project

From the Biome repository root, with the Rust toolchain and `just` installed:

**Run the focused test:**
```bash
just test-lintrule useUnifiedTypeSignatures
```

**Run broader analyzer coverage:**
```bash
cargo test -p biome_js_analyze
```

---
*Prepared for the MachZero Open-Source Hackathon.*