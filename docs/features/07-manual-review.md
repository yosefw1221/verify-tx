# 07 · Manual review

| | |
| --- | --- |
| Package | `@proofline/review` |
| Postgres schema | `review` |
| Depends on | platform, 01 tenancy, 04 bank connectors (`BankHealth` only), 05 verification, 06 receipts, 12 billing (`PlansApi` for the SLA) |
| Build step | 7 |
| Mockups | Admin · Review queue (SLA chips, flags, tabs, Take next task); Admin · Review workspace (receipt viewer, "Receipt against expectation", risk signals, decision panel, second-approval banner, timeline, keyboard keys); portal sidebar "Review queue" for Hybrid tenants |

## 1. Purpose and scope

**Does**

- Turns each `verification.review_required` into a review task with an SLA.
- Runs the queue: priority order, "Take next", claiming with a 10-minute lock, Review Lead reassignment.
- Records decisions: approve, reject with a reason, ask for a clearer image. Applies them through `VerificationApi`.
- Enforces second approval above the tenant's threshold (four-eyes, R8).
- Runs Hybrid mode: the tenant's Owners and Admins get the task first, then it escalates to the platform queue (R7).
- Runs SLA timers: amber at 80%, breach escalation to a Review Lead (R9).
- Builds the review workspace: one call that assembles receipt, comparison, risk signals, bank status, history and timeline.
- Does QA sampling and flags overturned decisions.
- Runs the sandbox reviewer for test mode.

**Out of scope**

- Post-decision bank rechecks and reversals. Doc 05 runs those; this module only records the outcome on the task.
- Expiring a verification at `review.max_age_hours`. Doc 05's timer does that, and the task closes from the event.
- Receipt storage and reading (doc 06).

## 2. Screens and user stories

| Screen | Surface | Mockup |
| --- | --- | --- |
| Review queue | Admin | Admin · Review queue |
| Review workspace | Admin | Admin · Review workspace |
| Review queue and workspace (tenant stage) | Portal | Not drawn; same components, reached from the "Review queue" nav item |
| QA sampling list | Admin, Review Lead | Not drawn; a table on the Review queue page under a "QA" tab |

**US-07.1 Task created**

- On `verification.review_required`, exactly one open task exists for that verification.
- It gets the next human number (shown as `RV-20418`), stage `platform` (Platform mode) or `tenant` (Hybrid), the SLA from the plan (30 min, or 240 min on Starter) or the tenant's Hybrid window (15 min), and `high_value` set when the amount is at or above the tenant's `review.second_approval_above`.

**US-07.2 Queue order and tabs**

- The platform queue lists open and claimed platform-stage tasks: escalated first, then by `due_at`, then by amount (descending). This matches the mockup's "sorted by time left".
- The tabs count All, Mine, Due within 10 min, High value and Unassigned.
- Each row shows time left (amber at 80% of the SLA, red when breached), task number, tenant, bank, amount, why in review, flags, waiting time, assignee and Open.

**US-07.3 Take next**

- "Take next task" (key N) claims the highest-priority task the user is allowed to decide and opens the workspace.
- Two reviewers pressing it together get different tasks.

**US-07.4 Lock**

- A claimed task is locked to its reviewer for 10 minutes. The workspace heartbeat extends the lock while the page stays open; the header shows "Locked to you · 8:12".
- Another reviewer opening it sees it read-only, with the holder's name.
- A lock lapses 10 minutes after the last heartbeat.

**US-07.5 Workspace content**

- The workspace shows the receipt (zoom, rotate, contrast, Original or Enhanced), the QR-read bar, and image facts (file name, size, upload time, "EXIF stripped", sha256 prefix).
- When the bank couldn't be reached, a bank banner says so and when it was last tried.
- The comparison table lists Reference, Amount, Paid to, Account, Paid by, Paid at and Status, each with the receipt value plus confidence, the merchant's expectation, and the check result.
- Also shown: risk signals and history, the decision panel and the timeline. All of it comes from one `GET …/workspace` call.

**US-07.6 Approve below threshold**

- Approve (key A) applies `approve`. The verification becomes `verified` with `assurance: manual_review`, the task closes with outcome `approved`, and `review.decided` is emitted.

**US-07.7 Approve above threshold**

- The banner reads "Second approval needed". Approve with a note (required) moves the task to `awaiting_second`; the verification is unchanged.
- A Review Lead (platform stage) or a *different* tenant Owner or Admin (tenant stage) confirms, which applies the approval, or declines with a reject reason, which applies the rejection.
- The first approver can't give the second approval: `409 same_approver`.

**US-07.8 Reject**

- Reject (key R) requires a reason from: receipt looks altered, receipt can't be read, paid to a different account, amount is less than expected, payment is too old, receipt already used, transfer not completed. It also requires a note.
- The verification becomes `rejected` with that reason.

**US-07.9 Clearer image**

- "Ask for a clearer image" (key I) closes the task with outcome `image_requested`. The verification returns to `receipt_required/image_requested` with a new upload link, and the merchant receives `verification.receipt_required`.

**US-07.10 Escalate**

- "Escalate to a Review Lead" (key E) sets `escalated_at`, moves the task to the top of the queue, notifies Review Leads, and releases the lock.

**US-07.11 SLA**

- At 80% of the SLA the row turns amber and `review.sla_amber` is emitted.
- At 100% on the platform stage, the task is escalated automatically (`review.sla_breached`) and Review Leads are notified.
- At 100% on the tenant stage (Hybrid), the task moves to the platform stage with a fresh plan SLA (`review.escalated_to_platform`). That is a handoff, not a breach.

**US-07.12 Bank answers during review**

- A background attempt (doc 05) finalizes the verification. The task closes with outcome `bank_answered`.
- A reviewer still in the workspace sees "Bank answered: Verified" (or Rejected), and the decision buttons disable. A decision sent after that returns `409 task_closed`.

**US-07.13 Hybrid with a single tenant admin**

- In the tenant stage of a high-value task, if no second eligible tenant approver exists, the first approval moves the task to the platform stage as `awaiting_second`, so a Review Lead gives the second approval.

**US-07.14 Overturned**

- When doc 05 reverses an approved verification, the task gets `overturned_at`, is added to QA automatically, and counts against the deciding reviewer's accuracy metric.

**US-07.15 QA sampling**

- Each day at 07:00 EAT, 5% of the previous day's platform decisions (at least 5, at most 50) plus every overturned or "bank confirmed after rejection" task are added to the QA list.
- A Review Lead marks each one Agree or Disagree with a note.

**US-07.16 Sandbox reviewer**

- In test env, an open task is decided 30 s after creation: approve, or reject with `unreadable_receipt` when the verification's input reference is `TEST_REVIEW_REJECT`.
- If a tenant user has claimed it first, the sandbox reviewer leaves it alone.

**US-07.17 Reviewer stats**

- The queue header shows the reviewer's decided-today count, median decision time, and QA disagreements in the last 30 days, as in the mockup.

## 3. Module design

```text
libs/features/review/src/
├─ public/
│  ├─ index.ts                 ReviewModule, ReviewApi, events
│  ├─ review-api.ts
│  └─ events.ts
├─ domain/
│  ├─ task-state.ts            transitions (section 6.1)
│  ├─ eligibility.ts           who may claim, decide or second-approve (pure)
│  ├─ priority.ts              ordering key
│  └─ comparison.ts            builds the workspace comparison rows (pure)
├─ application/
│  ├─ intake.handler.ts        @OnEvent verification.review_required
│  ├─ lifecycle.handler.ts     @OnEvent verification.finalized, .reversed, .assurance_upgraded, .bank_confirmed_after_rejection
│  ├─ queue.service.ts         list, counts, take-next, claim, heartbeat, release, reassign
│  ├─ decision.service.ts      decide, second approval, escalate, request image
│  ├─ timers.processor.ts      queue review-timers (amber, due)
│  ├─ workspace.service.ts     aggregates for the UI
│  ├─ qa.service.ts            sampling job and verdicts
│  └─ sandbox-reviewer.processor.ts
├─ infrastructure/ tasks.repo.ts, actions.repo.ts, qa.repo.ts
├─ http/
│  ├─ admin/review.controller.ts       /admin/v1/review/*
│  └─ portal/review.controller.ts      /portal/v1/review/* (tenant stage)
└─ db/schema.ts, db/migrations/
```

```ts
export interface ReviewApi {
  openTaskForVerification(verificationId: string): Promise<ReviewTaskSummary | null>;   // portal drawer, admin explorer
  counts(scope: { kind: 'platform' } | { kind: 'tenant'; tenantId: string; env: EnvName }): Promise<QueueCounts>;  // sidebar badges, dashboard
  backlog(range: { from: Date; to: Date }): Promise<ReviewBacklog>;                    // admin dashboard (doc 11); platform stage, live
  findByNumber(number: number): Promise<ReviewTaskSummary | null>;                     // admin search "RV-20418" (doc 11)
}
export interface ReviewBacklog {
  under10m: number; from10to30m: number; over30m: number;                              // open platform-stage tasks by time left to due_at
  oldest: { taskId: string; number: number; waitingSeconds: number; tenantId: string; slaMinutes: number } | null;
  breaches: number; breachesEscalated: number;                                         // sla_breached actions in range
}
export interface ReviewTaskSummary {
  id: string; number: number; stage: 'tenant' | 'platform';
  status: 'open' | 'claimed' | 'awaiting_second' | 'closed';
  reviewReason: ReviewReason; dueAt: Date; amberAt: Date; highValue: boolean; escalated: boolean;
}
export interface QueueCounts { all: number; mine: number; dueWithin10m: number; highValue: number; unassigned: number;
  breachingSoon: number; oldestWaitingSeconds: number | null }
```

## 4. Data model

```sql
CREATE SEQUENCE review.task_number_seq START 20000;

CREATE TABLE review.tasks (
  id                       uuid PRIMARY KEY,
  number                   bigint NOT NULL UNIQUE DEFAULT nextval('review.task_number_seq'),
  tenant_id                uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                      text NOT NULL,
  verification_id          uuid NOT NULL,
  verification_created_at  timestamptz NOT NULL,
  review_reason            text NOT NULL,
  bank_code                text NULL,
  amount_minor             bigint NOT NULL,
  mode                     text NOT NULL CHECK (mode IN ('platform','hybrid')),        -- snapshot
  stage                    text NOT NULL CHECK (stage IN ('tenant','platform')),
  high_value               boolean NOT NULL,
  second_approval_above_minor bigint NULL,                                            -- snapshot
  plan_sla_minutes         integer NOT NULL,                                          -- snapshot from PlansApi
  status                   text NOT NULL CHECK (status IN ('open','claimed','awaiting_second','closed')),
  outcome                  text NULL CHECK (outcome IN ('approved','rejected','image_requested','bank_answered','expired','cancelled')),
  sla_started_at           timestamptz NOT NULL,
  amber_at                 timestamptz NOT NULL,
  due_at                   timestamptz NOT NULL,
  escalated_at             timestamptz NULL,
  escalation_cause         text NULL CHECK (escalation_cause IN ('sla_breach','manual')),
  claimed_by_kind          text NULL CHECK (claimed_by_kind IN ('staff','user')),
  claimed_by_id            uuid NULL,
  claimed_at               timestamptz NULL,
  locked_until             timestamptz NULL,
  first_decision           jsonb NULL,       -- {decision, reason, note, actor_kind, actor_id, at}
  second_actor_kind        text NULL,
  second_actor_id          uuid NULL,
  second_at                timestamptz NULL,
  decided_by_kind          text NULL,        -- who applied the final decision: staff or user
  decided_at               timestamptz NULL,
  closed_at                timestamptz NULL,
  overturned_at            timestamptz NULL,
  qa_state                 text NOT NULL DEFAULT 'none' CHECK (qa_state IN ('none','sampled','reviewed')),
  created_at               timestamptz NOT NULL DEFAULT now(),
  updated_at               timestamptz NOT NULL DEFAULT now(),
  version                  integer NOT NULL DEFAULT 0
);
CREATE UNIQUE INDEX tasks_one_open ON review.tasks (verification_id) WHERE status <> 'closed';
CREATE INDEX tasks_platform_queue ON review.tasks (escalated_at DESC NULLS LAST, due_at, amount_minor DESC)
  WHERE status IN ('open','claimed','awaiting_second') AND stage = 'platform' AND env = 'live';
CREATE INDEX tasks_tenant_queue ON review.tasks (tenant_id, env, due_at)
  WHERE status IN ('open','claimed','awaiting_second') AND stage = 'tenant';
CREATE INDEX tasks_claimed_by ON review.tasks (claimed_by_id) WHERE status = 'claimed';
CREATE INDEX tasks_decided ON review.tasks (decided_at) WHERE outcome IN ('approved','rejected');

CREATE TABLE review.actions (
  id          uuid PRIMARY KEY,
  task_id     uuid NOT NULL REFERENCES review.tasks(id),
  tenant_id   uuid NOT NULL, env text NOT NULL,
  at          timestamptz NOT NULL DEFAULT now(),
  actor_kind  text NOT NULL CHECK (actor_kind IN ('staff','user','system')),
  actor_id    uuid NULL,
  action      text NOT NULL CHECK (action IN ('created','claimed','released','lock_lapsed','reassigned','decided','second_approved',
                 'second_declined','escalated','stage_changed','sla_amber','sla_breached','closed','qa_sampled','qa_reviewed','overturned')),
  data        jsonb NOT NULL DEFAULT '{}'::jsonb      -- reason codes and notes only; no payer PII
);
CREATE INDEX actions_task ON review.actions (task_id, at);

CREATE TABLE review.qa_reviews (
  id          uuid PRIMARY KEY,
  task_id     uuid NOT NULL REFERENCES review.tasks(id),
  sampled_at  timestamptz NOT NULL DEFAULT now(),
  sample_cause text NOT NULL CHECK (sample_cause IN ('random','overturned','bank_confirmed_after_rejection')),
  reviewer_id uuid NULL,                    -- staff (Review Lead)
  verdict     text NULL CHECK (verdict IN ('agree','disagree')),
  note        text NULL CHECK (length(note) <= 1000),
  reviewed_at timestamptz NULL,
  UNIQUE (task_id)
);
```

- **RLS:** the standard tenant and env policy on `tasks` and `actions`. Platform-stage queries from `admin-api` use the `pl_admin` cross-tenant `SELECT`. Writes run in a transaction scoped to the task's tenant (overview 5.4).
- **Migration:** `0001_review_init.sql`.

## 5. API contracts

### Admin (`/admin/v1/review`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /tasks` | `platform.review.read` | `?tab=all\|mine\|due_10m\|high_value\|unassigned&bank_code&review_reason&amount_band&tenant_id&mode&cursor&limit` | `200 { data: QueueRow[], counts: QueueCounts, me: ReviewerStats, next_cursor }` | none |
| `POST /tasks/next` | `platform.review.decide` | none | `200 { task_id }` (claimed) or `204` (nothing eligible) | none |
| `POST /tasks/{id}/claim` | `platform.review.decide` | none | `200 TaskView` | `409 task_locked` (`details.holder`), `409 task_closed`, `403 forbidden` (not eligible) |
| `POST /tasks/{id}/heartbeat` | Holder | none | `200 { locked_until }` | `409 lock_lost` |
| `POST /tasks/{id}/release` | Holder | none | `204` | `409 lock_lost` |
| `POST /tasks/{id}/reassign` | `platform.review.reassign` | `{ staff_id }` | `200 TaskView` | `400 invalid_field` (target lacks permission) |
| `GET /tasks/{id}/workspace` | `platform.review.read` | none | `200 Workspace` (6.6) | `404` |
| `POST /tasks/{id}/decision` | Holder with `platform.review.decide` | `{ decision: "approve"\|"reject"\|"request_image", reason?, note? }` | `200 TaskView` | `400 missing_field` (reason or note), `409 lock_lost`, `409 task_closed` (`details.verification_status`) |
| `POST /tasks/{id}/second-approval` | `platform.review.approve_second` | `{ action: "confirm"\|"decline", reason?, note? }` (decline needs a reason) | `200 TaskView` | `409 same_approver`, `409 invalid_state` (not `awaiting_second`), `409 task_closed` |
| `POST /tasks/{id}/escalate` | Holder, or `platform.review.reassign` | `{ note }` | `200 TaskView` | `409 task_closed` |
| `POST /tasks/{id}/retry-bank` | Holder | none | `202` (calls `VerificationApi.retryBankNow`) | none |
| `GET /qa` | `platform.review.qa` | `?state=sampled\|reviewed&cursor` | `200 Page<QaRow>` | none |
| `POST /qa/{task_id}` | `platform.review.qa` | `{ verdict, note? }` | `200` | `409 invalid_state` |

**Note rules:** `note` is 3–1,000 characters. It is required on reject, on approving a high-value task, on a second-approval decline, and on escalation.

### Portal (`/portal/v1/review`), tenant stage, Hybrid tenants only

| Method and path | Permission | Notes |
| --- | --- | --- |
| `GET /tasks` | `review.read` | The tenant's tenant-stage tasks, plus platform-stage tasks read-only (with the label "With Proofline reviewers") |
| `POST /tasks/next`, `/tasks/{id}/claim`, `/heartbeat`, `/release`, `/decision`, `/escalate` | `review.decide` | Same contracts as admin, restricted to tenant-stage tasks. `escalate` moves the task to the platform stage immediately |
| `POST /tasks/{id}/second-approval` | `review.approve_second` | Different user from the first approver |
| `GET /tasks/{id}/workspace` | `review.read` | Same `Workspace` shape. Other-tenant signal counts are shown only as "Seen at another merchant" (no numbers). The workspace reminds the reviewer: "Check your bank statement for this payment" |

### Workspace response (abridged)

```json
{
  "task": { "id": "rvw_01J…", "number": 20418, "stage": "platform", "status": "claimed", "high_value": true,
            "second_approval_above": "10000.00", "due_at": "2026-10-04T10:46:00+03:00", "amber_at": "2026-10-04T10:40:00+03:00",
            "locked_until": "2026-10-04T10:48:12+03:00", "holder": { "kind": "staff", "name": "Mekdes Alemu", "is_me": true },
            "first_decision": null },
  "verification": { "id": "ver_01J…", "tenant": { "id": "ten_01J…", "name": "Habesha Mart" }, "external_id": "order_88231",
                    "status": "in_review", "review_reason": "bank_unavailable", "bank": { "code": "TELEBIRR", "name": "telebirr" },
                    "expected": { "amount": "12500.00", "paid_after": "2026-10-04T10:05:00+03:00", "receiving_account": { "holder": "Habesha Mart PLC", "masked": "****4401" } } },
  "receipt": { "receipt_id": "rcpt_01J…", "display_urls": ["https://…60s"], "content_type": "image/jpeg", "page_count": 1,
               "file": { "name_hint": "IMG_2041.jpg", "size_bytes": 1258291, "uploaded_at": "2026-10-04T10:16:03+03:00", "source": "hosted",
                         "exif_stripped": true, "sha256_short": "9f3c…a71e" },
               "qr": { "decoded": true, "reference": "DJK7Q4P2ZA", "bank_code": "TELEBIRR" } },
  "bank_status": { "answered": false, "breaker": "open", "since": "2026-10-04T10:12:00+03:00", "attempts": 7,
                   "last_attempt_at": "2026-10-04T10:39:00+03:00", "last_failure": "transient", "next_attempt_at": "2026-10-04T10:44:00+03:00" },
  "comparison": [
    { "field": "reference", "on_receipt": { "value": "DJK7Q4P2ZA", "confidence": 0.99, "method": "qr" }, "expected": null, "check": { "result": "pass", "label_key": "review.check.valid_format" } },
    { "field": "amount", "on_receipt": { "value": "12500.00", "confidence": 0.98, "method": "vision" }, "expected": "12500.00", "check": { "result": "pass", "label_key": "review.check.matches" } },
    { "field": "paid_to", "on_receipt": { "value": "HABESHA MART PLC", "confidence": 0.95 }, "expected": "Habesha Mart PLC", "check": { "result": "pass", "label_key": "review.check.registered_account" } },
    { "field": "account", "on_receipt": { "value": "****4401", "confidence": 0.92 }, "expected": "****4401", "check": { "result": "pass", "label_key": "review.check.matches" } },
    { "field": "paid_by", "on_receipt": { "value": "HANNA BEKELE", "confidence": 0.96 }, "expected": null, "check": { "result": "info", "label_key": "review.check.for_reference" } },
    { "field": "paid_at", "on_receipt": { "value": "2026-10-04T10:14:52+03:00", "confidence": 0.97 }, "expected": "after 2026-10-04T10:05:00+03:00", "check": { "result": "pass", "label_key": "review.check.inside_window" } },
    { "field": "status", "on_receipt": { "value": "Successful", "confidence": 0.97 }, "expected": null, "check": { "result": "pass", "label_key": "review.check.completed" } }
  ],
  "signals": [ { "level": "info", "key": "review.signal.first_payment_from_payer" },
               { "level": "ok", "key": "review.signal.image_not_seen" },
               { "level": "ok", "key": "review.signal.reference_not_claimed" },
               { "level": "ok", "key": "review.signal.no_editing_traces" } ],
  "history": { "same_payer_verified_30d": 0, "same_image_receipts": [], "same_reference_verifications": [] },
  "timeline": [ { "at": "2026-10-04T10:14:52+03:00", "key": "timeline.paid_per_receipt" }, "…" ]
}
```

### New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `task_locked` | 409 | Claimed by someone else; `details.holder.name` |
| `lock_lost` | 409 | Caller no longer holds the lock |
| `task_closed` | 409 | Task closed (bank answered, cancelled, expired, already decided); `details.outcome`, `details.verification_status` |
| `same_approver` | 409 | Second approver equals the first approver |

### Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `review.task_created` | `{ task_id, verification_id, stage, due_at, high_value }` | 06 (hosted In review time) |
| `review.decided` | `{ task_id, verification_id, decision, reason, stage, decided_by_kind, second_approval: boolean }` | none at launch (doc 05 learns decisions through the direct call) |
| `review.sla_amber` | `{ task_id, stage }` | none at launch |
| `review.sla_breached` | `{ task_id, tenant_id, number, amount_minor }` | 11 (alerts feed), platform `Notify` (staff with the Review Lead role) |
| `review.escalated_to_platform` | `{ task_id, verification_id, cause: "hybrid_window" \| "tenant_escalated" \| "no_second_tenant_approver", due_at }` | 06 (updates the hosted In review time) |
| `review.task_closed` | `{ task_id, outcome }` | none at launch |

**Consumes:** `verification.review_required`, `verification.finalized`, `verification.reversed`, `verification.assurance_upgraded`, `verification.bank_confirmed_after_rejection`, and `receipt.claimed_fields_ready` (pushes a refresh hint to open workspaces).

## 6. Business rules and state machine

### 6.1 Task states

| From | To | Trigger | Guard | Side effects |
| --- | --- | --- | --- | --- |
| (none) | `open` | `verification.review_required` | No open task for the verification (the unique index makes the handler idempotent) | Snapshot mode, stage, threshold and SLA; arm amber and due timers; `review.task_created` |
| `open` | `claimed` | Claim, or Take next | Eligible (6.3) | `locked_until = now + 10 min` |
| `claimed` | `claimed` | Heartbeat by the holder | Lock held | `locked_until = now + 10 min` |
| `claimed` | `open` | Release, lock lapse (read lazily: `locked_until < now`), escalate | none | Action `released` or `lock_lapsed` |
| `claimed` | `closed` | Decision `approve` (not high-value), `reject`, or `request_image` | Lock held | Calls `VerificationApi.applyReviewDecision` or `requestNewReceipt` in the same request. On success, outcome is `approved`, `rejected` or `image_requested`, and `review.decided` is emitted |
| `claimed` | `awaiting_second` | Decision `approve` on a high-value task | Lock held; note given | Store `first_decision`; release the lock; notify eligible second approvers |
| `awaiting_second` (tenant stage) | `awaiting_second` (platform stage) | First approval made and no other eligible tenant approver exists | none | `review.escalated_to_platform` with cause `no_second_tenant_approver` |
| `awaiting_second` | `closed` | Second approval `confirm` | Actor ≠ first approver; eligible | Apply approve with `secondApprover`; outcome `approved` |
| `awaiting_second` | `closed` | Second approval `decline` with a reason | Same | Apply reject; outcome `rejected` |
| `open`, `claimed`, `awaiting_second` (tenant stage) | Same status, platform stage | `due_at` on the tenant stage, or tenant escalate | none | Stage `platform`; `sla_started_at = now`; new amber and due from `plan_sla_minutes`; keep `first_decision`; release the lock; `review.escalated_to_platform` |
| any open (platform stage) | Same status | `due_at` on the platform stage | none | `escalated_at = now`, `escalation_cause = sla_breach`; `review.sla_breached` |
| any not closed | `closed` | `verification.finalized` not caused by this task | none | Outcome: `bank_answered` (status verified or rejected via bank), `expired` or `cancelled` |

- **Concurrency:** every transition locks the task row `FOR UPDATE` and checks `version`.
- **Calling doc 05:** decisions call doc 05 *after* the task row is locked and *before* the task commits. If doc 05 refuses with `409 invalid_state`, the task transaction is rolled back. A fresh transaction then reads the verification status and closes the task with the matching outcome, and the API returns `409 task_closed`.

### 6.2 SLA

| Mode and stage | SLA | Amber | At due |
| --- | --- | --- | --- |
| Platform, platform stage | Plan `review_sla_minutes`: 30 (Growth, Enterprise default) or 240 (Starter) | 80% elapsed | Escalate (breach) |
| Hybrid, tenant stage | Tenant `review.hybrid_tenant_window_minutes` (default 15) | 80% elapsed | Move to the platform stage with a fresh plan SLA |
| Hybrid, platform stage | Plan SLA from the handoff | 80% | Escalate (breach) |

- `amber_at = sla_started_at + 0.8 × SLA` and `due_at = sla_started_at + SLA`.
- Timers are jobs `rt:{taskId}:amber:{dueAtEpoch}` and `rt:{taskId}:due:{dueAtEpoch}` on the `review-timers` queue. A stage change cancels the old pair and adds a new pair; the epoch in the ID keeps the new jobs from colliding with the old ones. A fired timer re-checks the state under lock and does nothing if it no longer applies.
- `review.max_age_hours` (default 24) is enforced by doc 05. When the verification expires, the task closes with outcome `expired`.

### 6.3 Eligibility

| Action | Platform stage | Tenant stage |
| --- | --- | --- |
| See queue | `platform.review.read` | `review.read`, same tenant |
| Claim and decide | `platform.review.decide` | `review.decide` (Owner or Admin) |
| Second approval | `platform.review.approve_second` (Review Lead, Admin), not the first approver | `review.approve_second`, same tenant, not the first approver |
| Reassign | `platform.review.reassign` | Not available |
| Test-env tasks | Never visible to platform staff | Visible to the tenant; the sandbox reviewer decides after 30 s unless claimed |

### 6.4 Priority (Take next and queue order)

Ordering key, ascending:

1. Escalated first (`escalated_at IS NULL`).
2. `due_at` (earliest first).
3. `amount_minor` (largest first).
4. `created_at`.

Take next selects `FOR UPDATE SKIP LOCKED` from the eligible open tasks, treating a lapsed claim as open. For a second approver, it includes `awaiting_second` tasks and excludes those where the caller made the first decision.

### 6.5 Decision rules

| Decision | Requirements | Effect on the verification (doc 05) |
| --- | --- | --- |
| `approve` | Lock held. Note required if high-value | `verified`; assurance `bank_confirmed` if the bank already answered (tenant-policy or receiver reviews), else `manual_review`, with 72 h rechecks |
| `reject` | Lock held; a reason from {`suspected_forgery`, `unreadable_receipt`, `receiver_mismatch`, `amount_mismatch`, `stale`, `duplicate`, `not_completed`}; a note | `rejected` with that reason |
| `request_image` | Lock held; the verification has a receipt; at most 2 image requests per verification (`REVIEW_MAX_IMAGE_REQUESTS`). A 3rd is refused with `409 invalid_state` | `receipt_required/image_requested` with a new upload link |

`reviewed_by` sent to doc 05 is `platform` when the final decision was applied on the platform stage, else `tenant`. Doc 12 bills platform reviews.

### 6.6 Workspace assembly

| Part | Source |
| --- | --- |
| Task and holder | This module |
| Verification, expected values, receiving account | `VerificationApi.getForReview` (unmasked payer phone shown only on demand; each reveal is audited) |
| Receipt and QR bar | `ReceiptsApi.getEvidence`; if `claimed` is null, call `ensureClaimedFields` and show "Reading fields…" until `receipt.claimed_fields_ready` triggers a client refetch |
| Bank banner | `BankHealth.state(bank)` plus the attempts from `getForReview` |
| Comparison | `comparison.ts` (pure), from the expectation, claimed fields, candidates and the receiving account. Rows and labels are as in the mockup |
| Signals | Receipt signals (doc 06), `reference_seen_elsewhere` (doc 05), first payment from this payer (`findRelated`: same payer phone HMAC at this tenant in 30 days), PDF and EXIF facts |
| History | `VerificationApi.findRelated` |
| Timeline | Doc 05 timeline plus this module's actions, merged by time |

### 6.7 QA sampling

- **Daily job (07:00 EAT):** take the previous EAT day's platform-stage decisions with outcome `approved` or `rejected`. Sample 5% uniformly at random (`REVIEW_QA_SAMPLE_PCT`), at least 5 and at most 50.
- **Always added:** tasks with `overturned_at` set, and tasks whose verification later got `bank_confirmed_after_rejection`.
- **Reviewer accuracy** over 30 days is `1 − (disagree + overturned) / decided`, shown to Review Leads.

### 6.8 Numbers

| Rule | Value | Where |
| --- | --- | --- |
| Platform SLA | 30 min; 240 min on Starter (brief) | `billing.plans.review_sla_minutes` (doc 12) |
| Amber | 80% of SLA (brief) | `REVIEW_AMBER_RATIO` = 0.8 |
| Lock | 10 min (brief); heartbeat every 60 s | `REVIEW_LOCK_MINUTES`, `REVIEW_HEARTBEAT_SECONDS` |
| Hybrid tenant window | Tenant setting, default 15 min (R7) | Doc 01 `review.hybrid_tenant_window_minutes` |
| Second approval threshold | Tenant setting, default 10,000.00 ETB | Doc 01 `review.second_approval_above` |
| Image requests per verification | 2 | `REVIEW_MAX_IMAGE_REQUESTS` |
| QA sample | 5% (5–50 per day) | `REVIEW_QA_SAMPLE_PCT`, `_MIN`, `_MAX` |
| Sandbox reviewer delay | 30 s | `REVIEW_SANDBOX_DELAY_SECONDS` |
| Note length | 3–1,000 characters | Constant |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Queues` (`review-timers`), `Outbox`, `OnEvent`, `AuditLog`, `Notify`, `Clock`, `ids` |
| 01 tenancy | `TenancyApi.getSettings()` (mode, window, threshold); `listMemberEmails(tenantId, ['owner', 'admin'])` (second approvers, tenant stage); `getStaff()` (holder names); `listStaffEmails(['review_lead'])` (breach and escalation notices) |
| 04 bank connectors | `BankHealth.state()` (workspace banner). Adds the edge `review → bank-connectors` to the overview DAG |
| 05 verification | `getForReview`, `applyReviewDecision`, `requestNewReceipt`, `findRelated`, `retryBankNow`; its events |
| 06 receipts | `getEvidence`, `ensureClaimedFields`; event `receipt.claimed_fields_ready` (client refetch only) |
| 12 billing | `PlansApi.getEffectivePlan()` (`review_sla_minutes`). Adds the edge `review → billing` |

## 8. Implementation plan

1. **T07.1** Migration, sequence, RLS; tests for the one-open-task index.
2. **T07.2** `task-state.ts`, `eligibility.ts`, `priority.ts`, `comparison.ts`: pure tests for every row of 6.1, 6.3, 6.4 and the comparison labels.
3. **T07.3** Intake handler (idempotent), timers processor (amber, due, stage handoff, breach), lifecycle handler (close on finalize, overturned flags).
4. **T07.4** Queue service: list with tabs and counts, Take next with `SKIP LOCKED` (a concurrency test with 10 parallel takers gets 10 distinct tasks), claim, heartbeat, release, reassign, lock lapse.
5. **T07.5** Decision service: approve, reject, request image, escalate, second approval with four-eyes, the race with doc 05 (`task_closed`).
6. **T07.6** Workspace service and endpoint, with an integration test against fakes of docs 04, 05 and 06.
7. **T07.7** QA job and verdicts; reviewer stats.
8. **T07.8** Sandbox reviewer.
9. **T07.9** Admin UI: Review queue and Review workspace exactly as in the mockups, with keyboard shortcuts A, R, I, E, J, K and N, the lock countdown, the bank-answered banner, the QA tab.
10. **T07.10** Portal tenant-stage queue and workspace, EN/AM.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Two reviewers click Open on the same row | The first claim wins. The second gets `409 task_locked` and the workspace opens read-only with the holder's name |
| Reviewer's laptop sleeps | Heartbeats stop and the lock lapses after 10 min. On wake, the next action returns `409 lock_lost`, and the UI offers "Claim again" if the task is still open |
| Bank answers while a reviewer types a note | The task closes from `verification.finalized`. The workspace receives the status on its next heartbeat (the heartbeat response includes the task status) and disables actions. A late submit returns `409 task_closed` |
| Merchant cancels during review | Closed with outcome `cancelled`, the same way |
| Second approver unavailable for hours | The task stays `awaiting_second` and its SLA keeps running. Breach escalates to Review Leads (platform) or to the platform stage (tenant) |
| Tenant changes the review mode with tasks open | Open tasks keep their snapshot; new tasks use the new mode |
| Plan downgraded to Starter mid-day | New tasks get the 240 min SLA; existing tasks keep theirs |
| Staff member disabled while holding a lock | Their sessions are revoked by doc 01. The lock lapses within 10 min, or a Review Lead reassigns it immediately |
| Review-required event delivered twice | The unique open-task index makes the second insert a no-op (`ON CONFLICT DO NOTHING`) |
| Timer fires after the task is closed | It re-checks status under lock and does nothing |
| `applyReviewDecision` succeeds but the task commit fails | The verification is final. On retry, the lifecycle handler (on `verification.finalized`) sees `reviewed_by` and closes the task with the matching outcome, so no task stays open for a decided verification |
| Night with no reviewers on shift (Q6) | SLA timers still run and escalate. Merchants see `in_review` until decided or until `review.max_age_hours`, which expires it |

## 10. Security and privacy

| Endpoint | AuthN | AuthZ |
| --- | --- | --- |
| `/admin/v1/review/*` | Admin session | 6.3. Writes are scoped to the task's tenant transaction |
| `/portal/v1/review/*` | Portal session | `review.*`; tenant-stage tasks only; RLS |

- Receipt images come through 60 s signed URLs, and each view is audited (`receipt.viewed`).
- Unmasking the payer phone needs `platform.pii.reveal` or `pii.reveal` plus a recent re-auth, and is audited.
- **Audited actions:** claim, release, reassign, decide (with reason), second approval, escalate, QA verdict.
- Notes are free text; staff are told not to paste customer PII in notes. Notes are kept with the task and never sent to merchants or webhooks.
- Tenant reviewers never see other tenants' data. Cross-tenant signals appear as a flag only.

## 11. Performance

| Item | Target |
| --- | --- |
| Queue list | p95 under 100 ms (`tasks_platform_queue`) |
| Take next | p95 under 50 ms |
| Workspace | p95 under 400 ms (fan-out to docs 04, 05 and 06 in parallel) |
| Expected volume | Up to 1,000 tasks a day at launch scale (8% of 12,605 a day would be about 1,000); at most 50 open at once |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Transitions; eligibility matrix including four-eyes; priority ordering; comparison rows for each mockup field and result; SLA arithmetic |
| Integration | Task lifecycle with `FakeClock` (amber at 24 of 30 min; breach escalation; Hybrid handoff at 15 min then a 30 min platform SLA); 10 concurrent Take next; decision racing a bank answer (`task_closed`, task closed as `bank_answered`); single-tenant-admin high-value escalation; overturned flag after a doc 05 reversal; QA sampling bounds |
| End to end | Admin reviewer claims via Take next, sees the workspace as in the mockup, approves a high-value task, and a Review Lead confirms. Tenant Hybrid flow in the portal |
| Amharic | Portal review screens in `am`; reason labels for the reject picker |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_review_open_tasks` | gauge | `stage`, `status` |
| `pl_review_time_to_decision_seconds` | histogram | `stage`, `outcome` |
| `pl_review_sla_breaches_total` | counter | none |
| `pl_review_overturned_total` | counter | none |
| `pl_review_decisions_total` | counter | `stage`, `decision`, `reason` |
| `pl_review_lock_lapses_total` | counter | none |

| Alert | Condition | Severity |
| --- | --- | --- |
| Backlog under 10 min | More than 5 platform tasks with `due_at < now + 10 min` and unclaimed | Warning (Review Lead) |
| SLA breaches | More than 3 breaches in 1 h | Critical |
| Overturn | Any `overturned_total` increase | Warning |
| Oldest open task | Older than 2 h | Warning |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `REVIEW_AMBER_RATIO` | 0.8 | Amber point |
| `REVIEW_LOCK_MINUTES` | 10 | Lock length |
| `REVIEW_HEARTBEAT_SECONDS` | 60 | Client heartbeat |
| `REVIEW_MAX_IMAGE_REQUESTS` | 2 | Per verification |
| `REVIEW_QA_SAMPLE_PCT`, `_MIN`, `_MAX` | 5, 5, 50 | QA sampling |
| `REVIEW_SANDBOX_DELAY_SECONDS` | 30 | Test-env auto-decision |

**Per-tenant settings** (doc 01): `review.mode`, `review.hybrid_tenant_window_minutes`, `review.second_approval_above`, `review.max_age_hours`. **Plan values** (doc 12): `review_sla_minutes`.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 07-Q1 | Review desk hours (overview Q6). Does the SLA pause outside staffed hours? | No pause; timers run 24/7 |
| 07-Q2 | Liability wording for manual approvals (overview Q7) | Terms say a manual approval is a judgement on the receipt, reversible within 72 h |
| 07-Q3 | Should tenants be able to require second approval for *all* manual approvals, regardless of amount? | Set `review.second_approval_above = "1.00"` (the registry's minimum, doc 01), which covers every real payment |
| 07-Q4 | Should platform reviewers be allowed to contact the payer (for example by phone)? | No; decisions use the receipt and bank facts only |
