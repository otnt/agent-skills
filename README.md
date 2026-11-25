# agent-skills

Skills for coding agents, in the `SKILL.md` format used by Claude Code and other agents that read Agent Skills.

## Skills

| Skill | When it runs | What it does |
| --- | --- | --- |
| [`plan-pr-structure`](skills/plan-pr-structure/SKILL.md) | Planning time, before or while code is written | Cuts work into small, self-contained PRs with a written plan; splits an existing large branch |
| [`write-pr-description`](skills/write-pr-description/SKILL.md) | When a PR is ready | Writes a title and body a reviewer can approve from without reading the code, then checks every claim against the diff |

The two are separate on purpose. Splitting is cheap before the code exists and painful once one big branch exists, so the structure skill has to run early. The description skill runs on one finished change. They link lightly: the description skill stops and suggests a split when a PR has more than one purpose, and the structure skill records the decisions each description will need.

### Design goal of `write-pr-description`

Most PR-description skills assume the reviewer will still read the diff, so they aim for short summaries. This one aims for a description a reviewer can **trust instead of the code**. That needs three things:

1. **Complete:** every behavior change is in the body, including side effects and what does not change.
2. **True:** a separate check (ideally by a subagent that sees only the diff and the body) marks each claim verified, partial, unsupported, or contradicted, and lists changes the body never mentions. See [`verify-against-diff.md`](skills/write-pr-description/references/verify-against-diff.md).
3. **Honest about its limits:** a "Read the code here" list names the code the body does not fully specify.

It also refuses to invent the why: motivation must come from a source or be marked as not stated.

## Requires pstack

Both skills assume [pstack](https://github.com/cursor/plugins/tree/main/pstack) (Lauren Tan's engineering skills, MIT) is installed, and leave to it what it already covers rather than repeat it:

- Prose style and AI tells: `technical-writing`, `unslop`.
- What a change can break: `blast-radius`.
- Decision trail: `show-me-your-work`.
- Plan file format, unit sequencing, stacking, and opening the PR: poteto-mode playbooks `multi-phase-plan`, `opening-a-pr`, `autopilot-stack`, and `principle-sequence-verifiable-units`.

`write-pr-description` keeps pstack's PR body headings (Why, What changed, Scope, Tradeoffs, Blast Radius, Verification) and adds a Review guide. It deliberately replaces the opening-a-pr playbook's 40-line limit, because a body a reviewer approves from needs more room than a one-minute briefing. Without pstack the skills still work; they skip those steps and say so.

## Install

Copy a skill folder into one of the places your agent loads skills from. For Claude Code:

```bash
# for all your projects
cp -r skills/write-pr-description skills/plan-pr-structure ~/.claude/skills/
# or for one repository
cp -r skills/write-pr-description skills/plan-pr-structure <repo>/.claude/skills/
```

## Add your own examples

The best examples come from your own merged PRs. Add three to five to [`house-examples.md`](skills/write-pr-description/references/house-examples.md), following the template there. When that file has entries, the skill prefers them over the generic examples.

## Credits and sources

The skills are original text, built on ideas from these sources:

- Google Engineering Practices, [Writing good CL descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html) and [Small CLs](https://google.github.io/eng-practices/review/developer/small-cls.html), © Google, [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/). `plan-pr-structure` adapts the definition of a self-contained change, the size anchors, and the split strategies. `write-pr-description` adapts the what-plus-why body, the stand-alone first line, writing for future readers, and stating shortcomings.
- Warp's `write-pr-description` skill ([warpdotdev/common-skills](https://github.com/warpdotdev/common-skills), MIT): verifying commit claims against code, honest validation forms, decisions with rejected alternatives, blast-radius calibration, review-guide focus areas, and the "narrating the branch's history" anti-pattern.
- PostHog's `writing-pr-descriptions` skill ([PostHog/posthog](https://github.com/PostHog/posthog), MIT outside `ee/`): the body must stand without the diff, the need taken from a source and never invented, "the description is the only artifact in a PR that nothing validates", and keeping human content when editing a body.
- Shopware's `pr-description-writing` skill ([shopwareLabs/ai-coding-tools](https://github.com/shopwareLabs/ai-coding-tools), MIT): asking for missing context instead of guessing, and sizing by how much needs explaining rather than by diff size.
- tldraw's `write-pr` skill (source-available license, so ideas only, no text): coarse-to-fine ordering, and the Before/After shape for bug fixes.
- Sentry's `pr-writer` skill ([getsentry/skills](https://github.com/getsentry/skills), Apache-2.0): picking the smallest body shape for the change type.
- Rubric and rjmurillo's `validate-pr-description`: the idea of a separate claim-versus-diff check.

The examples in `examples.md` are fictional and written for this repository.

Sources were read on 2026-11-25. pstack was read from the backnotprop/pstack mirror of cursor/plugins at its 2026-11-25 sync.

## License

Apache-2.0, see [LICENSE](LICENSE). Text adapted from Google's guides remains under CC BY 3.0 with the attribution above.
