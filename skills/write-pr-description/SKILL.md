---
name: write-pr-description
description: Writes or refreshes a pull request title and body so a reviewer can understand what the change does, why, and how it is designed without reading the code, then checks every claim in the body against the diff. Use whenever opening a PR, filling a PR template, rewriting a stale description, or when the user says "open a PR", "write this up", or "put this up for review".
---

# Write a PR description

The goal of this skill is a description a reviewer can **trust instead of the code**.
A reviewer who reads only the body should know what the change does, why it exists, how it is designed, what it can break, and which parts they still have to read themselves.

That goal sets three requirements that ordinary "write a nice summary" skills do not:

1. **Complete.** Every behavior change is in the body. A behavior change that is only in the diff is a behavior change nobody reviewed.
2. **True.** Every claim matches the diff. The body is the one part of a PR that no CI checks, so this skill checks it (step 6).
3. **Honest about its limits.** Where the body does not fully describe the code, it says so and names the code the reviewer must read.

The description is also a permanent record. Someone who finds this change through `git log` or `git blame` in two years cannot ask you why. Write for them too.

## 0. Is this one PR?

Before writing anything, state the change's purpose in one sentence without "and".
If you cannot, the PR probably does more than one thing: for example a refactor plus a behavior change, or two unrelated fixes.
Stop and tell the user. Suggest the split (see the `plan-pr-structure` skill) and let them decide. If they keep one PR, describe each part under its own heading and say in the opening that the PR combines them.

A description cannot make a large PR safe to approve unread. A long reading order or a body over about 1,200 words is a sign that the PR wants splitting, not a better description.

## 1. Collect the facts

Gather these before drafting. Most of them are not in the diff.

- **The why.** What breaks, costs, or is impossible without this change, and for whom. Take it from a source: the user's request in this session, a linked issue, a design doc, a plan file, a commit body. **Never invent the why.** If no source gives it, ask the user. If they are not available, write "Motivation: not stated in any source; please confirm" rather than a plausible guess. An invented rationale is worse than none, because it is the claim a reviewer is least able to check.
- **Decisions.** Each choice with a real alternative, the alternative, and why it lost. Use what was actually discussed or tried in this session or in commits. Do not manufacture alternatives to fill the section.
- **The design.** New or changed contracts: APIs, schemas, data formats, invariants, state machines, concurrency rules, error semantics.
- **Behavior changes,** including side effects of the main change, and what is deliberately unchanged.
- **Blast radius.** What can break beyond the touched files. See the list in step 3.
- **Validation.** What you actually ran and what it showed.
- **Uncertainty.** What you are unsure of, what you inferred, what you did not handle.

Commands, when you did not write the branch yourself:

```bash
git --no-pager log --format='%h %s%n%b' <base>..HEAD   # commit bodies carry the why
git --no-pager diff --stat <base>...HEAD
git --no-pager diff <base>...HEAD
gh pr view <n> --json title,body,baseRefName,commits,files   # if a PR exists
```

Verify claims you carry over. A commit message that says "same pattern as X" or "no behavior change" is an assertion about code. Open X, or read the removed lines, before you repeat it.

If the session that did the work is still open, write the description there. A fresh session has only the diff and commit messages, and has lost most of the why.

## 2. Follow the repository's template

Look for `.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/`, `docs/pull_request_template.md`, or a contributing guide.
If a template exists, its headings, order, and instructions win. Put the content below inside its sections. Keep machine-read markers (changelog lines, issue keys). Answer a section that does not apply in a short clause ("No schema change") instead of deleting it.

If no template exists, use the structure in step 3.

## 3. Write top-down

Order the body from coarse to fine, so a reader can stop at any point with a correct picture: a skimmer gets the what and why from the first lines, a reviewer weighing the design reads into the decisions, and a close reviewer reaches the details.

Use only the sections the change needs. A one-line fix is a title and two or three sentences.

### Title

A short imperative sentence that says specifically what the PR does: "Retry Spanner commits that abort with a lock conflict", not "Fix bug" or "Update commit logic". It must stand alone in `git log --oneline`. Follow the repository's convention (for example `fix(scope): ...`) if it has one.

### Opening paragraph (no heading)

One to three sentences: what the change does and why. A reviewer who reads only this paragraph knows whether the PR is theirs to review and what is at stake.
Lead with the effect on a person or system, not with a file, class, or flag name.

### `## Context`

Only when the opening cannot carry the why. The problem, who has it, how often or how bad, and links (issue, design doc, incident, earlier PR). Restate what a link says in a sentence; links rot and some readers cannot open them.
The problem must stay true if a different solution were picked. "There is no cache" is a solution, not a problem.

### `## Design`

The heart of the description for any non-trivial change. Describe the shape of the solution at the level of contracts and behavior, not line by line:

- The new or changed interface, schema, or data format. Show a short before and after snippet when the shape matters.
- The mechanism, where it is not obvious from reading the code: invariants, ordering, concurrency, failure handling, state transitions. A small diagram helps for flows across components.
- **Behavior changes**, listed explicitly, including side effects. For a permission or access change, state the direction: what is now allowed that was not, and what is now refused.
- **What does not change.** Name the behavior a reviewer might worry about and say it is preserved, and how you know.

For a bug fix, use **Before** (the faulty mechanism, naming the function or path at fault) and **After** (what the code does now, at the same level of detail).

### `## Decisions`

Each decision with its rejected alternative and the reason it lost. This is the content a reviewer most needs and can least recover from the diff. One or two sentences each. Skip choices nobody would question.

### `## Risks and limitations`

Shortcomings of the approach, the blast radius, one-way doors (stored formats, public APIs, migrations), rollout and rollback notes, and what to watch after it ships. Say plainly what the change does **not** handle.

Wide blast radius, worth a line even when the diff is tiny: permissions and auth, tenancy boundaries, network exposure, schema migrations and stored formats, public API or wire contracts, feature-flag defaults, shared foundational code, anything other code already depends on.

### `## Validation`

Report what ran and what it showed, with a link or command. Use these honest forms when something did not run:

- "Not run: `<command>`. Needs `<environment>`, which this session does not have."
- "Tests added but not run."
- "Expected from CI job `<name>`: `<expected result>`." (an expectation, not an observation)
- "Verified by `<person>` per `<commit or comment>`." (attributed, not restated as yours)

Name the regression each new test would catch, and what is **not** covered. Do not list test function names. Never write an intention as a result. Never claim manual testing that did not happen.

### `## Review guide`

This section is what makes trusting the description safe. Put it directly under the opening paragraph when it matters, so nobody scrolls past checklists to find it. It has three parts, each optional:

- **Read the code here.** The places where the description does not fully specify the behavior, so a reviewer cannot approve them from the body alone: tricky concurrency, security-relevant lines, anything you are unsure of, anything step 6 flagged as uncovered. Name the file and symbol, and the question. If there are none, say "The description covers every behavior change in the diff."
- **Where I am unsure.** The call you made, why, and what would make it wrong.
- **Skimmable.** Generated code, renames, mechanical updates, lockfiles, with one clause saying why they are safe to skim.

A reading order is useful only for PRs with about ten or more non-generated files. Order by dependency: the new contract, its first consumer, the wiring, then the mechanical rest.

### `## Out of scope`

Follow-ups deliberately left out, so a reviewer does not ask for them. Link the planned PR or issue if one exists. If this PR is part of a planned series, say which part it is and link the plan or the previous PR.

## 4. Write plainly

- Short sentences, about 25 words at most. One fact per sentence or bullet.
- Active voice with a stated subject. The subject is the change or the code, not "I".
- One term per concept for the whole body. Do not vary words for style.
- No self-grading words: "robust", "comprehensive", "clean", "elegant", "properly".
- No explanations of how standard tools or languages work. Explain this change.
- No narration of the branch's history ("this PR previously...", "after review I removed..."). The reviewer sees one state against the base. Describe that state. Keep the reasoning, drop the transition: "There is no unit test at this layer, because it would only restate the match arms."
- Prose and bullets for reasoning; tables only for comparing several items across the same columns; code blocks for interfaces, config, and before/after shapes.

## 5. Cut

Drafts run long in the parts that feel obligatory. For each paragraph, name the decision it helps the reviewer make. If you cannot, delete it. Usual cuts:

- A file-by-file or line-by-line walkthrough of the diff.
- Paraphrases of code comments the diff adds.
- Lists of test names; "all tests pass" when CI already reports it.
- The same fact stated twice.
- Sentences about the description itself.

**Never cut** the why, a behavior change, a decision with its alternative, a risk, an open question, or a "read the code here" pointer. These are why the description exists. Cut mechanism detail before any of them.

Length follows the amount of design to explain, not the diff size. Rough anchors: a mechanical change, a short paragraph. A focused fix, 100 to 300 words. A feature or a new mechanism, 300 to 900 words. Over about 1,200 words, go back to step 0.

## 6. Check the body against the diff

Do this every time, after drafting and before creating or updating the PR. Follow [references/verify-against-diff.md](references/verify-against-diff.md). In short:

1. Extract every factual claim from the body as a list.
2. Mark each claim against the diff and repository: **verified**, **partial**, **unsupported**, or **contradicted**, with the file and line that decides it.
3. List the behavior changes in the diff that the body does not mention.
4. Fix the body: correct every contradicted claim, back or remove every unsupported one, and either describe each uncovered change or add it to "Read the code here".

If you can start a subagent, give the check to one that receives only the diff and the body, not this session's history. A checker that shares your context tends to agree with you.

Do not post the description while any claim is contradicted.

## 7. Self-check

- Does the title say specifically what the PR does?
- Does the opening paragraph give the what and the why, without a file name in the first sentence?
- Is every "why" traceable to a source, or marked as not stated?
- Could a reviewer predict the behavior of the changed code from the body alone, except where "Read the code here" says otherwise?
- Is every behavior change, including side effects, listed?
- Does every validation claim describe something that ran, or say it did not?
- Did the claim check pass with no contradicted claims?
- Does the body describe the branch as it is now, not how it got there?

## 8. Create or update the PR

When updating an existing PR, read the current body first and keep what a person put there: screenshots, links, ticked boxes, a personal note at the top. `gh pr edit --body-file` replaces the whole body.

```bash
gh pr view <n> --json body --jq .body > /tmp/pr-body.md   # edit, keep human content
gh pr edit <n> --title '<title>' --body-file /tmp/pr-body.md
gh pr create --draft --title '<title>' --body-file /tmp/pr-body.md
```

After later pushes that change behavior, scope, or risk, rewrite the body to match the branch, and run step 6 again. Do not append a revision log.

## Examples

[references/examples.md](references/examples.md) has full examples: a small fix, a bug fix with a non-obvious cause, a feature with a design to explain, and one step of a planned series. Each is followed by notes on why it works. Read it before writing your first description in a session.

[references/house-examples.md](references/house-examples.md) holds examples taken from this user's own merged PRs. When it has entries, they outrank the generic examples on tone, length, and section names.

## Credit

Builds on Google's [Writing good CL descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html) (CC BY 3.0) and on ideas from the Warp, PostHog, Shopware, tldraw, and Sentry PR-description skills. See the repository README for details.
