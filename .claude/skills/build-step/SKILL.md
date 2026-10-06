---
name: build-step
description: Implement one step of the Proofline build plan in docs/build/plan.md (for example S08) exactly as the feature docs specify, with tests, a green pnpm verify, and an updated progress file. Use only when the user asks to build or continue a plan step.
disable-model-invocation: true
argument-hint: "<step id, e.g. S08>"
---

# Build step $ARGUMENTS

You are implementing step **$ARGUMENTS** of `docs/build/plan.md`. The feature docs are the spec. Follow `CLAUDE.md` and the path-scoped rules in `.claude/rules/`.

## 1. Load the step

1. Open `docs/build/plan.md` and find the section headed `$ARGUMENTS`. If there is no such step, list the valid step IDs and stop.
2. Open `docs/build/progress.md`. Every step under **Needs** must be `done`. If one isn't, say which and stop, unless the user explicitly told you to go ahead anyway.
3. Mark the step `in progress` in `docs/build/progress.md`, with today's date.

## 2. Read before you write

- Read every doc section listed under **Read**, completely. Don't skim tables: the state tables, error codes, limits and payloads are the contract.
- Open every mockup listed (PNG for the look, HTML for exact copy and styles).
- Re-read `docs/00-overview.md` section 4 (boundaries) and the platform interfaces you will use (section 5).
- Check `docs/build/questions.md` for answered questions that affect this step.

## 3. Plan

Make a task list, one item per **Build** entry. Each entry maps to `Tnn.m` tasks in the doc's "Implementation plan". For each task, note:

- the files it creates or changes, inside the module's own package;
- the tests it needs (from the acceptance criteria and the doc's Testing section);
- any interface from another module it needs. If that module isn't built yet, use the fake from its `testing` export, and create the fake if it doesn't exist yet, matching the doc's interface exactly.

## 4. Implement, task by task

For each task:

1. Write the tests first for domain logic and for each acceptance criterion.
2. Implement until they pass. Keep `domain/` pure.
3. Run that package's tests: `pnpm --filter @proofline/<pkg> test`, plus `test:int` when the database or Redis is involved.

**When something is unclear or conflicting**, add an entry to `docs/build/questions.md` using its format.

- If the doc or overview gives a default, implement the default and continue.
- If there is no default and the choice changes behaviour a merchant, reviewer or auditor would notice, stop and ask the user. Show the question and your recommended answer.
- Never invent business rules, limits, codes or copy.

## 5. Verify

1. Run `pnpm verify` (lint and boundaries, typecheck, unit and integration tests). Fix until green.
2. If the step has screens, run the matching Playwright specs, and compare against the mockups in both `en` and `am`.
3. Go through **Done when** one item at a time, and write down the evidence for each: a test name, a command and its output, or a screenshot path.

## 6. Record and commit

1. Update `docs/build/progress.md`:
   - status `done` (or `blocked` with the reason);
   - the date;
   - a 3–6 line summary of what was built;
   - deviations, which must each be linked to a question;
   - follow-ups.
2. Commit on branch `step/$ARGUMENTS-<short-slug>` (create it from the current main branch if it doesn't exist). Use conventional commits, for example `feat(verification): create pipeline and state machine [$ARGUMENTS]`. Don't push, open a PR or merge unless the user asks.

## 7. Report back

Reply with:

- what was built, as a short list;
- the test totals;
- any questions you raised or defaults you applied;
- anything left for a later step;
- the next step ID and whether its needs are met.

Suggest running `/check-step $ARGUMENTS` for an independent audit.
