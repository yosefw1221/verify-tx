# 10 · Cashier app

| | |
| --- | --- |
| Package | `@proofline/cashier` (BFF) and `apps/web-portal/app/cashier` (installable PWA) |
| Postgres schema | `cashier` |
| Depends on | platform, 01 tenancy, 02 receiving accounts (`hasVerified`, `get`), 04 bank connectors (`detect`, `publicStatus`), 05 verification (including `addReceipt`), 07 review (`openTaskForVerification` for the wait time). Adds the edges `cashier → receiving-accounts`, `cashier → bank-connectors` and `cashier → review` to the overview DAG; none of them points back, so there is no cycle |
| Build step | 8 (in parallel with 09 and 11) |
| Mockups | Cashier · Scan (QR viewfinder, typed fallback, "telebirr detected", amount, Verify, "Today · 14 checks"); Cashier · Verified (full-screen green); Cashier · Do not release (አማ, full-screen red, Ethiopian date) |

## 1. Purpose and scope

**Does**

- Gives counter staff a phone screen that answers one question with a customer waiting: hand over the goods or not.
- **Device enrolment:** sign in once with the normal account (doc 01), set a 6-digit PIN, then unlock with the PIN. The session is bound to the device and can be revoked from Team.
- **Scan:** QR scanning with the torch, a gallery image, or a typed or pasted reference; live bank detection; amount entry on a numeric keypad.
- **Result:** a full-screen result in the status colour, with sound and vibration per outcome, plus the receipt-needed and in-review follow-ups.
- **History:** today's checks by this person, with totals.
- Ships as an installable PWA under `/cashier` on the portal origin; there is no app-store release.

**Out of scope**

- The verification rules (doc 05) and receipt reading (doc 06).
- The portal's own cashier view (Verify plus My checks inside the full portal), which is doc 09.
- Printing from the phone (open question 10-Q5).
- Test mode. The cashier app is live-only (doc 01, section 6.7), so training happens in the portal's Sandbox (open question 10-Q2).

## 2. Screens and user stories

| Screen | Mockup |
| --- | --- |
| Sign in, Set PIN, Unlock | Not drawn; the upload page's card layout at 360 px |
| Scan | Cashier · Scan |
| Result: Verified | Cashier · Verified |
| Result: Do not release | Cashier · Do not release (አማ) |
| Result: Wait (in review), Scan the receipt (receipt needed), Still checking | Not drawn; same full-screen layout in amber, blue and slate (UI spec "Mobile verify screen") |
| History | Not drawn; UI spec "History tab" |
| Offline and error banners | Not drawn |
| Devices list on a cashier's Team entry | Not drawn; a section in doc 01's Team drawer |

**US-10.1 Enrol a device**

- Given a user with `verifications.create` in an `active` tenant opens `/cashier` with no cashier session, they sign in with email and password, and MFA if their account has it (doc 01 `/auth/login` and `/auth/mfa/verify`).
- `POST /cashier/devices` then creates the device and the `cashier_device` session (cookie `pl_cashier`). The app ends the temporary portal session (`POST /auth/logout`).
- A Cashier's device uses their membership branch. Other roles pick a branch from a list during enrolment.
- The user gets an email: "A new cashier device signed in: Android · Chrome, Bole branch".

**US-10.2 Set the PIN**

- They enter a 6-digit PIN twice. PINs on the weak list (6.2) are refused with "Choose a less obvious PIN".
- On success the device is `active` and unlocked, and the Scan screen opens.

**US-10.3 Unlock**

- After 5 minutes without a request, or after "Lock" in the menu, the server treats the device as locked. Any call returns `401 device_locked`, and the app shows the Unlock screen with the person's name, the tenant and the branch.
- The correct PIN unlocks for another 5 minutes, sliding with use.
- After 5 wrong PINs in a row, the device is revoked, and the person must sign in with their password again. The screen shows how many tries are left from the 3rd failure on.

**US-10.4 Scan a QR**

- The camera fills the top half, pointed at the customer's phone. A frame is decoded about 8 times a second.
- When a QR decodes, the phone vibrates briefly, and the field fills with the detected reference with "telebirr detected" under it. Focus moves to the amount with the numeric keypad.
- "Torch" appears only when the camera supports it. When the camera is unavailable or permission is denied, the screen shows the typed field only, with "Camera unavailable. Type the reference instead."

**US-10.5 Type, paste or pick from the gallery**

- "Or type the reference or paste a link" accepts a bare reference, a share link or an SMS text, and detection runs after 300 ms.
- CBE Birr adds a "Payer phone" field.
- "From gallery" decodes a QR in the chosen image. If there is none, it offers "Verify with this receipt image", which sends the image as the receipt.

**US-10.6 Verify**

- "Amount to receive (ETB)" is required (at least 0.01, at most 2 decimals). Verify is enabled when a reference (or image) and an amount are present.
- Verify sends `POST /cashier/verifications` with a new idempotency key per tap sequence. The screen shows "Checking with the bank…", and the server waits up to 15 s.
- The result appears within 1 s of the response (UI spec).

**US-10.7 Verified**

- A full green screen shows the time, "VERIFIED", the amount in large type, the assurance chip and "Hand over the goods".
- Below: payer, phone (masked), bank, reference, "Paid: Today, 10:45", "Into: Habesha Mart · ****4401". It plays two rising tones and a short vibration. "Done" returns to Scan with the fields cleared.

**US-10.8 Do not release**

- A full red screen shows "Rejected", "DO NOT RELEASE" (Amharic: እቃውን አይስጡ) and the reason label. For a same-tenant duplicate it adds "First used: መስከረም 23, 2019 ዓ.ም. · 14:20" (Gregorian in English) and the earlier order.
- Below: amount, bank, reference, payer (none on `receiver_mismatch`, doc 05 section 6.9) and "Ask the customer to show another receipt." It plays a low double buzz and a long vibration pattern. "OK" returns to Scan.

**US-10.9 Wait (in review)**

- A full amber screen shows "WAIT · under review" and "Usually decided in about 18 min", taken from the review task's `dueAt`. In the second or two before the task exists, it shows "WAIT · under review" without a time.
- It polls every 10 s for up to 30 minutes and switches to the green or red screen with its sound when decided. "Check again" polls at once, and "Back to scan" leaves the check running; it stays in History.

**US-10.10 Scan the receipt (receipt needed)**

- A full blue screen shows "SCAN THE RECEIPT" and the reason ("Bank not responding", or "Receipt or share link needed").
- "Open camera" takes a photo (receipt mode, not QR mode), shows it with "Use this" or "Retake", and uploads it. The screen then shows "Reading receipt", "Contacting bank", and the outcome.
- "Let the customer upload" shows the upload link as a large QR for the customer's phone.

**US-10.11 Still checking**

- If the 15 s wait ends while the bank is still working, a slate screen shows "Still checking" with a spinner. It polls every 3 s for 2 minutes, then every 15 s, and never shows a success it didn't receive from the server.

**US-10.12 History**

- The History tab lists today's checks (EAT) by this person, newest first: time, status badge and note, amount, reference, payer.
- The top shows "14 checks · 12 verified · ETB 26,940.00 verified". The Scan screen's footer shows "Today · 14 checks".
- Tapping a row opens its result screen read-only, with "Check again" for non-final ones.
- When a pending check is decided, the History tab gets a dot.

**US-10.13 Offline**

- With no connection, a banner reads "No connection, cannot verify" and Verify is disabled.
- If the connection drops while a check is in flight, the screen shows "Connection lost. We don't know the result yet." "Check again" re-sends the same request with the same idempotency key, which replays the stored result (overview 5.16).

**US-10.14 Gates**

- Tenant not live-enabled: after unlock, the app shows "Live checks aren't on yet for Habesha Mart. Ask your account owner." and no Scan screen.
- Tenant suspended: "Checks are paused for Habesha Mart. Ask your account owner."
- Quota blocked (`402`): "This month's checks have run out. Ask your account owner."
- Lost permission (`403 forbidden`): the device is revoked and the app returns to sign-in with "Your access changed. Sign in again."

**US-10.15 Revoke from Team**

- An Owner or Admin opens a person in Team (doc 01) and sees their cashier devices: label, branch, enrolled, last seen. "Sign out this device" revokes it, and the device's next request returns `401 device_revoked`, which shows the sign-in screen.

**US-10.16 Language**

- The EN / አማ switch in the header changes the language at once and saves it to the user (`PATCH /cashier/me`). Amharic shows Ethiopian dates, `1,800.00 ብር`, and the 24-hour standard clock.

**US-10.17 Install and update**

- On Android Chrome, the app offers "Add to home screen" after the first verified check. It opens standalone in portrait.
- A new version shows "Update ready. Reload" on the Scan screen when no check is in flight. It never interrupts a result screen.

## 3. Module design

```text
libs/features/cashier/src/
├─ public/index.ts                CashierModule (BFF only)
├─ domain/
│  ├─ pin-policy.ts               length, weak list, attempts (pure)
│  ├─ headline.ts                 status → screen (6.4, pure)
│  └─ result-view.ts              VerificationView → CashierResult (pure)
├─ application/
│  ├─ devices.service.ts          enrol, set PIN, change PIN, unlock, lock, revoke, logout
│  ├─ unlock.guard.ts             Nest guard on /cashier/* (except session, unlock, pin set): device active and unlocked
│  ├─ verify.service.ts           create (channel cashier), get, receipt upload
│  ├─ history.service.ts
│  └─ membership.handler.ts       @OnEvent membership.removed / membership.updated → revoke devices; branch.archived → flag devices; tenant.status_changed (suspended, closed) → revoke all
├─ infrastructure/ devices.repo.ts, unlock.store.ts (Redis)
├─ http/portal/cashier.controller.ts           /portal/v1/cashier/*        (cookie pl_cashier)
├─ http/portal/cashier-devices.controller.ts   /portal/v1/cashier-devices/* (portal session, Team page)
└─ db/schema.ts, db/migrations/

apps/web-portal/app/cashier/
├─ layout.tsx          minimal shell; manifest link; no portal sidebar
├─ page.tsx            state router: sign-in → set-pin → unlock → scan / result / history
├─ manifest.webmanifest
└─ sw.ts               service worker (6.6)
apps/web-portal/src/features/cashier/
├─ scanner/            camera, BarcodeDetector or zxing-wasm worker, torch, wake lock
├─ result/             full-screen result views, sounds (Web Audio), vibration
├─ history/
└─ keypad/             amount input with inputmode="decimal" and an on-screen fallback
```

```ts
// domain/result-view.ts
export type CashierScreen = 'verified' | 'do_not_release' | 'wait' | 'scan_receipt' | 'checking';
export interface CashierResult {
  id: string; status: VerificationStatus; screen: CashierScreen;
  assurance: Assurance | null;
  reasonKey: string | null;                 // reason.*, receipt_reason.*, review_reason.* (doc 09 catalogue)
  amount: string;                           // credited if known, else expected
  bank: { code: BankCode; name: string } | null;
  reference: string | null;
  payer: { name: string | null; phoneMasked: string | null } | null;   // null on receiver_mismatch or no bank answer
  paidAt: string | null;
  paidInto: { label: string; accountMasked: string } | null;
  firstClaim: { externalId: string | null; claimedAt: string } | null; // same-tenant duplicate only
  reviewEtaMinutes: number | null;
  uploadUrl: string | null;                 // receipt_required only
  checkedAt: string;
}
```

## 4. Data model

```sql
CREATE TABLE cashier.devices (
  id                  uuid PRIMARY KEY,
  tenant_id           uuid NOT NULL REFERENCES tenancy.tenants(id),
  user_id             uuid NOT NULL,
  membership_id       uuid NOT NULL,
  branch_id           uuid NOT NULL,
  session_id          uuid NULL,               -- current tenancy.sessions row (kind cashier_device)
  label               text NOT NULL CHECK (length(label) BETWEEN 1 AND 60),   -- "Android · Chrome 129", editable
  status              text NOT NULL CHECK (status IN ('pending_pin','active','revoked')),
  pin_hash            text NULL,               -- argon2id, same parameters as passwords (doc 01)
  pin_set_at          timestamptz NULL,
  pin_failures        smallint NOT NULL DEFAULT 0,
  revoked_at          timestamptz NULL,
  revoked_by_user_id  uuid NULL,
  revoke_reason       text NULL CHECK (revoke_reason IN ('team','logout','pin_failures','device_limit','membership_changed',
                        'tenant_status','permission_lost')),
  created_at          timestamptz NOT NULL DEFAULT now(),
  last_seen_at        timestamptz NULL,
  last_ip             inet NULL,
  user_agent          text NULL CHECK (length(user_agent) <= 300)
);
CREATE INDEX devices_user ON cashier.devices (tenant_id, user_id) WHERE status <> 'revoked';
```

- **Env-less**, like tenancy tables: the cashier app is live-only. RLS checks `tenant_id`.
- **Unlock state** lives in Redis, not the database: key `pl:cashier:unlock:{deviceId}` with a TTL of 300 s, refreshed on each request (at most once every 15 s). Lock deletes the key. With Redis down, every request is treated as locked, which fails safe.
- `last_seen_at` is updated at most once a minute.
- **Migration:** `0001_cashier_init.sql`.

## 5. API contracts

### 5.1 Cashier API (`/portal/v1/cashier`, cookie `pl_cashier`)

Every mutating call needs `X-CSRF-Token`; the token comes from `GET /session`, `POST /pin` and `POST /unlock` and is kept in memory only. "Unlocked" means the device is `active` and the unlock key exists.

| Method and path | Needs | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `POST /devices` | Portal session (`pl_session`), `verifications.create`, tenant `active` | `{ branch_id? }` (required unless the role is Cashier) | `201 { device: { id, label }, status: "pending_pin" }`; sets `pl_cashier` | `400 missing_field` (`branch_id`); `403 forbidden`; `403 tenant_suspended`; `403 live_not_enabled` |
| `GET /session` | `pl_cashier` (locked is fine) | none | `200 CashierSession` | `401 device_revoked`; `401 unauthenticated` |
| `POST /pin` | Device `pending_pin` | `{ pin }` | `200 { unlocked_until, csrf_token }` | `400 weak_pin`; `409 invalid_state` |
| `POST /unlock` | Device `active` | `{ pin }` | `200 { unlocked_until, csrf_token }` | `401 pin_invalid` (`details.attempts_left`); `401 device_revoked` (5th failure); `429 rate_limited` |
| `POST /lock` | Unlocked | none | `204` | none |
| `POST /pin/change` | Unlocked | `{ current_pin, new_pin }` | `204`; audited | `401 pin_invalid`; `400 weak_pin` |
| `POST /logout` | `pl_cashier` | none | `204`; revokes the device (`logout`) and clears the cookie | none |
| `PATCH /me` | Unlocked | `{ locale }` | `200` | `400 invalid_field` |
| `POST /detect` | Unlocked | `{ input }` | `200` (same shape as doc 09's `POST /verify/detect`) | `400 invalid_field` |
| `POST /verifications` | Unlocked | JSON `{ reference, bank_code?, payer_phone?, amount, note? }` or multipart `data` + `file`; header `X-Idempotency-Key` (UUIDv4) | `200 CashierResult` (final) or `202 CashierResult` | `400 missing_field`, `invalid_field`, `invalid_amount`; `402 quota_exceeded`; `403 live_not_enabled`, `tenant_suspended`; `409 idempotency_conflict`, `idempotency_in_progress`; `413`; `415`; `422 unsupported_bank`, `bank_not_detected`, `invalid_reference`, `no_receiving_account`, `file_rejected` |
| `GET /verifications/{id}` | Unlocked; created by this user | none | `200 CashierResult` | `404` |
| `POST /verifications/{id}/receipt` | Unlocked; created by this user | multipart `file` | `202 CashierResult` (through `VerificationApi.addReceipt`, `source: cashier`) | `409 invalid_state` (not `receipt_required`); `413`; `415`; `422 file_rejected` |
| `GET /history` | Unlocked | `?cursor` | `200 { date, totals: { count, verified_count, verified_amount, rejected_count, pending_count }, items: CashierRow[], next_cursor }` | none |

```json
// CashierSession
{ "device": { "id": "dev_01J…", "label": "Android · Chrome 129", "status": "active" },
  "locked": true,
  "user": { "full_name": "Liya Kebede", "locale": "am" },
  "tenant": { "display_name": "Habesha Mart", "live_enabled": true, "status": "active" },
  "branch": { "name": "Bole" },
  "banks": [ { "code": "MPESA", "name": "M-Pesa", "status": "down" } ],
  "today_count": 14,
  "csrf_token": "b64url…" }
```

When locked, `today_count` and `banks` are null, so a locked phone shows names only.

```json
// CashierResult (Do not release, Amharic mockup)
{ "id": "ver_01JB7R4N8C2E6G0J3L5Q7S9U1W", "status": "rejected", "screen": "do_not_release", "assurance": null,
  "reason_key": "reason.duplicate", "amount": "1800.00", "bank": { "code": "TELEBIRR", "name": "telebirr" },
  "reference": "DET8FJGUJ4", "payer": { "name": "BIRUK ASFAW", "phone_masked": "+251 9•• ••• 507" },
  "paid_at": "2026-10-03T14:18:00+03:00", "paid_into": { "label": "Habesha Mart", "account_masked": "****4401" },
  "first_claim": { "external_id": "88102", "claimed_at": "2026-10-03T14:20:00+03:00" },
  "review_eta_minutes": null, "upload_url": null, "checked_at": "2026-10-04T10:39:03+03:00" }
```

`note` is stored as `external_id`. `channel` is always `cashier`, `env` always `live`, and the actor is `{ kind: 'user', userId, branchId: device.branch_id }`, or the membership branch for Cashiers. `expected.receiving_account_id` is omitted, so any of the tenant's verified accounts at the bank matches (doc 05).

### 5.2 Team page (`/portal/v1/cashier-devices`, portal session)

| Method and path | Permission | Notes |
| --- | --- | --- |
| `GET /cashier-devices?user_id=` | `team.read` | Non-revoked devices: id, label, branch, created, last seen |
| `POST /cashier-devices/{id}/revoke` | `team.manage` | Revokes the device and its session (`team`); audited |

### 5.3 New error codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `device_locked` | 401 | PIN unlock needed |
| `device_revoked` | 401 | Device signed out (Team, logout, PIN failures, access change); sign in again |
| `pin_invalid` | 401 | Wrong PIN; `details.attempts_left` |
| `weak_pin` | 400 | PIN fails 6.2 |

### 5.4 Events

**Emits:** none (enrolment emails go straight through `Notify`). **Consumes** (all from doc 01):

- `membership.removed`: revoke the user's devices.
- `membership.updated`: revoke when the new role lacks `verifications.create`, or when a Cashier's branch changes.
- `branch.archived`: devices of non-Cashier users on that branch must pick a new branch at the next unlock.
- `tenant.status_changed`: revoke all devices on `suspended` or `closed`.

## 6. Business rules

### 6.1 Sessions and locking

| Rule | Value |
| --- | --- |
| Session | Doc 01 kind `cashier_device`: cookie `pl_cashier`, path `/api/portal/v1/cashier`, `SameSite=Strict`, 30 days absolute |
| Unlock window | 300 s, sliding on each request (`CASHIER_UNLOCK_SECONDS`) |
| Wrong PINs | 5 in a row revoke the device (`CASHIER_PIN_MAX_FAILURES`); the count resets on success |
| Unlock rate | At most 10 unlock attempts per device per 15 min, on top of the failure count |
| Devices per user | 3 active. Enrolling a 4th revokes the least recently seen one (`device_limit`) and says so in the enrolment email |
| Env | Always `live` |
| Data scope | History and get: `created_by_user_id = me` and `channel = cashier`, whatever the role |

The unlock guard runs on every route except `GET /session`, `POST /pin`, `POST /unlock` and `POST /logout`. It loads the device (cached 30 s, cleared on revoke), checks `active` and the unlock key, then refreshes the key.

### 6.2 PIN policy

- Exactly 6 digits.
- Refused if all digits are the same; if it is an ascending or descending run (`123456`, `654321`, `012345`); or if it is on the weak list: `121212`, `112233`, `123123`, `159753`, `147258`, `696969`, `101010`, `202020`, `131313`, `789456`, `456789`, `987654`, `102030`, `000000`, `111222`.
- Stored as argon2id. It is never logged and never sent anywhere but `POST /pin`, `/unlock` and `/pin/change`.

### 6.3 Detection and input

- `POST /detect` calls `BankRegistry.detect(input)` and `needsPayerPhone` for CBE Birr. A decoded QR's raw text goes through the same call, so QR formats are known in one place (doc 04).
- A multipart verification (gallery image or receipt photo) goes through `VerificationApi.create` with `receipt`. Images are resized on the phone to 1,600 px on the longest side, JPEG quality 0.85.

### 6.4 Result screens

| Status | Screen | Colour | Headline (EN / AM) | Sub-line | Sound and vibration | Buttons |
| --- | --- | --- | --- | --- | --- | --- |
| `verified` | `verified` | Green `#1F8A4C` | VERIFIED / ተረጋግጧል | "Hand over the goods" / እቃውን ያስረክቡ † | Two rising tones (880 Hz then 1,320 Hz, 120 ms each); vibrate `[80]` | Done |
| `rejected`, `reversed` | `do_not_release` | Red `#C2362F` | DO NOT RELEASE / እቃውን አይስጡ | The reason label; then "Ask the customer to show another receipt." / ደንበኛው ሌላ ደረሰኝ እንዲያሳይ ይጠይቁ። | Two low buzzes (220 Hz square, 300 ms, 100 ms apart); vibrate `[300, 100, 300]` | OK |
| `in_review` | `wait` | Amber `#B7791F` | WAIT · under review / ይጠብቁ · በግምገማ ላይ † | "Usually decided in about {n} min" | One mid tone (660 Hz, 200 ms); vibrate `[150]` | Check again, Back to scan |
| `receipt_required` | `scan_receipt` | Blue `#2F6FDB` | SCAN THE RECEIPT / ደረሰኙን ይቃኙ † | `receipt_reason.*` | One mid tone; vibrate `[150]` | Open camera, Let the customer upload |
| `processing` | `checking` | Slate `#5B6B7A` | Still checking / አሁንም እየተረጋገጠ ነው † | "The bank is slow. This screen updates by itself." | none | Back to scan |
| `expired`, `cancelled` | `do_not_release` | Grey `#5F6870` | NOT VERIFIED / አልተረጋገጠም † | The expired reason, or "Cancelled" | Low buzz | OK |

- † marks Amharic drafts for the translator review (overview Q10).
- Large headline text is white on the status colour. Each pair is checked at 4.5:1 or more for the 32 px bold headline (3:1 is the large-text minimum, and we exceed it).
- Sound can be muted from the menu, stored per device in `localStorage`. Vibration follows the device setting.
- **Review ETA:** `ceil((task.dueAt − now) / 60 s)`, at least 1, from `ReviewApi.openTaskForVerification`. With no task yet (event lag), `reviewEtaMinutes` is `null` and the screen leaves out the time until the next poll.

### 6.5 Camera and scanning

| Item | Rule |
| --- | --- |
| Stream | `getUserMedia({ video: { facingMode: 'environment', width: { ideal: 1280 }, height: { ideal: 720 } } })` |
| Decoder | `BarcodeDetector` (`qr_code`) when available; else `zxing-wasm` in a Web Worker, lazy-loaded and cached by the service worker |
| Rate | About 8 decodes a second at a 640 px-wide downscaled frame; a decode is ignored if it repeats the last one within 3 s |
| Torch | `track.applyConstraints({ advanced: [{ torch: true }] })` when `getCapabilities().torch` is true |
| Battery | The stream stops when the screen is hidden, on a result screen, or after 60 s with no decode (tap to resume) |
| Wake lock | `navigator.wakeLock.request('screen')` while Scan is visible, released otherwise |
| Receipt photo mode | A still capture at full resolution, then resized (6.3) |

### 6.6 PWA

- **Manifest:** `name` "{Product} Cashier", `start_url` `/cashier`, `scope` `/cashier/`, `display` `standalone`, `orientation` `portrait`, `theme_color` `#0F6E6A`, `background_color` `#F6F7F5`, 192 and 512 px maskable icons.
- **Service worker:** precaches the app shell (JS, CSS, the Plex and Noto Sans Ethiopic font subsets, icons, the zxing-wasm bundle). Every `/api/*` request is network-only, so no API response is ever cached and nothing personal is stored on the phone. An offline navigation gets a cached page that says "No connection, cannot verify".
- **Updates:** a new worker waits. The app shows "Update ready. Reload" only on Scan with no check in flight.
- **Supported:** Android 8+ with Chrome or Samsung Internet (current and previous major); iOS Safari 16.4+ (open question 10-Q6).

### 6.7 Numbers

| Rule | Value | Where |
| --- | --- | --- |
| Server wait | 15 s | `CASHIER_WAIT_SECONDS` |
| Poll: checking | 3 s for 2 min, then 15 s up to 30 min | Client constant |
| Poll: wait | 10 s up to 30 min | Client constant |
| History refresh | 30 s while visible | Client constant |
| Unlock window | 300 s | `CASHIER_UNLOCK_SECONDS` |
| PIN failures | 5 | `CASHIER_PIN_MAX_FAILURES` |
| Devices per user | 3 | `CASHIER_MAX_DEVICES_PER_USER` |
| History page | 50 rows | `CASHIER_HISTORY_PAGE` |

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Redis`, `Outbox`/`OnEvent`, `AuditLog`, `Notify`, `RateLimiter`, `Clock`, `ids` (`dev`), `pii` (argon2id via `crypto`) |
| 01 tenancy | `SessionService.create` (kind `cashier_device`, `deviceId`), `revoke`; `TenancyApi.getTenant`, `getUser`, `getBranch`, `isTenantActive`; `AccessResolver` (permissions); events `membership.*`, `branch.archived`, `tenant.status_changed`. Sign-in itself is doc 01's `/auth/login` |
| 02 receiving accounts | `hasVerified` (live gate on the session), `get` (the "Into" label) |
| 04 bank connectors | `BankRegistry.detect`; `BankHealth.publicStatus` (down chips on the session) |
| 05 verification | `create` (channel `cashier`, wait 15), `get`, `list` (history, own), `addReceipt` (receipt photo, `source: cashier`) |
| 07 review | `ReviewApi.openTaskForVerification` (wait time) |

## 8. Implementation plan

1. **T10.1** Migration and RLS; device repository.
2. **T10.2** `pin-policy.ts`, `headline.ts`, `result-view.ts` with unit tests for every status, reason and masking case.
3. **T10.3** Devices service: enrol (from a portal session), set PIN, unlock with the failure count and rate limit, lock, change PIN, logout, revoke, the device-limit rule; the enrolment email.
4. **T10.4** Unlock guard and the Redis unlock store, including the Redis-down fail-safe.
5. **T10.5** Verify service: create, get, receipt upload, history with totals; idempotent replay.
6. **T10.6** Membership and tenant handlers; the Team devices endpoints.
7. **T10.7** PWA shell: manifest, service worker, update flow, state router (sign-in, set PIN, unlock, scan, result, history).
8. **T10.8** Scanner: camera, `BarcodeDetector` and zxing fallback, torch, wake lock, gallery, receipt photo mode.
9. **T10.9** Result screens with sounds and vibration; keypad; offline handling.
10. **T10.10** EN/AM strings, Ethiopian dates; Playwright mobile tests and screenshot comparison with the three mockups.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| QR on the customer's screen is too small or glares | No decode; the hint "Move closer, or type the reference" appears after 8 s; torch and typed entry remain |
| QR from an unsupported bank or a random QR | Detection returns null: "We can't read this QR. Type the reference." No request is made |
| The cashier taps Verify twice | Same idempotency key within the tap sequence, so one verification |
| The phone sleeps during the 15 s wait | On wake, the app re-sends with the same key and gets the stored result |
| Two cashiers check the same receipt | The second gets DO NOT RELEASE with `duplicate` and the first order (same tenant) |
| The customer shows a screenshot of someone else's receipt | Duplicate or receiver checks catch it; the screen is red |
| Bank down | `scan_receipt` (blue) right away, because `BankHealth` says it isn't callable (doc 05 6.1) |
| Device clock wrong | Only server times are shown; "Today" is decided by the server in EAT |
| Redis down | Every request is treated as locked; unlock fails with `503 service_unavailable` ("Can't unlock right now"). Nothing is shown unverified |
| Cashier moved to another branch | Doc 01's `membership.updated` (with the new branch) revokes their devices; they sign in again and the new branch applies |
| Owner's device branch archived | `branch.archived` flags the device; the next unlock asks to pick a branch, and checks are blocked until one is chosen |
| A stolen phone | Locked after 5 min; 5 wrong PINs revoke it; an Owner can revoke it from Team at once. No results are stored on the phone |
| Camera used by another app | "Camera is busy. Close other apps or type the reference." |
| iOS standalone mode loses camera permission between launches | The app asks again on Scan; typed entry always works |

## 10. Security and privacy

| Route | AuthN | AuthZ |
| --- | --- | --- |
| `/portal/v1/cashier/*` | `pl_cashier` device session; unlock guard; CSRF on mutations | `verifications.create`; own data only; live only |
| `POST /portal/v1/cashier/devices` | Portal session | `verifications.create`; tenant `active` |
| `/portal/v1/cashier-devices/*` | Portal session | `team.read`, `team.manage` |

- **Cookie scope:** `pl_cashier` is scoped to the cashier API path and `SameSite=Strict`, so it never travels with portal or third-party requests.
- **Server-side lock:** the PIN lock is enforced by the server. A client that skips the lock screen still gets `401 device_locked`.
- **No PII at rest on the phone:** nothing is in `localStorage` except the mute flag, nothing is cached by the service worker, and `GET /session` returns names only while locked.
- **Masking:** payer phones are masked. There is no reveal in the cashier app.
- **Audited:** device enrolled, PIN set and changed, unlock lockout (revocation after 5 failures), device revoked (with who and why).
- **CSP** is as in doc 09, with `media-src 'self' blob:` and `worker-src 'self' blob:` for the scanner. `Permissions-Policy: camera=(self)`.

## 11. Performance

| Item | Target |
| --- | --- |
| Unlock to camera ready | Under 2 s on a mid-range Android (Android 10, 3 GB RAM class) |
| Result shown after the server answers | Under 1 s (UI spec); result components are preloaded at unlock |
| Cashier API overhead | p95 under 150 ms besides the bank (overview 3.8) |
| Initial JS for `/cashier` | Under 200 KB gzip, excluding the lazy zxing bundle (fetched once, then cached) |
| QR decode | First decode within 1.5 s of a QR entering the frame at arm's length |

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | PIN policy (every weak case); headline mapping for all 8 statuses; result view masking (`receiver_mismatch`, no bank answer, same-tenant versus other-tenant duplicate); review ETA rounding |
| Integration | Enrol from a portal session, set PIN, unlock, lock; 5 wrong PINs revoke; the unlock rate limit; the guard returns `device_locked` after 300 s with `FakeClock`; Redis down treated as locked; a Team revoke takes effect on the next request; device-limit eviction; membership and tenant events revoke; history totals in EAT across midnight |
| End to end | Playwright with Pixel 5 emulation and Chrome's fake camera fed a video of a sandbox QR (`TEST_VERIFIED`): green screen with the mockup's fields; `TEST_DUPLICATE` gives the red Amharic screen; `TEST_BANK_DOWN` gives blue, then a photo upload and an outcome; `TEST_REVIEW_APPROVE` goes amber then green; offline mode disables Verify; `TEST_SLOW` shows Still checking, then green |
| PWA | Lighthouse CI installability; a test that the service worker never caches `/api/*` |
| Amharic and visual | Screenshots of Scan, Verified and Do not release compared with the mockups by a person; Ethiopian dates against the overview vectors |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_cashier_unlock_total` | counter | `outcome` (ok, pin_invalid, revoked, rate_limited) |
| `pl_cashier_devices_active` | gauge | none |
| `pl_cashier_checks_total` | counter | `screen` |
| `pl_cashier_result_render_seconds` | histogram | none (client RUM: response to first paint of the result) |
| `pl_cashier_qr_decode_total` | counter | `decoder` (native, zxing), `outcome` (ok, unsupported) |

| Alert | Condition | Severity |
| --- | --- | --- |
| PIN lockouts | More than 20 lockout revocations in 1 h across tenants | Warning (possible guessing) |
| Result render | p95 over 1 s for 1 h | Warning |

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `CASHIER_WAIT_SECONDS` | 15 | Server wait |
| `CASHIER_UNLOCK_SECONDS` | 300 | Unlock window |
| `CASHIER_PIN_MAX_FAILURES` | 5 | Wrong PINs before revocation |
| `CASHIER_UNLOCK_ATTEMPTS_PER_15_MIN` | 10 | Unlock rate limit |
| `CASHIER_MAX_DEVICES_PER_USER` | 3 | Active devices |
| `CASHIER_HISTORY_PAGE` | 50 | History page size |

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 10-Q1 | The UI spec calls the amount "optional", but the amount check (R6) and the API need it. | Required; the mockup's field has no "optional" label |
| 10-Q2 | Should the cashier app have a training mode on the sandbox? | No; live only (doc 01). Training happens in the portal's Sandbox |
| 10-Q3 | Is a native Android app needed later, for better camera control and sound in silent mode? | PWA only |
| 10-Q4 | May Owners and Admins use the cashier app, or only the Cashier role (overview 5.3 says Cashier)? | Anyone with `verifications.create`, with a branch chosen at enrolment |
| 10-Q5 | Should the app print slips on Bluetooth thermal printers? | No; Web Bluetooth support is too uneven. The portal prints (doc 09) |
| 10-Q6 | How much iOS support is needed at launch? | iOS Safari 16.4+ best effort; Android is the target |
| 10-Q7 | The Amharic mockup shows the earlier order as "88102" while the portal shows "order_88102". | Show `external_id` exactly as stored |
