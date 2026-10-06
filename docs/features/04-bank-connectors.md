# 04 · Bank connectors

| | |
| --- | --- |
| Package | `@proofline/bank-connectors` |
| Postgres schema | `bank_connectors` |
| Depends on | platform only |
| Build step | 4 (can start during Phase 0, in parallel with steps 1–3) |
| Mockups | Admin · Dashboard bank-health strip and alerts; Portal · Overview outage banner and bank status list; Portal · Verify bank chips ("M-Pesa · down"); review workspace "Bank record unavailable" banner. Admin Banks & egress is not drawn. |

> **Warning: every bank endpoint in this doc is unofficial.** They come from open-source projects that read the banks' public receipt pages (sources in the architecture doc). Base URLs, input formats and parsers are configuration and code that **Phase 0 must confirm with live test payments from an in-country host** (overview Q1, Q2) before any adapter ships. Nothing here assumes a bank agreement exists.

## 1. Purpose and scope

**Does**

- **Bank registry:** which banks exist, what input each needs, and how to recognise a bank from a share link, SMS text, QR payload or bare reference.
- **Input parsing** into a canonical lookup, including the "this input can't be looked up" outcome (`reference_insufficient`, R5).
- **Fetching** one attempt from the bank through an in-country egress node. That covers per-host TLS trust, timeouts, size caps and a few fresh-connection retries inside the attempt.
- **Parsing** the bank response into a `NormalizedReceipt` with a versioned parser per bank.
- **Classifying failures:** `transient`, `blocked`, `format_changed`, `input_invalid`.
- **Circuit breakers** per bank, plus egress-node health and selection.
- **Canary checks** every 2 minutes from each egress node.
- **Evidence storage:** the raw bank response, encrypted, for 30 days.
- **The sandbox connector** that answers in test mode without calling any bank.
- `GET /v1/banks` and the admin bank-management API.

**Out of scope**

- Whether a payment passes. Checks, claims and statuses belong to doc 05.
- Retries *between* attempts and their timing; doc 05 schedules every attempt.
- Reading receipt images (doc 06), though doc 06 calls `BankRegistry.detect()` and `validateReference()`.

## 2. Screens and user stories

| Screen | Surface | What this module supplies |
| --- | --- | --- |
| Dashboard bank-health strip (Healthy, Degraded, Down; Answered 1 h; p95; breaker note) | Admin | `BankHealth.snapshot()` |
| Recent alerts (breaker opened, node slow, parser deployed) | Admin | Events `bank.*` |
| Banks & egress (per-bank detail, breaker control, canaries, nodes) | Admin, not drawn | Admin endpoints (section 5) |
| Outage banner, bank status list | Portal | `GET /v1/banks` status, also used by doc 09 |
| Bank chips with "· down" | Portal Verify, cashier | `GET /v1/banks` |

**US-04.1 Detect**

- Given `https://transactioninfo.ethiotelecom.et/receipt/CHQ0FJ403O`, a pasted SMS containing it, or `CHQ0FJ403O` with `bank_code = TELEBIRR`, `detect()` or `parseInput()` returns `TELEBIRR` with reference `CHQ0FJ403O`.
- A bare 10-character reference with no `bank_code` is ambiguous between telebirr and M-Pesa, so it gets `422 bank_not_detected`.

**US-04.2 Insufficient input**

- A bare CBE `FT…` reference returns `{ kind: 'insufficient', reason: 'reference_insufficient' }` while `BANK_CBE_LEGACY_ENABLED = false`. Doc 05 maps it to `receipt_required`.

**US-04.3 Required extras**

- `CBEBIRR` without `payer_phone` returns `400 missing_field` with `field = payer_phone`. `BOA` needs no extra field from the caller; doc 05 supplies the receiving-account suffixes.

**US-04.4 Fetch and normalise**

- For each bank, a recorded "found" fixture produces exactly the expected `NormalizedReceipt` JSON, and a "not found" fixture produces `not_found`.

**US-04.5 Classify**

- With a mocked bank:
  - HTTP 403 → `blocked`;
  - HTTP 503, timeout or TLS reset → `transient`;
  - 200 with an unrecognised body → `format_changed`;
  - the bank's own "no such transaction" response → `not_found`;
  - HTTP 400 → `input_invalid`, except CBE's known intermittent 400 (section 6.4).

**US-04.6 Breaker**

- When at least 50% of at least 20 calls in 60 s fail for one bank, the breaker opens. `isCallable()` turns false, `bank.breaker_changed` is emitted, and the admin dashboard shows the bank as Down with "Breaker open since HH:MM".
- After the open period, the breaker goes half-open and probes. 2 successful probes close it.

**US-04.7 Canary**

- Every 120 s, each active canary reference is fetched through each egress node and compared with its expected values.
- A canary failing on both nodes for 10 min pages on-call. A bank with no canary reference shows "Canary not configured".

**US-04.8 Manual control**

- A platform Admin can force a breaker open or closed for 1–1440 minutes with a reason, disable an adapter, or drain an egress node. Each action is audited and visible on the dashboard within 30 s.

**US-04.9 Parser version**

- Each adapter declares `parserVersion`. When a worker boots with a new version, `parser_deployments` records it, and the dashboard shows "Parser 1.4.2 live", as in the mockup.

**US-04.10 Sandbox**

- In test env, the magic references in section 6.8 return their documented results for every bank code, and no network call is made. CI asserts that egress nodes receive zero requests during the test-env suite.

## 3. Module design

```text
libs/features/bank-connectors/src/
├─ public/
│  ├─ index.ts                BankConnectorsModule, BankRegistry, BankGateway, BankHealth, types, events
│  ├─ registry.ts  gateway.ts  health.ts  types.ts  events.ts
├─ domain/
│  ├─ bank-codes.ts           re-export of contracts BankCode plus display names
│  ├─ detection.ts            URL, QR and SMS patterns; ambiguity rules
│  ├─ classification.ts       HTTP or exception → FailureClass (pure)
│  ├─ breaker.ts              state machine (pure, clock injected)
│  ├─ misread.ts              OCR misread variants
│  └─ normalized-receipt.ts   type plus invariants check
├─ adapters/
│  ├─ adapter.ts              BankAdapter interface (below)
│  ├─ cbe/  telebirr/  boa/  dashen/  awash/  cbebirr/  mpesa/
│  │   ├─ index.ts            adapter (inputSpec, parseInput, buildRequest, parse, classifyBody)
│  │   └─ parser.ts           response → NormalizedReceipt
│  └─ sandbox/index.ts        test-env connector
├─ application/
│  ├─ gateway.service.ts      breaker check → node selection → adapter.fetch → classify → record → evidence
│  ├─ health.service.ts       snapshot, status derivation
│  ├─ canary.processor.ts     queue "canary"
│  ├─ breaker.service.ts      Redis Lua transitions, persistence, events
│  └─ admin.service.ts        manual controls, canary CRUD
├─ infrastructure/
│  ├─ egress-client.ts        undici ProxyAgent per node; TLS trust per host
│  ├─ node-selector.ts        EWMA scores per (bank, node)
│  ├─ evidence.store.ts       ObjectStore "bank-evidence" plus evidence table
│  ├─ samples.buffer.ts       batched health_samples inserts
│  └─ repos/*.repo.ts
├─ http/public-api/banks.controller.ts, http/admin/*.controller.ts
├─ db/schema.ts, db/migrations/
└─ test/fixtures/{bank}/{parserVersion}/{case}.{html|json|pdf} with {case}.expected.json
```

### Adapter contract

```ts
export interface BankAdapter {
  readonly code: BankCode;
  readonly parserVersion: string;                 // semver, bumped on any parser change
  readonly allowedHosts: readonly string[];       // URL builder and share-link parser refuse anything else
  readonly tls: { caFile?: string; pinSha256?: string[]; rejectUnauthorized: boolean };
  readonly keepAlive: boolean;                    // false for TELEBIRR (dead TLS flows)
  inputSpec(): InputSpec;
  detect(raw: string): DetectedInput | null;      // links, QR payloads, SMS fragments, bare refs
  parseInput(i: RawBankInput): ParsedInput;       // throws AppError invalid_reference / missing_field
  buildRequests(c: CanonicalBankInput, ctx: FetchContext): BankHttpRequest[];  // >1 for BOA suffix candidates
  parse(res: BankHttpResponse): ParseOutcome;     // found | not_found | unrecognised
  isBenignRetryStatus?(status: number, attemptTry: number): boolean;  // CBE 400 quirk
}
export type ParseOutcome =
  | { kind: 'found'; receipt: NormalizedReceipt }
  | { kind: 'not_found' }
  | { kind: 'unrecognised'; why: string };        // becomes format_changed
```

### Public interfaces

```ts
export interface BankRegistry {
  list(): BankInfo[];
  get(code: BankCode): BankInfo;                                   // throws 422 unsupported_bank
  detect(raw: string): DetectedInput | null;
  parseInput(i: { bankCode?: BankCode; reference: string; payerPhone?: string }): ParsedInput;
  validateReference(code: BankCode, ref: string): boolean;
  misreadVariants(code: BankCode, ref: string, max?: number): string[];  // default max 4, excludes ref itself
  notFoundRechecks(code: BankCode): number[];                       // seconds; config, default []
}
export interface BankGateway {
  fetch(input: CanonicalBankInput, ctx: FetchContext): Promise<BankResult>;  // never throws for bank problems
}
export interface BankHealth {
  isCallable(code: BankCode): Promise<boolean>;     // false when breaker open, forced open or adapter disabled
  state(code: BankCode): Promise<BreakerView>;
  snapshot(): Promise<BankHealthView[]>;            // cached 30 s
  publicStatus(code: BankCode): Promise<'operational' | 'degraded' | 'down'>;
}

export interface DetectedInput {                   // result of detect(); used by docs 05, 09 and 10
  bankCode: BankCode;
  kind: 'reference' | 'share_link' | 'sms_text' | 'qr_payload';
  displayReference: string;                        // normalised reference to show (uppercased where the bank is case-insensitive)
  needsPayerPhone: boolean;                        // true for CBEBIRR
}
export interface FetchContext {
  tenantId: string | null; env: EnvName; verificationId: string | null; attemptNo: number;
  source: 'live' | 'canary';
  boaSuffixes?: string[];                          // from ReceivingAccountsApi via doc 05
  forceNodeId?: string;                            // canaries only
  sandbox?: { expectedAmount: Money; receiver: { name: string; digits: string } | null };  // test env only
}
export type ParsedInput =
  | { kind: 'lookup'; canonical: CanonicalBankInput }
  | { kind: 'insufficient'; bankCode: BankCode; reason: 'reference_insufficient'; displayReference: string };
export interface CanonicalBankInput {
  bankCode: BankCode;
  lookupKind: 'cbe_token' | 'cbe_legacy' | 'telebirr_receipt' | 'boa_slip' | 'dashen_ref' | 'awash_token' | 'cbebirr_receipt' | 'mpesa_trx';
  ref: string;                                      // canonical form (uppercased where the bank is case-insensitive)
  payerPhone?: string;                              // CBEBIRR
  legacySuffix?: string;                            // CBE legacy only
  displayReference: string;                         // what the merchant sees
  inputKind: 'reference' | 'share_link' | 'sms_text' | 'qr_payload';
}
export type BankResult =
  | { outcome: 'found'; receipt: NormalizedReceipt; evidenceId: string | null; parserVersion: string; nodeId: string; latencyMs: number }
  | { outcome: 'not_found'; evidenceId: string | null; parserVersion: string; nodeId: string; latencyMs: number }
  | { outcome: 'failure'; failureClass: FailureClass; detail: string; evidenceId: string | null; nodeId: string | null; latencyMs: number };
export type FailureClass = 'transient' | 'blocked' | 'format_changed' | 'input_invalid' | 'circuit_open' | 'disabled';

export interface NormalizedReceipt {
  officialReference: string;            // the bank's own transaction ID; used for claims (doc 05)
  status: 'completed' | 'failed' | 'pending' | 'unknown';
  creditedAmount: Money;                // amount the receiver got (R6)
  fee: Money | null;
  totalDebited: Money | null;
  paidAt: Date;                         // naive timestamps are read as Africa/Addis_Ababa
  payer: { name: string | null; digits: Digits | null; phone: string | null };   // phone full E.164 when shown
  receiver: { name: string | null; digits: Digits | null };                       // → ReceiverFacts (doc 02)
  narrative: string | null;
  channel: string | null;
  rawFields: Record<string, string>;    // label → value as printed, at most 40 entries, each at most 200 chars
}
export interface Digits { value: string; position: 'last' | 'first' | 'full' }
```

## 4. Data model

```sql
CREATE TABLE bank_connectors.adapter_state (
  bank_code      text PRIMARY KEY,
  enabled        boolean NOT NULL DEFAULT true,
  disabled_reason text NULL,
  updated_by     uuid NULL,
  updated_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE bank_connectors.egress_nodes (
  id          text PRIMARY KEY,               -- 'et-egress-1'
  isp         text NOT NULL,                  -- 'ethio_telecom', 'safaricom'
  drained     boolean NOT NULL DEFAULT false,
  drained_reason text NULL,
  updated_by  uuid NULL,
  updated_at  timestamptz NOT NULL DEFAULT now()
);  -- rows upserted at boot from EGRESS_NODES; proxy URLs live only in config

CREATE TABLE bank_connectors.breaker_events (
  id          uuid PRIMARY KEY,
  bank_code   text NOT NULL,
  from_state  text NOT NULL CHECK (from_state IN ('closed','half_open','open')),
  to_state    text NOT NULL CHECK (to_state IN ('closed','half_open','open')),
  cause       text NOT NULL CHECK (cause IN ('failure_ratio','consecutive_blocked','format_changed','probe_ok','probe_failed','timer','forced','cleared')),
  forced_by   uuid NULL,
  reason      text NULL,
  at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX breaker_events_bank_time ON bank_connectors.breaker_events (bank_code, at DESC);

CREATE TABLE bank_connectors.canary_references (
  id                 uuid PRIMARY KEY,
  bank_code          text NOT NULL,
  input_enc          text NOT NULL,          -- Crypto.encrypt('platform', JSON of RawBankInput)
  expected_reference text NOT NULL,
  expected_amount_minor bigint NOT NULL CHECK (expected_amount_minor > 0),
  expected_receiver_digits text NULL,
  active             boolean NOT NULL DEFAULT true,
  created_by         uuid NOT NULL,
  created_at         timestamptz NOT NULL DEFAULT now(),
  last_ok_at         timestamptz NULL
);
CREATE INDEX canary_active ON bank_connectors.canary_references (bank_code) WHERE active;

CREATE TABLE bank_connectors.health_samples (
  at             timestamptz NOT NULL,
  bank_code      text NOT NULL,
  node_id        text NULL,
  source         text NOT NULL CHECK (source IN ('live','canary')),
  outcome        text NOT NULL CHECK (outcome IN ('found','not_found','failure')),
  failure_class  text NULL,
  http_status    smallint NULL,
  latency_ms     integer NOT NULL,
  parser_version text NOT NULL
) PARTITION BY RANGE (at);                    -- daily partitions, 30 days kept
CREATE INDEX health_samples_bank_time ON bank_connectors.health_samples (bank_code, at DESC);

CREATE TABLE bank_connectors.evidence (
  id              uuid PRIMARY KEY,
  tenant_id       uuid NULL,                  -- null for canaries
  env             text NOT NULL,
  verification_id uuid NULL,
  attempt_no      smallint NULL,
  bank_code       text NOT NULL,
  node_id         text NULL,
  http_status     smallint NULL,
  content_type    text NULL,
  size_bytes      integer NOT NULL,
  sha256          bytea NOT NULL,
  object_key      text NOT NULL,              -- bucket "bank-evidence"
  parser_version  text NOT NULL,
  classification  text NOT NULL,              -- found | not_found | failure class
  fetched_at      timestamptz NOT NULL DEFAULT now(),
  expires_at      timestamptz NOT NULL        -- fetched_at + 30 days (canary: 7 days)
);
CREATE INDEX evidence_verification ON bank_connectors.evidence (verification_id);
CREATE INDEX evidence_expiry ON bank_connectors.evidence (expires_at);

CREATE TABLE bank_connectors.parser_deployments (
  bank_code     text NOT NULL,
  version       text NOT NULL,
  first_seen_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (bank_code, version)
);
```

- **RLS:** `evidence` has the tenant and env policy, with an additional policy so `system` principals can read canary rows (`tenant_id IS NULL`). The other tables are platform-level and hold no tenant data.
- **Redis:** breaker state lives in `pl:bank:{code}:breaker`, a hash with fields `state`, `since`, `open_until`, `trips`, `probes_left`, `probe_ok`, `forced`, `forced_until`, `reason`. Rolling counters live in `pl:bank:{code}:w:{10s-bucket}` (TTL 120 s). Node scores live in `pl:bank:{code}:node:{id}` (TTL 600 s).
- **Migrations:** `0001_bank_connectors_init.sql`, `0002_health_samples_partitions.sql`, `0003_seed_adapter_state.sql` (7 rows, enabled).

## 5. API contracts

### Public: `GET /v1/banks`

Any valid key (R13). Also served unauthenticated, cached 30 s, to the portal and the upload page through their BFFs.

```json
{ "data": [
  { "code": "TELEBIRR", "name": "telebirr", "status": "degraded",
    "inputs": [
      { "name": "reference", "required": true, "pattern": "^[A-Z0-9]{10}$",
        "hint": { "en": "Receipt number from the telebirr SMS, e.g. CHQ0FJ403O", "am": "ከቴሌብር SMS የደረሰኝ ቁጥር፣ ለምሳሌ CHQ0FJ403O" } }
    ],
    "accepts_share_link": true, "accepts_sms_text": true },
  { "code": "CBEBIRR", "name": "CBE Birr", "status": "operational",
    "inputs": [
      { "name": "reference", "required": true, "pattern": "^[A-Z0-9]{8,12}$", "hint": { "en": "…", "am": "…" } },
      { "name": "payer_phone", "required": true, "pattern": "^251[79][0-9]{8}$", "hint": { "en": "Payer's phone, 2519XXXXXXXX", "am": "…" } }
    ], "accepts_share_link": false, "accepts_sms_text": true }
] }
```

`status` is `down` when the breaker is open (forced or not) or the adapter is disabled. It is `degraded` when answered-ratio over 1 h is below 95% or p95 latency is above 5 s. Otherwise it is `operational`.

### Admin (`/admin/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /banks` | `platform.banks.read` | none | `200 { data: BankHealthView[] }` (the dashboard strip) | none |
| `GET /banks/{code}` | `platform.banks.read` | none | `200` detail: breaker, answered ratio and p95 over 1 h and 24 h, failure-class counts, last canary per node, parser version, recent breaker events (20) | `404` |
| `POST /banks/{code}/breaker` | `platform.banks.manage` | `{ action: "force_open" \| "force_close" \| "clear", reason, minutes? }` (`minutes` 1–1440, required for force actions) | `200 BreakerView` | `400 invalid_field` |
| `POST /banks/{code}/enabled` | `platform.banks.manage` | `{ enabled, reason }` | `200` | none |
| `GET /banks/{code}/canaries` | `platform.banks.read` | none | `200 { data: CanaryView[] }` (no decrypted input) | none |
| `POST /banks/{code}/canaries` | `platform.banks.manage` | `{ reference, payer_phone?, expected_reference, expected_amount, expected_receiver_digits? }` | `201 CanaryView` (runs it once immediately, result included) | `422 canary_failed` (first run didn't match expected) |
| `DELETE /banks/{code}/canaries/{id}` | `platform.banks.manage` | none | `204` | none |
| `POST /banks/{code}/canaries/run` | `platform.banks.manage` | none | `200 { results: [{ node_id, outcome, latency_ms }] }` | none |
| `GET /egress-nodes` | `platform.banks.read` | none | `200` nodes with per-bank score, last error and drained flag | none |
| `POST /egress-nodes/{id}/drain` | `platform.banks.manage` | `{ drained, reason }` | `200` | `409 last_node` (can't drain every node) |
| `POST /evidence/{id}/access` | `platform.evidence.break_glass` | `{ reason }` (10–500 characters) | `200 { url, expires_at }` (60 s); audited `evidence.break_glass` | `404` |

### New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `canary_failed` | 422 | A new canary reference didn't return its expected values |
| `last_node` | 409 | Draining would leave no egress node |

### Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `bank.breaker_changed` | `{ bank_code, from, to, cause, forced }` | 05 (re-queue waiting verifications on `closed`), 11 (alerts feed) |
| `bank.format_changed` | `{ bank_code, parser_version, evidence_id }` | 11 (page on-call) |
| `bank.adapter_toggled` | `{ bank_code, enabled, reason }` | 11 |
| `bank.node_blocked` | `{ bank_code, node_id, until }` | 11 |
| `bank.parser_deployed` | `{ bank_code, version }` | 11 ("Parser 1.4.2 live") |

This module consumes no events.

## 6. Business rules and state machine

### 6.1 Adapters (initial; Phase 0 confirms each row)

| Bank | Accepted input | Request (base URL is config) | Format | "Not found" signal | Receiver exposure | Quirks |
| --- | --- | --- | --- | --- | --- | --- |
| `CBE` (new) | Share link `https://mbreciept.cbe.com.et/{token}`, the same link from the receipt QR, or a bare token `^[A-Za-z0-9]{10,64}$` not starting with `FT` | `GET {BANK_CBE_BASE_URL}/api/v1/transactions/public/transaction-detail/{token}` (default base `https://mb.cbe.com.et`) | JSON | HTTP 404, or JSON with no transaction object | Name and masked account (position confirmed in Phase 0) | Intermittent HTTP 400 for valid tokens (6.4) |
| `CBE` (legacy) | `^FT\d{5}[A-Z0-9]{5}$` plus 8-digit suffix | `GET {BANK_CBE_LEGACY_BASE_URL}/?id={FT}{suffix}` (default `https://apps.cbe.com.et:100`) | PDF | Non-PDF body | Name, last digits | **Disabled** (`BANK_CBE_LEGACY_ENABLED=false`); reported decommissioned. Bare FT returns `reference_insufficient` |
| `TELEBIRR` | `^[A-Z0-9]{10}$`; link `https://transactioninfo.ethiotelecom.et/receipt/{ref}`; SMS containing the link or "transaction number is {ref}" | `GET {BANK_TELEBIRR_BASE_URL}/receipt/{ref}` | HTML (label/value table) | Page without the receipt table | "Credited Party name", "Credited party account no" (masked) | Geo-restricted; about half of TLS handshakes fail from some hosts, so `keepAlive = false` and retries use a fresh connection |
| `BOA` | `^FT[0-9A-Z]{10}$` | `GET {BANK_BOA_BASE_URL}/api/onlineSlip/getDetails/?id={ref}{last5}`, tried once per suffix candidate (at most 5) | JSON | Empty or `null` details | Receiver account digits and name | Needs the receiving account's last 5 digits (supplied by doc 05). Encrypted receipt QR payloads are unsupported, so the user types the FT reference |
| `DASHEN` | `^[0-9]{3}[A-Z0-9]{13}$`; link `https://receipt.dashensuperapp.com/receipt/{ref}` | `GET {BANK_DASHEN_BASE_URL}/receipt/{ref}` | PDF (text layer via pdfjs) | HTTP 404 or non-PDF | "Receiver Name", "Receiver Account Number" | FlateDecode and ASCII85 streams |
| `AWASH` | Link `https://awashpay.awashbank.com:8225/-{token}`; token `^-?[A-Za-z0-9]{8,64}$` | `GET {BANK_AWASH_BASE_URL}/-{token}` | HTML | Error page | Receiver name and account | Self-signed certificate, pinned by SHA-256 (`BANK_AWASH_TLS_PIN_SHA256`). The leading `-` is part of the token and is added if missing |
| `CBEBIRR` | `^[A-Z0-9]{8,12}$` **plus** `payer_phone` `^251[79]\d{8}$` | `GET {BANK_CBEBIRR_BASE_URL}/aureceipt?TID={ref}&PH={phone}` | HTML | No receipt block | Receiver name and phone or account | Payer phone required (R5) |
| `MPESA` | `^[A-Z0-9]{10}$` | `GET {BANK_MPESA_BASE_URL}/api/receipt/getReceipt?trxNo={ref}` | JSON | Empty result | Receiver name and short code | Geo-restricted |

**Parser obligations.** Every parser:

- returns `creditedAmount` from the amount the receiver got, which for telebirr is "Settled Amount", not "Total Paid Amount" (R6);
- parses timestamps without an offset as Africa/Addis_Ababa;
- returns `status` from the bank's status field when one exists, otherwise `unknown`;
- returns `unrecognised` if the amount, `officialReference` or `paidAt` is missing or unparsable, or the currency isn't ETB;
- trims and caps `rawFields` at 40 entries.

### 6.2 Detection

Order: URL host match → QR payload patterns → SMS text (extract URLs first, then bank-specific phrases) → bare reference patterns.

| Input shape | Result |
| --- | --- |
| URL whose host is in an adapter's `allowedHosts` | That bank; token taken from the path |
| URL on any other host | `null`, and the caller returns `422 bank_not_detected`. No fetch is ever made to an unlisted host |
| SMS text | Extract the first allowed URL. Else telebirr's "transaction number is ([A-Z0-9]{10})". Else `null` |
| Bare `^[0-9]{3}[A-Z0-9]{13}$` | `DASHEN` |
| Bare `^FT…` (12 chars) | Ambiguous between BoA and CBE legacy: needs `bank_code` |
| Bare `^[A-Z0-9]{10}$` | Ambiguous between telebirr, M-Pesa and CBE Birr: needs `bank_code` |

When the caller gives `bank_code` and the input is a URL for a different bank, the result is `400 invalid_field` with `field = reference` and the message "Link is for {other bank}".

### 6.3 Attempt execution (`BankGateway.fetch`)

1. **Availability.** If the adapter is disabled, return `failure/disabled`. If the breaker is open, or half-open with no probe slot left, return `failure/circuit_open`. No network call is made in either case.
2. **Test env.** If `ctx.env = test`, delegate to the sandbox connector (6.8) and return.
3. **Requests.** Build the request list (1 request; for BoA, 1 per suffix, at most 5).
4. **Execute** within an attempt budget of `BANK_ATTEMPT_BUDGET_MS` (8,000 ms):
   - select a node (6.6);
   - connect timeout 3,000 ms, headers 6,000 ms, body 6,000 ms;
   - abort and classify `format_changed` if the body exceeds 5 MB;
   - no redirects.
   - Connection-level errors (`ECONNRESET`, `ETIMEDOUT` on connect, `UND_ERR_SOCKET`, TLS handshake timeout) retry **on a fresh connection through the other node if one is healthy**, at most 2 retries, never past the budget.
   - HTTP 502, 503 or 504 retries once after 500 ms. HTTP 429 doesn't retry and is `transient`.
   - BoA: stop at the first `found`. Every suffix `not_found` means `not_found`.
5. **Classify** (6.4). Store evidence (6.7). Record the sample and the breaker outcome (6.5). Return the `BankResult`.

### 6.4 Classification

| Observation | Class |
| --- | --- |
| 2xx and the parser says `found` | `found` |
| 2xx and the parser says `not_found`, or 404 where the adapter treats 404 as not found (CBE, Dashen) | `not_found` |
| 2xx and the parser says `unrecognised`; or any 3xx; or a body over 5 MB | `format_changed` |
| 400 at CBE, tries 1–2 | Retry immediately on a fresh connection (the known quirk). After 3 consecutive 400s: `not_found` |
| 400 at any other bank, or 422 | `input_invalid`. Doc 05 treats it as "the bank has no such transaction" |
| 401, 403, 451, or a known block-page body | `blocked`, and the node is marked blocked for this bank for 10 min (`bank.node_blocked`) |
| 429, 5xx, connection errors, timeouts, proxy 502/503 | `transient` |
| Unexpected exception inside the adapter | `format_changed`, with the stack trace logged |

### 6.5 Circuit breaker (per bank, shared by all nodes and pods)

| State | Calls allowed | Leaves when | To |
| --- | --- | --- | --- |
| `closed` | All | In the rolling 60 s window: at least 20 calls and failures (`transient` plus `blocked`) at least 50%; **or** 5 consecutive `blocked`; **or** 3 `format_changed` within 10 min | `open` |
| `open` | None (`circuit_open`) | `open_until` passes. The open period is 60 s × 2^(trips−1), capped at 600 s; `trips` resets after 30 min closed | `half_open` |
| `half_open` | 3 probe calls (a canary runs immediately on entry; live calls can take the remaining slots) | 2 probe successes (`found`, `not_found` or `input_invalid`) | `closed` |
| `half_open` | (same) | Any probe failure | `open` (trips + 1) |
| any | Forced: an admin `force_open` or `force_close` until `forced_until` (at most 24 h) | Expiry or `clear` | The state the counters indicate |

- Transitions run in one Redis Lua script, so pods agree.
- Each transition inserts a `breaker_events` row and publishes `bank.breaker_changed` through the outbox (system context).
- If Redis is lost, every breaker starts `closed`, the safe default: calls proceed and real failures re-open it within a window.

### 6.6 Egress node selection

- Each `(bank, node)` keeps an EWMA success score (α = 0.2, starting at 1.0) where success means `found`, `not_found` or `input_invalid`.
- Pick the highest-scoring node that isn't drained and isn't blocked for this bank, with round-robin among ties within 0.05.
- If every node is blocked or drained for the bank, use the least recently blocked one; if all are drained, return `failure/transient` with detail `no_egress_node`.
- Canaries pass `forceNodeId` and always hit their node, so each node's health stays measured.

### 6.7 Evidence

- Every network response body, including error pages, is stored encrypted with the tenant key (canaries use the platform key) in bucket `bank-evidence`, key `t/{tenantId or "platform"}/{env}/{yyyy}/{mm}/{evidenceId}`.
- Retention: 30 days for live evidence and 7 days for canaries (`expires_at`). The daily maintenance job deletes expired objects and rows.
- If storing evidence fails, the result is still returned with `evidenceId = null`, and `pl_bank_evidence_write_failures_total` is incremented.
- Access is limited to `platform.evidence.break_glass` (60 s URL, reason required, audited) and the fixture-capture tool, which anonymises names, phones and accounts before writing a fixture.

### 6.8 Sandbox connector (test env)

The connector ignores the bank code and keys only on the magic reference. Every non-magic reference returns `not_found`. The list below lives in `libs/contracts/src/sandbox.ts` as `SANDBOX_REFERENCES` (reference, outcome, description keys), so the connector and the portal's Sandbox page (doc 09) can't drift apart.

| Reference | Result |
| --- | --- |
| `TEST_VERIFIED` | `found`: `creditedAmount` = expected amount; receiver = `ctx.sandbox.receiver`; `paidAt` = now − 1 min; status `completed`; `officialReference` = `TEST_VERIFIED-{last 8 of verification ID}` |
| `TEST_NOT_FOUND` | `not_found` |
| `TEST_BANK_DOWN` | `failure/transient` on every attempt (doc 05 takes the receipt path) |
| `TEST_FORMAT_CHANGED` | `failure/format_changed` |
| `TEST_DUPLICATE` | `found` with `officialReference = TEST_DUPLICATE-{tenantId}`. Doc 05 pre-seeds that claim on `tenant.created`, so the result is `rejected/duplicate` |
| `TEST_AMOUNT_MISMATCH` | `found` with credited = expected − 1.00 (at least 0.01) |
| `TEST_RECEIVER_MISMATCH` | `found` with receiver name `SOMEONE ELSE`, digits `0000` |
| `TEST_STALE` | `found` with `paidAt` = now − 48 h |
| `TEST_NOT_COMPLETED` | `found` with status `failed` |
| `TEST_SLOW` | `found` after a 20 s delay, so `wait_seconds = 15` returns `202` and a webhook follows |
| `TEST_REVIEW_APPROVE` | `failure/transient`. Doc 07's sandbox reviewer approves the resulting review task 30 s after it opens |
| `TEST_REVIEW_REJECT` | Same, but the sandbox reviewer rejects with `unreadable_receipt` |

Magic references are case-insensitive and are also accepted inside a URL path or an SMS text, so the upload-page flow can be tested.

### 6.9 Misread variants (used by doc 06)

`misreadVariants(code, ref)` substitutes the confusable pairs O↔0, I↔1, L↔1, B↔8, S↔5 and Z↔2, one position at a time and then two at a time. It keeps only variants that pass `validateReference(code, …)`, ordered by fewest substitutions, then left to right. It returns at most `max` (default 4) and never the input itself.

### 6.10 Numbers

| Rule | Value | Where |
| --- | --- | --- |
| Attempt budget | 8,000 ms | `BANK_ATTEMPT_BUDGET_MS` |
| Connect, headers, body timeouts | 3,000 / 6,000 / 6,000 ms | `BANK_CONNECT_TIMEOUT_MS`, `BANK_RESPONSE_TIMEOUT_MS` |
| Max body | 5,242,880 bytes | `BANK_MAX_BODY_BYTES` |
| Retries inside an attempt | 2 (connection-level), 1 (502, 503, 504) | Constants |
| Breaker window, minimum calls, failure ratio | 60 s, 20, 0.5 | `BANK_BREAKER_WINDOW_S`, `BANK_BREAKER_MIN_CALLS`, `BANK_BREAKER_FAILURE_RATIO` |
| Breaker open period | 60 s base, ×2 per trip, 600 s cap; trips reset after 30 min | `BANK_BREAKER_OPEN_BASE_S`, `BANK_BREAKER_OPEN_MAX_S` |
| Half-open probes | 3 slots, 2 successes needed | Constants |
| Node blocked duration | 10 min | `BANK_NODE_BLOCK_MINUTES` |
| Canary interval | 120 s per bank per node | `BANK_CANARY_INTERVAL_S` (brief: 2 min) |
| Per-bank rate limit | TELEBIRR 10, CBE 10, others 5 requests per second | `BANK_{CODE}_MAX_RPS` |
| Per-bank worker concurrency | 8 per pod | `BANK_{CODE}_CONCURRENCY` |
| Evidence retention | 30 days live, 7 days canary | `BANK_EVIDENCE_RETENTION_DAYS` |
| Not-found rechecks | `[]` (final on the first answer) per bank | `BANK_{CODE}_NOT_FOUND_RECHECKS` (comma-separated seconds; overview Q15) |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `ObjectStore` (`bank-evidence`), `Crypto`, `Db`, `Queues` (`bank-fetch.*` processors live in doc 05; the `canary` queue is here), `Outbox`, `AuditLog`, `ConfigService`, `Clock`, `Money` |
| Infra | Squid forward proxies on the egress nodes, with a `dstdomain` allowlist of exactly the adapters' `allowedHosts` |

No feature modules: doc 05 passes in everything bank-independent (BoA suffixes, sandbox context).

## 8. Implementation plan

1. **T04.1** Migrations, Redis key helpers, adapter-state seed.
2. **T04.2** Types and invariants checker for `NormalizedReceipt`, with tests.
3. **T04.3** `classification.ts`, a table-driven test for every row of 6.4.
4. **T04.4** `breaker.ts`, a pure state machine with `FakeClock` tests for every row of 6.5; then the Redis Lua implementation and a two-instance concurrency test.
5. **T04.5** Egress client: undici `ProxyAgent` per node, per-host TLS (custom CA and pin check), timeouts, size cap, no redirects. Tested against a local HTTPS server with a self-signed certificate (pin accepted and pin rejected) and a slow server (timeouts).
6. **T04.6** Node selector (EWMA, drained and blocked rules).
7. **T04.7** Gateway service (6.3 steps), evidence store, samples buffer.
8. **T04.8** Sandbox connector (6.8), with a test that no egress calls happen in test env.
9. **T04.9** Registry and detection (6.2), `misreadVariants` (6.9), `GET /v1/banks`.
10. **T04.10 to T04.16** One adapter per bank: input patterns, request builder, parser, fixtures (`found`, `not_found`, plus at least one format variant). **Each adapter task is gated on Phase 0 having captured real responses.** Order: TELEBIRR, CBE, BOA, DASHEN, CBEBIRR, MPESA, AWASH.
11. **T04.17** Canary processor, canary CRUD, the `parser_deployments` record at boot.
12. **T04.18** Health snapshot and public status derivation; admin endpoints and audit.
13. **T04.19** Fixture-capture tool: `pnpm fixtures:capture --evidence <id> --case <name>`, with anonymisation.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Bank returns a different reference than the input (CBE token → official FT number) | Normal. `officialReference` is what doc 05 claims, and the input stays as `displayReference` |
| Receipt shows a currency other than ETB | `unrecognised` → `format_changed` (we only handle ETB) |
| Amount printed as `1,500.00 Birr` or `ETB 1,500` | `Money.fromBankText` handles both. More than 2 decimal places → `unrecognised` |
| Date in the Ethiopian calendar on a receipt | The parser converts with `fromEthiopian`. If ambiguous → `unrecognised` |
| Receiver block missing on an otherwise valid receipt | `receiver.name` and `.digits` are null. Doc 02 returns `unverifiable` and doc 05 sends it to review |
| Both egress nodes down | `failure/transient` with detail `no_egress_node`. The breaker opens within a window, and doc 05 moves new verifications to `receipt_required` |
| A node silently routed abroad (geo-block appears) | The bank returns 403 or block pages, classified `blocked`, and the node is blocked for 10 min with an alert. The other node takes over |
| Bank throttles us (429) | `transient`. The per-bank limiter (`BANK_{CODE}_MAX_RPS`) is lowered via config and redeploy |
| Parser fails on 3 responses within 10 min | Breaker opens with cause `format_changed`, `bank.format_changed` pages on-call, and verifications go to `receipt_required/bank_format_changed` and then review |
| Canary reference ages out (the bank purges old receipts) | The canary returns `not_found` and is counted as a failure. The admin replaces it. The dashboard shows `last_ok_at` |
| Evidence bucket unavailable | The fetch still succeeds (`evidenceId = null`). An alert fires if failures exceed 1% for 10 min |
| BoA tenant with no verified BoA account | Never reaches this module: doc 05 refuses with `422 no_receiving_account` first |
| HTML in a non-UTF-8 charset | Decoded using the `Content-Type` charset, then `<meta charset>`, then UTF-8 |

## 10. Security and privacy

- **SSRF:** requests are built only from templates with validated tokens, the host must be in `allowedHosts`, redirects are disabled, and the egress Squid allows only those hosts. User-supplied URLs are parsed for their token and never fetched as given.
- **TLS:** verification is always on. Self-signed bank hosts are accepted only through an explicit SHA-256 pin. There is no process-wide `NODE_TLS_REJECT_UNAUTHORIZED=0`; a CI lint forbids it.
- **Evidence** is class P data: encrypted, retained 30 days, break-glass access only, audited.
- **Canary inputs** are encrypted with the platform key and never returned by the API.
- **Admin endpoints** need `platform.banks.*`; every manual action is audited with its reason.
- **User-Agent** is `BANK_USER_AGENT` (default `Proofline-Verifier/1.0`). Its policy is open question 04-Q4.

## 11. Performance

| Target | Value |
| --- | --- |
| Gateway overhead excluding the bank | p95 under 20 ms |
| Bank p95 (healthy) | Under 3 s for most banks; telebirr is the slowest observed (p95 6.2 s in the mockup scenario during degradation) |
| Peak load | About 15 live fetches per second across banks, plus 7 canaries per minute |
| `health_samples` writes | Batched every 1 s; daily partitions; queries always bounded to the last 24 h |
| Snapshot | Computed from Redis counters plus one 1-hour p95 query per bank, cached 30 s |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Classification table; breaker state machine; detection table (6.2, including ambiguity); misread variants; money and date parsing per bank |
| Fixtures | Per bank and parser version: `found` (all fields), `not_found`, `unrecognised`, plus regression cases. Expected output is compared field by field. CI fails if an adapter's `parserVersion` changes without a new fixture directory |
| Integration | Gateway against a local mock bank (undici `MockAgent` or an HTTP server) for every class; two-pod breaker agreement via shared Redis; node failover; TLS pin; evidence write and expiry; canary run → sample → breaker probe |
| Sandbox | Every magic reference; zero egress calls in test env |
| Live (manual, Phase 0 and pre-release) | `pnpm bank:smoke --bank TELEBIRR --ref …` from an in-country host, printing the normalised receipt |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_bank_fetch_total` | counter | `bank`, `outcome`, `failure_class`, `node`, `source` |
| `pl_bank_fetch_duration_seconds` | histogram | `bank`, `outcome`, `node` |
| `pl_bank_breaker_state` | gauge (0 closed, 1 half-open, 2 open) | `bank` |
| `pl_bank_canary_ok` | gauge (0 or 1) | `bank`, `node` |
| `pl_bank_format_changed_total` | counter | `bank`, `parser_version` |
| `pl_bank_evidence_write_failures_total` | counter | none |
| `pl_bank_node_score` | gauge | `bank`, `node` |

| Alert | Condition | Severity |
| --- | --- | --- |
| Breaker open | `pl_bank_breaker_state == 2` for 1 min, not forced | Critical (page) |
| Format changed | Any `pl_bank_format_changed_total` increase | Critical (page) |
| Canary failing on all nodes | `pl_bank_canary_ok == 0` on every node for 10 min | Critical |
| Canary failing on one node | One node for 15 min | Warning |
| Node blocked | `bank.node_blocked` event | Warning |
| Canary not configured | A bank with no active canary, checked daily at 09:00 EAT | Info |

Logs include bank, node, class, latency and parser version. They never include payer or receiver names, phones or accounts.

## 14. Configuration

| Variable | Default |
| --- | --- |
| `EGRESS_NODES` | (required) JSON `[{"id":"et-egress-1","proxyUrl":"http://10.20.0.11:3128","isp":"ethio_telecom"},{"id":"et-egress-2","proxyUrl":"http://10.20.0.12:3128","isp":"safaricom"}]` |
| `BANK_CBE_BASE_URL` | `https://mb.cbe.com.et` |
| `BANK_CBE_LEGACY_ENABLED`, `BANK_CBE_LEGACY_BASE_URL` | `false`, `https://apps.cbe.com.et:100` |
| `BANK_TELEBIRR_BASE_URL` | `https://transactioninfo.ethiotelecom.et` |
| `BANK_BOA_BASE_URL` | `https://cs.bankofabyssinia.com` |
| `BANK_DASHEN_BASE_URL` | `https://receipt.dashensuperapp.com` |
| `BANK_AWASH_BASE_URL`, `BANK_AWASH_TLS_PIN_SHA256` | `https://awashpay.awashbank.com:8225`, (required when Awash is enabled) |
| `BANK_CBEBIRR_BASE_URL` | `https://cbepay1.cbe.com.et` |
| `BANK_MPESA_BASE_URL` | `https://m-pesabusiness.safaricom.et` |
| `BANK_USER_AGENT` | `Proofline-Verifier/1.0` |
| Plus every variable in 6.10 | |

There are no per-tenant settings. Feature flag: `bank.cbe_legacy` mirrors `BANK_CBE_LEGACY_ENABLED` so it can be switched without a redeploy.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 04-Q1 | Legal basis and agreements for each bank page (overview Q1) | Unofficial public pages, polite rate limits, transparent User-Agent |
| 04-Q2 | Test accounts and canary references per bank (overview Q2) | Banks without a canary show "not configured" |
| 04-Q3 | Exact input formats, not-found signals and receiver exposure per bank (6.1) | Patterns above, confirmed in Phase 0 |
| 04-Q4 | User-Agent policy: transparent with a contact URL, or a browser UA? | Transparent (`Proofline-Verifier/1.0`) |
| 04-Q5 | Is CBE's legacy FT lookup still alive? If yes, whose 8 digits does it take, the payer's or the receiver's? | Disabled |
| 04-Q6 | BoA's encrypted receipt QR: can BoA provide a decryption method or an official lookup? | Unsupported; the user types the FT reference |
| 04-Q7 | How long after payment does each bank publish the receipt (overview Q15)? | Final on the first answer |
