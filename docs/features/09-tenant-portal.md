# 09 · Tenant portal

| | |
| --- | --- |
| Package | `@proofline/portal` (BFF) and `apps/web-portal` (Next.js) |
| Postgres schema | `portal` |
| Depends on | platform, 01 tenancy, 02 receiving accounts, 03 API clients, 04 bank connectors (`BankRegistry.list/detect`, `BankHealth.snapshot`; adds the edge `portal → bank-connectors`), 05 verification, 06 receipts, 07 review, 08 webhooks, 12 billing |
| Build step | 8 (screens can start on contract stubs after step 5) |
| Mockups | Portal · Overview; Portal · Verify a payment (cashier view); Portal · Verifications with the detail drawer; Portal · API clients (shell only; the page is doc 03's) |

## 1. Purpose and scope

**Does**

- **The portal shell:** sidebar navigation filtered by permission, top bar (Live/Test, EN/አማ, search, docs, user menu), test-mode banner, usage meter, quota and session-timeout warnings, and error, empty and loading states.
- **`GET /portal/v1/bootstrap`:** everything the shell needs in one call, including `live_enabled` (doc 01 can't compute it).
- **Overview:** today's KPIs, outage banner, recent checks, rejections by reason, bank status, and a setup checklist until the tenant is live.
- **Verify a payment:** the no-integration check for shops and cashiers, with live bank detection, the result card in every state, receipt upload, the customer link, and a print slip.
- **Verifications log:** filters, built-in and saved views, search, totals, the detail drawer that combines five modules, and async CSV and Excel exports.
- **Sandbox page:** magic references and one-click test checks.
- **The status and reason vocabulary** (EN and AM) shared by every surface.

**Out of scope.** These pages live in the portal but are owned by other docs, which define their endpoints and rules:

| Page | Owner |
| --- | --- |
| Sign-up, sign-in, MFA, Team, Settings, KYB | 01 |
| Receiving accounts | 02 |
| API clients, Request logs | 03 |
| Review queue (Hybrid) | 07 |
| Webhooks | 08 |
| Billing & usage | 12 |
| The cashier PWA under `/cashier` | 10 |

This doc only places those pages in the shell, which gives every page the same frame, navigation and states.

## 2. Screens and user stories

| Screen | Mockup |
| --- | --- |
| Shell: sidebar, top bar, banners | Every portal mockup |
| Overview | Portal · Overview |
| Overview with setup checklist (not live) | Not drawn; the KPI cards are replaced by the checklist card |
| Verify a payment, verified result card | Portal · Verify a payment |
| Result card: rejected, in review, receipt needed | Not drawn. The mockup's note says "same card: red for rejected with the reason, amber for in review with the expected decision time, blue for receipt needed with an upload button" |
| Verifications log and detail drawer | Portal · Verifications |
| Export dialog and Exports list | Not drawn; a dialog from "Export CSV / Excel" and a list in the user menu |
| Sandbox | Not drawn; UI spec row "Sandbox" |
| Forbidden, Not found, Session expiring | Not drawn; the card layout from the upload page |

**US-09.1 Shell and navigation**

- After sign-in, `GET /bootstrap` returns the user, tenant, env, permissions, `live_enabled`, plan, usage and navigation.
- The sidebar shows only the items allowed by section 6.1. An Owner sees the full menu of the Overview mockup. A Cashier sees only Verify and "My checks today" with a count, as in the Verify mockup.
- The header shows the tenant's display name and "Growth plan · Live". The usage meter reads "Checks this month 6,412 / 10,000 · Resets 1 November". It reads "API checks this month" only for a plan that meters just the API channel (doc 12; none at launch).
- Opening a route the user may not use shows the Forbidden page, never a broken screen.

**US-09.2 Live and test**

- The Live/Test switch (needs `env.switch`) calls `POST /session/env` (doc 01), then reloads the bootstrap. It clears filters and closes any drawer, because IDs from the other env don't exist there.
- In test, an orange bar under the top bar reads "Test mode · No bank is called. Test data is kept separate from live."

**US-09.3 Language and calendar**

- EN / አማ calls `PATCH /me { locale }` (doc 01) and re-renders without a reload, keeping unsaved form input.
- In Amharic, dates show the Ethiopian calendar (`መስከረም 24, 2019 · 10:12`) with the Gregorian date on hover, unless the user's `calendar_pref` says otherwise. Money shows as `1,500.00 ብር`.

**US-09.4 Setup checklist**

- Until `live_enabled` is true, the Overview shows "Get ready to go live" in place of the KPI cards, with these steps and their states:
  1. Business details submitted and approved (KYB);
  2. A receiving account added and verified;
  3. Try a test check (any test-env verification exists);
  4. Create an API key (optional; "Skip if you only use the portal");
  5. Add a webhook (optional).
- Each step links to its page. When steps 1 and 2 are done, the card shows "You're live" once and then disappears.

**US-09.5 Overview KPIs**

- For the current EAT day (00:00 EAT until now), the four cards match `VerificationApi.stats`:
  - Verified: count, ETB total credited, and "98% confirmed by the bank";
  - Rejected: count and "Top reason: Receipt already used (11)";
  - In review: count and "Oldest waiting 24 min · 30 min SLA";
  - Receipt needed: count and "Waiting for customers to upload".
- Each card links to the log with the matching filter.
- The greeting follows the EAT hour: morning 05:00–11:59, afternoon 12:00–17:59, evening otherwise. In Amharic: እንደምን አደሩ, እንደምን ዋሉ, እንደምን አመሹ.

**US-09.6 Outage banner**

- When a bank where the tenant has a verified live receiving account has public status `down`, an amber banner says, for example: "M-Pesa isn't responding since 10:30. New M-Pesa checks ask your customer for a receipt, then go to review."
- It continues "Other banks are working" or, if any relevant bank is `degraded`, "Other banks are working; telebirr is slow".
- Several down banks are joined with ICU list formatting. "Bank status" scrolls to the bank list.
- The banner disappears within 60 s of recovery.

**US-09.7 Reversals banner**

- When any verification was `reversed` in the last 72 hours, a red banner reads "1 payment was reversed by the bank after a manual approval. Check before you ship." It links to the log filtered to `reversed`.

**US-09.8 Recent checks, rejections and bank status**

- "Recent checks" shows the newest 6 verifications (columns as in the log). "All verifications" opens the log.
- "Rejections this week" shows the total rejected over the last 7 EAT days and the top 5 reasons with bars scaled to the largest.
- "Bank status" lists all 7 banks with Working, Slow, or "Down · receipts go to review".

**US-09.9 Verify: input**

- The bank row offers "Auto-detect" (default) plus the 7 banks. A bank whose status is `down` shows "· down" and stays selectable; picking it shows "M-Pesa isn't responding. You'll be asked for the customer's receipt."
- Typing or pasting into "Reference, share link or SMS text" calls detection after 300 ms of no typing. The hint under the field reads "Looks like a telebirr receipt number", or "We can't tell the bank yet. Pick it above."
- Choosing CBE Birr, or detecting it, adds a required "Payer phone" field.
- Instead of a reference, a screenshot or PDF can be dropped, chosen or taken with the camera. JPG, PNG, WebP or PDF up to 10 MB are accepted. Images larger than 1,600 px are reduced in the browser before upload.
- "Expected amount (ETB)" is required (at least 0.01; at most 2 decimals).
- "Paid into" lists the tenant's verified accounts, narrowed to the selected or detected bank, with "Any of my {bank} accounts" first. A single match is preselected.
- "Order or note" is optional (at most 128 characters) and is stored as `external_id`.

**US-09.10 Verify: submit and wait**

- "Verify payment" sends `POST /portal/v1/verifications` (doc 05) with a new `X-Idempotency-Key` (UUIDv4) per form submission, so a double click can't create two checks. The server waits up to 15 s.
- The button shows "Checking with the bank…" and the form locks. The result card appears beside the form at 1,024 px and wider, and below it on narrower screens.

**US-09.11 Verify: result card**

| Status | Card | Content | Actions |
| --- | --- | --- | --- |
| `verified` | Green, "Verified" and the assurance chip | Amount large; payer, payer phone (masked), bank, reference (copy), paid at, paid into, note; 5 ticked checks with the plain labels in 6.4; verification ID | Print slip, Verify another |
| `rejected` | Red, "Rejected" and the reason label | The failed checks with details (6.4). For `duplicate` at the same tenant: "First used for order_88102 on 3 Oct, 14:20" with a link. No payer fields on `receiver_mismatch` | Print slip, Verify another |
| `in_review` | Amber, "In review" | "Usually decided within 30 min" (the plan SLA, or the Hybrid window when the tenant reviews first). "We'll update this card. You can also find it in Verifications." | Verify another |
| `receipt_required` | Blue, "Receipt needed" and the reason label | "The bank isn't answering. Upload the customer's receipt, or let them upload it on their phone." | Upload receipt (drop zone and camera), Show link to customer (QR code and Copy), Verify another |
| `processing` (wait ran out) | Slate, "Still checking" | "The bank is slow. This card updates by itself." | Verify another |

- Non-final cards poll `GET /portal/v1/verifications/{id}` every 3 s for 2 minutes, then every 15 s for up to 30 minutes. They update in place, with a polite live-region announcement for screen readers.
- After "Upload receipt", the card shows "Reading receipt", then "Contacting bank", then the outcome.

**US-09.12 Verify: gates**

- In live with `live_enabled` false, the form is replaced by "Live checks start once your business and a receiving account are verified", with links to both steps and "Try it in Test".
- When the plan meters the portal channel and usage is `blocked` (R12), the form is replaced by "You've reached this month's limit. Checks resume on 1 November." Users with `billing.manage` also get "Upgrade plan"; others get "Ask your account owner to upgrade".
- When `blocked_reason` is `unpaid` (doc 12 dunning), the form is replaced by "Live checks are paused until invoice INV-… is paid", with "Pay now" for `billing.manage`.
- If the server still answers `402 quota_exceeded` or `403 live_not_enabled`, the same panels appear.

**US-09.13 Print slip**

- "Print slip" opens the browser's print dialog with a slip laid out for 80 mm thermal paper (A4 when the printer reports it). It shows:
  - the tenant display name and status headline ("PAYMENT VERIFIED" or "DO NOT RELEASE");
  - the amount, bank, reference, payer, paid at in both calendars when the locale is `am`, paid into (masked), order or note;
  - the verification ID with a QR code of it, "Checked at", and "Checked by" (user name).
- Nothing else on the page prints.

**US-09.14 Verifications log**

- Title "Verifications" with the subtitle "Every check from the API, the portal and the cashier app."
- **Built-in views:** All; Needs action (`receipt_required` and `in_review`, with a count); Rejected; Manual review (`assurance = manual_review`). Saved views follow them, then "+ Save view".
- **Filter chips:** Date (Today, Yesterday, Last 7 days, This month, Custom; default Today), Status, Bank, Assurance; "More" adds Channel, Branch, Review reason and Payer name.
- **Find** accepts a verification ID (opens its drawer), a reference, an order ID or an exact payer name.
- **Columns:** Time, Status (badge plus reason note, 6.3), Amount (ETB), Bank, Reference, Payer, Order ID. The column picker adds Assurance, Channel, Branch, Created by and Verification ID.
- **Paging:** 25 rows by default (10, 25 or 50), Previous and Next, and "1–25 of 712 today". Totals above 10,000 show as "10,000+".
- The page refreshes every 30 s while visible, without moving the scroll position or the open drawer.

**US-09.15 Saved views**

- "+ Save view" stores the current filters and columns under a name (1–40 characters). It is private by default; "Share with my team" makes it visible to everyone in the tenant who can read the log.
- A user has at most 20 views and a tenant at most 50 shared views. Views can be renamed, reordered, updated with the current filters, or deleted (shared views only by their owner, or by anyone with `settings.manage`).

**US-09.16 Detail drawer**

- Clicking a row opens the drawer from the right without losing the list's scroll position or filters. The URL gains `?v=ver_…`, so Back closes it and the link can be shared.
- The drawer shows, as in the mockup:
  - the verification ID (copy), status and reason, assurance chip;
  - the "First claimed by order_88102 on 3 Oct, 14:20" box for same-tenant duplicates;
  - the amount and fields (bank, reference, payer, payer phone masked, paid at, paid into, order ID, plus channel, branch and created by);
  - the checks with details (6.4);
  - the receipt thumbnail when a receipt exists (opens the viewer, 60 s signed URLs);
  - the timeline (6.5), including webhook lines;
  - the raw API response, collapsed, for `verifications.read.raw`, labelled "· Developer role".
- **Actions by state:**
  - Resend webhook (`webhooks.manage`; doc 08);
  - Open first claim;
  - Cancel (non-final, `verifications.cancel`, with confirmation);
  - New upload link (doc 06) and Upload receipt (doc 05's `addReceipt`), both for `receipt_required` with `verifications.create`;
  - Open in review queue (Hybrid tenants with an open tenant-stage task).
- A non-final verification's drawer polls every 5 s.

**US-09.17 Export**

- "Export CSV / Excel" opens a dialog: format (CSV or Excel), the current filters summarised, and the date range (required, at most 92 days).
- Submitting returns at once with "We're preparing your file". When the file is ready (usually within a minute for 10,000 rows), a toast offers Download, and an email is sent.
- The file appears in the user menu's "Exports" for 7 days. More than 200,000 matching rows returns `422 export_too_large`, asking for a narrower range.

**US-09.18 Cashier view**

- A Cashier sees Verify and "My checks today". The latter is the log limited by doc 05's `own` scope to today, with the day's count and ETB verified.
- They have no filters other than Status, no export, no raw response, and no Cancel or Resend.

**US-09.19 Sandbox**

- In test, the Sandbox page lists every magic reference from doc 04 with its outcome, a copy button and "Try it". "Try it" runs a test verification with TELEBIRR (or a chosen bank) and amount 100.00, then shows the result card.
- `TEST_DUPLICATE` explains that it collides with a claim seeded for this tenant (official reference `TEST_DUPLICATE-ten_…`, doc 05), so it always ends `rejected/duplicate`.
- "Send test webhook" picks an endpoint and calls doc 08's test route. A curl example uses the tenant's test key ID with a placeholder secret.
- In live, the page shows "The sandbox works in Test mode" and a "Switch to Test" button.

**US-09.20 Errors, empty states and sessions**

- Every list has a skeleton while loading, an empty state with one action (for example "No checks yet today. Verify a payment"), and an error banner with the request ID and "Try again".
- `401` sends the user to sign-in with a return URL.
- `401 reauth_required` opens the re-authentication dialog (password and TOTP; doc 01), then repeats the action once.
- Two minutes before the 2-hour idle timeout, a dialog offers "Stay signed in".

**US-09.21 Accessibility and phones**

- Screens meet WCAG 2.2 AA (UI spec): status always as icon plus word, visible focus, full keyboard use (Esc closes the drawer; Enter submits Verify), and 4.5:1 text contrast (the tokens in 6.2 are checked).
- The portal works at 360 px: the sidebar collapses into a menu button, tables turn into stacked cards, and the drawer is full-screen.

## 3. Module design

```text
libs/features/portal/src/
├─ public/index.ts                 PortalModule (BFF only; no interface for other features)
├─ domain/
│  ├─ nav.ts                       navigation rules (section 6.1, pure)
│  ├─ overview.ts                  greeting, banner and checklist rules (pure)
│  ├─ check-text.ts                check → label key and parameters (section 6.4, pure)
│  ├─ timeline-merge.ts            doc 05 timeline + webhook lines → one ordered list (pure)
│  ├─ row-note.ts                  status → note under the badge (section 6.3, pure)
│  └─ export-format.ts             column list, CSV escaping, XLSX cell types (pure)
├─ application/
│  ├─ bootstrap.service.ts
│  ├─ overview.service.ts
│  ├─ detail.service.ts            drawer composition (parallel fan-out)
│  ├─ search.service.ts
│  ├─ saved-views.service.ts
│  ├─ exports.service.ts           create, list, download
│  ├─ export.processor.ts          queue "exports", job exp:{exportId}
│  └─ export-notifier.handler.ts   @OnEvent export.ready / export.failed → Notify.email
├─ infrastructure/ saved-views.repo.ts, exports.repo.ts, csv-writer.ts, xlsx-writer.ts (exceljs streaming)
├─ http/portal/*.controller.ts     routes in section 5.1
└─ db/schema.ts, db/migrations/

apps/web-portal/
├─ app/
│  ├─ layout.tsx                   <html lang>, fonts (IBM Plex Sans, Noto Sans Ethiopic, IBM Plex Mono), providers
│  ├─ (auth)/…                     doc 01 screens
│  ├─ (portal)/layout.tsx          Shell: Sidebar, TopBar, TestModeBanner, QuotaBanner, SessionTimer; loads bootstrap
│  ├─ (portal)/overview/page.tsx
│  ├─ (portal)/verify/page.tsx
│  ├─ (portal)/verifications/page.tsx     drawer through ?v=
│  ├─ (portal)/developers/sandbox/page.tsx
│  ├─ (portal)/{review,receiving-accounts,developers/api-clients,developers/webhooks,developers/request-logs,team,billing,settings}/…  pages owned by other docs
│  ├─ (portal)/forbidden/page.tsx, not-found.tsx
│  └─ cashier/…                    doc 10
├─ src/features/{shell,overview,verify,verifications,sandbox}/   components and TanStack Query hooks per area
├─ src/lib/api.ts                  fetch wrapper (below)
└─ next.config.ts                  security headers (section 10)
```

**Frontend stack.** These follow the UI spec and overview section 2:

- Next.js 16 App Router, React 19, Tailwind 4 with the `libs/ui` tokens, and Radix-based components from `libs/ui`.
- TanStack Query 5 for server state; React Hook Form with the zod schemas from `libs/contracts`; next-intl without locale routing (the locale comes from the user, not the URL).
- Pages are client components inside a server-rendered shell. All data goes through the BFF on the same origin: the browser calls `/api/portal/v1/…`, and the ingress strips `/api` before portal-api.
- Filters live in the URL's search parameters, so views are linkable, except `payer_name` and the Find text, which stay in component state (section 10).

```ts
// apps/web-portal/src/lib/api.ts
export async function api<T>(path: string, init?: RequestInit & { idempotencyKey?: string }): Promise<T>;
// Adds credentials: 'same-origin', X-CSRF-Token (from bootstrap) on non-GET, X-Idempotency-Key when given.
// Non-2xx → throws ApiError { status, code, message, field, requestId, retryable, details } from the overview 5.11 body.
// 401 unauthenticated → redirect to /login?next=…; 401 reauth_required → opens the re-auth dialog and retries once.
```

## 4. Data model

```sql
CREATE TABLE portal.saved_views (
  id             uuid PRIMARY KEY,
  tenant_id      uuid NOT NULL REFERENCES tenancy.tenants(id),
  owner_user_id  uuid NOT NULL,
  name           text NOT NULL CHECK (length(name) BETWEEN 1 AND 40),
  visibility     text NOT NULL CHECK (visibility IN ('private','tenant')),
  filters        jsonb NOT NULL,            -- VerificationFilters (zod), relative dates kept relative ("last_7_days")
  columns        text[] NULL,               -- NULL = default columns
  position       smallint NOT NULL DEFAULT 0,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX saved_views_owner_name ON portal.saved_views (tenant_id, owner_user_id, lower(name));
CREATE INDEX saved_views_tenant ON portal.saved_views (tenant_id) WHERE visibility = 'tenant';

CREATE TABLE portal.exports (
  id                    uuid PRIMARY KEY,
  tenant_id             uuid NOT NULL REFERENCES tenancy.tenants(id),
  env                   text NOT NULL CHECK (env IN ('live','test')),
  requested_by_user_id  uuid NOT NULL,
  scope                 jsonb NOT NULL,     -- the requester's DataScope at request time
  format                text NOT NULL CHECK (format IN ('csv','xlsx')),
  locale                text NOT NULL CHECK (locale IN ('en','am')),
  filters               jsonb NOT NULL,     -- resolved to an absolute range [from, to)
  status                text NOT NULL CHECK (status IN ('queued','running','ready','failed','expired')),
  row_count             integer NULL,
  object_key            text NULL,          -- bucket "exports"
  size_bytes            bigint NULL,
  sha256                bytea NULL,
  error_code            text NULL,
  created_at            timestamptz NOT NULL DEFAULT now(),
  started_at            timestamptz NULL,
  finished_at           timestamptz NULL,
  expires_at            timestamptz NULL,   -- finished_at + 7 days
  download_count        integer NOT NULL DEFAULT 0
);
CREATE INDEX exports_requester ON portal.exports (tenant_id, env, requested_by_user_id, created_at DESC);
CREATE INDEX exports_expiry ON portal.exports (expires_at) WHERE status = 'ready';
```

- **Saved views are env-less** (like tenancy tables, doc 01), because a filter means the same in live and test. RLS checks `tenant_id` only, and the repository adds `visibility = 'tenant' OR owner_user_id = current user`.
- **Exports** carry the standard tenant and env RLS policy.
- **Retention:** the `maintenance` job deletes ready exports' objects at `expires_at` and sets `status = expired`. Rows are kept 90 days for the audit trail, then deleted.
- **Migration:** `0001_portal_init.sql`.

## 5. API contracts

### 5.1 Routes owned by this doc (`/portal/v1`)

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /bootstrap` | Session | none | `200 Bootstrap` | `401` |
| `GET /badges` | Session | none | `200 { review_queue: number \| null, needs_action: number \| null, my_checks_today: number \| null }` (polled every 60 s) | none |
| `GET /overview` | `verifications.read` | none | `200 Overview` | none |
| `POST /verify/detect` | `verifications.create` | `{ input }` (at most 2,048 characters) | `200 { bank_code: BankCode \| null, kind: "reference" \| "share_link" \| "sms_text" \| "qr_payload" \| null, display_reference: string \| null, needs_payer_phone: boolean }` | `400 invalid_field` |
| `GET /verify/form` | `verifications.create` | none | `200 { banks: [{ code, name, status, inputs }], accounts: ReceivingAccountView[] }` (verified accounts in the session env, masked) | none |
| `GET /verifications/{id}/detail` | `verifications.read` or `.read.own` | none | `200 Detail` | `404` |
| `GET /verification-counts` | `verifications.read` or `.read.own` | The log's filters (doc 05) | `200 { count: number, capped: boolean }` | `400 invalid_field` (range over 31 days) |
| `POST /search` | `verifications.read` or `.read.own` | `{ q }` (2–128 characters) | `200 { exact: { verification_id } \| null, rows: VerificationRow[] }` (at most 20) | `400 invalid_field` |
| `GET /saved-views` | `verifications.read` | none | `200 { data: SavedView[] }` | none |
| `POST /saved-views` | `verifications.read` | `{ name, filters, columns?, visibility }` | `201 SavedView` | `400 invalid_field`; `409 conflict_name`; `422 saved_view_limit_reached` |
| `PATCH /saved-views/{id}` | Owner of the view, or `settings.manage` for shared views | `{ name?, filters?, columns?, visibility?, position? }` | `200 SavedView` | Same; `403 forbidden` |
| `DELETE /saved-views/{id}` | Same | none | `204` | `403 forbidden` |
| `POST /exports` | `verifications.export` | `{ format, filters, from, to }` | `202 Export` | `400 invalid_field` (range over 92 days); `422 export_too_large` (`details.count`); `429 rate_limited` |
| `GET /exports` | `verifications.export` | `?cursor` | `200 Page<Export>` (own exports only) | none |
| `GET /exports/{id}` | Requester | none | `200 Export` | `404` |
| `GET /exports/{id}/download` | Requester | none | `302` to a 300 s signed URL with `Content-Disposition: attachment`; audited `verifications.exported` | `404`; `409 invalid_state` (`details.reason: "expired"`) |
| `GET /sandbox` | `sandbox.use` | none | `200 { env, references: [{ reference, outcome_key, banks: "all" }], test_key_ids: string[], endpoints: [{ id, url }] }` | none |
| `POST /rum` | Session | `{ metric: "LCP" \| "INP" \| "CLS" \| "TTFB", value, route }` (batched, at most 20) | `204` | none |

### 5.2 Route map: who owns what under `/portal/v1`

| Prefix | Owner | Prefix | Owner |
| --- | --- | --- | --- |
| `/auth/*`, `/me`, `/session/*`, `/tenant*`, `/team/*`, `/branches*`, `/settings`, `/invitations/*` | 01 | `/receiving-accounts*` | 02 |
| `/api-clients*`, `/request-logs*` | 03 | `/banks` | 04 (served through this BFF) |
| `/verifications`, `/verifications/{id}`, `/verifications/{id}/timeline`, `/cancel`, `/raw`, `/reveal`, `/receipt` | 05 | `/verifications/{id}/upload-link` | 06 |
| `/review/*` | 07 | `/webhooks/*` | 08 |
| `/billing/*`, `/usage` | 12 | `/cashier/*` | 10 |
| everything in 5.1 | 09 | | |

### 5.3 Shapes

```json
// Bootstrap
{
  "me": { "…": "doc 01 Me" },
  "tenant": { "id": "ten_01J…", "display_name": "Habesha Mart PLC", "status": "active", "kyb_status": "approved" },
  "env": "live",
  "live_enabled": true,
  "plan": { "code": "growth", "name": "Growth" },
  "usage": { "period_end": "2026-11-01T00:00:00+03:00", "used": 6412, "included": 10000, "hard_limit": 12000,
             "metered_channels": ["api", "portal", "cashier"], "state": "ok", "blocked_reason": null },
  "nav": [ { "id": "overview", "href": "/overview" }, { "id": "verify", "href": "/verify" },
           { "id": "verifications", "href": "/verifications" }, { "id": "review", "href": "/review", "badge": 2 }, "…" ],
  "review_mode": "hybrid",
  "session": { "idle_timeout_seconds": 7200, "absolute_expires_at": "2026-10-04T20:02:00+03:00" },
  "links": { "docs": "https://docs.<domain>", "api_reference": "https://docs.<domain>/api" },
  "flags": { "vision_consent_required": false },
  "server_time": "2026-10-04T10:42:00+03:00"
}
```

`usage` is doc 12's `UsageApi.currentPeriod`. `state` is `ok` (under 80%), `warning` (80–99%), `over` (100% up to the hard limit) or `blocked` (at the hard limit, or `blocked_reason: unpaid`). Test env always reports `ok`.

```json
// Overview
{
  "date": "2026-10-04",
  "greeting": "morning",
  "setup": null,
  "banner": { "down": [ { "code": "MPESA", "name": "M-Pesa", "since": "2026-10-04T10:30:00+03:00" } ],
              "degraded": [ { "code": "TELEBIRR", "name": "telebirr" } ] },
  "reversals_72h": 0,
  "kpis": {
    "verified": { "count": 642, "amount": "1284350.00", "bank_confirmed_ratio": 0.98 },
    "rejected": { "count": 23, "top_reason": { "reason": "duplicate", "count": 11 } },
    "in_review": { "count": 2, "oldest_waiting_seconds": 1440, "sla_minutes": 30 },
    "receipt_required": { "count": 3 }
  },
  "recent": [ { "…": "VerificationRow × 6" } ],
  "rejections_week": { "total": 89, "by_reason": [ { "reason": "duplicate", "count": 38 }, { "reason": "receiver_mismatch", "count": 21 } ] },
  "banks": [ { "code": "CBE", "name": "CBE", "status": "operational" }, { "code": "MPESA", "name": "M-Pesa", "status": "down" } ]
}
```

When `setup` is set, it holds `{ "steps": [{ "key": "kyb" | "receiving_account" | "test_check" | "api_key" | "webhook", "state": "done" | "in_progress" | "todo", "optional": boolean }] }`, and `kpis` is null.

```json
// VerificationRow (log, recent, search)
{ "id": "ver_01JB7R4N8C2E6G0J3L5Q7S9U1W", "created_at": "2026-10-04T10:39:02+03:00", "status": "rejected",
  "note": { "key": "reason.duplicate" }, "assurance": null, "amount": "1800.00", "amount_kind": "credited",
  "bank_code": "TELEBIRR", "reference": "DET8FJGUJ4", "payer_name": "BIRUK ASFAW", "external_id": "order_88238",
  "channel": "api", "branch": null, "created_by": { "kind": "api_client", "name": "Online shop" } }

// Detail (drawer)
{
  "verification": { "…": "doc 05 portal view (adds created_by and api_client)" },
  "first_claim": { "verification_id": "ver_01JB6…", "external_id": "order_88102", "claimed_at": "2026-10-03T14:20:00+03:00" },
  "checks": [
    { "key": "receiver", "result": "pass", "label_key": "check.receiver", "detail_key": "check.receiver.pass", "params": { "bank": "telebirr" } },
    { "key": "duplicate", "result": "fail", "label_key": "check.duplicate", "detail_key": "check.duplicate.fail_same_tenant",
      "params": { "order": "order_88102", "at": "2026-10-03T14:20:00+03:00" } }
  ],
  "receipt": { "receipt_id": "rcpt_01J…", "content_type": "image/jpeg", "uploaded_at": "…", "source": "hosted" },
  "timeline": [ { "at": "2026-10-04T10:39:04+03:00", "kind": "webhook", "tone": "brand",
                  "key": "timeline.webhook_delivered", "params": { "type": "verification.rejected", "code": 200, "ms": 140 } } ],
  "review_task": null,
  "actions": { "cancel": false, "resend_webhook": true, "new_upload_link": false, "upload_receipt": false,
               "view_raw": true, "open_review": false }
}
```

`first_claim` is set only when doc 05's `details.first_claim` is present, which means a same-tenant duplicate. For a sandbox seed claim, `verification_id` and `external_id` are null; the box then reads "First claimed by the sandbox seed" and "Open first claim" is hidden. `actions` is computed from the status and the user's permissions, so the client never decides authorisation.

### 5.4 New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `export_too_large` | 422 | More than 200,000 rows match; `details.count` |
| `saved_view_limit_reached` | 422 | 20 per user, or 50 shared per tenant |
| `conflict_name` | 409 | A view with that name already exists for this user |

### 5.5 Events

| Event | Payload | Consumers |
| --- | --- | --- |
| `export.ready` | `{ export_id, requested_by_user_id, row_count }` | This module's notifier (email "Your export is ready" with a link to the Exports list, never to the file) |
| `export.failed` | `{ export_id, requested_by_user_id, error_code }` | This module's notifier |

**Consumes:** none.

## 6. Business rules

### 6.1 Navigation

| Item | Shown when | Badge |
| --- | --- | --- |
| Overview | `verifications.read` | none |
| Verify | `verifications.create` | none |
| Verifications | `verifications.read` | none |
| My checks today | `verifications.read.own` and not `verifications.read` | Today's own count |
| Review queue | `review.read` and the tenant's `review.mode = hybrid` | Open tenant-stage tasks (`ReviewApi.counts`) |
| Receiving accounts | `receiving_accounts.read` | none |
| *Developers:* API clients, Webhooks, Request logs, Sandbox | `api_clients.read`, `webhooks.read`, `request_logs.read`, `sandbox.use` | Webhooks: failing endpoint count, if any |
| *Account:* Team, Billing & usage, Settings | `team.read`, `billing.read`, `settings.read` | none |

- A group heading is hidden when none of its items is shown.
- The landing route is `/overview`, or `/verify` when Overview is hidden (Cashier).
- The same rules decide route access in the web app's middleware, and the BFF's permission checks are the real gate.

### 6.2 Status tokens and vocabulary

Colours come from the UI spec's status table and the mockups. Text on its background is at least 4.5:1, checked by a unit test.

| Status | Dot | Text | Background | Icon (Lucide) | English | Amharic |
| --- | --- | --- | --- | --- | --- | --- |
| `processing` | `#5B6B7A` | `#3D4A56` | `#EEF1F4` | `loader` | Processing | በሂደት ላይ |
| `receipt_required` | `#2F6FDB` | `#2457B0` | `#EAF1FC` | `upload` | Receipt needed | ደረሰኝ ያስፈልጋል |
| `in_review` | `#B7791F` | `#7F5310` | `#FBF1DE` | `eye` | In review | በግምገማ ላይ |
| `verified` | `#1F8A4C` | `#17683A` | `#E7F4EC` | `check-circle` | Verified | ተረጋግጧል |
| `rejected` | `#C2362F` | `#A12A24` | `#FCEDEC` | `x-circle` | Rejected | ውድቅ ተደርጓል |
| `reversed` | `#8A3FB8` | `#6B2F91` | `#F3EAF9` | `rotate-ccw` | Reversed | ተቀልብሷል |
| `expired` | `#8A949E` | `#5F6870` | `#F0F1F2` | `clock` | Expired | ጊዜው አልፏል |
| `cancelled` | `#8A949E` (outline) | `#5F6870` | transparent, 1 px `#8A949E` border | `slash` | Cancelled | ተሰርዟል |

Assurance chips: `bank_confirmed` "Bank confirmed" (በባንክ የተረጋገጠ) and `manual_review` "Manual review" (በሰው የተገመገመ).

**Reason catalogue** (`libs/contracts/i18n/{en,am}/reason.json`). Strings marked † are new drafts that need the translator review in overview Q10; the others are from the UI spec.

| Key | English | Amharic |
| --- | --- | --- |
| `reason.not_found` | Bank has no such transaction | ባንኩ እንዲህ ያለ ግብይት የለውም |
| `reason.receiver_mismatch` | Paid to a different account | ለሌላ ሂሳብ ተከፍሏል |
| `reason.amount_mismatch` | Amount is less than expected | መጠኑ ከሚጠበቀው ያነሰ ነው |
| `reason.duplicate` | Receipt already used | ደረሰኙ ቀድሞ ጥቅም ላይ ውሏል |
| `reason.stale` | Payment is too old | ክፍያው በጣም የቆየ ነው |
| `reason.not_completed` | Transfer not completed | ዝውውሩ አልተጠናቀቀም |
| `reason.unreadable_receipt` | Receipt can't be read | ደረሰኙ አይነበብም |
| `reason.suspected_forgery` | Receipt looks altered | ደረሰኙ የተቀየረ ይመስላል |
| `reason.bank_contradicted` | Bank contradicted the approval | ባንኩ ማረጋገጫውን ተቃርኗል |
| `receipt_reason.bank_unavailable` | Bank not responding | ባንኩ ምላሽ እየሰጠ አይደለም |
| `receipt_reason.reference_insufficient` | Receipt or share link needed | ደረሰኝ ወይም የማጋሪያ ሊንክ ያስፈልጋል |
| `receipt_reason.bank_format_changed` | Bank page changed, checking by hand | የባንኩ ገጽ ተቀይሯል፤ በእጅ እየተመረመረ ነው |
| `receipt_reason.image_requested` † | Clearer receipt requested | ይበልጥ ግልጽ ደረሰኝ ተጠይቋል |
| `review_reason.bank_unavailable` | Bank not responding | ባንኩ ምላሽ እየሰጠ አይደለም |
| `review_reason.bank_format_changed` | Bank page changed, checking by hand | የባንኩ ገጽ ተቀይሯል፤ በእጅ እየተመረመረ ነው |
| `review_reason.no_reference_found` † | No reference found on the receipt | በደረሰኙ ላይ የማጣቀሻ ቁጥር አልተገኘም |
| `review_reason.reference_unconfirmed` † | Bank couldn't confirm the reference | ባንኩ የማጣቀሻ ቁጥሩን ማረጋገጥ አልቻለም |
| `review_reason.receiver_unverifiable` † | Receiving account couldn't be checked | ተቀባዩ ሂሳብ ሊረጋገጥ አልቻለም |
| `review_reason.tenant_policy_amount` † | Above your review amount | ከግምገማ ገደብዎ በላይ ነው |
| `expired_reason.receipt_not_uploaded` † | Receipt never uploaded | ደረሰኝ አልተላከም |
| `expired_reason.review_timed_out` † | Not reviewed in time | በጊዜው አልተገመገመም |

The UI spec's table also lists `invalid_reference` and `manual_reject`. Neither is a verification reason under R2: `invalid_reference` is a `422` error shown on the form ("Reference format is wrong", የማጣቀሻ ቁጥሩ ቅርጸት ስህተት ነው, kept in `errors.json`), and reviewers always pick a specific reason (doc 07), so there is no `manual_reject`.

**Bank status labels:** `operational` "Working" (እየሰራ ነው); `degraded` "Slow" (ቀርፋፋ ነው); `down` "Down · receipts go to review" (አይሰራም · ደረሰኞች ወደ ግምገማ ይሄዳሉ) †.

### 6.3 The note under a status badge

| Status | Note |
| --- | --- |
| `verified` with `manual_review` | "Manual review" |
| `verified` with `bank_confirmed` | none |
| `rejected`, `reversed` | `reason.*` |
| `receipt_required` | `receipt_reason.*` |
| `in_review` | `review_reason.*` |
| `expired` | `expired_reason.*` |
| `cancelled`, `processing` | none |

**Amount column:** the credited amount when the bank answered (`amount_kind: credited`), otherwise the expected amount, shown muted with the tooltip "Expected amount" (`amount_kind: expected`).

### 6.4 Check texts

Two label sets: plain labels on the Verify card, and analytical labels in the drawer. The details are the same in both.

| Check | Card label | Drawer label | Detail by result |
| --- | --- | --- | --- |
| `receiver` | Paid into your account (ወደ ሂሳብዎ ተከፍሏል †) | Receiver | pass: "your registered {bank} account"; fail: "not one of your registered accounts"; unverifiable: "the account holder couldn't be confirmed" |
| `amount` | Amount matches (መጠኑ ይዛመዳል †) | Amount | pass: "{amount} as expected"; over: "{credited}, more than the {expected} expected" (passes); fail: "{credited}, less than the {expected} expected" |
| `bank_status` | Transfer completed (ዝውውሩ ተጠናቅቋል †) | Transfer completed | pass: "bank status Completed"; fail: "bank status Failed"; not_available: "the bank doesn't report a status" |
| `duplicate` | Not used before (ከዚህ በፊት ጥቅም ላይ አልዋለም †) | Not used before | pass: "first use of this receipt"; fail at the same tenant: "claimed by {order or verification ID} on {date}"; fail elsewhere: "already used for another payment" |
| `date_window` | Paid in the last {hours} h, or Paid after {time} when `paid_after` was given | Date window | pass: "paid {duration} before this check"; fail: "paid {duration} before this check, outside your {hours} h window" or "paid after this check" |
| any | n/a | n/a | not_run: "not checked" (greyed) |

`check-text.ts` produces the keys and parameters. The client formats amounts, durations and dates in the user's locale.

### 6.5 Timeline

- Doc 05 `timeline()` entries map to `timeline.<kind>` keys: `created` ("Created by API · order_88238", "Created in the portal by Liya Kebede", "Created by cashier Liya Kebede · Bole"), `bank_answered` ("Bank answered in 0.9 s"), `bank_failed` ("Bank didn't answer · trying again at 10:45"), `status_changed` ("Rejected: Receipt already used"), `receipt_uploaded`, `receipt_extracted` ("Reference read from the QR code"), `upload_link_issued`, `review_required`, `review_decided`, `assurance_upgraded`, `cancelled`, `expired`, `reversed`.
- `attempt_started` and `claim_conflict` are hidden from tenants.
- **Review decisions:** "Approved by a Proofline reviewer" on the platform stage (staff names are never shown to tenants), or "Approved by Hanna B." for a tenant reviewer.
- **Webhook lines** come from `WebhooksApi.deliveriesForVerification`, one per delivery at its last attempt time: "Webhook verification.rejected delivered · 200 in 140 ms", "Webhook verification.rejected failed · 503 · next try 10:45", or "Webhook verification.rejected failed after 10 tries".
- Entries are ordered by time, then by a fixed kind order (status before webhook), so the same-second entries in the mockup read in order.

### 6.6 Search

`POST /search` runs these lookups in parallel inside the caller's data scope:

- `q` matching `^ver_[0-9A-HJKMNP-TV-Z]{26}$`: `get`, which returns `exact` when visible;
- `reference`, normalised by `BankRegistry.detect(q)?.displayReference ?? q.toUpperCase()`;
- `external_id` as given;
- `payer_name`, exactly, after the normalisation in doc 02.

Results are merged by ID, newest first, at most 20.

### 6.7 Overview rules

- **Day boundaries:** EAT midnight. "This week" is the last 7 EAT days including today. The response is cached 30 s per tenant and env.
- **Banner:** banks with public status `down` where `ReceivingAccountsApi.listVerified(tenant, 'live')` has an account. `since` comes from `BankHealth.snapshot()`. Degraded banks are named only when there is a down bank. In test env the banner never shows, because test never calls a bank.
- **Checklist states:**

  | Step | `done` when | `in_progress` when |
  | --- | --- | --- |
  | `kyb` | `kyb_status = approved` | `submitted` |
  | `receiving_account` | `hasVerified(tenant, 'live')` | any live account exists that isn't verified |
  | `test_check` | Any test verification exists | n/a |
  | `api_key` | `ApiClientsApi.countActive(tenant, 'live') > 0` | a test key exists |
  | `webhook` | `WebhooksApi.health(tenant, 'live').enabled > 0` | a test endpoint exists |

### 6.8 Exports

| Column (en) | Source | Notes |
| --- | --- | --- |
| Verification ID | `id` | |
| Created at | `created_at` | ISO 8601 with `+03:00` |
| Status, Reason, Review reason, Assurance | | Codes plus a label column in the export's locale |
| Channel, Branch, Created by | | User or API client name |
| Bank, Reference, Official reference | | |
| Expected amount, Credited amount, Fee | | CSV: `1500.00`; Excel: numeric cells with `#,##0.00` |
| Payer name, Payer phone | | Phone masked, as in the API |
| Paid at, Paid into | | Account masked |
| Order ID, Metadata | `external_id`; `metadata` as compact JSON | |
| Completed at | | |
| Ethiopian date | | Only when the export locale is `am` |

- **Header row** in the requester's locale. Values are locale-neutral, except the label columns.
- **CSV:** UTF-8 with BOM (so Excel shows Amharic), CRLF, RFC 4180 quoting. Any cell starting with `=`, `+`, `-`, `@`, tab or CR is prefixed with `'` (formula injection).
- **Excel:** `exceljs` streaming writer, one sheet "Verifications", frozen header, auto-filter, dates as Excel date-times in EAT.
- **Processing:** the `exp:{id}` job runs with a `system` principal carrying the export's tenant, env and stored `scope`. It pages `VerificationApi.list` 500 rows at a time and streams to MinIO bucket `exports` (key `t/{tenantId}/{env}/{yyyy}/{mm}/{exportId}.{csv|xlsx}`). Names are resolved in batches through `TenancyApi.getUser` and `ApiClientsApi.listForTenant`, cached per job.
- **Limits:** a range of at most 92 days; at most 200,000 rows (checked first with `VerificationApi.count`, cap 200,001); 2 running exports per tenant (more wait in the queue); 10 per user per day.
- The file name is `verifications_{tenant display name slug}_{env}_{yyyymmdd-HHmm}.{csv|xlsx}`.

### 6.9 Polling and freshness

| What | Interval | Stops when |
| --- | --- | --- |
| Badges | 60 s | Tab hidden |
| Overview | 30 s | Tab hidden |
| Log page | 30 s (current page only) | Tab hidden, or the user has moved past page 1 |
| Open drawer, non-final | 5 s | Final status |
| Verify card, non-final | 3 s for 2 min, then 15 s up to 30 min | Final status or the card is closed |
| Export status | 3 s | `ready` or `failed` |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Queues` (`exports`), `Outbox`, `OnEvent`, `ObjectStore` (`exports`), `AuditLog`, `Notify`, `RateLimiter`, `Clock`, `i18n` |
| 01 tenancy | `Me` (through `GET /me` composition), `TenancyApi.getTenant`, `getSettings` (`review.mode`), `getUser`, `listBranches`, `isTenantActive`; `SessionService` (timeouts) |
| 02 receiving accounts | `hasVerified`, `listVerified` |
| 03 API clients | `ApiClientsApi.countActive`, `listForTenant` (names) |
| 04 bank connectors | `BankRegistry.list`, `detect`; `BankHealth.snapshot`, `publicStatus`. New edge `portal → bank-connectors` |
| 05 verification | `list`, `get`, `timeline`, `stats`, `rejectionReasons`, and the new `count(q, scope, cap)`. The portal's create, cancel and raw routes are doc 05's |
| 06 receipts | `latestForVerification` (drawer thumbnail); the upload-link route is doc 06's, and the receipt upload route is doc 05's (`addReceipt`) |
| 07 review | `ReviewApi.counts`, `openTaskForVerification` |
| 08 webhooks | `WebhooksApi.deliveriesForVerification`, `health` |
| 12 billing | `PlansApi.getEffectivePlan` (name, `reviewSlaMinutes`); `UsageApi.currentPeriod(tenantId)` (meter, state, `blockedReason`) |
| Contracts | `SANDBOX_REFERENCES` (the magic-reference catalogue shared with doc 04's sandbox connector) |

The drawer's fan-out runs in parallel, with a 2 s timeout per part. A part that fails or times out is returned as `null` with a `degraded: ["webhooks"]` list, so the drawer still opens and shows "Webhook history unavailable right now".

## 8. Implementation plan

1. **T09.1** `libs/ui` additions this doc relies on: StatusBadge, AssuranceChip, ReasonText, Money, BankChip, CopyRef, MaskedValue, DataTable (server paging, column picker), DetailDrawer, Timeline, EmptyState, ErrorBanner with request ID, skeletons. Includes the token contrast test.
2. **T09.2** BFF: bootstrap, badges, nav rules, route guard middleware in the web app.
3. **T09.3** Shell: sidebar, top bar (env switch, language, search, docs, user menu), test-mode bar, quota banner, session timer, re-auth dialog, Forbidden and Not found pages; responsive collapse at 360 px.
4. **T09.4** Overview service and page: KPIs, banner, reversals banner, recent, rejections, banks, setup checklist.
5. **T09.5** Verify page: form, detection, accounts, CBE Birr phone, upload with client resize, submit with idempotency, the result card in five states, receipt upload and customer link, gates, print slip.
6. **T09.6** Verifications log: filters in the URL, built-in views, counts, paging, column picker, Find.
7. **T09.7** Saved views (API and UI).
8. **T09.8** Detail drawer: composition endpoint, check texts, timeline merge, actions, raw panel, receipt viewer.
9. **T09.9** Exports: API, processor, CSV and XLSX writers, notifier, Exports list.
10. **T09.10** Sandbox page.
11. **T09.11** Cashier view in the portal (nav, My checks today).
12. **T09.12** i18n: every key in EN and AM, Ethiopian dates, Playwright Amharic screenshots; axe accessibility checks.
13. **T09.13** RUM endpoint and GlitchTip browser SDK (PII scrubbing on).

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| The user switches tenant or env with the drawer open | The drawer closes, filters reset, and the bootstrap reloads |
| The language changes mid-form | The form keeps its values; validation messages re-render in the new language |
| A verification changes status while its row is visible | The next 30 s refresh updates the row; the open drawer polls every 5 s |
| "Today" at 00:05 EAT | KPIs show the new day (near zero); "Yesterday" is one click away |
| The pasted reference has spaces, lowercase letters or is a full SMS | Detection normalises it; the server re-parses it anyway (doc 04) |
| Camera permission denied | "Allow camera access in your browser settings, or choose a file instead." |
| A file over 10 MB, or a HEIC photo | Refused in the browser with the size or type message, before upload; the server checks again (doc 06) |
| Double click on Verify | Same idempotency key, so one verification; the second response replays the first |
| Network drops while waiting | The card shows "Connection lost. We'll show the result when you're back online" and resumes polling with the verification ID (known from the `202` or a retried replay) |
| The bank answers after the 15 s wait | The card is `processing`, then updates by polling |
| A tenant with no receiving account at the detected bank | `422 no_receiving_account`: "You haven't added a {bank} account yet", linking to Receiving accounts |
| Quota blocked during a session | The next Verify returns `402`, and the panel replaces the form; badges and the meter update on the next bootstrap |
| Export while the plan changes | The export isn't affected; exports don't count toward quota |
| Export file deleted early by maintenance | `409 invalid_state` (`details.reason: "expired"`), shown as "This export has expired. Create a new one." |
| A shared saved view refers to a branch that was archived | The filter chip shows "Archived branch" and still filters by its ID |
| Very long payer names or Amharic names | Cells truncate with an ellipsis and a full-text tooltip; the drawer wraps |
| A drawer fan-out part fails | That section shows its own error line; the drawer stays usable (section 7) |
| Bootstrap fails | Full-page error with the request ID and "Try again"; nothing else renders |

## 10. Security and privacy

| Route | AuthN | AuthZ |
| --- | --- | --- |
| `/portal/v1/*` (this doc) | Portal session plus CSRF on mutations | Section 5.1 permissions; data scope from `AccessResolver` (cashiers get `own`); RLS |

- **Security headers** from `next.config.ts` and portal-api:
  - `Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{n}'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob: {STORAGE_PUBLIC_ORIGIN}; connect-src 'self' {GLITCHTIP_ORIGIN}; frame-ancestors 'none'; base-uri 'none'; form-action 'self'`;
  - `Referrer-Policy: same-origin`, `X-Content-Type-Options: nosniff`, `Permissions-Policy: camera=(self), microphone=(), geolocation=()`, and HSTS for 1 year with `includeSubDomains`.
- **No PII in URLs:** payer names and Find text never go into the browser URL. Search is a `POST`, and portal-api's access log records paths without query strings. The `payer_name` filter is sent as a query parameter only through `fetch`, never in the address bar.
- **Masking:** payer phones and accounts are masked everywhere in the portal. Unmasking calls doc 05's `POST /portal/v1/verifications/{id}/reveal` (`VerificationApi.revealPayer`), which needs `pii.reveal` plus re-auth and is audited.
- **Raw API response:** only for `verifications.read.raw`. It is the exact public JSON, so it carries nothing the API doesn't.
- **Exports** contain payer names, so they need `verifications.export`. Only the requester can download, through a 300 s signed URL. Each download is audited (`verifications.exported`, with the row count). Files are deleted after 7 days.
- **Cashiers** get the `own` data scope from doc 01. The BFF never widens it, and the composition endpoint passes the same scope to every module.
- **Client errors** sent to GlitchTip have request bodies, URLs' query strings and any field named in the logger's redaction list removed before sending.

## 11. Performance

| Item | Target |
| --- | --- |
| Bootstrap | p95 under 150 ms (all parts cached: settings 30 s, usage 60 s, plan 60 s) |
| Overview | p95 under 400 ms; under 50 ms from cache |
| Log page (25 rows) | p95 under 300 ms; count under 300 ms for ranges up to 31 days |
| Drawer detail | p95 under 400 ms (parallel fan-out) |
| Search | p95 under 300 ms |
| Verify | Server time is the bank's plus at most 150 ms (overview 3.8) |
| Export | 10,000 rows in under 30 s; 200,000 rows in under 6 min |
| Web | LCP under 2.5 s at p75 on a mid-range Android over 4G; initial JS under 350 KB gzip for the shell plus Overview; tables virtualised beyond 100 rows (UI spec) |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Nav rules for each role and review mode; greeting hours; banner rules (none, one down, two down plus slow, test env); checklist states; `check-text.ts` for every check and result; row notes; timeline merge ordering with same-second entries; CSV escaping (BOM, CRLF, quotes, formula prefixes, Amharic); XLSX cell types; token contrast ≥ 4.5:1 |
| Integration | Bootstrap with `live_enabled` combinations; overview against fakes of docs 02, 04, 05, 07, 12; drawer degraded part; search across the four lookups and the `own` scope; saved-view limits and visibility; export job end to end with Testcontainers MinIO (CSV and XLSX opened and parsed back), the 200,000-row guard, expiry |
| End to end | Playwright with the sandbox (doc 04): Owner overview; Verify with `TEST_VERIFIED`, `TEST_DUPLICATE`, `TEST_BANK_DOWN` (upload a receipt from the card), `TEST_REVIEW_APPROVE` (amber card turns green after the sandbox reviewer), `TEST_SLOW` (processing then verified); the drawer for `TEST_DUPLICATE` shows "First claimed by the sandbox seed" without a link (the linked first-claim box is covered by the drawer integration test with a fake of doc 05); save and share a view; export and download; cashier sees only Verify and My checks; Forbidden page on a direct URL |
| Accessibility | axe on every portal page in both locales (no serious or critical issues); keyboard-only walk-through of Verify and the drawer |
| Amharic | Screenshots of Overview, Verify (each card), log and drawer in `am`; Ethiopian dates against the overview 5.7 vectors; a check for missing keys |
| Visual | Playwright screenshots of the three drawn portal screens compared with the mockups at 1440 px; reviewed by a person, not pixel-matched |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_portal_bff_duration_seconds` | histogram | `route` |
| `pl_portal_drawer_degraded_total` | counter | `part` |
| `pl_portal_exports_total` | counter | `format`, `status` |
| `pl_portal_export_duration_seconds` | histogram | `format` |
| `pl_portal_export_rows` | histogram | none |
| `pl_portal_web_vitals` | histogram | `metric` (LCP, INP, CLS, TTFB), `route` |
| `pl_portal_client_errors_total` | counter | `route` |

| Alert | Condition | Severity |
| --- | --- | --- |
| BFF latency | Overview or log p95 over 800 ms for 10 min | Warning |
| Export failures | More than 3 failed in 1 h | Warning |
| Drawer degraded | More than 5% of drawer loads degraded over 15 min | Warning |
| Web vitals | LCP p75 over 2.5 s for 24 h | Info (weekly review) |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `PORTAL_EXPORT_MAX_ROWS` | 200000 | Export ceiling |
| `PORTAL_EXPORT_MAX_RANGE_DAYS` | 92 | Export range |
| `PORTAL_EXPORT_TTL_DAYS` | 7 | File retention |
| `PORTAL_EXPORTS_PER_USER_PER_DAY` | 10 | Rate limit |
| `PORTAL_EXPORTS_CONCURRENT_PER_TENANT` | 2 | Running at once |
| `PORTAL_SAVED_VIEWS_PER_USER`, `PORTAL_SHARED_VIEWS_PER_TENANT` | 20, 50 | View limits |
| `PORTAL_OVERVIEW_CACHE_SECONDS` | 30 | Overview cache |
| `PORTAL_COUNT_CAP` | 10000 | Log total cap |
| `PORTAL_DRAWER_PART_TIMEOUT_MS` | 2000 | Fan-out timeout per part |
| `DOCS_BASE_URL` | none (required) | Docs link |
| `STORAGE_PUBLIC_ORIGIN` | none (required) | Origin of signed object URLs, for the CSP |
| `NEXT_PUBLIC_GLITCHTIP_DSN` | empty (off) | Browser error reporting |

Client constants (in `apps/web-portal/src/config.ts`): the polling intervals in 6.9, 300 ms detection debounce, 1,600 px image resize, 10 MB file limit, 2-minute session warning.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 09-Q1 | The mockup's meter says "API checks this month", but accepted rule R12 counts every channel. | "Checks this month"; the label follows the plan's metered channels (doc 12) |
| 09-Q2 | Should exports be visible to everyone in the tenant with `verifications.export`, or only to the requester? | Requester only |
| 09-Q3 | Is 7 days long enough to keep export files? | 7 days |
| 09-Q4 | Do merchants want in-app notifications (a bell) besides email? | Email only at launch |
| 09-Q5 | Print slip widths: is 58 mm thermal paper common among target shops? | 80 mm and A4; 58 mm later |
| 09-Q6 | Should Platform-mode tenants see a read-only Review queue page? | No; the log's "In review" filter covers it |
| 09-Q7 | The drawer mockup shows "Date window" failing for a payment made 20 h before the order. Under R4's default 24 h window it would pass. | The drawer shows real check results; the mockup's sample is illustrative only |
