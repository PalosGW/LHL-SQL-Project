What are your risk areas? Identify and describe them.

Risk Areas and QA Components
	Risk Area 1: Data Integrity and Completeness
	Issue: Null or inconsistent data in critical fields such as country, city, or orderquantity.


	Risk Area 2: Duplicated Records
	Issue: Check for duplicate records in transactional data based on product_sku, country, city, and orderquantity.
	
	Risk Area 3: Outliers
	Issue: finding and accounting for outliers regarding time spent on site

# QA Process:

## Risk 1

---

```
WITH product_order_CTE AS (
    SELECT
        product_sku,
        sales_report.name,
        orderquantity,
        sales_report.totalordered
    FROM sales_by_sku
    JOIN sales_report USING (product_sku)
    JOIN products USING (product_sku)
)
SELECT 
    COUNT(*) AS total_records,
    SUM(CASE WHEN country IS NULL THEN 1 ELSE 0 END) AS null_country,
    SUM(CASE WHEN city IS NULL THEN 1 ELSE 0 END) AS null_city,
    SUM(CASE WHEN orderquantity IS NULL THEN 1 ELSE 0 END) AS null_orderquantity
FROM all_sessions
JOIN product_order_CTE USING (product_sku);
```
Logic: This query uses the product_order_CTE to join with all_sessions and checks for null values in the key fields, including orderquantity.

---

## Risk 2

---

```
WITH product_order_CTE AS (
    SELECT
        product_sku,
        sales_report.name,
        orderquantity,
        sales_report.totalordered
    FROM sales_by_sku
    JOIN sales_report USING (product_sku)
    JOIN products USING (product_sku)
)
SELECT 
    product_sku,
    country,
    city,
    orderquantity,
    COUNT(*) AS duplicate_count
FROM all_sessions
JOIN product_order_CTE USING (product_sku)
GROUP BY product_sku, country, city, orderquantity
HAVING COUNT(*) > 1;
```
Logic: This query joins product_order_CTE with all_sessions and identifies duplicate records based on the grouped dimensions.

---

## Risk 3

---

```
WITH userengagement_CTE AS (
    SELECT
        fullvisitorid,
        visitid,
        date,
        visitnumber,
        pageviews,
        bounces,
        visitstarttime,
        timeonsite,
        channelgrouping
    FROM analytics
    UNION ALL
    SELECT
        fullvisitorid,
        visitid,
        date,
        NULL AS visitnumber, -- Placeholder for visitnumber
        pageviews,
        NULL AS bounces, -- Placeholder for bounces
        NULL AS visitstarttime, -- Placeholder for visitstarttime
        timeonsite,
        channelgrouping
    FROM all_sessions
),

	userengagement_stats AS (
	    SELECT
	        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY timeonsite) AS q1,
	        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY timeonsite) AS q3
	    FROM userengagement_CTE
),
	userengagement_filtered_CTE AS (
	    SELECT 
	        u.*
	    FROM userengagement_CTE u
	    CROSS JOIN userengagement_stats s
	    WHERE timeonsite BETWEEN (s.q1 - 1.5 * (s.q3 - s.q1)) AND (s.q3 + 1.5 * (s.q3 - s.q1))
),
	deduplicated_CTE AS (
	    SELECT
	        *,
	        ROW_NUMBER() OVER (
	            PARTITION BY fullvisitorid, visitid, date -- Columns that define duplicates
	            ORDER BY visitstarttime DESC -- Prioritize rows to keep (e.g., latest visitstarttime)
	        ) AS row_num
	    FROM
	        userengagement_CTE
)
```
Logic: Removes the outliers for timeonsite

---