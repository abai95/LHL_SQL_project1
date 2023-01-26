# What issues will you address by cleaning the data?
- There should only be a single value per entry
- Data should be properly formatted
- Unecessary rows/columns should be removed
- Fill in missing values when appropriate
- Add primary and reference keys

# Queries:

## Preliminary Data Cleaning

Drop empty columns. Also drop irrelevant columns:

``` sql
ALTER TABLE all_sessions
DROP COLUMN product_refund_amount,
DROP COLUMN item_quantity,
DROP COLUMN item_revenue,
DROP COLUMN transaction_revenue,
DROP COLUMN search_keyword;

ALTER TABLE analytics
DROP COLUMN user_id;

ALTER TABLE all_sessions
DROP COLUMN ecommerce_action_option,
DROP COLUMN ecommerce_action_step,
DROP COLUMN ecommerce_action_type,
DROP COLUMN page_path_level1,
DROP COLUMN page_title;
```

Divide price related columns by 1,000,000 and format:

``` sql
ALTER TABLE all_sessions
ALTER COLUMN product_price TYPE FLOAT,
ALTER COLUMN total_transaction_revenue TYPE FLOAT;

ALTER TABLE analytics
ALTER COLUMN unit_price TYPE FLOAT

UPDATE all_sessions
SET product_price = ROUND((product_price / 1000000)::NUMERIC, 2)
WHERE product_price IS NOT NULL;

UPDATE all_sessions
SET total_transaction_revenue = ROUND((total_transaction_revenue / 
	1000000)::NUMERIC, 2)
WHERE total_transaction_revenue IS NOT NULL;

UPDATE analytics
SET unit_price = ROUND((unit_price / 1000000)::NUMERIC, 2)
WHERE unit_price IS NOT NULL;
```

Change date to DATE data type:

``` sql
ALTER TABLE all_sessions
ALTER COLUMN session_date TYPE DATE USING (session_date::TEXT)::DATE;

ALTER TABLE analytics
ALTER COLUMN visit_date TYPE DATE USING (visit_date::TEXT)::DATE;
```

There is whitespace in some values for product_sku in all_sessions. I'll also run the same query to remove them for other applicable tables.

``` sql
UPDATE all_sessions
SET product_sku = REPLACE(product_sku, ' ', '');

UPDATE products
SET sku = REPLACE(sku, ' ', '');

UPDATE sales_by_sku
SET product_sku = REPLACE(product_sku, ' ', '');

UPDATE sales_report
SET product_sku = REPLACE(product_sku, ' ', '');
```

Perusing through every table, there already are only one single value per entry

# Fill in Missing Values

## Country/City
Check for missing countries or cities:

``` sql
SELECT country
FROM all_sessions
GROUP BY country
ORDER BY country;

SELECT city
FROM all_sessions
GROUP BY city
ORDER BY city;
```
There are no NULL values, but some entries are listed as "not available in demo dataset" or "(not set)". There is no reason to try and fill in values for cities.

``` sql
SELECT country,
	   city
FROM all_sessions
WHERE country = '(not set)'
GROUP BY country,
		 city
ORDER BY country,
		 city;
```

All entries with missing countries also don't have listed cities, so no values can be filled in.

## Currency Code
Check for missing values:

``` sql
SELECT currency_code as cc,
	   COUNT(*)
FROM all_sessions
GROUP BY cc
ORDER BY cc;
```

The only non NULL value is 'USD', so I filled the rest of the entries as such.

``` sql
UPDATE all_sessions
SET currency_code = 'USD'
WHERE currency_code IS NULL;
```

# Removing non-matching Rows

## Product SKUs
There are many product_sku values present in the all_sessions table that do not exist in the products, sales_by_sku, and sales_report tables, so I removed those rows. I also removed one more row whose product_sku vale does not exist in only the products table:

``` sql
CREATE TEMP TABLE temp_all_sessions AS (
	SELECT *
	FROM all_sessions
);

WITH all_sku AS (
	SELECT DISTINCT product_sku
	FROM products
	UNION
	SELECT DISTINCT product_sku
	FROM sales_by_sku
)

DELETE FROM temp_all_sessions
WHERE product_sku NOT IN (SELECT * FROM all_sku);

DELETE FROM temp_all_sessions
WHERE product_sku = '9184677';
```

After verifying the results through a temporary table, apply the changes to the actual table.

# Add Primary/Reference Keys
I added an autoincrementing analytics_id and session_id columns to the analytics and all_sessions tables through PgAdmin 4 menus and made them the primary key

I set the product_sku columns of the products, sales_report, and sales_by_sku tables as the primary key through PgAdmin 4 menus and also set the product_sku column for sales_report as a foreign key referencing both of the product_sku columns for products and sales_by_sku:

``` sql
ALTER TABLE sales_report
ADD CONSTRAINT fk_sales_report_sales_by_sku
FOREIGN KEY (product_sku)
REFERENCES sales_by_sku(product_sku);

ALTER TABLE sales_report
ADD CONSTRAINT fk_sales_report_products
FOREIGN KEY (product_sku)
REFERENCES products(product_sku);
```

I set product_sku in all_sessions as the foreign key referencing product_sku in products.

``` sql
ALTER TABLE all_sessions
ADD CONSTRAINT fk_all_sessions_products
FOREIGN KEY (product_sku)
REFERENCES products(product_sku);
```