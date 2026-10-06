---
paths:
  - "libs/contracts/**"
---

# Contracts rules (zod schemas, errors, i18n, OpenAPI)

**zod schemas**

- `libs/contracts` is the single source of request and response schemas for `/v1`, `/hosted/v1`, `/portal/v1` and `/admin/v1`.
- Field names, formats, limits and enum values are copied from the feature doc's API section. Money is `^(0|[1-9]\d{0,11})(\.\d{1,2})?$` on input and always 2 decimals on output. Timestamps are RFC 3339 with `+03:00` on output.

**Public API changes**

- `/v1` is additive-only: new optional fields, new endpoints, new enum values and new event types. Removing or renaming anything needs `/v2` (overview 6.2).
- OpenAPI 3.1 is generated from the schemas, including the top-level `webhooks` section (doc 08). The CI diff fails on breaking changes.

**Error catalogue**

- `src/errors.ts` holds every code: the overview 6.4 table plus each doc's "New error codes". Each entry has the HTTP status, `retryable`, and the doc that defines it.
- Never reuse a code for a different meaning.

**i18n catalogues**

- `i18n/{en,am}/{common,status,reason,errors,email,portal,cashier,upload,admin}.json`.
- Every key exists in both locales, and the CI parity check enforces it.
- Amharic strings marked † in the docs are drafts. Keep them in `i18n/am/_review.json` (a list of keys awaiting translator sign-off) until reviewed (overview Q10).

**Shared constants**

- `SANDBOX_REFERENCES` (doc 04, 6.8), bank codes, statuses and reasons are defined once here and imported by both backend and frontend.
