# My Sales Calc — Technical Reference / 技术参考文档

> 自维护用途 · Self-maintenance reference  
> Last updated: 2026-05-09

---

## 1. Project Overview / 项目概述

**My Sales Calc** is an offline-capable single-file Progressive Web App (PWA) for generating GST-inclusive tax invoices for an Australian Korean food wholesale business.

**My Sales Calc** 是一个离线单文件 PWA，用于澳洲韩国食品批发商快速生成含GST税务发票。

| Item | Detail |
|------|--------|
| Build | Single HTML file, no build step / 单文件，无需构建 |
| Styling | Tailwind CSS v2.2.19 (CDN, pinned) |
| Icons | Lucide Icons v0.284.0 (CDN, pinned) |
| Offline | Service Worker (PWA) |
| Storage | localStorage + IndexedDB |

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

| Section | Line (approx) | Content |
|---------|---------------|---------|
| 1 | 182 | `PRODUCTS` array / 产品数组 |
| 2 | 244 | App state / 应用状态 |
| 3 | 256 | `parseCommand()` — command parser / 快捷码解析 |
| 4 | 296 | `parseTextarea()` — textarea parser / 多行输入解析 |
| 5 | 332 | Calculations / 金额计算 |
| 6–7 | 387 | Live feedback + controls / 实时反馈与控件 |
| 8 | 456 | Invoice number / 发票号 |
| 9 | 480 | `buildInvoiceHTML()` — invoice renderer / 发票HTML构建 |
| 10 | 610 | Modal actions / 模态框操作 |
| 11 | 679 | IndexedDB history / 历史记录 |
| 12 | 761 | localStorage persistence / 持久化 |
| 13 | 774 | Service Worker / 离线缓存 |
| 14 | 793 | Init / 初始化 |

---

## 3. Product Data / 产品数据

### CSV format / CSV格式
`references/products - Sheet1.csv`
```
no., shorts, products, unit price, ea/box
```

### JS mapping / JS字段映射
`PRODUCTS` array in index.html (line 182):

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
3. If it should have auto 10% off, add its shortcode to `CAT_A` (see Section 5).

### How to update a price / 如何修改价格

Edit the `price` field in the `PRODUCTS` array. Also update the CSV to keep them in sync.

---

## 4. Command Parsing / 快捷码解析

### Input format / 输入格式 (index.html line 256)

```
Line 1:   Store name              ← 店名
Line 2+:  [ex?][ShortCode][Qty][box?]   ← 产品指令
```

Spaces are stripped before parsing — `v 1 box` = `v1box`.  
解析前去掉所有空格，`v 1 box` 等同于 `v1box`。

### Unit suffix / 单位后缀

| Input example | Result |
|---------------|--------|
| `c10` | 10 ea of Mat Kimchi 450g |
| `c2box` | 2 × 20 = 40 ea (eaBox = 20) |
| `j1box` | 1 × 10 = 10 ea (eaBox = 10) |

### Exchange prefix / 换货前缀 `ex`

Prepend `ex` to mark an item as exchange — quantity is tracked but total shows $0.  
加 `ex` 前缀表示换货，数量正常显示，金额显示 $0。

```
exc10      ← exchange 10 × Mat Kimchi 450g
exj1box    ← exchange 1 box of Jarae Seaweed
```

### Parsing order / 解析顺序

Shortcodes are matched **longest-first** to prevent ambiguity (e.g. `W` vs `WC` vs `WK`).  
按快捷码**长度降序**匹配，防止短码误匹配（如 `W` 误匹配 `WC`）。

### Error handling / 错误处理

- Unrecognized lines: shown as yellow warnings above the invoice, skipped silently.
- Valid lines are still processed. / 有效行正常处理，不受错误行影响。

---

## 5. Discount Rules / 折扣规则

### Category A — Auto 10% Off (index.html line 235, 342)

Products with `shorts` = **`P`**, **`J`**, **`N`** always get 10% off.  
快捷码为 `P`、`J`、`N` 的产品固定9折。

```js
const CAT_A = new Set(['P', 'J', 'N']);
```

- **ALWAYS discounted, NEVER eligible for global 2%/4%.**
- **始终享受9折，排除在全局折扣之外。**

### Salt — Special label

`shorts = 'Salt'` shows **"Special"** in the Disc column. No percentage discount.  
显示 "Special" 标签，无百分比折扣，也不参与全局折扣。

### Category B — Global Discount

All other products are eligible for the global 2% or 4% discount button.  
其余产品参与全局 2%/4% 折扣按钮。

### Exchange items / 换货产品

Exchange items (`ex` prefix) are **excluded from all discounts** and show $0.  
换货产品不参与任何折扣，金额为 $0。

### Discount table summary / 折扣规则汇总

| Category | Condition | Auto discount | Global 2%/4% |
|----------|-----------|---------------|--------------|
| Cat A | shorts ∈ {P, J, N} | 10% off | ✗ Excluded |
| Salt | shorts = Salt | "Special" label | ✗ Excluded |
| Cat B | All others | None | ✓ Eligible |
| Exchange | `ex` prefix | None | ✗ Excluded |

---

## 6. Cash Rounding / 现金取整 (index.html line 373)

Australian standard rounding to nearest $0.05. Only applies in **Cash mode**.  
澳洲标准现金取整至最近5分，仅 **Cash 模式**生效。

```js
cashTotal    = Math.round(totalAfterDisc * 20) / 20;
cashRounding = cashTotal - totalAfterDisc;   // can be positive or negative
```

| Cents | Example | Rounds to |
|-------|---------|-----------|
| .01–.02 | $37.51 | $37.50 (down) |
| .03–.07 | $37.56 | $37.55 (to 5) |
| .08–.09 | $37.58 | $37.60 (up) |

Rounding is applied **after all discounts**.  
取整在**所有折扣之后**计算。

---

## 7. Invoice Footer / 发票页脚 (index.html line 531)

### Cash mode / 现金模式

```
Sub Total              $XXX.XX
Discount               -$X.XX    ← shown only if any discount exists
↳ Others 2% Off        -$X.XX    ← shown only if global discount active
↳ Seaweed 10% Off      -$X.XX    ← shown only if Cat A items present
Cash Rounding          ±$X.XX    ← shown only if non-zero
Cash Total             $XXX.XX   ← bold, highlighted
```

### Transfer mode / 转账模式

```
Sub Total              $XXX.XX
Discount               -$X.XX    ← same breakdown as above
↳ Others 2% Off        -$X.XX
↳ Seaweed 10% Off      -$X.XX
Total                  $XXX.XX   ← no rounding lines
```

**No GST line on either mode** — prices are GST-inclusive.  
**两种模式均无GST行** — 价格已含GST。

---

## 8. Invoice Number / 发票号 (index.html line 464)

Format / 格式: `DDMMYY` + store name normalized  
Store name normalization: uppercase, remove all non-alphanumeric characters.  
店名标准化：大写，去掉字母数字以外的所有字符（空格、`&` 等）。

```
"kfl flemington"  →  "KFLFLEMINGTON"  →  "090526KFLFLEMINGTON"
"D & K"           →  "DK"             →  "090526DK"
```

---

## 9. Data Persistence / 数据持久化

### localStorage — current cart / 当前购物车

| Key | Content |
|-----|---------|
| `orderText_v2` | Textarea content / 输入框内容 |
| `disc_v2` | Global discount value (0/2/4) |
| `transfer_v2` | Payment mode ("true"/"false") |

Restored automatically on page load. / 页面加载时自动恢复。

### IndexedDB — invoice history / 历史发票

| Property | Value |
|----------|-------|
| Database | `SalesCalcDB` |
| Object Store | `invoices` |
| Key | `invNum` (invoice number) |
| Stored fields | `invNum`, `dateStr`, `storeName`, `orderText`, `globalDiscount`, `isTransfer`, `summary`, `html` |

> ⚠️ IndexedDB is browser-local. Clearing browser data will erase history.  
> ⚠️ IndexedDB 绑定浏览器，清除浏览器数据会丢失历史记录。  
> Recommended: add an Export JSON button for backup. / 建议添加导出JSON功能备份。

---

## 10. PWA / Service Worker (index.html line 774)

- Registered inline via `URL.createObjectURL(new Blob([...]))` — no separate SW file needed.
- Caches Tailwind CSS and Lucide Icons on install.
- Serves cached assets on subsequent fetches — app works fully offline after first load.
- Cache name: `msc-v2`

---

## 11. Common Modification Scenarios / 常见修改场景

### Change a product price / 改产品价格
Edit `price` in the `PRODUCTS` array (index.html ~line 182). Update CSV too.

### Add a new product / 新增产品
1. Add to `PRODUCTS` array in index.html
2. Add to `references/products - Sheet1.csv`

### Change which products get auto 10% off / 改哪些产品固定9折
Edit `CAT_A` set (index.html ~line 235):
```js
const CAT_A = new Set(['P', 'J', 'N']);  // add or remove shortcodes here
```

### Add a new global discount tier / 新增折扣档位（如5%）
1. Add a button in the HTML controls section
2. Call `setDiscount(5)` on click

### Change the invoice header company info / 改发票抬头公司信息
Edit strings in `buildInvoiceHTML()` (~line 560):
```js
<div>From: Korean Foods P/L</div>
<div>ABN 65 122 947 586</div>
```

---

## 12. Verification Checklist / 验证检查清单

After any change, open index.html in a browser and test:  
修改后在浏览器打开 index.html 验证：

- [ ] Cat A product (J/P) + Cat B product + 2% → footer shows `Discount`, `↳ Others 2%`, `↳ Seaweed 10%`
- [ ] Exchange item (`ex` prefix) → $0 on invoice, not counted in any discount
- [ ] Cash mode → rounding line appears (if non-zero), total rounded to $0.05
- [ ] Transfer mode → no rounding line, shows `Total` not `Cash Total`
- [ ] Salt product → shows "Special" in Disc column, not affected by global discount
- [ ] `box` suffix → quantity = N × eaBox value
- [ ] Store name with spaces/special chars → invoice number strips them correctly
- [ ] Reload page → textarea + discount state restored from localStorage
- [ ] Save invoice → appears in History modal
