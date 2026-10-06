# 06 · Receipt intake

| | |
| --- | --- |
| Package | `@proofline/receipts` |
| Postgres schema | `receipts` |
| Depends on | platform, 01 tenancy, 04 bank connectors (registry only) |
| Build step | 6 (in parallel with 08) |
| Mockups | Upload · Choose (EN), Upload · Choose (አማ), Upload · In review (EN); Portal · Verify a payment (receipt drop zone, Use camera); review workspace receipt viewer (QR read, image facts). Upload preview, checking, verified, rejected and expired states are defined here. |

## 1. Purpose and scope

**Does**

- **Upload links:** creates, reissues and expires them (30 min, 3 uploads).
- **The hosted upload page** (`apps/web-upload`) and its API (`/hosted/v1`).
- **Receipt upload** through the public API, portal and cashier app.
- **Intake safety:** size and type checks by magic bytes, malware scan, EXIF stripping, safe PDF handling, sha256.
- **Reading the receipt:** share link, then QR code, then PDF text layer, then vision model, with a confidence score per field and OCR misread variants.
- **Image fraud signals:** exact and similar images seen before, edited-file traces, QR-versus-text mismatch.
- **Evidence for reviewers** (doc 07): images, extracted fields, signals.
- **Retention:** images deleted after the tenant's retention period (default 90 days).

**Out of scope**

- Deciding the payment (doc 05).
- The cross-merchant **reference** signal ("reference already used at another merchant"). It comes from doc 05's claim registry, because doc 06 may not read doc 05's data. Doc 07 shows both kinds of signal side by side.
- The review queue (doc 07).

## 2. Screens and user stories

| Screen | Surface | Mockup |
| --- | --- | --- |
| Choose: context, Take a photo, Choose file, paste link, QR tip, expiry, privacy, language switch | Upload page | Upload · Choose (EN) and (አማ) |
| Preview (Use this or Retake), Checking ("Reading receipt", then "Contacting bank") | Upload page | Not drawn; same card layout |
| In review | Upload page | Upload · In review (EN) |
| Verified, Rejected, Couldn't read (retry), Link expired, Closed | Upload page | Not drawn; same card layout with the state colours from the UI spec |
| Receipt drop zone and Use camera | Portal Verify | Portal · Verify a payment |
| Receipt viewer and image facts | Admin and portal review | Admin · Review workspace (doc 07 renders it) |

**US-06.1 Link issued**

- When doc 05 moves a verification to `receipt_required`, an upload link is created with a 30-minute TTL, 3 uploads, and the tenant's default locale.
- `next_action.upload_url` is `https://pay.<domain>/r/{token}?lang=en|am`.

**US-06.2 Open the link**

- The page shows the merchant's upload-page display name and logo, the expected amount (in the locale's format), the order number (`external_id`), the bank, and "This link expires in N min".
- The reason line varies: "Your bank isn't responding right now…" for `bank_unavailable`, or "Upload your receipt or paste the share link from your SMS" for `reference_insufficient`.

**US-06.3 Upload a photo**

- On a 360 px Android phone, Take a photo opens the camera. The image is resized on the device to at most 1,600 px on its longest side before upload, then previewed with Use this and Retake.
- Use this uploads it. The page shows the "Reading receipt" step, then "Contacting bank".

**US-06.4 Read by QR**

- For a telebirr or CBE receipt screenshot whose QR code is visible, extraction yields the bank and reference with `method: qr` and confidence 1.0 within 2 s (p95), and emits `receipt.extracted`.

**US-06.5 Read by PDF text**

- A Dashen PDF receipt yields the reference from the PDF text layer with `method: pdf_text`, plus the claimed amount, payer and receiver.

**US-06.6 Read by vision**

- A phone photo with no readable QR goes to the vision provider when one is configured. Each field carries a confidence.
- A reference with confidence below 0.98 also yields up to 3 misread variants (doc 04, section 6.9).
- With `VISION_PROVIDER = none`, the step is skipped and the receipt counts as unreadable.

**US-06.7 Couldn't read: retry**

- When no reference can be read from a hosted upload, the page shows "We couldn't read this receipt. Try a clearer photo with the QR code visible." with the number of tries left, and the verification stays `receipt_required`.
- After the 3rd failed read, or when the customer taps "Send for review anyway", `receipt.extraction_failed` with `final: true` sends the verification to review.

**US-06.8 Paste a share link**

- Pasting `https://transactioninfo.ethiotelecom.et/receipt/CHQ0FJ403O` (or any allowed link) yields a candidate with `method: share_link` and confidence 1.0, with no image stored.

**US-06.9 Result states**

- The page polls the status and shows Verified, In review (mockup), Rejected (reason label, no retry, "Contact {merchant}"), Link expired (ask the merchant for a new link), or Closed (the verification was cancelled or finished before the upload).

**US-06.10 Signals**

- Uploading the exact same file twice to the same tenant flags `exact_duplicate` with scope `same_tenant` on the second receipt, linking the first receipt's verification.
- A visually identical re-encode (Hamming distance at most 6) flags `similar_image`.
- A JPEG whose EXIF `Software` is an image editor flags `edited_with_software`.
- A QR reference that differs from the vision-read reference flags `qr_text_mismatch`.

**US-06.11 EXIF stripped**

- No stored image derivative contains EXIF, XMP or GPS data. The test reads every stored object and asserts no metadata segments.

**US-06.12 Expired link**

- Opening a link after 30 minutes shows Link expired.
- `POST` to it returns `410 upload_link_expired`. After 3 accepted uploads, it returns `410 upload_link_used`.

**US-06.13 Reissue**

- `POST /v1/verifications/{id}/upload-link`, or "New upload link" in the portal drawer, invalidates the previous active link and returns a new one. Allowed only in `receipt_required`.

**US-06.14 Consent**

- When `VISION_PROVIDER_CROSS_BORDER = true`, the page shows a consent checkbox above the upload buttons. Uploading without it returns `400 consent_required`.
- Consent is stored on the receipt with the text version.

**US-06.15 Amharic**

- Every page state renders in `am` with Noto Sans Ethiopic, without overflow at 320 px wide, matching Upload · Choose (አማ).

## 3. Module design

```text
libs/features/receipts/src/
├─ public/
│  ├─ index.ts              ReceiptsModule, ReceiptsApi, ReceiptEvidence, events
│  ├─ receipts-api.ts
│  └─ events.ts
├─ domain/
│  ├─ magic-bytes.ts        sniff(buffer16) → 'image/jpeg' | 'image/png' | 'image/webp' | 'application/pdf' | null
│  ├─ candidates.ts         assemble, dedupe, rank, add variants
│  ├─ signals.ts            rules (section 6.6) as pure functions
│  ├─ dhash.ts              64-bit difference hash, Hamming distance
│  └─ link-state.ts         upload-link state rules
├─ application/
│  ├─ staging.service.ts    stage(): stream → object store, sha256, AV scan
│  ├─ attach.service.ts     attachStaged(tx), attachShareLink(tx)
│  ├─ verification-state.handler.ts  @OnEvent verification.status_changed, review.task_created, review.escalated_to_platform → verification_states mirror
│  ├─ links.service.ts      createUploadLink(tx), invalidate(tx), resolve(token)
│  ├─ hosted.service.ts     hosted context, upload, status
│  ├─ extract.processor.ts  queue "receipt-extract" (kinds: reference, claimed_fields)
│  ├─ sanitize.ts           sharp pipeline; pdfjs worker
│  ├─ qr.ts                 zxing-wasm decode with preprocessing variants
│  ├─ pdf-text.ts           per-bank label regexes → candidates plus claimed fields
│  ├─ vision/
│  │  ├─ provider.ts        VisionProvider interface plus schema
│  │  ├─ none.ts  anthropic.ts  openai.ts  google.ts  self-hosted.ts
│  └─ retention.processor.ts  daily purge
├─ infrastructure/ repos, clamd client, fingerprint queries
├─ http/
│  ├─ hosted/hosted.controller.ts           mounted by apps/api under /hosted/v1
│  ├─ public-api/receipts.controller.ts     /v1/verifications/{id}/receipt, /upload-link
│  └─ portal/receipts.controller.ts
└─ db/schema.ts, db/migrations/

apps/web-upload/                            Next.js, static export plus client fetches to /hosted/v1
├─ app/r/[token]/page.tsx                   state machine UI (section 6.8)
├─ lib/resize.ts                            canvas downscale to 1,600 px, JPEG q0.85
└─ messages/ → libs/contracts/i18n/{en,am}/upload.json
```

### Public interface

```ts
export interface ReceiptsApi {
  stage(file: IncomingFile, ctx: { tenantId: string; env: EnvName }): Promise<StagedFile>;     // validates, stores, scans; throws 413/415/422
  attachStaged(tx: Tx, a: { stagingId: string; verificationId: string; verificationCreatedAt: Date;
                            tenantId: string; env: EnvName; source: ReceiptSource }): Promise<{ receiptId: string }>;
  createUploadLink(tx: Tx, a: { verificationId: string; verificationCreatedAt: Date; tenantId: string; env: EnvName;
                                locale: Locale; reason: ReceiptRequiredReason;            // enum from libs/contracts
                                display: { amountMinor: bigint; externalId: string | null; bankCode: BankCode | null } }): Promise<UploadLinkView>;
  invalidateUploadLinks(tx: Tx, verificationId: string): Promise<void>;
  activeUploadLink(verificationId: string): Promise<UploadLinkView | null>;  // decrypts token_enc; null unless status = 'active' and not expired
  attachShareLink(tx: Tx, a: { link: string; verificationId: string; verificationCreatedAt: Date;   // called by doc 05's addReceipt
                               tenantId: string; env: EnvName; source: ReceiptSource }): Promise<{ receiptId: string }>;
  latestForVerification(verificationId: string): Promise<ReceiptSummary | null>;
  getEvidence(receiptId: string, viewer: Actor): Promise<ReceiptEvidence>;     // signed display URLs (60 s), fields, signals; audited
  ensureClaimedFields(receiptId: string): Promise<void>;                     // enqueues vision for claimed fields if missing (doc 07)
}
export type ReceiptSource = 'hosted' | 'api' | 'portal' | 'cashier';
export interface StagedFile { stagingId: string; sha256: string; contentType: AllowedMime; sizeBytes: number }
export interface UploadLinkView { uploadLinkId: string; url: string; expiresAt: Date; uploadsLeft: number }
export interface ReceiptEvidence {
  receiptId: string; verificationId: string; source: ReceiptSource; contentType: AllowedMime; pageCount: number;
  displayUrls: string[];                                  // one per page, 60 s
  candidates: Candidate[]; claimed: ClaimedFields | null;
  signals: Signal[]; meta: ForensicMeta; sha256Short: string; uploadedAt: Date;
}
export interface Candidate { bankCode: BankCode; reference: string; method: 'share_link' | 'qr' | 'pdf_text' | 'vision' | 'variant'; confidence: number }
export interface ClaimedFields {
  amount: { value: string; confidence: number } | null; payerName: { value: string; confidence: number } | null;
  receiverName: { value: string; confidence: number } | null; receiverAccountMasked: { value: string; confidence: number } | null;
  paidAt: { value: string; confidence: number } | null; status: { value: string; confidence: number } | null;
  source: 'pdf_text' | 'vision';
}
export type Signal =
  | { kind: 'exact_duplicate'; scope: 'same_tenant'; firstReceiptId: string; firstVerificationId: string; firstSeenAt: string }
  | { kind: 'exact_duplicate'; scope: 'other_tenant'; count: number }
  | { kind: 'similar_image'; scope: 'same_tenant'; receiptIds: string[]; minDistance: number }
  | { kind: 'similar_image'; scope: 'other_tenant'; count: number }
  | { kind: 'edited_with_software'; software: string }
  | { kind: 'pdf_modified'; producer: string | null; createdAt: string | null; modifiedAt: string | null }
  | { kind: 'qr_text_mismatch'; qrReference: string; textReference: string }
  | { kind: 'no_exif_on_photo' };                         // informational only
```

## 4. Data model

```sql
CREATE TABLE receipts.upload_links (
  id                      uuid PRIMARY KEY,
  tenant_id               uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                     text NOT NULL,
  verification_id         uuid NOT NULL,
  verification_created_at timestamptz NOT NULL,
  token_hash              bytea NOT NULL UNIQUE,              -- sha256(token), used for lookup
  token_enc               text NOT NULL,                      -- Crypto.encrypt(tenantId, token): lets doc 05 repeat next_action.upload_url (GET, webhooks) while active
  display                 jsonb NOT NULL,                     -- {amount_minor, external_id, bank_code} passed by doc 05 for the hosted page
  reason                  text NOT NULL,                      -- receipt_required reason at issue time
  locale                  text NOT NULL CHECK (locale IN ('en','am')),
  max_uploads             smallint NOT NULL DEFAULT 3,
  uploads_used            smallint NOT NULL DEFAULT 0,
  failed_reads            smallint NOT NULL DEFAULT 0,
  status                  text NOT NULL CHECK (status IN ('active','exhausted','expired','invalidated')),
  expires_at              timestamptz NOT NULL,
  created_at              timestamptz NOT NULL DEFAULT now(),
  invalidated_at          timestamptz NULL
);
CREATE UNIQUE INDEX upload_links_one_active ON receipts.upload_links (verification_id) WHERE status = 'active';

CREATE TABLE receipts.verification_states (   -- read model fed by events (section 5, Consumes)
  verification_id         uuid PRIMARY KEY,
  tenant_id               uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                     text NOT NULL,
  status                  text NOT NULL,                      -- doc 05 status
  reason                  text NULL,                          -- reject or receipt reason code
  review_due_at           timestamptz NULL,                   -- from review.task_created, updated by review.escalated_to_platform
  updated_at              timestamptz NOT NULL                -- the event's occurredAt; older events are ignored
);

CREATE TABLE receipts.staging (
  id            uuid PRIMARY KEY,
  tenant_id     uuid NOT NULL, env text NOT NULL,
  object_key    text NOT NULL,
  sha256        bytea NOT NULL,
  content_type  text NOT NULL,
  size_bytes    integer NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  attached_at   timestamptz NULL
);
CREATE INDEX staging_orphans ON receipts.staging (created_at) WHERE attached_at IS NULL;

CREATE TABLE receipts.receipts (
  id                      uuid PRIMARY KEY,
  tenant_id               uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                     text NOT NULL,
  verification_id         uuid NOT NULL,
  verification_created_at timestamptz NOT NULL,
  upload_link_id          uuid NULL REFERENCES receipts.upload_links(id),
  source                  text NOT NULL CHECK (source IN ('hosted','api','portal','cashier')),
  input_kind              text NOT NULL CHECK (input_kind IN ('file','share_link')),
  share_link_bank         text NULL,
  content_type            text NULL,
  size_bytes              integer NULL,
  sha256                  bytea NULL,
  page_count              smallint NULL,
  original_object_key     text NULL,          -- PDFs only; images keep derivatives only
  display_object_keys     text[] NOT NULL DEFAULT '{}',
  meta                    jsonb NOT NULL DEFAULT '{}'::jsonb,   -- ForensicMeta (no GPS)
  status                  text NOT NULL CHECK (status IN ('stored','processing','extracted','unreadable','failed')),
  candidates              jsonb NOT NULL DEFAULT '[]'::jsonb,
  claimed                 jsonb NULL,
  signals                 jsonb NOT NULL DEFAULT '[]'::jsonb,
  consent_at              timestamptz NULL,
  consent_version         text NULL,
  created_at              timestamptz NOT NULL DEFAULT now(),
  processed_at            timestamptz NULL,
  purge_at                timestamptz NOT NULL,                -- created_at + tenant retention.receipts_days
  purged_at               timestamptz NULL
);
CREATE INDEX receipts_verification ON receipts.receipts (verification_id, created_at DESC);
CREATE INDEX receipts_purge ON receipts.receipts (purge_at) WHERE purged_at IS NULL;

CREATE TABLE receipts.extractions (
  id            uuid PRIMARY KEY,
  receipt_id    uuid NOT NULL REFERENCES receipts.receipts(id),
  tenant_id     uuid NOT NULL, env text NOT NULL,
  kind          text NOT NULL CHECK (kind IN ('reference','claimed_fields')),
  method        text NOT NULL CHECK (method IN ('share_link','qr','pdf_text','vision')),
  provider      text NULL, model text NULL,
  outcome       text NOT NULL CHECK (outcome IN ('ok','empty','error','timeout')),
  output        jsonb NULL,                -- schema-validated; no raw prompt
  duration_ms   integer NOT NULL,
  cost_micros   bigint NULL,               -- provider billing currency micro-units
  cost_currency char(3) NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX extractions_receipt ON receipts.extractions (receipt_id);

CREATE TABLE receipts.image_fingerprints (
  receipt_id   uuid PRIMARY KEY REFERENCES receipts.receipts(id),
  tenant_id    uuid NOT NULL, env text NOT NULL,
  sha256       bytea NOT NULL,
  dhash        bigint NOT NULL,
  band1 smallint NOT NULL, band2 smallint NOT NULL, band3 smallint NOT NULL, band4 smallint NOT NULL,
  verification_id uuid NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  expires_at   timestamptz NOT NULL           -- created_at + 365 days
);
CREATE INDEX fp_sha ON receipts.image_fingerprints (env, sha256);
CREATE INDEX fp_b1 ON receipts.image_fingerprints (env, band1);
CREATE INDEX fp_b2 ON receipts.image_fingerprints (env, band2);
CREATE INDEX fp_b3 ON receipts.image_fingerprints (env, band3);
CREATE INDEX fp_b4 ON receipts.image_fingerprints (env, band4);
```

- **RLS:** the standard tenant and env policy on every table. Cross-tenant fingerprint matching goes through a `SECURITY DEFINER` function that returns other-tenant **counts** only:

  ```sql
  CREATE FUNCTION receipts.match_fingerprint(p_env text, p_tenant uuid, p_sha bytea, p_dhash bigint, p_b1 smallint,
                                             p_b2 smallint, p_b3 smallint, p_b4 smallint, p_max_distance int)
  RETURNS TABLE (same_exact_receipt uuid, same_exact_verification uuid, same_exact_at timestamptz,
                 other_exact_count int, same_similar uuid[], same_min_distance int, other_similar_count int)
  -- body: exact = sha match; similar = any band equal AND bit_count(dhash # p_dhash) <= p_max_distance (excluding exact)
  ```

- **Object keys:** bucket `receipts`. Staging uses `t/{tenantId}/{env}/staging/{stagingId}`; stored receipts use `t/{tenantId}/{env}/{yyyy}/{mm}/{receiptId}/{original.pdf | page-1.jpg …}`.
- **Migrations:** `0001_receipts_init.sql` and `0002_fingerprint_function.sql`.

## 5. API contracts

### Hosted (`/hosted/v1`, on `apps/api`, CORS limited to `UPLOAD_BASE_URL`, no cookies)

| Method and path | Auth | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /links/{token}` | Token | none | `200 HostedContext` | `404 not_found` (unknown token), `410 upload_link_expired`, `410 upload_link_used` (body still includes `state` so the page renders the right screen) |
| `POST /links/{token}/receipts` | Token; 30 per minute per IP, 10 per minute per link | multipart `file` plus `consent` (`"true"` when required); or JSON `{ "share_link": "…", "consent": true }` | `202 HostedStatus` | `400 consent_required`, `413 file_too_large`, `415 unsupported_file_type`, `422 file_rejected`, `409 invalid_state` (verification no longer accepts receipts), `410` as above, `429` |
| `POST /links/{token}/send-for-review` | Token | none | `202 HostedStatus` | `409 invalid_state` (no unreadable receipt on this link) |
| `GET /links/{token}/status` | Token; 60 per minute per link | none | `200 HostedStatus` | `404`, `410` |

```json
// HostedContext
{ "state": "awaiting_upload", "locale": "en",
  "merchant": { "display_name": "Habesha Mart", "logo_url": "https://pay.<domain>/l/…" },
  "amount": "12500.00", "order": "88231", "bank": { "code": "TELEBIRR", "name": "telebirr" },
  "reason": "bank_unavailable", "expires_at": "2026-10-04T10:45:00+03:00", "uploads_left": 3,
  "consent": { "required": false, "version": null, "text": null },
  "support_code": "ver_01JB7Q8K2M" }

// HostedStatus
{ "state": "checking" | "retry" | "in_review" | "verified" | "rejected" | "expired" | "closed",
  "step": "reading" | "contacting_bank" | null,
  "reason": "duplicate" | "receiver_mismatch" | … | null,      // rejected only, label rendered client-side
  "uploads_left": 2, "review_eta_minutes": 30 | null }
```

`state` is derived from the link and the `verification_states` mirror (6.8). `review_eta_minutes` is `ceil((review_due_at − now) / 60 s)`, at least 1, or `null` when unknown; the page then leaves out the time. An invalidated link still answers `GET /links/{token}` and `/status` for 24 h with the current state, so the customer sees the outcome, but refuses uploads with `409 invalid_state`. `HostedContext` takes amount, order and bank from the link's `display`. The page never receives amounts from the bank or payer details.

### Public API

| Method and path | Scope | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `POST /v1/verifications/{id}/receipt` | `receipts:write` | Owned by doc 05 (`VerificationApi.addReceipt`), which checks the status and calls this module's `stage` and `attachStaged`/`attachShareLink`. File rules (size, type, ClamAV, `file_rejected`) are this module's | | |
| `POST /v1/verifications/{id}/upload-link` | `verifications:write` | none | `201 { "upload_url", "expires_at", "uploads_left" }` | `409 invalid_state` |

### Portal (`/portal/v1`)

| Method and path | Permission | Notes |
| --- | --- | --- |
| `POST /verifications/{id}/receipt` | `verifications.create` | Owned by doc 05 (`addReceipt`, `source: portal`): staff upload on the customer's behalf |
| `POST /verifications/{id}/upload-link` | `verifications.create` | "New upload link" in the drawer; also returns the link as a QR (SVG) for showing on screen |
| `GET /verifications/{id}/receipt` | `verifications.read` or `.read.own` | `ReceiptEvidence` without `signals` other-tenant counts; display URLs valid 60 s |

### New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `file_rejected` | 422 | Malware detected, or the file is unsafe to process (polyglot, decompression bomb) |
| `consent_required` | 400 | Cross-border processing consent not given |

### Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `receipt.uploaded` | `{ receipt_id, verification_id, verification_created_at, source }` | 05 |
| `receipt.extracted` | `{ receipt_id, verification_id, verification_created_at, bank_code, candidates: [{ value, method, confidence }], has_claimed_fields }` | 05, 12 (image credits) |
| `receipt.extraction_failed` | `{ receipt_id, verification_id, verification_created_at, cause: "no_reference" \| "unreadable" \| "unsupported_qr" \| "processing_error", final: boolean }` | 05 (acts only on `final: true`) |
| `receipt.claimed_fields_ready` | `{ receipt_id, verification_id }` | 07 (refreshes the workspace) |

**Consumes** (schemas from `libs/contracts`, so no package dependency on the emitters):

- `verification.status_changed` (doc 05): upserts `receipts.verification_states` with the new status and reason, for `HostedStatus`.
- `review.task_created` and `review.escalated_to_platform` (doc 07): store `review_due_at` there, for the "usually within N minutes" line on the In review page.
- `tenant.settings_changed` (doc 01): recomputes `purge_at` when retention is shortened (section 9).

## 6. Business rules and state machine

### 6.1 Upload link

| From | To | Trigger |
| --- | --- | --- |
| (none) | `active` | `createUploadLink`. Any previous active link for the verification becomes `invalidated` in the same transaction |
| `active` | `exhausted` | `uploads_used` reaches `max_uploads` (3) |
| `active` | `expired` | `expires_at` passes (checked when the link is read; the daily job also sweeps) |
| `active` | `invalidated` | Reissue, or the verification moves to `in_review`, `verified`, `rejected`, `expired` or `cancelled` (doc 05 calls `invalidateUploadLinks`) |

- **Token:** 32 random bytes encoded base64url (43 characters). `sha256(token)` is stored for lookup, and the token is also stored encrypted with the tenant's key (`token_enc`) so the same link can be repeated in `GET /v1/verifications/{id}` and webhook payloads while it is active. Possession of the link is the same capability the merchant already received.
- **TTL:** exactly 1,800 s (`UPLOAD_LINK_TTL_SECONDS`). **Max uploads:** 3 (`UPLOAD_LINK_MAX_UPLOADS`).
- An upload is accepted only if the link is `active` and the verification is `receipt_required`. The check runs under a lock on the link row; doc 05 re-checks the verification status under its own lock.

### 6.2 Intake (`stage`)

1. **Size and file count:** multipart is limited to 10,485,760 bytes and exactly one file part; over the limit, abort and return `413`.
2. **Type:** sniff the first 16 bytes:
   - JPEG `FF D8 FF`;
   - PNG `89 50 4E 47 0D 0A 1A 0A`;
   - WebP `52 49 46 46 ?? ?? ?? ?? 57 45 42 50`;
   - PDF `25 50 44 46 2D`.
   Anything else returns `415`. The declared content type is ignored.
3. **Store:** stream to the staging key with SSE while computing sha256.
4. **Scan:** ClamAV (`clamd INSTREAM`) on the staged object. Infected returns `422 file_rejected` and deletes the object. ClamAV unreachable returns `503 service_unavailable` (fail closed).
5. **Record:** insert the `staging` row and return `StagedFile`.

Staging rows not attached within 60 minutes are deleted with their objects by the maintenance job.

### 6.3 Sanitising (in `receipt-extract`)

**Images (sharp)**

1. Read with `limitInputPixels: 40_000_000` and `failOn: 'error'`. Any decoder error marks the receipt `unreadable`.
2. Read the forensic metadata first: EXIF `Software`, `DateTimeOriginal`, `ModifyDate`, `Make`, `Model`, and whether EXIF was present at all. **GPS and every other tag are discarded.**
3. Rotate by EXIF orientation, convert to sRGB, and resize so the longest side is at most 2,000 px.
4. Encode JPEG at quality 85 with `withMetadata(false)`, and store it as `page-1.jpg`.
5. The original image object is **deleted** once the derivative is stored. Only the derivative and `meta` remain (rule: strip EXIF).

**PDFs (pdfjs-dist in a worker thread)**

- Options: `isEvalSupported: false`, `disableFontFace: true`, no network, 30 s timeout, 512 MB worker memory cap.
- At most the first 5 pages are processed. Each page renders to JPEG at scale 2 (longest side at most 2,000 px).
- The text layer is extracted, along with metadata `Producer`, `Creator`, `CreationDate` and `ModDate`.
- The original PDF is kept encrypted until `purge_at`, because the text layer is evidence.
- Encrypted PDFs, or PDFs with JavaScript or embedded files, are processed for rendering only and get the signal `pdf_modified`; they are never executed.

### 6.4 Extraction order

| Step | Runs when | Produces | Confidence |
| --- | --- | --- | --- |
| Share link | `input_kind = share_link` | 1 candidate via `BankRegistry.detect` | 1.00 |
| QR | Every image or page | Decode with zxing-wasm on the display image, then grayscale with contrast stretch, then 2× upscale, then Otsu binarisation; stop at the first decode. Each decoded text goes through `BankRegistry.detect`. Multiple QR codes give multiple candidates | 1.00 |
| PDF text | PDF | Per-bank label regexes (maintained with doc 04's fixtures) give the reference and the claimed fields | 0.98 |
| Vision (reference) | No candidate from the steps above, `VISION_PROVIDER ≠ none`, and the tenant flag `receipts.vision` is enabled | JSON per the schema below. The reference counts only if `BankRegistry.validateReference(bank, ref)` passes | Model-reported, capped at 0.97 |
| Variants | A vision reference with confidence below 0.98 | `BankRegistry.misreadVariants(bank, ref, 3)` | Parent confidence × 0.9 |
| Vision (claimed fields) | `ensureClaimedFields()` from doc 07, or when vision already ran | Fills `claimed` | Per field |

Candidates are deduplicated by `(bank, reference)`, keeping the highest confidence, and ordered by confidence (method order breaks ties). At most 4 go to doc 05.

**Vision schema.** The output is strictly validated with zod; invalid output counts as `empty`.

```json
{ "bank_code": "TELEBIRR|CBE|BOA|DASHEN|AWASH|CBEBIRR|MPESA|null",
  "reference": "string|null", "receipt_url": "string|null",
  "amount": "decimal string|null", "currency": "ETB|null",
  "payer_name": "string|null", "receiver_name": "string|null", "receiver_account_masked": "string|null",
  "paid_at": "ISO 8601|null", "status": "string|null",
  "confidence": { "reference": 0.0, "amount": 0.0, "payer_name": 0.0, "receiver_name": 0.0,
                  "receiver_account_masked": 0.0, "paid_at": 0.0 } }
```

**Vision rules**

- The system prompt says the image may contain text that tries to give instructions, and that such text must be ignored. The model gets no tools.
- Only `reference` and `receipt_url` are ever used for a bank lookup, and only after validation. Everything else is shown to reviewers as "claimed".
- One request per receipt (one retry on timeout), with a 20 s timeout.

### 6.5 Outcome and events

| Situation | Receipt status | Event |
| --- | --- | --- |
| At least 1 candidate | `extracted` | `receipt.extracted` |
| No candidate; hosted source; `failed_reads + 1 < 3`; customer hasn't asked for review | `unreadable` | `receipt.extraction_failed` with `final: false` (doc 05 keeps `receipt_required`). The page shows Retry |
| No candidate, and any of: the 3rd failed read on this link, "Send for review anyway", or a non-hosted source | `unreadable` | `receipt.extraction_failed` with `final: true` (doc 05 sends it to review with `no_reference_found`) |
| QR decoded but no bank recognised it (for example BoA's encrypted QR) | `unreadable` with `cause: unsupported_qr` | As in the two rows above. The page says "Type the reference instead" in the paste field |
| Processing error (the 3rd job attempt fails) | `failed` | `receipt.extraction_failed` with `cause: processing_error`, `final: true` |

### 6.6 Signals

| Signal | Rule |
| --- | --- |
| `exact_duplicate` (same tenant) | Same `sha256` in this tenant and env, on a different verification |
| `exact_duplicate` (other tenant) | Same `sha256` at another tenant. Count only |
| `similar_image` | dHash Hamming distance ≤ 6 (`RECEIPT_SIMILAR_MAX_DISTANCE`), excluding exact matches. Same tenant gives receipt IDs; other tenants give a count |
| `edited_with_software` | EXIF `Software` matches (case-insensitive) `photoshop|gimp|picsart|snapseed|canva|pixlr|lightroom|facetune|remini|photoroom` |
| `pdf_modified` | `ModDate − CreationDate > 60 s`; or `Producer` not in the bank's known-producer list (`pdf-text.ts` per-bank constants from fixtures); or the PDF is encrypted, scripted or has attachments |
| `qr_text_mismatch` | A QR reference and a vision or PDF-text reference both exist and differ after normalisation |
| `no_exif_on_photo` | A JPEG photo (not a screenshot by its dimensions) with no EXIF. Informational only |

Signals never auto-reject. They feed doc 07's risk panel; doc 05 decides only on bank facts and claims.

### 6.7 Retention

- At `purge_at` (`created_at` + tenant `retention.receipts_days`, default 90), the daily job at 02:00 EAT deletes the display and original objects and sets `purged_at`. Rows keep `candidates`, `claimed`, `signals` and `meta`.
- Fingerprints are deleted at `expires_at` (365 days).
- Extractions rows are kept with their receipts.

### 6.8 Upload page state machine (client)

| Server `state` | Screen | User actions |
| --- | --- | --- |
| `awaiting_upload` | Choose (mockup) | Take photo, Choose file, Paste link, language switch |
| (client only) | Preview | Use this or Retake |
| `checking` with step `reading` or `contacting_bank` | Checking | none (polling) |
| `retry` | Couldn't read, with tries left | Take another photo; "Send for review anyway" |
| `in_review` | In review (mockup) | Back to merchant (link to `support_code` only, no URL back) |
| `verified` | Payment confirmed | Back to merchant |
| `rejected` | We couldn't confirm this payment, plus the reason label (doc 07 UI spec table), "Contact {merchant}" | none |
| `expired` | This link has expired | none |
| `closed` | This payment check is closed | none |

**Polling:** `GET /status` every 3 s for 60 s, then every 10 s. It stops on `in_review`, `verified`, `rejected`, `expired` or `closed`.

**Bundle budget:** at most 150 KB of JS and CSS (gzipped), enforced in CI with `size-limit`. Fonts: Noto Sans Ethiopic subset plus IBM Plex Sans 400 and 600, served from our origin.

**Customer copy:**

| Key | English | Amharic (draft, translator review, overview Q10) |
| --- | --- | --- |
| `upload.title` | Confirm your payment | ክፍያዎን ያረጋግጡ |
| `upload.reason.bank_unavailable` | Your bank isn't responding right now. Upload your receipt and we'll confirm your payment. | ባንክዎ አሁን ምላሽ እየሰጠ አይደለም። ደረሰኝዎን ይስቀሉ፤ ክፍያዎን እናረጋግጣለን። |
| `upload.reason.reference_insufficient` | Upload your receipt or paste the share link from your SMS. | ደረሰኝዎን ይስቀሉ ወይም ከSMS መልዕክትዎ ሊንኩን ይለጥፉ። |
| `upload.take_photo` | Take a photo | ፎቶ ያንሱ |
| `upload.choose_file` | Choose file | ፋይል ይምረጡ |
| `upload.qr_tip` | Keep the QR code in the picture. | QR ኮዱ በፎቶው ውስጥ በግልጽ እንዲታይ ያድርጉ። |
| `upload.retry` | We couldn't read this receipt. Try a clearer photo with the QR code visible. | ይህን ደረሰኝ ማንበብ አልቻልንም። QR ኮዱ የሚታይበት ግልጽ ፎቶ ይሞክሩ። |
| `upload.in_review` | We're checking your receipt. {merchant} will update you. | ደረሰኝዎን እየመረመርን ነው። {merchant} ያሳውቅዎታል። |
| `upload.verified` | Payment confirmed | ክፍያዎ ተረጋግጧል |
| `upload.rejected` | We couldn't confirm this payment | ይህን ክፍያ ማረጋገጥ አልቻልንም |
| `upload.expired` | This link has expired. Ask {merchant} for a new one. | ይህ ሊንክ ጊዜው አልፏል። ከ{merchant} አዲስ ይጠይቁ። |
| `upload.privacy` | We use your receipt only to confirm this payment. | ደረሰኝዎን የምንጠቀመው ይህን ክፍያ ለማረጋገጥ ብቻ ነው። |

### 6.9 Numbers

| Rule | Value | Where |
| --- | --- | --- |
| Link TTL | 1,800 s | `UPLOAD_LINK_TTL_SECONDS` (brief: about 30 min) |
| Uploads per link | 3 | `UPLOAD_LINK_MAX_UPLOADS` |
| Max file | 10,485,760 bytes | `RECEIPT_MAX_BYTES` (brief) |
| Accepted types | JPG, PNG, WebP, PDF | Constant (brief) |
| PDF pages processed | 5 | `RECEIPT_MAX_PDF_PAGES` |
| Pixel limit | 40,000,000 | Constant |
| Display size | Longest side at most 2,000 px, JPEG q85 | Constant |
| Similar-image distance | ≤ 6 of 64 bits | `RECEIPT_SIMILAR_MAX_DISTANCE` |
| Vision timeout | 20 s, 1 retry | `VISION_TIMEOUT_MS` |
| Variant count | 3 (plus 1 primary = 4 candidates) | Constant |
| Retention | Tenant `retention.receipts_days` (default 90); fingerprints 365 days | Doc 01 settings, `RECEIPT_FINGERPRINT_DAYS` |
| Staging orphan TTL | 60 min | Constant |
| Client-side resize | Longest side 1,600 px, JPEG 0.85 | `apps/web-upload/lib/resize.ts` |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `ObjectStore` (`receipts`), `Crypto` (SSE keys through OpenBao), `Db`, `Queues` (`receipt-extract`), `Outbox`, `FeatureFlags` (`receipts.vision`), `RateLimit`, `AuditLog` (evidence views), `Clock` |
| 01 tenancy | `TenancyApi.getSettings()` (upload-page branding, retention, default locale), `getTenant()` (display name) |
| 04 bank connectors | `BankRegistry.detect`, `validateReference`, `misreadVariants`, `get` (bank names) |
| Infra | ClamAV (`clamd`, 2 replicas), the vision provider endpoint (Q4) |

Doc 05 calls into this module; this module never calls doc 05. It learns the verification's state from events (the mirror below) and from the upload link, which doc 05 creates and invalidates inside its own transactions (doc 05, section 6.2), so an active link is the authoritative gate for the customer's hosted uploads. Merchant-side receipts (API, portal, cashier) go through doc 05's `addReceipt`, which checks the verification status itself.

## 8. Implementation plan

1. **T06.1** Migrations (including `verification_states`) and the fingerprint function; RLS tests; function visibility test; the mirror handler.
2. **T06.2** `magic-bytes.ts` and the `stage()` pipeline with ClamAV (EICAR test file → `422`), size abort, and the staging orphan sweep.
3. **T06.3** Upload links: create, invalidate, resolve, the state rules, and the one-active-link index under concurrency.
4. **T06.4** Sanitising: the sharp pipeline (EXIF-stripped assertion), the pdfjs worker (a scripted PDF isn't executed; page cap; timeout).
5. **T06.5** QR decode with preprocessing (corpus tests), PDF-text per-bank regexes (fixtures shared with doc 04).
6. **T06.6** `attachStaged`, `attachShareLink`, the `verification_states` mirror, the extraction processor, candidate assembly and events (6.5).
7. **T06.7** dHash and signals (6.6), including the cross-tenant counts.
8. **T06.8** Vision provider interface, the `none` provider, and the self-hosted (OpenAI-compatible) provider; contract tests with recorded responses; schema validation; prompt-injection test (an image containing "ignore previous instructions…").
9. **T06.9** Hosted API: context, upload, send-for-review, status, rate limits, CORS.
10. **T06.10** Public and portal endpoints; `getEvidence` with audit; `ensureClaimedFields`.
11. **T06.11** `apps/web-upload`: every state in 6.8, EN/AM, client resize, the consent box, the bundle budget.
12. **T06.12** Retention job.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Customer double-taps Use this | The client disables the button. If two requests still arrive, both lock the link row; each counts as an upload; both receipts are processed; doc 05 handles the first extraction and records the second |
| Upload arrives as the link expires | Checked under the link lock. A request that started before expiry but locks after it gets `410` |
| Upload after the verification left `receipt_required` (bank answered meanwhile) | `409 invalid_state`. The page re-fetches status and shows the result |
| Decompression bomb (huge PNG dimensions) | Refused by `limitInputPixels`; status `unreadable`; with `final` per 6.5 |
| Polyglot file (PDF header with ZIP inside) | Treated by magic bytes as PDF and rendered by pdfjs without execution; ClamAV flags known malicious polyglots (`422`) |
| Vision provider down or slow | `timeout` or `error` is recorded. For the reference: no candidate, so retry or review per 6.5. For claimed fields: the workspace shows "Not available" |
| Vision returns a valid-looking reference for another bank than the QR | Both candidates are kept with `qr_text_mismatch`. QR outranks vision |
| Very large PDF page (A0 at 600 dpi) | Rendering is capped at 2,000 px longest side; the worker memory cap triggers `processing_error` if exceeded |
| ClamAV down | Uploads fail with `503`, and the page shows "Try again in a minute". It is alerted |
| Object store down | `stage()` fails with `503`. Extraction jobs retry twice (10 s, then 60 s), then `processing_error` (`final: true`) |
| A tenant shortens retention | Applies to new receipts. Existing `purge_at` values are recomputed by a one-off job triggered on `tenant.settings_changed` (only when shortened) |
| Same share link pasted twice | Allowed (consumes an upload). Doc 05's claim registry stops a double claim |

## 10. Security and privacy

| Endpoint | AuthN | AuthZ |
| --- | --- | --- |
| `/hosted/v1/links/{token}*` | Possession of the token (256-bit) | Scoped to one verification; no tenant data beyond the context shown |
| `/v1/verifications/{id}/receipt`, `/upload-link` | API key | Scopes in doc 03; tenant and env |
| Portal receipt routes | Session | `verifications.create` or `.read`; own scope for Cashiers |

- **Untrusted files:** magic-byte allowlist, ClamAV, sharp with pixel limits, pdfjs in a worker with no eval and no network, and a separate worker pool (`WORKER_GROUPS=receipt`) running as a non-root user with a read-only filesystem except `/tmp`.
- **PII:** receipt images and PDFs are class P, encrypted at rest and purged on schedule. Display URLs last 60 s, and every evidence view is audited (`receipt.viewed`).
- **No EXIF or GPS** survives in stored derivatives. Original images are deleted after derivation.
- **Upload origin isolation:** `pay.<domain>` has no cookies. Its CSP: `default-src 'self'; img-src 'self' blob: data:; connect-src https://api.<domain>; frame-ancestors 'none'`.
- **Cross-border:** when the vision provider is outside Ethiopia (`VISION_PROVIDER_CROSS_BORDER = true`), consent is required and recorded, and the tenant flag `receipts.vision` lets a tenant opt out entirely.
- **Rate limits:** as in section 5. Token guessing is infeasible.

## 11. Performance

| Item | Target |
| --- | --- |
| Hosted upload request (10 MB over 3G excluded) | Server time p95 under 800 ms (stream, sha, scan) |
| Extraction, QR path | p95 under 2 s |
| Extraction, vision path | p95 under 15 s |
| Peak throughput | 5 receipts per second; 4 jobs per pod, CPU-bound; scale pods on `receipt-extract` queue depth above 20 |
| Fingerprint lookup | p95 under 20 ms (band indexes) |
| Upload page | Under 150 KB JS and CSS; LCP under 2.5 s on 3G |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Magic bytes; link state; candidate ranking; dHash; every signal rule; vision schema validation |
| Corpus | An anonymised corpus of at least 30 receipts per bank: screenshots, phone photos and PDFs, with expected candidates. CI reports QR and PDF-text success rates and fails on regression |
| Integration | Stage → attach → extract → events; EICAR rejection; EXIF strip assertion; scripted PDF; pixel bomb; retry flow (3 unreadable → `final: true`); concurrency on one link |
| Vision | Recorded-response contract tests per provider; timeout handling; prompt-injection image |
| End to end | Playwright on a 360 × 800 viewport: open link (EN and AM), upload a photo, see Checking then In review; the Retry flow; the Expired screen; the consent box when enabled |
| Amharic | All states in `am` at 320 px wide: no overflow, Noto Sans Ethiopic loaded |
| Budget | `size-limit` for `apps/web-upload` |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_receipt_uploads_total` | counter | `source`, `content_type`, `outcome` (accepted, rejected_type, too_large, malware) |
| `pl_receipt_extraction_total` | counter | `method`, `outcome`, `bank` |
| `pl_receipt_extraction_duration_seconds` | histogram | `method` |
| `pl_receipt_vision_cost_micros_total` | counter | `provider`, `currency` |
| `pl_receipt_signals_total` | counter | `kind`, `scope` |
| `pl_receipt_clamav_errors_total` | counter | none |

| Alert | Condition | Severity |
| --- | --- | --- |
| ClamAV unavailable | Any `clamav_errors` for 2 min | Critical |
| QR success drop | QR success below 60% of images for a bank over 1 h (baseline from the corpus) | Warning (receipt layout may have changed) |
| Vision error rate | Above 10% over 15 min | Warning |
| Extraction backlog | `receipt-extract` waiting above 100 for 5 min | Warning |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `UPLOAD_LINK_TTL_SECONDS` | 1800 | Link lifetime |
| `UPLOAD_LINK_MAX_UPLOADS` | 3 | Uploads per link |
| `RECEIPT_MAX_BYTES` | 10485760 | Max file size |
| `RECEIPT_MAX_PDF_PAGES` | 5 | Pages processed |
| `RECEIPT_SIMILAR_MAX_DISTANCE` | 6 | dHash threshold |
| `RECEIPT_FINGERPRINT_DAYS` | 365 | Fingerprint retention |
| `CLAMD_HOST`, `CLAMD_PORT` | (required), 3310 | ClamAV |
| `VISION_PROVIDER` | `none` | `none`, `anthropic`, `openai`, `google` or `self_hosted` |
| `VISION_MODEL` | (provider-specific) | Model name |
| `VISION_ENDPOINT` | (required for `self_hosted`) | OpenAI-compatible base URL |
| `VISION_API_KEY` | from OpenBao | Provider key |
| `VISION_TIMEOUT_MS` | 20000 | Per request |
| `VISION_PROVIDER_CROSS_BORDER` | `false` | Shows the consent box when `true` |
| `VISION_CONSENT_VERSION` | `2026-10-01` | Version stored with each consent |

- **Per-tenant settings** (doc 01): `upload_page.display_name`, `upload_page.logo_object_key`, `locale.default`, `retention.receipts_days`.
- **Feature flag:** `receipts.vision` (default on), per tenant.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 06-Q1 | Vision provider: hosted abroad or self-hosted in-country (overview Q4)? | `none` until decided; unreadable receipts go to review |
| 06-Q2 | Is 90-day receipt retention right (overview Q13)? | 90 days, tenant-configurable 30–365 |
| 06-Q3 | Should the upload page offer "Try another receipt" after a **bank** rejection, as the UI spec draft said? A rejection is final (doc 05), so this doc only allows retries for **unreadable** receipts | Retries for unreadable only; the UI spec row is corrected |
| 06-Q4 | Should the merchant's logo be allowed as SVG on the upload page, given the XSS risk? | PNG only for logos on the upload page (SVG is rasterised at upload) |
