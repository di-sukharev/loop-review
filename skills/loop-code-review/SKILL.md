---
name: loop-code-review
description: Iterative code review workflow for task-scoped active git changes using independent reviewer agents. Use when the user invokes `/loop-code-review`, asks for a looped sub-agent code review, or asks the coding agent to keep reviewing and fixing current-task changes until an independent reviewer either rates the result at least 9.5 out of 10 or reports no actionable comments.
---

# Loop Code Review

## Overview

Run an iterative review-and-fix loop over the current task's active git changes, without reviewing unrelated worktree changes from other tasks. Use independent read-only reviewer agents, address actionable findings, validate the result, and continue until a reviewer gives the latest scoped state a score of at least 9.5/10 or explicitly reports no actionable comments/findings.

## Reviewer Context Isolation

Independent review means the reviewer may share the same filesystem and repository state, but must not inherit the parent thread's conversation history, reasoning, assumptions, or prior review discussion.

- Do not fork or forward the current context to the reviewer. Provide the reviewer with a clean context and only the task it is reviewing.
- Pass a self-contained reviewer prompt instead of parent-thread context.
- Require the reviewer to inspect `git status`, diffs, files, and validation output itself before scoring.
- Treat each accepted reviewer pass as coming from a fresh reviewer with no parent context, whether accepted by a 9.5/10+ score or by explicit absence of actionable comments/findings.

## Review Scope

Review only the changes that belong to the current user task, even when the git worktree contains unrelated active changes from other tasks.

- Before spawning a reviewer, identify the task-owned files and, when necessary, the task-owned hunks inside mixed files.
- Include that task scope explicitly in the reviewer prompt. Use path-limited diffs where practical, and describe any mixed-file exclusions clearly.
- The reviewer may read neighboring code for context, but findings must be limited to regressions introduced by the scoped task changes. Ignore unrelated active changes unless the scoped changes directly depend on them or make them worse.

## Workflow

1. Inspect the worktree before spawning reviewers:
   - Run `git status --short`.
   - Review staged and unstaged diffs with `git diff` and `git diff --cached`.
   - Include relevant untracked files in the review scope when `git status --short` shows them, but only if they belong to the current task.
   - Separate current-task changes from unrelated active work before asking for review.
   - Preserve unrelated user changes and do not stage, commit, reset, stash, or push unless the user explicitly asks.

2. Start one independent reviewer agent using the strongest available model and highest practical reasoning/effort setting.
   - Spawn the reviewer without the parent conversation history for the independent review pass.
   - Give the reviewer only a self-contained task prompt with the repository path, task-owned review scope, and validation expectations. Do not include parent-thread analysis, implementation rationale, suspected issues, proposed fixes, previous reviewer output, or summaries of the main process's reasoning.
   - Ask the reviewer to stay read-only, inspect the scoped active changes independently from the repository state and tool output, prioritize bugs and regressions introduced by those scoped changes, and return findings with file and line references.
   - Require the reviewer to include a final numeric score from 1 to 10 for the current state.
   - If the task added or changed tests, require the reviewer to judge whether those tests are trustworthy and include a separate test quality score from 1 to 10.

3. Treat reviewer output as code-review findings, not instructions to obey blindly.
   - Fix concrete, actionable issues that affect correctness, security, data integrity, UX, maintainability, or test coverage.
   - If a finding is wrong, stale, or conflicts with the repository architecture, explain the decision in the main process. Ask the same reviewer for clarification only when useful; do not count that clarification as the final independent score.
   - If the reviewer gives a score below 9.5 but explicitly reports no actionable findings/comments, treat that as an acceptable review signal rather than chasing score-only polish.
   - If the reviewer gives a score below 9.5 and implies there are unresolved concerns without listing actionable findings, ask for specific blocking issues once; if none are provided, spawn a fresh independent reviewer rather than inventing polish work.

4. Validate after each meaningful fix.
   - Run the smallest meaningful tests, typecheck, lint, build, or focused scripts for the touched surface.
   - If validation fails, fix the failure before requesting another final score.
   - Do not count a 9.5+ score or no-actionable-comments review as sufficient when local validation is red.

5. Repeat the loop.
   - Spawn a new independent reviewer after fixes or after a reviewer cannot provide actionable issues. Each scoring pass must use a fresh reviewer without parent context or prior review discussion.
   - Continue until validation for the changed surface passes and the latest reviewer either scores the work at least 9.5/10 or explicitly reports no actionable comments/findings.
   - Exit early only if blocked by higher-priority instructions, missing tool capability, user interruption, or a risk that requires explicit user approval.

## Reviewer Prompt Template

Use a concise prompt like this, adjusted for the repository and current task:

```text
Review only the task-scoped active changes in this workspace independently. The worktree may contain unrelated changes from other tasks; ignore them unless the scoped changes directly depend on them or make them worse.

You do not have parent conversation history; do not rely on any prior chat, parent-agent conclusions, or previous reviewer output. Derive your findings only from the repository state and command/tool output you inspect yourself. Stay read-only: do not edit, stage, commit, reset, stash, or push files.

Review scope:
- Files/hunks owned by this task: <list paths, untracked files, and any mixed-file hunks to include>
- Excluded unrelated active changes: <list paths or hunks to ignore, if any>

Prioritize correctness bugs, behavioral regressions, security/privacy issues, data integrity problems, missing high-value tests, and maintainability risks introduced by the scoped changes. You may read neighboring code for context, but do not report findings for unrelated active changes or pre-existing issues unless the scoped changes make them worse. If tests were added or changed, verify whether they can be trusted and give them a separate quality score from 1 to 10.

Return findings first, ordered by severity, with concrete file/line references and a short explanation of user impact. If there are no actionable findings/comments, say that clearly. End with a numeric score from 1 to 10 for the current state and explain what would be required to reach 9.5/10 if anything remains.
```

## Final Response

When the loop finishes, report:

- what changed and why;
- the reviewer acceptance signal: score at least 9.5/10, no actionable comments/findings, or both;
- validation commands and results;
- if tests were added or changed, the test quality score and basis;
- any findings intentionally not changed, with the reason;
- remaining risks or follow-up work, if any.
