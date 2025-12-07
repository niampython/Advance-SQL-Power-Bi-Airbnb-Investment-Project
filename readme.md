# Kenya Airbnb Investment SQL Case Study & Power BI Dashboard

## 1. Project Overview

This project performs a full investment analysis of short-term rentals in Kenya, using detailed Airbnb performance data combined with FX rates and local real-estate pricing.

The core business questions are:

- Is investing in short-term rentals in Kenya worth it?  
- Which cities and neighborhoods provide the strongest returns?  
- What daily rate should an investor target?  
- What type of property is most attractive from an investment standpoint?  
  - Number of bedrooms and bathrooms  
  - Property type (house, apartment, B&B, etc.)  
  - Listing type (entire home, private room, shared room)

Data focuses on major Kenyan markets such as Nairobi, Mombasa, Nakuru, Kiambu, and Kisumu, and includes:

- Trailing 12-month revenue, ADR, and occupancy metrics  
- Future occupancy and ADR projections  
- Bedroom-level revenue  
- City-level and neighborhood-level performance  
- Nairobi real-estate price data for purchase vs. rent trade-off  

The SQL problems below are framed as analytics interview-style questions, each answered with a documented T-SQL solution and a sample output.

---
## 📊 Presentation Preview

### Slide 1 — Nairobi Airbnb Investment Analysis
<img src="Power%20BI/images/slide1.png" width="700">

---

### Slide 2 — Real Estate Pricing Analysis
<img src="Power%20BI/images/slide2.png" width="700">

---

### Slide 3 — Kenya Airbnb Metrics Dashboard
<img src="Power%20BI/images/slide3.png" width="700">

### 📽 Presentation Slide Deck

👉 **[Download Power BI Presentation (.pptx)](https://github.com/niampython/Advance-SQL-Power-Bi-Airbnb-Investment-Project/raw/main/Power%20BI/Advance%20SQL-Power%20Bi%20%20Airbnb%20Investment%20Project%20Power%20Point.pptx)**  
*(Click to download — opens in PowerPoint)*


---

> 📥 **Download the full slide deck above under _Presentation Slide Deck_**
---

## 2. Tools & Technologies

### Database & Querying
- Microsoft SQL Server  
- SQL Server Management Studio (SSMS)  

### Data Engineering
- Python (pandas, SQLAlchemy, pyodbc) for loading CSVs into SQL Server  

### Analytics & Visualization
- Microsoft Power BI (DAX, Power Query, drill-through, bookmarks)  

### Supporting Assets
- Excel / CSV exports from Airdna / Airbnb data  
- Nairobi real-estate price tables  

### Data Sources
- Airbnb metrics by listing, neighborhood, city, and time  
- Kenyan exchange rates (`kenya_exchange_rates_daily`)  
- Nairobi property listings & pricing (`Nairobi_Property_Pricing`)  
- Occupancy, ADR, and revenue tables (historical & future)  

---

## 💡 Key SQL Questions & Solutions

Each section below contains:

- The business-focused SQL problem  
- The full annotated SQL solution  
- The sample output  

---

## 1️⃣ Neighborhood Revenue Ranking

### 🧩 Business Question

Aggregate Airbnb data at the **Neighborhood** level and compute:

- Total `ttm_revenue_native` per neighborhood  
- Average `ttm_avg_rate_native` per neighborhood  

Then **rank neighborhoods within their City** by total `ttm_revenue_native`.

---

 🧮 SQL Solution
```sql
------------------------------------------------------------
-- 1. Neighborhood Revenue Ranking
------------------------------------------------------------
-- Goal:
--  - Aggregate Airbnb data at the Neighborhood level
--  - Compute:
--      • Total TTM Revenue (Native)
--      • Average TTM Rate (Native)
--  - Rank neighborhoods within each City by TTM Revenue
------------------------------------------------------------

WITH CTE AS (
    SELECT 
        A.City,
        N.[Neighborhood],

        -- Step 1: Aggregate TTM revenue and average TTM rate per Neighborhood
        ROUND(SUM(A.[ttm_revenue_native]), 2) AS [Trailing Twelve Months Revenue Native],
        ROUND(AVG(A.[ttm_avg_rate_native]), 2) AS [Trailing Twelve Months Average Rate Native],

        -- Step 2: Rank neighborhoods within each City by TTM revenue (descending)
        RANK() OVER (
            PARTITION BY A.City 
            ORDER BY SUM(A.[ttm_revenue_native]) DESC
        ) AS TTM_Revenue_Native_by_Neighborhood_Rank
    FROM [Airbnb_data].[dbo].[nairobi_airbnb_data] AS A
    LEFT JOIN [dbo].[coordinates_kenya] AS N
        ON A.[latitude]  = N.[latitude]
       AND A.[longitude] = N.[longitude]
    INNER JOIN [dbo].[kenya_exchange_rates_daily] AS R 
        ON A.City = R.City
    GROUP BY 
        N.[Neighborhood],
        A.City
)

SELECT 
    Country = 'Kenya',
    City,
    Neighborhood = CTE.[Neighborhood],

    -- Step 3: Convert TTM revenue to USD using the latest exchange rate
    FORMAT(
        ROUND([Trailing Twelve Months Revenue Native] * ksh.[Exchange_Rate_KES_USD], 0),
        'C', 'en-US'
    ) AS [Trailing Twelve Months Revenue Native],

    -- Step 4: Convert average TTM rate to USD using the latest exchange rate
    FORMAT(
        ROUND([Trailing Twelve Months Average Rate Native] * ksh.[Exchange_Rate_KES_USD], 0),
        'C', 'en-US'
    ) AS [Trailing Twelve Months Average Rate Native],

    -- Step 5: Keep the rank within City
    TTM_Revenue_Native_by_Neighborhood_Rank
FROM CTE
CROSS APPLY (
    -- Step 6: Grab the latest (highest) KES→USD exchange rate
    SELECT TOP 1 [Exchange_Rate_KES_USD]
    FROM [dbo].[kenya_exchange_rates_daily]
    ORDER BY [Exchange_Rate_KES_USD] DESC
) AS ksh
ORDER BY 
    TTM_Revenue_Native_by_Neighborhood_Rank ASC;
```
📊 Sample Output
Top-performing neighborhoods in Nairobi by TTM revenue:

![Neighborhood Revenue Output](SQL%20Docs/output1.png)

## 2️⃣ Replace Missing Fees & Adjust Cleaning Fee Revenue
### 🧩 Business Question

Replace any missing fee values (e.g. cleaning_fee) with city-level average cleaning fee, then compute adjusted city / neighborhood cleaning-fee metrics.

🧮 SQL Solution
```sql
------------------------------------------------------------
-- 2. Replace Missing Cleaning Fees with City-Level Averages
------------------------------------------------------------
-- Goal:
--  - Compute average cleaning fee per City & Neighborhood
--  - Replace missing or zero cleaning_fee with this average
--  - Return average adjusted cleaning fee per City & Neighborhood
------------------------------------------------------------

-- Step 1: Compute average cleaning fee by City and Neighborhood
WITH avg_cleaning_fee AS (
    SELECT 
        A.City,
        N.Neighborhood,
        AVG(A.cleaning_fee) AS AVG_Cleaning_Fee
    FROM [Airbnb_data].[dbo].[nairobi_airbnb_data] AS A
    LEFT JOIN [dbo].[coordinates_kenya] AS N
        ON A.[latitude]  = N.[latitude]
       AND A.[longitude] = N.[longitude]
    WHERE A.cleaning_fee <> 0  -- Ignore zero/placeholder fees
    GROUP BY 
        A.City,
        N.Neighborhood
),

-- Step 2: Replace NULL/0 cleaning fees with the average cleaning fee
Fee_Replacement AS (
    SELECT 
        avg_cleaning_fee.City,
        avg_cleaning_fee.Neighborhood,
        COALESCE(AI.cleaning_fee, avg_cleaning_fee.AVG_Cleaning_Fee) AS Cleaning_Fee
    FROM [dbo].[nairobi_airbnb_data] AS AI
    INNER JOIN [dbo].[coordinates_kenya] AS C
        ON AI.[latitude]  = C.[latitude]
       AND AI.[longitude] = C.[longitude]
    INNER JOIN avg_cleaning_fee
        ON C.Neighborhood = avg_cleaning_fee.Neighborhood
)

-- Step 3: Compute the final average cleaning fee per City & Neighborhood
SELECT 
    Country = 'Kenya',
    City,
    Neighborhood,
    FORMAT(AVG(Cleaning_Fee), 'C', 'en-US') AS [Average Cleaning Fee]
FROM Fee_Replacement
GROUP BY 
    City,
    Neighborhood
ORDER BY 
    AVG(Cleaning_Fee) DESC;
```
📊 Sample Output
Average cleaning fee by Nairobi neighborhood after imputation:

![Neighborhood Revenue Output](SQL%20Docs/output2.png)

## 3️⃣ Occupancy Buckets (Last 90 Days, Pivot)
### 🧩 Business Question

Create a pivot-style query that, for each neighborhood, buckets average last 90 reserved days into:

Low Occupancy: 0–30 days

Moderate Occupancy: 31–60 days

High Occupancy: 61–90 days

🧮 SQL Solution

```sql
------------------------------------------------------------
-- 3. Occupancy Buckets for Last 90 Days (Pivot)
------------------------------------------------------------
-- Goal:
--  - For each Neighborhood, compute average reserved days in
--    the last 90 days
--  - Bucket into:
--      • Low_Occupancy (0–30)
--      • Moderate_Occupancy (31–60)
--      • High_Occupancy (61–90)
--  - Pivot these buckets into columns
------------------------------------------------------------

SELECT 
    Neighborhood,
    [Low_Occupancy],
    [Moderate_Occupancy],
    [High_Occupancy]
FROM (
    -- Step 1: Compute average reserved days and assign bucket
    SELECT DISTINCT
        N.Neighborhood,
        AVG(A.[l90d_reserved_days]) AS AVG_Reserved_Days,
        CASE 
            WHEN AVG(A.[l90d_reserved_days]) <= 30 THEN 'Low_Occupancy'
            WHEN AVG(A.[l90d_reserved_days]) BETWEEN 31 AND 60 THEN 'Moderate_Occupancy'
            WHEN AVG(A.[l90d_reserved_days]) BETWEEN 61 AND 90 THEN 'High_Occupancy'
        END AS Occupancy_Count_by_Neighborhood
    FROM [Airbnb_data].[dbo].[nairobi_airbnb_data] AS A
    LEFT JOIN [dbo].[coordinates_kenya] AS N
        ON A.[latitude]  = N.[latitude]
       AND A.[longitude] = N.[longitude]
    GROUP BY 
        N.Neighborhood
) AS Last_Three_Months_Reserved_Days
PIVOT (
    -- Step 2: Pivot the average reserved days by occupancy bucket
    SUM(AVG_Reserved_Days)
    FOR Occupancy_Count_by_Neighborhood IN (
        [Low_Occupancy],
        [Moderate_Occupancy],
        [High_Occupancy]
    )
) AS PivotTableOccupancy
ORDER BY 
    [High_Occupancy] DESC,
    [Moderate_Occupancy] DESC,
    [Low_Occupancy] DESC;
Sample Output

Neighborhood-level occupancy buckets for the last 90 days:
```
📊 Sample Output
Neighborhood-level occupancy buckets for the last 90 days:

![Neighborhood Revenue Output](SQL%20Docs/output3.png)

## 4️⃣ Real Estate Price-to-Average Nightly Rate Ratio
### 🧩 Business Question

Join Airbnb neighborhoods with Nairobi real estate locations and, for each Location:

Calculate annual stay cost (ADR × 365)

Compute Real Estate Price–to–Avg Nightly Rate ratio to compare buying a property vs. renting it as a guest.

🧮 SQL Solution

```sql
------------------------------------------------------------
-- 4. Real Estate Price-to-Avg Nightly Rate Ratio
------------------------------------------------------------
-- Goal:
--  - Join Airbnb neighborhoods to real estate Locations
--  - Compute:
--      • Annual stay cost (ADR × 365)
--      • Real estate price / annual stay cost ratio
--  - Convert to USD using a fixed exchange rate
------------------------------------------------------------

DECLARE @Exchange_Rate DECIMAL(30, 10) = 0.0077399997971952;  -- KES → USD

SELECT DISTINCT
    R.Location,

    -- Step 1: Annual stay cost (ADR × 365 × FX)
    FORMAT(
        (AVG(A.ttm_avg_rate_native) * 365) * @Exchange_Rate,
        'C', 'en-US'
    ) AS [Annual Stay Cost],

    -- Step 2: Real Estate Price-to-Avg Nightly Rate ratio (in USD)
    FORMAT(
        (
            SUM(
                CAST(
                    REPLACE(
                        REPLACE(R.[Price], 'KSH ', ''), 
                        ' ', ''
                    ) AS DECIMAL(18, 2)
                )
            )
            / (AVG(A.ttm_avg_rate_native) * 365)
        ) * @Exchange_Rate,
        'C', 'en-US'
    ) AS [Real Estate Price–to–Avg Nightly Rate ratio]
FROM [Airbnb_data].[dbo].[nairobi_airbnb_data] AS A
LEFT JOIN [dbo].[coordinates_kenya] AS N
    ON A.[latitude]  = N.[latitude]
   AND A.[longitude] = N.[longitude]
LEFT JOIN [dbo].[Nairobi_Property_Pricing] AS R 
    ON N.[Neighborhood] LIKE R.Location + '%'
    OR N.[Neighborhood] LIKE '%' + R.Location + '%'
WHERE 
    R.Location IS NOT NULL
GROUP BY 
    R.Location;
```
📊 Sample Output
Real estate locations ranked by price vs. short-term rental economics:

![Neighborhood Revenue Output](SQL%20Docs/output4.png)


## 5️⃣ City / Neighborhood Contribution to National Revenue
### 🧩 Business Question

Aggregate Airbnb listings by city / neighborhood and compute:

Total ttm_revenue_native

National total revenue

Each city’s (or neighborhood’s) percent contribution

Rank by total revenue

🧮 SQL Solution

```sql
------------------------------------------------------------
-- 5. City/Neighborhood Contribution to National Revenue
------------------------------------------------------------
-- Goal:
--  - Aggregate TTM revenue by Neighborhood (City proxy)
--  - Convert to USD
--  - Compute:
--      • City-wide total revenue
--      • Each neighborhood's revenue
--      • Neighborhood % of total
--      • Rank by revenue
------------------------------------------------------------

DECLARE @Exchange_Rate DECIMAL(30, 2) = 0.0077399997971952;  -- KES → USD

-- Step 1: Compute total revenue per Neighborhood
WITH Total_Revenue AS (
    SELECT 
        N.Neighborhood AS Neighborhood,
        SUM(A.[ttm_revenue_native]) AS Total_Revenue
    FROM [Airbnb_data].[dbo].[nairobi_airbnb_data] AS A
    LEFT JOIN [dbo].[coordinates_kenya] AS N
        ON A.[latitude]  = N.[latitude]
       AND A.[longitude] = N.[longitude]
    GROUP BY 
        N.Neighborhood
)

-- Step 2: Compute neighborhood revenue, city-wide total, % contribution, and rank
SELECT 
    Neighborhood,

    -- City-wide total revenue (sum over all neighborhoods)
    FORMAT(
        SUM(Total_Revenue * @Exchange_Rate) OVER (),
        'C', 'en-US'
    ) AS City_Wide_Total_Revenue,

    -- Revenue by Neighborhood
    FORMAT(
        Total_Revenue * @Exchange_Rate,
        'C', 'en-US'
    ) AS Total_Revenue_By_Neighborhood,

    -- Percentage contribution of each Neighborhood
    CONCAT(
        ROUND(
            (Total_Revenue / (SUM(Total_Revenue * @Exchange_Rate) OVER ())) 
            * 100 * @Exchange_Rate,
            0
        ),
        '%'
    ) AS [City’s Percentage Contribution],

    -- Rank neighborhoods by revenue
    RANK() OVER (
        ORDER BY Total_Revenue * @Exchange_Rate DESC
    ) AS [Rank of Each City Percentage Contribution]
FROM Total_Revenue;
```
📊 Sample Output
Top Nairobi neighborhoods by share of total rental revenue:

![Neighborhood Revenue Output](SQL%20Docs/output5.png)


## 6️⃣ Strongest Investment Fundamentals (Last 12 Months)
### 🧩 Business Question

“Which cities delivered the strongest investment fundamentals over the last 12 months?”

Use the composite score:

Score = (Average Daily Rate × Average Occupancy) + Average Monthly Revenue Change

🧮 SQL Solution
```sql
/*===========================================================
  6. Strongest Investment Fundamentals (Last 12 Months)
===========================================================
  Score = (Average Daily Rate × Average Occupancy)
          + Average Monthly Revenue Change

  Tables used:
    - revenueAverage_last_12_month      (Revenue)
    - rateByDailyAverage_last_12_month  (ADR)
    - occupancy_last_12_month           (Occupancy)
===========================================================*/

DECLARE @Exchange_Rate DECIMAL(30, 10) = 0.0077399997971952;  -- KES → USD

-- Step 1: Build revenue trend and MoM change
WITH RevenueTrend AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Revenue,
        Revenue - LAG(Revenue) OVER (
            PARTITION BY City 
            ORDER BY TRY_CONVERT(date, [Date])
        ) AS Revenue_MoM_Change
    FROM revenueAverage_last_12_month
),

-- Step 2: Clean ADR by city and date
CleanADR AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Average_Daily_Rate
    FROM rateByDailyAverage_last_12_month
),

-- Step 3: Clean Occupancy by city and date
CleanOcc AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Occupancy
    FROM occupancy_last_12_month
),

-- Step 4: Aggregate metrics per city
CityMetrics AS (
    SELECT
        rt.City,
        AVG(a.Average_Daily_Rate) AS Avg_ADR,
        AVG(o.Occupancy)          AS Avg_Occupancy,
        AVG(rt.Revenue_MoM_Change) AS Avg_MoM_Revenue_Growth
    FROM RevenueTrend rt
    JOIN CleanADR a
        ON rt.City = a.City 
       AND rt.DateValue = a.DateValue
    JOIN CleanOcc o
        ON rt.City = o.City 
       AND rt.DateValue = o.DateValue
    GROUP BY 
        rt.City
)

-- Step 5: Compute Investment Score and return Top 5 cities
SELECT TOP 5
    City,

    -- Occupancy as percentage (no currency)
    CONCAT(ROUND(Avg_Occupancy, 0), '%') AS Avg_Occupancy_Pct,

    -- Average MoM revenue growth, converted to USD
    FORMAT(Avg_MoM_Revenue_Growth * @Exchange_Rate, 'C', 'en-US') 
        AS Avg_MoM_Growth_USD,

    -- Combined Investment Score (numeric)
    ROUND(
        (Avg_ADR * Avg_Occupancy) 
        + (Avg_MoM_Revenue_Growth * @Exchange_Rate),
        0
    ) AS Investment_Score
FROM CityMetrics
ORDER BY 
    Investment_Score DESC;
```
📊 Sample Output
Cities ranked by investment fundamentals:

![Neighborhood Revenue Output](SQL%20Docs/output6.png)


## 7️⃣ Highest Revenue per Square Foot by Bedroom Type
### 🧩 Business Question

“Which bedroom type yields the highest revenue per square foot in each city?”

Use:

Score = Avg Monthly Revenue / Estimated SqFt

🧮 SQL Solution

```sql
/*===========================================================
  7. Highest Revenue per SqFt by Bedroom Type
===========================================================
  Score = Avg_Monthly_Revenue / Estimated SqFt

  Table used:
    - revenueByBedroom_last_12_month
===========================================================*/

DECLARE @Exchange_Rate DECIMAL(30, 10) = 0.0077399997971952;  -- KES → USD

-- Step 1: Unpivot revenue by bedroom type and convert to USD
WITH Unpivoted AS (
    SELECT 
        City,
        -- Clean bedroom type to human-friendly format
        REPLACE(REPLACE(Bedroom_Type, '_', ' '), '+', ' plus') AS Bedroom_Type,

        -- Convert revenue to USD
        CAST(Revenue * @Exchange_Rate AS DECIMAL(18, 2)) AS Revenue_Converted
    FROM revenueByBedroom_last_12_month
    UNPIVOT (
        Revenue FOR Bedroom_Type IN (
            [1_bedroom],
            [2_bedroom],
            [3_bedroom],
            [4_bedroom],
            [5_bedroom],
            [6+_bedroom]
        )
    ) AS u
),

-- Step 2: Compute average revenue and map to estimated SqFt
BedroomRevenue AS (
    SELECT
        City,
        Bedroom_Type,
        ROUND(AVG(Revenue_Converted), 1) AS Avg_Revenue_Converted,
        CASE 
            WHEN Bedroom_Type = '1 bedroom'        THEN  650
            WHEN Bedroom_Type = '2 bedroom'        THEN  950
            WHEN Bedroom_Type = '3 bedroom'        THEN 1300
            WHEN Bedroom_Type = '4 bedroom'        THEN 1600
            WHEN Bedroom_Type = '5 bedroom'        THEN 2000
            WHEN Bedroom_Type = '6 plus bedroom'   THEN 2600
            ELSE 700
        END AS SqFt
    FROM Unpivoted
    GROUP BY 
        City,
        Bedroom_Type
),

-- Step 3: Compute revenue per SqFt and rank within each city
RevenuePerSqFt AS (
    SELECT
        City,
        Bedroom_Type,
        Avg_Revenue_Converted,
        SqFt,
        ROUND(Avg_Revenue_Converted / SqFt, 2) AS Revenue_Per_SqFt,
        ROW_NUMBER() OVER (
            PARTITION BY City
            ORDER BY Avg_Revenue_Converted / SqFt DESC
        ) AS rn
    FROM BedroomRevenue
)

-- Step 4: Return the top bedroom type per city
SELECT
    City,
    Bedroom_Type,
    CONCAT('$', FORMAT(Avg_Revenue_Converted, 'N1')) AS Avg_Revenue,
    SqFt,
    CONCAT('$', FORMAT(Revenue_Per_SqFt, 'N2')) AS Revenue_Per_SqFt
FROM RevenuePerSqFt
WHERE rn = 1
ORDER BY 
    Revenue_Per_SqFt DESC;
```
📊 Sample Output
Best bedroom configuration by revenue per square foot:

![Neighborhood Revenue Output](SQL%20Docs/output7.png)


## 8️⃣ Strongest Future Booking Pace & ADR Trend
### 🧩 Business Question

“Which cities show the strongest future booking pace, and how does future ADR compare to historical ADR?”

🧮 SQL Solution

```sql
/*===========================================================
  8. Strongest Future Booking Pace vs Historical Metrics
===========================================================
  Tables:
    - occupancyFuture_next_30_days
    - occupancy_last_12_month
    - rateFuture_next_180_days
    - rateByDailyAverage_last_12_month
===========================================================*/

-- Step 1: Historical occupancy and ADR
WITH Historical AS (
    SELECT
        o.City,
        AVG(o.Occupancy)          AS Hist_Occupancy,
        AVG(r.Average_Daily_Rate) AS Hist_ADR
    FROM occupancy_last_12_month AS o
    JOIN rateByDailyAverage_last_12_month AS r
        ON o.City = r.City
       AND o.[Date] = r.[Date]
    GROUP BY 
        o.City
),

-- Step 2: Future 30-day occupancy
FutureMetrics AS (
    SELECT
        City,
        AVG([Last_30_Days]) AS Future_30Day_Occupancy
    FROM occupancyFuture_next_30_days
    GROUP BY 
        City
),

-- Step 3: Future 180-day ADR
FutureADR AS (
    SELECT
        City,
        ROUND(AVG([Daily_Rate]), 2) AS Future_180Day_ADR
    FROM rateFuture_next_180_days
    GROUP BY 
        City
)

-- Step 4: Compare historical vs future metrics
SELECT
    h.City,
    ROUND(Hist_Occupancy, 2)   AS Hist_Occupancy,
    Future_30Day_Occupancy,
    Hist_ADR,
    Future_180Day_ADR,

    -- Occupancy trend (future vs historical)
    ROUND(Future_30Day_Occupancy - Hist_Occupancy, 2) AS Occupancy_Trend,

    -- ADR trend (future vs historical)
    (Future_180Day_ADR - Hist_ADR) AS ADR_Trend
FROM Historical AS h
JOIN FutureMetrics AS f 
    ON h.City = f.City
JOIN FutureADR AS a 
    ON h.City = a.City
WHERE 
    -- Step 5: Only keep improving markets
    Future_30Day_Occupancy > Hist_Occupancy
    AND Future_180Day_ADR > Hist_ADR
ORDER BY
    -- Strongest combined improvement first
    (Future_30Day_Occupancy - Hist_Occupancy)
    + (Future_180Day_ADR - Hist_ADR) DESC;
```

📊 Sample Output
Historical vs future occupancy and ADR (raw values):

![Neighborhood Revenue Output](SQL%20Docs/output8.png)


## 9️⃣ Future Booking Pace vs Historical (Final – USD & % Formatting)
### 🧩 Business Question

Use the real production tables to compare historical vs future occupancy & ADR, fully formatted:

Historical vs Future occupancy (%)

Historical vs Future ADR (USD)

Trend columns with clean formatting

🧮 SQL Solution

```sql
Copy code
/*===========================================================
  9. Future Booking Pace & ADR Trend (Final Version)
===========================================================
  Future Booking Pace     = AVG(Last_30_Days) from occupancyFuture_next_30_days
  Historical Occupancy    = AVG(Occupancy)    from occupancy_last_12_month

  Future ADR              = AVG(Daily_Rate)          from rateFuture_next_180_days
  Historical ADR          = AVG(Average_Daily_Rate)  from rateByDailyAverage_last_12_month

  Includes:
    - Date normalization
    - USD conversion via exchange rate
    - % and currency formatting
===========================================================*/

DECLARE @Exchange_Rate DECIMAL(30, 10) = 0.0077399997971952;  -- KES → USD

-- Step 1: Clean & normalize historical occupancy
WITH HistOcc AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Occupancy
    FROM occupancy_last_12_month
),

-- Step 2: Clean & normalize historical ADR
HistADR AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Average_Daily_Rate
    FROM rateByDailyAverage_last_12_month
),

-- Step 3: Clean & normalize future occupancy (next 30 days)
FutureOcc AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Last_30_Days
    FROM occupancyFuture_next_30_days
),

-- Step 4: Clean & normalize future ADR (next 180 days)
FutureADR AS (
    SELECT
        City,
        TRY_CONVERT(date, [Date]) AS DateValue,
        Daily_Rate
    FROM rateFuture_next_180_days
),

-- Step 5: Aggregate metrics per city
CityMetrics AS (
    SELECT
        h.City,

        -- Historical occupancy (12-month average)
        AVG(h.Occupancy) AS Hist_Occ,

        -- Future 30-day occupancy
        AVG(f.Last_30_Days) AS Future_Occ_30,

        -- Historical ADR (USD)
        AVG(a.Average_Daily_Rate * @Exchange_Rate) AS Hist_ADR_USD,

        -- Future ADR (USD)
        AVG(fr.Daily_Rate * @Exchange_Rate) AS Future_ADR_USD
    FROM HistOcc AS h
    JOIN HistADR AS a
        ON h.City = a.City
    JOIN FutureOcc AS f
        ON h.City = f.City
    JOIN FutureADR AS fr
        ON h.City = fr.City
    GROUP BY 
        h.City
)

-- Step 6: Present formatted comparison & trends
SELECT
    City,

    -- Historical vs Future Occupancy (%)
    CONCAT(ROUND(Hist_Occ,      0), '%') AS Historical_Occupancy,
    CONCAT(ROUND(Future_Occ_30, 0), '%') AS Future_30Day_Occupancy,

    -- Occupancy trend (%)
    CONCAT(ROUND(Future_Occ_30 - Hist_Occ, 0), '%') AS Occupancy_Trend,

    -- Historical ADR (USD)
    CONCAT('$', FORMAT(Hist_ADR_USD, 'N2')) AS Historical_ADR_USD,

    -- Future ADR (USD)
    CONCAT('$', FORMAT(Future_ADR_USD, 'N2')) AS Future_ADR_USD,

    -- ADR Trend (USD)
    CONCAT('$', FORMAT(Future_ADR_USD - Hist_ADR_USD, 'N2')) AS ADR_Trend_USD
FROM CityMetrics
WHERE 
    -- Only show cities with improving fundamentals
    Future_Occ_30 > Hist_Occ        -- stronger booking pace
    AND Future_ADR_USD > Hist_ADR_USD  -- ADR rising
ORDER BY 
    -- Strongest combined improvement first
    (Future_Occ_30 - Hist_Occ) 
    + (Future_ADR_USD - Hist_ADR_USD) DESC;
```

📊 Sample Output
Formatted comparison of historical vs future occupancy and ADR (USD + %):

![Neighborhood Revenue Output](SQL%20Docs/output9.png)


## ⚙️ Challenges & How I Overcame Them
### 🔗 Joining Airbnb Listings to Neighborhoods

***Challenge***: Airbnb data was keyed by latitude/longitude, while neighborhood names lived in a separate coordinates_kenya table.

Solution: Used precise lat/long joins plus careful cleaning of coordinate columns to map listings to the correct neighborhood, enabling neighborhood-level revenue and occupancy analysis.

### 💱 Cleaning Currency & Converting KES → USD

***Challenge***: Monetary values (like Price in Nairobi_Property_Pricing) were stored as strings with prefixes like "KSH " and embedded spaces.

Solution: Applied REPLACE and CAST in SQL to turn them into numeric types, then consistently applied a KES → USD exchange rate to standardize all financial metrics for investors.

### 🧽 Handling Missing & Zero Cleaning Fees

***Challenge***: Many hosts had cleaning_fee = 0 or missing values, which distorted average fee calculations.

Solution: Built a two-stage CTE:

Compute city + neighborhood–level average cleaning fee.

Use COALESCE to impute missing/zero cleaning fees based on those averages.

### ⏱️ Time-Series Alignment Across Multiple Metric Tables

***Challenge***: Historical and future tables (occupancy_last_12_month, occupancyFuture_next_30_days, rateByDailyAverage_last_12_month, rateFuture_next_180_days, etc.) used different date formats and granularities.

Solution: Used TRY_CONVERT(date, [Date]) to normalize all date columns and joined on clean date keys, enabling accurate calculation of MoM revenue trends and future booking pace.

### 🖥️ Environment & Connectivity (Python → SQL Server)

***Challenge***: On a new machine, Python could not connect to SQL Server due to driver issues and missing environments (ODBC error 08001, ipykernel issues in VS Code).

Solution:

Installed Miniconda and created an isolated environment airbnb-sql with pandas, sqlalchemy, and pyodbc.

Registered the environment as a Jupyter kernel for VS Code.

Used SQLAlchemy’s connection string with Windows authentication and the correct SQL Server instance to reliably load all county CSVs into [Airbnb_data].

### 📈 Modeling Investment Metrics for Real-World Decisions

***Challenge***: Translating raw Airbnb metrics into something an investor can act on (e.g., “Is Nakuru better than Nairobi?”).

Solution:

Defined clear investment scores combining ADR, occupancy, and revenue trends.

Built per-sqft revenue metrics by bedroom type to guide property selection.

Surfaced everything in Power BI with investor-friendly visuals and filterable views.

## 🧾 Investor Takeaways

From this analysis, an investor can:

Identify the best cities and neighborhoods in Kenya for short-term rental investment (e.g., top Nairobi wards like Kilimani, Kileleshwa, Karen).

Set data-driven nightly rates based on historical ADR and forward-looking ADR trends.

Choose optimal property types:

Which bedroom count yields the highest revenue per square foot in each city.

Whether to focus on entire homes, apartments, houses, or private rooms, guided by the performance of similar listings.

Quantify how future occupancy and ADR trends are moving compared to the last 12 months, to avoid over-investing in markets that are already slowing down.

This README ties together the SQL analytics layer and the Power BI visualization layer to provide a full, end-to-end view of short-term rental investment potential in Kenya.

## 🚀 How to Use This Repository
### 1. Run the SQL Scripts

Create and load the underlying tables in SQL Server using the Python ETL scripts.

Execute each query in this README to reproduce the analytical outputs.

### 2. Connect Power BI

Point Power BI to the same SQL Server database (Airbnb_data).

Recreate or extend the provided dashboard visuals.

### 3. Adapt to Other Markets

Replace Kenya-specific tables with other countries/regions.

Keep the same query patterns for:

Ranking neighborhoods

Future vs historical booking trends

Property-type and bedroom-level performance.



