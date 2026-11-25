---
name: write-pr-description
description: Writes or refreshes a pull request title and body so a reviewer can understand what the change does, why, and how it is designed without reading the code, then checks every claim in the body against the diff. Use whenever opening a PR, filling a PR template, rewriting a stale description, or when the user says "open a PR", "write this up", or "put this up for review".
---

# Write a PR description

The goal of this skill is a description a reviewer can **trust instead of the code**.
A reviewer who reads only the body should know what the change does, why it exists, how it is designed, what it can break, and which parts they still have to read themselves.

That goal sets three requirements:

1. **Complete.** Every behavior change is in the body. A behavior change that is only in the diff is a behavior change nobody reviewed.
2. **True.** Every claim matches the diff. The body is the one part of a PR that no CI checks, so this skill checks it (step 5).
3. **Honest about its limits.** Where the body does not fully describe the code, it says so and names the code the reviewer must read.

The description is also a permanent record. Someone who finds this change through `git log` or `git blame` in two years cannot ask you why. Write for them too.

## Works with pstack

This skill assumes [pstack](https://github.com/cursor/plugins/tree/main/pstack) is installed and leaves to it what it already covers. pstack's skills are user-invoked only, so read their `SKILL.md` files from where pstack is installed instead of calling them. If pstack is not installed, skip those steps and say in the PR that they were skipped.

| Need | Use |
| --- | --- |
| Prose style and AI tells | pstack `technical-writing` and `unslop`, on the finished body |
| What the change can break | pstack `blast-radius` |
| The decision trail | pstack `show-me-your-work` log (`decisions.tsv` or `.audit/<task>.tsv`), if the run kept one |
| Rationale from history | pstack `why` |
| Plan and PR order | `plan-pr-structure`, and pstack's multi-phase-plan playbook |

**Relation to pstack's opening-a-pr playbook.** That playbook handles the mechanics (worktree, commits, title, opening the PR) and sets a body of at most 40 lines. This skill replaces only its body rules, because a body a reviewer approves from needs more room than a one-minute briefing. Keep that playbook's headings and order, so pstack users see one vocabulary; this skill adds a `## Review guide` section and says what goes under each heading.

## 0. Is this one PR?

State the change's purpose in one sentence without "and".
If you cannot, the PR probably does more than one thing: for example a refactor plus a behavior change, or two unrelated fixes.
Stop and tell the user. Suggest the split (see `plan-pr-structure`) and let them decide. If they keep one PR, describe each part under its own sub-heading and say in the opening that the PR combines them.

A description cannot make a large PR safe to approve unread. A body over about 1,200 words, or a review guide that needs a long reading order, means the PR wants splitting, not a better description.

## 1. Collect the facts

Most of what the body needs is not in the diff.

- **The why.** What breaks, costs, or is impossible without this change, and for whom. Take it from a source: the user's request in this session, a linked issue, a design doc, the plan file, the decision log, a commit body. **Never invent the why.** If no source gives it, ask the user. If they are not available, write "Motivation: not stated in any source; please confirm". An invented rationale is the claim a reviewer is least able to check.
- **Decisions.** Each choice with a real alternative, the alternative, and why it lost. Take them from the decision log and the session, not from memory of the finished code. Do not manufacture alternatives.
- **The design.** New or changed contracts: APIs, schemas, data formats, invariants, state machines, concurrency rules, error semantics.
- **Behavior changes,** including side effects, and what is deliberately unchanged.
- **Blast radius.** For anything beyond a mechanical change, run pstack `blast-radius` and use its confirmed risks and the one fact it proved.
- **Verification.** What you actually ran and what it showed.
- **Uncertainty.** What you are unsure of, what you inferred, what you did not handle.

When you did not write the branch yourself:

```bash
git --no-pager log --format='%h %s%n%b' <base>..HEAD   # commit bodies carry the why
git --no-pager diff --stat <base>...HEAD
git --no-pager diff <base>...HEAD
gh pr view <n> --json title,body,baseRefName,commits,files   # if a PR exists
```

Verify claims you carry over. "Same pattern as X" or "no behavior change" in a commit message is an assertion about code. Check it before you repeat it.

If the session that did the work is still open, write the description there. A fresh session has only the diff and commit messages.

## 2. Follow the repository's template

Look for `.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/`, or a contributing guide. If one exists, its headings, order, and instructions win over both this skill and pstack's playbook. Put the content below inside its sections, keep machine-read markers, and answer a section that does not apply in a short clause ("No schema change") instead of deleting it.

## 3. Write top-down

Order the body from coarse to fine, so a reader can stop at any point with a correct picture: a skimmer gets the what and why from the first lines, a reviewer weighing the design reads into the decisions, and a close reviewer reaches the details. Drop any heading the change does not need. A one-line fix is a title and two or three sentences under `## Why`.

**Title.** `type(scope): subject`, per pstack. The subject says specifically what the PR does and stands alone in `git log --oneline`: "Retry commits that abort with a lock conflict", not "Fix bug".

**`## Why`.** First, one to three sentences: what the change does and why, so a reviewer who reads only this knows whether the PR is theirs and what is at stake. Lead with the effect on a person or system, not a file or class name. Then, if needed, the problem in more depth: who has it, how often or how bad, and links. Restate what a link says in a sentence; links rot. The problem must stay true if a different solution were picked: "there is no cache" is a solution, not a problem.

**`## Review guide`** (added by this skill; place it right after Why). This is what makes trusting the body safe. Each part is optional:

- **Read the code here.** The places the body does not fully specify, so a reviewer cannot approve them from the text: tricky concurrency, security-relevant lines, anything you are unsure of, and anything step 5 left uncovered. Name the file and symbol, and the question to answer. If there are none, write "The description covers every behavior change in the diff."
- **Where I am unsure.** The call you made, why, and what would make it wrong.
- **Skimmable.** Generated code, renames, mechanical edits, lockfiles, each with a clause on why it is safe to skim.
- **Reading order,** only for about ten or more non-generated files: the new contract, its first consumer, the wiring, the mechanical rest.

**`## What changed`.** The design, at the level of contracts and behavior, never a file-by-file list:

- The new or changed interface, schema, or format. Show a short before and after snippet when the shape matters.
- The mechanism, where the code does not make it obvious: invariants, ordering, concurrency, failure handling, state transitions.
- **Every behavior change,** including side effects. For a permission change, state the direction: what is now allowed, what is now refused.
- **What does not change.** Name the behavior a reviewer might worry about and say how you know it is preserved.

For a bug fix, use **Before** (the faulty mechanism, naming the function or path at fault) and **After** (what the code does now, at the same level of detail).

**`## Scope`.** What is deliberately out of this PR, and, for a planned series, which part this is, with a link to the plan or the previous PR.

**`## Tradeoffs`.** Each decision with its rejected alternative and why it lost, plus the shortcomings of the chosen approach. This is the content a reviewer most needs and can least recover from the diff.

**`## Blast Radius`.** From pstack `blast-radius`: the confirmed risks with their cost, the one fact the change is safe because of and how far it was proved (pointed at a line, ran a script, reproduced), one-way doors (stored formats, public APIs, migrations), and rollback.

**`## Verification`.** What ran and what it showed, with a link or command. When something did not run, use one of these forms:

- "Not run: `<command>`. Needs `<environment>`, which this session does not have."
- "Tests added but not run."
- "Expected from CI job `<name>`: `<result>`." (an expectation, not an observation)
- "Verified by `<person>` per `<commit or comment>`." (attributed, not restated as yours)

Name the regression each new test would catch and what is **not** covered. Never write an intention as a result.

### PR-specific writing rules

Run pstack `technical-writing` and `unslop` over the finished body. In addition, two rules specific to PR bodies:

- **Describe the branch as it is, not how it got there.** No "this PR previously..." or "after review I removed...". Keep the reasoning, drop the transition: "There is no unit test at this layer, because it would only restate the match arms."
- **Cut what does not help the reviewer decide something:** a walkthrough of the diff, paraphrases of new code comments, lists of test names, a fact stated twice. **Never cut** the why, a behavior change, a decision with its alternative, a risk, or a "read the code here" pointer. Length follows the design to explain, not the diff size: a mechanical change is a short paragraph, a focused fix 100 to 300 words, a feature 300 to 900. Over about 1,200, go back to step 0.

## 4. Self-check

- Does the title say specifically what the PR does?
- Is every "why" traceable to a source, or marked as not stated?
- Could a reviewer predict the changed code's behavior from the body alone, except where "Read the code here" says otherwise?
- Is every behavior change, including side effects, listed?
- Does every verification claim describe something that ran, or say it did not?
- Does the body describe the branch as it is now?

## 5. Check the body against the diff

Every time, before creating or updating the PR. Follow [references/verify-against-diff.md](references/verify-against-diff.md). In short:

1. Extract every factual claim from the body.
2. Mark each one **verified**, **partial**, **unsupported**, or **contradicted** against the diff, citing `file:line`.
3. List the behavior changes in the diff that the body does not mention.
4. Fix the body until nothing is contradicted, and each uncovered change is either described or listed under "Read the code here".

Give the check to a subagent that receives only the diff and the body, not this session's history. A checker that shares your context tends to agree with you. Do not post while any claim is contradicted.

## 6. Create or update the PR

Use pstack's opening-a-pr playbook for the mechanics if you are in a pstack run; otherwise `gh pr create --body-file`. When updating an existing PR, read the current body first and keep what a person put there: screenshots, links, ticked boxes, a personal note at the top, because `gh pr edit --body-file` replaces the whole body. After later pushes that change behavior, scope, or risk, rewrite the body to match and run step 5 again. Do not append a revision log.

## Examples

[references/examples.md](references/examples.md) has full examples, each followed by notes on why it works: a small fix, a bug fix with a non-obvious cause, a feature with a design to explain, one step of a planned series, and an annotated bad draft. Read it before your first description in a session.

[references/house-examples.md](references/house-examples.md) holds examples from this user's own merged PRs. When it has entries, they outrank the generic ones on tone, length, and conventions.

## Credit

Builds on Google's [Writing good CL descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html) (CC BY 3.0) and on ideas from the Warp, PostHog, Shopware, tldraw, and Sentry PR-description skills. See the repository README.
