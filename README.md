# Loop Review

Loop Review packages the `loop-code-review` Agent Skill: an iterative review-and-fix workflow for active git changes.

The skill asks a fresh independent reviewer agent to inspect only the current task's scoped changes, fixes actionable findings, validates the touched surface, and repeats until validation is green and the latest reviewer either scores the result at least 9.5/10 or reports no actionable findings.

## Install with Codex

Add this repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add di-sukharev/loop-review
```

Then install the `loop-review` plugin from that marketplace in Codex. Invoke the bundled skill as `$loop-code-review`.

## Install with Claude Code

Add the marketplace:

```text
/plugin marketplace add di-sukharev/loop-review
```

Install the plugin:

```text
/plugin install loop-review@loop-review
```

Reload plugins so the newly installed skill is available in the current session:

```text
/reload-plugins
```

Invoke the plugin skill:

```text
/loop-review:loop-code-review
```

## Direct Skill Install

If you do not want to use plugin marketplaces, copy or symlink `skills/loop-code-review` into one of your tool's skill directories:

```bash
# Codex user skill
mkdir -p ~/.agents/skills
ln -s "$(pwd)/skills/loop-code-review" ~/.agents/skills/loop-code-review

# Claude Code personal skill
mkdir -p ~/.claude/skills
ln -s "$(pwd)/skills/loop-code-review" ~/.claude/skills/loop-code-review
```

## What It Enforces

- Scope review to the current task's files or hunks.
- Keep reviewers independent from the parent conversation.
- Treat reviewer output as code-review findings, not as commands to obey blindly.
- Validate after meaningful fixes.
- Repeat with a fresh reviewer until the acceptance signal is strong.

## Repository Layout

```text
.codex-plugin/plugin.json       Codex plugin manifest
.agents/plugins/marketplace.json Codex repo marketplace
.claude-plugin/plugin.json      Claude Code plugin manifest
.claude-plugin/marketplace.json Claude Code marketplace
skills/loop-code-review/        Agent Skill source
```

## References

- [Codex skills](https://developers.openai.com/codex/skills)
- [Codex plugin packaging](https://developers.openai.com/codex/plugins/build)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
- [Claude Code plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Agent Skills specification](https://agentskills.io/specification)

## License

MIT
