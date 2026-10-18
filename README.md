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
SELECT format_date('%b %Y',sale.ModifiedDate) as period
      ,subprd.Name as name
      ,sum(OrderQty) as qty 
      ,sum(LineTotal) as total_sales
      ,count(distinct SalesOrderID) as order_cnt 
FROM adventureworks2019.Production.Product as prd
INNER JOIN adventureworks2019.Production.ProductSubcategory as subprd 
  ON cast (subprd.ProductSubcategoryID as string) = prd.ProductSubcategoryID
INNER JOIN adventureworks2019.Sales.SalesOrderDetail as sale 
  USING (ProductID)
WHERE 
  date_diff(date ((select max(ModifiedDate) from adventureworks2019.Sales.SalesOrderDetail))
            ,date(sale.ModifiedDate)
            ,month) <12
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
        ,extract (year from sale.ModifiedDate) as yr
        ,sum(OrderQty) as qty
  FROM adventureworks2019.Production.Product as prd 
  INNER JOIN adventureworks2019.Sales.SalesOrderDetail as sale
    USING (ProductID)
  INNER JOIN adventureworks2019.Production.ProductSubcategory as subprd
    ON cast (subprd.ProductSubcategoryID as string) = prd.ProductSubcategoryID
  GROUP BY 1,2)

SELECT name 
      ,qty 
      ,pre_qty 
      ,round ((qty - pre_qty)/pre_qty, 2 ) as qty_diff
FROM 
  (select * 
        ,lag(qty) over (partition by name order by yr) as pre_qty
  from year_pre ) as pre_out
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
  (SELECT extract (year from detail.ModifiedDate) as yr 
        ,head.TerritoryID as territory_id 
        ,sum(OrderQty) as order_cnt
  FROM adventureworks2019.Sales.SalesOrderHeader as head
  INNER JOIN adventureworks2019.Sales.SalesOrderDetail as detail 
    USING (SalesOrderID)
  GROUP BY 1,2) 

SELECT *
FROM 
  (select *
        ,dense_rank() over (partition by yr order by order_cnt DESC) as rk
  from year_rk) as rk_out
WHERE rk in (1,2,3)
ORDER BY 1 DESC, 4;
```

**Result:**

![Query 03 Result](images/Query_3.png)

---

### Query 04 — Seasonal Discount Cost by Subcategory

**Query:**
```sql
SELECT extract (year from sale.ModifiedDate) as yr
      ,subprd.Name as name 
      ,round(
            sum((UnitPrice  * OrderQty) - LineTotal)
            ,2) as total_cost
FROM adventureworks2019.Production.Product as prd
INNER JOIN adventureworks2019.Production.ProductSubcategory as subprd
  ON cast (subprd.ProductSubcategoryID as string) = prd.ProductSubcategoryID
INNER JOIN adventureworks2019.Sales.SalesOrderDetail as sale
  USING (ProductID)
INNER JOIN adventureworks2019.Sales.SpecialOffer as special
  USING (SpecialOfferID)
WHERE type like '%Seasonal Discount%'
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
            ,min (extract (month from ModifiedDate)) OVER (partition by CustomerID order by ModifiedDate) as month_join
            ,extract (month from ModifiedDate) as month
      FROM adventureworks2019.Sales.SalesOrderHeader
      WHERE extract (year from ModifiedDate) = 2014
            and Status = 5) as join_out)

SELECT month_join 
      ,concat('M - ',diff) as month_diff
      ,count(distinct CustomerID) as customer_cnt   
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
        ,extract (month from stock.ModifiedDate) as mth
        ,extract (year from stock.ModifiedDate) as yr
        ,sum(StockedQty) as stock_qty
  FROM adventureworks2019.Production.WorkOrder as stock 
  INNER JOIN adventureworks2019.Production.Product as prd 
    USING (ProductID)
  WHERE extract (year from stock.ModifiedDate) = 2011 
  GROUP BY 1,2,3) 

SELECT *
      ,ifnull(round (100*(stock_qty - stock_pre)/stock_pre,1) 
              ,0) as diff
FROM 
  (SELECT *
        ,lag(stock_qty) over (partition by name order by mth) as stock_pre
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
          ,extract (month from ModifiedDate) as mth
          ,sum(StockedQty) as stock
    FROM adventureworks2019.Production.WorkOrder
    WHERE extract (year from ModifiedDate) = 2011
    GROUP BY 1,2)
  ,sale_join AS 
  (SELECT ProductID
          ,extract (month from ModifiedDate) as mth
          ,sum(OrderQty) as sales
    FROM adventureworks2019.Sales.SalesOrderDetail
    WHERE extract (year from ModifiedDate) = 2011
    GROUP BY 1,2)

SELECT stock_join.mth
      ,2011 as yr 
      ,prd.ProductID as product_id 
      ,prd.Name as name 
      ,sales
      ,stock
      ,round (stock/sales,2) as ratio
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
SELECT extract (year from ModifiedDate) as yr 
      ,1 as status
      ,count (distinct PurchaseOrderID) as order_cnt
      ,sum(TotalDue) as value 
FROM adventureworks2019.Purchasing.PurchaseOrderHeader
WHERE Status = 1 
      and extract (year from ModifiedDate) = 2014 
GROUP BY 1,2;
```

**Result:**

![Query 08 Result](images/Query_8.png)

---

## 🔎 Final Conclusion & Recommendations

## 🔎 Final Conclusion & Recommendations

| # | Insight | Recommendation |
|---|---------|----------------|
| 1 | **More Orders ≠ More Money**: In Sep 2013, Helmets (480 orders) and Bottles and Cages (380 orders) had the most orders. But Bottles and Cages sold 803 units for only $4.7K (about $5.8 per unit). Cranksets sold only 75 units in 20 orders but made $14.0K (about $186 per unit). | **Use Different Strategies for Cheap and Expensive Products**: Sell cheap, popular items (Helmets, Bottles and Cages) together with bikes so each order is worth more. Promote expensive parts like Cranksets with special campaigns, and ask resellers to suggest them to customers. |
| 2 | **Frames Grow the Fastest**: Mountain Frames (+521%, 510 → 3,168 units), Socks (+421%, 523 → 2,724) and Road Frames (+389%, 1,137 → 5,564) grew the most compared with the previous year. Frames, the main products, are growing fastest, and Socks are a cheap item that is also growing quickly. | **Focus on Fast-Growing Products**: Promote Mountain and Road Frames more, and plan how many to produce based on expected demand. Sell Socks and other clothing together with frames and bikes. |
| 3 | **Top Regions Stay the Same**: Territories 4, 6 and 1 were the top 3 regions by units ordered every year from 2011 to 2014. Territory 4 was always #1, with its highest number in 2013 (26,682 units). In 2014, all three dropped by 49–57% at the same time. This is likely because the 2014 data covers only part of the year, not because of a problem in one region. | **Protect the Best Regions & Copy What Works**: Keep giving Territories 4, 6 and 1 priority for stock and sales staff. Compare 2014 with the same months in 2013 before making conclusions. Use what works in Territory 4 in weaker regions. |
| 4 | **Customers Often Return After 3 Months**: In 2014, only 2–4% of new customers bought again in the next month. But many more came back in the third month: 12.1% of customers who first bought in January, and 13.0% of those who first bought in February. Also, seasonal discounts were only given on Helmets, and the cost of these discounts almost doubled, from $828 (2012) to $1,606 (2013). | **Plan Promotions Around When Customers Return**: Send reminder emails and special offers about 3 months after a customer's first purchase. Also contact customers in months 1 and 2 so more of them come back early. Check whether Helmet discounts really increase sales before using discounts on other products. |
| 5 | **Too Much Stock & Many Waiting Orders**: In Dec 2011, some HL Mountain Frames had up to 27 units in stock for every unit sold (Black, 48: 27 in stock vs 1 sold). Finished bikes like Road-150 and Mountain-100 had only 1.1–1.5 units in stock per unit sold. Stock levels also changed a lot from month to month (BB Ball Bearing +144% in July, -42% in December 2011). In 2014, 224 purchase orders worth $3.87M were still waiting for approval. | **Balance Stock & Handle Waiting Orders**: Make fewer of the frames that have too much stock, and sell the extra stock in sets or with discounts. Plan production based on real demand so stock does not go up and down so much. Review the 224 waiting purchase orders, and handle orders first for products that are running low. |
