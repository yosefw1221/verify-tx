# 12 · Billing and usage

| | |
| --- | --- |
| Package | `@proofline/billing` |
| Postgres schema | `billing` |
| Depends on | platform, 01 tenancy. Payments are verified by calling Proofline's own public API over HTTP with a house API key, as any merchant would (section 6.9). That is deliberately not an import of doc 05, which would create a cycle |
| Build step | **12a** (plans, subscriptions, metering, quota, usage page) in step 2, before the verification engine. **12b** (invoices, payments, dunning, plan self-service, admin billing) in step 9, after the pilot |
| Mockups | Portal sidebar usage meter ("API checks this month 6,412 / 10,000 · Resets 1 November · Usage"); Admin · Review queue's "4 h Starter SLA" (the plan's review SLA). The Billing pages are not drawn |

## 1. Purpose and scope

**Does**

- **12a:**
  - the plan catalogue;
  - one subscription per tenant (Sandbox on signup);
  - metering of billable outcomes from events into append-only usage events;
  - Redis counters reconciled with the database;
  - `QuotaService.check()` for doc 05, enforcing R12's 120% hard limit per metered channel;
  - `PlansApi` (rate limits for doc 03, review SLA for doc 07);
  - `UsageApi` (portal meter and pages);
  - 80%, 100% and 120% notices;
  - the portal "Billing & usage" page (plan and usage parts);
  - admin plan assignment.
- **12b:**
  - monthly invoices in ETB with 15% VAT, gapless numbers and bilingual PDFs;
  - payment by telebirr or bank transfer, verified through our own API;
  - credit notes and credit balance;
  - dunning and suspension for non-payment;
  - self-service upgrade and downgrade with proration;
  - the admin Billing section and the tenant Plan & usage tab.

**Out of scope**

- Holding or moving money. Tenants pay into our accounts; we only verify.
- Card payments and payment gateways.
- Plan entitlements other than quota, rate limit and review SLA (receiving-account, branch and client limits stay flat in docs 01–03; open question 12-Q4).
- Fiscal-device integration. Whether invoices must go through a sales register machine or the tax authority's e-invoicing service is open question 12-Q2, and it gates paid launch.

## 2. Screens and user stories

| Screen | Surface | Part | Mockup |
| --- | --- | --- | --- |
| Usage meter in the sidebar | Portal | 12a | Every portal mockup |
| Billing & usage: plan card, meters, daily usage | Portal | 12a | Not drawn |
| Billing & usage: upcoming charges, invoices, payments, billing details, Change plan, Pay invoice | Portal | 12b | Not drawn |
| Tenant tab "Plan & usage" | Admin | 12a (plan and usage), 12b (invoices) | Not drawn |
| Billing: invoices, invoice detail, payments, plan catalogue | Admin | 12b | Not drawn |

**US-12.1 Sandbox on signup (12a)**

- On `tenant.created`, the tenant gets a subscription to the current Sandbox plan. Its period is the current EAT calendar month, and it is `active`.
- The sidebar meter shows "Checks this month 0 / 50".

**US-12.2 Meter label**

- Under R12 (accepted), every seeded plan meters all three channels, so the meter reads "Checks this month 6,412 / 10,000". The mockup's "API checks" wording is replaced (09-Q1).
- A plan that meters only the API channel (possible through `metered_channels`, none at launch) shows "API checks this month".
- The meter turns amber at 80%, red at 100%, and shows "Paused" when blocked.

**US-12.3 Quota check (12a)**

- `QuotaService.check({ tenantId, env, channel })` returns `allow` in test env, and for a channel the plan doesn't meter.
- Otherwise it returns `block` when the period's metered count is at or above `included × hard_limit_pct / 100`, or when the subscription is `suspended`. It returns `allow` otherwise. Doc 05 maps `block` to `402 quota_exceeded`.
- With Growth (10,000 included, 120%): the 12,000th billable verification (any channel) is allowed, and the next create is refused on every channel. On a plan that meters only the API, a portal or cashier check would still be allowed.

**US-12.4 Metering (12a)**

Usage events are inserted exactly once per source; replays and duplicate event deliveries change nothing:

| Usage event | Inserted on |
| --- | --- |
| `verification` | Each live `verification.finalized` with `billable: true` |
| `platform_review` | Each live `verification.finalized` with `reviewed_by: platform` |
| `image_credit` | The first live `receipt.extracted` of a receipt in which any candidate came from `vision` |

The counter for the period goes up within 5 s of the event.

**US-12.5 Notices (12a)**

- When the metered count first reaches 80%, 100% and 120% of the included quota in a period, `usage.threshold_reached` is emitted once per level. The tenant's alert emails and Owners receive:
  - "You've used 80% of this month's checks";
  - at 100%: "…all included checks; up to 2,000 more are billed at ETB 1.00 each";
  - at 120%: "…live checks are paused until 1 November or until you upgrade".

**US-12.6 Billing & usage page, usage part (12a)**

- The page (needs `billing.read`) shows:
  - **Plan card:** name, monthly fee before VAT, period dates, "Resets 1 November".
  - **Meters:** the quota meter with markers at 100% and the hard limit; image credits used of included; platform reviews this period.
  - **Daily usage:** a 30-day bar chart stacked by channel (API, portal, cashier), where unmetered channels appear hatched with the legend "Not counted on your plan".

**US-12.7 Change plan (12b)**

- "Change plan" (needs `billing.manage` and re-authentication) lists the public plans with fee, included checks, overage, image credits, review fee and rate limit.
- **An upgrade** takes effect at once. The new plan's included quota applies to the whole current month, and proration lines appear on the next invoice (6.5).
- **A downgrade** is scheduled for the next period start and shows "Changes to Starter on 1 November". It can be cancelled until then.
- Enterprise isn't self-service: "Talk to us" opens a mailto link to `BILLING_SALES_EMAIL`.

**US-12.8 Monthly invoice (12b)**

- At 00:10 EAT on the 1st, each tenant on a paid plan gets one invoice. It contains:
  - the new month's base fee (in advance);
  - the previous month's verification overage, image overage and platform reviews (in arrears), each as quantity × unit price;
  - any proration lines and credits;
  - the subtotal, 15% VAT and total.
- It is numbered `INV-2026-000042` from a gapless yearly series, issued, emailed to the billing email and Owners, and due in 10 days.
- An invoice with a total of 0.00 after credits is issued and marked paid at once.

**US-12.9 Invoice PDF (12b)**

- The PDF carries bilingual labels (English and Amharic) and:
  - our legal name, TIN, address and VAT registration;
  - the tenant's legal name, TIN and billing address as they were at issue;
  - the number, issue date in both calendars, due date, period, the lines, VAT and total in ETB;
  - payment instructions (the pay-to accounts and "Pay in the portal").
- Invoice PDFs are stored and can't be changed after issue. Corrections are credit notes.

**US-12.10 Pay an invoice (12b)**

- "Pay" shows the amount due and our pay-to accounts with full numbers and copy buttons. It says: "Pay exactly ETB 9,085.00 or more to one of these accounts, then enter the reference or upload the receipt."
- Submitting creates a payment `verifying`. The billing worker verifies it through our own API (6.9).
- If verified, the payment is `confirmed`, the invoice `paid`, and any amount above the due becomes credit balance.
- If rejected, the payment shows the reason label from doc 09, for example "Paid to a different account".
- If in review, the payment shows "We're checking your payment, usually within 30 minutes".

**US-12.11 Dunning (12b)**

- Reminders go out 3 days before the due date, on it, and 3 days after.
- The day after the due date, the subscription is `past_due` and the portal shows an amber banner.
- 14 days after the due date, the subscription is `suspended`: quota returns `block` for every live channel, and the portal shows "Live checks are paused until invoice INV-2026-000042 is paid".
- A confirmed payment that clears every overdue invoice sets the subscription back to `active` at once.

**US-12.12 Admin billing (12b)**

- Finance (`platform.billing.read`, and `.manage` for actions) lists invoices by status, tenant and period, and opens an invoice with its lines, payments, credit notes and PDF.
- "Mark paid" records a manual payment with a reference and note, for enterprise bank transfers, and can optionally verify it through the house API.
- "Void" applies only before any payment and needs a reason.
- "Issue credit" takes an amount and reason, and applies to an open invoice or to the credit balance.
- "Payments awaiting confirmation" lists payments in `pending_review`.

**US-12.13 Plan catalogue (12a for assignment, 12b for editing)**

- Admins with `platform.billing.manage` list plans and create a new version of a plan. Plans are immutable once a subscription uses them; a new price means a new version, and existing subscribers move only when an admin migrates them, effective next period.
- They can create a private plan for an enterprise deal, retire a plan (no new subscriptions), and change a tenant's plan with a reason. Every change is audited.

**US-12.14 Account closure (12b)**

- When the tenant becomes `closed`, the subscription is `cancelled`. A final invoice covers arrears up to the closure date, with no base fee for the next month. Credit balance is shown for manual refund (open question 12-Q5).

## 3. Module design

```text
libs/features/billing/src/
├─ public/
│  ├─ index.ts                 BillingModule, QuotaService, PlansApi, UsageApi, events
│  └─ billing-api.ts
├─ domain/
│  ├─ period.ts                EAT calendar months, period keys "2026-10" (pure)
│  ├─ quota.ts                 decision from plan, count, subscription (pure)
│  ├─ thresholds.ts            levels reached between two counts (pure)
│  ├─ invoice-builder.ts       lines, proration, VAT, rounding (pure)
│  └─ dunning.ts               schedule and status transitions (pure)
├─ application/
│  ├─ subscription.service.ts  create Sandbox, change plan, schedule, cancel
│  ├─ metering.handler.ts      @OnEvent verification.finalized, receipt.extracted
│  ├─ tenant.handler.ts        @OnEvent tenant.created, tenant.status_changed
│  ├─ quota.service.ts         QuotaService implementation (Redis fast path)
│  ├─ reconcile.processor.ts   bill:reconcile hourly
│  ├─ invoicing.processor.ts   bill:close:{period}                      (12b)
│  ├─ dunning.processor.ts     bill:dunning:{date}                      (12b)
│  ├─ payments.service.ts      submit, verify, confirm, manual          (12b)
│  ├─ payment-verify.processor.ts  bill:pay:{paymentId}:{n}             (12b)
│  ├─ house-api.client.ts      HTTP client for our own public API       (12b)
│  ├─ pdf.service.ts           pdfmake invoice rendering                (12b)
│  └─ notifier.handler.ts      threshold, invoice and dunning emails
├─ infrastructure/ plans.repo.ts, subscriptions.repo.ts, usage.repo.ts, counters.store.ts, invoices.repo.ts, payments.repo.ts
├─ http/portal/billing.controller.ts    /portal/v1/billing/*, /portal/v1/usage
├─ http/admin/billing.controller.ts     /admin/v1/billing/*, /admin/v1/tenants/{id}/plan, /usage
└─ db/schema.ts, db/migrations/
```

```ts
// public/billing-api.ts
export type MeteredChannel = 'api' | 'portal' | 'cashier';

export interface QuotaService {
  /** Called by doc 05 before creating a live verification. Never throws for quota; throws only if storage is down. */
  check(q: { tenantId: string; env: EnvName; channel: MeteredChannel }): Promise<QuotaDecision>;
}
export type QuotaDecision =
  | { decision: 'allow'; state: UsageState }
  | { decision: 'block'; state: 'blocked'; reason: 'quota' | 'unpaid' };
export type UsageState = 'ok' | 'warning' | 'over' | 'blocked';

export interface PlansApi {
  getEffectivePlan(tenantId: string): Promise<EffectivePlan>;      // cached 60 s; invalidated on subscription.changed
}
export interface EffectivePlan {
  planId: string; code: string; version: number; name: string;
  rateLimitPerMinute: number;                // doc 03
  reviewSlaMinutes: number;                  // doc 07
  includedVerifications: number | null;      // null = unlimited (custom enterprise)
  meteredChannels: MeteredChannel[];
  hardLimitPct: number | null;               // null = no hard limit
  includedImageCredits: number | null;       // null = not metered
}

export interface UsageApi {
  currentPeriod(tenantId: string): Promise<CurrentUsage>;          // cached 60 s; portal meter, docs 09 and 11
  daily(tenantId: string, range: { from: Date; to: Date }): Promise<DailyUsage[]>;   // at most 92 days
}
export interface CurrentUsage {
  periodKey: string; periodStart: Date; periodEnd: Date;
  used: number; included: number | null; hardLimit: number | null;
  meteredChannels: MeteredChannel[];
  state: UsageState; blockedReason: 'quota' | 'unpaid' | null;
  imageCreditsUsed: number; imageCreditsIncluded: number | null;
  platformReviews: number;
}
export interface DailyUsage { date: string; api: number; portal: number; cashier: number; imageCredits: number; platformReviews: number }
```

## 4. Data model

```sql
CREATE TABLE billing.plans (
  id                              uuid PRIMARY KEY,
  code                            text NOT NULL,                 -- 'sandbox', 'starter', 'growth', 'ent_<slug>'
  version                         integer NOT NULL,
  name                            text NOT NULL,
  status                          text NOT NULL CHECK (status IN ('active','retired')),
  public                          boolean NOT NULL,              -- shown in Change plan
  monthly_fee_minor               bigint NOT NULL CHECK (monthly_fee_minor >= 0),
  included_verifications          integer NULL,
  metered_channels                text[] NOT NULL CHECK (metered_channels <@ ARRAY['api','portal','cashier']::text[]),
  hard_limit_pct                  integer NULL CHECK (hard_limit_pct BETWEEN 100 AND 1000),
  overage_per_verification_minor  bigint NULL,                   -- NULL = no overage (blocked at the hard limit)
  included_image_credits          integer NULL,
  overage_per_image_minor         bigint NULL,
  platform_review_fee_minor       bigint NOT NULL DEFAULT 0,
  rate_limit_per_minute           integer NOT NULL CHECK (rate_limit_per_minute BETWEEN 1 AND 100000),
  review_sla_minutes              integer NOT NULL CHECK (review_sla_minutes BETWEEN 5 AND 1440),
  sort                            smallint NOT NULL DEFAULT 0,
  created_by                      uuid NULL,
  created_at                      timestamptz NOT NULL DEFAULT now(),
  UNIQUE (code, version)
);

CREATE TABLE billing.subscriptions (
  id                    uuid PRIMARY KEY,
  tenant_id             uuid NOT NULL UNIQUE REFERENCES tenancy.tenants(id),
  plan_id               uuid NOT NULL REFERENCES billing.plans(id),
  status                text NOT NULL CHECK (status IN ('active','past_due','suspended','cancelled')),
  current_period_key    char(7) NOT NULL,                        -- '2026-10'
  scheduled_plan_id     uuid NULL REFERENCES billing.plans(id),  -- downgrade at the next period
  started_at            timestamptz NOT NULL DEFAULT now(),
  cancelled_at          timestamptz NULL,
  updated_at            timestamptz NOT NULL DEFAULT now(),
  version               integer NOT NULL DEFAULT 0
);

CREATE TABLE billing.subscription_changes (
  id               uuid PRIMARY KEY,
  subscription_id  uuid NOT NULL REFERENCES billing.subscriptions(id),
  tenant_id        uuid NOT NULL,
  from_plan_id     uuid NULL,
  to_plan_id       uuid NOT NULL,
  kind             text NOT NULL CHECK (kind IN ('initial','upgrade','downgrade','admin','migration')),
  effective_at     timestamptz NOT NULL,
  actor_kind       text NOT NULL CHECK (actor_kind IN ('user','staff','system')),
  actor_id         uuid NULL,
  reason           text NULL CHECK (length(reason) <= 500),
  proration_done   boolean NOT NULL DEFAULT false,
  created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE billing.usage_events (
  id           uuid NOT NULL,
  occurred_at  timestamptz NOT NULL,
  tenant_id    uuid NOT NULL,
  kind         text NOT NULL CHECK (kind IN ('verification','platform_review','image_credit')),
  channel      text NULL CHECK (channel IN ('api','portal','cashier')),
  source_id    uuid NOT NULL,                     -- verification_id, or receipt_id for image_credit
  period_key   char(7) NOT NULL,                  -- EAT month of occurred_at
  PRIMARY KEY (id, occurred_at),
  UNIQUE (kind, source_id, occurred_at)
) PARTITION BY RANGE (occurred_at);             -- monthly; kept 24 months
CREATE INDEX usage_tenant_period ON billing.usage_events (tenant_id, period_key, kind, channel);

CREATE TABLE billing.usage_dedupe (              -- global uniqueness, which a partitioned table can't enforce across months
  kind       text NOT NULL,
  source_id  uuid NOT NULL,
  PRIMARY KEY (kind, source_id)
);

CREATE TABLE billing.usage_counters (
  tenant_id      uuid NOT NULL,
  period_key     char(7) NOT NULL,
  meter          text NOT NULL,                   -- 'metered', 'api', 'portal', 'cashier', 'image_credit', 'platform_review'
  count          integer NOT NULL,
  reconciled_at  timestamptz NOT NULL,
  PRIMARY KEY (tenant_id, period_key, meter)
);

CREATE TABLE billing.threshold_notices (
  tenant_id   uuid NOT NULL,
  period_key  char(7) NOT NULL,
  pct         smallint NOT NULL CHECK (pct IN (80, 100, 120)),
  sent_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, period_key, pct)
);

-- 12b
CREATE TABLE billing.profiles (
  tenant_id      uuid PRIMARY KEY REFERENCES tenancy.tenants(id),
  billing_email  citext NULL,
  address_line   text NULL CHECK (length(address_line) <= 200),
  city           text NULL CHECK (length(city) <= 80),
  updated_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE billing.invoice_number_counters (
  year  integer PRIMARY KEY,
  last  integer NOT NULL
);

CREATE TABLE billing.invoices (
  id                     uuid PRIMARY KEY,
  number                 text NULL UNIQUE,              -- set at issue: 'INV-2026-000042'
  tenant_id              uuid NOT NULL REFERENCES tenancy.tenants(id),
  kind                   text NOT NULL CHECK (kind IN ('monthly','final')),
  period_key             char(7) NOT NULL,              -- the month whose base fee it carries ('final': closure month)
  status                 text NOT NULL CHECK (status IN ('draft','issued','paid','void')),
  currency               char(3) NOT NULL DEFAULT 'ETB' CHECK (currency = 'ETB'),
  subtotal_minor         bigint NOT NULL,
  vat_minor              bigint NOT NULL,
  total_minor            bigint NOT NULL,
  credit_applied_minor   bigint NOT NULL DEFAULT 0,
  amount_paid_minor      bigint NOT NULL DEFAULT 0,
  bill_to                jsonb NOT NULL,                -- snapshot: legal_name, tin, address, email
  issued_at              timestamptz NULL,
  due_at                 timestamptz NULL,
  paid_at                timestamptz NULL,
  voided_at              timestamptz NULL,
  void_reason            text NULL,
  pdf_object_key         text NULL,                     -- bucket "invoices"
  pdf_sha256             bytea NULL,
  created_at             timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX invoices_one_per_period ON billing.invoices (tenant_id, period_key, kind) WHERE status <> 'void';
CREATE INDEX invoices_status_due ON billing.invoices (status, due_at);

CREATE TABLE billing.invoice_lines (
  id                uuid PRIMARY KEY,
  invoice_id        uuid NOT NULL REFERENCES billing.invoices(id),
  kind              text NOT NULL CHECK (kind IN ('base_fee','proration_credit','proration_charge','verification_overage',
                      'image_overage','platform_review','credit','adjustment')),
  description_key   text NOT NULL,                      -- i18n key; params below
  params            jsonb NOT NULL DEFAULT '{}'::jsonb,
  quantity          integer NOT NULL,
  unit_price_minor  bigint NOT NULL,
  amount_minor      bigint NOT NULL,                    -- negative for credits
  sort              smallint NOT NULL
);

CREATE TABLE billing.payments (
  id                      uuid PRIMARY KEY,
  invoice_id              uuid NOT NULL REFERENCES billing.invoices(id),
  tenant_id               uuid NOT NULL,
  method                  text NOT NULL CHECK (method IN ('telebirr','bank_transfer','manual')),
  bank_code               text NULL,
  reference               text NULL CHECK (length(reference) <= 2048),
  house_verification_id   text NULL,                    -- 'ver_…' in the house tenant
  status                  text NOT NULL CHECK (status IN ('verifying','pending_review','confirmed','rejected','failed')),
  amount_minor            bigint NULL,                  -- credited amount once confirmed
  reject_reason           text NULL,
  note                    text NULL CHECK (length(note) <= 500),
  submitted_by_kind       text NOT NULL CHECK (submitted_by_kind IN ('user','staff')),
  submitted_by_id         uuid NOT NULL,
  confirmed_at            timestamptz NULL,
  created_at              timestamptz NOT NULL DEFAULT now(),
  updated_at              timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX payments_reference_once ON billing.payments (bank_code, reference)
  WHERE status IN ('verifying','pending_review','confirmed') AND reference IS NOT NULL;

CREATE TABLE billing.credit_notes (
  id                  uuid PRIMARY KEY,
  number              text NOT NULL UNIQUE,              -- 'CN-2026-000007'
  tenant_id           uuid NOT NULL,
  amount_minor        bigint NOT NULL CHECK (amount_minor > 0),
  reason              text NOT NULL CHECK (length(reason) BETWEEN 3 AND 500),
  source              text NOT NULL CHECK (source IN ('staff','overpayment')),
  issued_by           uuid NULL,
  applied_invoice_id  uuid NULL REFERENCES billing.invoices(id),
  applied_at          timestamptz NULL,
  created_at          timestamptz NOT NULL DEFAULT now()
);
```

- **Tenant-level, env-less:** billing rows are per tenant (only live usage is billed). RLS checks `tenant_id` on subscriptions, usage, invoices, lines, payments, credit notes and profiles. `plans` and the counters table are platform-level, readable by `pl_app`, and writable only by billing's repositories.
- **Partitions:** `usage_events` monthly, created 3 months ahead, dropped after 24 months. Invoices keep the line totals forever.
- **Bucket:** a new `invoices` bucket (overview 5.18), key `t/{tenantId}/invoices/{number}.pdf`, retained 10 years (open question 12-Q3).
- **Seed:** `0003_seed_plans.sql` inserts the plans in 6.2.
- **Migrations:** `0001_billing_init.sql` (12a tables), `0002_usage_partitions.sql`, `0003_seed_plans.sql`, `0004_invoicing.sql` (12b tables).

## 5. API contracts

### 5.1 Portal (`/portal/v1`)

| Method and path | Permission | Request | Success | Errors | Part |
| --- | --- | --- | --- | --- | --- |
| `GET /usage` | Session (any role; the sidebar meter) | none | `200 CurrentUsage` (snake_case) | none | 12a |
| `GET /billing/overview` | `billing.read` | none | `200 { plan: EffectivePlan, subscription: { status, scheduled_plan, period_end }, usage: CurrentUsage, upcoming: UpcomingCharges }` | none | 12a (`upcoming` in 12b) |
| `GET /billing/usage/daily` | `billing.read` | `?from&to` (at most 92 days) | `200 { data: DailyUsage[] }` | `400 invalid_field` | 12a |
| `GET /billing/plans` | `billing.read` | none | `200 { data: PublicPlan[] }` | none | 12b |
| `POST /billing/plan` | `billing.manage`, plus re-auth | `{ plan_code }` | `200 { subscription, effective_at, proration_preview }` | `401 reauth_required`; `409 invalid_state` (same plan, or a downgrade already scheduled to it); `422 plan_not_available` | 12b |
| `DELETE /billing/plan/scheduled` | `billing.manage` | none | `200` (cancels a scheduled downgrade) | `409 invalid_state` | 12b |
| `GET /billing/invoices` | `billing.read` | `?status&cursor` | `200 Page<InvoiceRow>` | none | 12b |
| `GET /billing/invoices/{id}` | `billing.read` | none | `200 Invoice` (lines, payments) | `404` | 12b |
| `GET /billing/invoices/{id}/pdf` | `billing.read` | none | `302` to a 300 s signed URL | `404`; `409 invalid_state` (draft) | 12b |
| `GET /billing/pay-to` | `billing.read` | none | `200 { accounts: [{ bank_code, bank_name, account_name, account_number }] }` | none | 12b |
| `POST /billing/invoices/{id}/payments` | `billing.manage` | JSON `{ bank_code, reference }` or multipart `data` + `file`; `X-Idempotency-Key` | `202 Payment` | `409 invalid_state` (invoice not `issued`); `409 payment_reference_used`; `400 invalid_field`; `413`; `415` | 12b |
| `GET /billing/payments/{id}` | `billing.read` | none | `200 Payment` | `404` | 12b |
| `GET /billing/profile`, `PATCH /billing/profile` | `billing.read`; `billing.manage` | `{ billing_email?, address_line?, city? }` | `200 Profile` | `400 invalid_field` | 12b |

```json
// UpcomingCharges (12b), an estimate shown as "Next invoice on 1 November"
{ "issue_date": "2026-11-01", "lines": [
    { "kind": "base_fee", "label_key": "billing.line.base_fee", "params": { "plan": "Growth", "month": "2026-11" }, "amount": "7900.00" },
    { "kind": "verification_overage", "quantity": 0, "unit_price": "1.00", "amount": "0.00" },
    { "kind": "platform_review", "quantity": 31, "unit_price": "10.00", "amount": "310.00" } ],
  "subtotal": "8210.00", "vat": "1231.50", "total": "9441.50", "credit_balance": "0.00" }
```

### 5.2 Admin (`/admin/v1`)

| Method and path | Permission | Notes | Part |
| --- | --- | --- | --- |
| `GET /tenants/{id}/plan` | `platform.tenants.read` | Subscription, effective plan, changes history | 12a |
| `POST /tenants/{id}/plan` | `platform.billing.manage` | `{ plan_id, effective: "now" \| "next_period", reason }`; audited `billing.plan_changed` | 12a |
| `GET /tenants/{id}/usage` | `platform.tenants.read` | `CurrentUsage` and the last 30 days of `DailyUsage` | 12a |
| `GET /billing/plans` | `platform.billing.read` | All plans and versions with subscriber counts | 12a |
| `POST /billing/plans` | `platform.billing.manage`, plus re-auth | A new plan, or a new version of `code`; audited | 12b |
| `POST /billing/plans/{id}/retire` | `platform.billing.manage` | `409 invalid_state` if it is the default Sandbox plan | 12b |
| `POST /billing/plans/{id}/migrate` | `platform.billing.manage`, plus re-auth | `{ to_plan_id }`: moves every subscriber at the next period; audited | 12b |
| `GET /billing/invoices` | `platform.billing.read` | `?status&tenant_id&period_key&overdue&cursor` | 12b |
| `GET /billing/invoices/{id}` | `platform.billing.read` | Lines, payments, credit notes, PDF link | 12b |
| `POST /billing/invoices/{id}/mark-paid` | `platform.billing.manage` | `{ method: "bank_transfer" \| "manual", reference?, amount, note, verify_via_api: boolean }`; audited | 12b |
| `POST /billing/invoices/{id}/void` | `platform.billing.manage` | `{ reason }`; only with no payments; audited | 12b |
| `POST /billing/credit-notes` | `platform.billing.manage`, plus re-auth | `{ tenant_id, amount, reason, invoice_id? }`; audited | 12b |
| `GET /billing/payments` | `platform.billing.read` | `?status=pending_review\|verifying\|rejected&cursor` | 12b |
| `POST /billing/invoices/{id}/reissue-pdf` | `platform.billing.manage` | Regenerates the PDF from stored data, which must hash-match (template fixes only); audited | 12b |

### 5.3 New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `plan_not_available` | 422 | Plan is retired, private or not public |
| `payment_reference_used` | 409 | This reference is already attached to a payment |

`quota_exceeded` (402) is in the overview catalogue. Its `details` carry `{ reason: "quota" | "unpaid", period_end, used, hard_limit }`.

### 5.4 Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `usage.threshold_reached` | `{ tenant_id, period, pct }` | This module's notifier |
| `subscription.changed` | `{ tenant_id, from_plan_id, to_plan_id, effective_at, kind }` | 03 (drops its 60 s plan cache), 07 (none; snapshots at task creation) |
| `subscription.status_changed` | `{ tenant_id, from, to }` | This module's notifier |
| `invoice.issued` | `{ invoice_id, tenant_id, number, total_minor, due_at }` | This module's notifier |
| `invoice.paid` | `{ invoice_id, tenant_id }` | This module's notifier |

**Consumes:** `tenant.created`, `tenant.status_changed` (doc 01); `verification.finalized` (doc 05); `receipt.extracted` (doc 06).

## 6. Business rules

### 6.1 Periods

- A period is an EAT calendar month, from the 1st at 00:00 EAT up to the next 1st (`period.ts`). The key is `YYYY-MM` of the EAT start.
- Usage is attributed to the period of `occurred_at`, which is the event's `occurredAt`, not the verification's creation.
- A subscription's `current_period_key` rolls over in `bill:close:{period}` at 00:10 EAT on the 1st. That job applies scheduled downgrades first.

### 6.2 Seed plans

These defaults hold until open question Q5 is answered. They follow the architecture doc's proposal; the prices are placeholders priced near the market it cites (Check.et Pro 499 ETB a month, BirrVerify 1,500 ETB per 3 months). All amounts are ETB before VAT.

| Code | Name | Fee per month | Included | Metered channels | Hard limit | Overage per verification | Image credits | Image overage | Platform review | Rate limit | Review SLA |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `sandbox` | Sandbox | 0.00 | 50 | api, portal, cashier | 100% | none | not metered | none | 0.00 | 10 per min | 240 min |
| `starter` | Starter | 990.00 | 1,000 | api, portal, cashier | 120% | 1.50 | 100 | 3.00 | 15.00 | 60 per min | 240 min |
| `growth` | Growth | 7,900.00 | 10,000 | api, portal, cashier | 120% | 1.00 | 1,000 | 2.50 | 10.00 | 300 per min | 30 min |
| `ent_<slug>` | Enterprise (private, per deal) | Per contract | Per contract | Per contract | Per contract (null allowed) | Per contract | Per contract | Per contract | Per contract | Default 1,000 per min | 30 min |

- The architecture doc's proposal made portal checks unlimited on Starter. Accepted rule R12 counts every channel instead, so the seeds meter all three. `metered_channels` keeps the option open: setting a plan to `api` only brings back unlimited portal and cashier checks without code changes (12-Q1). Per-channel counts are recorded either way.
- Test env is never metered on any plan; doc 03's rate limit is its only limit.

### 6.3 Metering

- `metering.handler.ts` (consumer `billing.meter`) processes events in env `live` only. For each usage event, in one transaction:
  1. insert into `usage_dedupe` with `ON CONFLICT DO NOTHING`; a conflict means "already counted", so stop;
  2. insert into `usage_events`;
  3. upsert `usage_counters` (+1 for the meter, +1 for `metered` if the channel is metered by the plan at that moment).
- After commit, increment the Redis counters `pl:usage:{tenantId}:{periodKey}:{meter}` (TTL 70 days) and evaluate thresholds (6.4).
- `channel` comes from `verification.finalized.channel`. `image_credit` has no channel. `platform_review` takes the verification's channel, but it is never quota-metered.
- **Plan changes mid-period:** the metered flag is decided when each event arrives, so a later change to `metered_channels` applies only to new events (open question 12-Q6).

### 6.4 Quota and thresholds

```text
plan    = getEffectivePlan(tenant)
if env = test                                → allow, state ok
if subscription.status = suspended           → block (reason unpaid)
if channel ∉ plan.meteredChannels            → allow, state from the metered count
if plan.includedVerifications is null        → allow, state ok
used    = metered counter (Redis; on miss or Redis error, the DB counter)
limit   = plan.hardLimitPct is null ? ∞ : floor(included × hardLimitPct / 100)
if used ≥ limit                              → block (reason quota)
state   = used ≥ included ? 'over' : used ≥ 0.8 × included ? 'warning' : 'ok'
```

- The check runs before the verification exists, and counting happens at finalisation. In-flight verifications can therefore finish slightly above the limit: overshoot is at most the tenant's in-flight live verifications at that moment. This is accepted and billed as overage.
- `thresholds.ts` returns the levels (80, 100, 120) crossed between the old and new metered count. For each level not yet in `threshold_notices`, the service inserts the row and emits `usage.threshold_reached`. Levels above the plan's hard limit aren't sent; Sandbox sends 80 and 100 only.
- **Redis unavailable:** `check` reads `usage_counters` from Postgres (under 5 ms). If both are unavailable, it throws `503 service_unavailable`; it never allows silently.

### 6.5 Plan changes and proration (12b)

| Change | Effective | Billing effect |
| --- | --- | --- |
| Upgrade (higher fee) | Now | On the next invoice: `proration_credit` of `−old_fee × remaining_days / days_in_month`, and `proration_charge` of `new_fee × remaining_days / days_in_month`. `remaining_days` counts from the change's EAT day, inclusive. Each line is rounded half-up to the santim. The new included quota applies to the whole month |
| Downgrade | Next period start | None; the new plan applies from the 1st |
| Admin change, `now` | Now | Same as an upgrade if the fee rises; no proration when it falls (credit by hand if needed) |
| Sandbox to paid | Now | Only `proration_charge` (Sandbox's fee is 0) |

### 6.6 Invoice building (12b)

The `bill:close:{period}` job runs at 00:10 EAT on the 1st. For each subscription whose plan has a fee above 0, or that had chargeable usage in the closing month:

1. Lines:
   - `base_fee`: the current plan's fee for the new month (none if the subscription is `cancelled`);
   - proration lines from `subscription_changes` with `proration_done = false` (then set it to true);
   - `verification_overage`: `max(0, metered_used − included) × overage price` for the closed month, using the plan in effect at month end;
   - `image_overage`, computed the same way from image credits;
   - `platform_review`: count × fee;
   - `credit`: unapplied credit notes, up to the subtotal.
2. `subtotal = Σ amount_minor`, which can't go below 0 after credits. `vat = round_half_up(subtotal × 0.15)`, and `total = subtotal + vat`.
3. `bill_to` is a snapshot from `TenancyApi.getTenant` (legal name, TIN) and `billing.profiles`.
4. Issue in one transaction: take the next number from `invoice_number_counters` (`SELECT … FOR UPDATE` on the year row, which keeps the series gapless), set `issued_at`, and set `due_at` to issue plus 10 days at 23:59 EAT. Emit `invoice.issued`.
5. After commit, render the PDF (`pdf.service.ts`, pdfmake with embedded IBM Plex Sans and Noto Sans Ethiopic), store it, and record its sha256. The job is idempotent: `invoices_one_per_period` prevents doubles, and a rerun only completes missing PDFs.

Usage counts for invoices come from `usage_events`, never from counters (architecture doc).

### 6.7 Dunning (12b)

| Day relative to `due_at` | Action |
| --- | --- |
| −3, 0, +3 | Reminder email (skipped once paid) |
| +1 | Subscription `active` → `past_due`; amber portal banner "Invoice INV-… is overdue" |
| +14 | `past_due` → `suspended`; quota blocks every live channel (`unpaid`); red banner; email |
| Payment confirmed and no other overdue invoice | Back to `active` immediately; `subscription.status_changed` |

`dunning.processor.ts` runs daily at 08:00 EAT (`bill:dunning:{date}`). Staff can extend a due date with a reason (audited), which recomputes the schedule.

### 6.8 Payment states (12b)

| From | To | Trigger | Side effects |
| --- | --- | --- | --- |
| (none) | `verifying` | Tenant submits a reference or receipt | Unique reference check; job `bill:pay:{id}:1` |
| `verifying` | `confirmed` | House verification `verified` | `amount_minor` = credited; apply to the invoice; any excess becomes a credit note (`source = overpayment`); invoice `paid` when fully covered; `invoice.paid` |
| `verifying` | `pending_review` | House verification `in_review` or `receipt_required` | Polling continues |
| `pending_review` | `confirmed` or `rejected` | Final house status | As above, or `reject_reason` = the reason code |
| `verifying` | `rejected` | House verification `rejected` | `reject_reason`; email to the submitter |
| `verifying`, `pending_review` | `failed` | House API errors for 24 h, or the house verification `expired` or `cancelled` | Email; the tenant can try again |
| (none) | `confirmed` | Admin "Mark paid" without API verification | `method = manual`; audited |

An invoice is `paid` when `amount_paid + credit_applied ≥ total`. Payments below the amount due are rejected by the house verification's amount check (R6), so partial payments don't exist. The pay dialog says so.

### 6.9 Verifying payments through our own API (12b)

- A **house tenant** (a normal tenant, "Proofline billing", on a private Enterprise plan with no hard limit) owns verified receiving accounts for our pay-to accounts (doc 02) and one live API client with scopes `verifications:write`, `verifications:read`, `receipts:write`.
  - The key is stored in OpenBao (`BILLING_HOUSE_API_KEY_PATH`).
  - The key's IP allowlist is the cluster's egress range.
- `house-api.client.ts` calls `BILLING_HOUSE_API_BASE_URL` (the in-cluster `api` service URL) with:
  - `POST /v1/verifications`;
  - `Idempotency-Key: pay_{paymentId}`;
  - `expected.amount` = the amount due;
  - `external_id` = the invoice number;
  - `metadata = { tenant_id, invoice_id, payment_id }`;
  - `wait_seconds = 15`;
  - and either the reference or the receipt file.
- `payment-verify.processor.ts` polls `GET /v1/verifications/{id}`: every 30 s for 10 minutes, then every 5 minutes until final or 72 h.
- **Why HTTP and not an import:** billing must not import `@proofline/verification`, because verification already depends on billing (`QuotaService`). Treating billing as an external client of the public API keeps the code DAG acyclic, and uses exactly the contract and checks merchants get: receiver binding, global duplicate claims, review. This is the only place a module calls the platform over HTTP, and the overview records it as an exception.
- Pay-to accounts shown to tenants come from `BILLING_PAY_TO_JSON` and must match the house tenant's verified receiving accounts. A boot check compares their last 4 digits through the house API's `GET /v1/receiving-accounts`.

### 6.10 Numbers

| Rule | Value | Where |
| --- | --- | --- |
| VAT | 15%, on the subtotal, half-up to the santim | `BILLING_VAT_RATE_BP` = 1500 |
| Due | 10 days after issue | `BILLING_DUE_DAYS` |
| Suspension | 14 days after due | `BILLING_SUSPEND_AFTER_DAYS` |
| Reminders | −3, 0, +3 days | `BILLING_REMINDER_OFFSETS_DAYS` |
| Threshold notices | 80, 100, 120% | Constants (R12) |
| Counter reconciliation | Hourly | `bill:reconcile` |
| Payment polling | 30 s for 10 min, then 5 min, up to 72 h | `BILLING_PAY_POLL_*` |
| Plan cache | 60 s | `BILLING_PLAN_CACHE_SECONDS` |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Redis`, `Queues` (`billing`), `Outbox`, `OnEvent`, `ObjectStore` (`invoices`), `AuditLog`, `Notify`, `Crypto` (none stored; the house key comes from OpenBao), `Clock`, `money`, `i18n` (Ethiopian dates on PDFs) |
| 01 tenancy | `getTenant` (legal name, TIN, status), `listMemberEmails(tenantId, ['owner'])`, `getSettings` (`notifications.alert_emails`); events `tenant.created`, `tenant.status_changed` |
| 05 verification (events only) | `verification.finalized` (`billable`, `channel`, `reviewed_by`) |
| 06 receipts (events only) | `receipt.extracted` (candidate methods) |
| Our public API (12b) | `POST /v1/verifications`, `GET /v1/verifications/{id}`, `GET /v1/receiving-accounts`, with the house key |

Called by: 03 (`PlansApi`), 05 (`QuotaService`), 07 (`PlansApi`), 09 and 11 (`UsageApi`, `PlansApi`).

## 8. Implementation plan

**12a** (build step 2)

1. **T12.1** Migrations `0001`–`0003`, RLS, plan seed; `period.ts` with EAT month edges, including 23:59:59 EAT on the last day.
2. **T12.2** Subscriptions: Sandbox on `tenant.created`, admin plan change (now or next period), changes history, `subscription.changed`.
3. **T12.3** Metering handler with dedupe, counters (DB and Redis), threshold notices and emails.
4. **T12.4** `QuotaService` (the 6.4 algorithm, including the Redis fallback) and `PlansApi` with caching. A contract test is published in the package's `testing` export for docs 03, 05 and 07.
5. **T12.5** `UsageApi.currentPeriod` and `daily`; hourly reconciliation (Redis set from DB counts, DB counts checked against `usage_events`, any difference logged and alerted).
6. **T12.6** Portal `GET /usage`, Billing & usage page (plan card, meters, daily chart), admin tenant Plan & usage tab (plan and usage). EN/AM.

**12b** (build step 9, after the pilot)

7. **T12.7** Migration `0004`; invoice builder (pure, tests for every line kind, proration examples and VAT rounding); gapless numbering.
8. **T12.8** `bill:close` job, PDF rendering (bilingual, both calendars), invoice emails; portal invoices list and detail.
9. **T12.9** House tenant setup script (`pnpm --filter @proofline/worker run billing:setup-house`), house API client, payment submission, polling processor, overpayment credit notes; portal Pay dialog.
10. **T12.10** Dunning job, suspension through quota, reactivation; banners in the portal shell (doc 09 reads `blockedReason`).
11. **T12.11** Self-service Change plan with proration preview; scheduled downgrades.
12. **T12.12** Admin Billing: invoices, mark paid, void, credit notes, payments awaiting confirmation, plan catalogue with versions and migration.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| `verification.finalized` delivered twice | `usage_dedupe` conflict; nothing changes |
| A finalized event arrives after month end for a verification finalised at 23:59:58 EAT | Attributed to the month of `occurredAt` (the old month). If that month's invoice has already been issued, the usage appears as an `adjustment` line on the next invoice |
| Redis loses counters | Quota reads the DB counters until the hourly reconcile, or a cache miss, refills Redis |
| A tenant upgrades at 120% (blocked) | The new plan's limit applies at once; the plan cache is dropped on `subscription.changed` |
| A tenant downgrades below its current usage | Effective next period, so it can't block the current month |
| Reversed verification | Stays billable (doc 05: `billable` is never cleared). The reversal is our recheck working, not a failed outcome |
| A tenant pays with a reference already used for a merchant's order | The house verification rejects it as `duplicate` (global claims); the payment is `rejected` |
| A tenant pays the wrong amount (less) | `amount_mismatch`, so the payment is `rejected`; the dialog explains the minimum |
| The house API is down | The payment stays `verifying` and polling retries; after 24 h of errors it is `failed` and the tenant is told |
| Two invoices overdue | Reactivation needs all overdue invoices paid |
| Tenant closed mid-month | The final invoice is issued at once, with arrears to date and no next-month fee |
| A plan version edited while in use | Not possible: plans are immutable once referenced. A new version is created instead |
| Invoice PDF rendering fails | The invoice is still issued (the number is used, the data is final). The job retries the PDF, the portal shows "PDF being prepared", and admins get an alert after 3 failures |
| Clock or timezone mistakes at period edges | `period.ts` is tested at 2026-10-31T23:59:59 EAT, 2026-11-01T00:00:00 EAT and the equivalent UTC instants |

## 10. Security and privacy

| Route | AuthN | AuthZ |
| --- | --- | --- |
| `/portal/v1/usage` | Portal session | Any member (counts only) |
| `/portal/v1/billing/*` | Portal session plus CSRF | `billing.read`, `billing.manage`; re-auth for plan changes |
| `/admin/v1/billing/*`, tenant plan and usage | Admin session | `platform.billing.*`, `platform.tenants.read`; re-auth for plan creation, migration and credit notes |

- Billing holds no payer PII. Usage events carry IDs and channels only. Payment references belong to the tenant's own payment to us.
- **The house API key** lives in OpenBao. It is never logged and is scoped by IP to the cluster egress.
- **Invoices** are immutable after issue. Corrections are credit notes, both with gapless numbers. PDFs are hashed, and a regenerated PDF must match its stored data.
- **Audited:** plan changes, plan catalogue changes, migrations, mark paid, void, credit notes, due-date extensions.
- **Finance staff** see billing data but no verification content (UI spec role "Finance"). The billing screens never link to verification details.

## 11. Performance

| Item | Target |
| --- | --- |
| `QuotaService.check` | p95 under 3 ms (Redis); under 10 ms on the DB fallback |
| `PlansApi.getEffectivePlan` | p95 under 1 ms (in-process cache 60 s) |
| Metering lag | Event to counter under 5 s at p95 |
| `UsageApi.currentPeriod` | p95 under 20 ms (cached 60 s) |
| `bill:close` | 5,000 tenants in under 10 minutes, including PDFs (rendering 4 in parallel) |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | `period.ts` edges; quota algorithm table (each plan, channel, env, suspension, missing Redis); thresholds crossing several levels at once; invoice builder: base fee, each overage, proration examples (upgrade on 15 October: 17 remaining days of 31), credits capped at the subtotal, VAT rounding (subtotal 8,210.00 gives VAT 1,231.50; subtotal 0.03 gives VAT 0.00, half-up of 0.0045); dunning schedule |
| Integration | Metering idempotency under concurrent duplicate deliveries; counters equal `usage_events` after a forced Redis flush and a reconcile; Growth 12,000th API check allowed and the next blocked while a portal check passes; Sandbox blocked at 50 across channels; the invoice job twice in a row yields one invoice per tenant, with gapless numbers under 20 parallel issuers; PDF bytes stored with a matching hash |
| Payment flow | Against a running api with the sandbox connector and a house tenant in test: `TEST_VERIFIED` confirms and pays; `TEST_AMOUNT_MISMATCH` rejects; `TEST_REVIEW_APPROVE` passes through `pending_review` to confirmed; a reference used twice gives `payment_reference_used` |
| End to end | Owner sees the meter and the Billing page; upgrades Sandbox to Starter with re-auth; the meter shows the new included amount; admin issues a credit note; a monthly invoice appears with the credit applied |
| Amharic | Billing page, emails and invoice PDF labels in `am`, with Ethiopian dates on the PDF |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_billing_quota_checks_total` | counter | `decision`, `reason`, `source` (redis, db) |
| `pl_billing_usage_events_total` | counter | `kind`, `channel` |
| `pl_billing_metering_lag_seconds` | histogram | none |
| `pl_billing_counter_drift_total` | counter | none (reconcile differences) |
| `pl_billing_invoices_total` | counter | `status` |
| `pl_billing_payments_total` | counter | `status`, `method` |
| `pl_billing_tenants_blocked` | gauge | `reason` (quota, unpaid) |

| Alert | Condition | Severity |
| --- | --- | --- |
| Counter drift | Any drift found by the reconcile | Warning |
| Quota DB fallback | More than 5% of checks on the DB path for 10 min | Warning |
| Invoice job | `bill:close` not finished by 02:00 EAT on the 1st | Critical |
| PDF failures | 3 or more for one invoice | Warning |
| House API | 5 consecutive errors from the house client | Critical |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `BILLING_VAT_RATE_BP` | 1500 | VAT in basis points |
| `BILLING_DUE_DAYS` | 10 | Invoice terms |
| `BILLING_SUSPEND_AFTER_DAYS` | 14 | Days after due before suspension |
| `BILLING_REMINDER_OFFSETS_DAYS` | `-3,0,3` | Reminder days |
| `BILLING_PLAN_CACHE_SECONDS` | 60 | Plan cache |
| `BILLING_HOUSE_API_BASE_URL` | none (required in 12b) | In-cluster `api` URL |
| `BILLING_HOUSE_API_KEY_PATH` | none (required in 12b) | OpenBao path of the house key |
| `BILLING_PAY_TO_JSON` | none (required in 12b) | Pay-to accounts shown to tenants |
| `BILLING_PAY_POLL_FAST_SECONDS`, `_FAST_MINUTES`, `_SLOW_SECONDS`, `_MAX_HOURS` | 30, 10, 300, 72 | Payment polling |
| `BILLING_SALES_EMAIL` | none (required) | "Talk to us" link |
| `BILLING_SELLER_JSON` | none (required in 12b) | Our legal name, TIN, VAT number and address for invoices |

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 12-Q1 | Plan list, prices, included checks, overage, image and review fees, and whether 120% is right (overview Q5). Should any plan go back to unlimited portal checks, as the architecture doc proposed? | The seed in 6.2, metering every channel per R12 |
| 12-Q2 | Fiscal requirements for invoices: is a sales register machine or the tax authority's e-invoicing service needed for a SaaS invoice, and which VAT invoice fields are mandatory? | PDF invoices with gapless numbers and full VAT detail. An accountant confirms before the paid-launch gate |
| 12-Q3 | How long must invoices and payment records be kept? | 10 years for invoices, credit notes and payments; usage events 24 months |
| 12-Q4 | Should plans limit receiving accounts, branches or API clients (the architecture doc gives Starter 2 receiving accounts)? | No plan limits at launch; the flat limits in docs 01–03 apply |
| 12-Q5 | How are credit balances refunded when an account closes? | Shown to Finance for manual refund by bank transfer |
| 12-Q6 | If a plan's metered channels change mid-month, should earlier events be re-counted? | No; each event is counted by the rule in force when it arrived |
| 12-Q7 | Should image credits ever block (for example on Starter), or only bill overage? | Overage only; no blocking |
