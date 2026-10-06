# 02 · Receiving accounts

| | |
| --- | --- |
| Package | `@proofline/receiving-accounts` |
| Postgres schema | `receiving_accounts` |
| Depends on | platform, 01 tenancy |
| Build step | 2 |
| Mockups | Portal sidebar "Receiving accounts"; the "Paid into" select on Portal · Verify a payment; receiver rows in the review workspace and the verification drawer; the API clients screen note "Live keys work only after your receiving accounts are verified". The Receiving accounts screen itself is not drawn. |

## 1. Purpose and scope

**Does**

- Lets a tenant register the bank accounts and wallets it collects money into, per bank and per environment.
- Has platform staff confirm that the tenant owns each account before it counts.
- Answers one question for the verification engine: *"Was this payment credited to one of this tenant's verified accounts?"* (`matchReceiver`).
- Provides `hasVerified()` for the live gate (R14) and the BoA account suffixes the BoA connector needs (R5).

**Out of scope**

- Moving money. We never send or receive funds for an ownership check (see open question 02-Q1).
- Reading receipts and bank pages; doc 04 turns a bank response into `ReceiverFacts`.
- Restricting an account to a branch (open question 02-Q3).

## 2. Screens and user stories

| Screen | Surface |
| --- | --- |
| Receiving accounts: list, add, upload proof, submit, remove | Portal (not drawn; follows the portal table and drawer patterns) |
| "Paid into" select | Portal · Verify a payment; cashier app (default account per bank) |
| Receiving accounts approval queue and detail | Admin, inside Tenants (not drawn) |

**US-02.1 Add an account (live)**

- An Owner or Admin picks a bank and enters the account number or wallet number, the holder name exactly as the bank shows it, an optional label (for example "Main telebirr till") and up to 5 name aliases.
- The identifier is validated against the bank's pattern (section 6.2). The account is created with `status = pending_proof`.
- A second account with the same bank and identifier in the same tenant and env returns `409 duplicate_account`.

**US-02.2 Add an account (test)**

- In test mode the same form creates the account directly as `verified` with `proof_method = sandbox`. No proof is needed.

**US-02.3 Upload proof and submit**

- The tenant uploads one or more documents: a bank letter, or a statement header showing the account number and holder name (PDF, JPG or PNG, at most 10 MB, at most 5 files).
- Submitting with no document returns `400 missing_field`. Status becomes `proof_submitted`, and the account appears in the admin queue.

**US-02.4 Approve or reject (admin)**

- A platform Admin sees the account details, the documents (signed URLs valid 60 s) and the tenant's legal name.
- Approving requires ticking both checks: "number matches the document" and "holder matches the document". Status becomes `verified`, and `receiving_account.verified` is emitted.
- If the same bank and identifier is already verified for another tenant in the same env, approval is refused with `409 account_registered_elsewhere`, naming that tenant to the admin.
- Rejecting requires a reason of 10–500 characters, which is emailed to Owners and Admins.

**US-02.5 Remove**

- An Owner or Admin who re-authenticated in the last 5 minutes removes an account. Status becomes `removed`, and it stops matching new bank answers at once.
- If it was the tenant's last verified account in live, the portal shows a confirmation saying live API calls will stop with `403 live_not_enabled`.

**US-02.6 List for integrators**

- `GET /v1/receiving-accounts` with any valid key returns the tenant's verified accounts in the key's env, so `expected.receiving_account_id` can be set.

**US-02.7 Match**

- Given the bank says money went to a receiver, `matchReceiver` returns `match` with the account ID and the basis, or `no_match`, or `unverifiable`, according to the rules in section 6.4. A unit test exists for every row of the decision table.

## 3. Module design

```text
libs/features/receiving-accounts/src/
├─ public/
│  ├─ index.ts                     ReceivingAccountsModule, ReceivingAccountsApi, ReceiverFacts, ReceiverMatch, events
│  ├─ receiving-accounts-api.ts
│  └─ events.ts
├─ domain/
│  ├─ identifier-patterns.ts       per-bank validation and normalisation (section 6.2)
│  ├─ name-normalizer.ts           normaliseName(), namesEquivalent()
│  ├─ receiver-matcher.ts          pure decision table (section 6.4)
│  └─ account-status.ts            transitions
├─ application/
│  ├─ accounts.service.ts          add, submit, remove, list
│  ├─ approval.service.ts          admin approve and reject
│  └─ matching.service.ts          loads candidates and calls receiver-matcher
├─ infrastructure/
│  ├─ accounts.repo.ts, proofs.repo.ts
│  └─ tenant-closed.handler.ts     @OnEvent('tenant.status_changed')
├─ http/
│  ├─ public-api/receiving-accounts.controller.ts
│  ├─ portal/receiving-accounts.controller.ts
│  └─ admin/receiving-accounts.controller.ts
└─ db/schema.ts, db/migrations/
```

```ts
export interface ReceivingAccountsApi {
  hasVerified(tenantId: string, env: EnvName): Promise<boolean>;                    // cached 60 s
  listVerified(tenantId: string, env: EnvName, bankCode?: BankCode): Promise<ReceivingAccountView[]>;
  get(tenantId: string, env: EnvName, id: string): Promise<ReceivingAccountView>;   // 404 if not visible
  matchReceiver(q: {
    tenantId: string; env: EnvName; bankCode: BankCode;
    restrictToId?: string;              // expected.receiving_account_id
    facts: ReceiverFacts;
  }): Promise<ReceiverMatch>;
  boaSuffixes(tenantId: string, env: EnvName): Promise<{ receivingAccountId: string; last5: string }[]>; // verified BOA only
  countAwaitingApproval(): Promise<number>;                                          // live accounts in proof_submitted, all tenants (admin badge, doc 11)
}

export interface ReceiverFacts {
  name: string | null;                                  // as the bank printed it
  digits: { value: string; position: 'last' | 'first' | 'full' } | null;  // visible digits only, e.g. "4401" last
}
export type ReceiverMatch =
  | { result: 'match'; receivingAccountId: string; basis: 'digits_and_name' | 'digits' | 'name' }
  | { result: 'no_match' }
  | { result: 'unverifiable' };                         // bank exposed neither name nor digits

export interface ReceivingAccountView {
  id: string;                         // racc_…
  bankCode: BankCode; kind: 'bank_account' | 'wallet' | 'merchant_code';
  label: string | null; holderName: string; nameAliases: string[];
  accountMasked: string;              // "****4401"
  status: 'pending_proof' | 'proof_submitted' | 'verified' | 'rejected' | 'removed';
  env: EnvName; verifiedAt: Date | null; createdAt: Date;
}
```

## 4. Data model

```sql
CREATE TABLE receiving_accounts.accounts (
  id                 uuid PRIMARY KEY,
  tenant_id          uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                text NOT NULL CHECK (env IN ('live','test')),
  bank_code          text NOT NULL CHECK (bank_code IN ('CBE','TELEBIRR','BOA','DASHEN','AWASH','CBEBIRR','MPESA')),
  kind               text NOT NULL CHECK (kind IN ('bank_account','wallet','merchant_code')),
  identifier_enc     text NOT NULL,          -- Crypto.encrypt(tenantId, normalised identifier)
  identifier_hmac    bytea NOT NULL,         -- Crypto.hmac('account', bank_code || ':' || identifier)
  last4              char(4) NOT NULL,       -- for display and matching
  label              text NULL CHECK (length(label) <= 60),
  holder_name        text NOT NULL CHECK (length(holder_name) BETWEEN 2 AND 120),
  holder_name_norm   text NOT NULL,
  name_aliases       text[] NOT NULL DEFAULT '{}' CHECK (cardinality(name_aliases) <= 5),
  name_aliases_norm  text[] NOT NULL DEFAULT '{}',
  status             text NOT NULL CHECK (status IN ('pending_proof','proof_submitted','verified','rejected','removed')),
  proof_method       text NULL CHECK (proof_method IN ('document_review','sandbox')),
  decided_by         uuid NULL,              -- tenancy.staff.id
  decided_at         timestamptz NULL,
  reject_reason      text NULL,
  created_by         uuid NOT NULL,          -- tenancy.users.id
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  verified_at        timestamptz NULL,
  removed_at         timestamptz NULL,
  removed_by         uuid NULL
);
-- One record per identifier per tenant and env (removed ones excluded)
CREATE UNIQUE INDEX accounts_tenant_identifier_uq
  ON receiving_accounts.accounts (tenant_id, env, bank_code, identifier_hmac) WHERE status <> 'removed';
-- Q9 default: a verified identifier belongs to one tenant per env
CREATE UNIQUE INDEX accounts_verified_global_uq
  ON receiving_accounts.accounts (env, bank_code, identifier_hmac) WHERE status = 'verified';
CREATE INDEX accounts_match_idx ON receiving_accounts.accounts (tenant_id, env, bank_code) WHERE status = 'verified';
CREATE INDEX accounts_review_queue ON receiving_accounts.accounts (updated_at) WHERE status = 'proof_submitted';

CREATE TABLE receiving_accounts.proofs (
  id            uuid PRIMARY KEY,
  account_id    uuid NOT NULL REFERENCES receiving_accounts.accounts(id),
  tenant_id     uuid NOT NULL,
  env           text NOT NULL,
  kind          text NOT NULL CHECK (kind IN ('bank_letter','statement','other')),
  object_key    text NOT NULL,                -- bucket "kyb", key t/{tenantId}/racc/{accountId}/{proofId}
  sha256        bytea NOT NULL,
  content_type  text NOT NULL CHECK (content_type IN ('application/pdf','image/jpeg','image/png')),
  size_bytes    integer NOT NULL CHECK (size_bytes BETWEEN 1 AND 10485760),
  uploaded_by   uuid NOT NULL,
  uploaded_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX proofs_account_idx ON receiving_accounts.proofs (account_id);

ALTER TABLE receiving_accounts.accounts ENABLE ROW LEVEL SECURITY; ALTER TABLE receiving_accounts.accounts FORCE ROW LEVEL SECURITY;
CREATE POLICY t_iso ON receiving_accounts.accounts
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid AND env = current_setting('app.env', true))
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid AND env = current_setting('app.env', true));
-- same policy on proofs.
```

The global uniqueness check on approval needs to see other tenants' rows. The function below runs `SECURITY DEFINER`, is owned by `pl_migrator`, and returns only the owning tenant ID:

```sql
CREATE FUNCTION receiving_accounts.verified_owner(p_env text, p_bank text, p_hmac bytea)
RETURNS uuid LANGUAGE sql STABLE SECURITY DEFINER SET search_path = receiving_accounts AS $$
  SELECT tenant_id FROM receiving_accounts.accounts
  WHERE env = p_env AND bank_code = p_bank AND identifier_hmac = p_hmac AND status = 'verified' LIMIT 1
$$;
REVOKE ALL ON FUNCTION receiving_accounts.verified_owner FROM PUBLIC;
GRANT EXECUTE ON FUNCTION receiving_accounts.verified_owner TO pl_admin;
```

**Migration:** `0001_receiving_accounts_init.sql` (both tables, indexes, RLS, the function).

## 5. API contracts

### Public API (`apps/api`)

`GET /v1/receiving-accounts` takes any valid key (R13) and returns the verified accounts in the key's env.

```json
{ "data": [ { "id": "racc_01JA2B3C4D5E6F7G8H9J0K1M2N", "livemode": true, "bank_code": "TELEBIRR",
             "label": "Main till", "holder_name": "HABESHA MART PLC", "account_masked": "****4401",
             "created_at": "2026-08-12T09:30:00+03:00" } ],
  "next_cursor": null, "has_more": false }
```

### Portal (`/portal/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /receiving-accounts` | `receiving_accounts.read` | `?status&bank_code` | `200 { data: ReceivingAccount[] }` | none |
| `POST /receiving-accounts` | `receiving_accounts.manage` | `{ bank_code, account_identifier, holder_name, label?, name_aliases? }` | `201 ReceivingAccount` | `400 invalid_field` (identifier pattern), `409 duplicate_account`, `422 account_limit_reached` |
| `PATCH /receiving-accounts/{id}` | `receiving_accounts.manage` | `{ label?, holder_name?, name_aliases? }` | `200` | `409 invalid_state`: name fields may change only in `pending_proof` or `rejected` |
| `POST /receiving-accounts/{id}/proofs` | `receiving_accounts.manage` | multipart `kind`, `file` | `201 Proof` | `413`, `415`, `422 proof_limit_reached` (5), `409 invalid_state` |
| `DELETE /receiving-accounts/{id}/proofs/{proof_id}` | `receiving_accounts.manage` | none | `204` | `409 invalid_state` (after submit) |
| `POST /receiving-accounts/{id}/submit` | `receiving_accounts.manage` | none | `200` | `400 missing_field` (`proofs`), `409 invalid_state` |
| `DELETE /receiving-accounts/{id}` | `receiving_accounts.manage`, plus re-auth | none | `200` (status `removed`) | `401 reauth_required` |

`ReceivingAccount` in portal responses adds `status`, `reject_reason`, `proofs[]` (id, kind, uploaded_at) and `verified_at` to the public fields. The identifier is never returned unmasked.

### Admin (`/admin/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /receiving-accounts` | `platform.tenants.read` | `?status=proof_submitted&cursor` | `200 Page<AdminReceivingAccount>` (tenant name, bank, holder, `account_masked`, submitted_at) | none |
| `GET /receiving-accounts/{id}` | `platform.tenants.read` | none | `200` detail plus proof list | `404` |
| `GET /receiving-accounts/{id}/identifier` | `platform.pii.reveal`, plus re-auth | none | `200 { "identifier": "1000123454401" }`; audited `pii.revealed` | `401 reauth_required` |
| `GET /receiving-accounts/{id}/proofs/{proof_id}/url` | `platform.receiving_accounts.approve` | none | `200 { url, expires_at }` (60 s); audited | `404` |
| `POST /receiving-accounts/{id}/approve` | `platform.receiving_accounts.approve` | `{ number_matches: true, holder_matches: true, note? }` | `200` | `400 invalid_field` (either check not `true`), `409 account_registered_elsewhere`, `409 invalid_state` |
| `POST /receiving-accounts/{id}/reject` | `platform.receiving_accounts.approve` | `{ reason }` | `200` | `409 invalid_state` |

Admin calls run in a transaction scoped to the account's tenant (overview 5.4).

### New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `duplicate_account` | 409 | This tenant already registered this identifier in this env |
| `account_registered_elsewhere` | 409 | Another tenant holds this identifier verified (admin only) |
| `account_limit_reached` | 422 | 20 non-removed accounts per tenant per env |
| `proof_limit_reached` | 422 | 5 proofs per account |

### Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `receiving_account.verified` | `{ account_id, bank_code }` | 03 (live-gate cache flush) |
| `receiving_account.rejected` | `{ account_id, reason }` | platform `Notify` (email Owners and Admins) |
| `receiving_account.removed` | `{ account_id, bank_code }` | 03 (live-gate cache flush) |

**Consumes:** `tenant.status_changed` with `to = closed`, which marks every account of that tenant `removed` (`removed_by` null).

## 6. Business rules and state machine

### 6.1 Status

| From | To | Trigger | Guard |
| --- | --- | --- | --- |
| (none) | `pending_proof` | Add in live | Pattern valid, under the 20-account limit |
| (none) | `verified` | Add in test | Same; `proof_method = sandbox` |
| `pending_proof` or `rejected` | `proof_submitted` | Tenant submits | At least 1 proof |
| `proof_submitted` | `verified` | Admin approves | Both checks true; `verified_owner()` returns null or this tenant |
| `proof_submitted` | `rejected` | Admin rejects | Reason 10–500 characters |
| any except `removed` | `removed` | Tenant removes (re-auth) or tenant closed | none |

On `verified`, the cache key `pl:racc:hasverified:{tenantId}:{env}` is deleted and `receiving_account.verified` is emitted. The same happens on `removed`.

### 6.2 Identifier patterns

The identifier is normalised by stripping spaces, dashes and dots, and converting a leading `+251` or `0` to `251` for phone-like wallets. It is then validated against these patterns. They are best-known formats as of October 2026; the Phase 0 test payments (overview Q2) confirm them before launch, and a correction is a code change with a fixture test.

| Bank | Kind | Pattern after normalisation | `last4` |
| --- | --- | --- | --- |
| `CBE` | bank_account | `^1000\d{9}$` (13 digits) | Last 4 |
| `BOA` | bank_account | `^\d{7,16}$` | Last 4 (the BoA lookup also needs the last 5, read from the encrypted identifier) |
| `DASHEN` | bank_account | `^\d{13}$` | Last 4 |
| `AWASH` | bank_account | `^\d{10,16}$` | Last 4 |
| `TELEBIRR` | wallet or merchant_code | `^251[79]\d{8}$` (wallet) or `^\d{5,7}$` (merchant short code) | Last 4 |
| `CBEBIRR` | wallet | `^251[79]\d{8}$` | Last 4 |
| `MPESA` | wallet or merchant_code | `^2517\d{8}$` (wallet) or `^\d{5,7}$` (till) | Last 4 |

### 6.3 Name normalisation

`normaliseName(s)`:

1. Unicode NFKC, then trim.
2. Uppercase Latin letters. Ethiopic characters are left as they are.
3. Replace each of `. , ' " - & / ( ) _` with a space, then collapse runs of whitespace to one space.
4. Drop legal-form tokens: `PLC`, `P L C`, `SC`, `S C`, `SHARE COMPANY`, `LTD`, `LIMITED`, `PVT`, `PRIVATE`, `CO`, `COMPANY`, and their Amharic forms `ኃ.የተ.የግ.ማ`, `አ.ማ`, `ኃላ/የተ/የግ/ማህበር`.

`namesEquivalent(a, b)` is true if either holds:

- the token multisets are equal, or
- the shorter token set is a subset of the longer one and has at least 2 tokens.

So `HABESHA MART PLC` matches `Habesha Mart P.L.C` and `HABESHA MART TRADING`, but `MART` alone never matches anything. A receipt name matches an account if it is equivalent to `holder_name_norm` or to any of `name_aliases_norm`.

### 6.4 Receiver decision table

Candidates are the tenant's `verified` accounts for `bankCode` in `env`; with `restrictToId`, only that one, which must be verified and at that bank or the result is `no_match`. For each candidate:

- `digitsOk`: `null` if `facts.digits` is null. Otherwise it is true when the candidate's identifier has those digits at that position: `last` compares the identifier's last *n* digits, `first` the first *n*, `full` the whole identifier.
- `nameOk`: `null` if `facts.name` is null, otherwise `namesEquivalent`.

| `digitsOk` | `nameOk` | Candidate result |
| --- | --- | --- |
| true | true | match, basis `digits_and_name` |
| true | null | match, basis `digits` |
| true | false | no match. The digits collide but the name differs, so it is someone else's account with the same visible digits. |
| false | any | no match |
| null | true | match, basis `name`, only if the normalised name has at least 2 tokens and at least 8 characters |
| null | false | no match |
| null | null | unverifiable |

Overall result:

- If any candidate matches, the result is `match`. When several match, the first by `verified_at` wins, and the event is logged.
- Otherwise, if every candidate is unverifiable (the bank exposed nothing), the result is `unverifiable`.
- Otherwise, `no_match`.
- With no candidates at all, the result is `no_match`. The engine has already refused this case with `422 no_receiving_account` (R14).

Doc 05 maps `unverifiable` to `in_review` with `review_reason = receiver_unverifiable`; it never auto-verifies.

### 6.5 Numbers

| Rule | Value | Where |
| --- | --- | --- |
| Accounts per tenant per env | 20 non-removed | `RACC_MAX_PER_TENANT` |
| Proofs per account | 5 | Constant |
| Proof file size | At most 10 MB; PDF, JPG or PNG by magic bytes | Constant |
| `hasVerified` cache | 60 s; deleted on verify and remove | Constant |
| Re-auth for removal | 5 min (doc 01 session `reauth_at`) | Doc 01 |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Crypto` (`encrypt`, `hmac('account')`), `ObjectStore` (`kyb` bucket), `AuditLog`, `Outbox`, `OnEvent`, `Notify` |
| 01 tenancy | `TenancyApi.getTenant()` (legal name on the admin screen), `TenancyApi.listMemberEmails()` (rejection email) |
| Contracts | `BankCode` enum from `libs/contracts` (no dependency on doc 04) |

## 8. Implementation plan

1. **T02.1** Migration, Drizzle schema, RLS test, and a test that `verified_owner()` is not callable by `pl_app`.
2. **T02.2** `identifier-patterns.ts` with a valid and an invalid example per row of 6.2.
3. **T02.3** `name-normalizer.ts`, with table-driven tests including Amharic legal-form tokens.
4. **T02.4** `receiver-matcher.ts`, one test per row of 6.4 plus the overall-result rules.
5. **T02.5** Accounts service: add (live and test paths), proofs upload, submit, remove, limits.
6. **T02.6** Admin approval service, including the global-uniqueness refusal and the reveal endpoint with audit.
7. **T02.7** Public `GET /v1/receiving-accounts` (mounted after doc 03 provides the API-key guard; a contract test runs with a stub principal until then).
8. **T02.8** `ReceivingAccountsApi`: `hasVerified` with cache and invalidation, `listVerified`, `matchReceiver`, `boaSuffixes`.
9. **T02.9** Tenant-closed handler.
10. **T02.10** Portal screen (list, add drawer, proof upload, status chips, removal confirmation) and the admin queue and detail, with EN/AM strings in the portal.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Two tenants submit the same identifier | Both may be `proof_submitted`. The first approval wins; approving the second returns `409 account_registered_elsewhere`, and the admin rejects it with that reason. |
| Approval races removal | Both lock the row (`SELECT … FOR UPDATE`). Whichever commits second sees a status that no longer allows its transition and gets `409 invalid_state`. |
| Account removed while a verification is in flight | Matching happens when the bank answers. The removed account is no longer a candidate, so the check returns `receiver_mismatch`. The portal removal dialog warns about this. |
| Bank returns digits in a different position than expected | The parser in doc 04 always reports the position it saw. A mismatch yields `no_match`, never a false match. |
| Two verified accounts in one tenant share their last 4 digits | Both are candidates. Either match is correct because both belong to the tenant; the first by `verified_at` is reported. |
| Holder name on the bank receipt is truncated, e.g. `HABESHA MART TR` | The subset rule needs at least 2 shared full tokens. `HABESHA MART` is shared, so it matches; `TR` is ignored as an extra token in the longer side. |
| Ethiopic holder name on receipt, Latin on account | Not equivalent. The tenant adds the Ethiopic spelling as an alias before submitting (the form hint says so). |
| OpenBao unavailable when decrypting `boaSuffixes` | Throws `503 service_unavailable` inside the BoA fetch. Doc 04 classifies it as `transient`, and the engine retries per its schedule. |
| Proof upload fails half-way | The object is written before the DB row, in a `pending/` prefix. A row is inserted only after the upload completes; orphaned objects under `pending/` older than 24 h are deleted by the maintenance job. |

## 10. Security and privacy

| Endpoint | AuthN | AuthZ |
| --- | --- | --- |
| `GET /v1/receiving-accounts` | API key (doc 03) | Any valid key; tenant and env from the key |
| Portal routes | Portal session | Section 5 permissions; removal needs re-auth |
| Admin routes | Admin session | `platform.*` permissions; identifier reveal needs re-auth and is audited |

- Full identifiers are class P data: stored encrypted, and returned only through the audited admin reveal. Lookups use the HMAC column.
- Proof documents are class P, in the `kyb` bucket with server-side encryption. Signed URLs last 60 s and every fetch is audited.
- The cross-tenant uniqueness function is `SECURITY DEFINER`, executable only by `pl_admin`, and returns only a tenant ID.
- **Audited actions:** add, submit, approve, reject, remove, identifier reveal, proof view.

## 11. Performance

- `matchReceiver` runs once per bank answer, roughly 15 per second at peak. The query uses `accounts_match_idx`, returns at most 20 rows, and decrypts nothing: `last4` and the stored digits suffice for `last` matches of up to 4 digits. For longer digit strings (`first`, `full`, or `last` with more than 4 digits) it decrypts the candidate identifiers using the tenant data key cached for 5 min. Target p95 under 10 ms.
- `hasVerified` is a Redis hit in the steady state (60 s TTL).

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Patterns (6.2), normaliser (6.3, including Amharic), decision table (6.4), status transitions |
| Integration | Add, submit, approve, then `hasVerified` flips to true within one call (cache invalidated); global uniqueness refusal; RLS isolation; tenant-closed handler |
| End to end | Owner adds a telebirr account, uploads a statement, submits; Admin approves in the admin panel; Developer's live key now succeeds (with doc 03) |
| Amharic | Receiving accounts screen in `am`: status chips, form errors, removal dialog |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_racc_pending_review` | gauge | none |
| `pl_racc_match_total` | counter | `bank`, `result` (match, no_match, unverifiable), `basis` |
| `pl_racc_approval_duration_seconds` | histogram | none (submit to decision) |

| Alert | Condition | Severity |
| --- | --- | --- |
| Approval backlog | `pl_racc_pending_review` above 10 for 24 h | Info |
| Unverifiable spike | `result = unverifiable` above 20% of matches for one bank over 1 h | Warning (that bank's page probably stopped showing receiver details) |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `RACC_MAX_PER_TENANT` | 20 | Accounts per tenant per env |

There are no per-tenant settings and no feature flags.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 02-Q1 | Which ownership proof do we accept (overview Q8)? Options: a bank letter or statement reviewed by an Admin; a random small transfer from us (needs a Proofline bank account and transfers); the tenant paying a random amount from the account itself is **not** proof | Document review only |
| 02-Q2 | Can two tenants share a verified account (overview Q9)? | No: `accounts_verified_global_uq` |
| 02-Q3 | Should an account be restricted to specific branches, so a cashier at branch A can't verify payments into branch B's till? | No restriction |
| 02-Q4 | Exact account-number formats per bank (section 6.2) | Patterns above, confirmed in Phase 0 |
