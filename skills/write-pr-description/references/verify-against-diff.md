# Check the description against the diff

A description that a reviewer trusts instead of the code is only as good as its accuracy.
Prompt rules alone do not get it there: writers who follow good style rules still produce bodies with claims the code contradicts.
So the check is a separate step with its own output, not a reread.

## Who runs it

Best: a subagent that gets **only** the diff, the repository at the PR head, and the draft body. It does not get this session's history, the plan, or your reasoning. Its job is to find where the body and the code disagree, not to judge style.

Acceptable when no subagent is available: run it yourself, but work from a fresh `git diff` output, not from memory of what you wrote.

## Inputs

```bash
git --no-pager diff <base>...HEAD > /tmp/pr.diff
git --no-pager diff --stat <base>...HEAD
# the draft body, e.g. /tmp/pr-body.md
```

## Procedure

### 1. Extract claims

List every sentence in the body that asserts a fact. Split compound sentences so each item is one claim. Kinds of claims:

- **Behavior:** "Requests with an expired token now return 401 instead of 500."
- **Non-change:** "Read paths are unchanged." "No schema change."
- **Mechanism:** "The retry reuses the same transaction ID."
- **Scope:** "Only the staging project gets the new role."
- **Validation:** "The new test fails on main and passes here." "Ran the load test: p99 went from 40 ms to 12 ms."
- **Provenance:** "Fixes #123." "As decided in the design doc."

Claims in examples, code blocks, and diagrams count too: each arrow in a diagram and each before/after pair in an example is a claim. Skip opinions and the why (motivation is checked against its source in step 1 of the skill, not against the diff).

### 2. Mark each claim

| Mark | Meaning |
| --- | --- |
| verified | The diff or the repository at HEAD shows it. Cite `file:line`. |
| partial | True for some of the cases the claim covers, not all. Say which part fails. |
| unsupported | Nothing in the diff or repository shows it either way (common for validation and provenance claims). |
| contradicted | The code shows something else. Cite `file:line` and what the code actually does. |

Check non-change claims with care. "X is unchanged" is contradicted by any hunk that changes X's behavior, including through a shared helper, a default value, or a config file.

Validation claims are verified only by evidence a reader can check: a CI run link, command output in the session, a test that is in the diff and was observed to run. A test that exists in the diff is not evidence that it ran.

### 3. Find uncovered changes

Walk the diff hunk by hunk. For each hunk that changes behavior, find the claim that covers it. List every hunk with no covering claim. Ignore hunks that are purely mechanical (formatting, renames with no semantic change, generated files, lockfiles), but list them under "mechanical" so the reviewer can see they were classified, not missed.

Pay extra attention to: default values, error handling paths, config and flag files, permission and auth lines, migrations, public signatures, and deleted code. Removed lines change behavior as often as added ones.

### 4. Find history leaks

List every sentence that describes this branch's development instead of its final state: versions or iterations ("v2", "the new approach", "originally", "after trying", "instead of the earlier"), "now" or "previously" that compares against an earlier commit of the branch rather than the base, and exploration narrative. Each one is a defect. Do the same for comments the diff adds.

### 5. Report

Write the result in this form (it is for the writer, not for the PR body):

```text
Claims: 14 verified, 1 partial, 2 unsupported, 1 contradicted

contradicted:
- "Read paths are unchanged" -> storage/reader.go:88 now calls normalizeKey(), which also lowercases keys on read.

partial:
- "All callers pass a deadline" -> api/batch.go:41 passes context.Background() with no deadline.

unsupported:
- "Fixes #4312" -> no issue reference in commits or branch name.
- "Load test p99 improved to 12 ms" -> no run or output in the session.

uncovered behavior changes:
- config/defaults.yaml:12 raises max_retries from 3 to 5.

history leaks:
- "The new approach batches writes instead of the earlier per-row version" -> the base has no per-row version; describe batching only.

mechanical (not described, safe to skim):
- *_pb.go regenerated; go.sum.
```

### 6. Fix

- **Contradicted:** fix the body. If the code is the one that is wrong, tell the user; do not quietly change the code inside a description task.
- **Partial:** narrow the claim to what is true, or name the exception.
- **Unsupported:** add the evidence (a link, the command and its output), mark it as not checked, or delete it.
- **History leak:** rewrite the sentence to describe the final state only, or delete it.
- **Uncovered:** describe the change in the body, or add it to the "Read the code here" list with the question a reviewer should answer.

Run the check again on the changed parts. The body is ready when nothing is contradicted and every uncovered behavior change is either described or listed for the reviewer to read.

## Limits

This check finds disagreement between the body and the code. It does not prove the code is correct, and it does not check the why. A checker can also be wrong: when it marks a claim contradicted, look at the cited line yourself before rewriting.
