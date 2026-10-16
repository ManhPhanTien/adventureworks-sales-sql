# AdventureWorks Sales & Inventory Analysis with BigQuery


All 8 queries below have been executed and validated directly in Google BigQuery. You can view the live project (queries, execution, and saved results) here:

👉 **[Open in BigQuery Console](https://console.cloud.google.com/bigquery?ws=!1m7!1m6!12m5!1m3!1suni-sql-project-1!2sus-central1!3s52278554-753e-4a60-bf9e-de9c63bb4ddd!2e1)**

---

## 📁 Project Overview

| | |
|---|---|
| **Dataset** | [AdventureWorks 2019](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure) — Microsoft's sample dataset simulating a bicycle manufacturing & distribution company (Sales, Production, Purchasing, HR schemas) |
| **Tool** | Google BigQuery (Standard SQL) |
| **Time period analyzed** | 2011 – 2014 (varies by query, see below) |
| **Focus areas** | Sales performance by category, YoY growth, territory ranking, discount/promotion cost, customer cohort retention, inventory (stock) trends, stock-to-sales efficiency, purchase order backlog |
| **Techniques used** | Window functions (`LAG`, `RANK`, `MIN() OVER`), CTEs (`WITH`), multi-table `INNER`/`LEFT JOIN`, cohort analysis pattern, conditional/date functions (`EXTRACT`, `DATE_DIFF`, `FORMAT_DATE`), `IFNULL` null-handling, ratio & growth-rate calculations |

### Why this dataset?

AdventureWorks is one of the most widely used **relational, multi-table business datasets** for analytics practice — it mirrors a real OLTP schema (separate Sales, Production, and Purchasing schemas with proper foreign keys) rather than one flat table. Working with it means writing SQL the way it's actually written in a company's data warehouse: **joining across normalized tables**, not just filtering a single spreadsheet-like source.

---

## 🎯 Business Questions & Key Findings

### Query 01 — Units Sold, Sales Value & Order Count by Subcategory (Last 12 Months)

**Query:**
```sql
SELECT format_date('%b %Y', sale.ModifiedDate) as period
      ,subprd.Name as name
      ,SUM(OrderQty) as qty
      ,SUM(LineTotal) as total_sales
      ,COUNT(DISTINCT SalesOrderID) as order_cnt
FROM adventureworks2019.Production.Product as prd
INNER JOIN adventureworks2019.Production.ProductSubcategory as subprd
  ON CAST(subprd.ProductSubcategoryID as STRING) = prd.ProductSubcategoryID
INNER JOIN adventureworks2019.Sales.SalesOrderDetail as sale USING (ProductID)
WHERE (
  DATE_DIFF(DATE((SELECT MAX(ModifiedDate) FROM adventureworks2019.Sales.SalesOrderDetail))
  ,DATE(sale.ModifiedDate)
  ,MONTH) <= 12
)
GROUP BY 1,2
ORDER BY 1 DESC,2;
```

**Result:**

![Query 01 Result](images/Query_1.png)

---

### Query 02 — Top 3 Subcategories by YoY Growth Rate

**Query:**
```sql
WITH year_pre AS
  (SELECT subprd.Name as name
        ,EXTRACT(YEAR FROM sale.ModifiedDate) as year
        ,SUM(OrderQty) as qty
  FROM adventureworks2019.Production.Product as prd
  INNER JOIN adventureworks2019.Sales.SalesOrderDetail as sale
    USING (ProductID)
  INNER JOIN adventureworks2019.Production.ProductSubcategory as subprd
    ON CAST(subprd.ProductSubcategoryID as STRING) = prd.ProductSubcategoryID
  GROUP BY 1,2)

SELECT name
      ,qty
      ,pre_qty
      ,ROUND((qty - pre_qty) / pre_qty, 2) as qty_diff
FROM
  (SELECT *
        ,LAG(qty) OVER (PARTITION BY name ORDER BY year) as pre_qty
  FROM year_pre) as pre_out
ORDER BY 4 DESC
LIMIT 3;
```

**Result:**

![Query 02 Result](images/Query_2.png)

---

### Query 03 — Top 3 Territories by Order Quantity, Every Year

**Query:**
```sql
WITH year_rk AS
  (SELECT EXTRACT(YEAR FROM detail.ModifiedDate) as year
        ,head.TerritoryID as territoryID
        ,SUM(OrderQty) as order_cnt
  FROM adventureworks2019.Sales.SalesOrderHeader as head
  INNER JOIN adventureworks2019.Sales.SalesOrderDetail as detail
    USING (SalesOrderID)
  GROUP BY 1,2)

SELECT *
FROM
  (SELECT *
        ,RANK() OVER (PARTITION BY year ORDER BY order_cnt DESC) as rk
  FROM year_rk) as rk_out
WHERE rk IN (1,2,3)
ORDER BY 1 DESC, 4;
```

**Result:**

![Query 03 Result](images/Query_3.png)

---

### Query 04 — Seasonal Discount Cost by Subcategory

**Query:**
```sql
SELECT EXTRACT(YEAR FROM sale.ModifiedDate) as year
      ,subprd.Name as name
      ,SUM((UnitPrice * OrderQty) - LineTotal) as total_cost
FROM adventureworks2019.Production.Product as prd
INNER JOIN adventureworks2019.Production.ProductSubcategory as subprd
  ON CAST(subprd.ProductSubcategoryID as STRING) = prd.ProductSubcategoryID
INNER JOIN adventureworks2019.Sales.SalesOrderDetail as sale
  USING (ProductID)
INNER JOIN adventureworks2019.Sales.SpecialOffer as special
  USING (SpecialOfferID)
WHERE type LIKE '%Seasonal Discount%'
GROUP BY 1,2
ORDER BY 2,3;
```

**Result:**

![Query 04 Result](images/Query_4.png)

---

### Query 05 — Customer Retention Cohort Analysis (2014, Shipped Orders)

**Query:**
```sql
WITH diff_gr AS
  (SELECT *
        ,month - month_join as diff
  FROM
      (SELECT CustomerID
            ,MIN(EXTRACT(MONTH FROM ModifiedDate)) OVER (PARTITION BY CustomerID ORDER BY ModifiedDate) as month_join
            ,EXTRACT(MONTH FROM ModifiedDate) as month
      FROM adventureworks2019.Sales.SalesOrderHeader
      WHERE EXTRACT(YEAR FROM ModifiedDate) = 2014
            AND Status = 5) as month_diff)

SELECT month_join
      ,CONCAT('M', diff) as month_diff
      ,COUNT(DISTINCT CustomerID)
FROM diff_gr
GROUP BY month_join, month_diff
ORDER BY 1,2;
```

**Result:**

![Query 05 Result](images/Query_5.png)

---

### Query 06 — Stock Level Trend & MoM % Change (2011, All Products)

**Query:**
```sql
WITH stock_pre AS
  (SELECT prd.Name as name
        ,EXTRACT(MONTH FROM stock.ModifiedDate) as mth
        ,EXTRACT(YEAR FROM stock.ModifiedDate) as yr
        ,SUM(StockedQty) as stock_qty
  FROM adventureworks2019.Production.WorkOrder as stock
  INNER JOIN adventureworks2019.Production.Product as prd
    USING (ProductID)
  WHERE EXTRACT(YEAR FROM stock.ModifiedDate) = 2011
  GROUP BY 1,2,3)

SELECT *
      ,IFNULL(ROUND(100 * (stock_qty - stock_pre) / stock_pre, 1), 0) as diff
FROM
  (SELECT *
        ,LAG(stock_qty) OVER (PARTITION BY name ORDER BY mth) as stock_pre
  FROM stock_pre) as pre_out
ORDER BY 1, 2 DESC;
```

**Result:**

![Query 06 Result](images/Query_6.png)

---

### Query 07 — Stock-to-Sales Ratio (2011, by Product, by Month)

**Query:**
```sql
WITH stock_join AS
  (SELECT ProductID
          ,EXTRACT(MONTH FROM ModifiedDate) as mth
          ,SUM(StockedQty) as stock
    FROM adventureworks2019.Production.WorkOrder
    WHERE EXTRACT(YEAR FROM ModifiedDate) = 2011
    GROUP BY 1,2),

sale_join AS
  (SELECT ProductID
          ,EXTRACT(MONTH FROM ModifiedDate) as mth
          ,SUM(OrderQty) as sales
    FROM adventureworks2019.Sales.SalesOrderDetail
    WHERE EXTRACT(YEAR FROM ModifiedDate) = 2011
    GROUP BY 1,2)

SELECT sale_join.mth
      ,'2011' as yr
      ,prd.ProductID as productID
      ,prd.Name as name
      ,sales
      ,stock
      ,ROUND(stock / sales, 1) as ratio
FROM stock_join
LEFT JOIN sale_join
  ON sale_join.ProductID = stock_join.ProductID
  AND sale_join.mth = stock_join.mth
LEFT JOIN adventureworks2019.Production.Product as prd
  ON stock_join.ProductID = prd.ProductID
ORDER BY 1 DESC, 7 DESC;
```

**Result:**

![Query 07 Result](images/Query_7.png)

---

### Query 08 — Pending Purchase Orders (2014)

**Query:**
```sql
SELECT EXTRACT(YEAR FROM ModifiedDate) as year
      ,1 as status
      ,COUNT(DISTINCT PurchaseOrderID) as order_cnt
      ,SUM(TotalDue) as value
FROM adventureworks2019.Purchasing.PurchaseOrderHeader
WHERE Status = 1
      AND EXTRACT(YEAR FROM ModifiedDate) = 2014
GROUP BY 1,2;
```

**Result:**

![Query 08 Result](images/Query_8.png)

---

## 🔎 Final Conclusion & Recommendations

| # | Insight | Recommendation |
|---|---------|----------------|
| 1 | **Volume ≠ Value**: In Sep 2013, Helmets (480 orders) and Bottles and Cages (380 orders) had the most orders, but Bottles and Cages sold 803 units for only $4.7K ($5.8 per unit). Cranksets sold just 75 units in 20 orders yet brought in $14.0K ($186 per unit). | **Tailor Strategy to Each Product Role**: Use high-volume, low-price items (Helmets, Bottles and Cages) as add-ons and bundle them with bikes to lift order value. Promote high-value components such as Cranksets through targeted campaigns and reseller upselling. |
| 2 | **Frames Lead Growth**: Mountain Frames (+521%, 510 → 3,168 units), Socks (+421%, 523 → 2,724) and Road Frames (+389%, 1,137 → 5,564) had the highest year-over-year growth in units sold. Core frames are growing fastest, with Socks as a fast-growing low-cost accessory. | **Invest in High-Growth Lines**: Feature Mountain and Road Frames in marketing and plan their production from demand forecasts. Bundle Socks and other apparel with frame and bike purchases to keep accessory growth going. |
| 3 | **Stable Territory Ranking**: Territories 4, 6 and 1 held the top 3 spots by units ordered every year from 2011 to 2014, with Territory 4 always #1 and peaking at 26,682 units in 2013. In 2014, all three fell by 49–57% at the same time, which points to 2014 data covering only part of the year rather than a territory-specific problem. | **Protect Core Territories & Replicate Their Playbook**: Keep priority stock and sales resources for Territories 4, 6 and 1. Compare 2014 against the same months of 2013 before drawing conclusions, and apply what works in Territory 4 to lower-ranked territories. |
| 4 | **Repeat Purchases Peak at Month 3**: In 2014, only 2–4% of new customers bought again in the next month, but repeat purchases spiked in month 3 (12.1% of the January cohort, 13.0% of the February cohort). Meanwhile, seasonal discounts were applied only to Helmets, and their cost nearly doubled from $828 (2012) to $1,606 (2013). | **Time Promotions Around the Repurchase Cycle**: Send re-engagement emails and offers around month 3 after the first purchase, and add earlier touchpoints in months 1–2 to lift low early retention. Measure the sales uplift from Helmet seasonal discounts before expanding them to other subcategories. |
| 5 | **Uneven Inventory & Purchasing Backlog**: In Dec 2011, some HL Mountain Frames held up to 27 units of stock per unit sold (Black, 48: 27 in stock vs 1 sold), while finished bikes like Road-150 and Mountain-100 stayed at 1.1–1.5. Stock also swung sharply month to month (BB Ball Bearing +144% in July, -42% in December 2011). In 2014, 224 purchase orders worth $3.87M were still pending. | **Rebalance Inventory & Clear the PO Backlog**: Cut production of overstocked frames and clear excess stock through bundles or promotions. Move to demand-based production planning to smooth monthly swings, and review the 224 pending purchase orders, prioritizing items with low stock-to-sales ratios. |
