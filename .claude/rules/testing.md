---
paths:
  - "**/test/**"
  - "**/*.test.ts"
  - "**/*.spec.ts"
  - "e2e/**"
---

# Testing rules (overview 6.7)

**What to test**

- Tests come from the doc. Each user story's acceptance criteria and each row of the doc's Testing section becomes at least one test.
- Name tests after the doc reference, for example `describe('US-05.3 duplicate claim', …)` or `it('6.2 row: processing → receipt_required on 3rd transient', …)`.
- `domain/` code gets table-driven unit tests covering every row of the doc's state, decision and precedence tables.

**Unit tests** (Vitest)

- No I/O, no timers, no network.
- Use `FakeClock` from `@proofline/platform/testing` for anything time-based: SLAs, windows, expiries, retries.

**Integration tests** (Vitest with Testcontainers: Postgres 18, Redis 7, MinIO)

- Each package migrates only its own schema plus `platform`, `audit` and `tenancy`.
- Other features are replaced by the fakes their packages publish from the `testing` export. Never reach into another feature's tables.
- Every tenant table gets an RLS test: another tenant and the other env read nothing and can't write.
- Concurrency cases named in the docs are tested with real parallelism, for example the claim race, Take next with 10 workers, gapless invoice numbers and the webhook semaphore.

**No network**

- No test calls a bank, a vision provider or the internet.
- Bank parsers are tested against fixtures in `libs/features/bank-connectors/test/fixtures/{bank}/{parserVersion}/`, which must be anonymised (doc 04).
- HTTP clients are tested against local servers: a local HTTPS server with a test CA for webhooks and egress.

**End to end** (Playwright, `e2e/`)

- Runs against Docker Compose with the sandbox connector, using the magic references from doc 04 section 6.8 (`SANDBOX_REFERENCES`).
- Includes Amharic screenshot runs (`e2e/i18n/`) and axe checks with no serious or critical violations.

**Gates**

- Coverage: 90% lines in `domain/`, 80% per package. The CI contract test fails on any breaking change to `/v1` in the OpenAPI diff.
- A step is not done while `pnpm verify` is red, while a test is skipped without a linked entry in `docs/build/questions.md`, or while a required test from the doc is missing.
