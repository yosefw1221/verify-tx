> **Reference only.** Exported from the Claude Docs version of 4 October 2026 (https://claude.ai/code/artifact/b1798dd4-d875-4b2e-8dbd-8715e423c147). It predates the accepted decisions. Where it differs from `docs/00-overview.md` or `docs/features/`, those win.

# UI spec

The product ships four surfaces from one design system: an admin panel for our staff, a portal for merchants, a hosted page where a merchant's customer uploads a receipt, and a phone screen for counter staff. All four are bilingual English and Amharic from day one, and every screen uses the same status words the API returns.

## Design direction

Calm and trustworthy: dense where staff work all day, very simple where a customer acts once. **Proofline** is a working name used in the mockups until you choose the real one.

| Token | Value | Used for |
| --- | --- | --- |
| `brand-600` | #0F6E6A (highland teal) | Primary buttons, links, focus rings |
| `brand-700` | #0B5753 | Hover and pressed states |
| `brand-50` | #E8F3F2 | Selected rows, info panels |
| `accent-500` | #C8862A (coffee amber) | Count badges and highlights, sparingly |
| `ink-900` | #14202B | Body text |
| `ink-600` | #536170 | Secondary text, labels |
| `line-200` | #E2E6E3 | Borders, dividers |
| `canvas` | #F6F7F5 | Page background |
| `surface` | #FFFFFF | Cards, tables, drawers |

Status colours are defined with the status vocabulary below. No bank's brand colour is used anywhere, so no provider looks favoured, and bank names appear as text chips until each bank gives written permission to use its logo.

- **Type:** IBM Plex Sans for Latin text, Noto Sans Ethiopic for Amharic, IBM Plex Mono for references and IDs. Amounts use tabular figures.
- **Sizes:** 14 px base in the admin panel and portal; 16 px on customer-facing pages; Amharic one pixel larger with 1.6 line height.
- **Grid and shape:** 4 px spacing grid; 8 px radius on cards, 6 px on inputs, fully rounded badges.
- **Density:** 40 px table rows in admin, 48 px in the portal, 56 px touch targets on phones.
- **Icons:** Lucide at 20 px, always next to a word on status and actions.
- **Dark mode:** admin panel only, for reviewers on night shifts.
- **Build:** Tailwind tokens matching this table and Radix-based components in the Next.js app.

## Language and formats

Every surface has an EN | አማ switch in the top bar, defaulting to the browser language. The upload page follows the merchant's default and a `lang` parameter in the link.

| Item | English UI | Amharic UI |
| --- | --- | --- |
| Money | ETB 1,500.00 | 1,500.00 ብር |
| Date | 4 Oct 2026, 10:12 | Ethiopian calendar by default, Gregorian on hover; switchable per user |
| Time | 24-hour, East Africa Time | Same; standard clock, not local hour counting |
| Phone | +251 9•• ••• 234 (masked) | Same |
| Reference | `CHQ0FJ403O` in monospace with a copy button | Same, never translated |
| Payer name | As the bank returns it, no automatic re-casing | Same |

- **Catalogues, not strings in code.** All text lives in next-intl message files with ICU plurals; sentences are never built by joining fragments.
- **Room to grow.** Layouts allow 30% wider text; buttons never truncate a label.
- **Translation quality.** Amharic strings in this spec are drafts and need review by a native-speaking translator before launch; machine translation is for first drafts only.
- **Staff view of time.** The admin panel shows EAT with UTC on hover; merchants never see UTC.

## Navigation map

Staff and merchants work in full applications with a sidebar; customers and cashiers get single-task screens built for a phone.

> Diagram in the live doc: navigation map · 4 surfaces and their menus

The admin review workspace and the portal review queue share one component, so a fix to one reaches both.

## Shared vocabulary and components

The same eight statuses and the same reason labels appear on every surface, always as icon plus word, never colour alone.

| Status | Badge colour | Icon | English | Amharic |
| --- | --- | --- | --- | --- |
| `processing` | Slate #5B6B7A | spinner | Processing | በሂደት ላይ |
| `receipt_required` | Blue #2F6FDB | upload | Receipt needed | ደረሰኝ ያስፈልጋል |
| `in_review` | Amber #B7791F | eye | In review | በግምገማ ላይ |
| `verified` | Green #1F8A4C | check-circle | Verified | ተረጋግጧል |
| `rejected` | Red #C2362F | x-circle | Rejected | ውድቅ ተደርጓል |
| `reversed` | Purple #8A3FB8 | rotate-ccw | Reversed | ተቀልብሷል |
| `expired` | Grey #8A949E | clock | Expired | ጊዜው አልፏል |
| `cancelled` | Grey outline | slash | Cancelled | ተሰርዟል |

A verified badge always carries an assurance chip: **Bank confirmed** (በባንክ የተረጋገጠ) or **Manual review** (በሰው የተገመገመ).

| Reason code | English label | Amharic label |
| --- | --- | --- |
| `not_found` | Bank has no such transaction | ባንኩ እንዲህ ያለ ግብይት የለውም |
| `receiver_mismatch` | Paid to a different account | ለሌላ ሂሳብ ተከፍሏል |
| `amount_mismatch` | Amount is less than expected | መጠኑ ከሚጠበቀው ያነሰ ነው |
| `duplicate` | Receipt already used | ደረሰኙ ቀድሞ ጥቅም ላይ ውሏል |
| `stale` | Payment is too old | ክፍያው በጣም የቆየ ነው |
| `not_completed` | Transfer not completed | ዝውውሩ አልተጠናቀቀም |
| `invalid_reference` | Reference format is wrong | የማጣቀሻ ቁጥሩ ቅርጸት ስህተት ነው |
| `unreadable_receipt` | Receipt can't be read | ደረሰኙ አይነበብም |
| `suspected_forgery` | Receipt looks altered | ደረሰኙ የተቀየረ ይመስላል |
| `manual_reject` | Rejected by reviewer | በገምጋሚ ውድቅ ተደርጓል |
| `bank_contradicted` | Bank contradicted the approval | ባንኩ ማረጋገጫውን ተቃርኗል |
| `bank_unavailable` | Bank not responding | ባንኩ ምላሽ እየሰጠ አይደለም |
| `reference_insufficient` | Receipt or share link needed | ደረሰኝ ወይም የማጋሪያ ሊንክ ያስፈልጋል |
| `bank_format_changed` | Bank page changed, checking by hand | የባንኩ ገጽ ተቀይሯል፤ በእጅ እየተመረመረ ነው |

### Components used everywhere

- **StatusBadge** and **AssuranceChip** as above; **ReasonText** renders the label, with the code on hover for developers. In dense tables the status icon shrinks to a coloured dot; the word always stays.
- **Money** right-aligned with tabular figures; **BankChip** a text chip with the bank code.
- **CopyRef** monospace value with a copy button, for references, verification IDs and key IDs.
- **MaskedValue** shows `5678`; revealing more needs the permission and a re-authentication, and is written to the audit log.
- **DataTable** with sortable columns, filter chips, saved views, column picker, CSV and Excel export, server-side paging.
- **DetailDrawer** slides in from the right so the list keeps its scroll position.
- **Timeline** of events for one verification: created, each bank attempt, receipt upload, review decision, webhooks.
- **SlaTimer** counts down; turns amber at 80% of the window and red at breach.
- **States:** skeleton rows while loading; an empty state with one clear action; an error banner that shows the request ID.

## Admin panel

A left sidebar with ten sections, a top bar with global search (verification ID, reference, order ID, tenant) and the signed-in staff member. The review queue and workspace are where most staff hours go, so they get the most care.

| Screen | Who | Purpose | Key content | Main actions |
| --- | --- | --- | --- | --- |
| Dashboard | All staff | Today's health at a glance | Verifications today, share bank-confirmed, review backlog, SLA breaches; per-bank health strip; hourly volume | Jump to queue or bank |
| Review queue | Reviewer, Review Lead | Pick the next task | Rows: SLA timer, tenant, bank, amount, reason, risk flags, age, assignee; filters for mode, bank, amount band, "mine" | **Take next**, claim, reassign (Lead) |
| Review workspace | Reviewer, Review Lead | Decide one task | See layout below | Approve, reject, request image, escalate |
| Verifications | Ops, Support (masked) | Find any verification | Search and filters; detail drawer with timeline | Retry bank (Ops), view raw response (Engineer, break-glass) |
| Tenants | Ops Admin | Manage merchants | Profile and KYB documents, plan and limits, receiving accounts, review policy, usage | Approve KYB, approve account, change plan, suspend |
| Banks & egress | Ops, Engineer | Operate adapters | Per bank: status, success rate 1 h and 24 h, p95 latency, breaker, last canary, failure classes; egress node table | Open or close breaker, disable adapter, pin node |
| Staff & roles | Super Admin | Access control | Staff list, roles, permission matrix, MFA status | Invite, assign role, revoke |
| Audit log | Auditor, Super Admin | Who did what | Filterable event stream with a "chain verified" badge | Export |
| Billing | Finance | Invoices and payments | Invoices, payment confirmations, plan catalogue | Mark paid, issue credit |
| Settings | Super Admin | Platform defaults | SLA targets, thresholds, retention, extraction provider, message templates | Edit |

### Review workspace

One screen, three columns, no scrolling for a normal task on a 1440 px display.

- **Header:** back to queue, task ID, tenant, bank chip, expected amount, SLA timer, lock owner ("You, 8 min left"), previous and next.
- **Left, receipt (45%):** zoom, rotate, contrast, original or enhanced, page thumbnails for PDFs; a box over the decoded QR; a warning bar when the image hash matches an earlier upload.
- **Middle, evidence (30%):** a comparison table with rows for reference, bank, amount, receiver name, receiver account, payer and date, and columns *From receipt* (with a confidence dot), *Merchant expected*, *Bank record* (when the bank is down, a warning banner above the table takes its place), and a match mark. Below it, risk signals and history of the same payer, reference or image.
- **Right, decision (25%):** Approve and Reject buttons, reason picker, note (required on reject and above the high-value threshold), Request clearer image, Escalate, Retry bank with the last attempt time. A banner states when a second approval is needed.
- **Keyboard:** A approve, R reject, I request image, E escalate, J and K next and previous.

## Tenant portal

The portal serves two kinds of merchant: shops that never integrate and use the Verify screen, and developers who live in the Developers section. A live/test switch sits in the top bar; test mode shows an orange banner saying no bank is being called.

The portal adds one role to the architecture's list: **Cashier**, who can run checks from the Verify screen and see their own history, nothing else.

| Screen | Who | Purpose | Key content | Main actions |
| --- | --- | --- | --- | --- |
| Overview | Owner, Admin, Viewer | How payments are going | Verified today, ETB verified, rejected with top reasons, in review; bank status pills; recent checks; a setup checklist until live | Open log, finish setup |
| Verify | Cashier and above | Check a payment with no integration | Bank auto-detect or picker, reference or link field, receipt upload or camera, expected amount, order note; result card beside the form, below it on narrow screens | Verify |
| Verifications | All roles | The payment log | Time, status, amount, bank, reference, payer, order ID, assurance; filters, saved views | Export, cancel, resend webhook |
| Verification detail | All roles (raw JSON: Developer) | One payment's full story | Status and assurance, normalised fields, checks with pass or fail, timeline, receipt thumbnail, raw response | Copy, resend webhook, new upload link |
| Review queue | Tenant Reviewer (tenant or hybrid mode) | Decide held payments | The admin workspace, simplified, with a reminder to check the bank statement | Approve, reject, request image |
| Receiving accounts | Admin | Where you get paid | Per bank: masked number, holder name, status, how it was proven | Add (bank, number, holder, proof), remove |
| API clients | Developer | Keys | Name, key ID, mode, scopes, IP allowlist, last used, creator | Create (secret shown once), rotate, revoke |
| Webhooks | Developer | Event delivery | Endpoints and their events, secret (re-auth to reveal), delivery log with status codes | Add, send test, replay |
| Request logs | Developer | Debugging | Method, path, status, latency, request ID; masked JSON body | Filter, copy request ID |
| Sandbox | Developer | Testing | Magic references with copy buttons | Send test webhook |
| Team | Owner, Admin | People | Users, roles, MFA status, last active | Invite, change role, remove |
| Billing & usage | Owner | Plan and spend | Usage against plan, daily usage, invoices | Upgrade, pay by telebirr or transfer |
| Settings | Owner, Admin | Policies | Date window, amount rule and tolerance, review mode, high-value threshold, retention, upload-page branding, alert emails | Save |

**Detail first, then act.** Clicking a row opens the detail drawer over the log, so a support agent can move down the list without losing filters. Every rejected row shows its reason label inline, so the merchant rarely needs to open it.

## Hosted receipt upload page

A customer reaches this page from a 30-minute signed link the merchant sends, usually on a phone over mobile data, with no account. It shows the merchant's logo and name, a small "secured by Proofline" line, and the language switch.

1. **Context.** Merchant, expected amount and order reference, with one sentence on why: "Your bank isn't responding right now. Upload your receipt and we'll confirm your payment." For `reference_insufficient` it asks for the receipt or the share link from the SMS instead.
2. **Choose.** Two large buttons, *Take a photo* and *Choose file* (image or PDF), plus a *Paste share link* field. A tip says to keep the QR code in view.
3. **Preview.** The photo with *Use this* and *Retake*. The phone shrinks the image to 1600 px before upload.
4. **Checking.** Two visible steps, "Reading receipt" then "Contacting bank", which usually finish in a few seconds.
5. **Result.** One of the states below, each with a single next action.

| State | Message (English) | Message (Amharic) | Next action |
| --- | --- | --- | --- |
| Verified | Payment confirmed | ክፍያዎ ተረጋግጧል | Return to merchant |
| In review | We're checking your receipt; the merchant will update you | ደረሰኝዎን እየመረመርን ነው፤ ነጋዴው ያሳውቅዎታል | Return to merchant |
| Rejected | We couldn't confirm this payment, plus the reason label | ይህን ክፍያ ማረጋገጥ አልቻልንም | Try another receipt (up to 3 tries) |
| Link expired | This link has expired | ይህ ሊንክ ጊዜው አልፏል | Ask the merchant for a new link |

The page carries a two-line privacy notice with a link to the full policy. If the legal decision is to use a vision provider abroad, a consent checkbox appears above the upload buttons. No third-party analytics run on this page.

## Mobile verify screen

A cashier must know in one glance, with a customer waiting, whether to hand over the goods. It ships as an installable web app (PWA) built from the portal's Verify screen, so there is no app store release at first.

- **Sign-in:** a Cashier account on first use, then a 6-digit PIN to unlock on that device. Sessions bind to the device and can be revoked from Team.
- **Main screen:** a camera viewfinder for the receipt QR fills the top half. Below it: a reference field that auto-detects the bank, an optional expected amount on a numeric keypad, and a full-width **Verify** button.
- **Result screen:** fills the whole display in the status colour, with sound and vibration that differ by outcome.

| Outcome | Headline | Detail shown | Button |
| --- | --- | --- | --- |
| Verified | VERIFIED, then the amount in large type | Payer, bank, time, assurance chip | Done |
| Rejected | DO NOT RELEASE | Reason label, e.g. "Receipt already used, first used 3 Oct 14:20" | Done |
| In review | WAIT, under review | Expected decision time from the SLA | Check again |
| Receipt needed | SCAN THE RECEIPT | "Bank not responding" or "Receipt or share link needed" | Open camera |

- **History tab:** today's checks by this cashier, with counts and the ETB total verified.
- **Offline:** "No connection, cannot verify". The app never shows a success it did not receive from the server.

## Accessibility, responsiveness and performance

WCAG 2.2 AA everywhere, and budgets set for cheap Android phones on mobile data, because that is how most customers and cashiers will reach us.

| Surface | Designed for | Smallest supported | Budget |
| --- | --- | --- | --- |
| Admin panel | 1440 px desktop | 1024 px | Lists paged on the server; workspace usable by keyboard alone |
| Tenant portal | 1280 px desktop | 360 px phone | Tables virtualised beyond 100 rows |
| Upload page | 360 px phone | 320 px | Under 150 KB of JS and CSS; largest content paint under 2.5 s on 3G |
| Mobile verify | 360 px phone | 320 px | Result shown within 1 s of the server answer |

- Text contrast at least 4.5:1; visible focus rings; every action reachable by keyboard.
- Status badges read their word to screen readers; icons alone never carry meaning.
- Secrets and unmasked personal data need a fresh re-authentication; warnings appear before a session times out.
- Every screen has its loading, empty and error state designed, not left to defaults.

## Mockups

Thirteen screens are drawn on the [Proofline UI mockups canvas](https://claude.ai/artifact/EYXDJVaKXKekjhuW3SPZdY). They link to each other, so pressing Play on any screen lets you click through a flow. All names, amounts and references are sample data, and they line up across screens: the same telebirr payment appears in the review queue, the review workspace, the merchant's log and the customer's upload page.

| Screen | Surface | What it shows |
| --- | --- | --- |
| Dashboard | Admin | Day totals, per-bank health with telebirr degraded and M-Pesa down, hourly volume, outcomes, backlog by time left, alerts |
| Review queue | Admin | SLA-sorted tasks with flags, filters, saved tabs and **Take next task** |
| Review workspace | Admin | Receipt viewer with QR read, receipt-against-expectation table, risk signals, decision panel with second-approval rule, timeline |
| Overview | Portal | Outage banner, today's KPIs, recent checks, rejections by reason, bank status |
| Verify a payment | Portal (Cashier) | Cashier-only menu, bank picker, reference field, receipt drop zone, verified result card |
| Verifications + detail | Portal | Log with saved views and a drawer explaining a duplicate rejection |
| API clients | Portal | One-time secret display, client table with rotate and revoke, quick-start request |
| Choose (EN) and Choose (አማ) | Upload page | The same customer screen in English and Amharic |
| In review (EN) | Upload page | What the customer sees when the bank is down and a reviewer takes over |
| Scan, Verified, Do not release (አማ) | Cashier app | QR scan with typed fallback, full-screen green and red outcomes, Ethiopian-calendar date |

Two additions came out of drawing the screens: the **Cashier** role (also added to the roles table in the main tab), and the review workspace showing the bank's outage as a banner instead of an empty column.

