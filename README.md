# SQL-Portfolio-Project

This project demonstrates an end-to-end analytics pipeline using SQL Server and Power BI. It is based on a simulated retail sales dataset covering the years 2015 to 2017. The goal was to analyze customer behavior, segment value groups, calculate churn, and visualize actionable business insights through an interactive dashboard.

---

## Tools and Technologies

- Microsoft SQL Server (T-SQL)
- Power BI (Data Visualization)
- DAX (Data Analysis Expressions)
- Power Query

---

## Dataset Overview

The dataset includes:

- Yearly product sales tables: `Product_Sales_2015`, `Product_Sales_2016`, `Product_Sales_2017`
- Product reference table: `products`
- Returns: Product_Returns

---

## SQL Queries and Transformation Steps

### 1. Merging Raw Data

All three yearly sales tables were merged using `UNION` to create a unified table:

```sql
SELECT * INTO sales
FROM [Product_Sales_2015]
UNION
SELECT * FROM [Product_Sales_2016]
UNION
SELECT * FROM [Product_Sales_2017];
```

### 2. Net Revenue Calculation

A view was created to calculate the net revenue for each customer and product combination.

```sql
CREATE VIEW client_revenue AS
SELECT 
  CAST(SUM((s.OrderQuantity * p.productprice) - (s.OrderQuantity * p.productcost)) AS DECIMAL(19,2)) AS NetRevenue,
  productname, p.ProductKey, s.CustomerKey, s.orderdate, p.modelname
FROM sales AS s
LEFT JOIN products AS p ON s.ProductKey = p.ProductKey
GROUP BY productname, s.CustomerKey, s.orderdate, p.modelname, p.ProductKey;
```

### 3. Cohort Analysis

The cohort year for each customer was determined based on their first purchase year. This was used to analyze customer retention.

```sql
WITH cohort_year AS (
  SELECT DISTINCT
    YEAR(MIN(orderdate) OVER (PARTITION BY customerkey)) AS cohort_year,
    customerkey
  FROM sales
)
SELECT 
  cy.cohort_year,
  YEAR(cr.orderdate) AS Purchase_Year,
  COUNT(DISTINCT cr.customerkey) AS UniqueCustomer
INTO cohort_analysis
FROM client_revenue AS cr
LEFT JOIN cohort_year AS cy ON cr.CustomerKey = cy.CustomerKey
GROUP BY YEAR(cr.orderdate), cy.cohort_year;
```

### 4. Customer Segmentation

Customers were segmented into high, medium, and low value groups using the PERCENTILE_CONT() function.

```sql
WITH get_customer_revenue AS (
  SELECT modelname, customerkey, SUM(netrevenue) AS netrevenue
  FROM client_revenue
  GROUP BY modelname, customerkey
), 
customer_segment AS (
  SELECT 
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY netrevenue) OVER (PARTITION BY modelname) AS "25_percentile",
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY netrevenue) OVER (PARTITION BY modelname) AS "75_percentile",
    *
  FROM get_customer_revenue
),
segment_summary AS (
  SELECT 
    CASE 
      WHEN netrevenue < [25_percentile] THEN '1-Low Value Client'
      WHEN netrevenue < [75_percentile] THEN '2-Low Value Client'
      ELSE '3-High Value Client' 
    END AS customer_segment, *
  FROM customer_segment
)
SELECT customer_segment, SUM(netrevenue) AS netrevenue
INTO customer_segmentation
FROM segment_summary
GROUP BY customer_segment;
```

### 5. Churn Calculation

Churned customers were defined as those who had not made a purchase in the last six months. The most recent order date per customer was used to define their status.

```sql
WITH getlastpurchase AS (
  SELECT 
    ROW_NUMBER() OVER (PARTITION BY customerkey ORDER BY CAST(orderdate AS DATE) DESC) AS rn,
    *
  FROM client_revenue
)
SELECT *, 
  CASE 
    WHEN orderdate < DATEADD(MONTH, -6, '2017-06-30') THEN 'Churn'
    ELSE 'Active'
  END AS customer_status
INTO churndate
FROM getlastpurchase
WHERE rn = 1;
```

## Power BI Dashboard Overview

The Power BI dashboard was built using the transformed data outputs from SQL. It provides a clear and interactive view of customer behavior, revenue performance, and business health over the years 2015 to 2017.

![Alt Text](https://github.com/Sofiya-Banmala/SQL-Portfolio-Project/blob/main/SQL%20Portfolio.JPG)

### Dashboard Features

**Main Visualizations:**

- **Revenue Contribution by Customer Segment**  
  Pie chart visualizing the share of revenue from Low, Medium, and High Value Clients.
  
- **Customer Activity Status**  
  Donut chart showing the distribution of Active and Churned customers based on their most recent purchase.

-**Top 10 Most Returned Products**
  Bar chart ranked by return count, with tooltips showing return rates.

- **Cohort Retention by Year of Acquisition**  
  Stacked column chart displaying retention trends for customer cohorts (first purchase year vs. returning purchase years).

- **KPI Cards**  
  At the top of the report to summarize key metrics:
  - Total Revenue
  - Total Customers
  - Churn Rate


*Business Insights*

- High Value Clients generate over 76% of total revenue, despite being a smaller customer segment.
- Churn Rate is approximately 40%, revealing a major opportunity for retention improvement.
- Product Returns are highest for accessories like bottles and tubes, suggesting possible product quality or expectation issues.
- Cohort retention has improved over time, especially in 2017 — signaling better engagement or acquisition strategies.

