# 01 · Tenancy, auth and RBAC

| | |
| --- | --- |
| Package | `@proofline/tenancy` |
| Postgres schema | `tenancy` |
| Depends on | platform kernel only |
| Build step | 1 (section 7 of the overview) |
| Mockups | Portal sidebar tenant switcher, Live/Test toggle, Cashier-view menu (Portal · Verify a payment); admin user chip. Sign-in, Team, Settings, admin Tenants and Staff screens are not drawn; this doc defines them. |

## 1. Purpose and scope

**Does**

- Tenants (merchants), their KYB (know-your-business) record, and tenant status.
- The two environments, `live` and `test`, as a property of every tenant-scoped session.
- Branches.
- Tenant users, memberships and roles; platform staff and their roles.
- Portal sign-up, email verification, sign-in, TOTP MFA, recovery codes, password reset, re-authentication.
- Server-side sessions for the portal and the admin panel, plus the session record that the cashier app (doc 10) builds on.
- Invitations.
- The permission catalogue and role matrix, and the `AccessResolver` port that the platform's guards call.
- The tenant settings registry that every other feature reads.

**Out of scope**

- API-key authentication (doc 03).
- Cashier device registration and the PIN lock (doc 10). That doc reuses `SessionService` from here.
- Receiving accounts (doc 02) and plans and subscriptions (doc 12).
- Audit storage. This module calls `AuditLog.record()` from the platform.

## 2. Screens and user stories

| Screen | Surface | Status |
| --- | --- | --- |
| Sign up, verify email, sign in, MFA, forgot password | Portal | Not in the mockups; uses the portal design tokens and the same card layout as the upload page |
| Tenant switcher (sidebar) and Live/Test toggle (top bar) | Portal | Mockups: Portal · Overview and the other portal screens |
| Cashier-view menu (Verify plus My checks only) | Portal | Mockup: Portal · Verify a payment |
| Team (members, invitations) | Portal | Not drawn |
| Settings (tenant policies) | Portal | Not drawn |
| Tenants list and detail (KYB, suspend) | Admin | Not drawn |
| Staff & roles | Admin | Not drawn |

Acceptance criteria are written so each one maps to an automated test (section 12).

**US-01.1 Sign up**

- Given a visitor submits legal name, display name, email, phone, password and locale on `/signup`,
- when the email is new, then a tenant is created with `status = pending_kyb` and `kyb_status = not_submitted`, along with a user, an `owner` membership and a default branch named "Main", and a 6-digit code is emailed;
- when the email already belongs to a user, then the response is the same `202`, and that user receives a "you already have an account" email instead (no account enumeration).
- The response never reveals which case happened.

**US-01.2 Verify email**

- A correct code within 15 minutes and at most 5 attempts sets `email_verified_at`, signs the user in, and returns `mfa_enrollment_required: true`.
- A 6th wrong attempt invalidates the code (`400 invalid_token`).
- A resend sooner than 60 s returns `429 rate_limited`.

**US-01.3 MFA enrollment**

- An Owner, Admin or Developer who has not enrolled TOTP can reach only `/auth/mfa/totp/*` and `/auth/logout`. Every other portal route returns `403 mfa_enrollment_required`.
- Confirming with a valid code returns 10 recovery codes exactly once.

**US-01.4 Sign in**

- Correct email and password return `mfa_required: true` with a `challenge_id`. A correct TOTP or unused recovery code then sets the `pl_session` cookie.
- A user whose only memberships are Cashier signs in with password only, because no TOTP is required for Cashiers (section 6.2).
- Wrong credentials return `401 invalid_credentials`, without saying which part was wrong.

**US-01.5 Lockout**

- After 10 consecutive failed password or TOTP attempts, the account is locked for 15 minutes: `401 account_locked`, with `Retry-After` in seconds.
- A successful login resets the counter.

**US-01.6 Password reset**

- `forgot` always returns `202` and emails a 6-digit code if the user exists.
- `reset` with a valid code sets the new password and revokes every session the user has, including cashier device sessions.

**US-01.7 Switch Live/Test**

- A member with `env.switch` changes the session's env, and every following request uses it.
- A Cashier gets `403 forbidden`.
- Switching to `live` while the tenant is not live-enabled is allowed for viewing. Live actions are blocked by the live gate in docs 03 and 05.

**US-01.8 Switch tenant**

- A user with several active memberships switches the session's tenant.
- The env resets to `live` if the new tenant is `active`, otherwise to `test`.

**US-01.9 Invite**

- A member with `team.manage` invites an email with a role, plus a branch if the role is Cashier.
- The invitee gets a link valid for 7 days. Accepting creates the user if new (name and password), or asks an existing user to sign in, and then creates the membership.
- Inviting an existing member returns `409 already_member`. A Cashier invite without `branch_id` returns `400 missing_field`.

**US-01.10 Change role or remove member**

- Changing a role, changing a Cashier's branch, or removing a member takes effect on that user's next request: their cached session is deleted at once.
- Removing or demoting the last active Owner returns `409 last_owner`.

**US-01.11 Branches**

- Create, rename and archive branches.
- Archiving a branch that still has active Cashiers returns `409 branch_has_cashiers`.
- The default branch cannot be archived (`409 invalid_state`).

**US-01.12 Settings**

- Owners and Admins edit tenant policies (section 6.6).
- Values outside the allowed range return `400 invalid_field` naming the key.
- A successful change emits `tenant.settings_changed`, and readers see the new value within 30 s.

**US-01.13 KYB (tenant side)**

- Upload a trade licence and a TIN certificate (PDF, JPG or PNG, at most 10 MB each), then submit.
- Submitting with either document type missing returns `400 missing_field`.
- `kyb_status` becomes `submitted`.

**US-01.14 KYB (admin side)**

- A platform Admin opens the documents (each signed URL lives 60 s) and approves or rejects with a reason.
- Approval sets `kyb_status = approved` and `status = active`. Rejection sets `kyb_status = rejected` and emails the reason. The tenant can resubmit.

**US-01.15 Suspend and reactivate**

- A platform Admin suspends a tenant with a reason. Within 60 s, every API key for that tenant returns `403 tenant_suspended`, and the portal becomes read-only (mutations return `403 tenant_suspended`).
- Reactivating restores the previous status.

**US-01.16 Staff sign-in**

- Staff sign in through Keycloak with WebAuthn or TOTP enforced by Keycloak.
- A Keycloak user with no active `tenancy.staff` row gets `403 forbidden` and no session.
- An Admin can invite staff by email, set roles and disable staff. Disabling deletes their sessions.

## 3. Module design

```text
libs/features/tenancy/src/
├─ public/
│  ├─ index.ts                 TenancyModule, TenancyApi, SessionService, PERMISSIONS, ROLE_PERMISSIONS, TenantSettings type, events
│  ├─ tenancy-api.ts
│  ├─ session-service.ts
│  ├─ permissions.ts           catalogue and matrix (section 6.4), exported as const
│  ├─ settings.ts              SETTINGS_SCHEMA (zod) and defaults
│  └─ events.ts
├─ domain/
│  ├─ tenant.ts                status and KYB transitions (pure)
│  ├─ membership-rules.ts      last-owner rule, cashier needs branch
│  ├─ password-policy.ts       length, breached-list check
│  └─ totp.ts                  RFC 6238 verify with replay guard
├─ application/
│  ├─ signup.service.ts        signup, verify email
│  ├─ auth.service.ts          login, MFA challenge, lockout, reauth, reset
│  ├─ team.service.ts          invitations, role changes, removal
│  ├─ branch.service.ts
│  ├─ settings.service.ts
│  ├─ kyb.service.ts
│  ├─ staff.service.ts         OIDC callback mapping, staff CRUD
│  └─ access-resolver.ts       implements platform AccessResolver
├─ infrastructure/
│  ├─ repositories/*.repo.ts   Drizzle repos, one per table
│  ├─ session-store.ts         Postgres plus Redis cache
│  ├─ keycloak-oidc.ts         auth-code + PKCE client
│  └─ breached-passwords.ts    loads data/breached-100k.txt into a Set at boot
├─ http/
│  ├─ portal/                  auth.controller.ts, me.controller.ts, tenant.controller.ts, team.controller.ts, branch.controller.ts, settings.controller.ts
│  └─ admin/                   admin-auth.controller.ts, tenants.controller.ts, staff.controller.ts
├─ guards/
│  └─ session.guard.ts         reads cookie, loads session, builds Principal (used by portal-api and admin-api)
└─ db/schema.ts, db/migrations/
```

### Public interface

```ts
export interface TenancyApi {
  getTenant(tenantId: string): Promise<TenantView>;
  getSettings(tenantId: string): Promise<TenantSettings>;                  // cached 30 s
  getBranch(tenantId: string, branchId: string): Promise<BranchView>;
  listBranches(tenantId: string): Promise<BranchView[]>;
  getUser(userId: string): Promise<UserView>;                              // no secrets
  listMemberEmails(tenantId: string, roles: TenantRole[]): Promise<{ userId: string; email: string; locale: Locale }[]>;
  isTenantActive(tenantId: string): Promise<boolean>;                      // status === 'active'
  searchTenants(q: string, limit: number): Promise<TenantView[]>;          // admin search (doc 11): display or legal name contains q, case-insensitive
  countKybSubmitted(): Promise<number>;                                    // admin Tenants badge (doc 11)
  getStaff(staffId: string): Promise<StaffView>;                           // reviewer names (docs 07, 11)
  listStaffEmails(roles: PlatformRole[]): Promise<{ staffId: string; email: string; name: string }[]>;  // Review Lead notices (doc 07)
}

export interface SessionService {
  create(input: { principalKind: 'user' | 'staff'; subjectId: string; tenantId: string | null; env: EnvName | null;
                  kind: 'portal' | 'admin' | 'cashier_device'; deviceId?: string; ip: string; userAgent: string | null;
                  mfaVerified: boolean }): Promise<{ token: string; csrfToken: string; session: SessionView }>;
  resolve(token: string): Promise<SessionView | null>;   // checks idle and absolute expiry, revocation, membership
  touch(sessionId: string): Promise<void>;                // last_seen_at, throttled to once per 60 s
  setEnv(sessionId: string, env: EnvName): Promise<void>;
  setTenant(sessionId: string, tenantId: string): Promise<void>;
  markReauth(sessionId: string): Promise<void>;
  revoke(sessionId: string, reason: SessionRevokeReason): Promise<void>;
  revokeAllForUser(userId: string, reason: SessionRevokeReason, except?: string): Promise<void>;
  revokeAllForStaff(staffId: string, reason: SessionRevokeReason): Promise<void>;
}

export type TenantRole = 'owner' | 'admin' | 'developer' | 'cashier';
export type PlatformRole = 'reviewer' | 'review_lead' | 'admin';

export interface TenantView {
  id: string; legalName: string; displayName: string; tin: string | null; tradeLicenseNo: string | null;
  status: 'pending_kyb' | 'active' | 'suspended' | 'closed';
  kybStatus: 'not_submitted' | 'submitted' | 'approved' | 'rejected';
  defaultLocale: Locale; createdAt: Date;
}
export interface BranchView { id: string; tenantId: string; name: string; code: string; isDefault: boolean; archivedAt: Date | null }
export interface StaffView { id: string; name: string; email: string; roles: PlatformRole[]; status: 'active' | 'disabled' }
```

`AccessResolver` (the platform port) is implemented by `access-resolver.ts` and registered with `PlatformAuthzModule.forFeature(AccessResolverImpl)`.

## 4. Data model

All tables live in schema `tenancy`. Tenancy tables are tenant-level, not per environment, so they carry `tenant_id` but no `env`. Their RLS policy checks only `tenant_id`. `users`, `staff`, `sessions` and `email_tokens` are global and have no RLS; only this module's repositories touch them.

```sql
CREATE TABLE tenancy.tenants (
  id               uuid PRIMARY KEY,
  legal_name       text NOT NULL CHECK (length(legal_name) BETWEEN 2 AND 200),
  display_name     text NOT NULL CHECK (length(display_name) BETWEEN 2 AND 80),
  phone            text NOT NULL CHECK (phone ~ '^251[79][0-9]{8}$'),
  tin              text NULL CHECK (tin ~ '^[0-9]{10}$'),
  trade_license_no text NULL CHECK (length(trade_license_no) <= 64),
  status           text NOT NULL DEFAULT 'pending_kyb' CHECK (status IN ('pending_kyb','active','suspended','closed')),
  status_before_suspension text NULL CHECK (status_before_suspension IN ('pending_kyb','active')),
  status_reason    text NULL,
  kyb_status       text NOT NULL DEFAULT 'not_submitted' CHECK (kyb_status IN ('not_submitted','submitted','approved','rejected')),
  kyb_decided_by   uuid NULL,              -- tenancy.staff.id
  kyb_decided_at   timestamptz NULL,
  kyb_reject_reason text NULL,
  default_locale   text NOT NULL DEFAULT 'en' CHECK (default_locale IN ('en','am')),
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now(),
  closed_at        timestamptz NULL
);
CREATE INDEX tenants_status_idx ON tenancy.tenants (status, created_at DESC);
CREATE INDEX tenants_kyb_idx ON tenancy.tenants (kyb_status) WHERE kyb_status = 'submitted';
CREATE INDEX tenants_name_trgm ON tenancy.tenants USING gin (display_name gin_trgm_ops);  -- admin search

CREATE TABLE tenancy.kyb_documents (
  id          uuid PRIMARY KEY,
  tenant_id   uuid NOT NULL REFERENCES tenancy.tenants(id),
  kind        text NOT NULL CHECK (kind IN ('trade_license','tin_certificate','other')),
  object_key  text NOT NULL,             -- bucket "kyb"
  sha256      bytea NOT NULL,
  content_type text NOT NULL CHECK (content_type IN ('application/pdf','image/jpeg','image/png')),
  size_bytes  integer NOT NULL CHECK (size_bytes BETWEEN 1 AND 10485760),
  uploaded_by uuid NOT NULL,             -- users.id
  uploaded_at timestamptz NOT NULL DEFAULT now(),
  deleted_at  timestamptz NULL
);
CREATE INDEX kyb_documents_tenant_idx ON tenancy.kyb_documents (tenant_id) WHERE deleted_at IS NULL;

CREATE TABLE tenancy.branches (
  id          uuid PRIMARY KEY,
  tenant_id   uuid NOT NULL REFERENCES tenancy.tenants(id),
  name        text NOT NULL CHECK (length(name) BETWEEN 1 AND 80),
  code        text NOT NULL CHECK (code ~ '^[A-Z0-9-]{1,16}$'),
  is_default  boolean NOT NULL DEFAULT false,
  created_at  timestamptz NOT NULL DEFAULT now(),
  archived_at timestamptz NULL,
  UNIQUE (tenant_id, code)
);
CREATE UNIQUE INDEX branches_one_default ON tenancy.branches (tenant_id) WHERE is_default;

CREATE TABLE tenancy.users (
  id                 uuid PRIMARY KEY,
  email              citext NOT NULL UNIQUE,
  phone              text NULL CHECK (phone ~ '^251[79][0-9]{8}$'),
  full_name          text NOT NULL CHECK (length(full_name) BETWEEN 1 AND 120),
  password_hash      text NOT NULL,       -- argon2id encoded string
  email_verified_at  timestamptz NULL,
  locale             text NOT NULL DEFAULT 'en' CHECK (locale IN ('en','am')),
  calendar_pref      text NOT NULL DEFAULT 'auto' CHECK (calendar_pref IN ('auto','gregorian','ethiopian')),
  status             text NOT NULL DEFAULT 'active' CHECK (status IN ('active','disabled')),
  failed_auth_count  integer NOT NULL DEFAULT 0,
  locked_until       timestamptz NULL,
  last_login_at      timestamptz NULL,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE tenancy.user_mfa (
  user_id              uuid PRIMARY KEY REFERENCES tenancy.users(id),
  totp_secret_enc      text NOT NULL,     -- platform Crypto, key "platform-users"
  totp_confirmed_at    timestamptz NULL,
  totp_last_counter    bigint NULL,       -- replay guard
  recovery_code_hashes text[] NOT NULL DEFAULT '{}',  -- argon2id; removed when used
  updated_at           timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE tenancy.memberships (
  id          uuid PRIMARY KEY,
  tenant_id   uuid NOT NULL REFERENCES tenancy.tenants(id),
  user_id     uuid NOT NULL REFERENCES tenancy.users(id),
  role        text NOT NULL CHECK (role IN ('owner','admin','developer','cashier')),
  branch_id   uuid NULL REFERENCES tenancy.branches(id),
  status      text NOT NULL DEFAULT 'active' CHECK (status IN ('active','removed')),
  invited_by  uuid NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now(),
  removed_at  timestamptz NULL,
  CHECK ((role = 'cashier') = (branch_id IS NOT NULL))
);
CREATE UNIQUE INDEX memberships_active_uq ON tenancy.memberships (tenant_id, user_id) WHERE status = 'active';
CREATE INDEX memberships_user_idx ON tenancy.memberships (user_id) WHERE status = 'active';
CREATE INDEX memberships_branch_idx ON tenancy.memberships (branch_id) WHERE status = 'active' AND role = 'cashier';

CREATE TABLE tenancy.invitations (
  id          uuid PRIMARY KEY,
  tenant_id   uuid NOT NULL REFERENCES tenancy.tenants(id),
  email       citext NOT NULL,
  role        text NOT NULL CHECK (role IN ('owner','admin','developer','cashier')),
  branch_id   uuid NULL REFERENCES tenancy.branches(id),
  token_hash  bytea NOT NULL UNIQUE,       -- sha256 of 32-byte token
  invited_by  uuid NOT NULL,
  expires_at  timestamptz NOT NULL,
  accepted_at timestamptz NULL,
  revoked_at  timestamptz NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CHECK ((role = 'cashier') = (branch_id IS NOT NULL))
);
CREATE UNIQUE INDEX invitations_open_uq ON tenancy.invitations (tenant_id, email)
  WHERE accepted_at IS NULL AND revoked_at IS NULL;

CREATE TABLE tenancy.email_tokens (
  id          uuid PRIMARY KEY,
  user_id     uuid NOT NULL REFERENCES tenancy.users(id),
  purpose     text NOT NULL CHECK (purpose IN ('verify_email','reset_password')),
  code_hash   bytea NOT NULL,              -- HMAC(platform key "email-code", code)
  attempts    smallint NOT NULL DEFAULT 0,
  expires_at  timestamptz NOT NULL,
  used_at     timestamptz NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX email_tokens_lookup ON tenancy.email_tokens (user_id, purpose) WHERE used_at IS NULL;

CREATE TABLE tenancy.staff (
  id            uuid PRIMARY KEY,
  email         citext NOT NULL UNIQUE,
  full_name     text NOT NULL,
  keycloak_sub  text NULL UNIQUE,          -- linked on first sign-in
  roles         text[] NOT NULL CHECK (roles <@ ARRAY['reviewer','review_lead','admin']::text[] AND cardinality(roles) >= 1),
  status        text NOT NULL DEFAULT 'active' CHECK (status IN ('active','disabled')),
  created_by    uuid NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  last_login_at timestamptz NULL
);

CREATE TABLE tenancy.sessions (
  id              uuid PRIMARY KEY,
  token_hash      bytea NOT NULL UNIQUE,   -- sha256(token)
  kind            text NOT NULL CHECK (kind IN ('portal','admin','cashier_device')),
  principal_kind  text NOT NULL CHECK (principal_kind IN ('user','staff')),
  user_id         uuid NULL REFERENCES tenancy.users(id),
  staff_id        uuid NULL REFERENCES tenancy.staff(id),
  tenant_id       uuid NULL REFERENCES tenancy.tenants(id),
  env             text NULL CHECK (env IN ('live','test')),
  device_id       uuid NULL,               -- cashier.devices.id (doc 10)
  csrf_hash       bytea NOT NULL,
  mfa_verified_at timestamptz NULL,
  reauth_at       timestamptz NULL,
  ip              inet NOT NULL,
  user_agent      text NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  last_seen_at    timestamptz NOT NULL DEFAULT now(),
  absolute_expires_at timestamptz NOT NULL,
  revoked_at      timestamptz NULL,
  revoke_reason   text NULL,
  CHECK ((principal_kind = 'user') = (user_id IS NOT NULL)),
  CHECK ((principal_kind = 'staff') = (staff_id IS NOT NULL))
);
CREATE INDEX sessions_user_idx ON tenancy.sessions (user_id) WHERE revoked_at IS NULL;
CREATE INDEX sessions_staff_idx ON tenancy.sessions (staff_id) WHERE revoked_at IS NULL;

CREATE TABLE tenancy.tenant_settings (
  tenant_id  uuid NOT NULL REFERENCES tenancy.tenants(id),
  key        text NOT NULL,
  value      jsonb NOT NULL,
  updated_by uuid NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, key)
);

CREATE TABLE tenancy.role_permissions (     -- seeded by migration; read-only at runtime
  role       text NOT NULL,                 -- tenant or platform role name
  permission text NOT NULL,
  PRIMARY KEY (role, permission)
);

-- RLS on tenant-level tables
ALTER TABLE tenancy.kyb_documents ENABLE ROW LEVEL SECURITY; ALTER TABLE tenancy.kyb_documents FORCE ROW LEVEL SECURITY;
CREATE POLICY t_iso ON tenancy.kyb_documents USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
-- same policy on branches, memberships, invitations, tenant_settings.
-- tenants: SELECT/UPDATE allowed where id = app.tenant_id; pl_admin has the cross-tenant SELECT policy from the overview (5.4).
```

**Migrations:** `0001_tenancy_init.sql` (tables, indexes, RLS, `pg_trgm` and `citext` extensions), `0002_seed_role_permissions.sql` (section 6.4 matrix). Seeds are idempotent: `INSERT … ON CONFLICT DO NOTHING`, plus deletion of pairs no longer in the matrix.

**Object storage:** KYB files go to a new bucket `kyb`, with keys `t/{tenantId}/kyb/{documentId}`. This adds `kyb` to the overview's `Bucket` type.

## 5. API contracts

All portal routes are under `/portal/v1` and all admin routes under `/admin/v1`. Every mutating request needs the `X-CSRF-Token` header; the only exceptions are the `/auth/*` routes before a session exists.

### Portal: auth and session

| Method and path | Auth | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `POST /auth/signup` | Public, 5 per hour per IP | `{ legal_name, display_name, email, phone, password, locale }` | `202 { "status": "verification_sent" }` | `400 missing_field`, `invalid_field`, `weak_password`; `429` |
| `POST /auth/verify-email` | Public | `{ email, code }` | `200 Me`, sets cookies | `400 invalid_token` |
| `POST /auth/verify-email/resend` | Public | `{ email }` | `202` | `429` |
| `POST /auth/login` | Public, 20 per minute per IP | `{ email, password }` | `200 { "mfa_required": true, "challenge_id": "…" }` or `200 Me` (Cashier-only user) | `401 invalid_credentials`, `account_locked`; `403 email_not_verified` |
| `POST /auth/mfa/verify` | Public | `{ challenge_id, code }` or `{ challenge_id, recovery_code }` | `200 Me`, sets cookies | `401 mfa_invalid`, `account_locked`; `400 invalid_token` (challenge expired, 5 min) |
| `POST /auth/mfa/totp/enroll` | Session, enrollment pending | none | `200 { "secret_base32", "otpauth_url" }` | `409 invalid_state` (already enrolled) |
| `POST /auth/mfa/totp/confirm` | Session | `{ code }` | `200 { "recovery_codes": ["…" × 10] }` | `401 mfa_invalid` |
| `POST /auth/reauth` | Session | `{ password, code }` | `204` | `401 invalid_credentials`, `mfa_invalid` |
| `POST /auth/logout` | Session | none | `204` | none |
| `POST /auth/password/forgot` | Public, 5 per hour per IP | `{ email }` | `202` | `429` |
| `POST /auth/password/reset` | Public | `{ email, code, new_password }` | `204` | `400 invalid_token`, `weak_password` |
| `GET /me` | Session | none | `200 Me` | `401` |
| `PATCH /me` | Session | `{ full_name?, locale?, calendar_pref? }` (the portal's EN / አማ switch and calendar choice) | `200 Me` | `400 invalid_field` |
| `POST /session/env` | Session, `env.switch` | `{ env }` | `200 Me` | `403 forbidden` |
| `POST /session/tenant` | Session | `{ tenant_id }` | `200 Me` | `404 not_found` (no active membership) |

```json
// Me
{
  "user": { "id": "usr_01J…", "email": "yared@example.et", "full_name": "Yared Tesfaye", "locale": "en", "calendar_pref": "auto",
            "mfa_enrolled": true },
  "memberships": [
    { "id": "mem_01J…", "tenant": { "id": "ten_01J…", "display_name": "Habesha Mart PLC", "status": "active" },
      "role": "owner", "branch": null }
  ],
  "current": { "tenant_id": "ten_01J…", "env": "live", "role": "owner", "branch_id": null,
               "permissions": ["verifications.read", "…"] },
  "csrf_token": "b64url…"
}
```

`Me` carries no live-gate flag, because this module may not call doc 02. The portal's `GET /portal/v1/bootstrap` (doc 09) adds `live_enabled` by combining `TenancyApi.isTenantActive()` with `ReceivingAccountsApi.hasVerified()`. The gate itself is enforced in docs 03 and 05.

### Portal: tenant, KYB, team, branches, settings

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /tenant` | `tenant.read` | none | `200 TenantView` | none |
| `PATCH /tenant` | `tenant.manage` | `{ display_name?, phone?, default_locale?, tin?, trade_license_no? }` | `200 TenantView` | `400 invalid_field`; `409 invalid_state` (tin or licence after KYB approved) |
| `POST /tenant/kyb/documents` | `tenant.manage` | multipart `kind`, `file` | `201 KybDocument` | `413 file_too_large`, `415 unsupported_file_type` |
| `DELETE /tenant/kyb/documents/{id}` | `tenant.manage` | none | `204` | `409 invalid_state` (KYB submitted or approved) |
| `POST /tenant/kyb/submit` | `tenant.manage` | none | `200 TenantView` | `400 missing_field` (`trade_license`, `tin_certificate`, `tin`, `trade_license_no`); `409 invalid_state` |
| `GET /team/members` | `team.read` | `?cursor&limit` | `200 Page<Member>` | none |
| `PATCH /team/members/{membership_id}` | `team.manage` | `{ role?, branch_id? }` | `200 Member` | `409 last_owner`; `400 missing_field` (`branch_id` for Cashier) |
| `DELETE /team/members/{membership_id}` | `team.manage` | none | `204` | `409 last_owner` |
| `GET /team/invitations` | `team.read` | none | `200 { data: Invitation[] }` | none |
| `POST /team/invitations` | `team.manage` | `{ email, role, branch_id? }` | `201 Invitation` | `409 already_member`; `409 invitation_pending` |
| `POST /team/invitations/{id}/resend` | `team.manage` | none | `202` | `429` (once per 10 min) |
| `DELETE /team/invitations/{id}` | `team.manage` | none | `204` | none |
| `POST /invitations/accept` | Public | `{ token, full_name?, password? }` | `200 Me` (new user), or `200 { "sign_in_required": true }` | `400 invalid_token`, `weak_password` |
| `GET /branches` | `branches.read` | none | `200 { data: BranchView[] }` | none |
| `POST /branches` | `branches.manage` | `{ name, code }` | `201 BranchView` | `409 conflict_code` |
| `PATCH /branches/{id}` | `branches.manage` | `{ name? }` | `200 BranchView` | none |
| `POST /branches/{id}/archive` | `branches.manage` | none | `200 BranchView` | `409 branch_has_cashiers`, `invalid_state` (default branch) |
| `GET /settings` | `settings.read` | none | `200 TenantSettings` | none |
| `PATCH /settings` | `settings.manage`, plus re-auth within 5 min for `review.*` and `verification.*` keys | partial `TenantSettings` | `200 TenantSettings` | `400 invalid_field`; `401 reauth_required` |

### Admin

| Method and path | Permission | Request | Success | Errors |
| --- | --- | --- | --- | --- |
| `GET /auth/login` | Public, admin origin | `?return_to` | `302` to Keycloak (PKCE, state, nonce) | none |
| `GET /auth/callback` | Public | `?code&state` | `302` to app, sets `pl_admin_session` | `403 forbidden` (no active staff row) |
| `POST /auth/logout` | Staff session | none | `204` plus Keycloak end-session redirect URL | none |
| `GET /me` | Staff session | none | `200 StaffMe` (`permissions[]`) | none |
| `GET /tenants` | `platform.tenants.read` | `?status&kyb_status&q&cursor&limit` | `200 Page<AdminTenantRow>` | none |
| `GET /tenants/{id}` | `platform.tenants.read` | none | `200 AdminTenantDetail` (profile, KYB docs, branches, members count) | `404` |
| `GET /tenants/{id}/kyb/documents/{doc_id}/url` | `platform.tenants.manage` | none | `200 { "url", "expires_at" }` (60 s) | `404` |
| `POST /tenants/{id}/kyb/approve` | `platform.tenants.manage` | `{ note? }` | `200 AdminTenantDetail` | `409 invalid_state` |
| `POST /tenants/{id}/kyb/reject` | `platform.tenants.manage` | `{ reason }` (10–500 chars) | `200 AdminTenantDetail` | `409 invalid_state` |
| `POST /tenants/{id}/suspend` | `platform.tenants.manage` | `{ reason }` | `200` | `409 invalid_state` |
| `POST /tenants/{id}/reactivate` | `platform.tenants.manage` | none | `200` | `409 invalid_state` |
| `GET /staff` | `platform.staff.manage` | none | `200 { data: Staff[] }` | none |
| `POST /staff` | `platform.staff.manage` | `{ email, full_name, roles[] }` | `201 Staff` | `409 conflict_email` |
| `PATCH /staff/{id}` | `platform.staff.manage` | `{ roles[] }` | `200 Staff` | `409 last_admin` |
| `POST /staff/{id}/disable` | `platform.staff.manage` | none | `200 Staff` | `409 last_admin` |

### New error codes (added to `libs/contracts/src/errors.ts`)

| Code | HTTP | Meaning |
| --- | --- | --- |
| `invalid_credentials` | 401 | Email or password wrong |
| `account_locked` | 401 | Too many failures; `Retry-After` set |
| `mfa_invalid` | 401 | Wrong or replayed TOTP or recovery code |
| `reauth_required` | 401 | Sensitive action without re-authentication in the last 5 min |
| `email_not_verified` | 403 | Login before email verification |
| `mfa_enrollment_required` | 403 | TOTP not yet enrolled for a role that needs it |
| `tenant_suspended` | 403 | Tenant is suspended or closed |
| `weak_password` | 400 | Fails the policy in section 6.2 |
| `invalid_token` | 400 | Email code, invitation or challenge invalid, expired or exhausted |
| `already_member` | 409 | Invitee already active in the tenant |
| `invitation_pending` | 409 | An open invitation exists for this email |
| `last_owner` | 409 | Would leave the tenant without an active Owner |
| `last_admin` | 409 | Would leave the platform without an active Admin |
| `branch_has_cashiers` | 409 | Branch still has active Cashiers |
| `conflict_code` | 409 | Branch code already used in this tenant |
| `conflict_email` | 409 | Staff email exists |

### Events emitted (outbox)

| Event | Payload | Consumers |
| --- | --- | --- |
| `tenant.created` | `{ tenant_id }` | 12 (starts the Sandbox subscription), 05 (seeds the sandbox duplicate claim) |
| `tenant.status_changed` | `{ tenant_id, from, to, reason }` | 02 (accounts removed on `closed`), 03 (cache flush), 08 (endpoints disabled on `closed`), 10 (devices revoked on `suspended` or `closed`), 12 (subscription cancelled on `closed`) |
| `tenant.kyb_decided` | `{ tenant_id, decision, reason }` | none (the decision email is sent by this module) |
| `membership.created`, `membership.updated`, `membership.removed` | `{ tenant_id, membership_id, user_id, role, branch_id }` | 10 (revokes cashier devices on removal, a lost permission or a Cashier's branch change) |
| `tenant.settings_changed` | `{ tenant_id, keys[] }` | Settings cache invalidation (this module), 06 (retention recompute) |
| `branch.archived` | `{ tenant_id, branch_id }` | 10 |

This module consumes no events.

## 6. Business rules and state machine

### 6.1 Tenant status and KYB

| From | To | Trigger | Guard | Side effects |
| --- | --- | --- | --- | --- |
| (none) | `pending_kyb` | Signup verified | none | Create default branch "Main" (code `MAIN`), emit `tenant.created` |
| `pending_kyb` | `active` | Admin approves KYB | `kyb_status = submitted` | `kyb_status = approved`, email Owners, audit `tenant.kyb_approved` |
| `pending_kyb` or `active` | `suspended` | Admin suspends | reason given | Store `status_before_suspension`, revoke API-key caches (event), audit |
| `suspended` | previous status | Admin reactivates | none | Audit |
| any except `closed` | `closed` | Owner closes (`tenant.close`, re-auth) or Admin | none | Revoke all sessions and API clients (event), schedule purge after the retention period (doc 12 stops billing) |

| KYB from | To | Trigger |
| --- | --- | --- |
| `not_submitted` or `rejected` | `submitted` | Tenant submits with both document kinds plus TIN and licence number |
| `submitted` | `approved` | Admin approves |
| `submitted` | `rejected` | Admin rejects with a reason |

The live gate (docs 03 and 05) requires `status = active` **and** at least one verified receiving account. That extends R14 with KYB approval; see open question 01-Q1.

### 6.2 Credentials

| Rule | Value | Configured in |
| --- | --- | --- |
| Password length | 12–128 characters, any characters, no composition rules | `password-policy.ts` constants |
| Breached passwords | Rejected if found in the bundled list of the 100,000 most common passwords | `data/breached-100k.txt` |
| Hashing | argon2id, m = 19,456 KiB, t = 2, p = 1 | `AUTH_ARGON2_*` env |
| Lockout | 10 consecutive failed password or MFA attempts lock for 15 min | `AUTH_LOCKOUT_THRESHOLD`, `AUTH_LOCKOUT_MINUTES` |
| Per-IP login limit | 20 per minute | `AUTH_LOGIN_RPM_PER_IP` |
| Email codes | 6 digits, 15 min, 5 attempts, resend after 60 s, 5 per hour | Constants |
| TOTP | RFC 6238, SHA-1, 6 digits, 30 s step, ±1 step window, counter must exceed `totp_last_counter` | Constants |
| Recovery codes | 10 codes of 10 characters (Crockford base32), single use, regenerated on request with re-auth | Constants |
| MFA required | Owner, Admin, Developer. Cashier: not required, because cashiers protect the device with a PIN (doc 10) | `ROLES_REQUIRING_MFA` |
| Invitation TTL | 7 days | `TEAM_INVITATION_TTL_DAYS` |

### 6.3 Sessions

| Kind | Cookie | Idle timeout | Absolute | Re-auth window | Notes |
| --- | --- | --- | --- | --- | --- |
| `portal` | `pl_session` (HttpOnly, Secure, SameSite=Lax, Path=/) and `pl_csrf` (readable by JS) | 2 h | 12 h | 5 min | Env and tenant stored on the session |
| `admin` | `pl_admin_session` on the admin origin | 15 min | 8 h | 5 min (OIDC `max_age=300`) | Staff only |
| `cashier_device` | `pl_cashier` (HttpOnly, Secure, SameSite=Strict, Path=/api/portal/v1/cashier, so it reaches only the cashier API) | Server-enforced PIN lock after 5 min (doc 10) | 30 days | n/a | Created by doc 10 through `SessionService.create` |

- **Tokens:** 32 random bytes encoded base64url. The DB stores `sha256(token)`.
- **Cache:** `resolve()` checks the Redis cache `pl:sess:{hash}` (TTL 60 s), then the DB.
- **Revocation:** `revoke*` deletes the cache keys synchronously, so revocation takes effect on the next request.
- **Membership check:** `resolve()` also checks that the membership is still active and reads its current role and branch, cached in the same entry. Membership changes delete the affected users' session cache keys.
- **CSRF:** the `X-CSRF-Token` header must equal the session's CSRF token, compared by hash.

### 6.4 Permission catalogue and role matrix

| Permission | Owner | Admin | Developer | Cashier |
| --- | --- | --- | --- | --- |
| `tenant.read` | ✓ | ✓ | ✓ | |
| `tenant.manage` | ✓ | ✓ | | |
| `tenant.close` | ✓ | | | |
| `settings.read` | ✓ | ✓ | ✓ | |
| `settings.manage` | ✓ | ✓ | | |
| `team.read` | ✓ | ✓ | ✓ | |
| `team.manage` | ✓ | ✓ | | |
| `branches.read` | ✓ | ✓ | ✓ | |
| `branches.manage` | ✓ | ✓ | | |
| `receiving_accounts.read` | ✓ | ✓ | ✓ | |
| `receiving_accounts.manage` | ✓ | ✓ | | |
| `api_clients.read` | ✓ | ✓ | ✓ | |
| `api_clients.manage` | ✓ | | ✓ | |
| `webhooks.read` | ✓ | ✓ | ✓ | |
| `webhooks.manage` | ✓ | ✓ | ✓ | |
| `request_logs.read` | ✓ | ✓ | ✓ | |
| `sandbox.use` | ✓ | ✓ | ✓ | |
| `verifications.read` | ✓ | ✓ | ✓ | |
| `verifications.read.own` | | | | ✓ |
| `verifications.read.raw` | ✓ | | ✓ | |
| `verifications.create` | ✓ | ✓ | ✓ | ✓ |
| `verifications.cancel` | ✓ | ✓ | ✓ | |
| `verifications.export` | ✓ | ✓ | ✓ | |
| `review.read` | ✓ | ✓ | | |
| `review.decide` | ✓ | ✓ | | |
| `review.approve_second` | ✓ | ✓ | | |
| `billing.read` | ✓ | ✓ | | |
| `billing.manage` | ✓ | | | |
| `pii.reveal` | ✓ | ✓ | | |
| `env.switch` | ✓ | ✓ | ✓ | |

| Platform permission | Reviewer | Review Lead | Admin |
| --- | --- | --- | --- |
| `platform.dashboard.read` | ✓ | ✓ | ✓ |
| `platform.review.read` | ✓ | ✓ | ✓ |
| `platform.review.decide` | ✓ | ✓ | ✓ |
| `platform.review.approve_second` | | ✓ | ✓ |
| `platform.review.reassign` | | ✓ | ✓ |
| `platform.review.qa` | | ✓ | ✓ |
| `platform.verifications.read` | ✓ | ✓ | ✓ |
| `platform.verifications.retry_bank` | | ✓ | ✓ |
| `platform.tenants.read` | | ✓ | ✓ |
| `platform.tenants.manage` | | | ✓ |
| `platform.receiving_accounts.approve` | | | ✓ |
| `platform.banks.read` | ✓ | ✓ | ✓ |
| `platform.banks.manage` | | | ✓ |
| `platform.evidence.break_glass` | | | ✓ |
| `platform.staff.manage` | | | ✓ |
| `platform.audit.read` | | | ✓ |
| `platform.billing.read` | | | ✓ |
| `platform.billing.manage` | | | ✓ |
| `platform.settings.manage` | | | ✓ |
| `platform.pii.reveal` | | ✓ | ✓ |

- A staff member with several roles gets the union of their permissions.
- **Data scope:** `staff` → `platform`; a user whose current role is `cashier` → `own`, carrying their `branch_id`; every other user → `tenant`. API clients → `tenant`, decided in doc 03.
- The four-eyes rule (the second approver differs from the first) is a check in doc 07, not a permission.

### 6.5 Membership rules

- Exactly one active membership per `(tenant, user)`.
- Every tenant always has at least one active Owner. Any change that would leave none returns `409 last_owner`, checked in the same transaction with `SELECT … FOR UPDATE` on the tenant's Owner memberships.
- A Cashier membership must have exactly one non-archived `branch_id`.
- Removing a member revokes all of that user's sessions for that tenant, emits `membership.removed`, and leaves their past verifications untouched.

### 6.6 Tenant settings registry

| Key | Type | Default | Allowed | Read by |
| --- | --- | --- | --- | --- |
| `verification.date_window_hours` | int | 24 | 1–720 | 05 |
| `verification.force_review_above` | decimal string or null | null (off) | 1.00–999,999,999.99 | 05 |
| `verification.receipt_wait_hours` | int | 24 | 1–168 | 05 |
| `review.mode` | enum | `platform` | `platform`, `hybrid` | 07 |
| `review.hybrid_tenant_window_minutes` | int | 15 | 5–240 | 07 |
| `review.second_approval_above` | decimal string or null | `"10000.00"` | null or 1.00–999,999,999.99 | 07 |
| `review.max_age_hours` | int | 24 | 1–72 | 05, 07 |
| `locale.default` | enum | `en` | `en`, `am` | 06, 09, 10 |
| `upload_page.display_name` | string | tenant `display_name` | 2–80 characters | 06 |
| `upload_page.logo_object_key` | string or null | null | Set through the portal logo upload (PNG or SVG, at most 512 KB) | 06 |
| `notifications.alert_emails` | string[] | Owner emails | 0–5 valid emails | platform `Notify` |
| `retention.receipts_days` | int | 90 | 30–365 | 06 |

Settings are stored per key in `tenant_settings`. `getSettings()` merges stored values over the defaults and validates them with `SETTINGS_SCHEMA`; an invalid stored value logs an error and falls back to the default. The result is cached in Redis at `pl:tset:{tenantId}` for 30 s and invalidated on change.

### 6.7 Environments

- **Default env:** a new portal session starts in `live` if the tenant is `active`, otherwise in `test`.
- **Switching:** requires `env.switch`. Cashier sessions are always `live`.
- **Scope:** env is not part of any tenancy table. It only scopes data in other features through `Ctx`.

## 7. Dependencies

| Uses | Through |
| --- | --- |
| Platform | `Db`, `Ctx`, `Crypto` (TOTP secrets), `ObjectStore` (`kyb`), `Notify` (emails), `AuditLog`, `Outbox`, `Idempotency` (not used), `RateLimit`, `ConfigService`, `Clock` |
| Other features | None. `@proofline/receiving-accounts` is **not imported** (it depends on this module); the portal BFF (doc 09) combines the two where a screen needs both |
| Keycloak | OIDC discovery at `KEYCLOAK_ISSUER_URL` |

## 8. Implementation plan

Each task ends with passing tests and is mergeable on its own.

1. **T01.1 Schema and seeds.** Migrations 0001 and 0002, the Drizzle schema, and an RLS integration test proving a tenant-scoped query can't read another tenant's branches.
2. **T01.2 Permission catalogue.** `permissions.ts` (catalogue and matrix as const), a seed generated from it, and a unit test that seed and code agree.
3. **T01.3 Password policy and argon2id.** Plus the breached-list loader. Unit tests cover the length bounds, the breached list and a hash round-trip.
4. **T01.4 TOTP.** `totp.ts` with RFC 6238 test vectors and a replay-guard test.
5. **T01.5 SessionService.** Postgres plus Redis, idle and absolute expiry, revocation, CSRF. Integration tests run on `FakeClock`.
6. **T01.6 SessionGuard and AccessResolver.** Registered with platform authz; the boot-time check fails any route with no permission decorator; data-scope tests.
7. **T01.7 Signup and email verification.** Endpoints, no-enumeration behaviour, tenant creation in one transaction with branch, membership and outbox event.
8. **T01.8 Login, MFA challenge, enrollment, recovery codes, lockout, re-auth, password reset.**
9. **T01.9 Me, env switch, tenant switch.**
10. **T01.10 Team.** Invitations (create, resend, revoke, accept), role change, removal, last-owner rule with a concurrency test.
11. **T01.11 Branches.**
12. **T01.12 Settings registry.** Schema, endpoints, cache and invalidation event.
13. **T01.13 KYB.** Document upload to the `kyb` bucket, submit, admin approve and reject, status transitions.
14. **T01.14 Admin OIDC sign-in with Keycloak.** PKCE, state, nonce, ID-token validation, linking the staff row on first sign-in, staff CRUD, last-admin rule.
15. **T01.15 Suspend, reactivate and close.** Events and audit entries.
16. **T01.16 Bootstrap CLI.** `pnpm --filter @proofline/worker run tenancy:create-admin --email --name` creates the first platform Admin (fails if one exists).
17. **T01.17 Portal and admin screens.** Sign-in, sign-up, MFA, Team, Settings, and admin Tenants and Staff, with EN/AM strings for the portal screens.

## 9. Edge cases and failure modes

| Case | Behaviour |
| --- | --- |
| Two signups with the same email at the same moment | Unique `users.email` lets one insert win. The other request takes the "existing account" email branch and returns the same `202`. |
| Signup transaction fails after the email is sent | Not possible: the email is sent from an outbox handler after commit, never inside the transaction. |
| Email provider down | `Notify` retries 5 times over 30 min. The user can press resend after 60 s. |
| Two admins demote the last two owners concurrently | Both transactions lock the Owner rows `FOR UPDATE`. The second sees one remaining Owner and gets `409 last_owner`. |
| Removed user still has an open portal tab | The next request finds the session's membership removed. The response is `401 unauthenticated` and the frontend redirects to sign-in. |
| A Cashier's branch is archived | Blocked: archiving requires no active Cashiers. Moving a Cashier to another branch emits `membership.updated`, and doc 10 revokes their devices. |
| TOTP code reused within its 30 s window | Rejected by `totp_last_counter` (`401 mfa_invalid`), counted toward lockout. |
| Clock skew on the user's phone of more than 30 s | Fails. The error message suggests syncing the phone's time; recovery codes remain available. |
| Keycloak down | Admin sign-in returns `503 service_unavailable`. Existing admin sessions keep working until their own expiry. |
| Redis down | Sessions are read from Postgres directly (cache bypass, warning logged). Rate limits fail closed for login (`503`) and open for other routes. |
| Tenant suspended while a cashier is mid-verification | The verification already accepted continues. The next create returns `403 tenant_suspended`. |
| Invitation accepted by a signed-in different user | The membership is created for the email on the invitation only. A signed-in user with a different email gets `400 invalid_token`. |
| Settings change racing a verification | Verifications read settings when created. A change applies to verifications created afterwards. |

## 10. Security and privacy

| Endpoint group | Authentication | Authorization |
| --- | --- | --- |
| `/portal/v1/auth/*` (public) | None; rate limited per IP and per email | n/a |
| `/portal/v1/*` (others) | `pl_session` plus CSRF | `@RequirePermission` per route (section 5), tenant from the session, RLS |
| `/admin/v1/*` | `pl_admin_session`, network allowlist at the ingress | `platform.*` permissions |

- No account enumeration on signup, forgot-password or login: the same responses and the same timing. A dummy argon2 verify runs when the user doesn't exist.
- TOTP secrets are encrypted with OpenBao key `platform-users`. Password and recovery-code hashes never leave the module, and `UserView` has no secret fields.
- KYB documents are class P data, stored encrypted. Only the tenant itself and platform Admins can fetch signed URLs, which last 60 s, and every fetch is audited (`tenant.kyb_document_viewed`).
- **Audited actions:** login success and failure, lockout, MFA enroll and reset, password reset, session revoke, invitation create, accept and revoke, role change, member removal, branch archive, settings change, KYB submit and decide, suspend, reactivate, close, staff changes.
- **Input validation:** zod schemas in `libs/contracts/portal/tenancy.ts`. Emails are lowercased and checked against RFC 5322 (simplified), at most 254 characters. Phones must be E.164 without `+` and start with `2519` or `2517`.

## 11. Performance

- **Load:** sign-ins are low volume, but `SessionService.resolve()` runs on every portal and admin request. Expected peak: 200 requests per second across portal-api.
- **Targets:** `resolve()` p95 under 3 ms on a cache hit and under 15 ms on a miss. Login p95 under 400 ms, mostly argon2.
- **Indexes:** as in section 4. `memberships_user_idx` serves `Me`, and `sessions` lookups go by the unique `token_hash`.
- **Settings:** `getSettings()` is a Redis hit in the steady state, with a 30 s TTL.

## 12. Testing

| Level | Tests |
| --- | --- |
| Unit | Password policy; TOTP RFC 6238 vectors and replay; tenant and KYB transition table; membership rules; settings schema bounds; permission matrix equals seed |
| Integration (Testcontainers) | Signup → verify → enroll → login → reauth; lockout after 10 failures and unlock after 15 min (`FakeClock`); RLS isolation for every tenant-level table; last-owner concurrency (two parallel transactions); session revocation visible on the next request; settings cache invalidation |
| End to end (Playwright) | Sign up, verify, enroll TOTP (codes computed from the shown secret), invite a Cashier, accept, Cashier sees only Verify and My checks (matches the Portal · Verify mockup), Owner switches Live/Test |
| Amharic | Sign-in, Team and Settings screens rendered in `am`; no missing keys; Ethiopian dates on the Team "added on" column |

## 13. Observability

| Metric | Type | Labels |
| --- | --- | --- |
| `pl_auth_logins_total` | counter | `surface` (portal, admin), `outcome` (success, invalid, locked, mfa_invalid) |
| `pl_auth_lockouts_total` | counter | none |
| `pl_session_resolve_duration_seconds` | histogram | `cache` (hit, miss) |
| `pl_tenancy_kyb_pending` | gauge | none |

| Alert | Condition | Severity |
| --- | --- | --- |
| Login failure spike | More than 50 `invalid` per minute for 5 min | Warning (possible credential stuffing) |
| KYB backlog | `pl_tenancy_kyb_pending` above 20 for 24 h | Info |
| Keycloak unreachable | Admin login 503s for 2 min | Critical |

Logs never contain passwords, codes, tokens or TOTP secrets (they are on the pino redaction list).

## 14. Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `AUTH_ARGON2_MEMORY_KIB` | 19456 | argon2id memory |
| `AUTH_ARGON2_ITERATIONS` | 2 | argon2id time cost |
| `AUTH_LOCKOUT_THRESHOLD` | 10 | Failures before lock |
| `AUTH_LOCKOUT_MINUTES` | 15 | Lock length |
| `AUTH_LOGIN_RPM_PER_IP` | 20 | Login attempts per minute per IP |
| `SESSION_PORTAL_IDLE_MINUTES` | 120 | Portal idle timeout |
| `SESSION_PORTAL_ABSOLUTE_HOURS` | 12 | Portal absolute timeout |
| `SESSION_ADMIN_IDLE_MINUTES` | 15 | Admin idle timeout |
| `SESSION_ADMIN_ABSOLUTE_HOURS` | 8 | Admin absolute timeout |
| `TEAM_INVITATION_TTL_DAYS` | 7 | Invitation lifetime |
| `KEYCLOAK_ISSUER_URL` | (required) | e.g. `https://sso.<domain>/realms/proofline-staff` |
| `KEYCLOAK_CLIENT_ID` | `admin-web` | OIDC client |
| `KEYCLOAK_CLIENT_SECRET` | (required) | Confidential client secret |

Per-tenant settings are listed in section 6.6. There are no feature flags.

## 15. Open questions

| # | Question | Default until answered |
| --- | --- | --- |
| 01-Q1 | Does going live also require KYB approval, on top of a verified receiving account (R14)? | Yes: live requires `status = active` |
| 01-Q2 | Which KYB documents are mandatory? | Trade licence plus TIN certificate |
| 01-Q3 | Should Cashiers be required to use TOTP when they sign in to the full portal (not the PWA)? | No; their permissions are limited to Verify and their own checks |
| 01-Q4 | Who are the first platform staff, and does an existing identity provider need to federate into Keycloak? | Local Keycloak users created by the bootstrap Admin |
