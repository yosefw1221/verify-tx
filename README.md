# verify-tx (Proofline, working name)

Payment verification for Ethiopian banks and wallets: CBE, telebirr, Bank of Abyssinia, Dashen, Awash, CBE Birr and M-Pesa.

A merchant sends a transaction reference or a receipt image or PDF, plus the amount it expects. Proofline fetches the bank's own record, checks that the money reached the merchant's account, wasn't already used, was completed, and matches the amount and date. It answers `verified` or `rejected` with a reason. When a bank is down, the customer uploads their receipt, a reviewer decides, and the bank is re-checked for 72 hours.

**Status:** specification complete, no application code yet. The code is written step by step with Claude Code, following the docs in this repository.

## What's here

| Path | Contents |
| --- | --- |
| `docs/00-overview.md` | Architecture, module boundaries, shared platform interfaces, conventions, build order, accepted decisions (R1–R15), open questions |
| `docs/features/` | 12 implementation docs, one per module, each with 15 sections: scope, screens and acceptance criteria, module design, data model, API, rules, dependencies, task plan, edge cases, security, performance, tests, observability, configuration, open questions |
| `docs/mockups/` | The 13 designed screens as PNG and static HTML (source of truth for the UI) |
| `docs/reference/` | The original architecture research and UI spec, exported for reference |
| `docs/build/plan.md` | The build broken into steps S00–S17 plus Track B, with the prompt for each |
| `docs/build/progress.md` | Step status, Phase 0 checklist and step log |
| `docs/build/questions.md` | Questions raised while building, with defaults and answers |
| `CLAUDE.md` | Rules every Claude Code session follows in this repo |
| `.claude/rules/` | Extra rules that load only when Claude works on matching files (backend, database, frontend, tests, contracts) |
| `.claude/skills/` | `/build-step` and `/check-step` |
| `.claude/settings.json` | Project permission rules: common dev commands allowed; push, merge and resets ask first; `.env` files unreadable |

## Building with Claude Code

1. **Install Claude Code** (it needs a Pro, Max, Team, Enterprise or Console account). On macOS, Linux or WSL:

   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```

   On Windows PowerShell: `irm https://claude.ai/install.ps1 | iex`. Other options: [setup guide](https://code.claude.com/docs/en/setup).

2. **Install the project's tools:** Node.js 24 LTS, pnpm (`corepack enable`), Docker with Compose, and Git.

3. **Clone the repo and start Claude Code:**

   ```bash
   git clone https://github.com/yosefw1221/verify-tx.git
   cd verify-tx
   claude
   ```

4. **Run the first step:**

   ```text
   /build-step S00
   ```

   Claude reads the docs the step lists, builds it with tests, runs `pnpm verify`, updates `docs/build/progress.md` and commits on a `step/S00-…` branch. It doesn't push.

5. **Audit it in a fresh session** (`/clear`):

   ```text
   /check-step S00
   ```

   Ask Claude to fix any gaps it reports. Then review and merge the branch.

6. **Repeat for each step** in the order of `docs/build/plan.md`. Start each step with a cleared session. Steps that don't depend on each other (for example S07 next to S03–S06, or S12, S14 and S15) can run in parallel in separate git worktrees.

When Claude hits something the docs don't settle, it writes it to `docs/build/questions.md`. If the doc has a default, Claude uses it and keeps going; if not, it stops and asks you.

## What only people can do (Phase 0)

The things in `docs/build/progress.md` → Phase 0 can't be done by Claude Code:

- the legal opinion on using banks' receipt pages;
- accounts and test payments at the seven providers;
- in-country hosting and ISPs;
- capturing real bank responses from an in-country host (`pnpm fixtures:capture`, built in S07);
- the vision-provider decision;
- prices;
- an Amharic reviewer;
- the product name.

The real bank adapters (Track B) wait for those fixtures. Everything else runs on the sandbox connector, which never calls a bank.

## Design references

- UI mockups canvas: https://claude.ai/artifact/EYXDJVaKXKekjhuW3SPZdY
- Architecture and UI spec (live doc): https://claude.ai/code/artifact/b1798dd4-d875-4b2e-8dbd-8715e423c147
