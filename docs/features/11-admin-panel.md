# 11 · Admin panel

| | |
| --- | --- |
| Package | `@proofline/admin` (BFF) and `apps/web-admin` (Next.js) |
| Postgres schema | `admin` |
| Depends on | platform (including the new `AuditReader`, `ConfigRegistry.describe` and `FeatureFlagsAdmin`), 01 tenancy, 02 receiving accounts, 03 API clients, 04 bank connectors, 05 verification, 06 receipts, 07 review, 08 webhooks, 12 billing |
| Build step | 8 (in parallel with 09 and 10) |
| Mockups | Admin · Dashboard; the admin shell in Admin · Review queue and Admin · Review workspace (those two pages are doc 07's) |

## 1. Purpose and scope

**Does**

- **The admin shell:**
  - dark sidebar with badges, shift and presence line;
  - top bar with global search, deployment label and live EAT clock, user menu;
  - light and dark themes (dark is admin-only, UI spec);
  - error, empty and loading states.
- **Dashboard:** volume, bank-confirmed share, review backlog, SLA breaches, rejections, per-bank health, hourly volume, outcomes, backlog by time left, and the alerts feed.
- **Verifications explorer:** cross-tenant search with masked payer data. The detail view combines attempts, timeline, receipt, webhooks and the review task, and offers Retry bank and break-glass evidence access.
- **Tenant detail frame:** the tabs, plus the Overview, Integrations and Activity tabs.
- **Banks & egress screens** on doc 04's endpoints.
- **Audit log:** API and screen, with the chain-verified badge and CSV export.
- **Settings:** feature flags, review shifts, and a read-only view of the effective configuration.
- The alerts feed read model, and staff presence.

**Out of scope.** These pages sit in the admin shell but are built by their owners:

| Page | Owner |
| --- | --- |
| Review queue and workspace, QA | 07 |
| Tenants list; Profile & KYB tab; Staff & roles; admin sign-in | 01 (T01.17) |
| Receiving-account approvals queue and tab | 02 (T02.10) |
| Billing section; Plan & usage tab | 12 |

## 2. Screens and user stories

| Screen | Owner | Mockup |
| --- | --- | --- |
| Shell (sidebar, top bar, shift and presence, search) | 11 | Every admin mockup |
| Dashboard | 11 | Admin · Dashboard |
| Review queue, Review workspace | 07 | Admin · Review queue, Admin · Review workspace |
| Verifications explorer and detail | 11 | Not drawn; UI spec "Verifications" row; portal table and drawer patterns at admin density |
| Tenants list | 01 | Not drawn |
| Tenant detail frame; Overview, Integrations, Activity tabs | 11 | Not drawn |
| Tenant tabs: Profile & KYB (01), Receiving accounts (02), Plan & usage (12), Settings (01, read-only for staff) | Owners | Not drawn |
| Banks & egress: list, bank detail, egress nodes | 11 (endpoints 04) | Not drawn; the dashboard strip's style |
| Staff & roles | 01 | Not drawn |
| Audit log | 11 | Not drawn |
| Billing | 12 | Not drawn |
| Settings | 11 | Not drawn |

**US-11.1 Access and shell**

- The admin panel is reachable only from the allowlisted networks (ingress, plus `ADMIN_ALLOWED_CIDRS` checked again in admin-api). Sign-in is Keycloak OIDC with mandatory WebAuthn or TOTP, and a 15-minute idle timeout (doc 01).
- **Sidebar:** Dashboard, Review queue, Verifications, Tenants, Banks & egress, Staff & roles, Audit log, Billing, Settings, filtered by permission (6.1).
  - The Review queue badge shows all open platform-stage tasks (amber).
  - Banks & egress shows the number of banks not `operational` (red).
  - Tenants shows submitted KYB files plus accounts awaiting approval (amber).
- **Sidebar footer:** "Day shift · 06:00–14:00" and "5 reviewers online · 1 Review Lead" (6.5).
- **Top bar:** Search (the `/` key focuses it), "Production · 10:42 EAT" (deployment label and a ticking EAT clock with UTC in the tooltip), and the user menu with name, role, theme (Light, Dark, System) and sign out.
- The EN / አማ switch appears only when `ADMIN_LOCALES` includes `am` (overview Q12; default English only).

**US-11.2 Dashboard KPIs**

- Range tabs: Today (default), 7 days, 30 days. All live env, and days are EAT days. The four cards:
  - Verifications: count created in range, plus "98.2% of verified payments confirmed by the bank";
  - Review backlog: open platform-stage tasks, plus "2 due within 10 min · Open queue" (always "now", whatever the range);
  - SLA breaches: count in range, plus "Both escalated to a Review Lead" or "1 of 2 escalated";
  - Rejected: count, plus "Top reason: Receipt already used (271)".
- The subtitle reads "Sunday 4 October 2026 · live, updated 10:42 EAT". Data refreshes every 30 s.

**US-11.3 Bank health strip**

- One tile per bank: name; state (Healthy, Degraded, Down); "Answered" over the last hour (the bank replied, found or not found); p95 latency; and a note (6.3).
- The legend reads "Answered = the bank replied (found or not found) in the last hour · canaries every 2 min from 2 egress nodes". A tile opens that bank's detail page.

**US-11.4 Volume chart**

- For Today: 24 bars, one per EAT hour up to now. Completed hours are teal; the current hour is pale and labelled "Hour in progress". The axis caption reads "Hour of day, East Africa Time".
- For 7 and 30 days: one bar per EAT day. Hovering a bar shows the count.

**US-11.5 Outcomes**

- For verifications created in range, a stacked bar and a list: Verified · bank confirmed, Verified · manual review, Rejected, Reversed (shown only when above 0), Expired or cancelled, Still open, each with count and percentage.
- Under it: "Still open: 118 receipt needed · 37 in review · 11 processing".

**US-11.6 Review backlog by time left**

- Three buckets (Under 10 min, 10 to 30 min, Over 30 min) by time left to `due_at` for open platform-stage tasks, plus "Open queue".
- Then "Oldest: RV-20402, waiting 52 min, Sheger Academy on a 4 h Starter SLA", where the task number links to the workspace.

**US-11.7 Recent alerts**

- The newest 8 alerts from the feed (6.4), each with time, coloured dot, title and detail, as in the mockup. "All alerts" opens the full list with filters by kind and bank.
- A Review Lead or Admin can acknowledge an alert; it then shows the person's name and time.

**US-11.8 Global search**

- The search box accepts a verification ID, a task number (`RV-20418`), a tenant ID, a reference, an order ID, or a tenant name (at least 3 characters).
- Results are grouped by type (Verifications, Review tasks, Tenants), at most 10 per group. An exact ID match opens directly.

**US-11.9 Verifications explorer**

- Filters: tenant, status, bank, channel, assurance, review reason, created range (default today, at most 31 days). Free text uses the search rules.
- Columns: Time, Tenant, Status and note, Amount, Bank, Reference, Payer (masked, 6.7), Order ID. Rows are 40 px, 50 per page, cursor paging.
- Test-env verifications are hidden unless the "Include test" toggle is on (Admins only).

**US-11.10 Verification detail**

- A drawer at admin density with these parts:
  - header: ID, tenant (link), status, reason, assurance, channel;
  - expected values;
  - the normalised transaction with payer fields masked;
  - checks;
  - attempts: number, trigger, node, outcome or failure class, latency, parser version, and an evidence link when one exists;
  - timeline (doc 05) merged with review actions and webhook delivery lines;
  - receipt thumbnail;
  - review task link;
  - webhooks for this verification (metadata only).
- **Actions:**
  - "Retry bank" (`platform.verifications.retry_bank`): asks for confirmation, returns `202`, is audited, and the drawer polls.
  - "View bank response" (`platform.evidence.break_glass`): asks for a reason of 10–500 characters, opens a 60 s URL, and is audited.
  - "Reveal payer details" (`platform.pii.reveal` plus re-auth): audited.
- Viewing a receipt image is audited as `receipt.viewed` (doc 06).

**US-11.11 Tenant detail**

- **Header:** display name, legal name, status chip (Pending KYB, Active, Suspended, Closed), plan, created date, and the actions their owners provide (Approve KYB, Suspend; doc 01).
- **Tabs:**
  - Overview (this doc): 30-day volume sparkline, rejection rate, top 3 reasons, manual-review share, open review tasks, live API clients, webhook health, usage this period;
  - Profile & KYB (01);
  - Receiving accounts (02);
  - Plan & usage (12);
  - Settings (01: the tenant's policy values, read-only to staff);
  - Integrations (this doc: API clients from doc 03, without secrets, with "Revoke key" for `platform.tenants.manage` after a reason and re-auth, through doc 03's admin route; webhook endpoints from doc 08, metadata only);
  - Activity (this doc: audit events for this tenant, newest first).

**US-11.12 Banks & egress**

- **List:** the 7 banks with state, answered ratio and p95 over 1 h and 24 h, breaker state and since, adapter enabled, parser version, last canary per node.
- **Bank detail:**
  - failure-class counts (1 h, 24 h);
  - the 20 latest breaker events with causes;
  - canaries (add, run now, delete);
  - "Force breaker open" or "Force closed" for 1–1440 minutes, or "Clear force";
  - "Disable adapter" or "Enable adapter".
  - Every control needs `platform.banks.manage`, a reason (10–500 characters) and a confirmation dialog naming the effect ("New M-Pesa checks will ask customers for a receipt").
- **Egress nodes table:** ISP, drained flag, per-bank score, last error, and "Drain" or "Undrain" with a reason (`409 last_node` blocks draining the last one).
- Each change appears on the dashboard within 30 s (doc 04).

**US-11.13 Audit log**

- `platform.audit.read`: a filterable stream (actor, action, target type and ID, tenant, time range) with 50 rows per page.
- A "Chain verified · last checked 03:00" badge shows the nightly result. On a break it turns red, "Chain break at seq 182334", and the alert fires.
- Rows open a drawer with the full event, including metadata and request ID.
- "Export CSV" streams the filtered rows (at most 31 days, at most 100,000 rows), and is itself audited.

**US-11.14 Settings**

- **Feature flags** (`platform.settings.manage` plus re-auth): key, description, default, per-tenant rules (first match wins, overview 5.1), last changed by and when. Changes are audited and take effect within 30 s.
- **Review shifts:** a list of named EAT time ranges covering 24 h without overlap (default Day 06:00–14:00, Evening 14:00–22:00, Night 22:00–06:00), used for the sidebar line and reviewer stats.
- **Effective configuration:** a read-only table per module of every variable with its value and source (`env` or `default`). Secrets show as `set` or `not set`, never their value. This is where an operator checks SLA targets, thresholds, retention and the extraction provider. Changing them is a deploy, not a click (open question 11-Q2).

**US-11.15 Dark mode**

- The theme follows the user's choice (stored per staff member) or the system. Every admin screen, including the review workspace, works in both themes with the contrast in 6.8.

## 3. Module design

```text
libs/features/admin/src/
├─ public/index.ts                    AdminModule (BFF only)
├─ domain/
│  ├─ nav.ts                          permissions → items and badges (6.1, pure)
│  ├─ bank-note.ts                    BankHealthView → tile note (6.3, pure)
│  ├─ alert-mapper.ts                 event → alert row (6.4, pure)
│  ├─ shift.ts                        current shift from the list and an instant (pure)
│  ├─ search-parse.ts                 classify the query (6.6, pure)
│  └─ mask.ts                         payer name and phone masks for staff views (6.7, pure)
├─ application/
│  ├─ bootstrap.service.ts, badges.service.ts
│  ├─ dashboard.service.ts            fan-out with a 30 s cache per range
│  ├─ alerts.handler.ts               @OnEvent bank.*, review.sla_breached
│  ├─ presence.middleware.ts          marks staff online on each admin-api request
│  ├─ search.service.ts
│  ├─ explorer.service.ts             verification detail composition
│  ├─ tenant-tabs.service.ts          Overview, Integrations, Activity
│  ├─ audit.service.ts                list, chain status, CSV stream (through AuditReader)
│  └─ settings.service.ts             flags, shifts, config description
├─ infrastructure/ alerts.repo.ts, shifts.repo.ts, presence.store.ts (Redis)
├─ http/admin/*.controller.ts
└─ db/schema.ts, db/migrations/

apps/web-admin/
├─ app/(admin)/layout.tsx             Shell; loads /admin/v1/bootstrap
├─ app/(admin)/dashboard/page.tsx
├─ app/(admin)/verifications/page.tsx       drawer through ?v=
├─ app/(admin)/tenants/[id]/layout.tsx      tab frame; tabs from owners
├─ app/(admin)/banks/page.tsx, banks/[code]/page.tsx, banks/egress/page.tsx
├─ app/(admin)/audit/page.tsx
├─ app/(admin)/settings/page.tsx
├─ app/(admin)/{review,tenants,staff,billing}/…    pages owned by 07, 01, 12
└─ src/features/{shell,dashboard,explorer,tenant,banks,audit,settings}/
```

The stack is the portal's (doc 09, section 3): Next.js 16, React 19, Tailwind 4 with `libs/ui`, TanStack Query, next-intl. Charts use a small SVG bar component in `libs/ui`, with no chart library, to keep the bundle small and the colours on tokens.

```ts
// New platform interfaces this doc needs (overview section 5 gains them)
export interface AuditReader {                                   // @proofline/platform/audit
  list(f: AuditFilter, page: { cursor?: string; limit: number }): Promise<Page<AuditEventView>>;
  chainStatus(): Promise<{ checkedAt: Date | null; ok: boolean; brokenAtSeq: number | null }>;
  stream(f: AuditFilter): AsyncIterable<AuditEventView>;         // CSV export
}
export interface ConfigRegistry {                                // @proofline/platform/config
  describe(): { module: string; key: string; value: string | null; secret: boolean; source: 'env' | 'default' }[];
}
export interface FeatureFlagsAdmin {                             // @proofline/platform/config
  list(): Promise<FlagView[]>;
  set(key: FlagKey, v: { defaultEnabled: boolean; rules: FlagRule[] }, actor: Actor): Promise<FlagView>;   // audited
}
```

## 4. Data model

```sql
CREATE TABLE admin.alerts (
  id               uuid PRIMARY KEY,
  at               timestamptz NOT NULL,
  kind             text NOT NULL CHECK (kind IN ('breaker_opened','breaker_half_open','breaker_closed','breaker_forced',
                     'format_changed','adapter_toggled','node_blocked','parser_deployed','sla_breach')),
  severity         text NOT NULL CHECK (severity IN ('critical','warning','info','ok')),
  bank_code        text NULL,
  node_id          text NULL,
  task_id          uuid NULL,
  tenant_id        uuid NULL,
  params           jsonb NOT NULL DEFAULT '{}'::jsonb,    -- template parameters only; no payer data
  source_event_id  uuid NOT NULL UNIQUE,
  acknowledged_by  uuid NULL,
  acknowledged_at  timestamptz NULL
);
CREATE INDEX alerts_time ON admin.alerts (at DESC);

CREATE TABLE admin.review_shifts (
  id          uuid PRIMARY KEY,
  name        text NOT NULL CHECK (length(name) BETWEEN 1 AND 30),
  starts_at   time NOT NULL,                   -- EAT wall clock
  ends_at     time NOT NULL,                   -- may be earlier than starts_at (crosses midnight)
  position    smallint NOT NULL,
  updated_by  uuid NULL,
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE admin.staff_prefs (
  staff_id    uuid PRIMARY KEY,
  theme       text NOT NULL DEFAULT 'system' CHECK (theme IN ('light','dark','system')),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
```

- These are platform tables with no tenant data, so they have no RLS. Only `pl_admin` (admin-api) has privileges on them; the worker writes `alerts` through the same role with a narrow grant (`INSERT`, `SELECT`).
- **Retention:** alerts are kept 90 days, deleted by the `maintenance` job.
- **Seed:** `0002_seed_shifts.sql` with Day, Evening and Night.
- **Presence** lives in Redis only: sorted set `pl:presence:staff`, scored by last-seen epoch, member `staffId`, plus `pl:presence:role:{role}` sets. A staff member counts as online if seen in the last 300 s. Writes are throttled to once per 30 s per staff member.
- **Migrations:** `0001_admin_init.sql`, `0002_seed_shifts.sql`.

## 5. API contracts

### 5.1 Routes owned by this doc (`/admin/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /bootstrap` | Admin session | none | `200 AdminBootstrap` | `401` |
| `GET /badges` | Admin session | none | `200 { review_queue, banks_unhealthy, tenants_pending }` (each null without the permission); polled every 30 s | none |
| `GET /dashboard` | `platform.dashboard.read` | `?range=today\|7d\|30d` | `200 Dashboard` | `400 invalid_field` |
| `GET /alerts` | `platform.dashboard.read` | `?kind&bank_code&cursor&limit` | `200 Page<Alert>` | none |
| `POST /alerts/{id}/ack` | `platform.review.reassign` or `platform.banks.manage` | none | `200 Alert` | `409 invalid_state` (already acknowledged) |
| `POST /search` | `platform.verifications.read` (verifications), `platform.review.read` (tasks), `platform.tenants.read` (tenants) | `{ q }` (2–128 characters) | `200 { exact: { type, id } \| null, verifications: [], tasks: [], tenants: [] }` | `400 invalid_field` |
| `GET /verifications/{id}/detail` | `platform.verifications.read` | none | `200 AdminVerificationDetail` | `404` |
| `GET /tenants/{id}/overview` | `platform.tenants.read` | none | `200 TenantOverview` | `404` |
| `GET /tenants/{id}/integrations` | `platform.tenants.read` | `?env=live\|test` | `200 { api_clients: ApiClientView[], webhooks: { endpoints, health } }` | `404` |
| `GET /tenants/{id}/activity` | `platform.audit.read` | `?cursor&limit` | `200 Page<AuditEventView>` | `404` |
| `GET /audit` | `platform.audit.read` | `?actor_kind&actor_id&action&target_type&target_id&tenant_id&from&to&cursor&limit` | `200 Page<AuditEventView>` | `400 invalid_field` |
| `GET /audit/chain` | `platform.audit.read` | none | `200 { checked_at, ok, broken_at_seq }` | none |
| `GET /audit/export.csv` | `platform.audit.read` | Same filters; range at most 31 days | `200 text/csv` streamed; audited `audit.exported` | `400 invalid_field`; `422 export_too_large` (over 100,000) |
| `GET /settings/flags` | `platform.settings.manage` | none | `200 { data: FlagView[] }` | none |
| `PUT /settings/flags/{key}` | `platform.settings.manage`, plus re-auth | `{ default_enabled, rules }` | `200 FlagView` | `400 invalid_field`; `401 reauth_required`; `404` |
| `GET /settings/review-shifts` | `platform.dashboard.read` | none | `200 { data: Shift[] }` | none |
| `PUT /settings/review-shifts` | `platform.settings.manage` | `{ shifts: [{ name, starts_at, ends_at }] }` (2–6 shifts) | `200 { data: Shift[] }` | `400 invalid_field` (gap or overlap; `details`) |
| `GET /settings/config` | `platform.settings.manage` | none | `200 { data: ConfigEntry[] }` | none |
| `PATCH /me/prefs` | Admin session | `{ theme }` | `200` | `400 invalid_field` |

### 5.2 Route map under `/admin/v1`

| Prefix | Owner |
| --- | --- |
| `/auth/*`, `/tenants`, `/tenants/{id}` (profile, KYB, suspend), `/tenants/{id}/settings`, `/staff*` | 01 |
| `/receiving-accounts*` (approvals, reveal) | 02 |
| `/banks*`, `/egress-nodes*`, `/evidence/*` | 04 |
| `/verifications`, `/verifications/{id}`, `/verifications/{id}/retry-bank`, `/verifications/{id}/reveal` | 05 |
| `/review/*` | 07 |
| `/tenants/{id}/webhooks/*` | 08 |
| `/tenants/{id}/api-clients/{id}/revoke` | 03 |
| `/billing/*`, `/tenants/{id}/plan`, `/tenants/{id}/usage` | 12 |
| everything in 5.1 | 11 |

### 5.3 Shapes

```json
// AdminBootstrap
{
  "staff": { "id": "stf_01J…", "name": "Mekdes Alemu", "roles": ["review_lead"], "theme": "system" },
  "permissions": ["platform.dashboard.read", "platform.review.read", "…"],
  "nav": [ { "id": "dashboard", "href": "/dashboard" }, { "id": "review", "href": "/review", "badge": 37 }, "…" ],
  "shift": { "name": "Day", "starts_at": "06:00", "ends_at": "14:00" },
  "presence": { "reviewers_online": 5, "review_leads_online": 1 },
  "deployment": { "label": "Production", "tone": "danger" },
  "locales": ["en"],
  "session": { "idle_timeout_seconds": 900 },
  "server_time": "2026-10-04T10:42:00+03:00"
}
```

`deployment.tone` is `danger` for production (red label, so nobody mistakes production for staging), `warning` for staging and `neutral` otherwise.

```json
// Dashboard (range=today)
{
  "range": { "key": "today", "from": "2026-10-04T00:00:00+03:00", "to": "2026-10-04T10:42:00+03:00" },
  "updated_at": "2026-10-04T10:42:00+03:00",
  "kpis": {
    "verifications": { "count": 12605, "bank_confirmed_ratio": 0.982 },
    "review_backlog": { "open": 37, "due_within_10m": 2 },
    "sla_breaches": { "count": 2, "escalated": 2 },
    "rejected": { "count": 652, "top_reason": { "reason": "duplicate", "count": 271 } }
  },
  "banks": [ { "code": "TELEBIRR", "name": "telebirr", "state": "degraded", "answered_1h": 0.847, "p95_ms": 6200,
               "note": { "key": "bank_note.half_open_since", "params": { "since": "2026-10-04T10:12:00+03:00" } } } ],
  "volume": { "unit": "hour", "buckets": [ { "start": "2026-10-04T00:00:00+03:00", "count": 260, "complete": true } ] },
  "outcomes": { "verified_bank": 11184, "verified_manual": 203, "rejected": 652, "reversed": 0, "expired_or_cancelled": 400,
                "open": { "total": 166, "receipt_required": 118, "in_review": 37, "processing": 11 } },
  "backlog": { "under_10m": 2, "from_10_to_30m": 11, "over_30m": 24,
               "oldest": { "task_id": "rvw_01J…", "number": 20402, "waiting_seconds": 3120, "tenant": { "id": "ten_01J…", "name": "Sheger Academy" },
                           "plan_name": "Starter", "sla_minutes": 240 } },
  "alerts": [ { "id": "…", "at": "2026-10-04T10:30:00+03:00", "severity": "critical", "kind": "breaker_opened",
                "title_key": "alert.breaker_opened.title", "detail_key": "alert.breaker_opened.detail", "params": { "bank": "M-Pesa" } } ]
}
```

`AdminVerificationDetail` has `verification` (doc 05 admin view with masked payer fields), `tenant` (`id`, `display_name`, plan), `attempts[]`, `timeline[]` (merged as in doc 09, section 6.5, but staff see reviewer names), `receipt` (summary), `review_task` (summary with the workspace link), `webhooks` (`VerificationDeliveryLine[]`), and `actions` (`retry_bank`, `view_evidence`, `reveal_pii`).

### 5.4 New error codes

None. This doc reuses `invalid_field`, `invalid_state`, `reauth_required` and `export_too_large` (doc 09).

### 5.5 Events

**Emits:** none. **Consumes:** `bank.breaker_changed`, `bank.format_changed`, `bank.adapter_toggled`, `bank.node_blocked`, `bank.parser_deployed` (doc 04), `review.sla_breached` (doc 07). They feed the alerts table (6.4).

## 6. Business rules

### 6.1 Navigation and badges

| Item | Permission | Badge |
| --- | --- | --- |
| Dashboard | `platform.dashboard.read` | none |
| Review queue | `platform.review.read` | Open platform-stage tasks (`ReviewApi.counts({ kind: 'platform' }).all`), amber |
| Verifications | `platform.verifications.read` | none |
| Tenants | `platform.tenants.read` | KYB `submitted` count (doc 01) plus accounts in `proof_submitted` (doc 02), amber |
| Banks & egress | `platform.banks.read` | Banks whose public status isn't `operational`, red |
| Staff & roles | `platform.staff.manage` | none |
| Audit log | `platform.audit.read` | Red dot when the chain check failed |
| Billing | `platform.billing.read` | Owned by doc 12 (for example, payments awaiting confirmation) |
| Settings | `platform.settings.manage` | none |

### 6.2 Dashboard definitions

- **Range:** live env only. `today` runs from EAT midnight to now. `7d` and `30d` are the last 7 or 30 EAT days including today.
- **Cohort:** verifications created in the range (`VerificationApi.platformStats`). Outcomes are the current status of that cohort, so the numbers add up to the count.
- **`bank_confirmed_ratio`:** `verified_bank / (verified_bank + verified_manual)`.
- **Review backlog** and the time-left buckets are "now" in every range. **SLA breaches** counts `review.sla_breached` actions in the range, and `escalated` counts those that reached a Review Lead (all, in Platform mode).
- **Caching:** results are cached 30 s per range; `7d` and `30d` for 5 minutes. Platform statistics read from the replica.

### 6.3 Bank tile note

Rules apply in order, and the first match wins:

1. Adapter disabled: "Adapter disabled: {reason}".
2. Breaker forced: "Forced {open \| closed} until {time}".
3. Breaker open: "Breaker open since {time}; new checks ask for a receipt".
4. Breaker half-open: "Breaker half-open since {time}".
5. A parser was deployed in the last 24 h: "Parser {version} live".
6. A node was blocked in the last hour: "Slow on {node}; traffic shifted".
7. Otherwise: "Breaker closed".

The state label maps doc 04's public status: `operational` "Healthy", `degraded` "Degraded", `down` "Down".

### 6.4 Alerts feed

| Event | Kind | Severity | Title | Detail |
| --- | --- | --- | --- | --- |
| `bank.breaker_changed` to `open` | `breaker_opened` | critical | "{Bank} breaker opened" | "{cause}. New {Bank} checks now return receipt_required." |
| `bank.breaker_changed` to `half_open` | `breaker_half_open` | warning | "{Bank} breaker half-open" | "Probing with 3 requests." |
| `bank.breaker_changed` to `closed` | `breaker_closed` | ok | "{Bank} recovered" | "Breaker closed; waiting checks are being retried." |
| `bank.breaker_changed` with `forced` | `breaker_forced` | warning | "{Bank} breaker forced {open \| closed}" | "By {staff} until {time}: {reason}" |
| `bank.format_changed` | `format_changed` | critical | "{Bank} page format changed" | "Parser {version} can't read responses. Checks go to review." |
| `bank.adapter_toggled` | `adapter_toggled` | warning | "{Bank} adapter {enabled \| disabled}" | "{reason}" |
| `bank.node_blocked` | `node_blocked` | warning | "{Bank} slow on {node}" | "Traffic shifted to the other node until {time}." |
| `bank.parser_deployed` | `parser_deployed` | ok | "{Bank} parser {version} deployed" | "Fixtures and canaries passing." |
| `review.sla_breached` | `sla_breach` | warning | "SLA breach on RV-{number}" | "{tenant} · {amount} · escalated to a Review Lead." |

The handler inserts with `source_event_id` unique, which makes it idempotent. Text is rendered from `alert.*` keys, never stored.

### 6.5 Shifts and presence

- The current shift is the one whose EAT range contains now. Ranges may cross midnight.
- Saving shifts validates that they cover 24 h with no gap or overlap.
- **Presence:** `presence.middleware.ts` runs on every authenticated admin-api request. "Reviewers online" counts online staff with `platform.review.decide`; "Review Leads" counts online staff with the `review_lead` role.
- Presence is shown as counts only, never as names, to avoid turning it into monitoring (open question 11-Q3).

### 6.6 Search

| Query looks like | Lookup |
| --- | --- |
| `ver_` plus 26 Crockford characters | `VerificationApi.get` with platform scope → `exact` |
| `RV-` plus digits, or digits of 5 or more | `ReviewApi.findByNumber` → `exact` |
| `ten_` plus 26 characters | `TenancyApi.getTenant` → `exact` |
| Anything else | In parallel: `VerificationApi.platformSearch(q)` (reference and order ID, last 90 days, at most 10) and `TenancyApi.searchTenants(q, 10)` (display or legal name contains `q`) |

Each group needs its own permission; a group without permission is omitted.

### 6.7 Masking in staff views

- **Payer name:** first letter of each word, then bullets for the rest: "BIRUK ASFAW" becomes "B•••• A••••".
- **Payer phone and accounts:** masked as everywhere (overview 5.10).
- **Reveal:** "Reveal payer details" calls doc 05's `POST /admin/v1/verifications/{id}/reveal` (`VerificationApi.revealPayer`) with `platform.pii.reveal` and a re-auth within 5 minutes. It is audited `pii.revealed` and shows the values for 5 minutes in that drawer only.
- **Receivers:** a receiving account's holder name is the tenant's business data and is shown unmasked.
- **Webhooks:** payloads and response bodies are never shown (doc 08).

### 6.8 Theme tokens (dark mode, admin only)

| Token | Light | Dark |
| --- | --- | --- |
| `canvas` | `#F6F7F5` | `#0F161D` |
| `surface` | `#FFFFFF` | `#16202A` |
| `line-200` | `#E2E6E3` | `#2A3642` |
| `ink-900` | `#14202B` | `#E6EBEF` |
| `ink-600` | `#536170` | `#A3AFBA` |
| `brand-600` (links, focus) | `#0F6E6A` | `#4FB3AC` (button text `#0F161D`) |
| Sidebar background and active item | `#14202B`, `#22313E` | Same |

Dark status pairs (text on background): verified `#6FD39A` on `#16302A`, rejected `#F28B84` on `#3A1E1F`, in review `#E9B867` on `#3A2E18`, receipt needed `#8DB6F5` on `#1A2A44`, reversed `#C99AE8` on `#2E2140`, expired and processing `#B4BDC6` on `#252E37`. All exceed 6:1, checked by the same unit test as doc 09.

### 6.9 Network allowlist

- The ingress for `admin.<domain>` accepts only `ADMIN_ALLOWED_CIDRS`.
- admin-api checks again with the client IP from the trusted proxy header (`X-Forwarded-For`, rightmost untrusted hop), and returns `403 forbidden` with a request ID otherwise. This also blocks a misconfigured ingress.

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db` (replica for statistics), `Redis` (presence), `OnEvent`, `AuditLog`, `AuditReader`, `ConfigRegistry`, `FeatureFlagsAdmin`, `Clock`, `i18n` |
| 01 tenancy | `getTenant`, `searchTenants(q, limit)`, `countKybSubmitted()`, `getStaff()` (names in alerts and the explorer); `SessionService` timeouts |
| 02 receiving accounts | `countAwaitingApproval()` (new) for the Tenants badge |
| 03 API clients | `ApiClientsApi.listForTenant`, `countActive` |
| 04 bank connectors | `BankHealth.snapshot`, `publicStatus`; admin endpoints (UI only); events `bank.*` |
| 05 verification | `get` (platform scope), `platformStats(range)`, `platformSearch(q)`, `timeline`; admin endpoints, including `/verifications/{id}/reveal` (`revealPayer`) |
| 06 receipts | `latestForVerification`, `getEvidence` (audited `receipt.viewed`) |
| 07 review | `ReviewApi.counts`, `openTaskForVerification`, `backlog(range)` (buckets, oldest, breaches in range), `findByNumber(n)` (new); event `review.sla_breached` |
| 08 webhooks | `WebhooksApi.deliveriesForVerification`, `health`; doc 08's admin endpoints |
| 12 billing | `PlansApi.getEffectivePlan` (tenant header and the oldest task's SLA text); `UsageApi.currentPeriod` (Tenant Overview tab) |

The dashboard fans out in parallel, with a 3 s timeout per part. A part that fails comes back `null`, and its card shows "Unavailable · retrying".

## 8. Implementation plan

1. **T11.1** Platform additions: `AuditReader`, `ConfigRegistry.describe`, `FeatureFlagsAdmin` (in `libs/platform`, with tests).
2. **T11.2** Migration and seed; alerts handler with idempotency; presence middleware and store; allowlist guard.
3. **T11.3** Bootstrap, badges, nav rules; admin shell with theme tokens and dark mode, search box, shift and presence footer, deployment label and clock.
4. **T11.4** Dashboard service and page exactly as the mockup: KPIs, bank strip, volume chart, outcomes, backlog, alerts; range tabs.
5. **T11.5** Search service and results popover.
6. **T11.6** Verifications explorer list and detail with Retry bank, break-glass evidence, PII reveal.
7. **T11.7** Tenant detail frame; Overview, Integrations, Activity tabs; slots for the owners' tabs.
8. **T11.8** Banks & egress screens on doc 04's endpoints, with reason dialogs.
9. **T11.9** Audit log screen, chain badge, CSV stream.
10. **T11.10** Settings: flags editor, shifts editor, effective configuration.
11. **T11.11** Keyboard, accessibility and dark-mode review across admin screens, including doc 07's pages.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Replica lagging | Statistics read from the replica may trail by seconds; "updated" shows the cache time. If replica lag exceeds 60 s, the service reads the primary for `today` and keeps the replica for 7 and 30 days |
| Midnight EAT | `today` resets to the new day; the chart shows hour 00 in progress |
| A bank event burst (a flapping breaker) | Every event becomes an alert row; the feed collapses consecutive same-kind alerts for one bank within 10 min into one line with "×4" |
| Staff member opens a test-env verification by ID | Shown with a "Test" chip; test verifications are excluded from all dashboard numbers |
| Presence store unavailable (Redis down) | The footer shows "Presence unavailable"; nothing else is affected |
| Audit chain broken | Red badge, an alert from the platform maintenance job's daily chain check (overview 5.6), and the audit page header shows the first broken `seq` |
| Shifts edited mid-shift | The footer updates on the next bootstrap or badges poll |
| A staff member loses a permission while a page is open | The next call returns `403`; the page shows Forbidden, and the nav updates on the next bootstrap |
| Search for a reference used at many tenants | At most 10 results, newest first, each with its tenant |
| Break-glass URL expires before the file opens | "This link expired. Request access again"; a new request needs a new reason |

## 10. Security and privacy

| Route | AuthN | AuthZ |
| --- | --- | --- |
| `/admin/v1/*` (this doc) | Keycloak OIDC admin session, MFA, CSRF on mutations; network allowlist | Platform permissions per 5.1; cross-tenant access read-only (overview 5.4) |

- Every mutating action here is audited: alert ack, flags, shifts, theme (not audited, personal), audit export, reveal, retry bank, evidence access. The audited actions are written by their owning module (doc 04, 05) or this one.
- Staff never see webhook payloads, request-log bodies, API secrets or webhook secrets.
- **Security headers:** CSP as in doc 09 with `frame-ancestors 'none'`, plus `X-Robots-Tag: noindex`. The admin origin sets no cookies outside `admin.<domain>`.
- Reveal flows and break-glass need re-auth (OIDC `max_age=300`, doc 01) and a reason where stated.

## 11. Performance

| Item | Target |
| --- | --- |
| Dashboard | p95 under 500 ms uncached; under 50 ms cached |
| Badges | p95 under 100 ms |
| Search | p95 under 500 ms |
| Explorer list | p95 under 400 ms for a 1-day range; 31-day ranges under 1.5 s |
| Verification detail | p95 under 500 ms |
| Audit list | p95 under 300 ms (index on `at`, `tenant_id`, `action`); CSV stream at 5,000 rows a second or more |
| Web | Designed for 1440 px, usable at 1024 px; lists paged on the server; review workspace keyboard-only (doc 07) |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Nav and badges per role; bank note rules in order; alert mapper for each event; shift selection across midnight and validation of gaps and overlaps; search parsing; masks; dark and light token contrast |
| Integration | Dashboard against fakes of docs 04, 05, 07, 12, including a failed part; alerts idempotency and collapsing; presence counts with `FakeClock`; allowlist guard with forwarded headers; audit list, chain status and CSV stream; flags update visible within 30 s |
| End to end | Playwright: Review Lead lands on the dashboard (sandbox data seeded through the API), sees KPIs and the alerts feed; forcing the M-Pesa breaker open on Banks & egress shows a Down tile and an alert within 30 s, and a sandbox check in a tenant returns `receipt_required`; explorer search by reference, open detail, Retry bank; audit log shows the breaker action |
| Visual and accessibility | Dashboard screenshot compared with the mockup at 1440 px by a person; axe on every admin page in light and dark; keyboard walk-through |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_admin_bff_duration_seconds` | histogram | `route` |
| `pl_admin_dashboard_part_failures_total` | counter | `part` |
| `pl_admin_alerts_total` | counter | `kind`, `severity` |
| `pl_admin_staff_online` | gauge | `role` |
| `pl_admin_allowlist_denied_total` | counter | none |

| Alert | Condition | Severity |
| --- | --- | --- |
| Allowlist denials | Any denial in 5 min (the ingress should have stopped it) | Warning |
| Dashboard parts failing | More than 10% failures over 10 min | Warning |
| Nobody on the review desk | `pl_admin_staff_online{role="reviewer"} == 0` for 15 min while open platform tasks exist | Critical (pages the on-call Review Lead) |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `ADMIN_ALLOWED_CIDRS` | none (required) | Networks allowed to reach admin-api |
| `ADMIN_TRUSTED_PROXY_CIDRS` | none (required) | Proxies whose `X-Forwarded-For` is trusted |
| `ADMIN_LOCALES` | `en` | `en` or `en,am` (overview Q12) |
| `DEPLOY_ENV_LABEL` | none (required) | "Production", "Staging", … |
| `DEPLOY_ENV_TONE` | `neutral` | `danger`, `warning` or `neutral` |
| `ADMIN_DASHBOARD_CACHE_SECONDS` | 30 | Cache for `today` (7 and 30 days: 300) |
| `ADMIN_PRESENCE_WINDOW_SECONDS` | 300 | "Online" window |
| `ADMIN_DASHBOARD_PART_TIMEOUT_MS` | 3000 | Fan-out timeout per part |
| `ADMIN_ALERTS_RETENTION_DAYS` | 90 | Alert retention |

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 11-Q1 | The admin mockups show an EN / አማ switch, but overview Q12 defaults to English only. | All admin strings live in catalogues; the switch appears when `ADMIN_LOCALES` includes `am` |
| 11-Q2 | The UI spec's Settings page edits "SLA targets, thresholds, retention, extraction provider, message templates". Should those be runtime-editable? | No. They stay deploy-time configuration (reviewed, versioned) and show read-only; only flags and shifts are editable |
| 11-Q3 | Is it acceptable to show which staff are online by name? | Counts only |
| 11-Q4 | The UI spec names Ops, Support, Engineer, Finance, Auditor and Super Admin roles; doc 01 has Reviewer, Review Lead and Admin. | Ship the three roles. Others are added as data (roles are rows) when the team grows |
| 11-Q5 | Should SLA clocks pause outside staffed shifts (overview Q6)? | No; shifts are informational |
