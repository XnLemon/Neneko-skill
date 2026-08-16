---
name: neneko-cr
description: Repository-agnostic Neneko CR workflow for triaging, reviewing, fixing, reporting, approving, and optionally merging GitHub pull requests across any project. It separates external-PR review from own-PR maintenance, checks current-head CI and review threads, verifies problem necessity and solution fit, requires an evidence chain in every posted review comment, preserves auditable reports, and captures workflow improvements. Use when the user says "执行 neneko-cr", "run neneko-cr", "run CR", or asks Codex to batch-review pull requests, inspect review feedback, maintain own PRs, post evidence-backed inline findings, approve or merge eligible changes, generate a review report, or improve the review workflow.
---

# Neneko CR

## Overview

Run a repository-agnostic maintainer code-review loop for the target GitHub
repository selected by the user or discovered from the current workspace. The
skill name is `neneko-cr`, and the workflow is generic CR: adapt to the target
project's language, build system, CI, branch policy, review policy, and report
conventions instead of assuming a particular repository.

Resolve the target repository in this order:

1. An explicit repository, PR URL/number, local checkout, or target named by
   the user.
2. `CR_TARGET_REPOSITORY` (`<owner>/<repo>`).
3. The `origin` remote of the current checkout when that checkout is the target
   project rather than a dedicated review workspace.
4. The repository associated with an explicitly supplied local path.
5. If the current checkout looks like a dedicated review workspace and no
   target is configured, ask for the target instead of reviewing the workspace
   itself.
6. If more than one target remains plausible, ask the user to choose; do not
   silently review the wrong repository.

Resolve the local target checkout in this order:

1. `CR_TARGET_CHECKOUT`.
2. A local path supplied by the user.
3. The current checkout when its `origin` matches the target repository.
4. A sibling checkout with a matching remote when one is available.
5. GitHub/API-only review when no local checkout is available and the task does
   not require local edits or tests.

Resolve the review workspace and report root from `CR_WORKSPACE` and
`CR_REPORT_ROOT` when set. Otherwise use the current repository when it
contains this skill and report tooling, then its existing report convention,
and finally a repo-relative `reports/` directory. Prefer repo-relative paths;
never rely on a user-specific absolute path.

Useful optional configuration includes `CR_OWN_AUTHORS` (additional own-PR
logins), `CR_DEFAULT_BASE` (base branch override), `CR_SOFT_CHECKS` (checks
allowed to remain non-green during review), and `CR_MERGE_POLICY` (whether
final merge requires a human). A repository's documented policy takes
precedence over these defaults.

Prefer `gh` when it is authenticated; otherwise use the GitHub connector tools.

## Workflow

1. Identify the target repository from the user request, PR URL/number,
   `CR_TARGET_REPOSITORY`, or the current `origin` remote when the current
   checkout is the target rather than a dedicated review workspace. Identify
   the review workspace from `CR_WORKSPACE` or the current repository when it
   contains `skill/SKILL.md`; otherwise use the target repository's existing
   review tooling and report convention. Resolve the target checkout using the
   order in the overview. Record the selected repository, checkout, base
   branch, and report root before reviewing any PR.
2. Determine the authenticated reviewer login. Treat PRs authored by that login
   as own PRs. Also treat logins listed in `CR_OWN_AUTHORS` or the repository's
   explicit maintainer configuration as own PR authors; never hard-code a
   person, organization, or project-specific username. Own PRs are not skipped
   by default. They enter the own-PR maintenance path below: review them for
   real issues, fix valid issues directly when the branch is pushable, and
   optionally leave a self-review record when useful. Do not self-merge them
   from this run.
3. Start with incremental planning, but keep a lightweight full-open-PR ledger:
   - Collect a cheap index for every open PR before doing any expensive diff,
     thread-body, log, source, or report-writing work. This index is a cache
     invalidation input, not a review artifact. It must include at least title,
     author, author association, labels, draft/WIP state, base branch, head SHA,
     mergeability, review decision, locked state and active lock reason,
     updated/latest-activity time, commit/check fingerprint,
     review-thread fingerprint, and
     comment/review fingerprint. Build those
     fingerprints from stable IDs, authors, states, timestamps, check names and
     conclusions, head SHAs, close/reopen/lock/unlock timeline events, and
     changed-file path summaries. Do not put full PR bodies, full
     review/comment bodies, diff hunks, source files, workflow logs, or long
     automated-review comments into the cheap index.
     When small excerpts from external GitHub text are needed for fingerprints
     or blocker evidence, sanitize invalid Unicode and truncate by Unicode code
     point rather than UTF-16 index so emoji or other surrogate-pair text cannot
     produce invalid JSON.
   - Compare that index with the latest successful run-state snapshot by using
     the review workspace's configured incremental planner when one exists.
     When no planner exists, create an equivalent plan with the same inputs and
     conservative decisions. Use a small overlap window instead of a strict
     timestamp boundary so updates that happen during the prior run are not
     missed. An unchanged PR should enter a lightweight `refresh_probe`, not a
     `heavy_review`; use a full sweep only when the user explicitly asks for a
     full code re-review or the metadata looks corrupt. If the previous state
     is missing, unreadable, or does not contain the needed fingerprints, fall
     back to the conservative behavior and treat all open PRs as
     `heavy_review`.
   - After planning, generate a cache manifest with the configured cache
     manifest tool when one exists. For `carry_forward` and `refresh_probe` entries
     whose `metadata_cache_key` matches the previous key, reuse the previous
     full metadata/report-entry cache and do not read diffs, source, full
     comments, full threads, logs, or subagent output. For `refresh_probe`,
     refresh only the cheap index fields; if the probe changes any fingerprint
     or cache key, rerun the planner and promote that PR to `heavy_review`.
     For `heavy_review`, fetch full metadata once and write it under the
     manifest's full-metadata cache path so the next run can reuse it.
   - A PR enters the heavy path when it is new, its head SHA changed, CI/check
     fingerprint changed, review/comment/thread fingerprint changed, readiness
     markers changed, mergeability/base changed, it was previously
     `not_reached`, it has activity after the overlapped watermark that was not
     already seen in the previous snapshot, or a `refresh_probe` discovers a
     changed cache key. Missing required fingerprints must also promote the PR
     to the heavy path. A periodic freshness sweep by itself is only a
     `refresh_probe`; it is not a reason to reread the diff or re-run xhigh
     review. A PR with an unchanged concrete blocker can be carried forward in
     the ledger, but only as a carried-forward blocker with its previous
     evidence and `last_verified_at`; never approve, merge, post comments, or
     mark an own PR maintained from carried-forward data alone.
   - For every PR in the `heavy_review` queue, collect the full metadata needed
     for the existing gates: requested reviewers, review threads, review
     submissions, commit statuses, pull-request workflow runs, mergeability,
     changed files, and diff/source details. For review threads, keep a
     thread-level audit, not only the aggregate GitHub `reviewDecision`: thread
     id or URL, reviewer login, author login, review state, file and line,
     resolved/outdated state, latest author reply, latest reviewer reply,
     whether the thread still maps to the current diff, and the concrete reason
     it is still actionable or no longer actionable.
   - For every candidate that will appear in the reviewed-PR section, also
     build a PR-content summary from the PR description, changed files, and
     diff. This summary must describe what the PR itself changes, not what the
     reviewer did. Capture the original problem or user/developer need, the new
     behavior or API shape, the concrete packages/files/modules touched, the
     important added, modified, or removed code paths, data/control flow
     changes, compatibility impact, docs/examples/tests added, and any design
     tradeoffs visible from the diff. Gather enough source detail to let the
     final report stand alone as an engineering brief; do not rely on vague
     title-level summaries.
   - For every bugfix, behavior fix, compatibility fix, or PR that claims to
     close a linked issue, build a problem-evidence summary before treating it
     as a review candidate. Read the linked issue or source problem report,
     the PR body, maintainer/author discussion, test evidence, logs, payloads,
     and the current base-branch code path needed to judge whether the problem
     still exists on the target base. Classify the premise as
     `confirmed`, `plausible_but_unproven`, `contradicted_or_stale`, or
     `product_decision_needed`. CI green, a plausible diff, generated tests, or
     an implementation that matches the original reporter's proposed fix are
     not enough by themselves to establish necessity.
   - Treat source-problem evidence as a first-class gate. A candidate can be
     approved or merged only when the problem premise is confirmed, or when it
     is an explicitly scoped feature/design change whose motivation is clear
     from the PR discussion. If the latest base behavior, authoritative logs,
     maintainer discussion, or upstream/provider behavior contradicts the
     claimed bug, block it as `contradicted_or_stale`. If the issue report lacks
     the raw request/response/event data, base-branch reproduction, or code-path
     link needed to identify the root cause, block it as
     `plausible_but_unproven` and ask for the smallest missing evidence instead
     of approving a speculative fix. If the change is a policy/product decision
     rather than a correctness fix, block or defer it until the intended
     contract is stated.
   - Separate the user's reported pain from any proposed implementation in the
     issue or PR. A linked issue, bot linked-issue check, or author-proposed fix
     can describe a real symptom while still suggesting the wrong control
     surface. Judge whether the submitted solution is a good fit for the
     underlying product/API/runtime contract: does it preserve useful defaults,
     avoid misleading mode switches, keep configuration semantics coherent, and
     solve the pain at the right abstraction layer? If the literal requested
     implementation would make the API less clear or less maintainable, prefer
     a better design such as clearer docs, an explicit narrowly scoped option,
     or a separate contract decision; block the PR as `solution_fit` when the
     mismatch affects merge readiness.
   - Treat `manual_review` as a first-class gate, separate from ordinary code
     quality, unresolved review threads, and merge permissions. Before
     approving or merging a linked-issue PR, inspect issue labels, issue body,
     PR body, maintainer comments, assignees, active competing PRs for the same
     issue, and code owner requests for signals that the accept/reject decision
     belongs to a human maintainer, mentor, code owner, or product/architecture
     owner.
     Examples include mentorship or contest tracks, issues exclusive to a
     program, or tasks where a mentor selects the accepted contribution,
     labels or text saying the issue is exclusive to a program, "students claim
     this task", "mentor selects winners", explicit "manual review required",
     multiple active PRs competing for one issue, security-boundary changes,
     broad public API or architecture decisions, broad test or verification
     contracts that set expectations for multiple backends, and roadmap/product
     choices.
     In those cases Neneko CR may still audit the code and post real review
     findings, but it must not submit an approval or merge unless a maintainer
     explicitly states that this exact PR is the accepted merge candidate. An
     approval submitted by this workflow, even from a maintainer account used
     by the run, is not evidence that the human decision happened. Report the gate as
     `manual_review` with the concrete label, issue text, maintainer statement,
     or competing PR evidence that made automation unsafe.
   - Track merge authority separately from code-review blockers. When the
     repository or the user requires a human to perform the final merge for a
     class of PRs, record `merge_policy: { mode: "human_required", ... }` with
     the reason and required evidence. Neneko CR may still review the code and, when
     otherwise allowed, approve it, but it must leave the PR unmerged and report
     the human merge requirement explicitly. Apply this policy by default to PRs
     whose author association is not `OWNER`, `MEMBER`, or `COLLABORATOR`,
     unless the user explicitly authorizes Neneko CR to merge that PR class.
   - Maintain a run-level reviewability ledger for every open PR. Each PR must
     end the scan in exactly one auditable bucket: processed, heavy-review
     eligible candidate, carried-forward unchanged blocker, intentionally not
     reached, or blocked by a concrete gate. Do not allow an open PR to
     disappear from the run because of incremental planning, candidate ordering,
     stale aggregate review state, vague soft-CI uncertainty, or a broad skip
     group.
4. Split PRs into two auditable processing paths before applying the candidate
   gates:
   - External PR review path: PRs not authored by an own login. These are the
     only PRs eligible for third-party inline review comments, policy-compliant
     approval, and merge.
   - Own-PR maintenance path: PRs authored by an own login. For these PRs, do
     the same quality audit you would do for an external PR. When the branch is
     pushable, convert valid findings into direct branch fixes first. Prefer a
     separate `git worktree` for the PR branch so existing local changes in the
     main target checkout are not disturbed. Fetch the PR head, check out the
     contributor branch when it is pushable from the authenticated account, run
     the relevant tests/tooling, commit with project-style commit messages when
     changes are needed, and push back to the PR branch. Self-review comments
     are allowed when they add useful audit evidence. A self-review approval may
     be attempted only when GitHub permits it; if GitHub rejects approval on the
     author's own PR, fall back to a normal PR comment as the self-review audit
     record. Self-review records must not replace the direct-fix path for
     pushable own branches and must not be treated as permission to self-merge.
     Do not request changes, submit an approval, merge, or resolve review threads on
     own PRs as if you were a third-party reviewer. The goal is to leave each
     own PR at the quality level where the external PR path would have approved
     it while keeping the final merge decision auditable.
5. Keep external review candidates only when they satisfy every gate:
   - state is open
   - title and labels do not indicate WIP
   - PR is not draft and is ready for review
   - base branch is `CR_DEFAULT_BASE`, the repository default branch, or the
     branch explicitly selected by the user, in that order
   - for bugfixes and linked-issue fixes, the source problem has sufficient
     current evidence on the target base. A PR whose premise is only inferred
     from the diff, copied from a suggested issue fix, or supported only by
     branch-local tests must remain blocked until the current base behavior and
     root-cause path are verified. A stale, non-reproducible, or contradicted
     problem premise is a hard approval and merge blocker even when CI is green.
   - the implementation approach is a reasonable fit for the confirmed problem
     and the repository's public contract. Do not approve a PR merely because it
     satisfies the issue's literal proposed fix or a bot's linked-issue check if
     that fix uses the wrong knob, weakens default behavior, hides a contract
     decision in documentation, or broadens semantics beyond the actual need.
   - the PR is not under a manual-review gate. If issue labels,
     issue text, maintainer comments, code owner requests, active competing PRs,
     security/architecture scope, or product/roadmap ownership indicate that a
     human maintainer must choose the accepted PR, block approval and merge as
     `manual_review` until that explicit selection exists.
   - all commit statuses and workflow runs for the current head SHA are green
     before starting review, except the soft CI cases described below
   - pending, queued, in-progress, cancelled, timed out, missing required
     checks, or failures outside the soft CI cases mean do not review yet
   - CI readiness must be computed from the current head's effective latest
     checks, not from an undeduplicated `statusCheckRollup` that may include
     historical runs left behind by close/reopen cycles. Prefer `gh pr checks
     --watch=false`, current commit status, or latest check-suite/check-run
     data grouped by check name and workflow for the current head. If
     `statusCheckRollup` contains both an old failure and a newer success for
     the same effective check on the same head, treat the newer effective check
     as authoritative and record the stale rollup discrepancy instead of
     skipping the PR as CI-failed.
   - if the only non-green checks are checks explicitly marked as soft by
     repository metadata, repository documentation, or `CR_SOFT_CHECKS`, do
     not skip automatically. Inspect the PR diff, PR description, reviewer
     discussion, and the concrete failure details. Treat the PR as reviewable
     only when the soft failure is understood and is not hiding a correctness,
     compatibility, security, or test-quality risk. Record the judgment in the
     report. If the failure exposes a real blocker, skip or comment as
     appropriate.
   - A non-green soft check may allow review or approval when repository policy
     permits it, but it must not be treated as permission to merge when the
     check is required by repository policy or `CR_MERGE_POLICY`. Before merge,
     re-evaluate every required and merge-blocking check on the current head.
   - human review comments are either resolved, or the PR author has replied
     clearly that they fixed the issue or gave an explanation that we judge
     acceptable after checking the current diff; a forgotten GitHub "Resolve"
     click alone must not block review
   - `reviewDecision=CHANGES_REQUESTED` is not, by itself, a skip reason.
     Treat it as a signal to run the thread-level audit above. If every
     actionable human thread has been resolved, is outdated because the changed
     code moved, or has a clear author `fixed` / `done` / acceptable
     explanation that is verified against the latest diff, the PR remains
     eligible for review. Record the stale aggregate state in the report, but
     do not exclude the PR only because GitHub still displays
     `CHANGES_REQUESTED`.
   - A human-review blocker must name the concrete still-actionable thread or
     review: reviewer login, thread/comment URL, file and line when available,
     the reviewer request, the latest author response if any, and why that
     response or the current diff is insufficient. If this evidence cannot be
     gathered, gather more metadata instead of using a broad reason such as
     "unresolved review", "Changes Requested", or "API discussion".
   - unresolved comments from bots are not hard blockers by themselves; inspect
     whether they describe a concrete current correctness, compatibility, or
     CI risk, and ignore weak, stale, stylistic, or unreasonable bot comments
     after recording that judgment
   - comments previously raised by this workflow or on our behalf must be
     verified against the latest head; if fixed, proceed even if the thread was
     not clicked resolved, and resolve our own/on-behalf threads when allowed
   - locked pull-request conversations must be tracked explicitly. A lock that
     limits comments to collaborators is not automatically a review blocker for
     an authenticated maintainer/collaborator, but it is a comment-delivery
     risk and can block bot/non-collaborator comments. Before posting inline
     comments or approving a locked PR, verify the authenticated account's
     repository permission and, after posting, verify the review/comment URLs.
     If GitHub rejects the review because the conversation is locked or
     otherwise write-restricted, report a first-class comment-delivery blocker
     with the attempted target and requeue the PR after unlock; do not let the
     review silently disappear from the report.
   - after the first gate pass, run a second pass over all non-draft, non-WIP
     external PRs. Any PR with green CI or only acceptable soft-CI failures, no
     merge conflict, and no verified live blocker must be either reviewed in
     this run or listed as `not_reached` with a concrete reason why capacity or
     ordering stopped it. It must not be hidden under CI, human review,
     `Changes Requested`, or "needs discussion" unless the exact current
     blocker has been named and verified.
6. For own PRs, apply the same readiness audit, but use own-PR outcomes:
   - If an own PR is draft, WIP, has non-soft failing CI, has a merge conflict,
     or has a live human-review blocker that requires product/design input,
     record the exact blocker. Self-review comments are allowed only when they
     add useful audit evidence and must not obscure the blocker.
    - If an own PR is ready enough to inspect, perform the full review locally.
      Run one independent reviewer/subagent at the configured depth (default
      `xhigh`) when the change is non-trivial, verify every
     finding yourself, then fix valid issues directly in the PR branch. Prefer
     creating a dedicated `git worktree` for the target branch and then running
      `gh pr checkout <number> --repo <owner>/<repo>` inside that
     worktree, or use explicit fetch/checkouts when that is more reliable.
     Preserve unrelated worktree changes and never reuse a dirty checkout for
     own-PR fixes.
   - After pushing fixes, re-check the PR's CI and latest comments/threads.
     If CI is pending, report it as waiting for CI. If CI is green or only has
     acceptable soft-CI findings, report the PR under `maintained` with the
     commits, tests, residual risks, and any self-review record submitted. Do
     not self-merge it.
   - If the audit finds no needed code changes, report the PR under
     `maintained` as "no direct fix needed" with the evidence that would have
     made the external PR path approve it. A self-review approval, when GitHub
     permits one, is allowed only as an audit record; if GitHub rejects it, use
     a normal PR comment instead. Neither form is sufficient authority to merge
     the PR.
7. For skipped PRs, record the concrete skip reason for the final report. The
   reason must identify the exact gate that blocked review. Use structured
   `blockers` entries when possible. Do not group PRs under vague combined
   reasons such as "human review thread, Changes Requested, or API discussion"
   unless each item also lists the exact thread/comment/check/conflict that is
   actually blocking it. A `manual_review` gate must name the specific label,
   issue sentence, maintainer statement, code owner decision point, competing
   PR, security/architecture scope, broad verification contract, or
   product/roadmap ownership reason that makes Neneko CR approval or merge unsafe.
   If a PR is otherwise reviewable but was not
   processed because of time or candidate ordering, say that plainly as "not
   reached this run" and put it in follow-up, not in a blocker group.
   Before processing candidates and again before writing the report, compare
   the reviewability ledger with the processed list. If a PR looks reviewable
   but was not processed, promote it into the candidate queue when practical;
   otherwise report it as `not_reached` and include the specific reason it was
   deferred. The report must make this visible enough that the user can spot
   missed CR opportunities immediately.
8. Process external candidates one at a time. When the active tool policy
   permits delegation, spawn exactly one independent reviewer/subagent per
   candidate at the configured reasoning depth (default `xhigh`). Give it the
   PR number, repository, diff access
   instructions, and ask for a code-review result only; do not let the subagent
   post comments, approve, merge, or modify files. If the active tool policy or
   runtime does not permit subagent spawning, perform the same review depth
   locally and record the delegation limitation in the report's run capability
   or skill-evolution section instead of silently pretending a subagent ran.
9. Ask the subagent to first assess problem necessity and implementation
   derivation, then code quality. It must state whether the PR's claimed
   problem is confirmed on the current base, what evidence supports or
   contradicts it, which code path explains the root cause, and whether the
   chosen implementation follows from that evidence. Then ask it to focus on
   design, compatibility, likely bugs, cross-module impact, severity, and the
   smallest correct fix. Require code findings to include `path`, changed-side
   `line` or `position`, the reviewed head SHA, severity, concise English body,
   Chinese translation, why the issue matters, and a complete evidence chain.
   The evidence chain must connect the trigger, code path, observed behavior,
   violated contract, impact, validation, and minimal fix. Allow a non-inline
   `necessity` or `implementation_scope`
   blocker when the problem premise or solution path is not sufficiently
   established and no changed line is the right anchor. Allow a `solution_fit`
   blocker when the user pain is real but the chosen API, default, option,
   documentation, or runtime behavior is the wrong abstraction for that pain.
10. Verify each returned finding against the current PR diff and the current head
    SHA. Keep only concrete, actionable issues that can be anchored to changed
    lines. Discard style, preference, broad design commentary, or findings
    outside the diff unless the changed line directly creates the risk. Verify
    each returned necessity or implementation-scope blocker against the source
    issue/report, latest PR discussion, and current base code path; keep it only
    when the missing or contradicting evidence would make approval unsafe. Verify
    each returned solution-fit blocker against the intended user pain, public
    contract, alternative designs, and current implementation constraints; keep
    it when the submitted approach would leave a confusing or brittle contract
    even if the original symptom is real. Do not post a finding whose evidence
    chain is incomplete; gather more evidence or classify it as unclear.
11. For valid external-PR findings, submit a GitHub review with only inline file
    comments when a changed-line anchor exists. Follow the target repository's
    established review format; if none is specified, use concise English first
    and put the Chinese translation in
    `<details><summary>中文</summary>...</details>`.

    Every posted review comment that asserts a finding must carry its own
    compact evidence chain, not merely refer to evidence hidden in the report.
    Before posting, refresh and record the current head SHA, then write the
    chain in this order:

    - `Trigger`: concrete input, state, configuration, or precondition.
    - `Path`: file/symbol/line and the relevant call, data, or control flow.
    - `Observation`: the exact current-head behavior, output, or deterministic
      code-level proof.
    - `Contract`: the violated requirement, invariant, issue statement, docs,
      test expectation, or repository policy, with a source when available.
    - `Impact`: user-visible consequence, affected caller/configuration, and
      severity.
    - `Validation`: reproduction command, test, log, trace, or static proof,
      including the result and the head SHA checked.
    - `Request`: the smallest fix or decision that closes the finding and why it
      belongs in this PR.

    The labels above define required facts, not mandatory literal headings. A
    default inline finding should read like a direct reviewer sentence rather
    than an audit checklist:

    ```markdown
    `<severity>`: When `<trigger>`, `<component>` can `<bad behavior>`. In
    `<file:line>`, `<path>` leads to `<observation>`, which violates `<contract>`
    and causes `<impact>`. I verified this on HEAD `<sha>` with `<validation>`.
    Please `<minimal fix>` because it closes the PR's declared `<outcome>`.

    <details>
    <summary>中文</summary>

    `<严重级别>`：当 `<触发条件>` 时，`<组件>` 可能 `<错误行为>`。在
    `<file:line>`，`<代码路径>` 会导致 `<当前行为>`，违反 `<契约>` 并造成
    `<影响>`。我已在 HEAD `<sha>` 通过 `<验证方式>`确认。请 `<最小修复>`，
    因为它直接闭合本 PR 声明的 `<目标结果>`。

    </details>
    ```

    Prefer this compact prose for ordinary findings. For a complex race,
    cross-module flow, or unanchored blocker, use labeled fields or a short
    `Evidence: trigger -> path -> observation; Contract: ...; Impact: ...;
    Validation: ...; Request: ...` line instead. Keep the chain compact enough
    for an inline comment, but never remove facts needed to reproduce or audit
    the claim. The Chinese translation must preserve the concrete facts, not
    replace them with a generic summary. Store the full chain in the report's
    `evidence_chain` record as well. Do not post a top-level PR comment
    when an inline comment is available. If the only valid blocker is a
    problem-evidence, implementation-scope, or solution-fit blocker that cannot
    be anchored to a changed line, do not invent an anchor. Leave one concise
    top-level review comment carrying the same evidence-chain fields, ask for
    the missing reproduction/raw payload/log/current-base verification/better
    control surface/contract decision, and report the PR as blocked; do not
    approve or merge.
12. Before any approving review, refresh the PR head SHA, linked issue metadata,
    review threads, review decision, maintainer comments, and manual-review
    evidence. If `manual_review` still applies, do not approve. Record the
    blocker and the evidence that still requires a human decision.
13. If an external PR has no valid findings and is not blocked by
    `manual_review`, approve it with a GitHub pull-request review whose state is
    `approve` and whose body exactly follows the target repository or user
    policy. Use exactly `LGTM` only when that policy requires it. This must be
    the same visible effect as a human submitting an approving review from the
    GitHub UI, not a standalone issue comment.
14. After approval, refresh the current head SHA, effective checks,
    mergeability, required/soft-check state, `manual_review` state, and
    `merge_policy`
    before attempting a merge. If `merge_policy.mode` is `human_required`, do
    not merge; leave the PR approved or reviewed-clean as appropriate and report
    the required human merge evidence. If `manual_review` became applicable
    during the refresh, do not merge even if Neneko CR already approved earlier in
    the run.
15. Merge the external PR using the repository-supported merge method only when
    every required and merge-blocking check is green, the latest head still
    matches the reviewed head, and no `merge_policy` blocks Neneko CR from merging.
    Use the repository's allowed/default merge method; do not assume squash,
    rebase, or merge-commit semantics. Preserve the hosting provider's default
    generated merge title and body unless the user explicitly requests a custom
    message. Pass an expected head SHA when the tool supports it, and do not
    use a tool path that silently rewrites the repository's default merge
    metadata. If a soft or required check is still non-green, leave the PR
    unmerged and report the exact check, policy, and evidence.
16. Before ending, fetch each processed PR's latest comments and review threads
    again. For external PRs, if new actionable comments appeared and can be
    fixed by the author or require a different outcome, handle them before
    reporting; otherwise include them in the report. For own PRs, convert any
    newly discovered actionable issue into another direct branch fix when
    possible, then re-check CI and threads. If any PR head moved after comments
    or fixes, require CI to be all green again before approving or merging,
    except for documented soft CI failures that were explicitly re-evaluated
    after the head movement. A re-evaluated soft failure can still allow
    external review/approval only when repository policy permits it, but never
    merge while a required or merge-blocking check remains non-green.
17. Report all reviewed PRs, maintained own PRs, skipped PRs with reasons,
    comments posted, direct fixes, approvals, merge results, and any blocked
    operations. Generate the
    interactive HTML report described below before sending the final answer.
    Follow the detailed reporting contract below. Treat report writing as an
    xhigh-quality pass, not as a short run log: synthesize the PR content,
    implementation design, review judgment, and follow-up priorities so the
    user can understand the engineering state without opening GitHub or reading
    a diff.
18. Before committing or final-answering a meaningful run, run the post-report
    QA pass from the configured review workspace. Prefer the repository's
    existing archive/normalization/renderer commands when available; otherwise
    validate the structured report, check all required fields, and render the
    repository's supported artifact format (HTML, Markdown, or JSON). Run the
    target project's documented verification command when report or workflow
    code changed. Treat quality failures as workflow failures, not cosmetic
    warnings.
19. Inspect the generated report artifact, not only its source data. At minimum
    verify that reviewed PR entries, skipped groups, follow-up entries, inline
    comments with their evidence chains, and self-evolution notes are present
    and not empty placeholders. When a renderer or UI changed, use a browser or
    static DOM check when available to confirm the expected sections before
    publishing.

## Reporting Contract

In the final report:

- Use Chinese as the primary reporting and analysis language. When writing
  Chinese sections, summaries, skip reasons, follow-up judgments, timelines,
  and plain-text final answers, make them as fully Chinese as practical.
  Preserve original English only for source material that should stay exact,
  such as PR titles, code identifiers, file paths, check names, branch names,
  commit messages, quoted review bodies, and GitHub UI/status literals.
- Create a structured summary of the run in the repository's supported format
  (prefer JSON when machine-readable reporting is available) and render it
  with the configured repository-local renderer when one exists. A rendered
  report should be Chinese-first and switchable to English when the project
  supports bilingual output; it must expose status totals, reviewed PRs,
  skipped PR groups, comment details, evidence chains, CI/thread state, search,
  and status filtering when the chosen format supports those interactions.
- Include the incremental planner result or equivalent fields in the JSON
  summary as `incremental_plan`, and include the next run-state snapshot as
  `run_state`. These fields are the workflow's memory: they should record the
  lightweight open-PR index, planning reasons such as `head_sha_changed` or
  `checks_changed`, carried-forward blockers, and the latest successful
  verification time. The public report renderer may ignore these fields, but
  future Neneko CR runs must be able to use them to avoid repeating expensive work.
- Every posted finding represented in `inline_comments` must include an
  `evidence_chain` object with `head_sha`, `trigger`, `path`, `observation`,
  `contract`, `impact`, `validation`, and `request`. The same chain, or a
  faithful compact form of it, must appear in the actual GitHub review comment;
  a report-only explanation is not sufficient. If a field is unavailable,
  record the evidence gap and do not present the finding as proven.
- The HTML report should feel like a polished engineering dashboard, inspired
  by high-quality developer tools such as GitHub/Graphite review timelines,
  Linear-style dense status lists, Vercel-style deployment/status cards, and
  Raycast-style detail panes. Keep it self-contained, fast, responsive, and
  readable: clear hierarchy, restrained color, strong scanability, sticky
  search/filter controls, status dots, compact metrics, readable long-form PR
  details, and obvious reviewed / commented / maintained / skipped / follow-up
  sections.
  Avoid decorative clutter, marketing-page layout, or generic blog styling.
- The report must be a self-contained engineering brief, not merely an action
  log. A reader should be able to understand, from the report alone, what each
  reviewed PR changes, why the change exists, which modules and code paths are
  affected, how the implementation works, what behavior/API/docs/tests changed,
  why the review outcome was chosen, and which items still deserve attention.
  If this cannot be understood without opening the PR diff, the report is not
  good enough.
- Every reviewed PR entry must include beginner-friendly technical background
  for all important modules and concepts touched by the PR, not just the PR's
  immediate motivation. Explain the underlying technical principles from zero:
  what the module is responsible for, how it fits into the project's runtime,
  service, library, or application architecture, what data flows through it,
  and which lifecycle, API, state, storage, security, concurrency, or
  deployment concepts matter. When a PR touches multiple modules, explain each
  module's role and how they cooperate.
  The goal is that a reader unfamiliar with this area of the codebase can first
  learn the relevant domain model, then understand the PR.
- Use xhigh-level care when writing the report. Before rendering, make one
  dedicated pass over every reviewed PR entry and ask whether a maintainer who
  has not read the code could still explain the PR's goal, implementation,
  touched modules, behavior impact, test/docs coverage, and review risk. Expand
  any entry that fails that check.
- The overall report must include a rich top-level narrative: how many PRs were
  considered, how they were partitioned, which PRs changed code versus docs or
  examples, which areas of the repository were touched, what was merged, what
  was blocked by real findings, what was blocked by readiness gates, and what
  the maintainer should look at next. Avoid presenting only counters.
- In the structured JSON summary, fields named `problem`, `approach`, or shown
  as "解决的问题" / "实现方式" must describe the PR itself. Never fill these
  fields with review-process narration such as "ran a subagent", "checked CI",
  "reviewed the diff", "posted comments", or "resolved a thread". Put review
  actions only in `outcome`, `ci_state`, `risk`, `inline_comments`, timeline,
  or the final plain-text summary.
- For each reviewed PR, include `technical_background` (shown as "技术背景").
  This field is not a PR summary. It should teach the reader the module-level
  and concept-level background needed to understand the PR: core abstractions,
  responsibilities, request/response or control flow, state/lifecycle model,
  callback/event model, storage/runtime model, security boundary, serialization
  format, graph/query model, deployment model, or any other domain knowledge the
  PR depends on. Write it for an absolute beginner, but keep it directly tied
  to the touched code so it stays useful to maintainers.
- For "解决的问题", explain the concrete product, developer, API, runtime,
  compatibility, correctness, performance, or documentation gap the PR is
  trying to address. Include the old behavior or missing capability, why it
  matters, and the intended user-visible or maintainer-visible result. If the
  PR description does not state the problem directly, infer it from the diff
  and say that it is inferred. Do not treat an inferred problem statement as
  sufficient approval evidence by itself; separately record whether the source
  problem is confirmed, unproven, contradicted, stale, or awaiting a product or
  contract decision.
- For every reviewed PR, add a problem-framing and root-cause analysis before
  or near the implementation discussion. Explain what the PR appears to treat
  as the underlying problem, what deeper root cause may be driving that problem,
  and whether the same user/developer pain might be better understood from a
  different angle. If there is a more fundamental architecture, API,
  lifecycle, observability, compatibility, or product-model issue underneath
  the immediate bug or feature request, say so plainly.
- For every bugfix, behavior fix, compatibility fix, or linked-issue fix, add a
  problem-evidence section. Record what source problem report, PR discussion,
  logs, raw payloads, reproduction steps, base-branch behavior, tests, docs, or
  maintainer statements were checked. State whether the issue reproduces or is
  otherwise confirmed on the current target base. If evidence is missing,
  stale, contradictory, or only proves behavior on the PR branch, say that
  plainly and make it a blocker unless the PR is explicitly reframed as a
  design/product change.
- For "实现方式", describe the PR's design and implementation in enough detail
  that the user can understand the change without opening the PR diff. Mention
  the main packages/files touched, new public APIs or options, changed defaults,
  important algorithms or control flow, persistence/session/streaming behavior,
  error handling, compatibility behavior, docs/examples, and test coverage when
  relevant. The goal is for the report reader to judge whether the PR's design
  and implementation direction look reasonable before reading code.
- For every reviewed PR, analyze the solution space, not only the submitted
  implementation. Describe the design approach the PR chose, other plausible
  approaches the author could have taken, and the tradeoffs among them. Discuss
  dimensions such as API simplicity, backward compatibility, correctness,
  safety, maintainability, extensibility, observability, runtime cost, test
  scope, migration cost, and how much complexity is pushed onto users.
- For every reviewed PR, include a solution-fit assessment. Distinguish the
  user pain or product need from any implementation suggested by the issue,
  bot, or author. State whether the PR chose the right control surface: code
  behavior, public API, option, default, documentation, example, migration note,
  or a separate design decision. If the PR satisfies the literal issue request
  but would create a confusing mode switch, misleading option semantics,
  weaker default, hidden behavior change, or inappropriate operational
  guarantee, treat that as a blocker or explicit residual risk.
- For every reviewed PR, explain implementation derivation: why the chosen code
  path is the right place to solve the confirmed problem, why the patch scope is
  neither too broad nor too narrow, and which evidence connects the observed
  symptom to this implementation. If the PR appears to patch a guessed cause,
  broadens semantics beyond the reported scenario, or masks an upstream/proxy
  contract violation without a stated compatibility policy, treat that as a
  design blocker or follow-up risk depending on severity.
- For every reviewed PR, include a design assessment. State whether the PR's
  chosen design is a good fit for the root problem, whether it reaches a
  reasonable local optimum across the relevant tradeoffs, and whether there may
  be a better architecture or simpler solution. If a better approach exists,
  explain it concretely enough for the maintainer to judge whether to ask the
  author for redesign, a follow-up PR, or no change.
- For each reviewed PR, include a concrete change inventory in the report body.
  Cover newly added symbols/options/files, modified behavior, removed or renamed
  behavior, updated docs/examples, and tests that prove the change. When the PR
  touches multiple modules, group the explanation by module or execution path
  instead of collapsing everything into one generic sentence.
- For each reviewed PR, explicitly describe the exported API surface change.
  List newly added, modified, renamed, or removed exported variables,
  constants, functions, methods, interfaces, structs, struct fields, types,
  options, constructors, tool names, config keys, environment variables, command
  names, JSON fields, document sections, and examples when they are part of the
  user-facing or maintainer-facing contract. If no exported/user-visible API
  changed, say that explicitly and explain why the change is internal.
- For each reviewed PR, explicitly describe semantic changes and module impact.
  Explain behavior changes, default changes, compatibility or migration impact,
  error-handling changes, lifecycle/session/streaming/persistence changes,
  concurrency or ordering changes, and performance or resource-use implications.
  Also explain which modules are directly affected and which other modules may
  be indirectly affected through shared interfaces, callbacks, tools, runtime
  contracts, docs/examples, or tests.
- Explain behavior from first principles. Prefer before/after descriptions,
  request/response or call-flow narratives, and small concrete examples when
  they make the change easier to understand. Define local concepts briefly when
  they are not obvious from the PR title.
- Make attention points explicit. Separate "must fix before merge" findings
  from "watch this later" residual risks, soft-CI judgments, compatibility
  notes, migration concerns, and reviewer assumptions. Do not hide these only
  inside inline comment text.
- Avoid low-information phrases such as "updates logic", "improves handling",
  "adds support", "refactors code", or "reviewed implementation" unless they
  are immediately followed by the concrete modules, code paths, data structures,
  or behaviors involved.
- Separate PR-content summary from review judgment. Design concerns found by
  the reviewer belong in `risk` or `inline_comments`; the `problem` and
  `approach` fields should remain a faithful explanation of the author's PR.
- In the HTML report, PRs that were not reviewed must be shown in a grouped
  section by exclusion reason, not only as a flat list. Each group should show
  the group reason, count, PR links, authors, concrete blockers, CI state, and
  any follow-up judgment.
- Store generated run reports under `CR_REPORT_ROOT`, the review workspace's
  existing archive, or the target repository's documented report directory.
  Prefer a timestamped, machine-readable summary plus a rendered artifact when
  the repository supports them; keep the target checkout's unrelated local
  state untouched. The report filename and archive layout must follow the
  target project's convention rather than a hard-coded project name.
- Treat meaningful report artifacts as auditable history. After a completed
  run, run the available verification command and commit/push report files only
  when the user or repository workflow explicitly authorizes that publication.
  Otherwise leave the report local and state that publication was not
  authorized or not possible.
- Include a clickable absolute local report path, or the published report URL
  when one exists, in the final answer. The plain Markdown answer should
  summarize the result, while the rendered artifact remains the detailed audit
  trail.
- Every PR reference must be a clickable Markdown link whose visible text is the
  PR number, for example `[#123](https://github.com/owner/repo/pull/123)`.
  Do not leave bare `#123` text anywhere in the report.
- Present the final report in this order:
  1. PRs approved under the repository's approval policy first, including the
     merge result or the exact required/soft-check blocker when approved but
     intentionally not merged.
  2. PRs that received inline comments second.
  3. Own PRs maintained directly, including any self-review record, commits
     pushed, tests run, CI state, and remaining external-review expectations.
  4. PRs that were not reviewed, grouped by the reason they were excluded.
- For every reviewed or maintained candidate PR, include:
  - author
  - title
  - beginner-friendly technical background for every important module and
    concept touched by the PR, including how those modules work and cooperate
  - what problem the PR tries to solve
  - necessity assessment: whether the source problem is confirmed on the
    current target base, plausible but unproven, contradicted/stale, or waiting
    on a product or contract decision
  - evidence checked: linked problem reports, discussion, logs, raw payloads,
    reproduction steps, current-base behavior, tests, docs, or maintainer
    statements used to validate the premise
  - reproduction on base: whether the problem was reproduced or otherwise
    verified against the latest target base, and any reason reproduction was not
    practical
  - evidence gaps: missing data that prevents a confident approval, such as raw
    event payloads, authoritative provider behavior, or a minimal current-base
    reproduction
  - the problem framing: what root cause the PR is addressing and whether the
    problem should be viewed from another angle
  - the implementation approach used by the PR
  - implementation derivation: why the edited code path and patch scope follow
    from the confirmed problem evidence
  - solution-fit assessment: whether the chosen API/default/config/docs/control
    surface is the right way to address the underlying user pain, not just the
    literal requested fix
  - plausible alternative designs or implementation strategies
  - the key tradeoffs between the PR's design and those alternatives
  - a design assessment: whether the PR's chosen approach is close to optimal,
    merely acceptable, over-engineered, under-designed, or likely needs a
    different architecture
  - the main modules, packages, files, or code paths changed by the PR
  - exported API surface changes: exported vars, consts, funcs, methods,
    interfaces, structs, fields, types, options, constructors, tool names,
    config keys, env vars, JSON fields, docs/examples, or explicit "none"
  - the specific APIs, options, data structures, behavior, docs, examples, or
    tests added, modified, or removed by the PR
  - semantic changes: defaults, runtime behavior, compatibility, migration,
    errors, lifecycle/session/streaming/persistence, concurrency, ordering,
    performance, or resource use
  - module impact and cross-module impact: direct package changes and indirect
    effects on callers, adapters, tools, examples, docs, tests, and shared
    contracts
  - the expected before/after behavior from the user's or maintainer's point of
    view
  - review outcome: commented, approved, merged, maintained, skipped after
    recheck, or blocked
  - possible remaining issues or compatibility risks
  - if approved, why it is acceptable to approve and whether any residual risk
    remains
  - if approved while a check is non-green, state whether it is a permitted soft
    failure or a merge blocker, why it was accepted for review, and why it was
    or was not merged
  - if maintained as an own PR, state explicitly whether a self-review comment
    or self-approval was submitted, that no self-merge was performed, and list
    the branch/commits pushed or say that no direct fix was needed
  - if comments were posted, the exact inline comment bodies, the review focus,
    severity, whether the user should pay extra attention, and the complete
    `evidence_chain` for each comment (`head_sha`, `trigger`, `path`,
    `observation`, `contract`, `impact`, `validation`, and `request`)
  - latest CI and review-thread state when it affects the outcome
  - `merge_policy` when Neneko CR intentionally leaves an otherwise reviewed PR
    unmerged because a human must perform the final merge; include the mode,
    reason, required evidence, and the current evidence checked
- For PRs that were not reviewed, group them by skip reason and still use
  clickable PR links. For each PR, include the author, title, concrete blocker,
  and whether the blocker is CI, draft/WIP/own PR state, unresolved human review,
  a still-valid bot finding, or another readiness issue.
- For each not-reviewed PR, include `blockers` when there is any blocker that
  can be described as a discrete fact. Each blocker should have `kind`
  (`ci`, `human_review`, `bot_review`, `merge_conflict`, `draft_wip`,
  `own_pr`, `soft_ci`, `necessity`, `insufficient_evidence`, `stale_issue`,
  `implementation_scope`, `solution_fit`, `manual_review`, `not_reached`, or
  `other`),
  `summary`, and, when applicable, `reviewer`, `url`, `path`, `line`,
  `latest_response`, and `verification`. A skipped item with
  `reviewDecision=CHANGES_REQUESTED` but no live human-review blocker must say
  that explicitly, then list the real remaining gate, such as a configured
  soft-check inspection, required-check/merge blocker, missing source-problem
  evidence, wrong solution/control surface, manual maintainer selection
  required, CI pending, or "not reached this run".
- A PR blocked by `manual_review` may also have `merge_policy`, but do not use
  `merge_policy` to hide the decision blocker. `manual_review` explains why
  Neneko CR cannot decide or approve the PR; `merge_policy` explains why Neneko CR cannot
  perform the final merge action.
- If a skipped group is about human review, each item in that group must name
  the exact unresolved human thread. If no exact still-actionable human thread
  is found, the PR does not belong in that group.
- After the not-reviewed section, call out which not-reviewed PRs may need
  follow-up attention and why, such as flaky/failing CI, unresolved review
  threads from important reviewers, long-stale high-impact changes, or blocked
  PRs that look close to ready.
- Keep the report readable, but prefer useful detail over brevity. The report is
  allowed to be long when the PRs are substantial. The quality bar is that the
  user can understand the relevant code-related information, implementation
  direction, and review outcome without opening GitHub or reading the PR code
  first.

## Self-Evolution Loop

- Treat the configured review workspace or skill repository as the workflow's
  long-term memory and implementation home. An installed copy of this skill
  should remain a portable bootstrap and must not assume a particular user's
  home directory or target project.
- At the end of each real Neneko CR run, capture concrete friction, repeated manual
  work, missed edge cases, brittle path assumptions, weak report fields,
  confusing UI, or unclear review gates in the review workspace's configured
  evolution/backlog file (for example, `data/evolution/backlog.md` when that
  convention exists).
- Treat report QA as part of self-evolution. Empty rendered fields, review
  actions written into PR-content fields, non-canonical display statuses,
  skipped groups without per-PR blockers, follow-up entries that render as
  blank cards, or missing HTML-visible self-evolution notes must produce a
  local fix or an explicit backlog item before the run is considered done.
- When the user explicitly asks for an independent reviewer, or the active tool
  policy otherwise permits it, spawn a fresh xhigh reviewer after the report is
  generated. The reviewer should inspect the latest JSON/HTML and Neneko CR flow
  read-only, then return concrete quality gaps and evolution suggestions. The
  main agent must validate those suggestions, implement low-risk local fixes,
  and record longer-term work in the configured evolution/backlog location.
- If subagent tools are unavailable or not permitted by the active tool policy,
  do the same checklist locally and record the limitation as a run capability
  or `skill_evolution` note so the report remains auditable.
- When the user asks for Neneko CR maintenance, or when a low-risk improvement
  is clearly local to the review workspace, update the relevant canonical skill,
  scripts, report, specification, or evolution files. Run the workspace's
  documented verification command and publish/commit only when authorized.
- Prefer environment overrides and repo-relative paths. Document the rare
  absolute fallback in `README.md`, `AGENTS.md`, or this skill instead of
  scattering hardcoded paths through scripts.
- Keep self-evolution changes separate from target PR review judgments. Do not
  let a skill improvement modify the target repository unless the user asked
  for that target-repo change.
- For durable design decisions, add a dated note under the configured
  evolution directory so future runs can explain why the skill or report
  workflow changed.

## Practical Notes

- If `gh` has no valid GitHub auth, use the GitHub connector instead of blocking
  unless a required action is unavailable there.
- Incremental planning is an optimization, not a correctness shortcut. The
  cheap full-open-PR index must still be refreshed every run, and any missing
  or suspicious planner input must make the run fall back to a conservative
  heavy scan. Before submitting review comments, approving, merging, or pushing
  own-PR fixes, refresh the PR's head SHA, checks, comments, and review threads
  even if the incremental plan selected it correctly.
- Use the configured incremental planner, if the review workspace provides one
  (for example, a repository-local planner script), to compare the current
  lightweight index with the latest archived report's `run_state`. The planner
  only decides which PRs need expensive work; it does not decide review
  outcomes. Treat `carry_forward` entries as reportable unchanged blockers,
  not as permission to approve or merge.
- Run a periodic lightweight freshness probe when the prior state is older than
  the planner's `--force-full-sweep-hours` threshold, or sooner if GitHub
  metadata looks inconsistent. The default threshold is 24 hours, but the
  default action is `probe`, not full heavy review. A freshness probe refreshes
  only cheap index fields and reuses the full-metadata cache if the
  `metadata_cache_key` stays unchanged. Use full heavy sweep only for corrupt
  metadata, missing fingerprints, explicit user requests, or changed cache
  keys.
- For CI, check both combined commit statuses and GitHub Actions workflow runs.
  Treat pending, queued, in-progress, cancelled, timed out, skipped unexpectedly,
  or missing required checks as not ready.
- Soft CI exception: when the only non-green checks are configured as soft,
  inspect each concrete failure instead of skipping by rule. Fetch check
  details/logs or reproduce the relevant behavior locally when practical, then
  decide whether the failure is intentional, documented, and acceptable for
  the PR. Review or approval can proceed only when repository policy permits it
  and the failure does not hide regression risk. Merge must not proceed while
  any required or merge-blocking check remains non-green. Record the soft-CI
  judgment and any approval-without-merge decision in the report.
- Human review comments can be considered addressed when the author clearly
  replied `fixed`, `done`, or gave a concrete acceptable explanation, even if
  the GitHub thread is still unresolved. Verify the latest diff before relying
  on the reply.
- Do not let a stale aggregate `reviewDecision=CHANGES_REQUESTED` hide a
  reviewable PR. Always enumerate the current human threads. A PR with no
  still-actionable human thread should remain reviewable when CI is green or
  only acceptable soft CI remains, even if GitHub still shows
  `CHANGES_REQUESTED` from an earlier review.
- Bot or automated review comments are advisory. Do
  not let weak, stale, stylistic, or unreasonable bot comments block review.
  Let them block only when our own inspection confirms a current, concrete bug,
  compatibility risk, or CI/test issue.
- "WIP" includes labels or title markers such as `WIP`, `[WIP]`, and draft PRs.
- "Ready for review" means not draft, not WIP, CI green or only acceptable
  soft CI failures, source-problem evidence is sufficient for the claimed
  fix/design change, the chosen solution/control surface is a reasonable fit
  for the underlying need, the PR is not under a `manual_review` gate,
  and the PR is not blocked by a still-actionable human or verified bot review
  issue.
- "Ready for merge" additionally means the latest pre-merge refresh found no
  `manual_review` blocker and no `merge_policy` that requires a human to
  perform the merge.
- Never merge a PR that received new inline findings during this run.
- Never merge a PR while a required or merge-blocking check is non-green, even
  if it has no review findings and has been approved.
- Never override the hosting provider's default merge commit message unless
  the user explicitly requests it. In particular, do not replace generated PR
  identifiers or repository-required metadata with the raw PR title.
- Keep the final answer concise but include enough detail for the user to audit
  what happened.

## Subagent Prompt

Use a prompt shaped like this for each candidate:

```text
Review <owner>/<repo> PR #<number> at the configured depth (default xhigh).
The target project may use any language, framework, build system, or review
policy; inspect its local conventions before judging the change.

First assess whether the PR's problem premise is established. Check the linked
issue or source problem report, PR description and discussion, available logs or
raw payloads, tests, and current base-branch code path. State whether the
problem is confirmed on the current base, plausible but unproven,
contradicted/stale, or waiting on a product/contract decision. Explain why the
edited code path follows from that evidence, or why it does not.

Separate the underlying user pain from any implementation proposed in the issue
or PR. Assess whether the selected API, default, configuration option,
documentation, or runtime control surface is the right fit, or whether a
different design would preserve a clearer contract.

Then focus only on changed behavior in this PR and directly related code.
Prioritize design soundness, API and semantic compatibility, potential bugs,
data races, lifecycle/concurrency/ordering semantics, cross-module impact, and
test gaps that expose real regression risk.

For every possible inline finding, build an evidence chain tied to the current
head SHA. Return:
- trigger/input/state that activates the issue
- exact path (file, symbol, changed-side line or diff position) and relevant flow
- observed behavior or deterministic proof on the current head
- violated contract/invariant and its source
- user/caller impact and severity
- validation command/test/log/static proof and result
- smallest fix or decision requested, including why it belongs in this PR

Do not post comments, approve, merge, or edit files. Return only:
- "no findings" if the PR is clean
- or a `necessity` / `implementation_scope` blocker with the missing or
  contradicting evidence and the smallest evidence needed to proceed
- or a `solution_fit` blocker when the problem is real but the chosen control
  surface, default, option, or documentation contract is not a good design fit
- or a list of findings with path, changed-side line or diff position, severity,
  exact problem, impact, minimal fix, and the complete evidence chain. Include
  an English body and Chinese translation for findings that may be posted.

Only report code issues that would justify an inline review comment. Necessity,
implementation-scope, or solution-fit blockers may be unanchored when no changed
line is the right place to ask for missing source-problem evidence or a better
contract decision.
```
