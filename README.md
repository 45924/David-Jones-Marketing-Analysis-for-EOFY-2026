# David Jones — Marketing & Customer Behaviour Analytics (MySQL)

A SQL analytics project that designs a relational database for an online retailer and uses it to answer 14 real marketing questions — channel efficiency, campaign ROI, customer lifetime value, product profitability, and more — to guide the **End of Financial Year (EOFY) 2026** marketing campaign.
---

## Table of contents
- [Overview](#overview)
- [Tech stack](#tech-stack)
- [Dataset](#dataset)
- [Database design](#database-design)
- [Repository structure](#repository-structure)
- [The 14 analyses](#the-14-analyses)
- [Headline findings](#headline-findings)
- [Limitations](#limitations)
- [Author](#author)
---

## Overview
David Jones wants to make its EOFY 2026 marketing spend more effective. Using a purpose-built MySQL database covering **1 January 2021 – 31 May 2026**, this project analyses customer behaviour and advertising performance to answer where ad budget converts best, which customers and products drive value, and how the next campaign should be shaped.

The work is split into two parts:

- **Part A — Database design:** entity modelling, a 12-table normalised schema, relationship/cardinality analysis and an ERD.
- **Part B — Business analytics:** 14 SQL queries, each answering a specific business concern, followed by EOFY 2026 recommendations.

## Tech stack

| Area | Tools |
|------|-------|
| Database | MySQL (built and run in MySQL Workbench) |
| Data generation | Python (synthetic dataset) |
| SQL features | CTEs, window functions (`RANK() OVER`), conditional aggregation, `CASE`, `NULLIF`/`COALESCE` guards, multi-table joins |

## Dataset

A **synthetic** dataset generated in Python to mimic realistic online retail and marketing behaviour. It covers 400 loyalty members across three tiers (Bronze, Silver, Gold), their online orders, and advertising activity across four platforms (Instagram, Facebook, WeChat, X) and three devices (Desktop, Tablet, Phone). All figures are illustrative and exist to demonstrate database and SQL skills, not to represent real David Jones performance.

## Database design

Twelve tables, normalised to third normal form, with associative entities resolving the many-to-many relationships between orders, products, advertisements and discounts.

**Core entities:** `customers`, `products`, `orders`, `campaigns`, `advertisements`, `discounts`

**Associative / event tables:** `order_products`, `interaction_events`, `advertisement_performance`, `advertisement_order`, `order_discount`, `order_product_discount`

Key relationships:
- A customer places many orders; an order contains many products (resolved by `order_products`).
- A campaign runs many advertisements; each advertisement logs many performance rows (`advertisement_performance`) and many interaction events (`interaction_events`).
- Advertisements drive orders through the `advertisement_order` bridge, which is how ad-attributed sales are measured.
- Discounts attach to whole orders (`order_discount`) or to individual order lines (`order_product_discount`).

> See `schema/erd.png` for the full entity-relationship diagram.

## Repository structure

```
.
├── README.md
├── data/
│   └── generate_synthetic_data.py     # Python data generator
├── schema/
│   ├── create_tables.sql              # DDL: 12-table schema
│   └── erd.png                        # Entity-relationship diagram
├── queries/
│   ├── 01_platform_ctr_conversion.sql
│   ├── 02_campaign_roi.sql
│   ├── ...
│   └── 14_avg_daily_profit.sql
├── report/
│   └── Assignment_1_Report.docx       # Full written analysis & recommendations
└── dashboard/
    └── PowerBi                        # Interactive Dashboard for Visualization
```
---

## The 14 analyses

Each query maps to one business concern. The analyses fall into four themes:

| Theme | Concerns |
|-------|----------|
| Advertising channel efficiency | 1, 8, 10, 11 |
| Campaign effectiveness | 2, 3 |
| Customer value & loyalty | 4, 13, 14 |
| Product, payment, place & device | 5, 6, 7, 9, 12 |

---

### 1. Platform CTR & click-to-conversion rate
**Question:** Which advertising platform has the highest click-through rate and the highest click-to-order conversion rate?

```sql
SELECT
    a.platform,
    SUM(ap.impressions)  AS total_impressions,
    SUM(ap.clicks)       AS total_clicks,
    SUM(ap.conversions)  AS total_conversions,
    ROUND(SUM(ap.clicks)      / NULLIF(SUM(ap.impressions), 0) * 100, 2) AS ctr_pct,
    ROUND(SUM(ap.conversions) / NULLIF(SUM(ap.clicks), 0)      * 100, 2) AS click_to_conversion_pct
FROM advertisements a
JOIN advertisement_performance ap ON a.ad_id = ap.ad_id
GROUP BY a.platform
ORDER BY ctr_pct DESC;
```

**Technique:** conditional ratio metrics with `NULLIF` to prevent divide-by-zero. <br>
**Finding:** Instagram leads on CTR (6.02%), but WeChat converts clicks to orders best (24.98%). The best click-getter and the best closer are on different platforms.

---

### 2. Campaign ROI
**Question:** Which campaigns return the most per dollar of ad spend?

```sql
WITH spend AS (
    SELECT campaign_id, SUM(allocated_budget) AS total_ad_spend
    FROM advertisements
    GROUP BY campaign_id
),
revenue AS (
    SELECT a.campaign_id, SUM(ap.platform_ad_revenue) AS total_ad_revenue
    FROM advertisements a
    JOIN advertisement_performance ap ON a.ad_id = ap.ad_id
    GROUP BY a.campaign_id
)
SELECT
    c.campaign_name,
    s.total_ad_spend,
    COALESCE(r.total_ad_revenue, 0) AS total_ad_revenue,
    ROUND((COALESCE(r.total_ad_revenue, 0) - s.total_ad_spend)
          / NULLIF(s.total_ad_spend, 0) * 100, 2) AS roi_pct
FROM campaigns c
JOIN spend   s ON c.campaign_id = s.campaign_id
LEFT JOIN revenue r ON c.campaign_id = r.campaign_id
ORDER BY roi_pct DESC;
```

**Technique:** two CTEs combined, `LEFT JOIN` + `COALESCE` to keep campaigns with no revenue.<br>
**Finding:** Every campaign is profitable (ROI > 190%). Autumn Fashion Launch 2024 tops the list at 294.30%; the EOFY 2024 benchmark sits mid-pack at 234.91%.

---

### 3. Actual sales revenue from ad-influenced orders
**Question:** How many orders, and how much real revenue, did each campaign's ads actually drive?

```sql
WITH attributed AS (
    SELECT DISTINCT c.campaign_id, c.campaign_name, ao.order_id
    FROM campaigns c
    JOIN advertisements a      ON c.campaign_id = a.campaign_id
    JOIN advertisement_order ao ON a.ad_id = ao.ad_id
),
order_rev AS (
    SELECT op.order_id, SUM(op.quantity * op.gross_price) AS order_revenue
    FROM order_products op
    JOIN orders o ON op.order_id = o.order_id
    WHERE o.order_status = 'Completed'
    GROUP BY op.order_id
)
SELECT
    at.campaign_name,
    COUNT(DISTINCT at.order_id)        AS total_attributed_orders,
    ROUND(SUM(orv.order_revenue), 2)   AS total_actual_sales_revenue
FROM attributed at
JOIN order_rev orv ON at.order_id = orv.order_id
GROUP BY at.campaign_id, at.campaign_name
ORDER BY total_actual_sales_revenue DESC;
```

**Technique:** attribution via the `advertisement_order` bridge; `DISTINCT` in the CTE prevents fan-out double counting before aggregation.<br>
**Finding:** Measures *real* completed-order revenue (distinct from the platform-reported figure in Concern 2). EOFY Stocktake Sale 2025 ranks third overall at $145,021 from 201 attributed orders.

---

### 4. Customer Lifetime Value by loyalty tier
**Question:** Which loyalty tier generates the most value per customer?

```sql
WITH cust_rev AS (
    SELECT c.customer_id, c.loyalty_tier,
           COALESCE(SUM(op.quantity * op.gross_price), 0) AS revenue
    FROM customers c
    LEFT JOIN orders o         ON c.customer_id = o.customer_id
                              AND o.order_status = 'Completed'
    LEFT JOIN order_products op ON o.order_id = op.order_id
    GROUP BY c.customer_id, c.loyalty_tier
)
SELECT loyalty_tier,
       COUNT(*)               AS customer_count,
       ROUND(AVG(revenue), 2) AS avg_ltv
FROM cust_rev
GROUP BY loyalty_tier
ORDER BY avg_ltv DESC;
```

**Technique:** per-customer aggregation in a CTE, then a second aggregation to average across the tier (avoids inflating the mean).<br>
**Finding:** Gold members average $46,369 in value — 5.3× Bronze ($8,732) — despite numbering only 57 against 244 Bronze.

---

### 5. Most engaging device
**Question:** Which device drives the most ad interactions, and which holds attention longest?

```sql
SELECT
    device,
    COUNT(interaction_id)            AS total_interactions,
    ROUND(AVG(duration_seconds), 2)  AS avg_engagement_seconds
FROM interaction_events
GROUP BY device
ORDER BY avg_engagement_seconds DESC;
```

**Technique:** straightforward grouped aggregation contrasting volume against average duration.<br>
**Finding:** Phone delivers the most interactions (8,203) but the shortest dwell time (28.1s); Desktop is the reverse (61.4s). Mobile = reach, desktop = depth.

---

### 6. Most popular payment method
**Question:** Which payment methods are used most, and which carry the largest baskets?

```sql
SELECT
    o.payment_method,
    COUNT(DISTINCT o.order_id)     AS total_orders,
    COUNT(DISTINCT o.customer_id)  AS total_customers_used,
    ROUND(SUM(op.quantity * op.gross_price), 2) AS total_revenue,
    ROUND(SUM(op.quantity * op.gross_price)
          / NULLIF(COUNT(DISTINCT o.order_id), 0), 2) AS avg_order_value
FROM orders o
JOIN order_products op ON o.order_id = op.order_id
WHERE o.order_status = 'Completed'
GROUP BY o.payment_method
ORDER BY total_revenue DESC;
```

**Technique:** `COUNT(DISTINCT ...)` to count orders/customers correctly across a joined line-item table.<br>
**Finding:** Debit/Credit Card drives the most revenue ($3.29M), but the **David Jones Credit Card** has the highest average order value ($794 vs $750) — store-card holders spend more per visit.

---

### 7. Best-selling category
**Question:** Which product categories drive revenue, units and profit?

```sql
SELECT
    p.category,
    SUM(op.quantity)                                       AS total_units_sold,
    ROUND(SUM(op.quantity * op.gross_price), 2)            AS total_revenue,
    ROUND(SUM(op.quantity * (op.gross_price - p.unit_cost)), 2) AS gross_profit
FROM products p
JOIN order_products op ON p.product_id = op.product_id
JOIN orders o          ON op.order_id  = o.order_id
WHERE o.order_status = 'Completed'
GROUP BY p.category
ORDER BY total_revenue DESC;
```

**Technique:** profit calculated inline as `gross_price - unit_cost` weighted by quantity.<br>
**Finding:** Womenswear leads on revenue ($1.74M) and profit ($843k); Beauty sells the most units (5,533). Kids and Food & Wine trail.

---

### 8. Platform × device synergy
**Question:** Which specific platform-and-device combination converts impressions most efficiently?

```sql
SELECT
    a.platform,
    ap.device,
    SUM(ap.impressions)  AS total_impressions,
    SUM(ap.conversions)  AS total_conversions,
    ROUND(SUM(ap.conversions) / NULLIF(SUM(ap.impressions), 0) * 100, 2) AS conversion_efficiency_pct
FROM advertisements a
JOIN advertisement_performance ap ON a.ad_id = ap.ad_id
GROUP BY a.platform, ap.device
ORDER BY conversion_efficiency_pct DESC;
```

**Technique:** multi-dimension grouping (platform × device) to expose interaction effects hidden in platform-level totals.<br>
**Finding:** **Instagram-on-Phone** is the single best placement (0.91%). Instagram leads on every device; every X placement collapses to ~0.19–0.20%.

---

### 9. Highest revenue by location
**Question:** Which Australian cities contribute the most online revenue?

```sql
SELECT
    c.location,
    COUNT(DISTINCT o.order_id) AS total_orders,
    ROUND(SUM(op.quantity * op.gross_price), 2) AS total_revenue,
    ROUND(SUM(op.quantity * op.gross_price)
          / NULLIF(COUNT(DISTINCT o.order_id), 0), 2) AS avg_order_value
FROM customers c
JOIN orders o          ON c.customer_id = o.customer_id
JOIN order_products op ON o.order_id    = op.order_id
WHERE o.order_status = 'Completed'
GROUP BY c.location
ORDER BY total_revenue DESC;
```

**Technique:** three-table join with distinct-order counting to derive AOV per location.<br>
**Finding:** Sydney CBD is the top market ($860k from 1,177 orders). High-volume metros run lower baskets; smaller affluent areas (Gold Coast, Canberra) post the highest AOV (~$820).

---

### 10. Cost-per-Acquisition (CPA) by platform
**Question:** What does it cost each platform to win one converting customer?

```sql
WITH spend AS (
    SELECT platform, SUM(allocated_budget) AS total_spend
    FROM advertisements
    GROUP BY platform
),
conv AS (
    SELECT a.platform, SUM(ap.conversions) AS total_conversions
    FROM advertisements a
    JOIN advertisement_performance ap ON a.ad_id = ap.ad_id
    GROUP BY a.platform
)
SELECT s.platform, s.total_spend, c.total_conversions,
       ROUND(s.total_spend / NULLIF(c.total_conversions, 0), 2) AS cpa
FROM spend s
JOIN conv  c ON s.platform = c.platform
ORDER BY cpa ASC;
```

**Technique:** spend and conversions aggregated separately, then joined — avoids mixing two different grains in one pass.<br>
**Finding:** Instagram is cheapest at $1.99 per acquisition; X is the outlier at $11.91 — roughly 6× more expensive.

---

### 11. Ad ranking — CTR vs budget
**Question:** Are the highest-performing ads actually the best funded?

```sql
WITH ad_perf AS (
    SELECT a.ad_id, a.platform, a.allocated_budget,
           SUM(ap.impressions) AS impressions,
           SUM(ap.clicks)      AS clicks,
           ROUND(SUM(ap.clicks) / NULLIF(SUM(ap.impressions), 0) * 100, 2) AS ctr_pct
    FROM advertisements a
    JOIN advertisement_performance ap ON a.ad_id = ap.ad_id
    GROUP BY a.ad_id, a.platform, a.allocated_budget
)
SELECT ad_id, platform, allocated_budget, impressions, clicks, ctr_pct,
       RANK() OVER (ORDER BY ctr_pct DESC)         AS ctr_rank,
       RANK() OVER (ORDER BY allocated_budget DESC) AS budget_rank
FROM ad_perf
ORDER BY ctr_rank;
```

**Technique:** **window functions** (`RANK() OVER`) to rank the same rows on two independent dimensions side by side.<br>
**Finding:** Top-CTR ads are all Instagram, but funding doesn't follow performance — the best ad by CTR ranks only 147th by budget. A clear misallocation to fix.

---

### 12. Most & least profitable products by year
**Question:** Which products consistently lead or lag on profit, year over year?

```sql
WITH prod_year AS (
    SELECT p.product_id, p.product_name,
           DATE_FORMAT(o.order_date, '%Y') AS yr,
           SUM(op.quantity * op.gross_price) AS gross_revenue,
           SUM(op.quantity * p.unit_cost)    AS total_cost
    FROM order_products op
    JOIN orders   o ON op.order_id   = o.order_id
    JOIN products p ON op.product_id = p.product_id
    WHERE o.order_status = 'Completed'
    GROUP BY p.product_id, p.product_name, yr
),
line_disc AS (
    SELECT opd.product_id,
           DATE_FORMAT(o.order_date, '%Y') AS yr,
           SUM(opd.product_discount_amount) AS line_discount
    FROM order_product_discount opd
    JOIN orders o ON opd.order_id = o.order_id
    GROUP BY opd.product_id, yr
),
prod_profit AS (
    SELECT py.yr, py.product_name,
           py.gross_revenue - COALESCE(ld.line_discount, 0) AS net_revenue,
           py.gross_revenue - COALESCE(ld.line_discount, 0) - py.total_cost AS profit
    FROM prod_year py
    LEFT JOIN line_disc ld ON py.product_id = ld.product_id AND py.yr = ld.yr
),
ranked AS (
    SELECT yr, product_name,
           ROUND(net_revenue, 2) AS net_revenue,
           ROUND(profit, 2)      AS profit,
           RANK() OVER (PARTITION BY yr ORDER BY profit DESC) AS most_profitable_rank,
           RANK() OVER (PARTITION BY yr ORDER BY profit ASC)  AS least_profitable_rank
    FROM prod_profit
)
SELECT yr, product_name, net_revenue, profit,
       most_profitable_rank, least_profitable_rank,
       CASE WHEN most_profitable_rank <= 3 THEN 'Top 3'
            ELSE 'Bottom 3' END AS profit_segment
FROM ranked
WHERE most_profitable_rank <= 3
   OR least_profitable_rank <= 3
ORDER BY yr, most_profitable_rank;
```

**Technique:** four chained CTEs, `PARTITION BY yr` windowed ranking (top and bottom in one pass), and a `CASE` segment label. The discount CTE is wired in even though discounts are unpopulated, so the logic is production-ready.<br>
**Finding:** A stable set of winners recurs yearly — Dyson (Electronics), La Mer (Beauty), Camilla / Sass & Bide (Womenswear), Coach (Accessories). Persistent laggards: Seed Heritage / Polo (Kids) and Penfolds (Food & Wine).

---

### 13. Customer tenure by loyalty tier
**Question:** Is loyalty tier earned through longevity, or through spend?

```sql
SELECT
    c.loyalty_tier,
    COUNT(*) AS customers,
    ROUND(AVG(DATEDIFF(CURDATE(), c.signup_date)), 0) AS avg_days_with_brand,
    MIN(DATEDIFF(CURDATE(), c.signup_date))           AS min_days,
    MAX(DATEDIFF(CURDATE(), c.signup_date))           AS max_days
FROM customers c
GROUP BY c.loyalty_tier
ORDER BY avg_days_with_brand DESC;
```

**Technique:** `DATEDIFF` against `CURDATE()` to derive tenure; min/max to show the spread.
**Finding:** Tenure is essentially flat across tiers (Bronze 1,156 days, Gold 1,090), so tier is driven by **spend intensity, not time with the brand**.

---

### 14. All-time average daily profit per customer
**Question:** Who are the highest-value customers — and do longer-tenured ones spend more?

> **Note:** reconstructed from the output columns; replace with your exact query file.

```sql
WITH cust_profit AS (
    SELECT c.customer_id, c.customer_name, c.loyalty_tier,
           DATEDIFF(CURDATE(), c.signup_date) AS days_with_brand,
           ROUND(SUM(op.quantity * (op.gross_price - p.unit_cost)), 2) AS total_profit
    FROM customers c
    JOIN orders   o ON c.customer_id = o.customer_id
    JOIN order_products op ON o.order_id = op.order_id
    JOIN products p ON op.product_id = p.product_id
    WHERE o.order_status = 'Completed'
    GROUP BY c.customer_id, c.customer_name, c.loyalty_tier, days_with_brand
)
SELECT customer_id, customer_name, loyalty_tier, days_with_brand, total_profit,
       ROUND(total_profit / NULLIF(days_with_brand, 0), 2) AS avg_daily_profit
FROM cust_profit
ORDER BY avg_daily_profit DESC;
```

**Technique:** normalises total profit by tenure to measure spending *velocity* rather than cumulative spend.<br>
**Finding:** The fastest spenders are **recently-acquired Gold members** (e.g. ~$498/day over 66 days), while some long-tenured customers sit near $27/day. Longer tenure does **not** predict higher value — a key reframing for retention vs acquisition strategy.

---

## Headline findings

1. **Instagram is the acquisition engine**: best CTR (6.02%), lowest CPA ($1.99), best placement (Instagram/Phone). **X underperforms on every metric** and is the obvious channel to wind back.<br>
2. **WeChat is the best closer** (24.98% click-to-conversion) pair it with Instagram in a reach-then-retarget funnel.<br>
3. **Value concentrates in a small Gold cohort** ($46k LTV), and the highest-velocity spenders are *newly acquired* Gold members — tenure does not drive value.<br>
4. **Budget doesn't follow ad performance** — top-CTR creatives are underfunded.<br>
5. **Hero brands and categories recur** — Womenswear and Beauty lead; Camilla, Sass & Bide, La Mer, Dyson, and Coach are perennial profit drivers.<br>

Full recommendations for EOFY 2026 are in `report/Assignment_1_Report.docx`.

## Limitations

This is a **synthetic** dataset created for academic and skill-demonstration purposes. Relationships in the data (e.g. Gold spending more, Instagram outperforming X) are properties of the generation logic, not real-world evidence. Attribution is a simplified click-based model; revenue is reported two ways (platform-estimated vs actual completed-order) and the two are not expected to reconcile; discount tables exist in the schema but are unpopulated, so margins reflect undiscounted pricing. Marketing metrics exclude external factors such as seasonality, competitor activity and ad fatigue.

## Author

**Thanh Phong Phung** — Master of Business Analytics, Sydney.
Project for BUSA8090 (Data and Visualisation for Business).
