# Kit

A collection of reusable skills and tools for AI coding agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [pr-ready](skills/pr-ready/) | Mark PRs ready for review, assign GitHub reviewers from a team, post to a daily Slack thread, and create a Linear ticket |

## Installation

Install a skill using the [skills CLI](https://skills.sh):

```sh
bunx skills add samcx/kit --skill pr-ready -a claude-code
# or
bunx skills add samcx/kit --skill pr-ready -a codex-cli
```

## Commands

| Command | Description |
|---------|-------------|
| [wt-cleanup](commands/wt-cleanup.md) | Remove worktrees whose branches have been merged into the default branch |

Install a command by copying it to `~/.claude/commands/`:

```sh
cp commands/wt-cleanup.md ~/.claude/commands/
```

**Prerequisites:**
- [`wt` (worktrunk)](https://github.com/max-sixty/worktrunk) — git worktree manager

## Agent Support

### pr-ready

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | ✅ |
| Other agents | ✖️ |

**Prerequisites:**
- [`gh` CLI](https://cli.github.com/) — authenticated with access to your GitHub org
- [Slack app](https://modelcontextprotocol.io/integrations/slack) — configured for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://github.com/openai/codex)
- [Linear app](https://modelcontextprotocol.io/integrations/linear) — configured for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://github.com/openai/codex)
