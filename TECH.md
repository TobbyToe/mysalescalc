# My Sales Calc — Technical Reference / 技术参考文档

> 自维护用途 · Self-maintenance reference  
> Last updated: 2026-05-09

---

## Changelog / 版本记录

### v3.0 — History, Expenses, Auth / 历史增强·费用·登录 (2026-05-09)
**New features / 新功能：**
- **Password login**: SHA-256 salted hash auth, session via `sessionStorage` (expires on tab close)
  - Password: `admin8888` · Salt: `msc$2026#` · Hash stored in `AUTH_HASH` constant
  - Admin status bar (top, 24px, dark blue) shows logged-in user + Log out button
- **Expenses tab** (4 items): Lunch, Petrol, Others-1, Others-2 — two "Others" have editable labels; all blur to 2 d.p.
- **History page** (replaces modal):
  - Grouped by day with sticky date subtitles + day-level checkboxes
  - Fuzzy search (normalises spaces / hyphens before compare)
  - Bulk actions bar: Del · CSV · Summary (PDF & Share removed)
  - Per-invoice: Edit (reloads to Sales), Delete, structured-text export (file-text icon)
  - All CSV / All JSON export buttons (top-right)
- **Export / 商家档案**:
  - Structured invoice text (`buildInvoiceText`) — Chinese format with line items
  - Visit note (`buildVisitNote`) — one-line CRM summary in Chinese
  - Per-item CSV (`buildCSV`) — 11 columns, suitable for Excel pivot
  - Markdown table share text (`buildShareText`)
- **Van Sales improvements**:
  - Load-in form auto-collapses after Confirm; chevron toggle button next to Edit
  - Summary & Clear Day moved to fixed bottom bar (`#van-action-bar`)
  - Summary modal: rows grouped by T/E/Q with colour coding + subtotals + audit box

**Changed / 修改：**
- Removed Print button from invoice modal
- Removed PDF & Share buttons from History bulk-action bar
- Service Worker cache bumped `msc-v2` → `msc-v3`; added `skipWaiting` + `activate` cleanup

### v2.0 — Van Sales Module / 车销管理模块 (2026-05-09)
**New features / 新功能：**
- Added **Van Sales tab** (bottom navigation: Sales | Van | Expenses | History)
- **Morning Load-In**: record daily stock taken out, stored in `localStorage` keyed by date
- **Real-time inventory deduction**: sold quantities calculated from today's saved invoices
- **Evening settlement**: sales totals grouped by T·Cash / E·Transfer / Q·Cheque
- **Day Summary**: one-tap text export (copy to clipboard) for boss/front desk
- **Clear Day**: wipes today's load-in only, invoice history preserved

**Changed / 修改：**
- Payment type: `isTransfer: boolean` → `paymentType: 'cash' | 'transfer' | 'cheque'`
- Invoice badge shows CASH / TRANSFER / CHEQUE with distinct colours
- `getInvPayType(inv)` helper for backward-compatible old invoice reads

### v1.0 — Initial release / 初始发布 (2026-04)
- Single-file offline PWA invoice generator
- GST-inclusive pricing, Cash/Transfer, Cat A/B discounts, IndexedDB history

---

## 1. Project Overview / 项目概述

**My Sales Calc** is an offline-capable single-file PWA for generating GST-inclusive tax invoices for an Australian Korean food wholesale business.

**My Sales Calc** 是一个离线单文件 PWA，用于澳洲韩国食品批发商快速生成含GST税务发票。

| Item | Detail |
|------|--------|
| Build | Single HTML file, no build step / 单文件，无需构建 |
| Styling | Tailwind CSS v2.2.19 (CDN, pinned) |
| Icons | Lucide Icons v0.284.0 (CDN, pinned) |
| Offline | Service Worker (PWA), cache name `msc-v3` |
| Storage | localStorage + IndexedDB + sessionStorage (auth) |

---

## 2. File Structure / 文件结构

```
mysalescalc-1/
├── index.html                    # All UI + logic / 全部代码
├── CLAUDE.md                     # AI coding rules / AI操作规范
├── TECH.md                       # This document / 本文档
└── references/
    ├── products - Sheet1.csv     # Source of truth for products / 产品数据源
    └── invoice-template.pdf      # Invoice layout reference / 发票排版参考
```

**index.html internal sections / index.html 内部分区：**

| Section | Content |
|---------|---------|
| CSS styles | Login overlay styles, btn-press, print media |
| Login overlay HTML | `#login-overlay` — full-screen auth gate |
| Auth bar HTML | `#auth-bar` — 24px top bar, admin label + Log out |
| Nav | 4 tabs: Sales · Van · Expenses · History |
| Sales page | Invoice calculator, textarea input |
| Van page | Load-in form, inventory table, fixed bottom bar |
| History page | Grouped day list, bulk-action bar, search |
| Expenses page | 4 expense rows with editable labels |
| Modals | Invoice modal, Summary modal, Export modal |
| `PRODUCTS` array | Product definitions (price, shortcode, eaBox) |
| App state | Global variables |
| `parseCommand()` | Single-line shortcode parser |
| `parseTextarea()` | Multi-line textarea parser |
| Calculations | Discount, rounding, totals |
| UI rendering | `renderControlButtons`, `onTextareaInput`, etc. |
| Invoice logic | `buildInvoiceHTML`, `saveInvoice`, invoice number |
| History logic | `renderHistoryPage`, `deleteSelected`, export fns |
| Expenses logic | `saveExpenses`, `loadExpenses`, `fmtExp` |
| Van Sales logic | Load-in, inventory, summary, `generateSummary` |
| Auth logic | `doLogin`, `doLogout`, `hashPassword`, `checkAuth` |
| Service Worker | Inline Blob SW, cache CDN assets |

---

## 3. Authentication / 登录验证

| Property | Value |
|----------|-------|
| Password | `admin8888` |
| Salt | `msc$2026#` |
| Algorithm | SHA-256 via `crypto.subtle` (WebCrypto API) |
| Stored | Hash only — plaintext never in source |
| Session | `sessionStorage` key `msc_auth_v1` |
| Expiry | Tab/window close — no time limit within session |

**To change the password / 修改密码：**
1. Compute new hash in terminal (use **single quotes** to prevent `$` expansion):
   ```bash
   echo -n 'msc$2026#NEWPASSWORD' | shasum -a 256
   ```
2. Replace `AUTH_HASH` constant in index.html with the new hash.

---

## 4. Product Data / 产品数据

### CSV format / CSV格式
`references/products - Sheet1.csv`
```
no., shorts, products, unit price, ea/box
```

### JS mapping / JS字段映射
`PRODUCTS` array in index.html:

| CSV column | JS field | Description |
|------------|----------|-------------|
| `shorts` | `shorts` | Shortcode used for input / 快捷码 |
| `products` | `name` | Full product name / 产品全名 |
| `unit price` | `price` | GST-inclusive unit price / 含GST单价 |
| `ea/box` | `eaBox` | Units per box / 每箱数量 |

**Important: all prices are GST-inclusive. Do NOT add GST on top.**  
**重要：所有价格已含GST，不要再叠加GST。**

### How to add a new product / 如何新增产品

1. Add a row to `references/products - Sheet1.csv`
2. Add a matching entry to the `PRODUCTS` array in index.html:
   ```js
   { no:60, shorts:'XX', name:'Product Name', price:9.90, eaBox:12 },
   ```
3. If it should have auto 10% off, add its shortcode to `CAT_A` (see Section 6).

---

## 5. Command Parsing / 快捷码解析

### Input format / 输入格式

```
Line 1:   Store name                      ← 店名
Line 2+:  [ex?][ShortCode][Qty][box?]    ← 产品指令
```

Spaces are stripped before parsing — `v 1 box` = `v1box`.

### Unit suffix / 单位后缀

| Input | Result |
|-------|--------|
| `c10` | 10 ea of Mat Kimchi 450g |
| `c2box` | 2 × 20 = 40 ea |
| `j1box` | 1 × 10 = 10 ea |

### Exchange prefix / 换货前缀 `ex`

```
exc10      ← exchange 10 × Mat Kimchi 450g  (qty shown, total $0)
exj1box    ← exchange 1 box of Jarae Seaweed
```

### Parsing order / 解析顺序

Shortcodes matched **longest-first** to prevent ambiguity (`W` vs `WC` vs `WK`).  
Note: `1.5K` and `1.5` both accepted as alias for Mat Kimchi 1.5kg.

### Error handling / 错误处理

Unrecognised lines shown as yellow warnings; valid items still processed.

---

## 6. Discount Rules / 折扣规则

```js
const CAT_A = new Set(['P', 'J', 'N']);
```

| Category | Condition | Auto discount | Global 2%/4% |
|----------|-----------|---------------|--------------|
| Cat A | shorts ∈ {P, J, N} | 10% off | ✗ Excluded |
| Salt | shorts = `Salt` | "Special" label | ✗ Excluded |
| Cat B | All others | None | ✓ Eligible |
| Exchange | `ex` prefix | None | ✗ Excluded |

---

## 7. Cash Rounding / 现金取整

Australian standard — nearest $0.05, Cash mode only.

```js
cashTotal    = Math.round(totalAfterDisc * 20) / 20;
cashRounding = cashTotal - totalAfterDisc;
```

Rounding applied **after all discounts**.

---

## 8. Invoice Footer / 发票页脚

### Cash mode
```
Sub Total              $XXX.XX
Discount               -$X.XX    ← if any discount
↳ Others 2% Off        -$X.XX    ← if global discount active
↳ Seaweed 10% Off      -$X.XX    ← if Cat A items present
Cash Rounding          ±$X.XX    ← if non-zero
Cash Total             $XXX.XX   ← bold
```

### Transfer / Cheque mode
```
Sub Total              $XXX.XX
Discount               -$X.XX
↳ Others 2% Off        -$X.XX
↳ Seaweed 10% Off      -$X.XX
Total                  $XXX.XX   ← no rounding
```

No GST line — prices are GST-inclusive.

---

## 9. Invoice Number / 发票号

Format: `DDMMYY` + store name normalised (uppercase, non-alphanumeric stripped).

```
"kfl flemington"  →  "090526KFLFLEMINGTON"
"D & K"           →  "090526DK"
```

---

## 10. Data Persistence / 数据持久化

### localStorage keys

| Key | Content |
|-----|---------|
| `orderText_v2` | Textarea content |
| `disc_v2` | Global discount (0 / 2 / 4) |
| `payType_v2` | Payment mode (`cash` / `transfer` / `cheque`) |
| `loadIn_YYYY-MM-DD` | Daily load-in quantities (Van tab) |
| `expenses_v1` | Expenses object `{lunch, petrol, others, others2, label1, label2}` |

### sessionStorage keys

| Key | Content |
|-----|---------|
| `msc_auth_v1` | `'1'` when authenticated; absent = not logged in |

### IndexedDB

| Property | Value |
|----------|-------|
| Database | `SalesCalcDB` |
| Object Store | `invoices` |
| Key | `invNum` |
| Stored fields | `invNum`, `dateStr`, `storeName`, `orderText`, `globalDiscount`, `paymentType`, `summary`, `html` |

> ⚠️ IndexedDB is browser-local. Clearing browser data erases all history.  
> Use **All CSV** or **All JSON** buttons in History to back up data.

---

## 11. History Page / 历史记录

- Invoices grouped by day; each day has a subtitle row with a day-level checkbox
- **Fuzzy search**: normalises spaces, hyphens, underscores, apostrophes before matching
- **Bulk actions**: Del · CSV (per-item, 11 columns) · Summary
- **Per-invoice actions**: file-text (export modal) · pencil (edit) · trash (delete)
- **Export modal** (`#export-modal`): structured text + Copy Text + Visit Note buttons
- **All CSV / All JSON** (top-right): export entire history at once

### CSV columns / CSV字段 (11 columns)
`Date · Invoice No · Store · Product · Shortcode · Qty · Unit Price · Discount · Line Total · Payment · Invoice Total`

---

## 12. Expenses Tab / 费用记录

4 rows: Lunch · Petrol · Others-1 (editable label) · Others-2 (editable label)  
- Labels editable inline; values blur to 2 decimal places (`fmtExp`)
- Stored in `expenses_v1` localStorage key
- Used in Van Summary audit box: Cash(T) − Expenses = Net Cash → Rounded (÷$0.50)

---

## 13. Van Sales Tab / 车销管理

### Load-in form / 早上装车
- Input quantities per product (ea or box depending on product type)
- Stored in `localStorage` keyed `loadIn_YYYY-MM-DD`
- Auto-collapses after Confirm; chevron toggle button to expand/collapse
- **Edit** button re-opens form for correction

### Inventory table / 库存追踪
- Remaining = Loaded − Sold (sold calculated from today's saved invoices)

### Summary modal / 结算汇总
- Rows grouped by T·Cash (green) / E·Transfer (blue) / Q·Cheque (orange)
- Subtotals per group; Grand Total; audit box (Cash − Expenses = Net)

### Fixed bottom bar / 固定底部栏
- **Summary** button → opens summary modal
- **Clear Day** → wipes today's load-in (invoice history preserved)

---

## 14. PWA / Service Worker

- Registered inline via `URL.createObjectURL(new Blob([...]))` — no separate SW file
- Cache name: `msc-v3`
- `skipWaiting` + `activate` handler clears old caches on update
- Caches Tailwind CSS + Lucide Icons; app works fully offline after first load

**To force update on device / 强制更新缓存：**  
Hard refresh: Cmd+Shift+R (desktop) or clear site data (mobile Safari).

---

## 15. Common Modification Scenarios / 常见修改场景

### Change a product price / 改产品价格
Edit `price` in `PRODUCTS` array. Update CSV too.

### Add a new product / 新增产品
1. Add to `PRODUCTS` array in index.html
2. Add to `references/products - Sheet1.csv`
3. If auto 10% off: add shortcode to `CAT_A`

### Change the login password / 修改登录密码
```bash
echo -n 'msc$2026#NEWPASSWORD' | shasum -a 256
```
Replace `AUTH_HASH` in index.html with the output hash.

### Change which products get auto 10% off / 改哪些产品固定9折
```js
const CAT_A = new Set(['P', 'J', 'N']);  // add or remove shortcodes
```

### Change invoice company info / 改发票公司抬头
Edit strings in `buildInvoiceHTML()`:
```js
<div>From: Korean Foods P/L</div>
<div>ABN 65 122 947 586</div>
```

---

## 16. Verification Checklist / 验证检查清单

After any change, test in browser:

- [ ] Login with `admin8888` → enters app; wrong password → error shown
- [ ] Log out → returns to login overlay
- [ ] Cat A (J/P) + Cat B + 2% → footer shows 3-line discount breakdown
- [ ] Exchange (`ex` prefix) → $0, excluded from discounts
- [ ] Cash mode → rounding line, total to $0.05
- [ ] Transfer/Cheque → no rounding line
- [ ] Salt → "Special" label, no global discount
- [ ] `box` suffix → qty = N × eaBox
- [ ] Store name with special chars → invoice number strips correctly
- [ ] Reload → textarea + discount state restored
- [ ] Save invoice → appears in History, grouped under today's date
- [ ] History search → fuzzy match works (spaces/hyphens normalised)
- [ ] History bulk select → Del · CSV · Summary work
- [ ] Expenses labels editable; values show 2 d.p. on blur
- [ ] Van load-in auto-collapses after Confirm; chevron toggles it
- [ ] Van Summary shows T/E/Q groups + audit box
