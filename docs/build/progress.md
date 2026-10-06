# Build progress

Claude Code updates this file at the start and end of every step (see `.claude/skills/build-step`). People update the Phase 0 table.

**Status values:** `todo`, `in progress`, `blocked` (say why), `done`.

## Steps

| Step | What | Needs | Status | Started | Finished | Branch | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| S00 | Scaffold, CI, dev stack | none | todo | | | | |
| S01 | Platform kernel A | S00 | todo | | | | |
| S02 | Platform kernel B | S01 | todo | | | | |
| S03 | 01 tenancy, auth, RBAC | S02 | todo | | | | |
| S04 | 12a plans, metering, quota | S03 | todo | | | | |
| S05 | 02 receiving accounts | S03 | todo | | | | |
| S06 | 03 API clients | S04, S05 | todo | | | | |
| S07 | 04 bank connectors (sandbox) | S02 | todo | | | | |
| S08 | 05 verification engine | S06, S07 | todo | | | | |
| S09 | 06 receipt intake, upload page | S08 | todo | | | | |
| S10 | 08 webhooks | S08 | todo | | | | |
| S11 | 07 manual review | S09 | todo | | | | |
| S12 | 09 portal core and sign-in screens | S08–S11 | todo | | | | |
| S13 | Portal owner pages | S12 | todo | | | | |
| S14 | 10 cashier app | S08, S09, S11, S12 | todo | | | | |
| S15 | 11 admin panel and admin screens | S10, S11 | todo | | | | |
| S16 | Infra, observability, pilot readiness | S13–S15 | todo | | | | |
| S17 | 12b invoices and payments | S16, pilot | todo | | | | |

## Track B: real bank adapters

| Step | Bank | Fixtures captured | Status | Notes |
| --- | --- | --- | --- | --- |
| B1 | TELEBIRR | no | todo | |
| B2 | CBE | no | todo | |
| B3 | BOA | no | todo | |
| B4 | DASHEN | no | todo | |
| B5 | CBEBIRR | no | todo | |
| B6 | MPESA | no | todo | |
| B7 | AWASH | no | todo | |
| T05.12 | Live path smoke test from in-country | after B1–B7 | todo | |

## Phase 0 (people)

| Item | Owner | Status | Answer or link |
| --- | --- | --- | --- |
| Q1 Legal opinion on bank receipt pages; bank outreach | | open | |
| Q2 Accounts at 7 providers; test payments | | open | |
| Q3 Hosting and two ISPs; webhook egress IPs (08-Q2) | | open | |
| Q4 Vision provider decision | | open | |
| Q5 Plans and prices | | open | Seeds in doc 12 §6.2 until decided |
| Q10 Amharic reviewer | | open | |
| Q11 Product name and domains | | open | |
| Q13 Retention periods | | open | Doc defaults apply |
| Q14 National Bank of Ethiopia licensing | | open | |

## Step log

New entries go on top. Use this format:

```text
### SNN · <title> · <status> · <YYYY-MM-DD>
- Built: …
- Tests: <counts>, pnpm verify green
- Defaults applied / questions: Qn in questions.md
- Follow-ups: …
```
