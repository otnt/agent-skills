---
name: design-tests
description: Designs the tests for a change from its contract instead of its code, keeps only tests that catch a regression no other test catches, proves each one fails when the code breaks, and gives the reviewer a short test contract to check instead of the test code. Use whenever you are about to add or change unit or integration tests, when a PR is about to include tests, or when asked to review or slim down the tests in a PR.
---

# Design tests

Your default is to read the code you wrote and write tests that agree with it. Those tests pass on a bug as easily as on correct code, and nobody reads them. This skill changes that default in four ways: expected values come from a source other than the code, each test earns its place and proves it can fail, tests run at the lowest level where a caller sees the outcome, and the reviewer gets a short contract instead of test code.

Project rules win. Read the repo's testing notes and the nearest test files first.

## Works with pstack

pstack skills are user-invoked, so read their files yourself:

- `principle-test-behavior-not-implementation` defines behavior versus implementation. This skill does not restate it.
- `tdd` is the loop for a bug fix: failing test first. Use it, plus steps 1, 2 and 5 here.
- `correct` is for a test mistake that keeps coming back: enforce it with a check, not more prose.

## 1. Write the contract before you look at the implementation

List the behaviors the change promises, one row each: **given** (input or state), **then** (what a caller observes), **source**. A source is not the code under test: it is the issue, a spec, the documented contract, how existing callers use it, or the user's words. Include the cases a source implies and a happy path skips: empty, the boundary and one past it, invalid input, the error a caller branches on.

- **Where code and source disagree, stop and report it.** Never resolve it by making the test match the code.
- No source for a behavior: ask once. If you cannot ask, mark the row's source `assumed` so the reviewer checks it.
- For a risky behavior, have a subagent write the expected values from the contract and public signatures only, without the implementation.
- A test that pins current behavior on purpose (before a refactor, or legacy code with no spec) is a **pin**. Give it source `current behavior`.

## 2. Keep only tests that earn their place

For each row, answer in one sentence: **what realistic regression does this catch that no existing test catches?** Name the code path and the input that breaks it. "Coverage" is not an answer.

- Search existing tests first. Add your case to the nearest test that covers the behavior, as a table or parameterized case.
- One behavior gets one test or one table. Add a variant only for a different rule or boundary.
- Skip framework behavior, getters, constants, defaults, and checks that a mock was called the way the code calls it.
- Tests you wrote only to check your work while coding are scaffold. Delete them before committing unless they pass this gate.
- If the test diff is larger than the code diff, say why in the PR.

## 3. Pick the level and the collaborators

Test at the lowest level where the behavior exists and a caller sees its outcome: a return value, an error, stored state, or the request that crosses a boundary.

- Drive the unit through its public interface. A unit can be several modules tested through their entry point with their real in-process collaborators.
- Fake only true boundaries: network, clock, randomness, other services. Never mock your own code. Behavior that lives in a query runs against the real database the test setup provides.
- Hard to test cheaply means the logic is tangled: extract a pure function, test it, and keep one higher-level test that proves the wiring.
- Use a property-based test when the code claims a rule for every input, such as a round trip or a match with a simple reference.

## 4. Write the test

- Expected values are literals from the contract, never computed by the code under test, its helpers, or a copy of its formula. A value you cannot write by hand gets an independent readback or a property.
- Compare the whole outcome a caller sees in one assertion where you can.
- No logic that recomputes results inside a test. No sleeps; pin clocks and seeds.
- Name the test after the behavior and its condition, in the contract's words.

## 5. Prove each test can fail

Break the line each new or changed test guards (flip the condition, drop the clause, or revert your change), run it, and confirm it fails for the reason you named in step 2. Restore the code. A test that stays green guards nothing: rewrite or delete it.

This shows the test has teeth, not that its expected value is right. Only the source in step 1 shows that.

## 6. When an existing test fails

Do not edit its expected value, loosen it, skip it, or delete it to get green unless you can cite the source that changed the requirement. Otherwise stop and report the test, what it expects, what the code now does, and which you think is right.

## 7. Report the test contract

Put this table in the PR body, under Verification when using `write-pr-description`:

| Behavior | Given | Then | Source | Test | Fails when |
| --- | --- | --- | --- | --- | --- |

`Fails when` is the break you ran in step 5. Mark `assumed` sources and pins, then add one line on what is not tested and why. The reviewer checks Given, Then and Source instead of reading the tests. More than about 15 rows suggests a split (`plan-pr-structure`).

## Reviewing or slimming existing tests

1. Rebuild the contract from the sources, without reading the tests.
2. Map each test to a row. A test that maps to no row, or restates the implementation, is a delete candidate; tests on the same row are merge candidates.
3. Run step 5 on the tests you keep.
4. Report keep, merge and delete with the lines saved, plus rows no test covers. Change nothing until the user agrees, unless they asked you to apply it.
