---
paths:
  - "libs/features/**/src/**/*.ts"
  - "libs/platform/**/*.ts"
  - "apps/api/**/*.ts"
  - "apps/portal-api/**/*.ts"
  - "apps/admin-api/**/*.ts"
  - "apps/worker/**/*.ts"
---

# Backend rules (NestJS feature modules and platform)

**Layers inside a feature** (overview section 4)

- `domain/` is pure TypeScript: no Nest, no I/O, no `Date.now()`. Pass `now` in. This is where the state machines, checks, decision tables and mappers live, and where coverage must reach 90%.
- `application/` orchestrates domain plus ports: services, event handlers (`@OnEvent`), queue processors (`@Processor`).
- `infrastructure/` holds Drizzle repositories, Redis stores and adapters. Repositories take a `DataScope` and add its `WHERE` clause.
- `http/` holds controllers only: validate with the zod schema from `libs/contracts`, call one application service, map to the response. No business logic.
- `public/index.ts` exports the Nest module, the interface(s) named in the doc and DTO types. Nothing else. Event payload schemas live in `libs/contracts/src/events/`.

**Transactions and events**

- Every state change runs in `Db.tx()`. It locks the row (`SELECT … FOR UPDATE`), checks `version`, writes, writes the timeline or audit entry, and calls `Outbox.publish(tx, …)`, all in that one transaction.
- Side effects that can't be rolled back (enqueue, Redis publish, email) happen after commit.
- Event handlers insert into `platform.event_inbox` in their own transaction first; a conflict means "already processed".
- Event payload schemas and shared enums come from `libs/contracts/src/events/`. Import them from there, never from the emitting feature, so consuming an event adds no package dependency.
- A synchronous call into another module that writes passes the caller's `tx` as the first argument. That is the only cross-module write.

**Queues**

- Job IDs are deterministic and come from overview 3.6 or the feature doc. Re-adding an existing job ID is a no-op by design.
- Job data is IDs only, validated by the job's zod schema when enqueued and when processed.
- Processors open a `system` principal context with the job's tenant and env, so RLS applies.
- Retry policy lives where the doc says. For bank fetches that is doc 05's scheduler, not BullMQ attempts.

**Errors and responses**

- Throw `AppError(code, httpStatus, …)` with a code from `libs/contracts/src/errors.ts`. Never invent a code inline; add it to the catalogue first and note which doc defines it.
- Domain outcomes (rejected, expired, …) are data, not exceptions.
- Responses follow overview 6.2:
  - snake_case JSON;
  - `livemode` on objects;
  - list shape `{ data, next_cursor, has_more }`;
  - `200` final, `202` pending, `201` other creates, `204` deletes.

**Security**

- Every route has `@RequirePermission`, `@RequireScope` or `@Public`.
- Mutating portal and admin routes check CSRF. Routes the doc marks "re-auth" check `reauth_at` within 5 minutes.
- Admin writes run in a transaction scoped to the target tenant. Cross-tenant access is read-only.
- Class P data (payer names and phones, account numbers, receipt files, raw bank responses) is encrypted with `Crypto.encrypt(tenantId, …)` at rest, masked on output, and never logged. Reveals need the permission plus a re-auth, and are audited.
- Outbound HTTP is limited to three paths:
  - banks, only through `BankGateway` and the egress nodes;
  - merchant webhooks, only through doc 08's `SafeHttpClient`;
  - the vision provider configured in doc 06 (none by default).

  Billing's house API client (doc 12, 6.9) calls our own `api` service inside the cluster. Nothing else makes network calls beyond the platform services.

**Config and observability**

- Feature env vars are declared in the feature's zod config schema with the defaults from the doc's Configuration section, and merged at boot. Invalid config exits with code 78.
- Metric names, types and labels come from the doc's Observability section. No tenant IDs in labels (except quota metrics).
- Log with the platform logger, which adds the context fields. Never log request bodies.
