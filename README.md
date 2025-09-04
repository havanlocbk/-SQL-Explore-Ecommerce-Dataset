# [SQL] eCommerce Analytics with SQL & BigQuery
Applied SQL in Google BigQuery to create queries, enabling data extraction and analysis for business decision-making
---
# eCommerce Analytics with SQL & BigQuery

![BigQuery](https://img.shields.io/badge/Google-BigQuery-blue?logo=google-cloud)
![SQL](https://img.shields.io/badge/Language-SQL-green)
![Analytics](https://img.shields.io/badge/Focus-Data%20Analytics-orange)

## 📑 Table of Contents

1. [Introduction](#introduction)
2. [Dataset](#dataset)
3. [Key Queries & Insights](#key-queries--insights)
4. [Tools & Skills](#tools--skills)
5. [Results](#results)
6. [Next Steps](#next-steps)

---

## 📌 Introduction

This project explores the **Google Analytics eCommerce dataset** using SQL on **Google BigQuery**.
The goal is to analyze **website performance, customer behavior, and revenue trends**, providing insights that can inform data-driven business decisions.

---

## 📂 Dataset

* **Source**: [Google Analytics Sample Dataset](https://console.cloud.google.com/marketplace/product/goog-public-data/google-analytics-sample)
* **Tables Used**: `bigquery-public-data.google_analytics_sample.ga_sessions_*`
* Covers **2017 eCommerce website sessions** including user traffic, transactions, and product activity.

---

## 🔍 Key Queries & Insights

1. **Traffic & Revenue Trends**

   * Calculated **visits, pageviews, transactions, and revenue** by month (Jan–Mar 2017).

2. **Bounce Rate Analysis**

   * Bounce rate by **traffic source** in July 2017.

3. **Revenue By Traffic Source*

   * Revenue contribution by **traffic source** (weekly & monthly).

4. **User Behavior**

   * Compared **pageviews of purchasers vs. non-purchasers** (June–July 2017).

5. **Average Number of Transactions per Session**

   * Average number of transactions per user that made a purchase in July 2017.

6. **Average Spend per Session**

   * Calculated the average amount of money spent per session for purchasers in July 2017.

7. **Also-Bought Products Analysis**

   * Identified **products purchased together** (e.g., with *YouTube Men's Vintage Henley*).

8. **Cohort Funnel**

   * Conversion funnel from **Pageview → Add to Cart → Purchase** (Jan–Mar 2017).
   * Calculated **add-to-cart rate** and **purchase rate**.
---
## 📜 SQL Queries
1) **Traffic KPIs (Visits/Pageviews/Transactions) — Jan–Mar 2017**

```sql

-- Query 01
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
  SUM(totals.visits)        AS total_visits,
  SUM(totals.pageviews)     AS total_pageviews,
  SUM(totals.transactions)  AS total_transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _TABLE_SUFFIX BETWEEN '0101' AND '0331'
GROUP BY month
ORDER BY month;
```

2) **Bounce Rate by Traffic Source — Jul 2017**
```sql
-- Query 02
SELECT
  trafficSource.source                    AS source,
  SUM(totals.visits)                      AS total_visit,
  SUM(totals.bounces)                     AS total_no_of_bounces,
  SAFE_DIVIDE(SUM(totals.bounces), SUM(totals.visits)) * 100 AS bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY source
ORDER BY total_visit DESC;
```
3) **Revenue By Traffic Source — Jun 2017**

```sql   
-- Query 03
(
  SELECT
    'Month' AS time_type,
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS time_period,
    trafficSource.source,
    SUM(product.productRevenue) / 1e6 AS revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE product.productRevenue IS NOT NULL
  GROUP BY time_type, time_period, trafficSource.source

  UNION ALL

  SELECT
    'Week' AS time_type,
    FORMAT_DATE('%Y%W', PARSE_DATE('%Y%m%d', date)) AS time_period,
    trafficSource.source,
    SUM(product.productRevenue) / 1e6 AS revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE product.productRevenue IS NOT NULL
  GROUP BY time_type, time_period, trafficSource.source
)
ORDER BY time_type, time_period, trafficSource.source;
```

4) **User Behavior — Purchasers vs Non-Purchasers (Jun–Jul 2017)**
```sql
-- Query 04
WITH table_purchaser AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    SUM(totals.pageviews) / COUNT(DISTINCT fullVisitorId) AS avg_pageviews_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '0601' AND '0731'
    AND product.productRevenue IS NOT NULL
    AND totals.transactions >= 1
  GROUP BY month
),
table_non_purchaser AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    SUM(totals.pageviews) / COUNT(DISTINCT fullVisitorId) AS avg_pageviews_nonpurchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '0601' AND '0731'
    AND totals.transactions IS NULL
  GROUP BY month
)
SELECT
  COALESCE(p.month, np.month)               AS month,
  p.avg_pageviews_purchase,
  np.avg_pageviews_nonpurchase
FROM table_purchaser p
FULL JOIN table_non_purchaser np ON p.month = np.month
ORDER BY month;
```

5) **Avg. Transactions per Purchasing User — Jul 2017**
```sql
-- Query 05 (ver 1: fixed month)
SELECT
  '201707' AS month,
  SAFE_DIVIDE(SUM(totals.transactions), COUNT(DISTINCT fullVisitorId)) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE totals.transactions >= 1
  AND product.productRevenue IS NOT NULL;

-- Query 05 (ver 2: flexible, group by month)
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
  SAFE_DIVIDE(SUM(totals.transactions), COUNT(DISTINCT fullVisitorId)) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE totals.transactions >= 1
  AND product.productRevenue IS NOT NULL
GROUP BY month;
```
6) **Avg. Spend per Session (Purchasers Only) — Jul 2017**
```sql
-- Query 06 (ver 1: fixed month)
SELECT
  '201707' AS month,
  ROUND( SAFE_DIVIDE(SUM(product.productRevenue) / 1e6, SUM(totals.visits)), 2 ) AS avg_spend_per_session
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE totals.transactions IS NOT NULL
  AND product.productRevenue IS NOT NULL;

-- Query 06 (ver 2, group by month)
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
  SAFE_DIVIDE(SUM(product.productRevenue), SUM(totals.visits)) / 1e6 AS avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE product.productRevenue IS NOT NULL
  AND totals.transactions >= 1
GROUP BY month;
```

7) **Also-Bought Products (Anchor: “YouTube Men's Vintage Henley”) — Jul 2017**
```sql
-- Query 07
WITH product_purchased_users AS (
  SELECT DISTINCT fullVisitorId
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
    AND product.productRevenue IS NOT NULL
    AND totals.transactions >= 1
)
SELECT
  product.v2ProductName AS other_purchased_products,
  SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE fullVisitorId IN (SELECT fullVisitorId FROM product_purchased_users)
  AND product.v2ProductName != "YouTube Men's Vintage Henley"
  AND product.productRevenue IS NOT NULL
GROUP BY other_purchased_products
ORDER BY quantity DESC;
```
8) **Funnel Cohort: Pageview → Add-to-Cart → Purchase — Jan–Mar 2017**

```sql
-- Query 08

--Option 1: single CTE
WITH CTE AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    COUNTIF(hits.eCommerceAction.action_type = '2') AS num_product_view,
    COUNTIF(hits.eCommerceAction.action_type = '3') AS num_addtocart,
    COUNTIF(hits.eCommerceAction.action_type = '6' AND product.productRevenue IS NOT NULL) AS num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '0101' AND '0331'
  GROUP BY month
)
SELECT
  month,
  num_product_view,
  num_addtocart,
  num_purchase,
  ROUND(SAFE_DIVIDE(num_addtocart, num_product_view) * 100, 2) AS add_to_cart_rate,
  ROUND(SAFE_DIVIDE(num_purchase,   num_product_view) * 100, 2) AS purchase_rate
FROM CTE
WHERE num_product_view > 0
ORDER BY month;
```

```sql
-- Option 2 (CTE breakdown)
WITH product_view AS (
  SELECT
    FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
    COUNT(product.productSKU) AS num_product_view
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
    AND hits.eCommerceAction.action_type = '2'
  GROUP BY 1
),
add_to_cart AS (
  SELECT
    FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
    COUNT(product.productSKU) AS num_addtocart
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
    AND hits.eCommerceAction.action_type = '3'
  GROUP BY 1
),
purchase AS (
  SELECT
    FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
    COUNT(product.productSKU) AS num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
    AND hits.eCommerceAction.action_type = '6'
    AND product.productRevenue IS NOT NULL
  GROUP BY 1
)
SELECT
  pv.month,
  pv.num_product_view,
  a.num_addtocart,
  p.num_purchase,
  ROUND(SAFE_DIVIDE(a.num_addtocart, pv.num_product_view) * 100, 2) AS add_to_cart_rate,
  ROUND(SAFE_DIVIDE(p.num_purchase,  pv.num_product_view) * 100, 2) AS purchase_rate
FROM product_view pv
LEFT JOIN add_to_cart a ON pv.month = a.month
LEFT JOIN purchase   p ON pv.month = p.month
ORDER BY pv.month;
```
```sql
-- Option 3 (COUNT CASE WHEN)
WITH product_data AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    COUNT(CASE WHEN hits.eCommerceAction.action_type = '2' THEN product.v2ProductName END) AS num_product_view,
    COUNT(CASE WHEN hits.eCommerceAction.action_type = '3' THEN product.v2ProductName END) AS num_add_to_cart,
    COUNT(CASE WHEN hits.eCommerceAction.action_type = '6' AND product.productRevenue IS NOT NULL THEN product.v2ProductName END) AS num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
    UNNEST(hits)  AS hits,
    UNNEST(hits.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
    AND hits.eCommerceAction.action_type IN ('2','3','6')
  GROUP BY month
)
SELECT
  month,
  num_product_view,
  num_add_to_cart,
  num_purchase,
  ROUND(SAFE_DIVIDE(num_add_to_cart, num_product_view) * 100, 2) AS add_to_cart_rate,
  ROUND(SAFE_DIVIDE(num_purchase,   num_product_view) * 100, 2) AS purchase_rate
FROM product_data
ORDER BY month;
```
---

## 🛠 Tools & Skills

* **SQL**: Aggregations, filtering, cohort analysis, and funnel metrics.
* **Google BigQuery**: Querying large-scale datasets efficiently.
* **Data Analytics**: Extracting **KPIs** such as bounce rate, conversion rate, revenue attribution.

---

## 📊 Results

Key findings from the dataset:

* Seasonal trends in **visits and revenue** (Q1 2017).
* High **bounce rates** in certain traffic sources indicating inefficient campaigns.
* Clear **drop-offs in conversion funnel**: only \~X% of product views → purchases.
* Purchasing patterns show opportunities for **cross-selling**.

*(screenshots of query results)*

---

## 🚀 Next Steps

* Visualize the results using **Power BI / Tableau**.
* Build an **interactive dashboard** showing conversion funnels, traffic breakdown, and product-level performance.
* Extend analysis with **Machine Learning** (e.g., customer segmentation, churn prediction).

---
