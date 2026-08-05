---
name: loop-review
description: Run scope-bounded adversarial code-review loops for PRs, branches, commit ranges, and cumulative stacks. Use when automating native Codex reviews with `codex exec review` or `/review`, iterating findings and fixes until safe for review, using `/goal` to preserve completion criteria, or preventing review loops from expanding scope or becoming endless.
---

# Loop Review

Use `/goal` to preserve one completion contract across turns and Codex's native
reviewer to obtain an independent review. Fix only demonstrated violations of
that contract, verify them, and continue automatically only while within the
scope circuit breaker.

## Bootstrap the loop through `/goal`

`/goal` is a client action. Never claim this skill invoked it.

Check whether an active goal exists before starting a review. When the skill is
activated without an active goal:

1. Resolve the exact target, base, head, guarantees, non-goals, named entry
   points, temporal boundary, allowed implementation surface, allowed fixes,
   and required verification.
2. Construct the goal body using this shape. Do not include `/goal` in the
   clipboard payload:

   ```text
   Use $loop-review to adversarially review <target> at <head> against <base>. Scope contract: <guarantees, non-goals, named entry points, temporal boundary, allowed implementation surface, allowed fixes, and required verification>. Authorize editing, committing, and pushing validated fixes to the existing PR head branch; never force-push or rewrite history. Fix validated in-scope blockers, verify each fix, and continue with fresh independent review passes only while within the scope circuit breaker, stopping when one pass finds no validated blockers. Do not broaden scope. Finish with the final diff, verification, pushed head, and safe-for-review verdict.
   ```

3. Keep the goal body within the 4,000-character `/goal` limit.
4. Copy the exact body with `printf '%s' '<safely shell-escaped goal body>' |
   pbcopy`. Run that pipeline directly; do not wrap it in `sh -c` or `zsh -lc`,
   and do not use redirection, command substitution, or a temporary file. Read
   it back with `pbpaste` and verify it matches byte-for-byte.
5. Tell the user to type or select `/goal`, paste the clipboard contents, and
   submit. Then stop; do not begin the review.

If clipboard access is unavailable, make the entire response the exact goal
body so the user can run `/copy`, then type `/goal` and paste it. When an active
goal exists, skip this bootstrap and continue the review loop.

Treat the active goal objective as the durable outcome, constraints, and
definition of done. Use goal tools to track it. Do not create a goal from
implicit skill activation alone.

If the scope is not yet clear, establish the contract before starting the goal.
Use `/plan` first when the outcome needs discussion, then put the agreed contract
in `/goal`. Goal text has a 4,000-character limit; for a longer contract, point
the goal to a file containing it.

## Select the native review runner

These runner-selection instructions apply only to the parent agent controlling
the loop. A native review subprocess performs exactly one review pass: it must
not invoke `codex exec review`, `/review`, this skill, or another reviewer.

Prefer these runners in order:

1. If shell execution is available and `codex exec help review` succeeds, set
   the process working directory to the exact repository worktree and run:

   ```sh
   codex --sandbox read-only --ask-for-approval never exec review --ephemeral "<custom review instructions>"
   ```

   Pass the custom instructions directly as the prompt argument. Do not use
   stdin, a shell wrapper, pipe, redirection, command substitution, or temporary
   prompt file. This invokes Codex's native review path with a fresh,
   non-interactive, read-only reviewer; do not substitute an ordinary
   `codex exec` turn or a same-context self-review. Custom instructions conflict
   with `--base` and `--commit`, so put the exact base and head in the prompt
   instead of combining those flags. If a launch fails, diagnose prompt
   quoting, permissions, and transient failures and retry when recoverable.
   Continue to the next runner only after establishing that the native
   subcommand cannot be invoked in the current environment.
2. Otherwise, invoke `/review` directly when the current client exposes that
   action to the agent.
3. Only when neither native runner is callable, emit the exact paste-ready
   `/review` command and wait for the user to run it.

When `codex exec review` is available, run it automatically. Do not make the
user paste `/review` between iterations. Keep output files outside the reviewed
repository, do not use bypass-sandbox options, and wait for the review to finish
before triaging its output.

Run exactly one reviewer process per pass. Retain its session and poll it until
it exits; do not relaunch it because output is quiet or a poll times out.

If a verification command cannot run only because the reviewer's read-only
sandbox blocks required writes, record it as unavailable rather than a code
finding. Re-run the required verification in the parent's authorized
environment and use that result.

The parent agent performs authorized fixes using the current session's
permissions. Keep the target worktree inside the session's writable roots. If
it is outside them, tell the user to reopen Codex in that worktree instead of
repeatedly requesting escalation. A skill cannot grant or auto-accept client
permissions.

## Freeze the review contract

Before the first review:

- Resolve the exact base and head, including every PR in a cumulative stack.
- Inspect the working tree before every pass. When authorized fixes are not
  committed, include all staged, unstaged, and intended untracked changes in
  the review target; do not assume the unchanged Git head contains them.
- Inspect the live diff and relevant repository instructions.
- State each guarantee the change intends to provide.
- State explicit non-goals, especially nearby architecture the change does not
  promise to solve.
- State the verification required for completion.

Treat the user's supplied contract as authoritative. Preserve its guarantees
and non-goals; normalize wording or request clarification without broadening
it. Otherwise, draft the smallest contract supported by the PR description and
diff, and ask only about ambiguities that would materially change the outcome.

Keep this contract fixed for every iteration. Add, remove, or broaden a
guarantee only when the user explicitly changes the goal or scope. Do not turn
adjacent discoveries into blockers by silently appending them to the contract.

## Scope circuit breaker

Before reviewing, freeze the named entry points, temporal boundary, and allowed
implementation surface.

Do not automatically implement a fix that introduces persistence, a schema or
migration, public API or SDK changes, a new dependency, or a new route or
client. Classify it as a scope decision, return `scope-check`, and request
approval before editing.

If a finding concerns only code introduced by an earlier loop fix, prefer
reverting or simplifying that fix. Two consecutive findings in loop-added code
require `scope-check`.

Count consecutive findings across successive review passes, not within one
reviewer response. User approval applies only to the named boundary expansion
and does not waive later `scope-check` requirements.

## Request the adversarial review

Build custom review instructions with this structure:

```text
Adversarially review <exact base through exact head>, including all current
staged, unstaged, and intended untracked changes in this worktree.

You are the independent review pass, not the loop controller. Do not invoke
`codex exec review`, `/review`, this skill, or another reviewer. Do not modify
files.

Strict scope contract:
1. <guarantee>
2. <guarantee>

Frozen implementation boundary:
- Named entry points: <entry points>
- Temporal boundary: <base through head and current worktree changes>
- Allowed implementation surface: <files, packages, and change types>

Explicit non-goals:
- <non-goal>

Try to break those exact guarantees, including interactions across the
cumulative diff.

For every finding, require:
- a concrete failing execution path or reproducible test;
- the exact guarantee violated;
- evidence the regression is introduced or materially worsened by this diff;
- the smallest fix that restores the guarantee.

Classify findings as:
- In-scope blocker: the diff fails a stated guarantee.
- Scope decision: a real issue whose smallest fix crosses the frozen
  implementation boundary.
- Follow-up: a real issue that requires a new guarantee or broader architecture.
- Pre-existing/out of scope: not introduced or materially worsened here.

Also inspect comments added or modified by this diff for low-value generated
prose: comments that restate obvious code, narrate implementation or PR history,
use generic filler, or overexplain straightforward logic. List concrete
candidates separately as "Comment cleanup" with the smallest deletion or
rewrite. Do not treat comment cleanup as `fix-first` unless a misleading or
incorrect comment violates a stated guarantee.

Return "scope-check" for a scope decision without returning "fix-first". Return
"fix-first" only for an in-scope blocker. List adjacent discoveries as
nonblocking follow-ups. Return "safe for review" when all stated guarantees
hold.
```

Pass that text unchanged to the selected native review runner. Prefix it with
`/review ` only for the client-command fallback. Native review output may use
Codex's structured `findings` and `overall_correctness` schema instead of the
requested verdict words; independently map a validated scope decision to
`scope-check`, validated in-scope findings to `fix-first`, and a correct patch
with neither to `safe for review`.

Add precise exclusions for adjacent architecture the contract does not promise.
Derive them from the current change; never carry one stack's non-goals into
another.

Review the entire cumulative diff on every pass, not only the latest fix.

## Triage findings before editing

Independently validate every finding against the source. Accept it as a blocker
only when all four required elements are present. Reclassify a real issue whose
smallest fix crosses the frozen implementation boundary as a scope decision and
stop for approval before editing. Reclassify a real but broader issue that does
not violate the frozen contract as a follow-up. Reject speculative, duplicate,
or pre-existing findings with a concrete explanation.

If a reviewer repeats a rejected finding without new evidence, do not change
code to appease it. Record the disposition once and preserve the contract.

## Fix the smallest proven blocker

Apply fixes, commit them, and push them only when the user or goal authorizes
those actions. Otherwise, report the validated blocker and smallest fix without
changing the working tree or remote branch.

For each accepted blocker:

- Reproduce it before editing when practical.
- Change only what restores the violated guarantee.
- Add or update the narrowest regression test when the behavior is testable;
  otherwise document the verification performed.
- Avoid new abstractions, generalized hardening, and opportunistic cleanup.
- Preserve unrelated user changes and repository instructions.
- In a stack, attribute the regression to the owning PR or commit. Do not
  rewrite or force-push stack history without authorization.
- Run focused tests and type or formatting checks proportional to the change.
- Stage only the authorized fix files or hunks after inspecting the staged diff;
  never include unrelated user changes.
- Commit the verified fixes as one coherent batch for the iteration and push
  normally to the existing PR head branch. Never force-push or rewrite history.
- Verify the live PR head matches the pushed commit before the next review pass.

Then inspect the cumulative diff again for interactions among the fixes.

## Review changed comments

Treat comment hygiene as a bounded quality pass:

- Inspect only comments added or modified by the reviewed diff.
- Prefer deleting comments that merely restate clear code.
- Keep concise comments that explain non-obvious constraints, invariants, or
  tradeoffs that the code cannot express.
- Remove PR-history narration, reviewer-directed prose, generic filler, and
  speculative explanations.
- Rewrite misleading comments to match current behavior.
- Do not expand this pass into unrelated code cleanup or replace one verbose
  comment with different verbose prose.

When edits are authorized, resolve accepted comment cleanup in the same pass.
Like every other edit, comment cleanup must be included in the next cumulative
native review before completing the loop.

## Continue or stop

Continue automatically only while within the scope circuit breaker.

After any fix or comment edit and its verification, update the target
coordinates and run the selected native reviewer again with the same contract.
The target must cover the fixed base through the exact head plus all current
staged, unstaged, and intended untracked changes. Do not grow the prompt with
every past finding or add new review focus; only those target coordinates may
change between passes. Emit a paste-ready `/review` command only when neither
native runner is callable.

Immediately before returning `safe for review` for a live PR, re-resolve the
PR head. If it differs from the head covered by the final pass, the verdict is
stale; update the target coordinates and run another pass.

Stop with:

- `safe for review` when no validated in-scope blocker or scope decision remains
  and required verification passes, after accepted changed-comment cleanup is
  resolved;
- `scope-check` when a validated issue requires approval to cross the frozen
  implementation boundary or consecutive findings occur in loop-added code;
- `blocked` when completion needs another unresolved user decision, unavailable
  authority, or external state change;
- `fix-first` only while a validated in-scope blocker remains.

Follow-ups do not prevent `safe for review`. A review that returns `safe for
review` completes the loop; it is not a reason to start another adversarial
pass. Mark an active goal complete only after this stopping condition is met.

## Report each iteration

Keep the update compact:

1. Verdict: `fix-first`, `scope-check`, `safe for review`, or `blocked`.
2. Accepted blockers and the guarantees they violate.
3. Reclassified or rejected findings and why.
4. Accepted changed-comment cleanup and the smallest deletion or rewrite.
5. Fixes and verification completed.
6. The next native review pass, or the exact fallback `/review` command when
   another pass is required but no native runner is callable.
