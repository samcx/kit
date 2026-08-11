# Kit

A collection of reusable skills and tools for AI coding agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [loop-review](skills/loop-review/) | Run scope-bounded adversarial review and fix loops until a PR or stack is safe for review |
| [pr-ready](skills/pr-ready/) | Create a Linear ticket, link it in the PR description, mark PRs ready for review, optionally add GitHub assignees, assign GitHub reviewers, and post to a daily Slack thread |
| [wt-cleanup](skills/wt-cleanup/) | Remove worktrees whose branches have been merged into the default branch |

## Installation

Install a skill using the [skills CLI](https://skills.sh).

### loop-review

```sh
bunx --bun skills add samcx/kit --skill loop-review -a codex
```

Invoke the skill with a PR, branch, or stack:

```text
$loop-review <PR, branch, or stack>
```

The first turn resolves the review contract, copies a goal body beginning with
`Use $loop-review ...`, and stops. Type or select `/goal`, paste the clipboard
contents, and submit to start the persistent loop. Native review passes run
read-only with approval requests disabled; authorized fixes still use the
parent Codex session's permissions, so start Codex in the target worktree. Use
`/permissions` and select **Approve for me** if you want Codex to evaluate
eligible parent-session approvals. The generated goal authorizes staging,
committing, and normally pushing validated fixes to the existing PR branch; it
never authorizes force-pushing or rewriting history.

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
