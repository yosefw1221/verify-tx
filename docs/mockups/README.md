# Mockups

The 13 screens from the [Proofline UI mockups canvas](https://claude.ai/artifact/EYXDJVaKXKekjhuW3SPZdY), exported on 4 October 2026 so they can be read without access to the canvas. They are the source of truth for layout, copy and colour. All names, amounts and references are sample data and line up across screens.

Each screen comes as a PNG (what it looks like) and a static HTML file (exact inline styles, copy and structure; open it in a browser, the fonts are in `fonts/`). Links between screens work in the HTML.

| Screen | Files | Size | Built in |
| --- | --- | --- | --- |
| Admin · Dashboard | `Main.png`, `Main.html` | 1440 × 1200 | doc 11 |
| Admin · Review queue | `AdminQueue.png`, `AdminQueue.html` | 1440 × 1000 | doc 07 (shell: doc 11) |
| Admin · Review workspace | `AdminReview.png`, `AdminReview.html` | 1440 × 1040 | doc 07 |
| Portal · Overview | `PortalOverview.png`, `PortalOverview.html` | 1440 × 1060 | doc 09 |
| Portal · Verify a payment (cashier view) | `PortalVerify.png`, `PortalVerify.html` | 1440 × 1060 | doc 09 |
| Portal · Verifications and detail drawer | `PortalVerifications.png`, `PortalVerifications.html` | 1440 × 1060 | doc 09 |
| Portal · API clients | `PortalApiClients.png`, `PortalApiClients.html` | 1440 × 1060 | doc 03 (shell: doc 09) |
| Upload · Choose (EN) | `UploadChoose.png`, `UploadChoose.html` | 390 × 844 | doc 06 |
| Upload · Choose (አማ) | `UploadChooseAm.png`, `UploadChooseAm.html` | 390 × 844 | doc 06 |
| Upload · In review (EN) | `UploadInReview.png`, `UploadInReview.html` | 390 × 844 | doc 06 |
| Cashier · Scan | `CashierScan.png`, `CashierScan.html` | 390 × 844 | doc 10 |
| Cashier · Verified | `CashierVerified.png`, `CashierVerified.html` | 390 × 844 | doc 10 |
| Cashier · Do not release (አማ) | `CashierRejectedAm.png`, `CashierRejectedAm.html` | 390 × 844 | doc 10 |

Where a feature doc deliberately departs from a mockup, the overview lists it under "Corrections to the UI spec and mockups" (section 1), and the doc's Open Questions say why. Screens that aren't drawn follow the patterns described in `docs/reference/ui-spec.md` and the owning feature doc.

Fonts: IBM Plex Sans, IBM Plex Mono and Noto Sans Ethiopic, from the `@fontsource` packages (SIL Open Font License 1.1).
