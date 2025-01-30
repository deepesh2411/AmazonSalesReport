SELECT
  *
FROM
  "CASESTUDY"."PUBLIC"."AMAZONSALE"
LIMIT
  10;


-- Create Fact and dimention table

CREATE OR REPLACE TABLE "CASESTUDY"."WAREHOUSE"."Dim_Product" (
    product_key INTEGER AUTOINCREMENT PRIMARY KEY,
    style VARCHAR(100),
    category VARCHAR(100),
    sku VARCHAR(100),
    size VARCHAR(50),
    asin VARCHAR(50)
);

CREATE OR REPLACE TABLE "CASESTUDY"."WAREHOUSE"."Dim_Location" (
    location_key INTEGER AUTOINCREMENT PRIMARY KEY,
    ship_city VARCHAR(100),
    ship_state VARCHAR(100),
    ship_country VARCHAR(50),
    ship_postal_code DECIMAL(10, 0)
);


CREATE OR REPLACE TABLE "CASESTUDY"."WAREHOUSE"."Dim_Date" (
    date_key INTEGER AUTOINCREMENT PRIMARY KEY,
    date DATE,
    day INT,
    month INT,
    year INT,
    quarter INT,
    season VARCHAR(50)
);

CREATE OR REPLACE TABLE "CASESTUDY"."WAREHOUSE"."Dim_Sales_Channel" (
    sales_channel_key INTEGER AUTOINCREMENT PRIMARY KEY,
    sales_channel VARCHAR(50)
);

CREATE OR REPLACE TABLE "CASESTUDY"."WAREHOUSE"."Fact_Sales" (
    sales_id INTEGER AUTOINCREMENT PRIMARY KEY,
    order_id VARCHAR(255),
    product_key INT REFERENCES "CASESTUDY"."WAREHOUSE"."Dim_Product"(product_key),
    location_key INT REFERENCES "CASESTUDY"."WAREHOUSE"."Dim_Location"(location_key),
    date_key INT REFERENCES "CASESTUDY"."WAREHOUSE"."Dim_Date"(date_key),
    sales_channel_key INT REFERENCES "CASESTUDY"."WAREHOUSE"."Dim_Sales_Channel"(sales_channel_key),
    qty INT,
    amount DECIMAL(38, 2)
);


SELECT
  *
FROM
  "CASESTUDY"."PUBLIC"."AMAZONSALE"
LIMIT
  10;

-- Populate Dimension table Dim_Product
INSERT INTO "CASESTUDY"."WAREHOUSE"."Dim_Product" (style, category, sku, size, asin)
SELECT DISTINCT "STYLE", "CATEGORY", "SKU", "SIZE", "ASIN"
FROM "CASESTUDY"."PUBLIC"."AMAZONSALE"

-- Populate Dimension table Dim_Location
INSERT INTO "CASESTUDY"."WAREHOUSE"."Dim_Location" (ship_city, ship_state, ship_country, ship_postal_code)
SELECT DISTINCT "ship-city", "ship-state", "ship-country", "ship-postal-code"
FROM "CASESTUDY"."PUBLIC"."AMAZONSALE"
where "ship-city" is not NULL

-- Populate Dimension table Dim_Sales_Channel
INSERT INTO "CASESTUDY"."WAREHOUSE"."Dim_Sales_Channel" (sales_channel)
SELECT DISTINCT "sales channel"
FROM "CASESTUDY"."PUBLIC"."AMAZONSALE";

-- Update date format
UPDATE "CASESTUDY"."PUBLIC"."AMAZONSALE"
SET Date = CASE
              WHEN LENGTH(Date) = 8 THEN TO_DATE(Date, 'MM-dd-yy')   -- For "MM-dd-yy"
              ELSE TO_DATE(Date, 'MM-dd-yyyy')                       -- For "MM-dd-yyyy"
           END
WHERE Date IS NOT NULL;

-- Populate Dimension table Dim_Date
INSERT INTO "CASESTUDY"."WAREHOUSE"."Dim_Date" (date, day, month, year, quarter, season)
SELECT DISTINCT
    TO_DATE(DATE, 'YYYY-mm-dd') AS Date,
    EXTRACT(DAY FROM TO_DATE(DATE, 'YYYY-mm-dd')) AS day,
    EXTRACT(MONTH FROM TO_DATE(DATE, 'YYYY-mm-dd')) AS month,
    EXTRACT(YEAR FROM TO_DATE(DATE, 'YYYY-mm-dd')) AS year,
    EXTRACT(QUARTER FROM TO_DATE(DATE, 'YYYY-mm-dd')) AS quarter,
    CASE 
        WHEN EXTRACT(MONTH FROM TO_DATE(DATE, 'YYYY-mm-dd')) IN (12, 1, 2) THEN 'Winter'
        WHEN EXTRACT(MONTH FROM TO_DATE(DATE, 'YYYY-mm-dd')) IN (3, 4) THEN 'Spring'
        WHEN EXTRACT(MONTH FROM TO_DATE(DATE, 'YYYY-mm-dd')) IN (5, 6, 7) THEN 'Summer'
        WHEN EXTRACT(MONTH FROM TO_DATE(DATE, 'YYYY-mm-dd')) IN (8, 9) THEN 'Rainy'
        WHEN EXTRACT(MONTH FROM TO_DATE(DATE, 'YYYY-mm-dd')) IN (10, 11) THEN 'Autumn'
    END AS season
FROM "CASESTUDY"."PUBLIC"."AMAZONSALE";

SELECT TO_DATE(DATE, 'mm-dd-YY') AS converted_date
FROM "CASESTUDY"."PUBLIC"."AMAZONSALE"
LIMIT 10


SELECT
  *
FROM
  "CASESTUDY"."WAREHOUSE"."Dim_Product"
LIMIT
  10;


--Populate Fact table Fact_Sales:
INSERT INTO "CASESTUDY"."WAREHOUSE"."Fact_Sales" (order_id, product_key, location_key, date_key, sales_channel_key, qty, amount)
SELECT 
    asr."order id", 
    p.product_key,
    l.location_key, 
    d.date_key, 
    sc.sales_channel_key,
    asr."QTY",
    asr."AMOUNT"
FROM "CASESTUDY"."PUBLIC"."AMAZONSALE" AS asr
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Product" AS p 
    ON asr."STYLE" = p.style AND asr."SKU" = p.sku
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Location" AS l 
    ON asr."ship-city" = l.ship_city AND asr."ship-postal-code" = l.ship_postal_code
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Date" AS d 
    ON asr."DATE" = d.date
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Sales_Channel" AS sc 
    ON asr."sales channel" = sc.sales_channel;

--Normal checks
select * from "CASESTUDY"."PUBLIC"."AMAZONSALE"
limit 10

ALTER TABLE "CASESTUDY"."PUBLIC"."AMAZONSALE"
DROP COLUMN "unnamed: 22";

select ship_country, ship_city, ship_state, ship_postal_code, location_key from "Dim_Location"
where ship_country is NULL
select count(*) from "Dim_Location"

TRUNCATE table "Dim_Location"

select * from "Dim_Product" where style is null
select * from "Dim_Date" where DATE is null
select * from "Fact_Sales" where QTY is null



-- Which product styles and categories are the most popular in the city of Mumbai?

SELECT dp.STYLE, dp.CATEGORY, COUNT(fs.ORDER_ID) as Total_Orders
FROM "CASESTUDY"."WAREHOUSE"."Fact_Sales" AS fs
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Product" dp ON fs.PRODUCT_KEY = dp.PRODUCT_KEY
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Location" dl ON fs.LOCATION_KEY = dl.LOCATION_KEY
WHERE dl.SHIP_CITY = 'MUMBAI'
GROUP BY dp.STYLE, dp.CATEGORY
ORDER BY Total_Orders DESC;


--What is the sales trend for different seasons?

SELECT dd.SEASON, dd.MONTH,
    CASE 
        WHEN month = 3 THEN 'March'
        WHEN month = 4 THEN 'April'
        WHEN month = 5 THEN 'May'
        WHEN month = 6 THEN 'June'
    END AS month_name,
    SUM(fs.AMOUNT * fs.QTY) as Total_Sales
FROM "CASESTUDY"."WAREHOUSE"."Fact_Sales" AS fs
JOIN "CASESTUDY"."WAREHOUSE"."Dim_Date" dd 
    ON fs.Date_Key = dd.Date_Key
GROUP BY dd.MONTH, dd.SEASON, month_name
ORDER BY dd.MONTH;
