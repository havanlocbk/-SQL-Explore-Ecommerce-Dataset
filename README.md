# eCommerce Analytics: Customer Behavior & Conversion Funnel using SQL on BigQuery

*Applied SQL in Google BigQuery to create queries, enabling data extraction and analysis for business decision-making*
---
# eCommerce Analytics with SQL on BigQuery

![BigQuery](https://img.shields.io/badge/Google-BigQuery-blue?logo=google-cloud)
![SQL](https://img.shields.io/badge/Language-SQL-green)
![Analytics](https://img.shields.io/badge/Focus-Data%20Analytics-orange)

## 📑 Table of Contents

📑 Table of Contents  
 [📌 I. Introduction](#i-introduction)  
 [📂 II. Dataset](#ii-dataset)  
 [📜 III. Business Questions & SQL Analysis](#iii-business-questions-0sql-analysis)  
 [🛠 IV. Tools & Skills](#iv-tools--skills)  
 [📊 V. Results](#v-results)  
 [🚀 VI. Next Steps](#vi-next-steps)  


---

## 📌 I. Introduction

This project explores the **Google Analytics eCommerce dataset** using SQL on **Google BigQuery**.
The goal is to analyze **website performance, customer behavior, and revenue trends**, providing insights that can inform data-driven business decisions.

---

## 📂 II. Dataset

* **Source**: [Google Analytics Sample Dataset](https://console.cloud.google.com/marketplace/product/goog-public-data/google-analytics-sample)
* **Tables Used**: `bigquery-public-data.google_analytics_sample.ga_sessions_*`
* Covers **2017 eCommerce website sessions** including user traffic, transactions, and product activity.

## 📜 III. Business Questions & SQL Analysis
1) **How did traffic and transactions trend in Q1 2017?**
   
*Traffic KPIs (Visits/Pageviews/Transactions) — Jan–Mar 2017*

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
**Result:**

<img width="973" height="187" alt="image" src="https://github.com/user-attachments/assets/4dcf96f4-c7ea-4d02-b204-57f91fce883f" />


**🔎Insight:** Transactions grew steadily in Q1 2017, with ~40% increase from Jan → Mar.  

---
2) **Which traffic sources caused the highest bounce rates?**

*Bounce Rate by Traffic Source — Jul 2017*

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
**Result:**

<img width="1108" height="431" alt="image" src="https://github.com/user-attachments/assets/9c252eda-9716-47a6-b0c5-87b18d2339d2" />


**🔎Insight:** Several major sources had bounce rates >50%, signaling inefficient acquisition campaigns.

---
3) **Which channels drove the most revenue in June 2017?**

*Revenue By Traffic Source — Jun 2017*

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
**Result:**

<img width="1380" height="523" alt="image" src="https://github.com/user-attachments/assets/027c2ef0-5571-4212-b54e-3b2afad234ba" />
<img width="1378" height="524" alt="image" src="https://github.com/user-attachments/assets/232c62bd-776a-4fbc-9206-0638f189064a" />


**🔎Insight:** Direct traffic contributed ~97K and Google search ~18.7K, together driving the bulk of revenue. 

---
4) ** Do purchasers behave differently from non-purchasers?**

*User Behavior — Purchasers vs Non-Purchasers (Jun–Jul 2017)*
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
**Result:**

<img width="902" height="153" alt="image" src="https://github.com/user-attachments/assets/0b4803fe-2b8c-445d-b07b-083d4a94f2a6" />


**🔎Insight:** Purchasers had fewer pageviews than non-purchasers → more focused journeys to checkout. 

---
5) **How many transactions did each purchaser make on average?**

*Avg. Transactions per Purchasing User — Jul 2017*

```sql
-- Query 05
-- Option 1:
SELECT
  '201707' AS month,
  SAFE_DIVIDE(SUM(totals.transactions), COUNT(DISTINCT fullVisitorId)) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE totals.transactions >= 1
  AND product.productRevenue IS NOT NULL;

-- Option 2:
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
**Result:**

<img width="660" height="104" alt="image" src="https://github.com/user-attachments/assets/e7861a50-4a14-4183-a3b4-c07748cced8f" />


**🔎Insight:** On average, each purchasing user made just over **1 transaction** in July 2017.  

---
6) **How much did purchasers spend per session?**

*Avg. Spend per Session (Purchasers Only) — Jul 2017*

```sql
-- Option 1:
SELECT
  '201707' AS month,
  ROUND( SAFE_DIVIDE(SUM(product.productRevenue) / 1e6, SUM(totals.visits)), 2 ) AS avg_spend_per_session
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)  AS hits,
  UNNEST(hits.product) AS product
WHERE totals.transactions IS NOT NULL
  AND product.productRevenue IS NOT NULL;

-- Option 2
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
**Result:**

<img width="643" height="96" alt="image" src="https://github.com/user-attachments/assets/adc27727-ff52-4306-aefa-49eeaa5ec1b9" />


**🔎Insight:** Avg. spend per purchaser session in Jul 2017 ≈ **43.9 units**.  

---
7) **Which products were most often bought together?**

*Also-Bought Products (Anchor: “YouTube Men's Vintage Henley”) — Jul 2017*

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
**Result:**

<img width="679" height="530" alt="image" src="https://github.com/user-attachments/assets/ff35d432-44d0-45cb-8d75-ab2fbee4ef79" />


**🔎Insight:** Top cross-sell: **Google Sunglasses** and other YouTube apparel with Henley buyers.  

---
8) **What was the conversion funnel (Pageview → Add-to-Cart → Purchase) rate?**

*Funnel Cohort: Pageview → Add-to-Cart → Purchase — Jan–Mar 2017*

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
**Result:**

<img width="1356" height="197" alt="image" src="https://github.com/user-attachments/assets/ef73648d-7820-4d6e-a6c9-c96e8e61e89d" />


**🔎Insight:** Only **8–13%** of product views turned into purchases, revealing significant funnel drop-offs. 

---

## 🛠 IV. Tools & Skills

* **SQL**: Aggregations, filtering, cohort analysis, and funnel metrics.
* **Google BigQuery**: Querying large-scale datasets efficiently.
* **Data Analytics**: Extracting **KPIs** such as bounce rate, conversion rate, revenue attribution.

---

## 📊 V. Results

Key findings from the dataset:

* Steady growth in **visits and transactions** across Q1 2017 (+40% transactions).
* High **bounce rates** from several traffic sources, suggesting inefficient campaigns.
* Strong **revenue contribution** from **Direct** (\~97K) and **Google** (\~18.7K) traffic.
* Purchasers had far fewer pageviews than non-purchasers, showing more **focused browsing behavior**.
* Funnel analysis shows **drop-offs**: only \~8–13% of product views converted to purchases.
* Also-bought patterns highlight clear **cross-sell opportunities** (e.g., Google Sunglasses with YouTube apparel).
* Average spend per purchaser session in Jul 2017 was **\~43.86**.

---

## 🚀 VI. Next Steps

* Visualize KPIs and funnels using **Power BI / Tableau**.
* Build an **interactive dashboard** tracking revenue by source, bounce rate, and conversion funnel.
* Leverage **cross-sell recommendations** to boost average order value.
* Extend analysis with **Machine Learning** (e.g., customer segmentation, purchase prediction).

---

