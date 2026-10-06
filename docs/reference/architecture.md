> **Reference only.** Exported from the Claude Docs version of 4 October 2026 (https://claude.ai/code/artifact/b1798dd4-d875-4b2e-8dbd-8715e423c147). It predates the accepted decisions. Where it differs from `docs/00-overview.md` or `docs/features/`, those win.

# Payment Verifier SaaS — Architecture & Spec

4 October 2026

## Summary

We are building a multi-tenant SaaS that tells a merchant, in seconds, whether an Ethiopian bank or wallet payment really happened and really reached them. Clients send a transaction reference or a receipt image/PDF; we fetch the official record from the bank, normalise it, check it against what the merchant expected, and return one consistent JSON shape. When the bank cannot be reached, we ask for the receipt and route it to a human reviewer.

First release covers CBE, telebirr, Awash, Dashen, Bank of Abyssinia (BoA), CBE Birr and M-Pesa, built on NestJS and PostgreSQL, verifying through the banks' public receipt-lookup pages.

Five decisions shape everything else:

- **The bank's record is the only source of truth.** AI reads an uploaded receipt only to find the bank and reference. Amounts and names printed on an image are never trusted as fact; they feed cross-checks and the reviewer's screen.
- **Verify the receiver, not just the transaction.** Each tenant registers the accounts they collect into. A genuine receipt paid to someone else is rejected, and payer details are only returned when the receiver matches. This also stops the API becoming a lookup tool for strangers' payments.
- **One receipt, one claim.** A bank reference can be claimed once per tenant, keyed on the reference the bank returns rather than what the client typed.
- **Run inside Ethiopia.** telebirr and M-Pesa receipt pages often block foreign IPs, and the 2024 data protection law requires local storage of personal data. All bank traffic and all personal data stay in-country.
- **Assurance is explicit.** Every verified result says how it was confirmed: `bank_confirmed` or `manual_review`. Manual approvals keep being re-checked against the bank and can be reversed.

Screen-by-screen UI spec and mockups:

## What the research changed

Four findings move the design away from "call a bank API with a txn number".

| Finding | What it means for us |
| --- | --- |
| **CBE changed platforms.** New CBE receipts are shared as `mbreciept.cbe.com.et` links and QR tokens served by a JSON API on `mb.cbe.com.et`. Some open-source verifiers report the old FT-number + 8-digit-suffix PDF service as decommissioned; others still support both ([cheki-mobile](https://github.com/L3von36/cheki-mobile), [cbe-verifier](https://github.com/Zahir-Seid/cbe-verifier), [VIP-bot](https://github.com/brexfolio/VIP-bot)). | A bare FT number may not be verifiable. For CBE the receipt QR or share link becomes the primary input, and an FT-only request returns `receipt_required`. Phase 0 must confirm live which path still works. |
| **Geo-restriction.** telebirr's receipt page often blocks or times out foreign IPs; M-Pesa is reported to behave the same. Projects hosted abroad fall back to relays or residential proxies inside Ethiopia ([verifier-api](https://github.com/hai-png/verifier-api), [verify-payment](https://github.com/sanitech/verify-payment)). | Bank egress must originate in Ethiopia, ideally from two ISPs. |
| **Flaky endpoints.** telebirr connections from an in-country host failed at the TLS handshake on about half of flows in one recent investigation ([verifier-api PR #5](https://github.com/hai-png/verifier-api/pull/5)). CBE's API intermittently returns HTTP 400 for valid tokens; Awash uses a self-signed certificate ([cheki-mobile](https://github.com/L3von36/cheki-mobile)). | Retries on fresh connections, per-host TLS trust, and a clear split between "bank down" and "receipt not found". |
| **Data localisation law.** Proclamation 1321/2024 entered into force on 24 July 2024. It requires personal data collected locally to be stored on servers in Ethiopia, registration with the Authority, and conditions on cross-border transfers ([Digital Policy Alert](https://digitalpolicyalert.org/change/12100-data-localisation-requirements-in-personal-data-protection-proclamation-proclamation-no-13212024), [proclamation text](https://www.metaappz.com/References/ethiopian_laws/federal/pr_1321_2024/en/txt)). | In-country hosting for databases and receipt storage. Sending receipt images to a foreign AI API is a cross-border transfer and needs a legal basis. |

**The basic lookup is already commoditised.** Verify.ET, Check.et, Veritas and BirrVerify sell multi-bank verification, Check.et offering 200 free checks and unlimited Pro at 499 ETB/month, BirrVerify at 1,500 ETB per 3 months ([Check.et](https://check.et/), [BirrVerify](https://birrverify.com/), [Veritas](https://veritas.et/), [Verify.ET](https://verify.et/)). Free open-source libraries cover most banks. We win on what they do thinly: receiver binding and fraud controls, managed manual review with SLAs, reliability, compliance, and an enterprise-grade API.

## Scope

The first release ships all seven providers with reference and receipt verification, the review queue and the tenant portal; billing automation and tenant self-review follow.

| Area | MVP (first release) | Phase 2 | Phase 3 |
| --- | --- | --- | --- |
| Providers | CBE, telebirr, BoA, Dashen, Awash, CBE Birr, M-Pesa, each with canary probes | Health-based routing across egress nodes | Zemen, Wegagen, Amhara, Siinqee, eBirr family; official bank APIs as agreements land |
| Inputs | Reference, share link, receipt image (JPG/PNG/WebP), receipt PDF | Batch verification (CSV / array), pasted SMS text | Email-forwarded receipts |
| Checks | Receiver binding, amount, date window, duplicate claim | Cross-tenant fraud signals, image re-use detection | Risk score per verification |
| Fallback | `receipt_required` + hosted upload link, platform review queue | Tenant self-review mode, four-eyes approval, continuous re-verification with reversal | Reviewer QA sampling and accuracy scoring |
| Clients | REST API, webhooks, sandbox keys | TypeScript, PHP and Python SDKs | Merchant mobile app for counter staff |
| Portal & admin | API keys, receiving accounts, verification log, review queue, roles, audit log | Billing and invoices, usage dashboards, status page | SSO for enterprise tenants |

**Non-goals.** We do not move or hold money, so we are not a payment gateway. We do not store full payer account numbers, only masked forms. A verified receipt proves a transfer was recorded; it does not guarantee against later bank-side reversals, and the terms of service must say so.

## System architecture

Two NestJS APIs take requests; all slow or failure-prone work runs in queue workers; and only the egress gateway talks to the outside world, through in-country lines.

> Diagram in the live doc: system architecture · clients, in-country services, external endpoints

The APIs also read and write PostgreSQL directly for idempotency keys and results. Each worker type scales on its own, so a telebirr outage that backs up its queue never slows CBE checks or webhook delivery.

## Verification lifecycle

Most verifications finish in one bank call. A human enters only when the bank cannot answer and a receipt is in hand, or when tenant policy asks for review.

> Diagram in the live doc: verification lifecycle · 7 statuses drawn, cancelled noted in the key

- **Bank answers:** all checks pass, so `verified` with `assurance: bank_confirmed`; any check fails, so `rejected` with a reason.
- `not_found`** is not final at once.** It is retried a few times over the first minutes in case a bank publishes receipts with a delay; Phase 0 measures the real delay per bank.
- **Transient errors** get 3 retries inside the client's wait window. If the bank's circuit breaker is open, the request goes straight to `receipt_required`.
- **Input too thin** (e.g. a bare CBE FT number) also goes to `receipt_required`, with `reason: reference_insufficient`.
- **Receipt uploaded:** extraction runs, then the bank is tried again. Still down, so `in_review`.
- **Time limits:** `receipt_required` expires after 24 hours by default (tenant setting); a hosted upload link lives 30 minutes and can be reissued. An `in_review` task past its SLA escalates in hybrid mode, otherwise expires.
- **After a manual approval** the bank is re-tried for 72 hours; a contradiction moves the verification to `reversed`.

## Bank adapter layer

Each provider is a plug-in adapter behind one interface, so a bank changing its page, or signing an official API deal with us, touches one module. Inputs differ per bank, and the API must ask for exactly what each needs.

| Provider (code) | Client sends | Endpoint as published by open-source verifiers | Format | Quirks to design for |
| --- | --- | --- | --- | --- |
| CBE, new (`CBE`) | Share link or QR token from the receipt | `mb.cbe.com.et/api/v1/transactions/public/transaction-detail/{id}` | JSON | Token is opaque; the official FT reference comes back in the response. Intermittent HTTP 400 on valid tokens |
| CBE, legacy (`CBE`) | FT reference (12 chars) + last 8 account digits | `apps.cbe.com.et:100/?id={FT}{suffix}` | PDF | Reported decommissioned; sources disagree whether the suffix is the payer's or receiver's. Keep behind a feature flag |
| telebirr (`TELEBIRR`) | Receipt number (10 chars, e.g. `CHQ0FJ403O`) or link | `transactioninfo.ethiotelecom.et/receipt/{ref}` | HTML | Geo-restricted; TLS handshake failures |
| Bank of Abyssinia (`BOA`) | FT reference + last 5 digits of the receiving account, or the receipt QR | `cs.bankofabyssinia.com/api/onlineSlip/getDetails/?id={ref}{last5}` | JSON | We fill the 5 digits from the tenant's registered account. QR payload is encrypted |
| Dashen (`DASHEN`) | 16-character reference | `receipt.dashensuperapp.com/receipt/{ref}` | PDF | Compressed PDF streams |
| Awash (`AWASH`) | Share link or token | `awashpay.awashbank.com:8225/-{token}` | HTML | Self-signed TLS; the leading dash is part of the token |
| CBE Birr (`CBEBIRR`) | Receipt number + payer phone (`2519XXXXXXXX`) | `cbepay1.cbe.com.et/aureceipt?TID={ref}&PH={phone}` | HTML | Needs the payer's phone |
| M-Pesa (`MPESA`) | Transaction number | `m-pesabusiness.safaricom.et/api/receipt/getReceipt?trxNo={ref}` | JSON | Geo-restricted |

Sources: [cheki-mobile](https://github.com/L3von36/cheki-mobile), [Verify.ET input fields](https://verify.et/verify), [telebirr-receipt](https://github.com/TheRealYT/telebirr-receipt), [cbe-verifier](https://github.com/Zahir-Seid/cbe-verifier). These are unofficial endpoints and must be re-tested in Phase 0.

### Adapter contract

```ts
export interface BankAdapter {
  readonly code: BankCode;            // 'CBE' | 'TELEBIRR' | 'BOA' | ...
  readonly parserVersion: string;     // stored on every attempt
  readonly allowedHosts: string[];    // SSRF allowlist for this adapter
  inputSpec(): InputField[];          // drives GET /v1/banks and validation
  parseInput(raw: RawInput, ctx: TenantCtx): CanonicalInput | InputError;
  fetch(input: CanonicalInput, http: EgressClient): Promise<RawEvidence>;
  parse(evidence: RawEvidence): NormalizedReceipt;
  classify(err: unknown, evidence?: RawEvidence): FailureClass;
}

type FailureClass =
  | 'not_found'       // bank answered: no such receipt
  | 'transient'       // timeout, 5xx, TLS reset -> retry
  | 'blocked'         // 403 or geo-block -> egress fault, page on-call
  | 'format_changed'  // 200 but parser failed -> parser drift, page on-call
  | 'input_invalid';  // fails the bank's own reference format
```

`parseInput` accepts whatever the customer has: a bare reference, a full link, a pasted SMS, or a decoded QR payload. It detects the bank when the client did not say, and rejects any URL whose host is not on the adapter's allowlist.

`NormalizedReceipt` is the same for every bank: official reference, amount, fees, total, currency, payer name, masked payer account or phone, receiver name, masked receiver account, `paid_at` in ISO 8601 with +03:00, status, narrative and channel. Raw strings are kept beside parsed values for reviewers.

### Resilience rules

- **One queue per bank.** A slow bank never delays the others; each queue has its own concurrency cap and rate limit so we stay polite and avoid being blocked.
- **Retry only what can recover.** `transient` gets 3 attempts with jittered backoff on a fresh connection. `blocked` and `format_changed` alert on-call immediately and never retry blindly.
- **Per-host TLS trust.** Pin the certificate or CA bundle per bank host. Never disable certificate checks process-wide, as some open-source examples do.
- **Circuit breaker per bank.** When a bank's error rate trips the breaker, new requests skip the wait and go straight to `receipt_required`, with background retries scheduled.
- **Canary probes.** Every few minutes, verify our own known-good test references through every egress node. The results drive the breaker, the status page and on-call alerts.
- **Fixtures as tests.** Every adapter ships with saved real responses (anonymised). A bank changing its markup fails CI before it fails customers.
- **Headless browser only if forced.** If a bank ever requires JavaScript rendering, run Playwright in a separate, isolated worker pool, never in the API process.

## Receipt intake and AI extraction

Extraction has one job: turn a receipt file into a candidate bank and reference for the normal bank lookup. It tries cheap, exact methods first and calls a vision model only when they fail.

1. **Intake and sanitise.** Accept JPG, PNG, WebP and PDF up to 10 MB and 5 pages. Check magic bytes, not extensions; virus-scan; re-encode images to strip EXIF/GPS and polyglot payloads. Parse PDFs in a sandboxed worker with scripting disabled.
2. **Fingerprint.** Store SHA-256 and a perceptual hash. A file already submitted to this tenant, or a near-identical image, is flagged as a reused screenshot.
3. **QR and barcode decode.** Most digital receipts carry a QR: CBE share links, BoA's encrypted payload, telebirr SuperApp codes, Awash and Dashen links. A clean decode gives bank and reference with near-certainty.
4. **PDF text layer.** Bank-issued PDFs (Dashen, legacy CBE) have selectable text, so per-bank patterns extract the reference without OCR. PDF producer and modification metadata are recorded as tamper signals.
5. **Vision model.** Screenshots and phone photos without a readable QR go to a vision model with a strict JSON schema: bank, reference, receipt URL, amount, payer, receiver, masked receiver account, date, and a confidence per field. The model must cope with Amharic labels.
6. **Validate and expand.** Each reference is checked against its bank's format. If it fails, generate likely misread variants (O/0, I/1, B/8, S/5, Z/2) and try at most a handful against the bank.
7. **Hand off.** The candidate enters the normal verification path. Fields read from the image are stored as `claimed_*` values and compared with the bank's record; a mismatch is a tamper signal. No candidate, or the bank is down, means manual review with these fields pre-filled.

### Model choice and data protection

The vision step sits behind an `ExtractionProvider` interface with two kinds of implementation: a hosted vision API (Claude, Gemini, GPT or Mistral) or an open-weight vision model on GPUs inside Ethiopia. Receipt images carry names and account numbers, so sending them abroad is a cross-border transfer under Proclamation 1321/2024 and needs a lawful basis plus a processing agreement. Legal counsel should pick the default; the provider is a per-tenant setting so enterprise tenants can require in-country processing.

- **Prompt injection.** Text on a receipt can be aimed at the model. Its output is untrusted data: schema-validated, only the bank and reference are used to query anything, and the model has no tools.
- **Cost.** QR and PDF paths handle digital receipts for free. Results are cached by file hash, and cost per extraction is recorded so image checks can be priced as credits.
- **Accuracy gate.** Keep a labelled, anonymised set of a few hundred real receipts per bank. Any model or prompt change must match or beat the current reference accuracy on it before release.

## Fraud and integrity controls

A receipt that exists at the bank is not enough; it must be paid to this merchant, for this amount, recently, and only once. Each check below runs on every verification and returns a specific reason code.

| Attack | Control | Result |
| --- | --- | --- |
| Edited or fabricated screenshot | Only the bank's record counts; image fields are compared with it | `rejected` / `not_found` or `amount_mismatch`, tamper flag |
| Genuine receipt paid to someone else | Receiver binding: the receiver's visible account digits and normalised name must match one of the tenant's registered receiving accounts | `rejected` / `receiver_mismatch`, payer details withheld |
| Same receipt used for two orders | Claim table, unique on tenant + bank + official reference | `rejected` / `duplicate`, with the first verification's ID |
| Old receipt reused for a new order | `paid_at` must fall inside the tenant's window (default: after `expected.paid_after`, max 24 h old) | `rejected` / `stale` |
| Underpayment | Amount must equal or exceed `expected.amount`, per tenant policy, with an optional tolerance | `rejected` / `amount_mismatch` |
| Failed transfer shown as success | Bank status must read completed where the bank exposes one | `rejected` / `not_completed` |
| Same screenshot sent again | Exact and perceptual image hash match | Flag, routed to review |
| Guessing references to harvest names | Payer data only on receiver match; rate limit on `not_found`; alert on a high not-found ratio per API client | Throttled, tenant notified |
| Same receipt at several merchants | Hashed reference seen across tenants | Fraud signal only, no data shared between tenants |
| Collusive or careless reviewer | Four-eyes above an amount threshold, sampled QA, full audit trail | Second approval required |

**Claim on the bank's reference, not the client's input.** A CBE payment can arrive as an opaque share token one day and as an FT number the next. Deduplicating on the official reference returned by the bank catches both. Re-sending the same `external_id` with the same reference is idempotent and returns the original result, not a duplicate.

**Receiving accounts are verified at onboarding.** A tenant adds each account it collects into, per bank. Ownership is proven by a bank letter or a micro-deposit of a random small amount that the tenant confirms. Banks mask account numbers differently, so each adapter declares which digits it exposes, and matching compares only those.

## Public API

One resource, the verification, created by reference or by receipt and settled by webhook. A rejected payment is a normal `200` response with `status: rejected`; HTTP errors mean the request itself was wrong.

### Authentication

The "username + password + API key" idea becomes an **API client** with a client ID and a secret, created in the tenant portal and shown once.

- **Default:** `Authorization: Bearer pvk_live_<key_id>.<secret>`. The `key_id` prefix finds the record; only a hash of the secret is stored.
- **Enterprise option:** OAuth2 client credentials. `POST /v1/oauth/token` with client ID and secret returns a 15-minute access token.
- **Per client:** scopes (`verifications:write`, `verifications:read`, `receipts:write`, `banks:read`), optional IP allowlist, live or test mode, last-used time, instant revoke and rotation with overlap.

### Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/v1/verifications` | Verify by reference (JSON) or by receipt file (multipart) |
| GET | `/v1/verifications/{id}` | Current state and result |
| GET | `/v1/verifications` | List by `external_id`, `status`, date range |
| POST | `/v1/verifications/{id}/receipt` | Upload a receipt for a verification in `receipt_required` |
| POST | `/v1/verifications/{id}/upload-link` | New hosted upload link to send the end customer |
| POST | `/v1/verifications/{id}/cancel` | Cancel while not yet final |
| GET | `/v1/banks` | Providers, required fields per provider, live health |
| POST | `/v1/oauth/token` | Client-credentials token (enterprise) |

Every POST takes an `Idempotency-Key` header, kept for 24 hours. The same key with a different body returns `409`.

### Request

```json
POST /v1/verifications
{
  "bank_code": "TELEBIRR",
  "reference": "CHQ0FJ403O",
  "payer_phone": null,
  "account_suffix": null,
  "expected": {
    "amount": "1500.00",
    "receiving_account_id": "racc_01JA2B3C4D5E6F7G8H9J0K1M2N",
    "paid_after": "2026-10-04T09:00:00+03:00"
  },
  "external_id": "order_88231",
  "metadata": { "branch": "Bole" },
  "wait_seconds": 10
}
```

`reference` accepts a bare reference, a share link or pasted SMS text. `bank_code` is optional when the input identifies the bank. The multipart form takes `file` plus the same fields. `wait_seconds` (0 to 15) holds the connection for a final answer; otherwise the response is `202` with the current status.

### Response

```json
{
  "id": "ver_01JB7Q8K2M4N6P8R0T2V4X6Z8A",
  "status": "verified",
  "assurance": "bank_confirmed",
  "reason": null,
  "external_id": "order_88231",
  "bank": { "code": "TELEBIRR", "name": "telebirr" },
  "transaction": {
    "reference": "CHQ0FJ403O",
    "amount": "1500.00",
    "fee": "0.00",
    "total": "1500.00",
    "currency": "ETB",
    "paid_at": "2026-10-04T10:12:44+03:00",
    "payer": { "name": "ABEBE KEBEDE", "phone_masked": "2519****1234" },
    "receiver": { "name": "SAMPLE TRADING PLC", "account_masked": "****5678",
                  "receiving_account_id": "racc_01JA2B3C4D5E6F7G8H9J0K1M2N" },
    "narrative": "Order 88231"
  },
  "checks": { "receiver": "pass", "amount": "pass", "date_window": "pass",
              "duplicate": "pass", "bank_status": "pass" },
  "evidence": { "source": "bank_receipt_page", "fetched_at": "2026-10-04T10:13:02+03:00" },
  "next_action": null,
  "created_at": "2026-10-04T10:13:00+03:00",
  "completed_at": "2026-10-04T10:13:02+03:00"
}
```

When the bank cannot be reached, the same object returns with `status: receipt_required`, `reason: bank_unavailable` and a `next_action` holding a hosted `upload_url` (valid 30 minutes) and the API upload path. Amounts are decimal strings; times are ISO 8601 in East Africa Time.

### Statuses and reasons

| Status | Meaning | Final | Client should |
| --- | --- | --- | --- |
| `processing` | Queued or fetching from the bank | No | Wait for the webhook |
| `receipt_required` | Bank unreachable, or the input cannot be verified alone (e.g. CBE FT number) | No | Upload the receipt or show `upload_url` to the customer |
| `in_review` | A reviewer is checking it | No | Show "payment under review" |
| `verified` | Confirmed; `assurance` says how | Yes | Release goods |
| `rejected` | Failed a check; `reason` says which | Yes | Do not release |
| `reversed` | A manual approval later contradicted by the bank | Yes | Hold or claw back the order |
| `expired` | Receipt never arrived, or review timed out | Yes | Ask the customer to retry |
| `cancelled` | Cancelled by the client | Yes | Nothing |

Reason codes: `not_found`, `receiver_mismatch`, `amount_mismatch`, `duplicate`, `stale`, `not_completed`, `invalid_reference`, `unreadable_receipt`, `suspected_forgery`, `manual_reject`, `bank_contradicted`; and for `receipt_required`: `bank_unavailable`, `reference_insufficient`, `bank_format_changed`.

### Errors

Errors return `{ "error": { "code", "message", "field", "request_id" } }`: `400 missing_field` (e.g. `payer_phone` for CBE Birr), `401 unauthenticated`, `402 quota_exceeded`, `403 forbidden_scope`, `404 not_found`, `409 idempotency_conflict`, `413 file_too_large`, `415 unsupported_file_type`, `422 unsupported_bank`, `429 rate_limited` with `Retry-After`.

### Webhooks

- Events: `verification.verified`, `verification.rejected`, `verification.receipt_required`, `verification.in_review`, `verification.reversed`, `verification.expired`.
- Signed with `PV-Signature: t=<unix>,v1=<HMAC-SHA256 of t.body>`; receivers reject timestamps more than 5 minutes old.
- At-least-once delivery with an event ID for deduplication, retried with backoff for 24 hours, with a delivery log and replay button in the portal.

### Sandbox

Test-mode keys (`pvk_test_`) never call a bank. Magic references return fixed outcomes: `TEST_VERIFIED`, `TEST_NOT_FOUND`, `TEST_BANK_DOWN`, `TEST_DUPLICATE`, `TEST_AMOUNT_MISMATCH`, `TEST_REVIEW_APPROVE`. The OpenAPI 3.1 document is generated from the NestJS controllers and published as the docs site.

## Manual review and admin panel

Reviewers decide only what the bank could not. Every decision is recorded, high-value approvals need two people, and the bank keeps being re-checked after a human says yes.

### Who reviews

A platform reviewer can only judge the receipt itself, while the merchant's own staff can check their bank statement. So review mode is a per-tenant setting.

| Mode | Reviewer | Strength | Billing |
| --- | --- | --- | --- |
| Platform | Our review team | Consistent, can run long hours; judges the receipt only | Charged per reviewed item |
| Tenant | Tenant's reviewers in their portal | Strongest: can check the actual account statement | Included |
| Hybrid | Tenant first, escalates to platform after the SLA | Coverage outside the tenant's hours | Per escalated item |

### Review workflow

1. **Task created** when the bank stays unavailable after a receipt upload, no reference can be extracted, a parser reports `format_changed`, or tenant policy demands it (e.g. every payment above a set amount).
2. **Queued** by SLA due time, then amount, then plan tier. SLA targets are set per plan.
3. **Claimed** by one reviewer with a 10-minute lock that auto-releases, so two people never work the same task.
4. **Reviewed** in a workspace showing: the receipt with zoom and rotate, image fields beside the merchant's expected values, the bank's health at request time, earlier verifications sharing the image hash, payer or reference, tamper signals, and a "retry bank now" button.
5. **Decided:** approve (becomes `verified` with `assurance: manual_review`), reject with a reason code, request a clearer image (back to `receipt_required`), or escalate. Rejections and high-value approvals require a note.
6. **Second approval** by a Review Lead above the tenant's amount threshold.
7. **Re-checked afterwards.** The scheduler keeps trying the bank for 72 hours. A bank confirmation upgrades the assurance to `bank_confirmed`; a contradiction moves it to `reversed`, fires a webhook and alerts the tenant.

### Admin panel modules

Dashboard (volume, outcomes, per-bank health) · Review queue · Verification explorer · Tenants (KYB, plan, limits, receiving-account approval) · Banks (adapter on/off, breaker state, canary results, egress nodes) · Staff and roles · Audit log · Billing · Settings.

### Roles

| Role | Side | Can | Cannot |
| --- | --- | --- | --- |
| Super Admin | Platform | Everything, including role management and break-glass access | Edit or delete audit records |
| Ops Admin | Platform | Manage tenants, plans, limits, bank adapters, queue settings | Change staff roles |
| Review Lead | Platform | Assign tasks, decide, give second approvals, run QA sampling, see reviewer metrics | Change tenant settings |
| Reviewer | Platform | Claim and decide tasks under the high-value threshold | Export data; see receipts outside assigned tasks |
| Support | Platform | Read verification metadata and webhook logs | See receipt images or payer details by default |
| Finance | Platform | Plans, invoices, payment records | See verification content |
| Auditor | Platform | Read audit log and review decisions | Change anything |
| Engineer (on-call) | Platform | Bank health, parse failures, raw bank responses via time-limited break-glass | Decide reviews |
| Owner | Tenant | Everything in the tenant, incl. billing and closing the account | Act on other tenants |
| Admin | Tenant | Users, receiving accounts, review policy, webhooks | Billing |
| Developer | Tenant | API clients, webhooks, request logs | Approve receiving accounts |
| Reviewer | Tenant | Decide review tasks (tenant or hybrid mode) | Manage keys or users |
| Viewer | Tenant | Read verifications and reports | Change anything |
| Cashier | Tenant | Run checks from the Verify screen and mobile app; see their own history | See other users' checks, settings or keys |

Roles are bundles of permission strings (`review.decide`, `review.decide.high_value`, `evidence.view_raw`, `apiclient.manage`, `audit.read`…), enforced by a NestJS guard plus tenant-scoped row filters. Staff sign in with SSO and mandatory WebAuthn or TOTP; the admin panel is reachable only from allowlisted networks. Every view of a receipt image is itself an audit event.

## Data model

Core tables live in PostgreSQL, every tenant-owned row carrying `tenant_id` under row-level security. Money is `NUMERIC(14,2)`, times are `timestamptz`, and IDs are UUIDv7 shown with prefixes in the API (`ver_`, `racc_`).

| Table | Key columns | Notes |
| --- | --- | --- |
| `tenants` | name, plan, review_mode, kyb_status, settings | Settings hold date window, amount policy, high-value threshold |
| `users` | email, kind (platform / tenant), mfa, status | One table for staff and tenant users |
| `memberships` | user, tenant (null = platform), role | A user can belong to several tenants |
| `roles`, `role_permissions` | role, permission string | Seeded system roles plus custom tenant roles later |
| `api_clients` | key_id (unique), secret_hash, mode, scopes, ip_allowlist, last_used_at, revoked_at | Secret shown once, never stored |
| `receiving_accounts` | bank, account_number (encrypted), visible digits, normalised holder name, verified_at | Basis of receiver binding |
| `verifications` | external_id, idempotency_key, bank, input kind, official_reference, status, assurance, reason, expected, result, checks | Partitioned by month; result PII encrypted |
| `verification_attempts` | adapter, parser_version, egress_node, http_status, latency, failure_class | One row per bank call |
| `evidence` | kind (bank response / upload), storage key, sha256, phash, mime, retention_until | Files live in object storage |
| `extractions` | method (qr / pdf_text / vision), provider, model, fields, confidence, cost | One row per extraction run |
| `claims` | tenant, bank, official_reference, verification, ref_hash | Unique per tenant; `ref_hash` feeds cross-tenant signals |
| `review_tasks` | status, priority, sla_due_at, assignee, locked_until, decision, reason, note, second_approver | One open task per verification |
| `upload_links` | token_hash, verification, expires_at, used_at | Hosted upload page for end customers |
| `webhook_endpoints`, `webhook_deliveries` | url, secret (encrypted), events; attempt, status code, next_retry_at | Delivery log and replay |
| `usage_events` | tenant, client, kind, billable, occurred_at | Append-only; drives invoices |
| `bank_health` | bank, egress node, outcome, latency, checked_at | Canary results |
| `audit_log` | actor, action, target, ip, at, prev_hash | Append-only, hash-chained, no update or delete grants |

The duplicate check is a database constraint, not application logic, so two concurrent requests cannot both win:

```sql
CREATE TABLE claims (
  tenant_id          uuid        NOT NULL,
  bank_code          text        NOT NULL,
  official_reference text        NOT NULL,
  verification_id    uuid        NOT NULL,
  ref_hash           bytea       NOT NULL,  -- HMAC(bank_code || reference), for cross-tenant signals
  claimed_at         timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, bank_code, official_reference)
);
CREATE INDEX claims_ref_hash_idx ON claims (ref_hash);
```

The claim insert and the move to `verified` run in one transaction; a conflict on the primary key turns the result into `rejected` / `duplicate`.

## Security, privacy and compliance

We hold names, phone numbers and account fragments for thousands of people, so the design treats this as financial data from day one. The legal items below need confirmation from Ethiopian counsel before launch.

### Data protection law (Proclamation 1321/2024)

- **Register** with the Authority as a data processor, and as a controller for the fraud signals we keep ([registration requirement](https://digitalpolicyalert.org/event/24926-implemented-personal-data-protection-proclamation-proclamation-no-13212024-including-registration-requirements)).
- **Store in Ethiopia.** Databases, object storage, backups and logs containing personal data stay on in-country servers.
- **Appoint a DPO and run a DPIA** before launch; large-scale processing of payment data is the textbook high-risk case ([compliance guide](https://www.mondaq.com/privacy-protection/1766200/ethiopian-personal-data-protection-law-proclamation-13212024-business-compliance-guide)).
- **Controller and processor roles.** Each tenant is the controller of its customers' payment data; we process it under a data processing agreement. Cross-tenant fraud signals use only hashed references and are disclosed in our privacy notice.
- **Nothing leaves the country by accident.** Foreign AI APIs, error trackers and email providers are all cross-border transfers ([transfer rules](https://digitalpolicyalert.org/event/24914-adopted-personal-data-protection-proclamation-proclamation-no-13212024-including-data-transfer-regulation)). Self-host observability; scrub PII from anything that must leave.
- **Rights and breaches.** Processes for access and erasure requests, and a written breach-notification runbook.
- **Bank and regulator stance.** Get a written opinion on the banks' website terms and on whether National Bank of Ethiopia rules touch a verification-only service, and open partnership talks early.

### Security controls

| Area | Control |
| --- | --- |
| Encryption | TLS 1.2+ everywhere; encrypted disks; envelope encryption for account numbers, phones and payer names, keys in an in-country Vault or HSM with per-tenant data keys |
| Secrets | API secrets stored as peppered SHA-256 hashes (they are high-entropy); webhook secrets encrypted; rotation with overlap |
| Outbound calls | Only adapter-allowlisted hosts; no redirects off the list; 5 MB response cap; strict timeouts; deny-by-default egress proxy |
| Uploads | Sanitised as in the extraction section; reviewers see files through short-lived signed URLs; no public buckets |
| Tenant isolation | Row-level security plus an application guard, with automated tests that attempt cross-tenant reads |
| Staff access | SSO, mandatory MFA, network allowlist, re-authentication for sensitive actions, time-limited break-glass |
| Logs | No PII in application logs; request IDs everywhere; audit log separate and hash-chained |
| API output | Payer accounts and phones masked; a receiver's full account number is never returned |
| Assurance | OWASP ASVS level 2 as the target, dependency scanning in CI, external penetration test before launch |

**Retention defaults (configurable per tenant):** receipt files 90 days, raw bank responses 30 days, normalised results for the tenant's contract term. Confirm these against tenants' own record-keeping obligations.

## Infrastructure, reliability and observability

Production runs in an Ethiopian data centre with bank traffic leaving through at least two egress nodes on different ISPs, so one blocked IP or one carrier outage never stops verification.

### Deployment

- **Hosting:** in-country colocation or cloud; evaluate local data-centre operators on power redundancy, carrier choice and a signed data-processing agreement. Two physical sites if the budget allows; otherwise one site plus in-country offsite backups.
- **Runtime:** containers on a small Kubernetes cluster (k3s is enough to start). The API, each worker type and the egress gateway scale independently.
- **Egress nodes:** static IPs on Ethio Telecom and Safaricom Ethiopia lines, health-checked by the canaries. Adapters choose a node by recent success per bank.
- **Data:** PostgreSQL primary plus streaming replica with point-in-time recovery; Redis with persistence and a replica for queues; S3-compatible object storage (MinIO) with erasure coding.

### Service objectives

| Objective | Target |
| --- | --- |
| API availability, excluding bank outages | 99.9% monthly |
| Reference verification, bank healthy | p95 under 5 seconds end to end |
| Webhook first delivery | p95 under 60 seconds after the status change |
| Manual review | Per-plan SLA, with breach alerts at 80% of the window |

### Observability

- **Traces:** OpenTelemetry from API request through queue, adapter and webhook, keyed by verification ID.
- **Metrics:** per-bank success rate, latency and failure class; queue depth; review backlog and SLA breaches; extraction method mix and cost.
- **Dashboards and alerts:** Grafana with alerting to the on-call phone. `blocked` and `format_changed` page immediately.
- **Errors and logs:** self-hosted error tracking and log storage, so stack traces with customer data never leave the country.
- **Status page:** public per-bank status driven by canaries, so tenants see "telebirr degraded" before they open a ticket.

Runbooks exist before launch for: a bank changing its receipt format, egress being geo-blocked, a review backlog surge, and a leaked API secret.

## SaaS model

Charge for confirmed outcomes, not attempts: a tenant pays when we deliver a final answer, never for our failures or a bank's outage. Entry pricing must sit near the market (Check.et Pro is unlimited for 499 ETB/month; BirrVerify is 1,500 ETB per 3 months), with margin earned on volume, manual review and enterprise features.

### Plans (proposal)

| Plan | For | Included per month | API rate | Review | Extras |
| --- | --- | --- | --- | --- | --- |
| Sandbox | Developers | Unlimited test calls, 50 live verifications | 10 req/min | Tenant mode | Full API and webhooks |
| Starter | Shops, counters | Unlimited portal checks; 1,000 API verifications, 100 receipt images | 60 req/min | Tenant mode | Portal verify screen, 2 receiving accounts |
| Growth | E-commerce, delivery, schools | 10,000 verifications, 1,000 receipt images, overage per item | 300 req/min | Tenant, or platform per item | Multiple branches, 99.9% SLA |
| Enterprise | Large merchants, institutions | Custom volume | Custom | Platform review with SLA and four-eyes | OAuth, IP allowlist, SSO, dedicated egress IP, signed DPA, invoicing |

### Metering rules

- **Billable:** a verification that ends `verified` or `rejected`. Idempotent replays, test mode, `expired` and `cancelled` are free.
- **Receipt images** consume image credits, because the vision step costs money. QR and PDF-text paths may later be made free to steer customers to digital receipts.
- **Platform review** is a separate per-item charge; tenant self-review is included.
- **Enforcement:** counters in Redis, warnings at 80% and 100%, then overage or a hard stop per plan. Invoices are built from `usage_events`, never from counters.
- **Collection:** invoices in ETB with VAT. Self-serve plans pay by telebirr or bank transfer, verified by our own API; enterprise pays against invoice.

### Onboarding and tenant portal

Sign up and verify email or phone, get sandbox keys at once, then submit KYB documents (trade licence, TIN), add receiving accounts and prove ownership. Live keys are issued only after both checks pass.

The portal holds: an overview dashboard; a **verify screen** for counter staff who paste a reference or snap a receipt without any integration; the verification log searchable by order ID and reference; the review queue in tenant mode; receiving accounts; API clients; webhooks with delivery log and replay; team and roles; usage and invoices; policy settings (date window, amount rule, review mode, high-value threshold, retention); and live bank status.

## Codebase and tech stack

One TypeScript monorepo (pnpm + Turborepo): three NestJS apps, one Next.js app, and shared libraries where all bank and verification logic lives, so the API and workers never duplicate rules.

```text
apps/
  api/          NestJS public API (/v1), API-client auth, idempotency, quotas
  console-api/  NestJS backend for admin panel and tenant portal (separate deploy, allowlisted)
  worker/       BullMQ processors: verify, extract, review timers, webhooks, canaries
  web/          Next.js: admin panel, tenant portal, hosted receipt-upload page
libs/
  verification/ orchestrator, state machine, checks (receiver, amount, date, duplicate)
  banks/        one folder per adapter + saved fixtures: cbe, telebirr, boa, dashen,
                awash, cbebirr, mpesa
  egress/       undici client, per-host TLS trust, node selection, breakers, rate limits
  extraction/   QR, PDF text, vision providers, reference validators
  auth/         API keys, OAuth client credentials, RBAC guards, permission catalogue
  db/           Drizzle schema, SQL migrations, row-level security policies
  billing/  webhooks/  audit/  domain/ (shared types, money, errors)
  sdk-ts/       generated from the OpenAPI document
```

**State changes go through one function.** The orchestrator checks an explicit transition table, updates the row with an optimistic version check, and writes an outbox event in the same transaction. Webhooks are sent from the outbox, so a crash can delay one but never lose it.

```ts
const transitions: Record<Status, Status[]> = {
  processing:       ['verified', 'rejected', 'receipt_required', 'in_review', 'cancelled'],
  receipt_required: ['processing', 'expired', 'cancelled'],
  in_review:        ['verified', 'rejected', 'receipt_required', 'expired', 'cancelled'],
  verified:         ['reversed'],   // only when assurance = 'manual_review'
  rejected: [], reversed: [], expired: [], cancelled: [],
};
```

| Concern | Choice | Why |
| --- | --- | --- |
| Framework | NestJS on the Fastify adapter | Modules, guards and interceptors fit a multi-tenant API; Fastify for throughput |
| Database access | Drizzle ORM with hand-written SQL migrations | Partitioning and row-level security are easier SQL-first than through Prisma |
| Queues | BullMQ on Redis | Per-bank queues, rate limiters, delayed retries, repeatable canary jobs |
| HTTP to banks | undici with a per-host Agent | Certificate pinning per host and fresh-connection retries |
| Parsing | cheerio (HTML), pdfjs-dist in a worker thread (PDF), zxing-wasm (QR), sharp (images) | Pure-Node, no external OCR service needed for QR and PDF paths |
| Validation | zod schemas shared with the Next.js app | One definition for API, workers and forms |
| API docs | @nestjs/swagger generating OpenAPI 3.1 | Docs and SDKs from the code |
| Front end | Next.js, React, TanStack Table, a headless component kit | Fast admin tables and a small hosted upload page |
| Staff identity | Self-hosted OIDC provider (e.g. Keycloak) with WebAuthn | SSO and MFA without sending staff data abroad |
| Keys | HashiCorp Vault transit engine | Envelope encryption with per-tenant keys |
| Tests | Jest or Vitest, adapter fixture tests, Testcontainers for Postgres and Redis, Playwright for the portal | Bank changes fail CI, not production |

## Delivery roadmap

Pilot tenants can be live 12 weeks from kickoff: a 2-week discovery phase that proves every bank works from inside Ethiopia, then a 10-week MVP. Paid launch follows a 6-week scale-up.

> Diagram in the live doc: delivery roadmap · 4 phases, 3 gates

Durations assume four to five engineers (two backend, one frontend, one full-stack with DevOps) plus a part-time designer, and are re-estimated at the Build gate. If Phase 0 finds a bank unreachable, that adapter moves to Phase 3 rather than delaying the pilot.

## Risks and open questions

The biggest risk is structural: we depend on unofficial bank pages that can change or close without notice. Everything else is manageable with the controls above.

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| A bank changes or closes its receipt endpoint | High | One provider down until fixed | Isolated adapters, canaries, fixture tests, receipt fallback, partnership talks with each bank |
| Legal challenge to commercial use of bank pages | Medium | Could stop a provider or the service | Written legal opinion, polite rate limits, proactive approach to banks, adapter contract ready for official APIs |
| Geo-blocking or carrier instability | High | Slow or failed checks | Two-ISP egress, breakers, immediate `receipt_required` fallback |
| Data protection non-compliance | Medium | Fines, forced shutdown, lost enterprise deals | In-country hosting, DPIA, DPO, registration before launch |
| Wrong manual approval | Medium | Merchant loss, disputes | Explicit assurance level, four-eyes, 72-hour re-checks with reversal, liability cap in terms |
| Price war in a commoditised market | High | Thin margins | Sell outcomes the cheap tools lack: receiver binding, review SLAs, compliance, enterprise API |
| Vision model cost or misreads | Medium | Margin and accuracy | QR and PDF first, accuracy gate, image credits |
| API abused to harvest payer names | Medium | Privacy breach, reputation | Payer data only on receiver match, not-found throttling, anomaly alerts |

### Decisions needed from you

- Default review mode: our reviewers, the tenant's staff, or hybrid?
- Hosting: which Ethiopian data centre or cloud, and is a second site in budget?
- Vision step: hosted AI API (needs a legal basis for the transfer) or a self-hosted model in-country?
- Liability: what do we promise on manual approvals in the terms of service?
- Test accounts: do we hold accounts at all seven providers for test payments and canaries?
- Product name and domain, used in upload links, key prefixes and webhook headers.
- Team size and target launch date, which set the roadmap below.

## Sources

Consulted on 4 October 2026. Bank endpoints come from unofficial open-source projects and must be re-tested live in Phase 0.

- [cheki-mobile (Mahtem) README](https://github.com/L3von36/cheki-mobile): endpoint, format and quirk table for all seven providers
- [cbe-verifier (Go)](https://github.com/Zahir-Seid/cbe-verifier): CBE legacy and new platforms
- [VIP-bot](https://github.com/brexfolio/VIP-bot): report that the CBE legacy PDF endpoint is dead
- [AdamaRent issue #3](https://github.com/AntenehTilaye/AdamaRent/issues/3): open question on the CBE account suffix
- [verifier-api](https://github.com/hai-png/verifier-api) and [its PR #5](https://github.com/hai-png/verifier-api/pull/5): geo-restriction and telebirr TLS failures
- [verify-payment](https://github.com/sanitech/verify-payment): in-country relays and residential proxies
- [telebirr-receipt](https://github.com/TheRealYT/telebirr-receipt): telebirr receipt URL
- [Verify.ET input fields](https://verify.et/verify), [Verify.ET](https://verify.et/), [Check.et](https://check.et/), [Veritas](https://veritas.et/), [BirrVerify](https://birrverify.com/): competitor features and pricing
- [Digital Policy Alert: data localisation](https://digitalpolicyalert.org/change/12100-data-localisation-requirements-in-personal-data-protection-proclamation-proclamation-no-13212024), [registration](https://digitalpolicyalert.org/event/24926-implemented-personal-data-protection-proclamation-proclamation-no-13212024-including-registration-requirements), [cross-border transfers](https://digitalpolicyalert.org/event/24914-adopted-personal-data-protection-proclamation-proclamation-no-13212024-including-data-transfer-regulation)
- [Proclamation 1321/2024 text](https://www.metaappz.com/References/ethiopian_laws/federal/pr_1321_2024/en/txt) and [Mondaq compliance guide](https://www.mondaq.com/privacy-protection/1766200/ethiopian-personal-data-protection-law-proclamation-13212024-business-compliance-guide)

