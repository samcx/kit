---
name: goal-review
description: Run scope-bounded adversarial code-review loops for PRs, branches, commit ranges, and cumulative stacks. Use when automating independent reviews with a read-only subagent, iterating findings and fixes until safe for review, using `/goal` to preserve completion criteria, or preventing review loops from expanding scope or becoming endless.
---

# Goal Review

Use `/goal` to preserve one completion contract across turns and a fresh,
read-only subagent to obtain an independent review. Fix only demonstrated
violations of that contract, verify them, and continue automatically only while
within the scope circuit breaker.

## Bootstrap the loop through `/goal`

`/goal` is a built-in command that only the user can run. The Skill tool cannot
invoke it, so never claim this skill set a goal.

Decide which mode applies:

- **Loop mode**: the invoking request already contains a full scope contract
  (it begins `Use /goal-review to adversarially review` and includes `Scope
  contract:`), which means `/goal` delivered it. Skip the bootstrap and run the
  review loop.
- **Bootstrap mode**: anything else. Do not begin the review.

In bootstrap mode:

1. Resolve the exact target, base, head, guarantees, non-goals, named entry
   points, temporal boundary, allowed implementation surface, allowed fixes,
   and required verification.
2. Construct the goal condition using this shape. Do not include `/goal` in the
   clipboard payload:

   ```text
   Use /goal-review to adversarially review <target> at <head> against <base>. Scope contract: <guarantees, non-goals, named entry points, temporal boundary, allowed implementation surface, allowed fixes, and required verification>. Authorize editing, committing, and pushing validated fixes to the existing PR head branch; never force-push or rewrite history. Fix validated in-scope blockers, verify each fix, and continue with fresh independent review passes only while within the scope circuit breaker. Do not broaden scope. The goal is met when the latest iteration report prints `Verdict: safe for review` for the re-resolved live head with required verification passing, or prints `Verdict: scope-check` or `Verdict: blocked` and waits for the user. Finish with the final diff, verification, pushed head, and verdict.
   ```

3. Keep the condition within the 4,000-character `/goal` limit. For a longer
   contract, write it to `~/.claude/goal-review/<repo>-<target>.md` (outside
   the reviewed repository) and have the condition point to that file.
4. Copy the exact condition with `printf '%s' '<safely shell-escaped
   condition>' | pbcopy`. Run that pipeline directly; do not wrap it in
   `sh -c` or `zsh -lc`, and do not use redirection, command substitution, or a
   temporary file. Read it back with `pbpaste` and verify it matches
   byte-for-byte.
5. Tell the user to type `/goal `, paste the clipboard contents, and submit.
   Recommend auto mode so goal turns run without per-tool prompts. Then stop.

If clipboard access is unavailable, make the entire response the exact
condition so the user can copy it, then type `/goal ` and paste it.

If the user explicitly asks to run without `/goal`, run the loop in the current
turn instead.

If the scope is not yet clear, establish the contract before producing the goal
condition. Use plan mode first when the outcome needs discussion, then put the
agreed contract in the condition.

### Write for the goal evaluator

After each turn, a small model reads the conversation and judges whether the
goal condition holds. It cannot run commands or read files. Therefore:

- End every iteration with the report below, starting with an exact
  `Verdict: <verdict>` line.
- Print the reviewed head SHA and the verification command results in the
  transcript; the evaluator only trusts what it can read.
- Do not print `Verdict: safe for review` speculatively or as a plan.

## Run the independent reviewer

The parent agent controls the loop. A reviewer performs exactly one review pass:
it must not spawn agents, invoke `/goal-review`, `/code-review`, or another
reviewer, or modify files.

Prefer these runners in order:

1. **Read-only subagent.** Launch one Agent tool call with `subagent_type:
   "Plan"`, which cannot edit or write files but can read code and run shell
   commands such as `git diff`. Use `Explore` if `Plan` is unavailable, or
   `general-purpose` as a last resort with the no-modification instruction
   stated in the prompt. Pass the custom review instructions below as the
   prompt. The subagent starts with a fresh context, so it does not share the
   parent's reasoning about the fixes.
2. **Headless CLI.** Only when the Agent tool is unavailable: set the process
   working directory to the exact repository worktree and run
   `claude -p --permission-mode plan "<custom review instructions>"`. Pass the
   instructions directly as the prompt argument. Do not use stdin, a shell
   wrapper, pipe, redirection, command substitution, or a temporary prompt
   file.

Do not substitute a same-context self-review or the built-in `/code-review`
skill; neither enforces the frozen contract or returns `scope-check`.

Run exactly one reviewer per pass. Wait for it to finish before triaging its
output; do not relaunch it because it is slow. If it fails, diagnose and retry
once when recoverable.

After each pass, confirm `git status --porcelain` matches its pre-review state.
If the reviewer changed the worktree, discard only those changes, record the
violation, and rerun the pass.

The reviewer does not run verification that writes to the worktree, such as
builds or test runs that emit artifacts. The parent runs required verification
in its own session and uses that result.

The parent performs authorized fixes using the current session's permissions. A
skill cannot grant or auto-accept permissions; if the target worktree needs
access the session lacks, tell the user rather than repeatedly requesting it.

## Freeze the review contract

Before the first review:

- Resolve the exact base and head, including every PR in a cumulative stack.
- Inspect the working tree before every pass. When authorized fixes are not
  committed, include all staged, unstaged, and intended untracked changes in
  the review target; do not assume the unchanged Git head contains them.
- Inspect the live diff and relevant repository instructions, including
  `CLAUDE.md` and `AGENTS.md`.
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
Adversarially review <exact base through exact head> in <absolute worktree
path>, including all current staged, unstaged, and intended untracked changes.

You are the independent review pass, not the loop controller. Do not spawn
agents or invoke /goal-review, /code-review, or another reviewer. Do not modify
files, and do not run builds or tests that write to the worktree.

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
rewrite. Do not treat comment cleanup as "fix-first" unless a misleading or
incorrect comment violates a stated guarantee.

End with exactly one verdict line. Return "scope-check" for a scope decision
without returning "fix-first". Return "fix-first" only for an in-scope blocker.
List adjacent discoveries as nonblocking follow-ups. Return "safe for review"
when all stated guarantees hold.
```

Pass that text unchanged to the selected runner. Independently map the
reviewer's validated findings to a verdict rather than copying its verdict line.

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
review before completing the loop.

## Continue or stop

Continue automatically only while within the scope circuit breaker.

After any fix or comment edit and its verification, update the target
coordinates and run a fresh reviewer again with the same contract. The target
must cover the fixed base through the exact head plus all current staged,
unstaged, and intended untracked changes. Do not grow the prompt with every past
finding or add new review focus; only those target coordinates may change
between passes.

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
pass.

## Report each iteration

Keep the update compact:

1. `Verdict: <fix-first | scope-check | safe for review | blocked>` on its own
   line, followed by the reviewed head SHA.
2. Accepted blockers and the guarantees they violate.
3. Reclassified or rejected findings and why.
4. Accepted changed-comment cleanup and the smallest deletion or rewrite.
5. Fixes and verification completed, with command results.
6. The next review pass, when another pass is required.
