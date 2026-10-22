# Kit

A collection of reusable skills and tools for AI coding agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [goal-review](skills/goal-review/) | Run scope-bounded adversarial review and fix loops until a PR or stack is safe for review |
| [pr-ready](skills/pr-ready/) | Create a Linear ticket, link it in the PR description, mark PRs ready for review, optionally add GitHub assignees, assign GitHub reviewers, and draft a Slack announcement in a channel or daily thread |
| [wt-cleanup](skills/wt-cleanup/) | Remove worktrees whose branches have been merged into the default branch |

## Installation

Install a skill using the [skills CLI](https://skills.sh).

### goal-review

```sh
bunx --bun skills add samcx/kit --skill goal-review -a codex
# or
bunx --bun skills add samcx/kit --skill goal-review -a claude-code
```

Invoke the skill with a PR, branch, or stack:

```text
$goal-review <PR, branch, or stack>    # Codex
/goal-review <PR, branch, or stack>    # Claude Code
```

The first turn resolves the review contract, copies a goal body beginning with
`Use $goal-review ...`, and stops. Type or select `/goal`, paste the clipboard
contents, and submit to start the persistent loop. Native review passes run
read-only with approval requests disabled; authorized fixes still use the
parent Codex session's permissions, so start Codex in the target worktree. Use
`/permissions` and select **Approve for me** if you want Codex to evaluate
eligible parent-session approvals. The generated goal authorizes staging,
committing, and normally pushing validated fixes to the existing PR branch; it
never authorizes force-pushing or rewriting history.

In Claude Code, the first turn likewise copies a goal condition beginning with
`Use /goal-review ...` and stops; type `/goal `, paste, and submit. Each review
pass runs in a fresh read-only subagent instead of `codex exec review`, and the
parent session applies authorized fixes. Run `/goal` in auto mode so goal turns
proceed without per-tool prompts.

### pr-ready

```sh
bunx skills add samcx/kit --skill pr-ready -a claude-code
# or
bunx skills add samcx/kit --skill pr-ready -a codex-cli
# or
bunx skills add samcx/kit --skill pr-ready -a cursor
```

**Prerequisites:**
- [`gh` CLI](https://cli.github.com/) — authenticated with access to your GitHub org
- [Slack app](https://modelcontextprotocol.io/integrations/slack) — configured for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex CLI](https://github.com/openai/codex), or [Cursor](https://cursor.com)
- [Linear app](https://modelcontextprotocol.io/integrations/linear) — configured for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex CLI](https://github.com/openai/codex), or [Cursor](https://cursor.com)

### wt-cleanup

```sh
bunx skills add samcx/kit --skill wt-cleanup
```

In Codex, type `$` and select **Worktree Cleanup** to run it without writing a
separate prompt.

**Prerequisites:**
- [`wt` (worktrunk)](https://github.com/max-sixty/worktrunk) — git worktree manager

## Agent Support

### goal-review

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | ✅ |

### pr-ready

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | ✅ |
| Cursor | ✅ |

### wt-cleanup

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | ✅ |
