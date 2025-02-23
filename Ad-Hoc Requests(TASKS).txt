-----------------------TASK-1------------------------------
/*Croma India Product-wise Sales Report for Fiscal Year 2021

Report of individual product sales aggregated on a monthly basis at the product level for Croma India customers for FY 2021 */

#Function to get fiscal year
CREATE FUNCTION `get_fiscal_year` (calendar_date DATE)
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE fiscal_year INT;
    SET fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));
    RETURN fiscal_year;
END;

#Query to Retrieve Sales Data
SELECT s.date, s.product_code, p.product,
       p.variant, s.sold_quantity, g.gross_price,
       ROUND(s.sold_quantity * g.gross_price, 2) AS gross_price_total
FROM fact_sales_monthly s
JOIN dim_product p ON p.product_code = s.product_code
JOIN fact_gross_price g 
    ON g.product_code = s.product_code 
    AND g.fiscal_year = get_fiscal_year(s.date)
WHERE customer_code = 90002002 
    AND get_fiscal_year(date) = 2021
ORDER BY date ASC
LIMIT 1000000;

-----------------------TASK-2------------------------------
/*Nova  Monthly Gross Sales Summary

Aggregating monthly gross sales report for Nova customer*/
  
SELECT s.date, s.product_code, p.product, 
       p.variant, s.sold_quantity, g.gross_price, 
       ROUND(s.sold_quantity * g.gross_price, 2) AS gross_price_total
FROM fact_sales_monthly s
JOIN dim_product p 
    ON p.product_code = s.product_code
JOIN fact_gross_price g 
    ON g.product_code = s.product_code 
    AND g.fiscal_year = get_fiscal_year(s.date)
WHERE customer_code = 90002002 
      AND get_fiscal_year(date) = 2021
ORDER BY date ASC
LIMIT 1000000;

#Stored procedures
CREATE PROCEDURE `get_monthly_gross_sales_for_customer` 
(in_customer_codes TEXT)
BEGIN
    SELECT s.date, SUM(ROUND(s.sold_quantity * g.gross_price, 2)) AS monthly_sales
    FROM fact_sales_monthly s
    JOIN fact_gross_price g 
        ON g.fiscal_year = get_fiscal_year(s.date) 
        AND g.product_code = s.product_code
    WHERE FIND_IN_SET(s.customer_code, in_customer_codes) > 0
    GROUP BY date;
END;

-----------------------TASK-3------------------------------
/*Determine Market Badge Based on Sales Quantity

Created a stored procedure that can determine the market badge based on the following logic:
If total sold quantity > 5 million, that market is considered Gold, else it is Silver.

#Get market badge

CREATE PROCEDURE get_market_badge (
    IN in_market VARCHAR(45),
    IN in_fiscal_year YEAR,
    OUT out_badge VARCHAR(45)
)
BEGIN
    DECLARE qty INT DEFAULT 0;

    # Retrieve total sold quantity for a given market and fiscal year
    SELECT SUM(sold_quantity) INTO qty
    FROM fact_sales_monthly s
    JOIN dim_customer c 
        ON s.customer_code = c.customer_code
    WHERE get_fiscal_year(s.date) = in_fiscal_year
          AND c.market = in_market
    GROUP BY c.market;

    # Determine market badge
    IF qty > 500000 THEN
        SET out_badge = 'Gold';
    ELSE
        SET out_badge = 'Silver';
    END IF;
END;

-----------------------TASK-4------------------------------

/*report for FY=2021 for top 10 markets by % net sales.*/

WITH cte1 AS (
    SELECT 
        c.market, 
        ROUND(SUM(s.net_sales) / 1000000, 2) AS net_sales_mln
    FROM net_sales s
    JOIN dim_customer c ON s.customer_code = c.customer_code
    WHERE s.fiscal_year = 2021
    GROUP BY c.market
)
SELECT 
    market, 
    net_sales_mln, 
    ROUND((net_sales_mln * 100) / SUM(net_sales_mln) OVER(), 2) AS pct_ns
FROM cte1
ORDER BY net_sales_mln DESC
LIMIT 10;

-----------------------TASK-5------------------------------

/*Query to Get Top 3 Products per Division by Quantity Sold*/

WITH cte1 AS (
    SELECT 
        p.division,
        p.product,
        SUM(sold_quantity) AS total_qty
    FROM fact_sales_monthly s
    JOIN dim_product p
    ON p.product_code = s.product_code
    WHERE fiscal_year = 2021
    GROUP BY p.product, p.division
),
cte2 AS (
    SELECT *,
           DENSE_RANK() OVER (PARTITION BY division ORDER BY total_qty DESC) AS drnk
    FROM cte1
)
SELECT * FROM cte2 WHERE drnk <= 3;

#stored procedure
CREATE PROCEDURE `get_top_n_products_per_division_by_qty_sold`(
    IN in_fiscal_year INT,
    IN in_top_n INT
)
BEGIN
    WITH cte1 AS (
        SELECT 
            p.division,
            p.product,
            SUM(sold_quantity) AS total_qty
        FROM fact_sales_monthly s
        JOIN dim_product p
        ON p.product_code = s.product_code
        WHERE fiscal_year = in_fiscal_year
        GROUP BY p.product, p.division
    ),
    cte2 AS (
        SELECT *, 
               DENSE_RANK() OVER (PARTITION BY division ORDER BY total_qty DESC) AS drnk
        FROM cte1
    )
    SELECT * FROM cte2 WHERE drnk <= in_top_n;
END;






