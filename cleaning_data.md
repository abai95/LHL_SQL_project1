## What issues will you address by cleaning the data?
- There should only be a single value per entry
- Data should be properly formatted
- Unecessary rows/columns should be removed
- Fill in missing values when appropriate
- Add primary and reference keys

## Queries:
Drop empty columns:

``` sql
ALTER TABLE all_sessions
DROP COLUMN product_refund_amount,
DROP COLUMN item_quantity,
DROP COLUMN item_revenue,
DROP COLUMN transaction_revenue,
DROP COLUMN search_keyword;
```

Divide price related columns by 1,000,000 and format:

``` sql
ALTER TABLE all_sessions
ALTER COLUMN product_price TYPE FLOAT,
ALTER COLUMN total_transaction_revenue TYPE FLOAT;

UPDATE all_sessions
SET product_price = ROUND((product_price / 1000000)::NUMERIC, 2)
WHERE product_price IS NOT NULL;

UPDATE all_sessions
SET total_transaction_revenue = ROUND((total_transaction_revenue / 
	1000000)::NUMERIC, 2)
WHERE total_transaction_revenue IS NOT NULL;
```

Change date to DATE data type:

``` sql
ALTER TABLE all_sessions
ALTER COLUMN session_date TYPE DATE USING (session_date::TEXT)::DATE;
```