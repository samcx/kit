---
name: loop-review
description: Run scope-bounded adversarial code-review loops for PRs, branches, commit ranges, and cumulative stacks. Use when iterating `/review` findings and fixes until safe for review, using `/goal` to preserve completion criteria, or preventing review loops from expanding scope or becoming endless.
---

# Loop Review

Use `/goal` to preserve one completion contract across turns and `/review` to
obtain an independent, read-only review. Fix only demonstrated violations of
that contract, verify them, and repeat until no in-scope blocker remains.

## Start the loop

Prefer this Codex invocation:

```text
/goal Use $loop-review to adversarially review <PR, branch, or stack> against the scope contract below, fix in-scope blockers, and continue until its stated guarantees are safe for review. <scope contract>
```

Treat the goal objective as the durable outcome, constraints, and definition of
done. If the user explicitly started or requested a goal and goal tools are
available, use them to track it. Do not create a goal from implicit skill
activation alone.

If the scope is not yet clear, establish the contract before starting the goal.
Use `/plan` first when the outcome needs discussion, then put the agreed contract
in `/goal`. Goal text has a 4,000-character limit; for a longer contract, point
the goal to a file containing it.

Slash commands are client actions. Never claim to have run `/goal` or `/review`
when the current surface cannot invoke them. Instead, emit the exact paste-ready
command and wait for its result; the active goal preserves continuity.

## Freeze the review contract

Before the first review:

- Resolve the exact base and head, including every PR in a cumulative stack.
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

## Request the adversarial review

Use custom `/review` instructions with this structure:

```text
/review Adversarially review <exact base through exact head>.

Strict scope contract:
1. <guarantee>
2. <guarantee>

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
- Follow-up: a real issue that requires a new guarantee or broader architecture.
- Pre-existing/out of scope: not introduced or materially worsened here.

Also inspect comments added or modified by this diff for AI slop: comments
that restate obvious code, narrate implementation or PR history, use generic
filler, or overexplain straightforward logic. List concrete candidates
separately as "Comment cleanup" with the smallest deletion or rewrite. Do not
treat comment cleanup as `fix-first` unless a misleading or incorrect comment
violates a stated guarantee.

Return "fix-first" only for an in-scope blocker. List adjacent discoveries as
nonblocking follow-ups. Return "safe for review" when all stated guarantees
hold.
```

Add precise exclusions for adjacent architecture the contract does not promise.
Derive them from the current change; never carry one stack's non-goals into
another.

Review the entire cumulative diff on every pass, not only the latest fix.

## Triage findings before editing

Independently validate every finding against the source. Accept it as a blocker
only when all four required elements are present. Reclassify a real but broader
issue as a follow-up. Reject speculative, duplicate, pre-existing, or
scope-expanding findings with a concrete explanation.

If a reviewer repeats a rejected finding without new evidence, do not change
code to appease it. Record the disposition once and preserve the contract.

## Fix the smallest proven blocker

Apply fixes only when the user or goal authorizes edits. Otherwise, report the
validated blocker and smallest fix without changing the working tree.

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
Comment cleanup alone does not require another adversarial `/review`; inspect
the final diff locally and complete the loop if no functional blocker remains.

## Continue or stop

After fixes and verification, issue the next paste-ready `/review` command with
the same contract. Do not grow the prompt with every past finding; include only
the fixed contract, exact range, and any concise regression focus needed to
exercise the latest changes.

Stop with:

- `safe for review` when no validated in-scope blocker remains and required
  verification passes, after accepted changed-comment cleanup is resolved;
- `blocked` when completion needs an unresolved user decision, unavailable
  authority, or external state change;
- `fix-first` only while a validated in-scope blocker remains.

Follow-ups do not prevent `safe for review`. A review that returns `safe for
review` completes the loop; it is not a reason to start another adversarial
pass. Mark an active goal complete only after this stopping condition is met.

## Report each iteration

Keep the update compact:

1. Verdict: `fix-first`, `safe for review`, or `blocked`.
2. Accepted blockers and the guarantees they violate.
3. Reclassified or rejected findings and why.
4. Accepted changed-comment cleanup and the smallest deletion or rewrite.
5. Fixes and verification completed.
6. The exact next `/review` command, only when another pass is required.
