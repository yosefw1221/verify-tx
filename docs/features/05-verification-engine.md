# 05 · Verification engine

| | |
| --- | --- |
| Package | `@proofline/verification` |
| Postgres schema | `verification` |
| Depends on | platform, 01 tenancy, 02 receiving accounts, 04 bank connectors, 06 receipts, 12 billing (quota) |
| Build step | 5 |
| Mockups | Portal · Verify a payment (result card and checks); Portal · Verifications + detail (status, reason, checks, timeline); Portal · Overview (KPIs, recent checks, rejections by reason); Cashier · Verified / Do not release; Upload · In review; review workspace "Receipt against expectation" and timeline |

## 1. Purpose and scope

**Does**

- The public verification API: create, get, list, cancel.
- The same operations for the portal and cashier app through `VerificationApi`.
- The verification state machine: 8 statuses, reasons and assurance (R1, R2).
- Scheduling every bank attempt: first try, quick retries, not-found rechecks, background retries while the bank is down, retries after a receipt is read, and post-decision rechecks for 72 h.
- The five checks and their precedence (R3, R4, R6, R15).
- The **claim registry**, which makes one bank transaction claimable once per environment across all tenants.
- Idempotency behaviour on create.
- Waiting up to `wait_seconds` for a final answer.
- The verification timeline shown in the portal drawer and the review workspace.
- Quota and live-gate checks on create.
- Reversals.
- Portal statistics.

**Out of scope**

- Talking to banks (doc 04).
- Storing and reading receipts and upload links (doc 06).
- The review queue, claiming and decisions UI (doc 07); this module only applies decisions.
- Webhook delivery (doc 08) and usage counters (doc 12); this module only emits events.

## 2. Screens and user stories

| Screen | What this module provides |
| --- | --- |
| Portal · Verify a payment | `VerificationApi.create(channel: portal, wait 15)`; result card fields and checks |
| Portal · Verifications + detail drawer | `list`, `get`, timeline, cancel |
| Portal · Overview | `stats`, `rejectionReasons`, recent list |
| Cashier · Scan → Verified / Do not release | `create(channel: cashier, wait 15)`; own-scope list |
| Upload · In review | Status read through doc 06's hosted status endpoint |
| Admin · Verifications explorer, review workspace | `getForReview`, `retryBankNow`, `findRelated` |

Acceptance criteria. Each runs in CI against the sandbox (doc 04, section 6.8) unless it says "mock bank".

**US-05.1 Verified**

- `POST /v1/verifications` with `TEST_VERIFIED`, `expected.amount = "1500.00"` and `wait_seconds = 10` returns `200`.
- The body has `status: verified`, `assurance: bank_confirmed`, `reason: null`, every check `pass`, `transaction.amount = "1500.00"` and `livemode: false`.
- A claim exists for the official reference, and exactly one `verification.finalized` event with `billable: false` (test env) is in the outbox.

**US-05.2 Receiver mismatch**

- `TEST_RECEIVER_MISMATCH` returns `200 rejected` with `reason: receiver_mismatch`, `checks.receiver: fail`, `transaction: null` (no payer data, R3), and no claim.

**US-05.3 Duplicate in the same tenant**

- `TEST_DUPLICATE` returns `200 rejected/duplicate`.
- When a live duplicate's first claim belongs to the same tenant, `details.first_claim = { verification_id, external_id, claimed_at }`.
- When the first claim belongs to another tenant, `details.first_claim` is absent; in that case the receiver check has normally failed first anyway (R3).

**US-05.4 Overpayment**

- A mock bank returns credited 1,600.00 against expected 1,500.00. The result is `verified` with `checks.amount: over`.

**US-05.5 Stale**

- `TEST_STALE` returns `rejected/stale`.
- With `expected.paid_after` set to 10 minutes before a mock payment time, the payment passes even though the 24 h default would also pass. With `paid_after` set after the payment time, it fails `stale`.

**US-05.6 Not completed:** `TEST_NOT_COMPLETED` returns `rejected/not_completed`.

**US-05.7 Not found:** `TEST_NOT_FOUND` returns `rejected/not_found` (no rechecks configured).

**US-05.8 Slow bank**

- `TEST_SLOW` with `wait_seconds = 15` returns `202 processing`.
- About 20 s later, the verification is `verified`, and `verification.status_changed` plus `verification.finalized` are in the outbox.

**US-05.9 Bank down**

- `TEST_BANK_DOWN` returns, within `wait_seconds`, `202 receipt_required` with `reason: bank_unavailable` and `next_action = { type: "upload_receipt", upload_url, expires_at }`, where `expires_at` is 30 min out.
- Background attempts are scheduled every 5 min.

**US-05.10 Breaker open**

- With telebirr's breaker forced open (live, mock), create returns `202 receipt_required/bank_unavailable` with zero bank calls.

**US-05.11 Insufficient reference**

- A bare CBE `FT…` reference with `bank_code = CBE` returns `202 receipt_required` with `reason: reference_insufficient`.

**US-05.12 Receipt first**

- A multipart create with a receipt file and no reference returns `202 processing`.
- After `receipt.extracted`, an attempt runs and the final status follows the bank's answer.

**US-05.13 Receipt in, bank still down**

- `TEST_REVIEW_APPROVE` with an uploaded receipt leads to `in_review` with `review_reason: bank_unavailable`, then `verification.review_required`.
- The sandbox reviewer approves, leading to `verified` with `assurance: manual_review`, and post-decision rechecks are scheduled for 72 h.

**US-05.14 Force review**

- With tenant setting `verification.force_review_above = "10000.00"`, a bank-confirmed 12,500.00 payment that passes all checks becomes `in_review` with `review_reason: tenant_policy_amount`. The claim is held.
- An approval gives `verified/bank_confirmed`.

**US-05.15 Receiver unverifiable**

- A mock bank answer with no receiver details leads to `in_review` with `review_reason: receiver_unverifiable`.

**US-05.16 Reversal**

- A manually approved verification whose recheck (mock bank) finds `receiver_mismatch` becomes `reversed` with `reason: bank_contradicted` and `details.failed_checks: ["receiver"]`, and `verification.reversed` is emitted.

**US-05.17 Upgrade**

- A manually approved verification whose recheck passes all checks keeps `status: verified`, changes `assurance` to `bank_confirmed`, and adds the timeline entry `assurance_upgraded`.

**US-05.18 Cancel**

- `POST …/cancel` in `processing`, `receipt_required` or `in_review` gives `cancelled`, invalidates upload links, releases claims, and cancels timers.
- On a final verification it returns `409 invalid_state`.

**US-05.19 Expiry**

- `receipt_required` with no upload for 24 h (`FakeClock`) becomes `expired` with `reason: receipt_not_uploaded`.
- `in_review` undecided for 24 h becomes `expired` with `reason: review_timed_out`.

**US-05.20 Idempotency**

- The same `Idempotency-Key` and the same body replays the first response with `Idempotent-Replayed: true` and creates no second verification.
- A different body returns `409 idempotency_conflict`.

**US-05.21 Quota:** a live tenant over the hard limit gets `402 quota_exceeded`.

**US-05.22 No account:** a live tenant with no verified BoA account sending `bank_code = BOA` gets `422 no_receiving_account`.

**US-05.23 Race**

- Two cashiers verify the same telebirr reference at the same instant (mock bank answers `found` to both). Exactly one ends `verified`, the other ends `rejected/duplicate`, and exactly one active claim exists.

**US-05.24 Bank recovers**

- Ten verifications wait in `receipt_required/bank_unavailable` for telebirr. When `bank.breaker_changed` reports `to: closed`, all ten are retried within 60 s (jittered).

**US-05.25 List**

- The portal list filters by status, bank, assurance, channel, branch, date range, external ID, reference and exact payer name, with cursor paging, newest first.
- A Cashier gets only their own verifications.

## 3. Module design

```text
libs/features/verification/src/
├─ public/
│  ├─ index.ts                       VerificationModule, VerificationApi, views, events, enums
│  ├─ verification-api.ts
│  ├─ enums.ts                       re-exports from libs/contracts: Status, RejectReason, ReversedReason, ReceiptRequiredReason, ReviewReason, ExpiredReason, Assurance, CheckValue
│  └─ events.ts
├─ domain/
│  ├─ state-machine.ts               TRANSITIONS table plus assertTransition()
│  ├─ decide.ts                      decide(bankResult, ctx) → Decision (pure, section 6.3)
│  ├─ checks.ts                      runChecks(receipt, ctx) → CheckResults (pure)
│  ├─ precedence.ts                  primaryReason(checks)
│  ├─ date-window.ts                 R4
│  ├─ schedule.ts                    next-attempt policy (pure, section 6.4)
│  └─ public-mapper.ts               row → public JSON, masking rules (section 6.9)
├─ application/
│  ├─ create.service.ts              create pipeline (section 6.1)
│  ├─ attempt.processor.ts           BullMQ processors bank-fetch.{bank}
│  ├─ apply-result.service.ts        lock row → decide → transition → claims → outbox
│  ├─ receipt-events.handler.ts      @OnEvent receipt.uploaded / .extracted / .extraction_failed
│  ├─ breaker.handler.ts             @OnEvent bank.breaker_changed
│  ├─ tenant-created.handler.ts      seeds sandbox duplicate claims
│  ├─ review-bridge.service.ts       applyReviewDecision, requestNewReceipt
│  ├─ timers.processor.ts            queue verification-timers
│  ├─ cancel.service.ts
│  ├─ query.service.ts               get, list, stats, findRelated
│  └─ waiter.ts                      wait for final up to N s (Redis pubsub plus re-reads)
├─ infrastructure/
│  ├─ verifications.repo.ts, attempts.repo.ts, claims.repo.ts, timeline.repo.ts
│  └─ pubsub.ts                      publish ver:{id} after commit
├─ http/
│  ├─ public-api/verifications.controller.ts
│  ├─ portal/verifications.controller.ts
│  └─ admin/verifications.controller.ts
└─ db/schema.ts, db/migrations/
```

### Public interface

```ts
export type VerificationStatus =
  'processing' | 'verified' | 'rejected' | 'receipt_required' | 'in_review' | 'expired' | 'reversed' | 'cancelled';
export type RejectReason =
  'duplicate' | 'receiver_mismatch' | 'not_found' | 'amount_mismatch' | 'stale' | 'not_completed'
  | 'suspected_forgery' | 'unreadable_receipt';
export type ReceiptRequiredReason = 'bank_unavailable' | 'reference_insufficient' | 'bank_format_changed' | 'image_requested';
export type ReviewReason = 'bank_unavailable' | 'bank_format_changed' | 'no_reference_found' | 'reference_unconfirmed'
  | 'receiver_unverifiable' | 'tenant_policy_amount';
export type ExpiredReason = 'receipt_not_uploaded' | 'review_timed_out';
export type ReversedReason = 'bank_contradicted';
export type Assurance = 'bank_confirmed' | 'manual_review';
export type Channel = 'api' | 'portal' | 'cashier';
export type CheckValue = 'pass' | 'fail' | 'over' | 'not_run' | 'unverifiable' | 'not_available';

export interface VerificationApi {
  create(input: CreateVerificationInput, opts: { waitSeconds: number }): Promise<{ verification: VerificationView; final: boolean }>;
  get(id: string, scope: DataScope): Promise<VerificationView>;                       // 404 outside scope
  getPublicJson(id: string, scope: DataScope): Promise<PublicVerificationJson>;       // exactly the GET /v1/verifications/{id} body (doc 08 payloads, portal raw panel)
  list(q: VerificationListQuery, scope: DataScope): Promise<Page<VerificationView>>;
  count(q: VerificationListQuery, scope: DataScope, cap: number): Promise<{ count: number; capped: boolean }>;  // range at most 92 days (doc 09 log totals use at most 31; the export guard up to 92)
  timeline(id: string, scope: DataScope): Promise<TimelineEntry[]>;
  cancel(id: string, scope: DataScope): Promise<VerificationView>;
  addReceipt(id: string, input: { file: IncomingFile } | { shareLink: string },      // merchant-side receipt (API, portal, cashier app)
             ctx: { scope: DataScope; actor: Actor; source: 'api' | 'portal' | 'cashier' }): Promise<VerificationView>;
             // stages through ReceiptsApi.stage, then in one transaction locks the row, requires status receipt_required
             // (else 409 invalid_state), and calls attachStaged(tx) or attachShareLink(tx); doc 06 enqueues extraction

  // doc 07 (review)
  getForReview(id: string): Promise<VerificationReviewView>;      // unmasked payer phone, attempts, bank health at request time, and
                                                                  // signals.reference_seen_elsewhere (claim registry, R: brief "references
                                                                  // already used at other merchants"); staff or tenant reviewer only
  applyReviewDecision(id: string, d: ReviewDecisionInput): Promise<VerificationView>;
  requestNewReceipt(id: string, d: { taskId: string; actor: Actor }): Promise<VerificationView>;
  findRelated(id: string): Promise<RelatedSummary>;

  // docs 09 and 11 (payer details behind MaskedValue)
  revealPayer(id: string, scope: DataScope, actor: Actor): Promise<{ payerName: string | null; payerPhone: string | null; payerAccount: string | null }>;
                                                                  // needs pii.reveal (tenant) or platform.pii.reveal (staff) plus a re-auth in the
                                                                  // last 5 min; audited pii.revealed; empty when the receiver check didn't pass (R3)

  // doc 11 (admin)
  retryBankNow(id: string, actor: Actor): Promise<void>;
  platformStats(range: { from: Date; to: Date }): Promise<PlatformStats>;            // live only, all tenants, replica; cohort = created in range
  platformSearch(q: string): Promise<VerificationView[]>;                            // reference or external_id, last 90 days, at most 10, masked payer

  // doc 09 (portal overview)
  stats(tenantId: string, env: EnvName, range: { from: Date; to: Date }): Promise<VerificationStats>;
  rejectionReasons(tenantId: string, env: EnvName, range: { from: Date; to: Date }): Promise<{ reason: RejectReason; count: number }[]>;
}

export interface CreateVerificationInput {
  tenantId: string; env: EnvName; channel: Channel;
  actor: { kind: 'api_client'; apiClientId: string } | { kind: 'user'; userId: string; branchId: string | null };
  idempotencyKey: string;
  bankCode?: BankCode;
  reference?: string;                    // bare reference, share link or SMS text (at most 2,048 chars)
  payerPhone?: string;
  receipt?: IncomingFile;                // exactly one of reference or receipt
  expected: { amount: Money; receivingAccountId?: string; paidAfter?: Date };
  externalId?: string;
  metadata?: Record<string, string>;
}
export interface ReviewDecisionInput {
  taskId: string;
  decision: 'approve' | 'reject';
  reason?: Exclude<RejectReason, 'not_found'>;         // required for reject
  reviewedBy: 'platform' | 'tenant';
  actor: Actor;                                         // first approver
  secondApprover?: Actor;                               // present when required (doc 07)
}
export interface PlatformStats {                        // doc 11 dashboard
  created: number;
  outcomes: { verifiedBank: number; verifiedManual: number; rejected: number; reversed: number; expired: number; cancelled: number };
  open: { processing: number; receiptRequired: number; inReview: number };
  bankConfirmedRatio: number | null;
  topRejectReason: { reason: RejectReason; count: number } | null;
  buckets: { start: Date; count: number }[];          // hourly when the range is at most 1 day, else daily (EAT)
}
```

`VerificationView` is the public JSON shape in section 5, as a typed object. `PublicVerificationJson` is the zod-inferred wire type from `libs/contracts`, produced by `public-mapper.ts`. `platformStats` and `platformSearch` run only for a `staff` principal through `pl_admin`'s cross-tenant `SELECT`, on the replica.

## 4. Data model

```sql
CREATE TABLE verification.verifications (
  id                         uuid NOT NULL,
  created_at                 timestamptz NOT NULL,         -- = ULID timestamp of id (exact, ms)
  tenant_id                  uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                        text NOT NULL CHECK (env IN ('live','test')),
  channel                    text NOT NULL CHECK (channel IN ('api','portal','cashier')),
  api_client_id              uuid NULL,
  created_by_user_id         uuid NULL,
  branch_id                  uuid NULL,
  external_id                text NULL CHECK (length(external_id) <= 128),
  metadata                   jsonb NOT NULL DEFAULT '{}'::jsonb,
  bank_code                  text NULL,                    -- null only while a receipt-first verification awaits extraction
  input_kind                 text NOT NULL CHECK (input_kind IN ('reference','share_link','sms_text','qr_payload','receipt')),
  input_reference            text NULL CHECK (length(input_reference) <= 128),   -- displayReference
  lookup                     jsonb NULL,                   -- CanonicalBankInput without payerPhone
  payer_phone_input_enc      text NULL,
  expected_amount_minor      bigint NOT NULL CHECK (expected_amount_minor > 0),
  expected_receiving_account_id uuid NULL,
  expected_paid_after        timestamptz NULL,
  date_window_start          timestamptz NOT NULL,
  policy                     jsonb NOT NULL,              -- snapshot: {force_review_above_minor, receipt_wait_hours, review_max_age_hours}
  status                     text NOT NULL CHECK (status IN ('processing','verified','rejected','receipt_required','in_review','expired','reversed','cancelled')),
  reason                     text NULL,
  assurance                  text NULL CHECK (assurance IN ('bank_confirmed','manual_review')),
  review_reason              text NULL,
  reviewed_by                text NULL CHECK (reviewed_by IN ('platform','tenant')),
  official_reference         text NULL,
  credited_amount_minor      bigint NULL,
  fee_minor                  bigint NULL,
  total_minor                bigint NULL,
  paid_at                    timestamptz NULL,
  bank_status                text NULL,
  payer_name_enc             text NULL,
  payer_name_hmac            bytea NULL,                  -- HMAC of normalised name, for exact search
  payer_phone_enc            text NULL,
  payer_phone_hmac           bytea NULL,
  payer_phone_masked         text NULL,
  payer_account_masked       text NULL,
  receiver_name              text NULL,
  receiver_account_masked    text NULL,
  matched_receiving_account_id uuid NULL,
  receiver_match_basis       text NULL,
  narrative                  text NULL CHECK (length(narrative) <= 200),
  checks                     jsonb NOT NULL,              -- {receiver,duplicate,bank_status,amount,date_window}: CheckValue
  details                    jsonb NOT NULL DEFAULT '{}'::jsonb,   -- {first_claim?, failed_checks?}
  evidence_source            text NULL CHECK (evidence_source IN ('bank','receipt','bank_and_receipt')),
  last_evidence_id           uuid NULL,
  receipt_id                 uuid NULL,
  attempt_count              smallint NOT NULL DEFAULT 0,
  next_attempt_at            timestamptz NULL,             -- set whenever an attempt is scheduled; read by the sweeper
  transient_streak           smallint NOT NULL DEFAULT 0,
  pending_streak             smallint NOT NULL DEFAULT 0,
  not_found_streak           smallint NOT NULL DEFAULT 0,
  not_found_first_at         timestamptz NULL,
  last_failure_class         text NULL,
  receipt_deadline_at        timestamptz NULL,
  review_deadline_at         timestamptz NULL,
  recheck_until              timestamptz NULL,
  billable                   boolean NOT NULL DEFAULT false,
  finalized_at               timestamptz NULL,
  updated_at                 timestamptz NOT NULL,
  version                    integer NOT NULL DEFAULT 0,
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX v_list      ON verification.verifications (tenant_id, env, created_at DESC, id DESC);
CREATE INDEX v_status    ON verification.verifications (tenant_id, env, status, created_at DESC);
CREATE INDEX v_external  ON verification.verifications (tenant_id, env, external_id) WHERE external_id IS NOT NULL;
CREATE INDEX v_inputref  ON verification.verifications (tenant_id, env, input_reference);
CREATE INDEX v_official  ON verification.verifications (tenant_id, env, official_reference) WHERE official_reference IS NOT NULL;
CREATE INDEX v_own       ON verification.verifications (tenant_id, env, created_by_user_id, created_at DESC) WHERE channel = 'cashier';
CREATE INDEX v_payer     ON verification.verifications (tenant_id, env, payer_name_hmac) WHERE payer_name_hmac IS NOT NULL;
CREATE INDEX v_waiting   ON verification.verifications (bank_code, status) WHERE status IN ('processing','receipt_required','in_review');
CREATE INDEX v_recheck   ON verification.verifications (recheck_until) WHERE recheck_until IS NOT NULL;
CREATE INDEX v_due       ON verification.verifications (next_attempt_at) WHERE next_attempt_at IS NOT NULL;

CREATE TABLE verification.attempts (
  id                       uuid NOT NULL,
  created_at               timestamptz NOT NULL,
  verification_id          uuid NOT NULL,
  verification_created_at  timestamptz NOT NULL,
  tenant_id uuid NOT NULL, env text NOT NULL,
  attempt_no               smallint NOT NULL,
  trigger                  text NOT NULL CHECK (trigger IN ('create','retry_transient','not_found_recheck','pending_recheck','receipt_candidate',
                              'background_retry','bank_recovered','post_decision_recheck','manual_retry')),
  lookup_ref               text NOT NULL,
  outcome                  text NULL CHECK (outcome IN ('found','not_found','failure')),
  failure_class            text NULL,
  node_id                  text NULL,
  latency_ms               integer NULL,
  evidence_id              uuid NULL,
  parser_version           text NULL,
  completed_at             timestamptz NULL,
  PRIMARY KEY (id, created_at),
  UNIQUE (verification_id, attempt_no, created_at)
) PARTITION BY RANGE (created_at);
CREATE INDEX attempts_verification ON verification.attempts (verification_id, attempt_no);

CREATE TABLE verification.claims (
  id                       uuid PRIMARY KEY,
  env                      text NOT NULL,
  bank_code                text NOT NULL,
  official_reference       text NOT NULL,
  reference_hmac           bytea NOT NULL,      -- Crypto.hmac('reference', bank_code||':'||official_reference)
  tenant_id                uuid NOT NULL,
  verification_id          uuid NULL,           -- null for sandbox seeds
  verification_created_at  timestamptz NULL,
  external_id              text NULL,
  basis                    text NOT NULL CHECK (basis IN ('bank','receipt_provisional','sandbox_seed')),
  status                   text NOT NULL CHECK (status IN ('active','released')),
  claimed_at               timestamptz NOT NULL DEFAULT now(),
  released_at              timestamptz NULL,
  release_reason           text NULL
);
CREATE UNIQUE INDEX claims_active_uq ON verification.claims (env, bank_code, official_reference) WHERE status = 'active';
CREATE INDEX claims_hmac ON verification.claims (env, reference_hmac) WHERE status = 'active';
CREATE INDEX claims_verification ON verification.claims (verification_id);

CREATE TABLE verification.timeline (
  id uuid NOT NULL, at timestamptz NOT NULL,
  verification_id uuid NOT NULL, tenant_id uuid NOT NULL, env text NOT NULL,
  kind text NOT NULL CHECK (kind IN ('created','attempt_started','bank_answered','bank_failed','status_changed','receipt_uploaded',
          'receipt_extracted','upload_link_issued','review_required','review_decided','assurance_upgraded',
          'bank_confirmed_after_rejection','cancelled','expired','reversed','claim_conflict')),
  data jsonb NOT NULL DEFAULT '{}'::jsonb,           -- no PII
  PRIMARY KEY (id, at)
) PARTITION BY RANGE (at);
CREATE INDEX timeline_verification ON verification.timeline (verification_id, at);
```

Timeline `data` per kind carries IDs and codes only. For `review_decided` it is `{ decision, reason, reviewed_by, actor_user_id }`, where `actor_user_id` is set only for a tenant-stage decision, so the portal can name tenant reviewers and never names platform staff (doc 09, section 6.5).

The claim registry is cross-tenant by design. RLS on `claims` limits `SELECT` to the caller's tenant and env, while the unique index enforces global uniqueness. Two `SECURITY DEFINER` functions, executable by `pl_app`, expose only the minimum:

```sql
CREATE FUNCTION verification.claim_owner(p_env text, p_bank text, p_ref text)
RETURNS TABLE (tenant_id uuid, verification_id uuid, external_id text, claimed_at timestamptz, basis text)
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = verification AS $$
  SELECT tenant_id, verification_id, external_id, claimed_at, basis FROM verification.claims
  WHERE env = p_env AND bank_code = p_bank AND official_reference = p_ref AND status = 'active' $$;

CREATE FUNCTION verification.reference_seen_elsewhere(p_env text, p_hmac bytea, p_tenant uuid)
RETURNS boolean LANGUAGE sql STABLE SECURITY DEFINER SET search_path = verification AS $$
  SELECT EXISTS (SELECT 1 FROM verification.claims WHERE env = p_env AND reference_hmac = p_hmac
                 AND tenant_id <> p_tenant AND status = 'active') $$;
```

Callers never return another tenant's `verification_id` or `external_id`; `public-mapper.ts` strips them unless `tenant_id` matches.

- **Partitions:** monthly for `verifications`, `attempts` and `timeline`, created 3 months ahead by the maintenance job, and never dropped automatically; live retention is open question 05-Q5.
- **Test-env purge:** the maintenance job deletes test-env verifications older than 30 days, with their attempts, timeline and claims, in batches of 1,000 (overview 5.4). Sandbox seed claims are kept.
- **Get by ID:** derive `created_at` from the ULID's timestamp and query `WHERE id = $1 AND created_at = $2`, which prunes to one partition.
- **Migrations:** `0001_verification_init.sql`, `0002_partitions_initial.sql`, `0003_claim_functions.sql`.

## 5. API contracts

### `POST /v1/verifications`

Scope: `verifications:write`, plus `receipts:write` for multipart. The `Idempotency-Key` header is required (1–255 printable ASCII characters).

```json
// JSON body
{
  "bank_code": "TELEBIRR",
  "reference": "CHQ0FJ403O",
  "payer_phone": null,
  "expected": { "amount": "1500.00", "receiving_account_id": "racc_01JA2B3C4D5E6F7G8H9J0K1M2N", "paid_after": null },
  "external_id": "order_88231",
  "metadata": { "branch": "Bole" },
  "wait_seconds": 10
}
```

The multipart form has two parts: `data` (the JSON above, without `reference`) and `file` (the receipt). Validation:

- `reference`: 1–2,048 characters. A bare reference, an allowed share link or an SMS text (doc 04, section 6.2).
- `payer_phone`: `^251[79]\d{8}$`. Required for `CBEBIRR`.
- `expected.amount`: the money regex. At least `0.01`.
- `expected.paid_after`: RFC 3339. Not more than 5 minutes in the future, not more than 30 days in the past.
- `external_id`: at most 128 printable characters.
- `metadata`: at most 20 keys matching `^[a-z0-9_]{1,40}$`; string values of at most 500 characters.
- `wait_seconds`: integer 0–15, default 0.
- Exactly one of `reference` or `file`; otherwise `400 invalid_request`.

Responses: `200` if final (`verified`, `rejected`); `202` otherwise (`processing`, `receipt_required`, `in_review`).

```json
{
  "id": "ver_01JB7Q8K2M4N6P8R0T2V4X6Z8A",
  "livemode": true,
  "status": "verified",
  "assurance": "bank_confirmed",
  "reason": null,
  "review_reason": null,
  "external_id": "order_88231",
  "channel": "api",
  "bank": { "code": "TELEBIRR", "name": "telebirr" },
  "input": { "kind": "reference", "reference": "CHQ0FJ403O" },
  "expected": { "amount": "1500.00", "receiving_account_id": "racc_01JA2B3C4D5E6F7G8H9J0K1M2N", "paid_after": null },
  "transaction": {
    "reference": "CHQ0FJ403O",
    "amount": "1500.00", "fee": "0.00", "total": "1500.00", "currency": "ETB",
    "paid_at": "2026-10-04T10:12:44+03:00",
    "status": "completed",
    "payer": { "name": "ABEBE KEBEDE", "phone_masked": "+251 9•• ••• 234", "account_masked": null },
    "receiver": { "name": "HABESHA MART PLC", "account_masked": "****4401", "receiving_account_id": "racc_01JA2B3C4D5E6F7G8H9J0K1M2N" },
    "narrative": "Order 88231"
  },
  "checks": { "receiver": "pass", "duplicate": "pass", "bank_status": "pass", "amount": "pass", "date_window": "pass" },
  "details": {},
  "next_action": null,
  "evidence": { "source": "bank", "fetched_at": "2026-10-04T10:13:02+03:00" },
  "metadata": { "branch": "Bole" },
  "branch_id": null,
  "created_at": "2026-10-04T10:13:00+03:00",
  "updated_at": "2026-10-04T10:13:02+03:00",
  "completed_at": "2026-10-04T10:13:02+03:00"
}
```

`next_action` for `receipt_required`:

```json
{ "type": "upload_receipt", "upload_url": "https://pay.<domain>/r/9fK2…?lang=en",
  "expires_at": "2026-10-04T10:45:00+03:00", "api_upload_path": "/v1/verifications/ver_01JB…/receipt" }
```

`next_action` is built on every read (create response, `GET`, webhook payloads) from `ReceiptsApi.activeUploadLink(id)` while the status is `receipt_required` and a link is active; otherwise it is `null`. Doc 06 keeps the link token encrypted as well as hashed for exactly this.

`details.first_claim` appears only on a same-tenant duplicate. `details.failed_checks` appears on `rejected` and `reversed`.

| Error | When |
| --- | --- |
| `400 missing_field`, `invalid_field`, `invalid_amount`, `invalid_request` | Validation (`field` names the JSON path) |
| `400 missing_field` (`Idempotency-Key`) | Header absent |
| `402 quota_exceeded` | Live and over the hard limit (R12) |
| `403 live_not_enabled`, `tenant_suspended` | Portal and cashier channels (the API channel is gated in doc 03) |
| `409 idempotency_conflict`, `idempotency_in_progress` | Overview 5.16 |
| `413 file_too_large`, `415 unsupported_file_type` | Multipart (validated by doc 06) |
| `422 unsupported_bank`, `bank_not_detected`, `invalid_reference`, `no_receiving_account` | Bank and receiving-account checks |

### Other public endpoints

| Method and path | Scope | Notes |
| --- | --- | --- |
| `GET /v1/verifications/{id}` | `verifications:read` | `200 Verification`; `404` if not in the key's tenant and env |
| `GET /v1/verifications` | `verifications:read` | Filters `status` (repeatable), `bank_code`, `external_id`, `reference`, `created_gte`, `created_lt`; cursor paging (overview 6.5) |
| `POST /v1/verifications/{id}/cancel` | `verifications:write` | `200 Verification` (status `cancelled`); `409 invalid_state` if final |
| `POST /v1/verifications/{id}/receipt` | `receipts:write` | `addReceipt`: multipart `file` or JSON `{ "share_link" }`; `Idempotency-Key` optional; `202 Verification`; `409 invalid_state` (not `receipt_required`), `413`, `415`, `422 file_rejected` (doc 06's file rules) |
| `POST /v1/verifications/{id}/upload-link` | `verifications:write` | Owned by doc 06 |

### Portal (`/portal/v1`)

| Method and path | Permission | Notes |
| --- | --- | --- |
| `POST /verifications` | `verifications.create` | Same body as public. `wait_seconds` is forced to 15. `X-Idempotency-Key` is required: the web app sends a UUIDv4 per submit |
| `GET /verifications` | `verifications.read` (tenant) or `verifications.read.own` | Extra filters: `assurance`, `channel`, `branch_id`, `payer_name` (exact, normalised), `review_reason`. Saved views (doc 09) are expanded into these filters by the web app; this endpoint knows nothing about them |
| `GET /verifications/{id}` | Same | Adds `created_by` (name) and `api_client` (name, key ID) for the drawer |
| `GET /verifications/{id}/timeline` | Same | `TimelineEntry[]` |
| `POST /verifications/{id}/cancel` | `verifications.cancel` | |
| `GET /verifications/{id}/raw` | `verifications.read.raw` | The exact public API JSON, for the "Raw API response" panel |
| `POST /verifications/{id}/receipt` | `verifications.create` | `addReceipt` with `source: portal` (staff upload the customer's receipt); same errors as the public route |
| `POST /verifications/{id}/reveal` | `pii.reveal`, plus re-auth | `200 { payer_name, payer_phone, payer_account }` unmasked; audited `pii.revealed`; `401 reauth_required` |

### Admin (`/admin/v1`)

| Method and path | Permission | Notes |
| --- | --- | --- |
| `GET /verifications` | `platform.verifications.read` | Cross-tenant search by ID, reference, external ID or tenant; masked payer fields |
| `GET /verifications/{id}` | `platform.verifications.read` | Includes attempts (no evidence bodies) and timeline |
| `POST /verifications/{id}/retry-bank` | `platform.verifications.retry_bank` | `202`; enqueues a `manual_retry` attempt; audited |
| `POST /verifications/{id}/reveal` | `platform.pii.reveal`, plus re-auth (OIDC `max_age=300`) | Same as the portal route; audited `pii.revealed` |

### Events emitted

| Event | Payload | Consumers |
| --- | --- | --- |
| `verification.created` | `{ verification_id, created_at, bank_code, channel, branch_id }` | none at launch |
| `verification.status_changed` | `{ verification_id, created_at, from, to, reason, review_reason, assurance }` | 06 (hosted-page mirror), 08 |
| `verification.finalized` | `{ verification_id, created_at, status, reason, assurance, amount_minor, bank_code, channel, billable, reviewed_by }` | 12, 07 |
| `verification.review_required` | `{ verification_id, created_at, review_reason, amount_minor, bank_code }` | 07 |
| `verification.assurance_upgraded` | `{ verification_id, created_at }` | 07 (QA) |
| `verification.reversed` | `{ verification_id, created_at, failed_checks }` | 07; 08 reacts to the matching `status_changed` |
| `verification.bank_confirmed_after_rejection` | `{ verification_id, created_at }` | 07 (QA flag), platform `Notify` (tenant Owners and Admins) |

`finalized` is emitted once per verification, at the first transition into `verified`, `rejected`, `expired` or `cancelled`. `reversed` is a later status change and doesn't re-emit `finalized`.

### Events consumed

| Event | Handler behaviour |
| --- | --- |
| `receipt.uploaded` | Timeline `receipt_uploaded`; status unchanged until extraction (the `receipt_deadline` timer keeps running; see 6.2) |
| `receipt.extracted` | Section 6.5 |
| `receipt.extraction_failed` with `final: true` | Transition to `in_review` with `no_reference_found` |
| `receipt.extraction_failed` with `final: false` | Timeline only; the customer can retry on the same link (doc 06, section 6.5) |
| `bank.breaker_changed` with `to = closed` | Enqueue a `bank_recovered` attempt for every verification of that bank in `receipt_required` or `in_review` with review reason `bank_unavailable` or `bank_format_changed`, and in `processing` with no in-flight attempt. Each job gets a 0–30 s random delay; at most 500 per batch, with the next batch 30 s later |
| `tenant.created` | Insert 7 `sandbox_seed` claims in env `test`: `(bank, 'TEST_DUPLICATE-{tenantId}')` for each bank code |

## 6. Business rules and state machine

### 6.1 Create pipeline

Steps run in order; the first failure returns its error and creates nothing.

1. Validate the body (zod).
2. For multipart: stream the file through doc 06's `ReceiptsApi.stage()` to compute sha256 and enforce type and size. The request hash covers the canonical JSON body plus the file's sha256.
3. `Idempotency.run(key, requestHash, …)`. A replay returns the stored response; a staged file from a replay is discarded.
4. Portal and cashier channels in live: tenant `active` and `ReceivingAccountsApi.hasVerified`, else `403 live_not_enabled`. Tenant suspended: `403 tenant_suspended`.
5. Parse the input with `BankRegistry.parseInput()`. Bank or input errors return `4xx`. A receipt with no `bank_code` leaves the bank unknown.
6. If the bank is known: `listVerified(bank)`; empty means `422 no_receiving_account`. If `expected.receiving_account_id` is given, it must be among them, else `400 invalid_field`.
7. Live only: `QuotaService.check({ tenantId, env, channel })` (doc 12); `block` means `402 quota_exceeded` with `details.reason` `quota` or `unpaid`.
8. Snapshot the policy from `TenancyApi.getSettings()`, and compute `date_window_start = paid_after ?? (created_at − date_window_hours)`.
9. In **one transaction**: insert the verification (`processing`, `checks` all `not_run`), timeline `created`, outbox `verification.created`, and:
   - `insufficient` → transition to `receipt_required/reference_insufficient` and call `ReceiptsApi.createUploadLink(tx, …)` (every call passes `display: { amountMinor, externalId, bankCode }` for the hosted page);
   - a receipt → `ReceiptsApi.attachStaged(tx, …)`; doc 06 enqueues extraction after commit;
   - a lookup with `BankHealth.isCallable` false → `receipt_required/bank_unavailable`, plus an upload link and the background-retry schedule;
   - a lookup otherwise → set `attempt_count = 1`.
10. After commit, enqueue `bank-fetch.{bank}` job `att:{id}:1` if needed, and publish `ver:{id}` if the status is already non-processing.
11. Wait (6.7). Respond.

### 6.2 State machine

| From | To | Trigger | Guard | Side effects |
| --- | --- | --- | --- | --- |
| `processing` | `verified` | Bank `found` | Checks pass, receiver `match`, amount below `force_review_above` (or no threshold) | Claim `bank` (6.6); `assurance = bank_confirmed`; `billable = live`; `finalized` |
| `processing`, `receipt_required`, `in_review`¹ | `in_review` | Bank `found` | Checks pass and `force_review_above` reached → `tenant_policy_amount`; or receiver `unverifiable` and the other checks pass → `receiver_unverifiable` | Claim `bank` (held); `review_deadline_at`; `review_required` |
| `processing`, `receipt_required`, `in_review`¹ | `rejected` | Bank `found` | Any check fails | Reason by precedence (R15); `details.failed_checks`; no claim; release any provisional claim; `billable = live`; `finalized` |
| `processing`, `receipt_required`, `in_review`¹ | `rejected` | Bank `not_found` or `input_invalid` | No rechecks left (6.4) **and** (input is not from a receipt, **or** the receipt reference came from QR or the PDF text layer, **or** vision confidence for the reference is at least 0.90) | Reason `not_found`; `finalized` |
| `processing` | `receipt_required` | 3rd consecutive `transient` or `blocked`; or `circuit_open` or `disabled` | none | Reason `bank_unavailable`; upload link; `receipt_deadline_at`; background retries |
| `processing` | `receipt_required` | `format_changed` | none | Reason `bank_format_changed`; same side effects |
| `processing` (receipt-first), `receipt_required` | `processing` | `receipt.extracted` with at least 1 candidate | none | Candidate attempts (6.5) |
| `processing`, `receipt_required` | `in_review` | Receipt in hand **and** (`receipt.extraction_failed` with `final: true`; or no candidates; or the bank still fails after the candidate attempt; or every candidate is `not_found` without enough confidence) | none | `review_reason` respectively `no_reference_found`, `no_reference_found`, `bank_unavailable` or `bank_format_changed`, `reference_unconfirmed`; claim `receipt_provisional` when a reference exists (6.6); `review_required` |
| `receipt_required` | `expired` | `receipt_deadline_at` passes | No receipt attached to this verification | Reason `receipt_not_uploaded`; invalidate links; `finalized` |
| `receipt_required` | `in_review` | `receipt_deadline_at` passes | At least one receipt exists (for example unreadable, non-final) | `review_reason = no_reference_found`; invalidate links; `review_required`. If an extraction job is still running, the timer is re-armed for 10 min instead |
| `in_review` | `verified` | `applyReviewDecision(approve)` | Task matches the open task | `assurance` = `bank_confirmed` if the bank already answered, else `manual_review`; `reviewed_by`; claim kept or created (provisional); if `manual_review`, `recheck_until = now + 72 h`; `finalized` |
| `in_review` | `rejected` | `applyReviewDecision(reject)` | Reason given | Reason as given; release claims; post-decision rechecks for 72 h if the bank never answered; `finalized` |
| `in_review` | `receipt_required` | `requestNewReceipt` | none | Reason `image_requested`; release the provisional claim; new upload link; new `receipt_deadline_at` |
| `in_review` | `expired` | `review_deadline_at` passes | none | Reason `review_timed_out`; release claims; `finalized` |
| `verified` (manual_review) | `verified` (bank_confirmed) | Post-decision recheck `found` | All checks pass | Replace the provisional claim with the bank claim; timeline `assurance_upgraded`; `recheck_until = null` |
| `verified` (manual_review) | `reversed` | Post-decision recheck `found` with a failing check; **or** 3 consecutive `not_found` spanning at least 60 min while the breaker is closed; **or** the bank claim conflicts with another active claim | none | Reason `bank_contradicted`; `details.failed_checks`; release claims; `recheck_until = null` |
| `rejected` (reviewer) | `rejected` | Post-decision recheck `found` and all checks pass | none | Status unchanged; timeline `bank_confirmed_after_rejection`; event of the same name |
| `processing`, `receipt_required`, `in_review` | `cancelled` | Client cancel | none | Release claims; invalidate links; cancel timers; `finalized` |

¹ An attempt result in `receipt_required` or `in_review` is applied only for background, recovery, candidate or manual attempts. Review tasks close on `verification.finalized`; doc 07 handles a decision racing this.

Every transition into `in_review`, `verified`, `rejected`, `expired` or `cancelled` also calls `ReceiptsApi.invalidateUploadLinks(tx, id)`, so a customer's hosted upload link works only while the verification still wants a receipt from them. Doc 06's hosted endpoints use that as their gate; merchant-side receipts go through `addReceipt`, which checks the status.

Every transition takes `SELECT … FOR UPDATE` on the verification row, runs `assertTransition`, increments `version`, writes the timeline, and in the same transaction emits `verification.status_changed` (plus `finalized`, `review_required` or `reversed` where listed). It publishes `ver:{id}` after commit.

### 6.3 `decide()` (pure)

Input: `BankResult`, the verification snapshot (expected amount, `date_window_start`, `created_at`, policy, input provenance), the receiver match, and the duplicate pre-check.

Output: `{ kind: 'final'; status; reason?; checks } | { kind: 'review'; reviewReason; checks } | { kind: 'schedule'; trigger; delayMs } | { kind: 'receipt_required'; reason }`.

### 6.4 Attempt scheduling

| Situation | Next attempt | Limit |
| --- | --- | --- |
| Create | Immediate | none |
| `transient` or `blocked` while `processing` | 2 s, then 5 s | 3 consecutive (`transient_streak`), then `receipt_required/bank_unavailable` |
| `not_found` | Per `BankRegistry.notFoundRechecks(bank)`, in seconds from the first `not_found` | Default `[]`: final on the first answer (overview Q15) |
| `found` with bank status `pending` | Every 60 s | 5 (`pending_streak`), then `rejected/not_completed` |
| `found` with bank status `unknown` | none | `checks.bank_status = not_available`, which passes |
| Waiting in `receipt_required/bank_unavailable` or `in_review` with `bank_unavailable` or `bank_format_changed` | Every 300 s (`background_retry`); skipped without counting while the breaker is open | Until the status leaves |
| Breaker closes | `bank_recovered` immediately, with 0–30 s jitter | One per verification per close event |
| Receipt extracted | `receipt_candidate` for each candidate in order (primary first, then misread variants; at most 4 candidates, doc 06); stop at the first `found` | 4 attempts |
| After a review decision with no bank answer (`manual_review` approve, or reject) | +5 min, +15 min, +30 min, +1 h, then hourly | Until 72 h after the decision |
| Admin retry | Immediate (`manual_retry`) | Does not reset counters |
| Sweeper (repeatable job `vt:sweep` every 30 s) | Re-enqueues any verification whose last attempt has `outcome IS NULL` and started more than 60 s ago, or whose `next_attempt_at` passed more than 30 s ago with no job; same job ID, so it is idempotent | Covers lost enqueues and crashed workers |

- **One attempt in flight:** the next `attempt_no` is assigned under the row lock, and the job ID is `att:{id}:{n}`.
- **Late results:** if a result arrives for an `attempt_no` lower than the current one, it is recorded in `attempts` and otherwise ignored.
- **Final verifications:** attempts are allowed only while `recheck_until > now`. Any other attempt on a final verification is recorded and ignored.

### 6.5 Receipt extraction results

- `receipt.extracted` carries `bank_code`, `candidates[]` (each `{ value, method: 'share_link' | 'qr' | 'pdf_text' | 'vision' | 'variant', confidence }`, primary first, at most 4; doc 06 section 5) and `has_claimed_fields`. Claimed fields themselves stay in doc 06 and reach reviewers through `ReceiptsApi.getEvidence`.
- If the verification's `bank_code` is null, it is set from the event. If that bank has no verified receiving account for the tenant, the result is `rejected/receiver_mismatch`, because money paid into a bank where the tenant has no account isn't theirs.
- If `bank_code` was given and differs from the extracted bank, the result is `in_review/reference_unconfirmed`. The reviewer sees both.
- Candidates run in order (6.4), then `decide()` as usual.
- The candidate that produced `found` is stored as `input_reference` with `input_kind = receipt`.

### 6.6 Claims

| Operation | When | Conflict handling |
| --- | --- | --- |
| `claimBank(tx, v, officialRef)` | Transition to `verified`, or to `in_review` on a bank answer with checks passing | Insert `basis = bank`. On a unique violation: if the conflicting claim is this verification's own provisional claim, update it to `bank`. Otherwise, before approval the result is `rejected/duplicate` (with `details.first_claim` if the same tenant); after a manual approval it is `reversed/bank_contradicted` |
| `claimProvisional(tx, v, bank, ref)` | Entering `in_review` with a reference from the receipt but no bank answer | Insert `basis = receipt_provisional` with the receipt's normalised reference. On conflict: `rejected/duplicate` (the same receipt reference is already claimed or under review elsewhere) |
| `release(tx, v, reason)` | Reject, cancel, expire, reverse, image requested | `status = released`, `released_at`, `release_reason` |
| Duplicate pre-check | Before `decide()`, read-only, via `claim_owner()` | Sets `checks.duplicate`. Final enforcement is the insert |

Claims are per env. Sandbox seeds make `TEST_DUPLICATE` reject deterministically.

### 6.7 Waiting (`wait_seconds`)

1. After commit and enqueue, subscribe to Redis channel `ver:{id}`.
2. Re-read the row; if it is already in a status that answers the client (anything except `processing`), return it.
3. Wait for a message or the timeout (`wait_seconds`, at most 15 s).
4. On a message or a timeout, re-read the row once and return it: `200` if final, `202` otherwise.

The two reads cover a publish that landed before the subscription and a lost pub/sub message. If Redis is down, poll the row every 500 ms until the timeout.

### 6.8 Checks and precedence

| Check | Pass | Fail | Other values |
| --- | --- | --- | --- |
| `receiver` | `matchReceiver` → `match` | `no_match` → `receiver_mismatch` | `unverifiable` → review |
| `duplicate` | No active claim by another verification | Active claim exists → `duplicate` | n/a |
| `bank_status` | `completed` | `failed` → `not_completed` | `pending` → recheck (6.4); `unknown` → `not_available` (passes) |
| `amount` | credited = expected → `pass`; credited > expected → `over` (passes) | credited < expected → `amount_mismatch` | n/a |
| `date_window` | `date_window_start ≤ paid_at ≤ created_at + 5 min` | Otherwise → `stale` | n/a |

The primary reason is the first failing check in the order receiver, duplicate, bank_status, amount, date_window (R15). All results are stored in `checks`.

### 6.9 Public mapping and masking

| Situation | `transaction` | Payer fields |
| --- | --- | --- |
| `verified`, `reversed`, `in_review` with a bank answer | Full | Name in full; phone masked `+251 9•• ••• 234`; account masked `****1234` |
| `rejected` with `receiver_mismatch` | `null` | none |
| `rejected` (other reasons) with a bank answer | Full | Same as verified |
| `rejected/not_found`, or no bank answer yet | `null` | none |
| Manual review with no bank answer | `null`; reviewers' claimed fields stay internal (doc 07) | none |

- Amounts are decimal strings and times are `+03:00`.
- `receiving_account_id` appears only when it is this tenant's.
- `getForReview` (doc 07) returns the unmasked payer phone to staff and tenant reviewers only, and each access is audited as `pii.revealed` when unmasked fields are actually shown.

### 6.10 Billing flag

- `billable` is set at the first transition into `verified` or `rejected` in live, and never cleared.
- Expired and cancelled verifications aren't billable.
- `reviewed_by` tells doc 12 whether to count a platform review.

### 6.11 Portal statistics

`stats()` returns, for the range:

- `verified_count` and `verified_amount_minor` (sum of credited amounts);
- `bank_confirmed_ratio`;
- `rejected_count` and its top reason with count;
- `in_review_count` and the age of the oldest;
- `receipt_required_count`.

The data comes from `v_status` scans bounded to the range (at most 31 days), cached 30 s per tenant, env and range.

### 6.12 Numbers

| Rule | Value | Configured in |
| --- | --- | --- |
| `wait_seconds` range | 0–15; portal and cashier 15 | `VERIFICATION_MAX_WAIT_SECONDS` |
| Quick transient retries | 2,000 ms, 5,000 ms (3 tries total) | `VERIFICATION_TRANSIENT_RETRY_DELAYS_MS` |
| Background retry | 300 s | `VERIFICATION_BACKGROUND_RETRY_SECONDS` |
| Pending recheck | 60 s × 5 | `VERIFICATION_PENDING_RECHECK_SECONDS`, `_MAX` |
| Post-decision recheck | 300, 900, 1800, 3600 s, then hourly, for 72 h | `VERIFICATION_POST_DECISION_SCHEDULE_S`, `VERIFICATION_RECHECK_HOURS` (brief: 72 h) |
| Reversal on not_found | 3 consecutive, spanning at least 60 min | `VERIFICATION_REVERSAL_NOT_FOUND_COUNT`, `_SPAN_MINUTES` |
| Clock skew allowance | 5 min | `VERIFICATION_CLOCK_SKEW_MINUTES` |
| Vision confidence for a final `not_found` | 0.90 | `VERIFICATION_VISION_FINAL_CONFIDENCE` |
| Date window | Tenant `verification.date_window_hours` (default 24) | Doc 01 settings |
| Receipt wait | Tenant `verification.receipt_wait_hours` (default 24) | Doc 01 settings |
| Review max age | Tenant `review.max_age_hours` (default 24) | Doc 01 settings |
| Force review | Tenant `verification.force_review_above` (default off) | Doc 01 settings |
| Recovery batch | 500 per 30 s, 0–30 s jitter | `VERIFICATION_RECOVERY_BATCH`, `_JITTER_S` |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Ctx`, `Idempotency`, `Queues`, `Outbox`, `OnEvent`, `Crypto`, `Money`, `ids`, `Clock`, `AuditLog` (reveals, admin retry), Redis pub/sub |
| 01 tenancy | `TenancyApi.getSettings()`, `isTenantActive()` |
| 02 receiving accounts | `hasVerified`, `listVerified`, `matchReceiver`, `boaSuffixes` |
| 04 bank connectors | `BankRegistry.parseInput`, `notFoundRechecks`; `BankGateway.fetch`; `BankHealth.isCallable` |
| 06 receipts | `ReceiptsApi.stage`, `attachStaged(tx)`, `attachShareLink(tx)`, `createUploadLink(tx)`, `invalidateUploadLinks(tx)`, `activeUploadLink`, `latestForVerification`; events `receipt.*` |
| 12 billing | `QuotaService.check({ tenantId, env, channel })` |

**Transactions across modules:** synchronous calls that write in the caller's transaction (`attachStaged`, `attachShareLink`, `createUploadLink`, `invalidateUploadLinks`) take the caller's `Tx` as their first argument. This is the only cross-module write pattern allowed, and it is listed in each callee's public interface.

## 8. Implementation plan

1. **T05.1** Migrations: partitioned tables, claim functions, RLS; tests for partition pruning by ULID time and for the functions' visibility.
2. **T05.2** Enums and the `TRANSITIONS` table; `assertTransition` with tests for every row of 6.2 and rejection of every other pair.
3. **T05.3** `checks.ts`, `precedence.ts`, `date-window.ts`: table tests for 6.8, R4 and R15.
4. **T05.4** `decide.ts` and `schedule.ts`: pure tests covering every row of 6.2 and 6.4.
5. **T05.5** Repositories and the public mapper (6.9), with snapshot tests of the JSON for each status.
6. **T05.6** Create pipeline (6.1) for JSON input, sandbox only. Then the waiter (6.7). Then `POST /v1/verifications` with idempotency.
7. **T05.7** Attempt processor and `apply-result` service with the claims (6.6); the race test (US-05.23).
8. **T05.8** Timers processor (expiries, rechecks); the breaker-recovery handler; the tenant-created seed.
9. **T05.9** Receipt path: multipart create, `receipt.*` handlers, candidate attempts (needs doc 06 T06.1–T06.6, or its in-memory fake).
10. **T05.10** Review bridge: `applyReviewDecision`, `requestNewReceipt`, post-decision rechecks, reversal, upgrade.
11. **T05.11** Cancel; get, list and timeline; portal and admin controllers; `stats` and `rejectionReasons`.
12. **T05.12** Live path against mock banks: real adapters through doc 04's gateway with `MockAgent`, then Phase 0 smoke against real banks.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Two verifications for the same reference finish at the same time | The unique claim index lets one insert win; the other becomes `rejected/duplicate` in its own transaction (US-05.23) |
| The same API request is retried after a network drop | The idempotency key returns the original response; no second verification |
| A worker crashes after the bank answered but before commit | The job isn't acknowledged, so BullMQ redelivers it. The attempt re-runs (the bank is called again) and the result is applied once, because the transition is guarded by status and `version` |
| A bank answer arrives after cancel | Recorded in `attempts`, otherwise ignored; status stays `cancelled` |
| The bank answers while a reviewer is deciding | Whichever transaction locks the row first wins. A later review decision gets `409 invalid_state` from `applyReviewDecision`, and doc 07 shows "Bank answered: verified" |
| Receipt uploaded just after `receipt_deadline_at` | Doc 06 refuses with `410 upload_link_expired`, because the link expired before or with the deadline. If the expiry job hasn't run yet, the upload is accepted and the status moves on; the expiry job re-checks status under lock and does nothing |
| Settings change mid-flight | The snapshotted `policy` and `date_window_start` apply; changes affect only new verifications |
| Redis pub/sub message lost | The waiter's final re-read returns the true state |
| Redis down | Wait falls back to polling the row. The after-commit enqueue fails and is logged. The verification stays `processing` and the create returns normally (`202` after the wait). The **sweeper** (6.4) re-enqueues the attempt within 30 s of Redis returning. Nothing is lost, because the row is the source of truth |
| Outbox relay lagging | Webhooks and usage are delayed, never lost; API responses are unaffected |
| Partition for next month missing | The maintenance job creates 3 months ahead, and an alert fires if fewer than 2 future partitions exist. Inserts would fail with a clear error and return `503` |
| `bank_code` given but the reference is a link for another bank | `400 invalid_field` from doc 04, before any row is created |
| Huge burst of verifications for a degraded bank | The breaker opens. New creates go straight to `receipt_required` with no bank calls; the queue doesn't grow unboundedly |
| Reviewer approves; a recheck then finds the payment claimed by another tenant | `reversed/bank_contradicted` with `details.failed_checks: ["duplicate"]`; the other tenant's claim stands |
| A tenant deletes its receiving account mid-flight | The receiver check fails at answer time (`receiver_mismatch`) |
| The bank's official reference differs from the input | The claim uses the official reference. Search finds the verification by either (`v_inputref`, `v_official`) |

## 10. Security and privacy

| Endpoint | AuthN | AuthZ |
| --- | --- | --- |
| `/v1/verifications*` | API key | Scopes per doc 03, section 6.3; tenant and env from the key; RLS |
| `/portal/v1/verifications*` | Session plus CSRF | `verifications.*`; data scope `own` for Cashiers (`created_by_user_id = me`) |
| `/admin/v1/verifications*` | Admin session | `platform.verifications.*`; cross-tenant read via `pl_admin`; retry audited |

- PII handling follows 6.9: payer names and phones are encrypted at rest with the tenant key, and search uses HMACs.
- Cross-tenant facts are never revealed (R3): the receiver check comes first, and `first_claim` is shown only for the same tenant.
- Input validation in 5. `metadata` values are stored as strings and never interpreted.
- The timeline holds no PII, only IDs, classes and statuses.

## 11. Performance

| Item | Target |
| --- | --- |
| Create overhead before the bank call | p95 under 150 ms (validation, 1 transaction, 1 enqueue) |
| End to end with a healthy bank | p95 under 5 s (overview 3.8) |
| Peak create rate | 15 per second sustained, 50 per second burst |
| Get by ID | p95 under 10 ms (single partition, PK lookup) |
| List | p95 under 80 ms with `v_list` and cursor paging |
| Stats | p95 under 200 ms, cached 30 s |

Hot rows: each verification row is updated a handful of times. The claims table sees one insert per final verification; its unique index stays small because released claims are excluded.

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Every transition (allowed and forbidden); `decide()` table; checks, precedence and date window; schedule; public mapper per status |
| Integration (Testcontainers plus mock banks) | Flow A; flow B (bank down → receipt → review → approve → recheck upgrade); reversal by failing check and by 3 not-founds; race of two creates; idempotency replay and conflict; expiry timers with `FakeClock`; breaker-recovery fan-out; cancel during attempt; crash redelivery (kill the worker mid-attempt) |
| End to end (sandbox) | Every magic reference through `POST /v1/verifications` and the portal Verify screen; the cashier verified and do-not-release screens |
| Load | k6: 50 creates per second for 10 min against the sandbox; p95 create overhead under 150 ms; no claim anomalies |
| Amharic | Reason and status labels rendered for every enum value in `am` (portal and cashier) |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_verification_created_total` | counter | `env`, `channel`, `bank`, `input_kind` |
| `pl_verification_finalized_total` | counter | `env`, `status`, `reason`, `assurance`, `bank` |
| `pl_verification_time_to_final_seconds` | histogram | `bank`, `path` (bank, receipt, review) |
| `pl_verification_wait_total` | counter | `outcome` (final, timeout) |
| `pl_verification_open` | gauge | `status` |
| `pl_verification_claim_conflicts_total` | counter | `kind` (duplicate, provisional, reversal) |
| `pl_verification_reversed_total` | counter | `bank` |

| Alert | Condition | Severity |
| --- | --- | --- |
| Slow finals with healthy banks | `time_to_final` p95 above 10 s for 10 min while no breaker is open | Warning |
| Receipt-required ratio spike | Above 20% of creates over 15 min | Warning |
| Any reversal | `reversed_total` increase | Warning (notify Review Lead) |
| Future partitions | Fewer than 2 | Critical |
| Stuck processing | More than 50 verifications in `processing` older than 5 min | Critical |

## 14. Configuration

The variables are those in 6.12. Per-tenant settings are read from doc 01: `verification.date_window_hours`, `verification.force_review_above`, `verification.receipt_wait_hours` and `review.max_age_hours`. There are no feature flags.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 05-Q1 | Should an assurance upgrade (manual → bank confirmed) send a webhook? | No webhook; visible in GET and the portal timeline |
| 05-Q2 | When the bank confirms a payment a reviewer rejected, should the status flip to `verified`? | No; it stays `rejected`, and the tenant and Review Lead are notified |
| 05-Q3 | How should a bank `pending` status be handled per bank? | 5 rechecks over 5 min, then `not_completed` |
| 05-Q4 | Should `amount`, `paid_at` and the reference be shown on `receiver_mismatch`? | No (`transaction: null`) |
| 05-Q5 | Retention for verifications, attempts and timeline (overview Q13) | Kept indefinitely until decided; partitions make dropping cheap later |
