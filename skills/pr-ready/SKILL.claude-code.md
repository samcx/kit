---
name: pr-ready
description: Mark one or more related PRs ready for review, assign GitHub reviewers from a team, post to a daily Slack thread, and create Linear tickets. Self-configures on first run.
---

Mark the current PR ready, prompt in chat to choose GitHub reviewers from a team, add reviewers with gh CLI, post to a daily Slack thread via Slack MCP tools, and create a Linear ticket via Linear MCP tools.

When the same request publishes multiple related PRs, such as backend and frontend halves of one change, treat them as one batch: use one reviewer selection and send one combined Slack reply listing all unannounced PRs.

## Prerequisites

- `gh` CLI authenticated with access to your GitHub org
- Slack MCP tools available (`mcp__claude_ai_Slack__slack_*`)
- Linear MCP tools available (`mcp__claude_ai_Linear__*`)

## Configuration

All settings are stored in `~/.claude/pr-ready.json`. On first run, the agent must check if this file exists, is valid JSON, and contains all required keys.

**If the config file is missing, invalid, or missing required keys**, run first-run setup:

1. **Auto-detect `github_org`**: Run `gh repo view --json owner --jq '.owner.login'` to get the org from the current repo. Present the detected value and ask the user to confirm or override.
2. **Auto-detect `github_team`**: Run `gh api "orgs/<github_org>/teams" --paginate --jq '.[].slug'` to list available teams. If there are 4 or fewer, use `AskUserQuestion`. Otherwise, list them and ask the user to type their choice.
3. **Ask for `channel_id`**: Ask the user for the Slack channel ID where daily PR threads are posted. Hint: they can find it by right-clicking a channel in Slack → "View channel details" → the ID is at the bottom.
4. **Ask for `thread_match`**: Ask the user what text pattern identifies the daily thread (e.g. `:pr:s for the day`). Suggest they copy a snippet from an existing thread message.
5. **Ask for `linear_team`**: Ask the user for the Linear team key (e.g. `VA`, `ENG`) where tickets should be created.

Write the config file and continue with the workflow.

**Config file format** (`~/.claude/pr-ready.json`):
```json
{
  "channel_id": "C08L8AXAE0M",
  "github_org": "vercel",
  "github_team": "support-platform",
  "linear_team": "VA",
  "thread_match": ":pr:s for the day",
  "slack_handles": {
    "<github-login>": { "id": "<slack-user-id>", "name": "<real name>" }
  }
}
```

`slack_handles` is optional and accumulates over time. Don't prompt for it during first-run setup — the workflow auto-populates it whenever a new reviewer's Slack ID is resolved via search (see step 10).

## Agent Workflow (Required)

1. **Load config** from `~/.claude/pr-ready.json`. If it is missing, invalid, or missing required keys, run first-run setup (see above).
2. Resolve current PR context (`gh pr view --json number,title,author,url,isDraft,additions,deletions`). If the same user request created or names multiple related PRs, resolve each PR and keep them together as one batch for the rest of this workflow.
3. **Create a Linear ticket for each PR** (see Linear Ticket below). Do this early so each ticket can be linked in its PR description.
4. **Link each Linear ticket in its PR description**: Append a `Linear: [TICKET-ID](url)` line to each existing PR body using `gh pr edit <number> --body`. Preserve each existing body — only append its Linear link.
5. Mark each PR as ready for review (`gh pr ready`). Skip any PR already marked ready.
6. Fetch candidate reviewers from the configured GitHub team:

```sh
gh api "orgs/<github_org>/teams/<github_team>/members" --paginate --jq '.[].login'
```

7. Exclude every PR author in the batch from candidates.
8. Prompt the user exactly once to choose 1+ reviewers (or `none`). If there are more than 4 candidates, list ALL candidates in chat first, then use `AskUserQuestion` with multiSelect showing up to 4 options — the user can select "Other" to type a name not shown. If there are 4 or fewer, use `AskUserQuestion` with multiSelect directly. If the workflow pauses for this choice, do not repeat the full reviewer prompt in a final/status message; say reviewer selection is pending.
9. Add selected reviewers to each PR with `gh pr edit <number> --add-reviewer <login>`.
10. **Resolve Slack user IDs** for each selected reviewer, in order:
    1. **Check `slack_handles[<login>].id` in the config** — if present, use it directly (no API calls needed). This is the fast path for known teammates.
    2. Get their display name: `gh api users/<login> --jq '.name'`.
    3. Search Slack: `mcp__claude_ai_Slack__slack_search_users` with that name. If multiple results, match by name.
    4. **Scan the target channel/thread** for prior `<@U.+|handle>` cc patterns from the PR author — daily PR threads often re-cc the same teammates, so a recent cc line can reveal the Slack ID when name search fails.
    5. If still unresolved, fall back to `<https://github.com/<login>|@<login>>` in the Slack message.
    6. **When steps 2-4 resolve a new mapping, append it to `slack_handles` in `~/.claude/pr-ready.json`** as `"<login>": { "id": "U...", "name": "Real Name" }` so future runs hit step 1.
11. Post to the daily Slack thread using MCP tools (see Slack Posting below). For a batch, post one combined message, not one message per PR.
12. Copy the PR URL to clipboard with `pbcopy`; for a batch, copy all PR URLs separated by newlines.
13. Report outcome for each PR: ready status, reviewers added, Slack post result, and Linear ticket link.

## Slack Posting via MCP Tools

Use the Slack MCP tools directly instead of calling a deployed endpoint.

Steps:

1. **Find the daily thread**: Use `mcp__claude_ai_Slack__slack_read_channel` with `channel_id` from config and `limit: 20`. Find the message whose text contains the `thread_match` pattern from config. Extract its `ts` (or `thread_ts`) as the thread timestamp.
2. **Check for duplicates**: Use `mcp__claude_ai_Slack__slack_read_thread` with the channel ID and the thread timestamp. Check each PR URL or `<repo>/pull/<number>` fragment in the batch. Omit any PR already announced; if all are already present, skip posting and report that the batch was deduped.
3. **Post one message**: Use `mcp__claude_ai_Slack__slack_send_message` with `channel_id`, `thread_ts`, and one formatted message containing all remaining unannounced PRs.

## Posted Slack Format

```
:pr: <https://github.com/<org>/<repo>/pull/<number>|#<number>>: <title> +<additions> -<deletions>
cc <@SLACK_USER_ID>, ...
```

- For a batch, put one `:pr:` line per PR in the same message, followed by one shared `cc` line:

```
:pr: <https://github.com/<org>/<repo-a>/pull/<number>|#<number>>: <title> +<additions> -<deletions>
:pr: <https://github.com/<org>/<repo-b>/pull/<number>|#<number>>: <title> +<additions> -<deletions>
cc <@SLACK_USER_ID>, ...
```

- Derive the org and repo from the PR URL returned by `gh pr view`.
- Make the `#<number>` a Slack mrkdwn link to the GitHub PR URL.
- Append `+<additions> -<deletions>` to the PR line using values returned by `gh pr view`; if either value is unavailable, omit the stats rather than guessing.
- Do NOT include an Author line.
- Use Slack user mention syntax `<@SLACK_USER_ID>` for reviewers so they get pinged.
- If a reviewer's Slack ID can't be found, fall back to `<https://github.com/<login>|@<login>>`.
- If no reviewers were selected, omit the cc line entirely.
- No bullet points or list markers — just plain lines.
- Do NOT include a "Sent using Claude" line in the message.

## Linear Ticket

Use the Linear MCP tools to create a ticket early in the workflow (step 3) so it can be linked in the PR description.

Steps:

1. Use `mcp__claude_ai_Linear__save_issue` with:
   - `title`: The PR title from `gh pr view`
   - `team`: The `linear_team` value from config
   - `assignee`: `"me"` (Linear MCP resolves this to the authenticated user)
   - `state`: `"In Review"`
   - `priority`: `2` (High)
   - `description`: A brief description of the PR changes, derived from the PR title and context. Include a link to the PR.
   - `links`: `[{"url": "<PR URL>", "title": "PR #<number>"}]`
2. Save the returned ticket identifier (e.g. `VA-1234`) and URL for use in step 4 (linking in PR description) and the outcome summary.

## Important Behavior

- You MUST use Slack MCP tools (`mcp__claude_ai_Slack__slack_*`) for all Slack interactions. NEVER fall back to browser automation, Claude in Chrome (`mcp__claude-in-chrome__*`), or the `slack` skill. If the Slack MCP tools are not available, stop and tell the user.
- You MUST use Linear MCP tools (`mcp__claude_ai_Linear__*`) for Linear ticket creation. If the Linear MCP tools are not available, skip the Linear step and tell the user.
- Do not rely on terminal `fzf` for agent-driven flows; always ask in chat first.
- Do not ask for reviewers more than once in the same `pr-ready` run. Reuse the user's reviewer choice for both GitHub review requests and Slack cc mentions.
- Treat related PRs created or prepared in the same request as a batch for Slack. Do not send separate daily-thread replies solely because the change spans repositories.
- If user chooses `none`, skip adding reviewers and omit the cc line.
- If the daily thread is not found, stop and tell the user.
- If the Slack post fails, show the error and do not claim success.
