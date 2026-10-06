---
paths:
  - "libs/**/db/**"
  - "**/migrations/**/*.sql"
  - "libs/platform/db/**"
---

# Database and migration rules

**Where things live**

- One Postgres schema per feature, named as in overview 3.2. Migrations are plain SQL in `libs/features/<pkg>/db/migrations/NNNN_<feature>_<what>.sql`, numbered per feature.
- The migration job runs features in DAG order (`platform`, `audit`, `tenancy`, `receiving_accounts`, `billing`, `api_clients`, `bank_connectors`, `receipts`, `verification`, `review`, `webhooks`, `portal`, `cashier`, `admin`) as `pl_migrator`.
- The Drizzle `schema.ts` must match the SQL exactly. A test compares `drizzle-kit` introspection with `schema.ts` in CI.

**Shape of every tenant table**

- Env-scoped tables have `tenant_id uuid NOT NULL` and `env text NOT NULL CHECK (env IN ('live','test'))`. The env-less tables listed in overview 5.4 have `tenant_id` only.
- Each one has `ENABLE ROW LEVEL SECURITY` and `FORCE ROW LEVEL SECURITY`, plus the `tenant_isolation` policy from overview 5.4 (`USING` and `WITH CHECK`). An integration test proves a second tenant and the other env see nothing.
- The only cross-schema foreign key allowed is `REFERENCES tenancy.tenants(id)`. Any other cross-feature reference is a plain `uuid` column.

**Types**

- IDs are `uuid` (ULIDs). Partitioned tables whose ID time equals `created_at` use `PRIMARY KEY (id, created_at)`.
- Money is `*_minor bigint NOT NULL CHECK (… >= 0)` plus `currency char(3) … CHECK (currency = 'ETB')` where the doc has a currency.
- Enums are `text` plus a `CHECK (col IN (...))` with exactly the values in the doc. No Postgres enum types.
- Timestamps are `timestamptz`, named `*_at`.
- Class P data columns end in `_enc` (ciphertext `v1:<keyVersion>:<b64>`), and equality lookups use a `*_hmac bytea` column beside them.

**Changes**

- Expand and contract only (overview 5.13): add nullable columns, tables and indexes. Drops and renames ship one release after code stops using them.
- Indexes on existing large tables use `CREATE INDEX CONCURRENTLY`, in their own migration file with no transaction.
- Partitioned tables register with the maintenance job's partition registry (period, look-ahead, retention) in the same step that creates them.
- `SECURITY DEFINER` functions only where the doc defines one (for example `verification.claim_owner`). They set `search_path`, return the minimum columns, and are granted explicitly to the role the doc names. That is usually `pl_app`; doc 02's `verified_owner()` is `pl_admin` only, and a test proves `pl_app` can't call it.

**Roles**

- `pl_app`: DML under RLS.
- `pl_admin`: adds cross-tenant `SELECT` when `app.principal_kind = 'staff'`.
- `pl_migrator`: DDL.
- No role has `BYPASSRLS`. `audit.events` grants `INSERT` and `SELECT` only.
