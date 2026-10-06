---
paths:
  - "apps/web-portal/**"
  - "apps/web-upload/**"
  - "apps/web-admin/**"
  - "libs/ui/**"
---

# Frontend rules (Next.js apps and libs/ui)

**Match the mockups.** Before building a screen, open its PNG and HTML in `docs/mockups/`. Match the layout, copy, spacing and colours. Undrawn screens follow the owning feature doc and the patterns in `docs/reference/ui-spec.md`.

**Tokens and components**

- Colours, type and radii come from the `libs/ui` Tailwind tokens: UI spec "Design direction", plus doc 09 section 6.2 (status) and doc 11 section 6.8 (dark admin). No raw hex values in app code.
- Fonts: IBM Plex Sans (Latin), Noto Sans Ethiopic (Amharic, one pixel larger, 1.6 line height), IBM Plex Mono for references and IDs. Amounts use tabular figures and are right-aligned.
- Status is always icon plus word (`StatusBadge`), never colour alone. A verified badge always has its assurance chip.
- Reuse `libs/ui`: StatusBadge, AssuranceChip, ReasonText, Money, BankChip, CopyRef, MaskedValue, DataTable, DetailDrawer, Timeline, SlaTimer, EmptyState, ErrorBanner.
- Every list and page has a skeleton while loading, an empty state with one action, and an error banner that shows the request ID.

**Text, dates and numbers**

- No literal user-facing strings. Use next-intl keys from `libs/contracts/i18n/{en,am}`, add both locales in the same change, and build sentences with ICU messages, never by joining fragments.
- Dates and times go through the platform i18n formatters:
  - EAT, 24-hour clock;
  - Amharic shows the Ethiopian calendar with the Gregorian date on hover, unless the user's `calendar_pref` says otherwise;
  - the admin panel shows UTC in the tooltip;
  - merchants never see UTC.
- Money is formatted with `Money.format` (`ETB 1,500.00` in English, `1,500.00 ብር` in Amharic). Never use JS floats for amounts; they arrive as decimal strings.

**Data and security**

- Fetch through each app's `api()` wrapper: same origin, CSRF header on mutations, typed `ApiError`, handling for `401` and re-auth.
- Server state lives in TanStack Query. Polling intervals are the ones in the docs (for example doc 09 section 6.9).
- Never put payer names, phones, search text or tokens in the browser URL. Filters that contain PII stay in component state.
- Authorisation comes from the server (the `actions` and `nav` the BFF returns). The client only hides; the BFF decides.
- Respect the CSP in docs 09 and 10: no third-party scripts, analytics or CDNs, and fonts are self-hosted. The upload page has no analytics at all.

**Accessibility and performance**

- WCAG 2.2 AA: visible focus, everything works by keyboard (doc 07's keys in the review workspace), 4.5:1 contrast, and live regions for status changes.
- Supported sizes: portal works at 360 px; admin is designed for 1440 px and usable at 1024 px.
- Budgets:
  - Upload page: under 150 KB of JS and CSS, LCP under 2.5 s on 3G.
  - Cashier: under 200 KB of initial JS.
  - Portal shell: under 350 KB.
  - Tables beyond 100 rows are virtualised.
- The cashier PWA's service worker never caches `/api/*`, and the app never shows a result the server didn't send (doc 10).
