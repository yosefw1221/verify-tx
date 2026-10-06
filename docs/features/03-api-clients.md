# 03 · API clients

| | |
| --- | --- |
| Package | `@proofline/api-clients` |
| Postgres schema | `api_clients` |
| Depends on | platform, 01 tenancy, 02 receiving accounts, 12 billing (plans, for rate limits) |
| Build step | 3 |
| Mockups | Portal · API clients: one-time secret panel, client table with Rotate and Revoke, quick-start request. Request logs and Sandbox are listed in the portal sidebar but not drawn. |

## 1. Purpose and scope

**Does**

- Creates, edits, rotates and revokes the API credentials merchants' systems use.
- Authenticates every public API request (`ApiClientAuthenticator`). That covers the key, the secret, scope, IP allowlist, tenant status, the live gate (R14) and the per-client rate limit.
- Records a masked log of every authenticated public API request, shown on the portal's Request logs screen and kept 30 days.

**Out of scope**

- What each endpoint does. That belongs to doc 04 (`/v1/banks`), doc 02 (`/v1/receiving-accounts`), doc 05 (`/v1/verifications*`) and doc 06 (`/v1/verifications/{id}/receipt`, `/upload-link`).
- Idempotency storage, which is in the platform (overview 5.16).
- Upload-link tokens (doc 06) and portal sessions (doc 01).

## 2. Screens and user stories

| Screen | Surface | Mockup |
| --- | --- | --- |
| API clients | Portal | Portal · API clients |
| Request logs | Portal | Not drawn; uses the portal DataTable and DetailDrawer |

**US-03.1 Create**

- A Developer or Owner opens "New API client" and enters a name (1–60 characters), the scopes, and optionally up to 20 IP addresses or CIDR ranges.
- The client is created in the session's current env. The full credential `pvk_{env}_{keyId}.{secret}` is shown once in the "Copy the secret now" panel, exactly as in the mockup. Closing the panel makes it unrecoverable; the API never returns the secret again.

**US-03.2 Authenticate**

- A request with a valid credential, an allowed IP and the needed scope reaches the endpoint with an `api_client` principal (tenant, env, scopes).
- A missing, malformed, unknown, wrong, revoked or expired credential gets `401 unauthenticated` with the same body and similar timing in every case.

**US-03.3 Live gate**

- A live key gets `403 live_not_enabled` until the tenant is `active` and has at least one verified receiving account. The portal's API clients screen shows the same rule as a notice.
- Test keys are never gated.

**US-03.4 IP allowlist**

- When the allowlist is non-empty, a request from any other IP gets `403 ip_not_allowed`. The response includes the IP we saw, so integrators can fix their list.

**US-03.5 Scope**

- A key without the scope an endpoint needs gets `403 forbidden_scope`, with `details.required_scope` set (section 6.3).

**US-03.6 Rotate**

- Rotate shows a new secret once. The old secret keeps working for 7 days, and the table shows "Rotating · old secret ends in N days", as in the mockup.
- Rotating again while an old secret is still active revokes that oldest secret immediately, so at most 2 secrets are valid at once. The confirm dialog says so.

**US-03.7 Revoke**

- Revoke (Owner or Developer, with re-auth) stops the key on the very next request across all API pods.

**US-03.8 Edit**

- The name, scopes and IP allowlist can change; the secret doesn't. Changes apply to the next request.

**US-03.9 Request logs**

- A Developer filters logs by API client, status class (2xx, 4xx, 5xx), method, path, date range and request ID, and opens one entry to see masked request and response bodies, duration, IP and idempotency key.
- Logs older than 30 days aren't shown and have been dropped.

## 3. Module design

```text
libs/features/api-clients/src/
├─ public/
│  ├─ index.ts                   ApiClientsModule, ApiClientAuthenticator, ApiClientsApi, ApiScope, events
│  ├─ authenticator.ts
│  └─ events.ts
├─ domain/
│  ├─ credential.ts              format, parse, generate (keyId 8 base36, secret 32 base62)
│  ├─ scopes.ts                  scope list and endpoint requirements (section 6.3)
│  ├─ cidr.ts                    parse and match IPv4 and IPv6 CIDR
│  └─ body-masker.ts             request-log masking rules (section 6.6)
├─ application/
│  ├─ clients.service.ts         create, update, rotate, revoke
│  ├─ authenticate.service.ts    ApiClientAuthenticator implementation
│  ├─ live-gate.service.ts       tenant active plus hasVerified, cached
│  └─ request-log.service.ts     buffer and batch insert, query
├─ infrastructure/
│  ├─ clients.repo.ts, secrets.repo.ts, request-logs.repo.ts
│  ├─ auth-cache.ts              Redis entries (section 6.4)
│  └─ cache-invalidation.handler.ts   @OnEvent tenant.status_changed, receiving_account.verified and .removed
├─ http/
│  ├─ public-api/api-key.guard.ts       Nest guard used by apps/api
│  ├─ public-api/request-log.interceptor.ts
│  └─ portal/api-clients.controller.ts, request-logs.controller.ts
└─ db/schema.ts, db/migrations/
```

```ts
export type ApiScope = 'verifications:read' | 'verifications:write' | 'receipts:write';

export interface ApiClientAuthenticator {
  /** Throws AppError 401/403/429 as in section 5. On success returns the principal to put in Ctx. */
  authenticate(req: { authorization: string | undefined; ip: string; method: string; routeId: RouteId }):
    Promise<Extract<Principal, { kind: 'api_client' }>>;
}
export interface ApiClientsApi {
  countActive(tenantId: string, env: EnvName): Promise<number>;           // admin tenant detail
  listForTenant(tenantId: string, env: EnvName): Promise<ApiClientView[]>; // admin, no secrets
  revokeByPlatform(clientId: string, a: { staffId: string; reason: string }): Promise<ApiClientView>;  // doc 11 Integrations tab; audited
}
export type RouteId =
  | 'verifications.create' | 'verifications.create_with_file' | 'verifications.get' | 'verifications.list'
  | 'verifications.cancel' | 'verifications.receipt' | 'verifications.upload_link'
  | 'banks.list' | 'receiving_accounts.list';
```

Every public controller method declares its `RouteId` with `@Route('verifications.create')`. The guard maps it to the required scope (6.3), so a new route can't skip a scope decision: the boot check fails if a public route has no `RouteId`.

## 4. Data model

```sql
CREATE TABLE api_clients.clients (
  id            uuid PRIMARY KEY,
  tenant_id     uuid NOT NULL REFERENCES tenancy.tenants(id),
  env           text NOT NULL CHECK (env IN ('live','test')),
  key_id        text NOT NULL UNIQUE CHECK (key_id ~ '^pvk_(live|test)_[0-9a-z]{8}$'),
  name          text NOT NULL CHECK (length(name) BETWEEN 1 AND 60),
  scopes        text[] NOT NULL CHECK (cardinality(scopes) >= 1
                  AND scopes <@ ARRAY['verifications:read','verifications:write','receipts:write']::text[]),
  ip_allowlist  cidr[] NOT NULL DEFAULT '{}' CHECK (cardinality(ip_allowlist) <= 20),
  created_by    uuid NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  last_used_at  timestamptz NULL,
  revoked_at    timestamptz NULL,
  revoked_by    uuid NULL,
  CHECK (substring(key_id from 5 for 4) = env)
);
CREATE INDEX clients_tenant_idx ON api_clients.clients (tenant_id, env) WHERE revoked_at IS NULL;

CREATE TABLE api_clients.secrets (
  id           uuid PRIMARY KEY,
  client_id    uuid NOT NULL REFERENCES api_clients.clients(id),
  tenant_id    uuid NOT NULL,           -- copied from the client, for RLS
  env          text NOT NULL,
  secret_hash  bytea NOT NULL,          -- HMAC-SHA256(API_KEY_PEPPER, secret)
  created_at   timestamptz NOT NULL DEFAULT now(),
  expires_at   timestamptz NULL,        -- set on rotation: created_at of the new secret + 7 days
  revoked_at   timestamptz NULL
);
CREATE INDEX secrets_client_active ON api_clients.secrets (client_id) WHERE revoked_at IS NULL;

CREATE TABLE api_clients.request_logs (
  id               uuid NOT NULL,
  started_at       timestamptz NOT NULL,
  tenant_id        uuid NOT NULL,
  env              text NOT NULL,
  client_id        uuid NOT NULL,
  request_id       text NOT NULL,
  method           text NOT NULL,
  route_id         text NOT NULL,
  path             text NOT NULL CHECK (length(path) <= 512),     -- with IDs, no query string
  query            jsonb NULL,                                    -- parsed and masked
  status           smallint NOT NULL,
  error_code       text NULL,
  duration_ms      integer NOT NULL,
  ip               inet NOT NULL,
  user_agent       text NULL CHECK (length(user_agent) <= 256),
  idempotency_key  text NULL,
  request_body     jsonb NULL,                                    -- masked, at most 8 KB
  response_body    jsonb NULL,                                    -- masked, at most 8 KB
  PRIMARY KEY (id, started_at)
) PARTITION BY RANGE (started_at);
-- Daily partitions request_logs_YYYYMMDD created 3 days ahead by the maintenance job; partitions older than 30 days dropped.
CREATE INDEX request_logs_tenant_time ON api_clients.request_logs (tenant_id, env, started_at DESC, id DESC);
CREATE INDEX request_logs_request_id ON api_clients.request_logs (request_id);

-- RLS: the standard tenant_id and env policy (overview 5.4) on clients, secrets and request_logs.
```

The authenticator must look up a key before any tenant context exists. It does that through a `SECURITY DEFINER` function that returns only what authentication needs:

```sql
CREATE FUNCTION api_clients.lookup_key(p_key_id text)
RETURNS TABLE (client_id uuid, tenant_id uuid, env text, scopes text[], ip_allowlist cidr[], revoked boolean,
               secret_hashes bytea[], secret_expiries timestamptz[])
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = api_clients AS $$
  SELECT c.id, c.tenant_id, c.env, c.scopes, c.ip_allowlist, c.revoked_at IS NOT NULL,
         array_agg(s.secret_hash ORDER BY s.created_at DESC) FILTER (WHERE s.revoked_at IS NULL),
         array_agg(s.expires_at  ORDER BY s.created_at DESC) FILTER (WHERE s.revoked_at IS NULL)
  FROM api_clients.clients c LEFT JOIN api_clients.secrets s ON s.client_id = c.id
  WHERE c.key_id = p_key_id GROUP BY c.id
$$;
GRANT EXECUTE ON FUNCTION api_clients.lookup_key TO pl_app;
```

**Migrations:** `0001_api_clients_init.sql` (tables, function, RLS) and `0002_request_logs_partitions.sql` (the parent table and the first 3 daily partitions).

## 5. API contracts

### Authentication on the public API

```
Authorization: Bearer pvk_live_k7q2mx9a.Zr3Q8vN1pL6tY0wE5hK2sB7dF4gJ9cMx
```

The credential is two parts joined by a single `.`:

- the key ID: `pvk_`, then `live` or `test`, `_`, then 8 characters from `[0-9a-z]`;
- the secret: 32 characters from `[0-9A-Za-z]`.

| Outcome | HTTP | `error.code` | Notes |
| --- | --- | --- | --- |
| Header missing, malformed, unknown key, wrong secret, revoked client, expired secret | 401 | `unauthenticated` | Identical body. An unknown key still computes one HMAC so the timing doesn't reveal it. `WWW-Authenticate: Bearer` |
| IP not on the allowlist | 403 | `ip_not_allowed` | `details.ip` = the address we saw |
| Scope missing | 403 | `forbidden_scope` | `details.required_scope` |
| Tenant `suspended` or `closed` | 403 | `tenant_suspended` | Any env |
| Live key, gate closed | 403 | `live_not_enabled` | `details.missing` = `["kyb_approval"]`, `["verified_receiving_account"]` or both |
| Over the rate limit | 429 | `rate_limited` | `Retry-After`; `RateLimit-*` headers on every response |

Checks run in this order: format → lookup → secret → revoked or expired → IP → tenant status → live gate → scope → rate limit. The first failure decides the response.

### Portal (`/portal/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /api-clients` | `api_clients.read` | none | `200 { data: ApiClient[] }` (current env) | none |
| `POST /api-clients` | `api_clients.manage` | `{ name, scopes[], ip_allowlist[]? }` | `201 { client: ApiClient, secret: "pvk_live_k7q2mx9a.Zr3Q…" }` | `400 invalid_field` (bad CIDR or unknown scope), `422 api_client_limit_reached` |
| `PATCH /api-clients/{id}` | `api_clients.manage` | `{ name?, scopes?, ip_allowlist? }` | `200 ApiClient` | `409 invalid_state` (revoked) |
| `POST /api-clients/{id}/rotate` | `api_clients.manage`, plus re-auth | none | `200 { client, secret, previous_secret_expires_at }` | `401 reauth_required`, `409 invalid_state` |
| `POST /api-clients/{id}/revoke` | `api_clients.manage`, plus re-auth | none | `200 ApiClient` | `401 reauth_required` |
| `GET /request-logs` | `request_logs.read` | `?client_id&status_class&method&route_id&request_id&started_gte&started_lt&cursor&limit` | `200 Page<RequestLogRow>` (no bodies) | `400 invalid_field` (range over 30 days) |
| `GET /request-logs/{id}` | `request_logs.read` | `?started_at` (needed for partition pruning; the UI passes it from the row) | `200 RequestLog` (with bodies) | `404` |

### Admin (`/admin/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `POST /tenants/{tenant_id}/api-clients/{id}/revoke` | `platform.tenants.manage`, plus re-auth | `{ reason }` (10–500 characters) | `200 ApiClient`; audited `api_client.revoked_by_platform`; the tenant's Owners are emailed | `401 reauth_required`; `404` |

```json
// ApiClient
{ "id": "apc_01J…", "key_id": "pvk_live_k7q2mx9a", "name": "POS backend", "env": "live",
  "scopes": ["verifications:write", "verifications:read"], "ip_allowlist": ["196.188.12.40/32"],
  "created_by": { "id": "usr_01J…", "name": "Dawit Haile" }, "created_at": "2026-10-04T10:51:00+03:00",
  "last_used_at": null, "revoked_at": null,
  "rotation": { "previous_secret_expires_at": "2026-10-10T09:00:00+03:00" } }
```

`rotation` is `null` when only one secret is active.

### New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `api_client_limit_reached` | 422 | 20 active clients per tenant per env |

### Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `api_client.created` | `{ client_id, key_id, scopes }` | none (audit only) |
| `api_client.updated` | `{ client_id, changed: ["scopes", …] }` | none (audit only) |
| `api_client.rotated` | `{ client_id, previous_expires_at }` | none |
| `api_client.revoked` | `{ client_id }` | none |

**Consumes:** `tenant.status_changed`, `receiving_account.verified`, `receiving_account.removed`. Each deletes `pl:apikey:gate:{tenantId}:{env}`. Also `subscription.changed` (doc 12), which drops the cached plan so a new rate limit applies at once instead of within 60 s.

## 6. Business rules and state machine

### 6.1 Client and secret lifecycle

| Object | States | Transitions |
| --- | --- | --- |
| Client | active → revoked | Revoke (irreversible). Creating is always in the session's current env. |
| Secret | active → (expiring) → expired; any → revoked | Rotate: the new secret is active, and every currently active secret without `expires_at` gets `expires_at = now + 7 days`. If an active secret already has `expires_at` (a rotation is pending), it is revoked immediately. Revoking the client revokes all its secrets. |

A secret is valid when `revoked_at IS NULL AND (expires_at IS NULL OR expires_at > now())`.

### 6.2 Credential generation

- `keyId`: 8 characters from `[0-9a-z]` drawn with `crypto.randomInt`, retried on a unique-constraint collision (at most 5 tries, then `500`).
- `secret`: 32 characters from `[0-9A-Za-z]` drawn with `crypto.randomInt` (about 190 bits).
- **Stored:** `HMAC-SHA256(API_KEY_PEPPER, secret)`. The pepper is 32 bytes from OpenBao `kv/api-key-pepper`, loaded at boot. Rotating the pepper requires re-issuing all keys, so it is a documented emergency procedure, not a routine one.
- **Compared** with `crypto.timingSafeEqual` against every active secret hash of the client.

### 6.3 Scopes per route

| `RouteId` | Endpoint | Required scope |
| --- | --- | --- |
| `verifications.create` | `POST /v1/verifications` (JSON) | `verifications:write` |
| `verifications.create_with_file` | `POST /v1/verifications` (multipart) | `verifications:write` **and** `receipts:write` |
| `verifications.get` | `GET /v1/verifications/{id}` | `verifications:read` |
| `verifications.list` | `GET /v1/verifications` | `verifications:read` |
| `verifications.cancel` | `POST /v1/verifications/{id}/cancel` | `verifications:write` |
| `verifications.receipt` | `POST /v1/verifications/{id}/receipt` | `receipts:write` |
| `verifications.upload_link` | `POST /v1/verifications/{id}/upload-link` | `verifications:write` |
| `banks.list` | `GET /v1/banks` | none (any valid key, R13) |
| `receiving_accounts.list` | `GET /v1/receiving-accounts` | none (any valid key) |

### 6.4 Caching

| Key | Value | TTL | Invalidated by |
| --- | --- | --- | --- |
| `pl:apikey:{keyId}` | Result of `lookup_key` | 300 s | Create, update, rotate, revoke: the key is deleted in the same request, after commit |
| `pl:apikey:gate:{tenantId}:{env}` | `{ status, hasVerified }` | 60 s | The events listed in section 5 |
| `pl:apikey:used:{clientId}` | `1` | 60 s (`SET NX`) | none; it throttles `last_used_at` updates to once per minute |

The cache is Redis only, with no in-process copy, so revocation is visible to every pod on the next request. If Redis is down, the authenticator calls `lookup_key` directly (adding about 3 ms) and logs a warning.

### 6.5 Rate limits

| Plan (doc 12) | Requests per minute per API client | Burst |
| --- | --- | --- |
| Sandbox | 10 | 10 |
| Starter | 60 | 30 |
| Growth | 300 | 100 |
| Enterprise | `plans.rate_limit_per_min` (contract) | 1/3 of the limit |

- The limit is a token bucket in Redis, keyed `pl:rl:apc:{clientId}`.
- The plan comes from `PlansApi.getEffectivePlan(tenantId)` (doc 12), cached 60 s.
- Test keys always use the Sandbox limit, whatever the tenant's plan.

### 6.6 Request-log masking

| Field (request or response) | Stored as |
| --- | --- |
| `payer_phone`, `transaction.payer.phone_masked` | Masked (`+251 9•• ••• 330`) |
| `transaction.payer.name` | First name plus last initial (`SELAMAWIT G.`) |
| `share_link`, or a `reference` that is a URL | Host plus first 6 characters of the token, then `…` |
| `next_action.upload_url` | Origin plus `/r/…` (the token is cut) |
| Multipart file parts | `{ "file": { "content_type", "size_bytes" } }` (no bytes, no filename) |
| `metadata` values | Kept, truncated to 100 characters each |
| Bodies over 8 KB after masking | `{ "_truncated": true, "_bytes": 12345 }` |
| `Authorization`, cookies | Never stored |

### 6.7 Request-log write path

- An interceptor on every public route builds the entry after the response is sent, so logging never delays the client.
- Entries go to a bounded in-process buffer of 5,000 entries. A multi-row insert flushes every 1 s or at 500 entries, whichever comes first, and the buffer is flushed on `SIGTERM` (graceful shutdown waits at most 5 s).
- When the buffer is full, the oldest entry is dropped and `pl_request_logs_dropped_total` is incremented.
- If a pod crashes, its unflushed entries are lost (at most 1 s of traffic). This is accepted and documented in the Request logs screen help text.
- Unauthenticated requests (401) aren't attributable to a tenant. They go to operational logs only.

### 6.8 Limits

| Rule | Value | Where |
| --- | --- | --- |
| Active clients per tenant per env | 20 | `API_CLIENT_MAX_PER_TENANT` |
| IP allowlist entries | 20 | Constant |
| Secret overlap on rotation | 7 days (from the brief) | `API_CLIENT_ROTATION_OVERLAP_DAYS` |
| Request-log retention | 30 days | `REQUEST_LOG_RETENTION_DAYS` (overview Q13) |
| Request-log query range | At most 30 days per query | Constant |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `RateLimit`, `Ctx`, `AuditLog`, `Outbox`, `OnEvent`, `ConfigService` (pepper), `Clock` |
| 01 tenancy | `TenancyApi.getTenant()` (status) |
| 02 receiving accounts | `ReceivingAccountsApi.hasVerified()` |
| 12 billing | `PlansApi.getEffectivePlan()` (rate limit). This adds the edge `api-clients → billing` to the overview DAG; billing depends only on tenancy, so there is no cycle. |

## 8. Implementation plan

1. **T03.1** Migrations: tables, partitions, `lookup_key` function, RLS. Tests: RLS isolation, and `lookup_key` works without a tenant context.
2. **T03.2** `credential.ts`: generate and parse, rejecting malformed input such as two dots, a wrong prefix or a wrong length.
3. **T03.3** `cidr.ts`: IPv4, IPv6 and IPv4-mapped IPv6 matching, with tests.
4. **T03.4** Client service: create (returns the secret once), update, rotate (including the second-rotation rule), revoke, limits, audit.
5. **T03.5** Authenticator: the ordered checks of section 5, Redis caching, the timing-equalising dummy HMAC, and client-IP resolution through `TRUSTED_PROXY_CIDRS`.
6. **T03.6** `ApiKeyGuard` and the `@Route` decorator in `apps/api`, with the boot check that every public route has a `RouteId`.
7. **T03.7** Rate limiting from the plan.
8. **T03.8** Request-log interceptor, masker, buffer and flusher, plus the partition maintenance job (create 3 days ahead, drop older than 30 days).
9. **T03.9** Portal endpoints and screens: API clients (mockup) and Request logs (table, filters, drawer), EN/AM.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Revoke while a request using the key is mid-flight | That request was already authenticated and completes. Every later request fails with `401`. |
| Rotate twice quickly | The second rotation revokes the oldest secret immediately, so at most 2 are valid. The UI confirm dialog states it. |
| Clock skew between pods around a secret's `expires_at` | All pods compare against the database server's time, because `lookup_key` returns expiries and the guard compares with `Clock.now()`, which is synced by NTP. A skew of under 1 s is accepted. |
| Customer behind a NAT changing IPs | Their requests get `403 ip_not_allowed` with `details.ip`. They update the list in the portal, effective on the next request. |
| `X-Forwarded-For` spoofed by the client | Only hops added by `TRUSTED_PROXY_CIDRS` are trusted. The client IP is the rightmost untrusted address. |
| Redis down | Auth falls back to the database. Rate limiting fails **open**, logged, with the metric `pl_ratelimit_bypass_total`, so a Redis incident doesn't take the API down. |
| Database down | Auth fails with `503 service_unavailable`. |
| Request-log buffer overflow during a traffic spike | The oldest entries are dropped and counted. API latency is unaffected. |
| Tenant suspended | Within 60 s at most (gate cache), immediately if the event handler ran: `403 tenant_suspended`. |
| A key leaks publicly | The tenant revokes it (immediate). Platform Admins can also revoke any key from the admin tenant detail screen (doc 11), using the same service and audited. |

## 10. Security and privacy

| Endpoint | AuthN | AuthZ |
| --- | --- | --- |
| Public `/v1/*` | API key | Scopes per 6.3; tenant and env from the key; RLS |
| Portal API clients | Session | `api_clients.read` or `.manage`; rotate and revoke need re-auth |
| Portal request logs | Session | `request_logs.read`; current env only |

- Secrets are class S: shown once and never stored in plaintext. They are never logged (the pino redaction list covers `authorization` and `secret`).
- Request logs hold class P fields only in masked form (6.6).
- Brute-forcing secrets is infeasible (about 190 bits). Key-ID enumeration is pointless without the secret, and an unknown key ID costs the same time as a wrong secret.
- **Audited actions:** create, update, rotate, revoke (actor, client, changed fields).

## 11. Performance

- `authenticate()` runs on every public request. Targets: p95 under 2 ms on a cache hit and under 8 ms on a miss. Peak is about 300 requests per second across `api` pods.
- Request logs at 300 requests per second come to about 26 million rows a day at the absolute peak. Batched inserts of 500 rows keep that to about 1 insert per second per pod. Daily partitions make retention a `DROP TABLE`.
- Log queries always include `tenant_id`, `env` and a `started_at` range of at most 30 days, served by `request_logs_tenant_time` with cursor pagination.

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Credential parse and generate; CIDR matching; masker table (6.6); check order (a table of inputs to the expected first failure) |
| Integration | Create → authenticate; rotate → both secrets work, then after 7 days (`FakeClock`) only the new one; second rotation revokes the oldest; revoke is immediate across two app instances sharing Redis; live gate flips after `receiving_account.verified`; rate-limit 429 with `Retry-After` |
| Timing | Statistical test: unknown key versus wrong secret have mean latency within 1 ms over 2,000 runs |
| End to end | Portal create shows the secret once, and a reload doesn't show it; the quick-start curl from the mockup works against test env |
| Amharic | API clients and Request logs screens in `am` |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_api_auth_total` | counter | `outcome` (ok, unauthenticated, ip_not_allowed, forbidden_scope, live_not_enabled, tenant_suspended, rate_limited), `env` |
| `pl_api_auth_duration_seconds` | histogram | `cache` |
| `pl_request_logs_dropped_total` | counter | none |
| `pl_request_logs_flush_duration_seconds` | histogram | none |
| `pl_ratelimit_bypass_total` | counter | none |

| Alert | Condition | Severity |
| --- | --- | --- |
| Auth failures spike | `unauthenticated` above 200 per minute for 5 min | Warning |
| Request logs dropping | Any increase over 10 min | Warning |
| Partition missing | Tomorrow's request-log partition absent at 06:00 EAT | Critical (inserts would fail) |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `API_KEY_PEPPER` | from OpenBao `kv/api-key-pepper` | HMAC pepper (32 bytes, base64) |
| `API_CLIENT_MAX_PER_TENANT` | 20 | Active clients per tenant per env |
| `API_CLIENT_ROTATION_OVERLAP_DAYS` | 7 | Old secret validity after rotation |
| `REQUEST_LOG_RETENTION_DAYS` | 30 | Partition drop age |
| `REQUEST_LOG_BUFFER_MAX` | 5000 | In-process buffer size |
| `TRUSTED_PROXY_CIDRS` | (required) | Ingress and load-balancer ranges whose `X-Forwarded-For` is trusted |

There are no per-tenant settings and no feature flags.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 03-Q1 | Request-log retention (overview Q13) | 30 days |
| 03-Q2 | Should the Enterprise plan offer OAuth2 client credentials, as in the architecture doc? | Not at launch; API keys only |
