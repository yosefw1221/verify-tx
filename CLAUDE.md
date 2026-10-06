# Proofline (working name): verify-tx

Multi-tenant SaaS that verifies Ethiopian bank and wallet payments (CBE, telebirr, BoA, Dashen, Awash, CBE Birr, M-Pesa). A merchant sends a reference or a receipt plus the amount it expects; we fetch the bank's record, run the checks, and answer `verified` or `rejected` with a reason. When the bank is down we collect the customer's receipt, a human reviews it, and we keep re-checking the bank for 72 h.

## Where the truth lives

Read before writing code. The docs are the spec; code follows them, not the other way round.

| What | Where |
| --- | --- |
| Architecture, conventions, platform interfaces, accepted decisions R1–R15 | `docs/00-overview.md` |
| One module per doc, 15 sections each (rules, data model, API, tests, config) | `docs/features/01-…` to `12-…` |
| Screens (source of truth for layout and copy) | `docs/mockups/*.png` and `*.html` |
| Design tokens, status vocabulary, surface specs | `docs/reference/ui-spec.md` (feature docs win on conflict) |
| Research and background | `docs/reference/architecture.md` (reference only) |
| Build steps, prompts, done criteria | `docs/build/plan.md` |
| What's done, what's next | `docs/build/progress.md` |
| Questions raised while building | `docs/build/questions.md` |

## Non-negotiable rules

1. **Implement what the docs say, exactly.** Names, types, status codes, error codes, limits, timings and event payloads come from the docs. Don't rename, round or "improve" them.
2. **Never guess a business rule.** If docs conflict, or something needed is unspecified, add an entry to `docs/build/questions.md`. If the doc gives a default, use it and say so; otherwise stop and ask. Never silently pick.
3. **Module boundaries** (overview section 4):
   - Import another feature only as `@proofline/<name>`; each package exports only `src/public/index.ts`.
   - Follow the DAG in overview section 3.2. A call against an arrow goes through an outbox event.
   - Each feature reads and writes only its own Postgres schema. No cross-schema joins. The only cross-schema FK is to `tenancy.tenants(id)`.
   - A cross-module write runs in the caller's transaction: the callee takes `tx: Tx` as its first argument.
   - No module calls another over HTTP. The only exception is billing's payment check through the public API (doc 12, 6.9).
   - `libs/platform` never imports from `libs/features`. Apps contain wiring only, no business logic.
4. **Tenancy and env isolation:**
   - Every query runs inside `Db.tx()`, which sets `app.tenant_id` and `app.env`. RLS is enabled and forced on every tenant table.
   - Repositories take a `DataScope` and filter by it. RLS is the second line of defence, not the first.
5. **Test env never calls a bank.** In `test`, `BankGateway` routes to the sandbox connector (doc 04, 6.8). No test, CI job or dev script makes a real bank request.
6. **Money is `bigint` santim.** Never a JS `number`. On the wire, amounts are decimal strings with 2 places. Use `@proofline/platform/money`.
7. **Time:**
   - Inject `Clock`; never call `new Date()` or `Date.now()` in domain or application code.
   - Store UTC `timestamptz`. Show `Africa/Addis_Ababa` (UTC+3) on a 24 h clock. Amharic shows Ethiopian dates through `@proofline/platform/i18n`.
8. **PII and secrets:**
   - No payer data, tokens, secrets or raw bank responses in logs, events, job payloads, metric labels or error messages.
   - Payer fields appear only after the receiver check passed (R3).
   - Secrets are never committed. Only `.env.example` lives in git, and its values are fake.
9. **Errors** come from the single catalogue in `libs/contracts/src/errors.ts` (overview 6.4). A rejected payment is a `200` with `status: rejected`, never an HTTP error.
10. **Every user-facing string goes through i18n keys, in both `en` and `am`.** The CI parity check must pass. Amharic drafts are marked for translator review (overview Q10).

## Stack (overview section 2)

- **Base:** TypeScript (strict), Node 24, pnpm and Turborepo.
- **Backend:** NestJS 11 on Fastify; PostgreSQL 18 with Drizzle and plain SQL migrations; BullMQ on Redis 7; zod contracts that generate OpenAPI 3.1.
- **Frontend:** Next.js 16, React 19, Tailwind 4 with Radix, TanStack Query, next-intl.
- **Services:** MinIO, OpenBao, Keycloak, ClamAV.
- **Testing:** Vitest, Testcontainers, Playwright.

Deployables: `api`, `portal-api`, `admin-api`, `worker`, `web-portal` (cashier PWA at `/cashier`), `web-upload`, `web-admin`.

## Repository layout

```text
apps/      api  portal-api  admin-api  worker  web-portal  web-upload  web-admin
libs/      platform/  contracts/  ui/  features/{tenancy,receiving-accounts,api-clients,bank-connectors,
           verification,receipts,review,webhooks,portal,cashier,admin,billing}
infra/     docker, helm, egress proxy configs, dashboards, alert rules
docs/      overview, features, mockups, reference, build
```

Every feature package has the same shape: `src/public` (the only export), `domain` (pure, no I/O), `application`, `infrastructure`, `http/{public-api,portal,admin}`, `db/{schema.ts,migrations}`, and `test/{unit,integration,fixtures}`.

## Commands

S00 creates these; keep their names stable.

| Command | Does |
| --- | --- |
| `pnpm install` | Install everything |
| `docker compose up -d` | Postgres, Redis, MinIO, OpenBao (dev), Keycloak, ClamAV, Mailpit |
| `pnpm db:migrate` / `pnpm db:reset` | Run feature migrations in DAG order / drop and recreate the dev database |
| `pnpm dev` | All apps in watch mode |
| `pnpm lint` / `pnpm typecheck` | ESLint (including the boundary rules) and dependency-cruiser / `tsc --noEmit` |
| `pnpm test` / `pnpm test:int` | Unit tests / integration tests (Testcontainers) |
| `pnpm e2e` | Playwright against Docker Compose with the sandbox connector |
| `pnpm verify` | lint, typecheck, test and test:int; must be green before any commit |
| `pnpm --filter @proofline/<pkg> test` | One package |

## How to work a step

Use `/build-step S0N` (skill in `.claude/skills/build-step`). By hand, the same loop is:

1. Read the step in `docs/build/plan.md`, then every doc section it lists, fully, and the listed mockups.
2. Check `docs/build/progress.md`: the steps it needs must be `done`.
3. Make a task list from the doc's Implementation plan tasks (`T05.3` and so on) named in the step.
4. For each task, write the tests from the doc's acceptance criteria and Testing section first, then implement, then run that package's tests.
5. Run `pnpm verify`. Fix until green. Coverage gates: 90% lines in `domain/`, 80% per package.
6. Tick every "Done when" item with evidence (test names or commands).
7. Update `docs/build/progress.md`. Commit on branch `step/S0N-<slug>` with `feat(<module>): <summary> [S0N]`.
8. Don't push, open PRs or touch other branches unless asked.

Then `/check-step S0N` audits the step against its docs before you call it done.

## Conventions (details in overview section 6)

| Thing | Convention |
| --- | --- |
| Files | kebab-case |
| Types | PascalCase |
| Functions and variables | camelCase |
| SQL | snake_case, plural tables, enums as `text` plus `CHECK` |
| JSON | snake_case; enum values lowercase |
| URL paths | kebab-case plural nouns, versioned (`/v1`, `/portal/v1`, `/admin/v1`, `/hosted/v1`) |
| IDs | ULIDs stored as `uuid`, prefixed at the edges (`ver_`, `rvw_`, …) through `@proofline/platform/ids` |
| Events | `<entity>.<past_tense>`, written to the outbox in the same transaction as the state change. Consumers are idempotent through the inbox |
| Queues | Deterministic job IDs (overview 3.6). Job data is IDs only, validated by zod on enqueue and on process |
| Routes | Every route declares `@RequirePermission`, `@RequireScope` or `@Public`; the boot check fails otherwise |
| Commits | Conventional commits, one step per branch |

## When something is unclear

- **Docs disagree:** the more specific doc wins over the overview for its own module, and the overview wins on shared conventions. Record the conflict in `docs/build/questions.md` either way.
- **Mockup versus doc:** the doc wins only where overview section 1 ("Corrections to the UI spec and mockups") or the doc's Open Questions say so. Otherwise match the mockup.
- **A doc's Open Questions** each carry a default. Build the default and don't ask again.
