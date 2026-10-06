---
name: check-step
description: Audit a finished or in-progress Proofline build step (for example S08) against its feature docs and report every gap, with evidence. Read-only; it doesn't change code. Use when the user asks to check, audit or review a plan step.
disable-model-invocation: true
argument-hint: "<step id, e.g. S08>"
allowed-tools: Read Grep Glob Bash(pnpm *) Bash(git status *) Bash(git diff *) Bash(git log *)
---

# Check step $ARGUMENTS

Audit step **$ARGUMENTS** of `docs/build/plan.md` against the docs it lists under **Read**. Be strict: the docs are the contract. **Don't edit any file.** Report only.

## Checklist

Go through each feature doc section the step lists and check the code.

1. **Scope (section 1):** everything under "Does" exists. Nothing under "Out of scope" was built here.
2. **User stories (section 2):** for each acceptance criterion, find the test(s) that prove it. Missing test → gap.
3. **Module design (section 3):**
   - the public interface signatures match exactly;
   - `src/public/index.ts` exports nothing else;
   - folders follow the layout;
   - imports follow the overview DAG (run `pnpm lint`).
4. **Data model (section 4):** every table, column, type, `CHECK`, index, RLS policy, partitioning rule and grant is in the migrations, and Drizzle `schema.ts` matches.
5. **API contracts (section 5):**
   - every route exists with the doc's method, path, permission or scope, request schema, success status and shape, and each listed error code;
   - no extra undocumented routes;
   - new error codes are in `libs/contracts/src/errors.ts`;
   - events are emitted with the documented payloads, and consumed with inbox idempotency.
6. **Business rules (section 6):** every state-table row, decision rule, number and limit is implemented and tested. Check the numbers literally.
7. **Edge cases (section 9):** each row has a test or an explicit handling path.
8. **Security (section 10):** authN and authZ per route; masking; audit entries written; no PII in logs, events or job payloads (grep for payer fields in logger calls).
9. **Performance (section 11):** indexes exist for the hot queries; budgets have a test or benchmark where the doc asks for one.
10. **Testing (section 12):** each listed test exists. Coverage gates are met (run `pnpm test -- --coverage` for the packages).
11. **Observability (section 13):** metric names, types and labels as listed; alert rules present in `infra/` if the step includes them.
12. **Configuration (section 14):** every variable is declared in the module's zod config with the documented default.
13. **Open questions (section 15):** defaults implemented; new questions logged in `docs/build/questions.md`.
14. **Mockups:** for steps with screens, compare the UI against `docs/mockups/` (layout, copy, colours, both locales).

## Run

- `pnpm verify` and report the result.
- `git status` and `git log --oneline -20` to see what changed in this step.

## Report format

Start with one line: **PASS**, **PASS WITH GAPS** or **FAIL**, plus the counts. Then a table:

| # | Doc ref | Expected (from doc) | Found | Status | Fix |
| --- | --- | --- | --- | --- | --- |

- **Status** is `ok`, `missing`, `differs` or `untested`.
- **Fix** is one concrete action, with the file path.

End with the top 5 fixes in order of risk. Don't make the fixes; the user decides.
