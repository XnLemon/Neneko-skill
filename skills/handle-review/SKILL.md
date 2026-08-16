---
name: handle-review
description: Triage and handle pull-request review threads with evidence-based scope decisions, minimal fixes, regression tests, proportional validation, and English-first replies with folded Chinese translations. Use when asked to inspect open or inline reviews, judge reviewer scope, distinguish required omissions from valid follow-ups, apply review fixes, verify whether comments are already addressed, or draft/post review responses.
---

# Handle Review

Handle review comments as claims to verify, not instructions to apply blindly. Define scope by the pull request's behavioral contract and the invariants changed by its diff, not only by title packages or touched files.

## Respect Authority

Match actions to the user's request before touching code or GitHub state.

- For analysis, explanation, or review, remain read-only.
- For a requested fix, edit and validate, but commit, push, reply, resolve, dismiss, or start a follow-up only when authorized.
- Preserve unrelated worktree changes and never rewrite the branch merely to simplify review handling.
- State any unavailable evidence or validation instead of guessing.

## Build The Evidence Set

Read enough context to reconstruct the intended outcome before classifying any thread:

1. Read the PR title, body, linked issue, base and head branches, commits, changed files, and current CI state.
2. Read the complete thread, including later replies, resolution and outdated state, and related comments from the same reviewer.
3. Inspect the latest HEAD implementation and tests around the cited line. Never judge only from the reviewed commit or stale diff position.
4. Search for sibling switches, clone paths, serializers, adapters, protocol converters, persistence paths, and tests that consume the changed type or behavior.
5. Read repository instructions and package contracts before proposing a public API, protocol, persistence, or lifecycle change.

Treat resolved and outdated as thread metadata, not proof that a claim is fixed. Treat severity as impact, not proof that the work belongs in the current PR.

## Translate Each Comment Into A Claim

Record these facts before deciding:

- The concrete input or state that triggers the issue.
- The observed failure: rejection, silent loss, aliasing, corruption, race, compatibility break, or missing representation.
- The contract or invariant allegedly violated.
- Whether the failure exists on current HEAD.
- The reviewer's proposed implementation and test, considered separately from the finding itself.

Reproduce the behavior or prove it from a complete code path. A valid diagnosis can still have an oversized remedy; accept, narrow, or reject the proposed remedy independently.

## Classify Validity And Scope Separately

Assign one validity result and one scope result to every actionable thread.

Validity results:

- `valid`: current HEAD still exhibits the claimed contract failure.
- `already-addressed`: current HEAD contains the fix and meaningful coverage.
- `not-reproducible`: the claim conflicts with current behavior or the documented contract.
- `unclear`: evidence is insufficient or the intended contract is undefined.

Scope results:

- `required-closure`: the current PR is incomplete or unsafe without the fix.
- `valid-adjacent`: the issue is real, but belongs to an independent follow-up.
- `scope-unclear`: competing contracts make a product or maintainer decision necessary.

Never collapse `valid` into `required-closure`. "This issue is real" and "this PR must fix it" are different conclusions.

## Decide Behavioral Scope

Classify a valid finding as `required-closure` when one or more of these conditions hold:

- The behavior is explicitly promised by the PR outcome, linked issue, documentation, or tests.
- The affected code is on the declared end-to-end path and the omission makes that path reject, drop, corrupt, race, or misrepresent supported input.
- The PR adds a variant to a public sum-like model and an existing generic, exhaustive operation now violates its established invariant, such as deep-copy isolation, equality, ownership, or serialization round trips.
- The PR regresses existing inputs, defaults, compatibility, security, concurrency, or persistence semantics.
- The smallest fix closes the same contract without creating a separate capability or protocol.

Classify a valid finding as `valid-adjacent` when all relevant evidence points to a separate capability boundary:

- The affected subsystem is optional or independently selected, such as a transport, exporter, storage wrapper, or integration not promised by the PR.
- The subsystem has an explicit support set rather than a generic all-variants invariant.
- Correct support requires its own protocol, schema, metadata, compatibility, migration, or full externalize/hydrate lifecycle.
- Deferring it leaves the PR's declared configurations and path correct.
- A focused follow-up can deliver and test the capability without changing the current PR's contract.

Use `scope-unclear` when evidence conflicts. Explain the competing contracts and request a decision only if choosing either direction would materially change the PR.

Do not use package names, changed-file lists, reviewer priority, implementation size, or personal preference as the sole scope test. A file outside the title can contain required closure; a nearby file can belong to a separate feature.

## Calibrate With The PR 2402 Pattern

Use these examples to distinguish closure from repository-wide parity:

- Gemini MIME normalization was required closure. The PR explicitly promised Gemini support, while values emitted by the new helpers and AG-UI path could be rejected at that provider boundary.
- Graph video deep copy was required closure despite `graph` being outside the title. Adding a pointer-bearing `Video` variant broke an existing generic deep-copy isolation guarantee and risked cross-branch mutation and races.
- Session video externalization was valid but adjacent. It was an opt-in persistence capability with a separate externalize/hydrate lifecycle, and the declared AG-UI-to-provider path remained correct without it.
- A2A video and URL-audio transport was valid but adjacent. Supporting it required request and response round trips, metadata contracts, and two independent converter versions.
- OTel and Langfuse media mapping was valid but adjacent. It affected an optional observability projection rather than request execution and could be delivered as a focused telemetry change.

The general rule is: close invariants introduced or broken by the changed abstraction, but do not expand the PR into parity work for every independent consumer of that abstraction.

## Apply The Smallest Complete Fix

For each `valid` and `required-closure` thread:

1. Patch every path owned by the same invariant. Do not fix one branch while leaving an equivalent branch silently broken.
2. Preserve defaults, compatibility, ownership, aliasing, cancellation, persistence, and wire behavior unless the review explicitly requires a documented change.
3. Add a regression test that fails before the fix and proves the caller-visible contract.
4. Make mutation and boundary tests non-vacuous: change values to genuinely different values, assert both source and copy, and cover URL/data or request/response forms only when those forms belong to the same contract.
5. Decline duplicate tests or the reviewer's exact implementation when existing coverage proves the shared layer. Explain the narrower remedy with evidence.
6. Avoid opportunistic refactoring and adjacent feature parity.

For `valid-adjacent` threads, do not modify code in the current PR. Record the exact missing capability, why it is separate, and the minimum follow-up boundary.

## Validate And Re-Review

Run validation in widening rings:

1. Run the smallest regression test for the changed behavior.
2. Run the owning package or module tests.
3. Run broader build, test, lint, race, protocol, or persistence checks proportional to the affected invariant and repository guidance.
4. Inspect the final diff for accidental scope growth and unrelated edits.
5. Re-read every handled thread against current HEAD.
6. Perform a separate public API and framework-contract review when exported or observable behavior changed.

Track actual commands and results. Never claim a check, commit, or fix that did not happen.

## Keep A Disposition Ledger

Use a compact table while working or in the final report:

| Thread | Validity | Scope | Evidence | Action | Validation |
| --- | --- | --- | --- | --- | --- |
| Short claim | valid / addressed / invalid / unclear | required / adjacent / unclear | Current-HEAD proof | fix / defer / none / ask | Actual checks |

Lead the final report with required fixes and unresolved risk. Then list valid follow-ups so genuine omissions are not lost merely because they were kept out of scope.

## Reply In English And Folded Chinese

Keep replies factual and specific. Name the affected contract, what changed or why it is deferred, regression coverage, and actual validation. Put English first and Chinese in a `<details>` block.

### Fixed

```markdown
Fixed in `<commit>`. `<component>` now `<behavioral outcome>`, preserving `<important invariant or compatibility point>`. Added `<regression coverage>`. Validation: `<actual command/result>`.

<details>
<summary>中文</summary>

已在 `<commit>` 中修复。`<component>` 现在会 `<行为结果>`，并保持 `<关键不变量或兼容性>`。已补充 `<回归测试>`。验证：`<实际命令/结果>`。

</details>
```

Omit the commit sentence when the change is not committed.

### Valid But Adjacent

```markdown
This is a valid issue: `<confirmed impact>`. It affects `<independent capability>` rather than this PR's declared `<current path/outcome>`, and complete support requires `<protocol/schema/lifecycle boundary>`. I am keeping it out of this PR and recommend a focused follow-up covering `<minimum follow-up scope>`.

<details>
<summary>中文</summary>

这个问题本身有效：`<已确认的影响>`。它影响的是 `<独立能力>`，而不是本 PR 声明的 `<当前链路/结果>`；完整支持还需要定义 `<协议/schema/生命周期边界>`。因此本 PR 不扩展该范围，建议用独立 follow-up 覆盖 `<最小后续范围>`。

</details>
```

### Already Addressed Or Not Reproducible

```markdown
I rechecked current HEAD. `<claim>` is already covered by `<code/test/commit>`, which `<specific evidence>`. No additional change is needed.

<details>
<summary>中文</summary>

我重新核对了当前 HEAD。`<问题>` 已由 `<代码/测试/提交>` 覆盖，具体证据是 `<证据>`，因此无需额外修改。

</details>
```

For an invalid finding, replace "already covered" with the documented reason it does not apply. Do not dismiss a concern without current-HEAD evidence.

### Partial Acceptance

```markdown
The underlying gap is valid, and I fixed `<required behavior>`. I did not add `<requested duplicate or broader change>` because `<existing shared contract/evidence>` already covers it or it belongs to `<separate capability>`. Validation: `<actual command/result>`.

<details>
<summary>中文</summary>

底层问题有效，已修复 `<必要行为>`。没有增加 `<建议中的重复或更宽修改>`，因为 `<现有共享契约/证据>` 已覆盖它，或它属于 `<独立能力>`。验证：`<实际命令/结果>`。

</details>
```

### Clarification Needed

```markdown
I can confirm `<observed fact>`, but the intended contract for `<boundary>` is not defined by the PR or existing tests. Should this PR guarantee `<option A>`, or should `<option B>` remain a focused follow-up? This choice changes `<compatibility/scope impact>`.

<details>
<summary>中文</summary>

我可以确认 `<已观察事实>`，但 PR 和现有测试都没有定义 `<边界>` 的预期契约。这个 PR 应保证 `<选项 A>`，还是将 `<选项 B>` 留给独立 follow-up？该选择会影响 `<兼容性/scope 影响>`。

</details>
```
