# Question 1: Can you figure out what sentiment score represents and if it correlates to order quantities

SQL Queries:
-- Using this CTE as baseline
```
WITH product_order_CTE AS (
	SELECT
		product_sku,
		sales_report.name AS product_name,
		orderquantity,
		products.sentimentscore
	FROM sales_by_sku
	JOIN sales_report USING (product_sku)
	JOIN products USING (product_sku)
)
```
-- Identifying range sentimentscore falls within (0.9 to -0.6)
```
SELECT 
	MAX(sentimentscore),
	MIN(sentimentscore)
FROM product_order_CTE
```
-- Comparing different orderings for sentimentscore and orderquantities
```
SELECT
	orderquantity,
	sentimentscore
FROM product_order_CTE
WHERE orderquantity > 1000
ORDER BY sentimentscore DESC
LIMIT 10
```
```
SELECT
	orderquantity,
	sentimentscore
FROM product_order_CTE
WHERE orderquantity > 1000
ORDER BY orderquantity DESC
LIMIT 10
```
```
SELECT
	orderquantity,
	sentimentscore
FROM product_order_CTE
WHERE orderquantity > 1000
ORDER BY orderquantity
LIMIT 10
```
---

Answer:
	Making the assumption based on the practical range the values fall within the scale is (1 to -1). Further developing this assumption
	the sentimentscore most likely represents how satisfied a customer is with 1 representing %100 satisfaction and -1 representing 100%
	disatisfaction. There seems to be a slight to medium correlation with orderquantities, while the top 10 for each ordering in a
	descending manner, seem to differ slightly. There are no orders of over 1000 in quantity with a negative sentiment.
	
---


# Question 2: What can you tell me about user engagement?


SQL Queries:
-- checking if any channelgrouping values differ between all_sessions and analytics (social only in analytics)
```
SELECT channelgrouping
FROM all_sessions
WHERE channelgrouping NOT IN (SELECT channelgrouping FROM analytics)
UNION
SELECT channelgrouping
FROM analytics
WHERE channelgrouping NOT IN (SELECT channelgrouping FROM all_sessions);
```
-- Creating CTE with analytics userengagement columns with distinct rows
```
-- Creating CTE with analytics and all_sessions for userengagement with distinct rows
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
)
```
-- Checking if visitstarttime which is stored as an integer is a timestamp by comparing it with the date column
```
SELECT 
    visitstarttime,
	date,
    TO_TIMESTAMP(visitstarttime) AS visitstarttime_casted
FROM analytics
WHERE 
	EXTRACT(YEAR FROM TO_TIMESTAMP(visitstarttime))
	!= EXTRACT(YEAR FROM date)
```
--Calculating and removing outliers before doing analysis
```
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
)

-- looking at min/max and avg time spent on site
SELECT 
    AVG(timeonsite)/60 AS avg_time_spent_cleaned, -- casted on minutes
    MAX(timeonsite)/60 AS max_time_spent_cleaned, -- casted as minutes
    MIN(timeonsite) AS min_time_spent_cleaned
FROM userengagement_filtered_CTE;
```
-- Looking at avg pageviews and avg number of times users visited the site
```
SELECT
	AVG(visitnumber) AS avgnumvisits,
	AVG(pageviews) AS avgpagesopened
FROM userengagement_CTE
```

---

Answer:
	Assumptions, timeonsite is measured in seconds, visitstarttime is a timestamp (verified by matching with dat column). 
	After removing the outliers the avg user spent approximately, 8.4 minutes and the maximum amount of time someone 
	spent was 34 minutes approximately. The avgerage numbver of visits rounded is 2 and the avgerage pages opened is 15
	rounded. 
	
---


# Question 3: What can you tell me about channelgrouping? How do customers land on our site?

SQL Queries:
-- Using question two's CTE
-- Looking at the distinct groupings customers fall into
SELECT
	channelgrouping
FROM userengagement_CTE
GROUP BY channelgrouping

-- Finding range for most to least successful channel groupings after removing duplicates
```
WITH userengagement_CTE AS (
    SELECT DISTINCT
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
    SELECT DISTINCT
        fullvisitorid,
        visitid,
        date,
        NULL::INTEGER AS visitnumber, -- Cast NULL to match the INTEGER type
        pageviews,
        NULL::INTEGER AS bounces, -- Cast NULL to match the INTEGER type
        NULL::BIGINT AS visitstarttime, -- Cast NULL to match the BIGINT type
        timeonsite,
        channelgrouping
    FROM all_sessions
)
SELECT
    channelgrouping,
    COUNT(*) AS visit_count
FROM
    userengagement_CTE
GROUP BY
    channelgrouping
ORDER BY
    visit_count DESC;
```
-- rankings without duplicates removed
```
SELECT
    channelgrouping,
    COUNT(*) AS visit_count
FROM
    userengagement_CTE
GROUP BY
    channelgrouping
ORDER BY
    visit_count DESC;
```
---
Answer:	Channel grouping looks to be the funnel categories for how customers land on our website. Assumptions and rankigns for
		channelgrouping categories are as follows:
1. **Organic Search**, count 84,857  
   - Seems to be people who search for our site organically without paid ad streams influencing this result.

2. **Referral**, count 31,260  
   - People who have landed on our site after hearing about it from another or being referred by them.

3. **Direct**, count 28,776  
   - Seems to be people with our site bookmarked or who know the URL directly to jump straight to the site.

4. **Social**, count 11,810  
   - People who land on our site from our social media accounts or from paid ads on social media platforms.

5. **Paid Search**, count 5,203  
   - Seems to be paid ads on web browsers or paying to increase SEO efficiency to be higher in search algorithms.

6. **Affiliates**, count 1,964  
   - Affiliate marketers who we have paid or made deals with to direct traffic to our site.

7. **Display**, count 1,297  
   - Slightly unclear; the best guess is banner and video ads on other websites.

8. **Other**, count 7  
   - Visitors who land on the website but don't fall into the other 7 categories.
 
	Found an inordinate amount of duplicates using the 9 columns from userengagement_CTE, total rows dropped from 4+
	million to 165,000. Fortunately the rankings remained the same from most to least successful grouping channels
	regardless.
---