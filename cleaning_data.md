Issues and Queries:
Below, provide the SQL queries you used to clean your data.

## Q2 Starting Questions
-- to answer Q2 of starting questions cleaning the all_sessions table by updating (not set)
-- and 'not available in demo dataset' values to null
```
UPDATE all_sessions
SET 
    country = CASE 
                WHEN country IN ('(not set)', 'not available in demo dataset') THEN NULL 
                ELSE country 
              END,
    city = CASE 
             WHEN city IN ('(not set)', 'not available in demo dataset') THEN NULL 
             ELSE city 
           END;
```
-- inspecting the duplicate values which might be affecting the top results for the avg_order_quantities
```
WITH product_order_CTE AS (
	SELECT DISTINCT
		product_sku,
		sales_report.name,
		orderquantity,
		sales_report.totalordered
	FROM sales_by_sku
	JOIN sales_report USING (product_sku)
	JOIN products USING (product_sku)
)

SELECT
    country,
    city,
    AVG(orderquantity) AS avg_order_quantity,
    MIN(orderquantity) AS min_order_quantity,
    MAX(orderquantity) AS max_order_quantity,
    COUNT(*) AS row_count
FROM all_sessions
JOIN product_order_CTE USING (product_sku)
GROUP BY country, city
ORDER BY avg_order_quantity DESC, country, city;
```
-- Looking into row 7 from highest avg_order_quantities to see the two distinct values accounting for avg
```
SELECT
	all_sessions.*
    orderquantity
FROM all_sessions
JOIN product_order_CTE USING (product_sku)
WHERE country = 'United States'
AND city = 'Bellflower'
ORDER BY country, city;
```-- Deemed irrelevant as both quantities are the same and does not affect average


## For Question 3 & 4 Starting Questions

-- For questions 3 and 4 updating table to have null values for anything (not set) or not available
```
UPDATE all_sessions
SET 
    v2productname = CASE 
                      WHEN LOWER(v2productname) IN ('(not set)', 'not available in demo dataset') THEN NULL 
                      ELSE v2productname 
                    END;

UPDATE products
SET 
    name = CASE 
             WHEN LOWER(name) IN ('(not set)', 'not available in demo dataset') THEN NULL 
             ELSE name 
           END;
```

## Question 2 for Starting Data

-- Cleaning for question 2 starting_with_data, after verifying visitstarttime is a timestamp. comparig it with date column
```
SELECT 
    visitstarttime,
    date,
    TO_TIMESTAMP(visitstarttime) AS visitstarttime_casted
FROM analytics
WHERE 
    EXTRACT(YEAR FROM TO_TIMESTAMP(visitstarttime)) != EXTRACT(YEAR FROM date)
    OR EXTRACT(MONTH FROM TO_TIMESTAMP(visitstarttime)) != EXTRACT(MONTH FROM date)
    OR EXTRACT(DAY FROM TO_TIMESTAMP(visitstarttime)) != EXTRACT(DAY FROM date);
``` --Looks like 45000+ rows don't match although looks like most are only off by a day.

-- Verifying to see if any are off by more than a day
```
SELECT 
    visitstarttime,
    date,
    TO_TIMESTAMP(visitstarttime) AS visitstarttime_casted
FROM analytics
WHERE 
    ABS(DATE_PART('day', TO_TIMESTAMP(visitstarttime) - date)) > 1
    OR ABS(DATE_PART('month', TO_TIMESTAMP(visitstarttime) - date)) > 0;
```
-- seems none are off by more than 1 day, deemed acceptable


## Question 3 for Starting Data

When trying to answer question 3 from start_with_data, found an inordinate amount of duplicates
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
        NULL::INTEGER AS visitnumber,
        pageviews,
        NULL::INTEGER AS bounces,
        NULL::BIGINT AS visitstarttime,
        timeonsite,
        channelgrouping
    FROM all_sessions
)
SELECT
    fullvisitorid,
    visitid,
    date,
    COUNT(*) AS duplicate_count
FROM
    userengagement_CTE
GROUP BY
    fullvisitorid, visitid, date, visitnumber, pageviews, bounces, visitstarttime, timeonsite, channelgrouping
HAVING
    COUNT(*) > 1
ORDER BY
    duplicate_count DESC;
```
