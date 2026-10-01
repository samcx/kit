---
name: pr-ready
description: Mark one or more related PRs ready for review, optionally add explicit GitHub assignees, assign GitHub reviewers from a team, post to a Slack channel or daily Slack thread (user picks), and create Linear tickets. Self-configures on first run.
---

Mark the current PR ready, optionally add explicitly requested GitHub assignees, prompt in chat to choose GitHub reviewers from a team, add reviewers with gh CLI, prompt in chat to choose a Slack destination (a channel or a daily thread), post there via Slack MCP tools, and create a Linear ticket via Linear MCP tools.

When the same request publishes multiple related PRs, such as backend and frontend halves of one change, treat them as one batch: use one reviewer selection and send one combined Slack reply listing all unannounced PRs.

GitHub assignees and reviewers are separate. Only add assignees when the user explicitly requests them. Treat "assign this to me" or "assign this to myself" as self-assignment; do not infer an assignee from reviewer selection or prompt for one by default.

## Prerequisites

- `gh` CLI authenticated with access to your GitHub org
- Slack MCP tools available (`slack_read_channel`, `slack_read_thread`, `slack_send_message`, `slack_search_users`)
- Linear MCP tools available (`save_issue`)

Discover Slack and Linear MCP tools with `GetMcpTools` before calling them. Typical Cursor server IDs are `plugin-slack-slack` and `plugin-linear-linear`.

## Configuration

All settings are stored in `~/.cursor/pr-ready.json`. On first run, the agent must check if this file exists, is valid JSON, and contains all required keys.

**If the config file is missing, invalid, or missing required keys**, run first-run setup:

1. **Auto-detect `github_org`**: Run `gh repo view --json owner --jq '.owner.login'` to get the org from the current repo. Present the detected value and ask the user to confirm or override.
2. **Auto-detect `github_team`**: Run `gh api "orgs/<github_org>/teams" --paginate --jq '.[].slug'` to list available teams. If there are 4 or fewer, use `AskQuestion`. Otherwise, list them and ask the user to type their choice.
3. **Ask for `slack_destinations`**: Ask the user for one or more Slack places to announce PRs. For each, collect:
   - `name`: a label shown in the destination prompt (e.g. `#team-ax daily PR thread`)
   - `type`: `thread` (reply in a daily thread) or `channel` (top-level post in the channel)
   - `channel_id`: resolve from the channel name with `slack_search_channels`, or ask for the ID (right-click channel → "View channel details" → ID at the bottom)
   - `thread_match` (only for `type: thread`): text that identifies the daily thread (e.g. `:pr:s for the day`). Suggest they copy a snippet from an existing thread message.
   - `linear_team` (optional): Linear team key for tickets announced here, if it differs from the default `linear_team` (e.g. `MONEY`)
4. **Ask for `linear_team`**: Ask the user for the default Linear team key (e.g. `VA`, `ENG`) where tickets should be created. Destinations can override it with their own `linear_team`.

Write the config file and continue with the workflow.

**Config file format** (`~/.cursor/pr-ready.json`):
```json
{
  "github_org": "vercel",
  "github_team": "support-platform",
  "linear_team": "VA",
  "slack_destinations": [
    { "name": "#team-ax daily PR thread", "type": "thread", "channel_id": "C08L8AXAE0M", "thread_match": ":pr:s for the day" },
    { "name": "#finfra-code-reviews", "type": "channel", "channel_id": "C0B26EZD7DH", "linear_team": "MONEY" }
  ],
  "slack_handles": {
    "<github-login>": { "id": "<slack-user-id>", "name": "<real name>" }
  }
}
```

**Legacy config**: if the file has top-level `channel_id` + `thread_match` instead of `slack_destinations`, treat them as a single `type: thread` destination and rewrite the file into the new shape.

`slack_handles` is optional and accumulates over time. Don't prompt for it during first-run setup — the workflow auto-populates it whenever a new reviewer's Slack ID is resolved via search (see step 13).

## Agent Workflow (Required)

1. **Load config** from `~/.cursor/pr-ready.json`. If it is missing, invalid, or missing required keys, run first-run setup (see above).
2. Resolve current PR context (`gh pr view --json number,title,author,assignees,url,isDraft,additions,deletions`). If the same user request created or names multiple related PRs, resolve each PR and keep them together as one batch for the rest of this workflow.
3. **Resolve explicitly requested GitHub assignees**. Use a named login as written. For `me` or `myself`, resolve the authenticated login with `gh api user --jq '.login'`. If the user did not request an assignee, skip assignment without prompting. For a batch, apply an unscoped request to every PR; honor any PR-specific scope.
4. Fetch candidate reviewers from the configured GitHub team:

```sh
gh api "orgs/<github_org>/teams/<github_team>/members" --paginate --jq '.[].login'
```

5. Exclude every PR author in the batch from reviewer candidates.
6. Prompt the user exactly once to choose 1+ reviewers (or `none`). If there are more than 4 candidates, list ALL candidates in chat first, then use `AskQuestion` with `allow_multiple: true` showing up to 4 options — the user can select "Other" to type a name not shown. If there are 4 or fewer, use `AskQuestion` with `allow_multiple: true` directly. If the workflow pauses for this choice, do not repeat the full reviewer prompt in a final/status message; say reviewer selection is pending.
7. **Choose the Slack destination**: Use `AskQuestion` (single choice) with one option per entry in `slack_destinations`, labeled with its `name`, plus a `Skip Slack` option. Ask this in the same `AskQuestion` call as the reviewer prompt in step 6 when possible, so the user answers both at once. If only one destination is configured, still ask (it may be skipped). The choice decides both where to post and which Linear team gets the ticket (see Linear Ticket below), so ask before creating tickets.
8. **Create a Linear ticket for each PR** in the Linear team for the chosen destination (see Linear Ticket below). Do this early so each ticket can be linked in its PR description.
9. **Link each Linear ticket in its PR description**: Append a `Linear: [TICKET-ID](url)` line to each existing PR body using `gh pr edit <number> --body`. Preserve each existing body — only append its Linear link.
10. Mark each PR as ready for review (`gh pr ready`). Skip any PR already marked ready.
11. Add each explicitly requested assignee to the intended PRs with `gh pr edit <number> --add-assignee <login>`. Skip logins already assigned. A PR author may be assigned to their own PR.
12. Add selected reviewers to each PR with `gh pr edit <number> --add-reviewer <login>`.
13. **Resolve Slack user IDs** for each selected reviewer, in order:
    1. **Check `slack_handles[<login>].id` in the config** — if present, use it directly (no API calls needed). This is the fast path for known teammates.
    2. Get their display name: `gh api users/<login> --jq '.name'`.
    3. Search Slack: `slack_search_users` with that name. If multiple results, match by name.
    4. **Scan the configured Slack destinations** for prior `<@U.+|handle>` cc patterns from the PR author — daily PR threads often re-cc the same teammates, so a recent cc line can reveal the Slack ID when name search fails.
    5. If still unresolved, fall back to `<https://github.com/<login>|@<login>>` in the Slack message.
    6. **When steps 2-4 resolve a new mapping, append it to `slack_handles` in `~/.cursor/pr-ready.json`** as `"<login>": { "id": "U...", "name": "Real Name" }` so future runs hit step 1.
14. Post to the chosen destination using MCP tools (see Slack Posting below). For a batch, post one combined message, not one message per PR.
15. Copy the PR URL to clipboard with `pbcopy`; for a batch, copy all PR URLs separated by newlines.
16. Report outcome for each PR: ready status, assignees added or unchanged, reviewers added, Slack destination and post result, and Linear ticket link.

## Slack Posting via MCP Tools

Use the Slack MCP tools directly instead of calling a deployed endpoint. Follow the steps for the chosen destination's `type`.

### `type: thread` (e.g. #team-ax daily PR thread)

1. **Find the daily thread**: Use `slack_read_channel` with the destination's `channel_id` and `limit: 20`. Find the message whose text contains the destination's `thread_match` pattern. Extract its `ts` (or `thread_ts`) as the thread timestamp.
2. **Check for duplicates**: Use `slack_read_thread` with the channel ID and the parent message timestamp. Check each PR URL or `<repo>/pull/<number>` fragment in the batch. Omit any PR already announced; if all are already present, skip posting and report that the batch was deduped.
3. **Post one message**: Use `slack_send_message` with `channel_id`, `thread_ts`, and one formatted message containing all remaining unannounced PRs.

### `type: channel` (e.g. #finfra-code-reviews)

1. **Check for duplicates**: Use `slack_read_channel` with the destination's `channel_id` and `limit: 50`. Omit any PR in the batch whose URL or `<repo>/pull/<number>` fragment already appears in a recent message; if all are present, skip posting and report that the batch was deduped.
2. **Post one top-level message**: Use `slack_send_message` with `channel_id` only (no `thread_ts`), using the same message format below.

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
- Do NOT include a "Sent using Cursor" line in the message.

## Linear Ticket

Use the Linear MCP tools to create a ticket early in the workflow (step 8), after the Slack destination is chosen, so it can be linked in the PR description.

Steps:

1. Use `save_issue` with:
   - `title`: The PR title from `gh pr view`
   - `team`: The chosen destination's `linear_team` if it has one; otherwise the top-level `linear_team` from config (also used when the user picks `Skip Slack`)
   - `assignee`: `"me"` (Linear MCP resolves this to the authenticated user)
   - `state`: `"In Review"`
   - `priority`: `2` (High)
   - `description`: A brief description of the PR changes, derived from the PR title and context. Include a link to the PR.
   - `links`: `[{"url": "<PR URL>", "title": "PR #<number>"}]`
2. Save the returned ticket identifier (e.g. `VA-1234`) and URL for use in step 9 (linking in PR description) and the outcome summary.

## Important Behavior

- You MUST use Slack MCP tools (`slack_read_channel`, `slack_read_thread`, `slack_send_message`, `slack_search_users`) for all Slack interactions. NEVER fall back to browser automation or the `slack` skill. If the Slack MCP tools are not available, stop and tell the user.
- You MUST use Linear MCP tools (`save_issue`) for Linear ticket creation. If the Linear MCP tools are not available, skip the Linear step and tell the user.
- GitHub assignment is opt-in. Do not add or prompt for an assignee unless the user explicitly requests one, and do not treat reviewer selection as an assignment request.
- Self-assignment is allowed. Excluding PR authors from reviewer candidates does not exclude them from assignees.
- Do not rely on terminal `fzf` for agent-driven flows; always ask in chat first.
- Do not ask for reviewers more than once in the same `pr-ready` run. Reuse the user's reviewer choice for both GitHub review requests and Slack cc mentions.
- Treat related PRs created or prepared in the same request as a batch for Slack. Do not send separate Slack messages solely because the change spans repositories.
- If user chooses `none`, skip adding reviewers and omit the cc line.
- If the user picks `Skip Slack`, skip posting and say so in the outcome.
- If the daily thread is not found (for a `thread` destination), stop and tell the user.
- If the Slack post fails, show the error and do not claim success.
