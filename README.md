# 🛒 ASOS Supply Chain Analytics — Phantom Revenue Audit

> **An independent data audit of ASOS's inventory gaps, identifying which brands and price points are losing the most revenue when best-sellers go out of stock.**

---

## The Business Problem

ASOS lists thousands of products across dozens of brands. But buried inside the size-availability data is a signal most analysts overlook: out-of-stock sizes on **live product pages**.

When a £260 Topshop leather jacket is still listed on the website but 7 of its sizes are unavailable, that is not just a logistics inconvenience — it is **quantifiable, preventable lost revenue** happening every single day.

This project approaches ASOS's catalog as an external data consultant and answers one question:

> **Which brands and products are costing ASOS the most money by failing to keep their best-sellers in stock?**

---

## Dataset

| Property | Detail |
|----------|--------|
| Source | Publicly scraped ASOS product catalog |
| File | `products_asos.csv` |
| Rows after cleaning | 18,000+ |
| Key columns | `name`, `size`, `category`, `price`, `color`, `description` |
| Challenge | No dedicated brand column — required custom extraction |

---

## Methodology

### Step 1 — Data Ingestion & Cleaning

Loaded raw CSV with `on_bad_lines='skip'` to handle malformed rows containing unescaped quote characters. Coerced `price` to numeric and dropped non-parseable values. Filtered to brands with more than 5 product listings to remove statistical noise.

### Step 2 — Brand Extraction via NLP Heuristic

The raw dataset contained **no brand column**. After inspecting the `description` field, a consistent pattern emerged across thousands of rows:

```
"Oversized t-shirt by New Look"
"Wax jacket by Barbour"
"Plisse midi dress by Mango"
```

A text-splitting function extracts the token immediately following `" by "`. A manual mapping layer then normalises extracted aliases to canonical brand names:

```python
brand_map = {
    'New': 'New Look',
    'River': 'River Island',
    'Miss': 'Miss Selfridge',
    'TopshopWelcome': 'Topshop'
}
```

> **Known limitation:** This heuristic fails when "by" appears in other sentence contexts (e.g., "inspired by..."). A production pipeline would use a curated brand taxonomy or a dedicated NER model.

### Step 3 — Phantom Revenue Calculation

The `size` column encodes availability inline, for example:

```
"UK 6, UK 8 - Out of stock, UK 10, UK 12 - Out of stock, UK 14"
```

A custom function parses each row to extract the out-of-stock count and rate:

```python
def calculate_phantom_revenue(size_str):
    sizes = size_str.split(',')
    total_sizes = len(sizes)
    out_of_stock_count = size_str.count('Out of stock')
    rate = out_of_stock_count / total_sizes if total_sizes > 0 else 0.0
    return out_of_stock_count, rate
```

Phantom Revenue is then estimated as:

```
Lost_Revenue = price × stockout_count
```

> ⚠️ **Methodological note:** This is a conservative upper-bound estimate. It assumes every unavailable size represents one foregone sale at full price. The figures are best interpreted as **relative signals for prioritisation**, not precise revenue forecasts.

### Step 4 — Brand Strategy Quadrant Analysis

Aggregated metrics per brand:

| Axis / Dimension | Metric | Business Signal |
|-----------------|--------|-----------------|
| X-axis | Average price (£) | Premium vs. budget positioning |
| Y-axis | Mean stockout rate | Demand intensity / supply failure |
| Bubble size | Total lost revenue | Urgency of restocking action |

Quadrant thresholds: **Price > £40 · Stockout rate > 0.40**

---

## Results

### Brand Volume in Dataset

| Brand | Products Listed |
|-------|----------------|
| ASOS | 4,844 |
| Topshop | 1,017 |
| New Look | 511 |
| River Island | 467 |
| Miss Selfridge | 429 |

### Top 5 Revenue-Losing Individual Products

| Brand | Product | Price | Sizes OOS | Est. Lost Revenue |
|-------|---------|-------|-----------|-------------------|
| Barbour | Beadnell Wax Jacket (Black) | £219 | 9 | £1,971 |
| Topshop | Premium Real Leather Jacket | £260 | 7 | £1,820 |
| ASOS | Premium Real Leather Trench Coat | £220 | 7 | £1,540 |
| ASOS | EDITION Geo Embellished Fringe Dress | £250 | 6 | £1,500 |
| Topshop | Baggy Co-ord Jeans (Green Cord) | £50 | 27 | £1,350 |

> The Topshop co-ord jeans entry is particularly striking — **27 missing sizes on a £50 item** points to a replenishment process failure, not a demand gap.

---

## Brand Strategy Quadrant

![Brand Strategy Analysis](Brand%20Strategy%20Analysis.png)

### How to Read This Chart

| Quadrant | Brands | Interpretation |
|----------|--------|----------------|
| 🔴 Top-Right (High Price + High Stockout) | TopshopLove, Pull&Bear, Mango, Vila, Naked | **Priority restock.** Customers are paying premium prices and buying faster than ASOS can replenish. Revenue is being lost daily. |
| 🟡 Top-Left (Low Price + High Stockout) | Various | High-volume essentials. Stockouts here affect the widest customer segment. |
| 🟠 Bottom-Right (High Price + Low Stockout) | — | Slow-moving premium items. Potential overstock — review for clearance. |
| ⚪ Bottom-Left (Low Price + Low Stockout) | Majority of catalogue | Stable, low-risk zone. Operationally sound. |

---

## Strategic Recommendations

**1. Prioritise top-right quadrant inventory**
Mango, Pull&Bear, TopshopLove, and Vila show both premium price tolerance and sustained demand. Increase safety stock and shorten replenishment cycles for these labels.

**2. Investigate the Topshop Baggy Co-ord operationally**
27 out-of-stock sizes on a single £50 product is not a demand problem — it is a process failure. This warrants a direct operational review of the replenishment trigger for this SKU.

**3. Audit bottom-right quadrant for clearance**
High-priced items with low stockout rates are sitting in inventory without generating demand signals. Targeted promotions would free up warehouse allocation for higher-velocity products.

**4. Strengthen the brand extraction pipeline**
The current heuristic is a good first pass but misses brands that don't follow the `[product] by [brand]` pattern. A named entity recognition (NER) model would significantly improve brand coverage and analytical accuracy.

---

## Tech Stack

```
Python 3.x
├── pandas        — data loading, cleaning, groupby aggregations
├── matplotlib    — figure layout, quadrant lines, text annotations
└── seaborn       — proportional bubble scatter plot
```

---

## Project Structure

```
asos-supply-chain-analytics/
├── asos_analysis.ipynb           # Full end-to-end analysis notebook
├── products_asos.csv             # Raw ASOS product data (scraped, via Git LFS)
├── Brand Strategy Analysis.png  # Output quadrant visualisation
└── README.md
```

---

## How to Reproduce

```bash
git clone https://github.com/JINZO-AI/asos-supply-chain-analytics
cd asos-supply-chain-analytics
pip install pandas matplotlib seaborn
jupyter notebook asos_analysis.ipynb
```

---

## Author

**Jawad** — Big Data Engineering student, EPI Digital School, Sousse, Tunisia

[GitHub](https://github.com/JINZO-AI)

---

*This project is an independent analytical exercise. All data used is publicly visible on the ASOS website. Not affiliated with or endorsed by ASOS.*
