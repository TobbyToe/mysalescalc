# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**My Sales Calc** is an offline single-file PWA for generating GST-inclusive tax invoices for an Australian Korean food wholesale business. All code lives in `index.html` — no build step, no NPM, no bundler.

## Running the App

Open `index.html` directly in a browser. No server required. Default login password: `admin8888`.

To test changes: save `index.html`, hard-refresh the browser (`Cmd+Shift+R` on Mac) to bypass the Service Worker cache.

## Reference Files (Source of Truth)

- `references/products - Sheet1.csv` — canonical product list: `shorts` (shortcode), `products` (full name), `unit price` (GST-inclusive), `ea/box` (box multiplier)
- `references/invoice-template.pdf` — visual layout reference for the Tax Invoice

**Always keep the `PRODUCTS` array in `index.html` in sync with the CSV.**

## Architecture

`index.html` is a single 2000+ line file with inline HTML, CSS, and JavaScript. Internal sections (top to bottom):

| Section | Key identifiers |
|---------|----------------|
| Auth gate | `#login-overlay`, `AUTH_HASH`, `doLogin()`, `doLogout()` |
| Admin bar + nav | `#auth-bar`, `switchTab()` |
| Sales page | `#order-input` textarea, `parseTextarea()`, `buildInvoiceHTML()` |
| Van Sales page | `#van-tab`, `getLoadIn()`, `renderInventoryTable()`, `generateSummary()` |
| History page | `#history-tab`, `renderHistoryPage()`, IndexedDB `SalesCalcDB` |
| Expenses page | `#expenses-tab`, `saveExpenses()`, `expenses_v1` localStorage key |
| Product data | `PRODUCTS` array, `CAT_A` Set |
| Service Worker | Inline Blob SW, cache name `msc-v3` |

## CDN Dependencies (Pinned — Do Not Upgrade)

- Tailwind CSS: `v2.2.19`
- Lucide Icons: `v0.284.0`

## Business Logic

### Command Parsing

Input textarea: Line 1 = store name, Lines 2+ = product commands. Spaces are stripped before parsing (`v 1 box` = `v1box`).

Format: `[ex?][ShortCode][Qty][box?]`

- `c10` → 10 ea of Mat Kimchi 450g
- `c2box` → 2 × eaBox value = 40 ea
- `exc10` → exchange 10 × Mat Kimchi (qty shown, line total $0, excluded from all discounts)

Shortcodes matched **longest-first** to avoid ambiguity (e.g. `W` vs `WC` vs `WK`). Unrecognised lines show yellow warnings; valid items still process.

### Discount Rules

```js
const CAT_A = new Set(['P', 'J', 'N']);
```

| Category | Condition | Auto discount | Global 2%/4% |
|----------|-----------|---------------|--------------|
| Cat A | shorts ∈ {P, J, N} | 10% off always | Excluded |
| Salt | shorts = `Salt` | "Special" label | Excluded |
| Cat B | All others | None | Eligible |
| Exchange (`ex` prefix) | Any | None | Excluded |

### Cash Rounding (Australian Standard)

Applied after all discounts, Cash mode only. Formula: `Math.round(total * 20) / 20` (nearest $0.05).

### Invoice Footer Order

Cash mode: Sub Total → Global Discount breakdown → Cash Rounding (if non-zero) → **Cash Total**

Transfer/Cheque mode: Sub Total → Global Discount breakdown → **Total** (no rounding lines)

No GST line — all prices are GST-inclusive.

### Invoice Number Format

`DDMMYY` + store name uppercased with non-alphanumeric stripped. E.g. `"kfl flemington"` → `"090526KFLFLEMINGTON"`.

### Payment Types

`paymentType`: `'cash'` | `'transfer'` | `'cheque'`. Old invoices used `isTransfer: boolean` — use `getInvPayType(inv)` for backward-compatible reads.

## Authentication

SHA-256 salted hash. Salt: `msc$2026#`. Hash stored in `AUTH_HASH` constant in `index.html`. Session stored in `sessionStorage` key `msc_auth_v1`; expires on tab close.

To change password:
```bash
echo -n 'msc$2026#NEWPASSWORD' | shasum -a 256
```
Replace `AUTH_HASH` in `index.html`.

## Data Persistence

### localStorage Keys

| Key | Content |
|-----|---------|
| `orderText_v2` | Textarea content |
| `disc_v2` | Global discount (0 / 2 / 4) |
| `payType_v2` | Payment mode |
| `loadIn_YYYY-MM-DD` | Daily Van load-in quantities |
| `expenses_v1` | Expenses `{lunch, petrol, others, others2, label1, label2}` |

### IndexedDB

Database: `SalesCalcDB`, Store: `invoices`, Key: `invNum`

Stored fields: `invNum`, `dateStr`, `storeName`, `orderText`, `globalDiscount`, `paymentType`, `summary`, `html`

## Output Constraints

- Always deliver the full, runnable `index.html` — not diffs or partial snippets.
- Comments and documentation should be in both English and Chinese.
- Do not upgrade pinned CDN versions.

## Verification Checklist (After Any Change)

- [ ] Login `admin8888` → enters app; wrong password → error shown
- [ ] Cat A (J/P) + Cat B + 2% → footer shows 3-line discount breakdown
- [ ] Exchange (`ex` prefix) → $0, excluded from discounts
- [ ] Cash mode → rounding line, total to $0.05
- [ ] Transfer/Cheque → no rounding line
- [ ] Salt → "Special" label, no global discount
- [ ] `box` suffix → qty = N × eaBox
- [ ] Reload → textarea + discount state restored from localStorage
- [ ] Save invoice → appears in History grouped by today's date
- [ ] Van load-in auto-collapses after Confirm; chevron toggles it
