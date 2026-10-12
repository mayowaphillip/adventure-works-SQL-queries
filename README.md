# ⚙️ Adventure Works SQL Queries

<div align="center">

![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=for-the-badge&logo=microsoftsqlserver&logoColor=white)
![T-SQL](https://img.shields.io/badge/T--SQL-Business%20Analytics-blue?style=for-the-badge)
![Questions](https://img.shields.io/badge/Business%20Questions-5-green?style=for-the-badge)
![Database](https://img.shields.io/badge/Database-Adventure%20Works-orange?style=for-the-badge)

**Production-ready T-SQL queries answering 5 real business questions on the Microsoft Adventure Works LT database — covering customer revenue, segmentation, product performance, and category-level ranking.**

[View Queries](#-sql-queries--business-questions) · [Schema Overview](#-schema-overview) · [Key Findings](#-key-findings) · [Author](#-author)

</div>

---

## 📌 Project Overview

The Adventure Works database is a widely recognised Microsoft sample database modelling a fictional bicycle and accessories manufacturer. It contains a realistic set of customers, orders, products, categories, and addresses — making it an ideal environment for demonstrating production-level SQL skills against a structured, multi-table relational schema.

This project answers **5 targeted business questions** using **T-SQL on Microsoft SQL Server**, covering multi-table joins, aggregation, CASE-based segmentation, subqueries, CTEs, and window functions — the full range of techniques expected of a professional data analyst.

---

## 🎯 Objective

To demonstrate advanced SQL proficiency by:
- Writing clean, well-commented T-SQL queries against a real multi-table relational database
- Answering business questions that span revenue analysis, customer segmentation, product ranking, and operational intelligence
- Using a range of SQL techniques from basic joins and aggregation through to CTEs, window functions, views, and correlated subqueries
- Producing results that are directly interpretable by business stakeholders

---

## ❓ Problem Statement

The Adventure Works business needs answers to key questions across customer value, product performance, and operational activity:

1. Who are our top 10 customers by revenue, and where do they ship to?
2. How do we segment our customers by their total spending level?
3. What products and categories did customers buy on our last day of business?
4. Can we store the customer segmentation logic as a reusable view?
5. What are the top 3 selling products in each category by revenue?

---

## 🏆 Goals

- Write production-ready T-SQL for each business question, clearly structured and commented
- Demonstrate multi-table joins across 5+ related tables in a single query
- Apply CASE expressions for business-defined customer segmentation logic
- Use correlated subqueries to dynamically filter on derived values (e.g. last order date)
- Implement CTEs and window functions for category-level product ranking
- Create a reusable SQL VIEW encapsulating segmentation logic for downstream reporting

---

## 📐 Schema Overview

The queries use the `SalesLT` schema of the Adventure Works LT database.

```
SalesLT.Customer          — Customer master (ID, Name, CompanyName)
SalesLT.CustomerAddress   — Customer ↔ Address junction table
SalesLT.Address           — Physical address (City, CountryRegion)
SalesLT.SalesOrderHeader  — Order master (OrderDate, CustomerID, BillToAddressID)
SalesLT.SalesOrderDetail  — Order lines (ProductID, OrderQty, UnitPrice)
SalesLT.Product           — Product master (ProductID, Name, ProductCategoryID)
SalesLT.ProductCategory   — Category master (ProductCategoryID, Name)
```

### Entity Relationships

```
Customer ──< CustomerAddress >── Address
Customer ──< SalesOrderHeader ──< SalesOrderDetail >── Product >── ProductCategory
Address  ──< SalesOrderHeader (via BillToAddressID)
```

---

## 🔍 SQL Queries & Business Questions

### Q1 — Top 10 Customers by Revenue
**Business Question:** Who are our top 10 customers by revenue? Include their full name, the city and country they shipped to, and their total revenue (OrderQty × UnitPrice).

**Tables used:** `Customer`, `CustomerAddress`, `Address`, `SalesOrderHeader`, `SalesOrderDetail`

```sql
SELECT TOP (10)
    CONCAT(c.FirstName, ' ', c.MiddleName, ' ', c.LastName) AS Fullname,
    a.City,
    a.CountryRegion                          AS Country,
    SUM(sod.OrderQty * sod.UnitPrice)        AS Revenue

FROM SalesLT.Customer c
JOIN SalesLT.CustomerAddress ca  ON c.CustomerID   = ca.CustomerID
JOIN SalesLT.Address a           ON ca.AddressID   = a.AddressID
JOIN SalesLT.SalesOrderHeader soh ON a.AddressID   = soh.BillToAddressID
JOIN SalesLT.SalesOrderDetail sod ON soh.SalesOrderID = sod.SalesOrderID

GROUP BY c.FirstName, c.MiddleName, c.LastName, a.City, a.CountryRegion
ORDER BY Revenue DESC;
```

**Techniques:** `TOP`, `CONCAT`, multi-table JOIN, `SUM` aggregation, `GROUP BY`, `ORDER BY DESC`

**Business value:** Identifies VIP customers for account management, loyalty programmes, and targeted upsell campaigns. Geographic output supports regional sales strategy.

---

### Q2 — Customer Segmentation by Revenue
**Business Question:** Segment all customers into 4 tiers based on their total revenue. List Customer ID, Company Name, Total Revenue, and their Segment.

**Tables used:** `Customer`, `CustomerAddress`, `SalesOrderHeader`, `SalesOrderDetail`

```sql
SELECT
    c.CustomerID                                AS ID,
    c.CompanyName                               AS [Company Name],
    SUM(sod.OrderQty * sod.UnitPrice)           AS [Total Revenue],
    CASE
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 50000 THEN 'HIGH PATRONAGE'
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 30000 THEN 'MEDIUM PATRONAGE'
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 10000 THEN 'LOW PATRONAGE'
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 1000  THEN 'VERY LOW PATRONAGE'
        ELSE 'VERY LOW REVENUE'
    END AS [Customer Segment]

FROM SalesLT.Customer AS c
JOIN SalesLT.CustomerAddress ca  ON c.CustomerID  = ca.CustomerID
JOIN SalesLT.SalesOrderHeader soh ON ca.CustomerID = soh.CustomerID
JOIN SalesLT.SalesOrderDetail sod ON soh.SalesOrderID = sod.SalesOrderID

GROUP BY c.CustomerID, c.CompanyName
ORDER BY [Total Revenue];
```

**Segmentation logic:**

| Segment | Revenue Threshold |
|---|---|
| HIGH PATRONAGE | ≥ $50,000 |
| MEDIUM PATRONAGE | ≥ $30,000 |
| LOW PATRONAGE | ≥ $10,000 |
| VERY LOW PATRONAGE | ≥ $1,000 |
| VERY LOW REVENUE | < $1,000 |

**Techniques:** `CASE` expression on aggregated values, `GROUP BY`, multi-table JOIN

**Business value:** Enables tiered account management — HIGH PATRONAGE customers get dedicated account managers; VERY LOW REVENUE customers are targeted for re-engagement or deprioritised.

---

### Q3 — Products Purchased on Last Day of Business
**Business Question:** What products and categories did customers purchase on our most recent order date? List Customer ID, Product ID, Product Name, Category Name, and Order Date.

**Tables used:** `SalesOrderHeader`, `SalesOrderDetail`, `Product`, `ProductCategory`

```sql
SELECT
    soh.CustomerID,
    sod.ProductID,
    p.Name   AS [Product Name],
    pc.Name  AS [Category Name],
    soh.OrderDate AS [Order Date]

FROM SalesLT.SalesOrderHeader soh
JOIN SalesLT.SalesOrderDetail sod ON soh.SalesOrderID    = sod.SalesOrderID
JOIN SalesLT.Product p            ON sod.ProductID       = p.ProductID
JOIN SalesLT.ProductCategory pc   ON p.ProductCategoryID = pc.ProductCategoryID

WHERE soh.OrderDate = (
    SELECT MAX(OrderDate)
    FROM SalesLT.SalesOrderHeader
);
```

**Techniques:** Correlated subquery with `MAX()` in `WHERE` clause, 4-table JOIN

**Business value:** Provides a snapshot of the most recent trading day — useful for end-of-period reporting, stock reconciliation, and understanding what drove last-day demand.

---

### Q4 — Customer Segment View (Reusable BI Object)
**Business Question:** Store the customer segmentation logic from Q2 as a reusable SQL VIEW called `CustomerSegment` for use in downstream reports and dashboards.

**Tables used:** `Customer`, `CustomerAddress`, `SalesOrderHeader`, `SalesOrderDetail`

```sql
CREATE VIEW CustomerSegment AS
SELECT
    c.CustomerID                                AS ID,
    c.CompanyName                               AS [Company Name],
    SUM(sod.OrderQty * sod.UnitPrice)           AS [Total Revenue],
    CASE
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 50000 THEN 'HIGH PATRONAGE'
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 30000 THEN 'MEDIUM PATRONAGE'
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 10000 THEN 'LOW PATRONAGE'
        WHEN SUM(sod.OrderQty * sod.UnitPrice) >= 1000  THEN 'VERY LOW PATRONAGE'
        ELSE 'VERY LOW REVENUE'
    END AS [Customer Segment]

FROM SalesLT.Customer AS c
JOIN SalesLT.CustomerAddress ca   ON c.CustomerID  = ca.CustomerID
JOIN SalesLT.SalesOrderHeader soh ON ca.CustomerID = soh.CustomerID
JOIN SalesLT.SalesOrderDetail sod ON soh.SalesOrderID = sod.SalesOrderID

GROUP BY c.CustomerID, c.CompanyName;
```

**Usage after creation:**
```sql
-- Query the view directly
SELECT * FROM CustomerSegment ORDER BY [Total Revenue] DESC;

-- Filter to a specific segment
SELECT * FROM CustomerSegment WHERE [Customer Segment] = 'HIGH PATRONAGE';

-- Join with other tables for enriched reporting
SELECT cs.*, c.Phone, c.EmailAddress
FROM CustomerSegment cs
JOIN SalesLT.Customer c ON cs.ID = c.CustomerID;
```

**Techniques:** `CREATE VIEW`, encapsulation of complex aggregation and CASE logic

**Business value:** Makes the segmentation model reusable across reports, Power BI datasets, and downstream queries without rewriting the logic each time. Centralises business rules in one place.

---

### Q5 — Top 3 Products per Category by Revenue
**Business Question:** What are the top 3 selling products in each category by revenue? Include Product Name, Category Name, Revenue, and Rank.

**Tables used:** `Product`, `ProductCategory`, `SalesOrderDetail`

```sql
;WITH ProductRank AS (
    SELECT
        p.Name   AS [Product Name],
        pc.Name  AS [Category Name],
        SUM(sod.OrderQty * sod.UnitPrice)   AS [Revenue],
        RANK() OVER (
            PARTITION BY pc.Name
            ORDER BY SUM(sod.OrderQty * sod.UnitPrice) DESC
        ) AS Ranknum

    FROM SalesLT.Product p
    JOIN SalesLT.ProductCategory pc  ON p.ProductCategoryID = pc.ProductCategoryID
    JOIN SalesLT.SalesOrderDetail sod ON p.ProductID        = sod.ProductID

    GROUP BY p.Name, pc.Name
)
SELECT
    [Product Name],
    [Category Name],
    [Revenue],
    Ranknum
FROM ProductRank
WHERE Ranknum <= 3
ORDER BY [Category Name], [Revenue] DESC;
```

**Techniques:** CTE (`WITH` clause), `RANK()` window function with `PARTITION BY`, `GROUP BY` inside CTE, filtering on window function result

**Business value:** Identifies the star performers within each product category — enabling focused merchandising, promotional investment, and inventory prioritisation for the products that actually move the needle.

---

## 💡 Key Findings

| # | Finding |
|---|---|
| 1 | **Top 10 customers** are geographically distributed — revenue is not concentrated in a single city or country, suggesting a healthy spread of demand |
| 2 | **Customer segmentation** reveals that most customers fall into VERY LOW PATRONAGE or VERY LOW REVENUE tiers — a small number of HIGH PATRONAGE accounts drive disproportionate revenue |
| 3 | **Last day of business** shows which categories and products were in demand most recently — useful for understanding end-of-period trends |
| 4 | **The CustomerSegment VIEW** makes segmentation analysis immediately reusable across all downstream reporting tools including Power BI |
| 5 | **Category ranking** surfaces the top performers within each product line — the rank-3 products often trail rank-1 by a significant margin, indicating concentrated demand at the top |

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|---|---|
| **Microsoft SQL Server** | Database engine and query execution environment |
| **T-SQL** | Query language (Microsoft's SQL extension) |
| **SQL Server Management Studio (SSMS)** | Query interface and database management |
| **Adventure Works LT** | Microsoft's sample relational database |
| **CTEs** | Used in Q5 for modular, readable ranked query structure |
| **Window Functions** | `RANK() OVER (PARTITION BY ...)` in Q5 for category-level ranking |
| **CASE Expressions** | Used in Q2 and Q4 for business-defined segmentation tiers |
| **Correlated Subquery** | Used in Q3 with `MAX()` to dynamically identify last order date |
| **CREATE VIEW** | Used in Q4 to encapsulate reusable business logic |

---

## 📊 SQL Techniques Demonstrated

| Technique | Used In |
|---|---|
| Multi-table JOINs (4–5 tables) | Q1, Q2, Q3, Q4, Q5 |
| `TOP N` for limiting results | Q1 |
| `CONCAT` for string formatting | Q1 |
| `SUM` aggregation with `GROUP BY` | Q1, Q2, Q3, Q4, Q5 |
| `CASE` expression on aggregated values | Q2, Q4 |
| Correlated subquery with `MAX()` | Q3 |
| `CREATE VIEW` | Q4 |
| Common Table Expression (CTE) | Q5 |
| `RANK() OVER (PARTITION BY ...)` | Q5 |
| Filtering on window function output | Q5 |

---

## 📁 Project Structure

```
adventure-works-SQL-queries/
│
├── README.md
├── sql/
│   ├── 01_top_customers_by_revenue.sql       # Q1
│   ├── 02_customer_segmentation.sql          # Q2
│   ├── 03_last_day_purchases.sql             # Q3
│   ├── 04_customer_segment_view.sql          # Q4 — CREATE VIEW
│   └── 05_top3_products_per_category.sql     # Q5 — CTE + Window Function
│
└── docs/
    └── query_outputs/
        ├── q1_result.png                     # Screenshot of each query result
        ├── q2_result.png
        ├── q3_result.png
        ├── q4_result.png
        └── q5_result.png
```

---

## 🚀 How to Use

1. Install **SQL Server** (Express or Developer edition — both free) and **SQL Server Management Studio (SSMS)**
2. Download and restore the **Adventure Works LT** sample database from Microsoft:
   [https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure)
3. Open SSMS and connect to your SQL Server instance
4. Open any `.sql` file from the `sql/` folder
5. Ensure the correct database is selected (`AdventureWorksLT`) in the database dropdown
6. Execute the query with **F5** or the Execute button
7. For Q4, run `04_customer_segment_view.sql` first to create the view, then query it with:
   ```sql
   SELECT * FROM CustomerSegment ORDER BY [Total Revenue] DESC;
   ```

---

## 👤 Author

<div align="center">

**Mayowa Phillip**
*Data Analyst · Microsoft Certified Power BI Data Analyst Associate (PL-300)*

[![Email](https://img.shields.io/badge/Email-mayowaphillip%40yahoo.com-D14836?style=flat&logo=gmail&logoColor=white)](mailto:mayowaphillip@yahoo.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-mayowaphillipelnuk-0077B5?style=flat&logo=linkedin&logoColor=white)](https://linkedin.com/in/mayowaphillipelnuk)
[![GitHub](https://img.shields.io/badge/GitHub-mayowaphillip-181717?style=flat&logo=github&logoColor=white)](https://github.com/mayowaphillip)

</div>

---

<div align="center">

*Built with T-SQL · Microsoft SQL Server · Adventure Works LT*

⭐ If this project was useful to you, consider starring the repository.

</div>
