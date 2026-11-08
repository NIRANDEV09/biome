# Biome Open-Source Hackathon
## Round 2 — Technical Investigation & Solution Planning

| Project detail | Information |
|---|---|
| Repository | [biomejs/biome](https://github.com/biomejs/biome) |
| Selected issue | [#10784](https://github.com/biomejs/biome/issues/10784) |
| Issue title | `useUnifiedTypeSignatures` unsafe fix changes `Parameters<>` output and breaks TypeScript |
| Primary language | Rust |
| Relevant module | `biome_js_analyze` |
| Investigation date | 7 November 2026 |
| Current stage | Issue analysis and implementation planning |

---

## 1. Round 2 objective

After selecting Biome issue #10784, our objective was to understand the problem beyond its description and identify where and why it occurs within the codebase.

We focused on four questions:

1. What exactly is going wrong in TypeScript?
2. Which part of Biome's implementation is responsible for the transformation?
3. Why can the transformation change the program's type-level behaviour?
4. How can we prevent the problem without breaking legitimate functionality?

Our priority was to establish a technically justified solution before modifying production code.

## 2. Understanding the issue

Biome provides a lint rule called `useUnifiedTypeSignatures`, which attempts to consolidate compatible TypeScript function overloads into a single signature.

Consider the following example:

```ts
interface Api {
    run(): void;
    run(input: string): void;
}
```

The two overloads allow calls with no argument or with a string argument.

The rule may attempt to simplify them into:

```ts
interface Api {
    run(input?: string): void;
}
```

Although these declarations appear similar in terms of permitted function calls, they do not necessarily preserve the same TypeScript type-level behaviour.

### The underlying problem: `Parameters<T>`

TypeScript provides the `Parameters<T>` utility type to extract a function's parameter types as a tuple.

For overloaded functions, TypeScript uses the final overload signature when evaluating this utility type.

With the original declarations:

```ts
type RunArgs = Parameters<Api["run"]>;
```

The resulting tuple represents a required parameter:

```ts
[input: string]
```

After the potential transformation, the extracted tuple becomes optional:

```ts
[input?: string | undefined]
```

This changes the observable type information.

For example:

```ts
const args = ["value"] as RunArgs;
const requiredArgs: [string] = args;
```

The original type is compatible with `[string]`, whereas the optional tuple is not.

The reported failure is a TypeScript `TS2322` error.

**Key finding:** A transformation that preserves ordinary function-call behaviour can still change the type-level behaviour of a TypeScript API.

This makes the issue a semantic correctness problem rather than a formatting or cosmetic problem.

## 3. Technical investigation

Our investigation moved from understanding the TypeScript behaviour to tracing the corresponding Biome implementation.

### 3.1 Relevant source file

The primary implementation file identified was:

`crates/biome_js_analyze/src/lint/style/use_unified_type_signatures.rs`

The following implementation components were examined:

| Component | Responsibility |
|---|---|
| `Rule::run()` | Entry point for rule execution and overload discovery |
| `try_merge_overloads()` | Determines whether two overloads are eligible for merging |
| `try_merge()` | Compares parameter lists and generates merge instructions |
| `MergeOverloadSignaturesInfo` | Carries information about the proposed merge |
| `MergeParameterOperation` | Describes the required parameter transformation |
| `MergeParameterOperation::MakeOptional` | Represents making a parameter optional |
| `Rule::action()` | Applies the proposed source-code transformation |
| `mutation.replace_node()` | Participates in syntax-tree modification |

### 3.2 Understanding the merging process

The implementation investigation identified the following conceptual flow:

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
Rule::action()
    |
    v
Apply syntax-tree mutations
```

The important distinction is between deciding whether a merge is possible and actually applying the transformation.

`try_merge_overloads()` performs compatibility checks before delegating parameter-level merging.

These checks include considerations such as:

- Generic type parameter compatibility.
- Return-type compatibility.
- Documentation-related configuration.
- Parameter availability and naming.
- Whether the parameter lists can be merged in either direction.

The investigation also explored Rust concepts used in this implementation, including `Option<T>`, `Some`, `None`, references, early returns, iterators, pattern matching, and transformation instructions.

### 3.3 The critical investigation point

The most important area identified was the parameter-merging logic inside `try_merge()`.

In particular:

```rust
MergeParameterOperation::MakeOptional
```

The relevant logic examines additional parameters in a longer overload and checks whether they can be made optional.

The discussed implementation includes:

```rust
if !self_param.can_be_optional() {
    return None;
}
```

When the parameter is eligible, the implementation can generate a `MakeOptional` instruction.

This is particularly relevant because the reported issue involves a required parameter becoming optional after overload consolidation.

The investigation also examined `Rule::action()`, where the generated instructions are applied through syntax-tree mutation.

**Current technical assessment:** `MakeOptional` is a key investigation point because it is connected to the reported change in `Parameters<T>`. However, the exact conditions under which the original reproduction triggers this behaviour still require validation against the current implementation.

## 4. Reproduction and experimental understanding

We examined the issue's reported reproduction and the distinction between TypeScript behaviour and Biome's actual transformation.

The reproduction discussion referenced:

- Biome version `2.5.1`.
- TypeScript version `5.9.2`.
- Bun as the package and command runner.
- The repository: [biome-use-unified-type-signatures-repro](https://github.com/jakeleventhal/biome-use-unified-type-signatures-repro).

The discussed command sequence was:

```bash
bun install
bunx tsc --noEmit
bunx biome check --write --unsafe .
bunx tsc --noEmit
```

The intended experiment was:

1. Install the reproduction project dependencies.
2. Type-check the original TypeScript code.
3. Apply Biome's unsafe automatic fix.
4. Type-check the transformed code again.
5. Compare the results.

The reproduction notes describe the initial checks passing and the subsequent TypeScript check reporting `TS2322`.

An output mentioned during the reproduction discussion was:

```text
Checked 4 files in 4ms. Fixed 1 file.
```

These details describe the reported reproduction workflow. They should not be interpreted as independently verified execution logs from every team member.

An important distinction emerged:

- **TypeScript-level reproduction:** Demonstrating that required and optional parameter tuples are different.
- **Biome-level reproduction:** Demonstrating that Biome's actual unsafe fix produces the problematic transformation.

The first explains the type-system behaviour. The second is necessary to confirm the exact implementation path.

## 5. Proposed solution direction

We considered how the rule could avoid changing observable TypeScript behaviour while preserving its useful functionality.

Two main directions emerged:

**Approach A — Conservative merge rejection**

Reject an overload merge when the resulting signature changes relevant observable type behaviour.

Advantages:
- Simpler safety-oriented approach.
- Avoids applying a transformation that changes the extracted parameter tuple.

Limitation:
- Must avoid rejecting legitimate overload simplifications unnecessarily.

**Approach B — More sophisticated type-aware merging**

Investigate whether a different unified signature can preserve the original type behaviour.

Advantages:
- Could retain more valid transformations.

Limitations:
- Requires deeper understanding of TypeScript overload semantics.
- Potentially more complex to implement and validate.

### Preferred investigation direction

Our current direction is to investigate a targeted safeguard around the conditions that generate `MakeOptional`.

We do not want to disable the entire lint rule or remove all optional-parameter merging.

The intended outcome is:

- Prevent the problematic transformation.
- Preserve legitimate overload simplifications.
- Add regression coverage for the reported behaviour.

These are proposed approaches, not an implemented or validated fix.

## 6. Testing and validation strategy

We identified the rule's relevant test area as:

`crates/biome_js_analyze/tests/specs/style/`

The testing plan is to inspect the existing rule-specific specifications and follow Biome's established regression-testing conventions.

The following commands were identified for investigation and validation:

```bash
rg "useUnifiedTypeSignatures"
rg "unified.*type.*signatures" crates
```

```bash
just test-lintrule useUnifiedTypeSignatures
cargo test -p biome_js_analyze
just f
just l
```

The planned validation criteria are:

| Validation | Expected outcome |
|---|---|
| Original TypeScript reproduction | Compiles successfully |
| Reported unsafe transformation | Reproduces the type incompatibility |
| Proposed safeguard | Prevents the problematic transformation |
| Regression test | Protects against recurrence |
| Existing lint-rule tests | Continue to pass |
| Legitimate overload merging | Remains supported |

The exact test commands and conventions should be confirmed against the repository's current contributor documentation before execution.

## 7. Key decisions and lessons learned

During the investigation, we established several important engineering principles:

1. **Syntax-level similarity does not guarantee type-level equivalence.** Two function declarations may permit similar calls but produce different utility-type results.

2. **Automatic fixes require semantic consideration.** Improving code readability should not silently change important type information.

3. **The decision and mutation stages must be distinguished.** The merging logic determines the proposed transformation, while the action applies it.

4. **A narrow safeguard is preferable to disabling useful functionality.** The goal is to prevent the problematic case without unnecessarily restricting valid transformations.

5. **Reproduction and implementation validation are separate responsibilities.** A manually written transformed signature does not prove that Biome generated it.

6. **Regression tests are essential.** A proposed fix must protect the reported behaviour while demonstrating that existing valid transformations still work.

## 8. Current progress and remaining work

| Area | Status |
|---|---|
| Repository and issue selection | Completed |
| Understanding the reported TypeScript behaviour | Initial investigation completed |
| Primary implementation file identified | Completed |
| Relevant merging and mutation functions identified | Initial investigation completed |
| `MakeOptional` identified as a key investigation point | Completed |
| Reproduction strategy documented | Completed |
| Exact Biome transformation independently verified | Pending |
| Root cause fully confirmed | Pending |
| Final implementation approach | Pending verification |
| Rust implementation | Not yet evidenced |
| Regression test | Pending |
| Relevant test suite validation | Pending |
| Commit and pull request | Not yet evidenced |

## 9. Next-round implementation roadmap

Our next stage will follow this sequence:

1. Reproduce the reported transformation using Biome itself.
2. Trace the complete execution path from `Rule::run()` through merging and action application.
3. Confirm precisely when the `MakeOptional` operation causes the reported type-level difference.
4. Determine the smallest safe condition for rejecting or modifying the transformation.
5. Implement the targeted change.
6. Add the original issue as a regression test.
7. Add coverage for legitimate overload merging.
8. Run the relevant lint-rule and analyzer tests.
9. Validate the resulting TypeScript behaviour.
10. Review and prepare the contribution for submission.

---

## 10. Round 2 conclusion

Round 2 moved our work from repository and issue selection into a focused investigation of Biome's overload-merging implementation.

We identified the relationship between TypeScript overloads, `Parameters<T>`, optional parameter tuples, and Biome's `MakeOptional` transformation. We also identified the main source file and the functions involved in deciding and applying overload merges.

Our most important finding is that overloads that appear equivalent in ordinary function calls may not be equivalent when examined through TypeScript's type-level utilities.

The next challenge is to translate this understanding into a narrow, tested implementation that prevents the reported regression without compromising legitimate functionality.

**Round 2 outcome:** Technical understanding and solution planning established. Implementation and regression validation remain the priorities for the next round.