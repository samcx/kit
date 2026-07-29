# Kit

A collection of reusable skills and tools for AI coding agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [loop-review](skills/loop-review/) | Run scope-bounded adversarial review and fix loops until a PR or stack is safe for review |
| [pr-ready](skills/pr-ready/) | Create a Linear ticket, link it in the PR description, mark PRs ready for review, assign GitHub reviewers, and post to a daily Slack thread |
| [wt-cleanup](skills/wt-cleanup/) | Remove worktrees whose branches have been merged into the default branch |

## Installation

Install a skill using the [skills CLI](https://skills.sh).

### loop-review

```sh
bunx skills add samcx/kit --skill loop-review -a codex
```

Start a persistent review loop in Codex with:

```text
/goal Use $loop-review to adversarially review <PR or stack> against the scope contract below, fix in-scope blockers, and continue until its stated guarantees are safe for review. <scope contract>
```

### pr-ready

```sh
bunx skills add samcx/kit --skill pr-ready -a claude-code
# or
bunx skills add samcx/kit --skill pr-ready -a codex-cli
```

**Prerequisites:**
- [`gh` CLI](https://cli.github.com/) — authenticated with access to your GitHub org
- [Slack app](https://modelcontextprotocol.io/integrations/slack) — configured for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://github.com/openai/codex)
- [Linear app](https://modelcontextprotocol.io/integrations/linear) — configured for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://github.com/openai/codex)

### wt-cleanup

```sh
bunx skills add samcx/kit --skill wt-cleanup
```

In Codex, type `$` and select **Worktree Cleanup** to run it without writing a
separate prompt.

**Prerequisites:**
- [`wt` (worktrunk)](https://github.com/max-sixty/worktrunk) — git worktree manager

## Agent Support

### loop-review

| Agent | Status |
|-------|--------|
| Claude Code | — |
| Codex CLI | ✅ |

### pr-ready

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | ✅ |

### wt-cleanup

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | ✅ |
