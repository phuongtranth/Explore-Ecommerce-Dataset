# [SQL] Website Performance Analysis

## I. Introduction
In this project, I applied advanced SQL techniques, including sliding window and aggregation queries, using Google BigQuery to analyze e-commerce data. I evaluated product performance, sales trends, discount strategies, customer retention, and inventory management. These insights supported the Marketing and Sales teams in making strategic, data-driven decisions to improve business outcomes.
## II. Dataset Access
The e-commerce dataset is stored in a public Google BigQuery dataset. To access the dataset, follow these steps:
- Log in to your Google Cloud Platform account and create a new project.
- Navigate to the BigQuery console and select your newly created project.
- In the navigation panel, select "Add Data" and then "Search a project".
- Enter the project ID "bigquery-public-data.google_analytics_sample.ga_sessions" and click "Enter".
- Click on the "ga_sessions_" table to open it.
## III. Key Focus Areas
- **Product Performance Analysis**: Evaluated subcategory performance through sales metrics and year-over-year growth rates.
- **Geographic Sales Patterns**: Identified top-performing territories by order quantity across multiple years.
- **Discount Strategy Assessment**: Analyzed seasonal discount costs across product subcategories.
- **Customer Retention Analysis**: Calculated retention rates for successfully shipped orders using cohort analysis.
- **Inventory Management**: Examined stock level trends, month-over-month changes, and stock-to-sales ratios.
- **Order Status Monitoring**: Quantified pending orders and their value to assess fulfillment efficiency.
## IV. Exploring the Dataset
### Query 1: Calculate total visit, pageview, transaction for Jan, Feb and March 2017 
```sql
SELECT FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
       SUM(totals.visits) AS visits, 
       SUM(totals.pageviews) AS pageviews,   
       SUM(totals.transactions) AS transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix BETWEEN '0101' AND '0331'
GROUP BY month
ORDER BY month;
```
![image](https://github.com/user-attachments/assets/4479030a-1b3d-45a2-a9ec-91b9095e8268)

Q1 2017 shows consistent website traffic, with March experiencing a notable spike in transactions (993), indicating improved conversion rates or seasonal effects.

### Query 2: Bounce rate per traffic source in July 2017
```sql
SELECT
    trafficSource.source as source,
    sum(totals.visits) as total_visits,
    sum(totals.Bounces) as total_no_of_bounces,
    (sum(totals.Bounces)/sum(totals.visits))* 100 as bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY source
ORDER BY total_visits DESC;
```
![image](https://github.com/user-attachments/assets/abe64d53-90c9-4d60-a53e-688b1bbe86b0)

Google drives the most traffic but has a high bounce rate. YouTube and Facebook have the highest bounce rates, while direct traffic shows better engagement.
### Query 3: Revenue by traffic source by week, by month in June 2017
```sql
SELECT *
FROM
(SELECT 
    'Month' AS time_type,
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS time,
    trafficSource.source AS source,
    FORMAT('%.4f',SUM(product.productRevenue)/1000000) AS revenue
FROM 
    `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
WHERE 
    product.productRevenue IS NOT NULL
GROUP BY 
    time, 
    source

UNION ALL

SELECT 
    'Week' AS time_type,
    FORMAT_DATE('%Y%W', PARSE_DATE('%Y%m%d', date)) AS time,
    trafficSource.source AS source,
    FORMAT('%.4f',SUM(product.productRevenue)/1000000) AS revenue
FROM 
    `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST(hits) AS hits,
    UNNEST(hits.product) AS product 
WHERE 
    product.productRevenue IS NOT NULL
GROUP BY 
    time, 
    source)

ORDER BY CAST(revenue AS FLOAT64) DESC;
```
![image](https://github.com/user-attachments/assets/3c6b2d82-c3f1-4fe0-a47d-e24a2fb43cc9)

Direct traffic consistently generates the highest revenue across weeks and months. Google is the second-largest source. June shows strong overall performance, especially for direct traffic.
### Query 4: Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017
```sql
WITH purchase AS
              (SELECT FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS month,
                            SUM(totals.pageviews)/COUNT(DISTINCT fullVisitorId) AS avg_pageviews_purchase
              FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`, 
              UNNEST (hits) AS hits, 
              UNNEST (hits.product) AS product 
              WHERE _table_suffix BETWEEN '0601' AND '0731' 
                      AND totals.transactions >=1 
                      AND product.productRevenue IS NOT NULL
              GROUP BY month),
    nonpurchase AS
              (SELECT FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS month,
                            SUM(totals.pageviews)/COUNT(DISTINCT fullVisitorId) AS avg_pageviews_non_purchase
              FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`, 
              UNNEST (hits) AS hits, 
              UNNEST (hits.product) AS product 
              WHERE _table_suffix BETWEEN '0601' AND '0731' 
                    AND totals.transactions IS NULL
                    AND product.productRevenue IS NULL
              GROUP BY month)

SELECT purchase.month,
        avg_pageviews_purchase,
        avg_pageviews_non_purchase
FROM purchase
FULL JOIN nonpurchase
USING (month)
ORDER BY purchase.month;
```
![image](https://github.com/user-attachments/assets/b6a4e04d-3ba0-49a7-b465-371591c31380)

Non-purchasers exhibit significantly higher average pageviews compared to purchasers. Both groups increased pageviews from June to July, with purchasers showing a larger relative increase.
### Query 5: Average number of transactions per user that made a purchase in July 2017
```sql
SELECT  FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS Month,
        SUM(totals.transactions)/COUNT(DISTINCT fullVisitorId) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` ,
UNNEST (hits) AS hits,
UNNEST (hits.product) AS product
WHERE totals.transactions >=1
      AND product.productRevenue is not null
GROUP BY Month
ORDER BY Month;
```
![image](https://github.com/user-attachments/assets/67fa0047-b26b-4eed-972d-166bb15cdfc9)

In July 2017, users who made purchases completed an average of 4.16 transactions. This suggests moderate repeat buying behavior among customers within the month.
### Query 6: Average amount of money spent per session. Only include purchaser data in July 2017
```sql
SELECT  FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS Month,
        ROUND(SUM(product.productRevenue)/SUM(totals.visits)/1000000,2) AS avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` ,
UNNEST (hits) AS hits,
UNNEST (hits.product) AS product
WHERE totals.transactions IS NOT NULL
      AND product.productRevenue IS NOT NULL
GROUP BY Month
ORDER BY Month;
```
![image](https://github.com/user-attachments/assets/7d7cf97b-0735-495a-9f12-57b195b3e594)

In July 2017, purchasing users spent an average of $43.86 per session.
This metric indicates the typical transaction value, useful for understanding customer spending patterns and optimizing pricing strategies.
### Query 7: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017
```sql
SELECT product.v2ProductName AS other_purchased_products,
        SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` ,
    UNNEST (hits) AS hits,
    UNNEST (hits.product) AS product
WHERE product.productRevenue IS NOT NULL 
      AND product.v2ProductName =! "YouTube Men's Vintage Henley"
      AND  fullVisitorId IN(SELECT DISTINCT fullVisitorId
        FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` ,
        UNNEST (hits) AS hits,
        UNNEST (hits.product) AS product
        WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
              AND product.productRevenue IS NOT NULL)
GROUP BY product.v2ProductName 
ORDER BY quantity DESC;
```
![image](https://github.com/user-attachments/assets/7175c609-2ada-4682-a200-e6703399579f)

Customers who bought the YouTube Men's Vintage Henley also favored Google-branded items, especially Sunglasses. There's a strong preference for casual wear and accessories across Google, YouTube, and Android product lines.
### Query 8: Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017
```sql
WITH viewnum AS        
        (SELECT FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month, 
                COUNT(*) AS num_product_view
        FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
        WHERE _table_suffix BETWEEN '0101' AND '0331'   
            AND hits.eCommerceAction.action_type = '2'
        GROUP BY month
        ORDER BY month),
    addtocartnum AS        
        (SELECT FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month, 
                COUNT(*) AS num_addtocart
        FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
        WHERE _table_suffix BETWEEN '0101' AND '0331'   
            AND hits.eCommerceAction.action_type = '3'
        GROUP BY month
        ORDER BY month),
    purchasenum AS        
        (SELECT FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month, 
                COUNT(*) AS num_purchase
        FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
        WHERE _table_suffix BETWEEN '0101' AND '0331'   
            AND product.productRevenue IS NOT NULL
            AND hits.eCommerceAction.action_type = '6'
        GROUP BY month
        ORDER BY month)  
SELECT viewnum.month,
	    num_product_view,
        num_addtocart,	
        num_purchase,
        ROUND(100.0*num_addtocart/num_product_view,2) AS add_to_cart_rate,
        ROUND(100.0*num_purchase/num_product_view,2) AS purchase_rate
    
FROM viewnum 
LEFT JOIN  addtocartnum
USING(month)
LEFT JOIN  purchasenum
USING(month)
ORDER BY month;
```
![image](https://github.com/user-attachments/assets/52eecd05-98c1-45a0-bd1b-d574a9e3c7c5)

Product view to purchase conversion rates improved consistently from January to March 2017. March saw the highest engagement, with 37.29% add-to-cart rate and 12.64% purchase rate. This trend indicates increasing effectiveness in converting browsers to buyers, possibly due to improved marketing or user experience.
## V. Conclusion
Overall, this project utilized Google BigQuery to analyze an e-commerce dataset, extracting valuable insights across product performance, customer behavior, and sales patterns. Key findings include improving conversion rates, cross-selling opportunities among branded items, and geographic sales trends. The analysis revealed areas for optimization in marketing strategies, inventory management, and customer experience. By leveraging big data analytics, the project provides a foundation for data-driven decision-making, enabling the business to enhance product offerings, refine pricing strategies, and improve customer retention. These insights position the company to make informed strategic choices, potentially leading to improved performance and growth in the competitive e-commerce landscape.
