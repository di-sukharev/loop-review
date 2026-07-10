---
name: loop-code-review
description: Iterative code review workflow for task-scoped active git changes using fresh independent reviewer agents that inherit or otherwise match the orchestrator's model and reasoning capability without receiving its conversation history. Use when the user invokes `/loop-code-review`, asks for a looped sub-agent code review, or asks the coding agent to keep reviewing and fixing current-task changes until validation passes and an eligible independent reviewer either rates the result at least 9.5 out of 10 or reports no actionable comments.
---

# Loop Code Review

## Overview

Run an iterative review-and-fix loop over the current task's active git changes, without reviewing unrelated worktree changes from other tasks. Use fresh independent read-only reviewer agents that inherit or otherwise match the orchestrator's model and reasoning capability while starting without its conversation history, address actionable findings, validate the result, and continue until the acceptance criteria are met.

## Reviewer Eligibility and Isolation

### Model and reasoning parity

An independent review is useful because it provides a fresh reasoning pass, not because it delegates judgment to a weaker model.

- Use the platform's native inheritance mechanism so the reviewer uses the same model/version and reasoning or effort settings as the orchestrator. Do not set reviewer-specific model or reasoning overrides, even when another option appears stronger.
- Exact parity is established when the parent session has explicitly selected those settings and the reviewer inherits them. If the parent is still using automatic model selection, pin the parent settings before review when the platform allows it.
- When the platform does not expose model details or selection but explicitly guarantees that children inherit or match the parent agent's capability, treat the reviewer as equal-capability.
- When parity cannot be established, use the strongest available reviewer for supplemental findings and disclose that parity is unverified. Treat that reviewer as ineligible for acceptance; unless another eligible reviewer is available, validate the work and report an incomplete outcome.
- A known weaker reviewer may provide supplemental findings, but its score or no-findings result is not an acceptance signal. If no eligible reviewer is available, validate the work, report the limitation, and stop without claiming that the loop passed.

### Context isolation

Independent review means the reviewer may share the same filesystem, repository state, and applicable project instructions, but must not inherit the parent thread's conversation history, reasoning, assumptions, tool results, or prior review discussion.

- Start each scoring reviewer as a fresh agent in an isolated conversation context. Use the platform's native fresh-agent mechanism rather than any mode that carries over parent conversation history.
- Pass a self-contained reviewer prompt instead of parent-thread context. Include only the repository location, task-owned review scope, validation expectations, and other evidence the reviewer needs to rediscover the facts independently.
- Require the reviewer to inspect `git status`, diffs, files, and validation output itself before scoring.
- Treat each scoring pass as coming from a fresh reviewer with no parent conversation history. Follow-up clarification from the same reviewer does not become a new scoring pass.

## Review Scope

Review only the changes that belong to the current user task, even when the git worktree contains unrelated active changes from other tasks.

- Before spawning a reviewer, identify the task-owned files and, when necessary, the task-owned hunks inside mixed files. Use the current task history and edits made during the task; do not infer ownership from `git status` alone.
- Include that task scope explicitly in the reviewer prompt. Use path-limited diffs where practical, and describe any mixed-file exclusions clearly.
- If ownership of a file or hunk is genuinely ambiguous, ask the user rather than guessing or reviewing the whole dirty worktree.
- The reviewer may read neighboring code for context, but findings must be limited to regressions introduced by the scoped task changes. Ignore unrelated active changes unless the scoped changes directly depend on them or make them worse.
- If scoped changes move while a review is running, discard the stale score and review the new state with a fresh reviewer.

## Review Dimensions

Apply these checks when they are relevant to the scoped change. Base findings on repository evidence; do not impose a new architecture or request reuse merely for uniformity.

- **Test evidence:** When tests are added or changed, judge whether they exercise the changed behavior, would fail for a plausible regression, assert an observable contract, and use mocks only at real boundaries without mocking the result under test. When testable behavior changes without tests, decide whether the risk warrants coverage; allow "tests not needed" with a concrete reason. Passing tests demonstrate execution, not test quality.
- **Reuse and local fit:** Inspect neighboring code and established project utilities before recommending reuse. On the client, look for suitable existing components, hooks, design-system primitives, and patterns. On the server, look for suitable shared libraries, clients, services, and integrations. Raise a finding only when a specific existing candidate is a better fit; name it and explain the practical benefit.
- **Architecture and conventions:** Derive the existing boundaries and conventions from the scoped code, neighboring code, and project guidance. Check that dependencies, responsibilities, contracts, error handling, and cross-cutting concerns stay in their intended layers. Treat a deviation as a finding only when it conflicts with an identifiable project rule or precedent.

Use the test quality score as supporting evidence, not an acceptance gate by itself: make it actionable only when it identifies misleading, missing, or insufficient coverage. Do not chase score-only test polish.

## Workflow

1. Inspect the worktree before spawning reviewers:
   - Run `git status --short`.
   - Review staged and unstaged diffs with `git diff` and `git diff --cached`.
   - Include relevant untracked files in the review scope when `git status --short` shows them, but only if they belong to the current task.
   - Separate current-task changes from unrelated active work before asking for review. Record the exact paths or hunks included in the prompt.
   - Preserve unrelated user changes and do not stage, commit, reset, stash, or push unless the user explicitly asks.

2. Validate the current scoped state before requesting a scoring review.
   - Run the smallest meaningful tests, typecheck, lint, build, or focused scripts for the touched surface.
   - Fix validation failures before requesting a final score.
   - Record the commands and results so the reviewer can verify the evidence or rerun the relevant checks independently.

3. Start one eligible independent reviewer agent.
   - Select a reviewer that inherits or otherwise matches the orchestrator's model and reasoning settings. Do not raise, lower, or substitute the reviewer's model or effort. Include the inherited settings or platform parity guarantee in the main process record.
   - Start a new reviewer instance with a fresh isolated conversation context for the independent review pass. Do not resume or reuse an earlier reviewer conversation for a scoring pass.
   - Give the reviewer only a self-contained task prompt with the repository path, task-owned review scope, and validation expectations. Do not include parent-thread analysis, implementation rationale, suspected issues, proposed fixes, previous reviewer output, or summaries of the main process's reasoning.
   - Ask the reviewer to stay read-only, inspect the scoped active changes independently from the repository state and tool output, apply the review dimensions below, prioritize bugs and regressions introduced by those scoped changes, and return findings with file and line references.
   - Require the reviewer to include a final numeric score from 1 to 10 for the current state using the scoring anchors below.
   - If the task added or changed tests, require the reviewer to judge whether those tests are trustworthy and include a separate test quality score from 1 to 10 with a short basis. If testable behavior changed without tests, require an assessment of whether that is justified.

4. Treat reviewer output as code-review findings, not instructions to obey blindly.
   - Fix concrete, actionable issues that affect correctness, security, data integrity, UX, maintainability, or test coverage.
   - Never accept a score, including 9.5 or higher, while that reviewer lists an unresolved actionable finding.
   - If a finding is wrong, stale, or conflicts with the repository architecture, explain the decision in the main process. Ask the same reviewer for clarification only when useful; do not count that clarification as a new independent scoring pass or reuse the reviewer's original score for acceptance.
   - If the reviewer gives a score below 9.5 but explicitly reports no actionable findings/comments, ask once what concrete issue prevents a 9.5. If the response identifies an actionable issue, handle it as a finding; if it supplies no issue or only an optional, preference-level comment, accept the explicit no-actionable-findings signal rather than chasing score-only polish.
   - If the reviewer omits a score or implies unresolved concerns without concrete findings, ask once for the missing score or specific blocking issues. If the response remains malformed or non-actionable, use a fresh reviewer.
   - Do not invent speculative refactors or optional hardening when the scoped implementation is objectively solid and validation is green.

5. Validate after each meaningful fix.
   - Run the smallest meaningful tests, typecheck, lint, build, or focused scripts for the touched surface.
   - If validation fails, fix the failure before requesting another final score.
   - Do not count a 9.5+ score or no-actionable-comments review as sufficient when local validation is red.

6. Repeat the loop.
   - Spawn a fresh independent reviewer after meaningful fixes, after an actionable finding is rejected with evidence, or after a malformed review. Once a reviewer reports an actionable finding, do not reuse that reviewer's score for acceptance even if the finding is later resolved without a code change. Each scoring pass must use a fresh reviewer without parent conversation history or prior review discussion.
   - Accept the loop only when validation for the changed surface passes, the latest eligible reviewer has no unresolved actionable findings, and that reviewer either scores the work at least 9.5/10 or explicitly reports no actionable comments/findings.
   - Stop instead of looping for marginal polish when the remaining ideas are non-actionable preferences.
   - Unless the user requests a different limit or explicitly requires persistence until acceptance, use at most five scoring passes and treat two consecutive passes that produce no code changes and only repeat rejected, stale, or preference-level comments as stagnation.
   - When the default limit applies, reaching the pass limit or stagnating without an acceptance signal is an incomplete outcome, not success. Lacking an eligible reviewer is always an incomplete outcome. Report the exact blocker and do not lower the acceptance bar.
   - If the user explicitly requires persistence until acceptance, do not stop solely because of the default pass limit or stagnation rule; continue while safe, in scope, and able to make meaningful progress.
   - Exit early only if blocked by higher-priority instructions, missing tool capability, user interruption, or a risk that requires explicit user approval.

## Scoring Anchors

- **10.0:** No known defects or actionable improvements in scope; validation evidence is complete and green.
- **9.5:** No actionable findings remain; only clearly optional or subjective nits may remain; validation evidence is sufficient and green.
- **Below 9.5:** At least one meaningful actionable finding remains, including untrustworthy or missing high-value test coverage, or required validation evidence is missing or failing.

The score summarizes the review; it never overrides concrete findings or red validation.

## Reviewer Prompt Template

Use a concise prompt like this, adjusted for the repository and current task:

```text
Review only the task-scoped active changes in this workspace independently. The worktree may contain unrelated changes from other tasks; ignore them unless the scoped changes directly depend on them or make them worse.

You do not have parent conversation history; do not rely on any prior chat, parent-agent conclusions, or previous reviewer output. Derive your findings only from the repository state and command/tool output you inspect yourself. Stay read-only: do not edit, stage, commit, reset, stash, or push files.

Review scope:
- Files/hunks owned by this task: <list paths, untracked files, and any mixed-file hunks to include>
- Excluded unrelated active changes: <list paths or hunks to ignore, if any>

Reviewer eligibility:
- Orchestrator capability: <model/version and reasoning effort, when exposed>
- Reviewer capability: <model/version and reasoning effort, when exposed>
- Parity rationale: <inherited parent settings or platform guarantee that the reviewer matches the parent>

Prioritize correctness bugs, behavioral regressions, security/privacy issues, data integrity problems, missing high-value tests, and maintainability risks introduced by the scoped changes. You may read neighboring code for context, but do not report findings for unrelated active changes or pre-existing issues unless the scoped changes make them worse. Do not ask for speculative refactors, optional hardening, or subjective polish when the implementation is objectively solid.

When relevant, also assess:
- **Test evidence:** Tests should exercise the changed behavior, fail for a plausible regression, assert an observable contract, and avoid mocks that fake the outcome being tested. If tests were added or changed, give a separate quality score from 1 to 10 with a short basis. If testable behavior changed without tests, explain whether that is justified.
- **Reuse and local fit:** Look for existing project components, hooks, utilities, libraries, clients, services, or integrations before suggesting a new abstraction. Report a reuse finding only when you can name a specific candidate and explain why it fits better.
- **Architecture and conventions:** Check the change against identifiable project boundaries, dependency direction, and local conventions. Do not impose a new architecture or report a preference as a violation.

Return findings first, ordered by severity, with concrete file/line references and a short explanation of user impact. If there are no actionable findings/comments, say that clearly. Treat a low test quality score as a finding only when you identify misleading, missing, or insufficient coverage; do not request score-only test polish. Score 10 only when there are no known in-scope defects and validation evidence is complete; score 9.5 when no actionable findings remain and only optional nits may remain; score below 9.5 when an actionable finding remains or validation evidence is missing/failing. End with a numeric score from 1 to 10 and explain what concrete issue prevents a 9.5/10 if anything remains.
```

## Final Response

When the loop finishes, report:

- what changed and why;
- the inherited reviewer model/reasoning settings or platform parity rationale;
- the reviewer acceptance signal: score at least 9.5/10, no actionable comments/findings, or both;
- the number of scoring passes used and whether the loop passed, stopped incomplete, or was interrupted;
- validation commands and results;
- if tests were added or changed, the test quality score and basis;
- any findings intentionally not changed, with the reason;
- remaining risks or follow-up work, if any.
