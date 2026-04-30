# Project Framework: My Sales Calc (Offline Single-File PWA)

## 1. Core Reference Files
AI must strictly refer to the following files in the `references/` directory for all logic and styling:
- **Data Source**: `references/products - Sheet1.csv`
  - Use this for `shorts` (shortcuts), `products` (full name), `unit_price`, and `ea/box` (box multiplier)[cite: 1].
- **UI Template**: `references/invoice-template.pdf`
  - Use this to replicate the visual layout, headers, and footer of the Tax Invoice.

## 2. Business Logic & Parsing
### A. Command Parsing (Shortcut System)
- The input must support `[ShortCode][Quantity][Unit]`.
- **Unit "box"**: If the input ends with `box` (e.g., `c2box`), Quantity = `2 * ea/box` value from the CSV[cite: 1].
- **Unit "ea" (Default)**: If no unit is specified (e.g., `c10`), Quantity = `10`.

### B. Strict Discount Hierarchy
- **Category A (Auto 10% Off)**: Products with `shorts` = `P`, `J`, or `N`[cite: 1].
  - These items ALWAYS get a 10% discount on the unit price.
  - **CRITICAL**: These items are EXCLUDED from global 2%/4% discounts.
- **Category B (General)**: All other products.
  - Eligible for Global Discount buttons (2% or 4%).
- **Special Case**: `shorts` = `Salt` should be labeled as "Special" in the discount column[cite: 1].

### C. Payment Method & Cash Rounding
- A **"Transfer" toggle button** must be present in the UI.
  - **Default state: OFF** (i.e., default payment is **Cash**).
  - When toggled ON, payment is **Transfer** — no rounding is applied.
- **Cash Rounding Rule** (Australian standard, applies only when Transfer is OFF):
  - Applied AFTER all discounts (Category A 10%, global 2%/4%) and GST are calculated.
  - Round the final "Total Inclusive G.S.T" to the nearest $0.05:
    - Cents ending in 1–2 → round **down** (e.g. $37.51 → $37.50)
    - Cents ending in 3–7 → round to **5** (e.g. $37.56 → $37.55)
    - Cents ending in 8–9 → round **up** (e.g. $37.58 → $37.60)
  - Formula: `Math.round(total * 20) / 20`
- **Invoice footer** must show a **"Cash Rounding"** line (positive or negative adjustment) and a **"Cash Total"** line when in Cash mode. Transfer mode shows no rounding lines.

## 3. Technical Specifications
- **Build**: Single-file HTML/JS (No NPM, No build step).
- **Styling**: Tailwind CSS v2.2.19 via CDN (Fixed version).
- **Icons**: Lucide Icons v0.284.0 via CDN.
- **Persistence**: 
  - Save current cart to `localStorage` (Offline safety).
  - Save finalized invoices to `IndexedDB`.
- **PWA**: Include a basic Service Worker script for offline caching.

## 4. Invoice Formatting Requirements
- **Invoice # Generation**: `[DDMMYY] + [Store Initial]` (e.g., `300426KFLFH`).
- **Layout**: 
  - Top Right: Invoice Number.
  - Body: No., QTY, Description, Discount, Total $.
  - Footer (Cash mode):
    - Sub Total
    - Global Discount (if applied, e.g. "Discount 2%")
    - G.S.T (10%)
    - Total Inclusive G.S.T
    - Cash Rounding (e.g. `-$0.01` or `+$0.04`)
    - **Cash Total** (highlighted)
  - Footer (Transfer mode):
    - Sub Total
    - Global Discount (if applied)
    - G.S.T (10%)
    - Total Inclusive G.S.T (no rounding lines)

## 5. Output Constraints
- All responses should provide the full, runnable code within the single HTML file.
- Documentation and comments should be provided in both English and Chinese.


/mysalescalc
├── claude.md
└── references/
    ├── products - Sheet1.csv
    └── invoice-template.pdf   