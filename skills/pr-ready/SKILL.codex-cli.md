---
name: pr-ready
description: Mark PRs ready for review, assign GitHub reviewers from a team, post to a daily Slack thread, and create a Linear ticket. Self-configures on first run.
---

Mark the current PR ready, prompt in chat to choose GitHub reviewers from a team, add reviewers with `gh` CLI, post to a daily Slack thread via Slack app tools, and create a Linear ticket via Linear app tools.

## Prerequisites

- `gh` CLI authenticated with access to your GitHub org
- Slack app tools available (`mcp__codex_apps__slack._slack_*`)
- Linear app tools available (`mcp__codex_apps__linear_mcp_server._*`)

## Configuration

All settings are stored in `~/.codex/pr-ready.json`. On first run, the agent must check if this file exists, is valid JSON, and contains all required keys.

If the config file is missing, invalid, or missing required keys, run first-run setup:

1. Auto-detect `github_org`: Run `gh repo view --json owner --jq '.owner.login'` to get the org from the current repo. Present the detected value and ask the user to confirm or override.
2. Auto-detect `github_team`: Run `gh api "orgs/<github_org>/teams" --paginate --jq '.[].slug'` to list available teams. If there are 4 or fewer teams, list them and ask the user to choose in chat. Otherwise, list all of them and ask the user to type their choice.
3. Ask for `channel_id`: Ask the user for the Slack channel ID where daily PR threads are posted. Hint: they can find it by right-clicking a channel in Slack, opening channel details, and copying the ID.
4. Ask for `thread_match`: Ask the user what text pattern identifies the daily thread, for example `:pr:s for the day`. Suggest they copy a snippet from an existing thread message.
5. Ask for `linear_team`: Ask the user for the Linear team key, for example `VA` or `ENG`, where tickets should be created.

Write the config file and continue with the workflow.

Expected format:

```json
{
  "channel_id": "C08L8AXAE0M",
  "github_org": "vercel",
  "github_team": "support-platform",
  "linear_team": "VA",
  "thread_match": ":pr:s for the day"
}
```

## Agent Workflow (Required)

1. Load config from `~/.codex/pr-ready.json`. If it is missing, invalid, or missing required keys, run first-run setup.
2. Resolve current PR context with `gh pr view --json number,title,author,url,isDraft`.
3. Mark the PR as ready for review with `gh pr ready`. Skip this step if the PR is already ready.
4. Fetch candidate reviewers from the configured GitHub team:

```sh
gh api "orgs/<github_org>/teams/<github_team>/members" --paginate --jq '.[].login'
```

5. Exclude the PR author from candidates.
6. Prompt the user to choose one or more reviewers, or `none`. If there are more than 4 candidates, list all candidates in chat first.
7. Add selected reviewers with `gh pr edit <number> --add-reviewer <login>`.
8. Resolve Slack user IDs for each selected reviewer:
   1. Get their display name with `gh api users/<login> --jq '.name'`
   2. Search Slack with `mcp__codex_apps__slack._slack_search_users`
   3. If multiple results are returned, match by name. If no results are returned, fall back to `<https://github.com/<login>|@<login>>` in the Slack message.
9. Post to the daily Slack thread using Slack app tools.
10. Create a Linear ticket using Linear app tools.
11. Copy the PR URL to the clipboard with `pbcopy`.
12. Report the outcome: PR ready status, reviewers added, Slack post result, and Linear ticket link.

## Slack Posting via Slack App Tools

Use the Slack app tools directly instead of browser automation.

Steps:

1. Find the daily thread with `mcp__codex_apps__slack._slack_read_channel` using `channel_id` from config and `limit: 20`. Find the message whose text contains `thread_match`. Extract its `ts` or `thread_ts` as the thread timestamp.
2. Check for duplicates with `mcp__codex_apps__slack._slack_read_thread` using the channel ID and thread timestamp. If any reply already contains `api/pull/<prNumber>`, skip posting and report that it was deduped.
3. Post the message with `mcp__codex_apps__slack._slack_send_message` using `channel_id`, `thread_ts`, and the formatted message.

## Posted Slack Format

```text
:pr: <https://github.com/<org>/<repo>/pull/<number>|#<number>>: <title>
cc <@SLACK_USER_ID>, ...
```

- Derive the org and repo from the PR URL returned by `gh pr view`.
- Make `#<number>` a Slack mrkdwn link to the GitHub PR URL.
- Do not include an Author line.
- Use Slack user mention syntax `<@SLACK_USER_ID>` for reviewers so they get pinged.
- If a reviewer's Slack ID cannot be found, fall back to `<https://github.com/<login>|@<login>>`.
- If no reviewers were selected, omit the `cc` line entirely.
- Do not use bullet points or list markers in the posted message.
- Do not include a "Sent using Codex" line.

## Linear Ticket

Use the Linear app tools to create a ticket after posting to Slack.

Steps:

1. Use `mcp__codex_apps__linear_mcp_server._save_issue` with:
   - `title`: the PR title from `gh pr view`
   - `team`: the `linear_team` value from config
   - `assignee`: `"me"`
   - `state`: `"In Review"`
   - `priority`: `2`
   - `description`: a brief description of the PR changes, derived from the PR title and context, including a link to the PR
   - `links`: `[{"url": "<PR URL>", "title": "PR #<number>"}]`
2. Report the created ticket identifier and link in the outcome summary.

## Important Behavior

- You must use Slack app tools (`mcp__codex_apps__slack._slack_*`) for all Slack interactions. Never fall back to `agent-browser`, browser automation, or the browser-based Slack skill. If the Slack app tools are not available, stop and tell the user.
- You must use Linear app tools (`mcp__codex_apps__linear_mcp_server._*`) for Linear ticket creation. If the Linear app tools are not available, skip the Linear step and tell the user.
- Do not rely on terminal `fzf` for agent-driven flows. Always ask in chat first.
- If the user chooses `none`, skip adding reviewers and omit the `cc` line.
- If the daily thread is not found, stop and tell the user.
- If the Slack post fails, show the error and do not claim success.
