# Power BI — Egypt Real Estate 2026
## Role & Context

You are a senior Power BI developer and data modeler.
Build a complete `.pbip` Power BI project from the Egypt Real Estate 2026 base dataset only.
No ML, NLP, or forecast files are required — all visuals are derived from the two source files below.

---

## Source Files to Import

| Table Name (in Power BI) | Source File | Notes |
|---|---|---|
| `Listings` | `egypt_re_clean.csv` | Primary fact source — 53 columns |
| `Amenities` | `egypt_re_amenities_binary.csv` | Binary amenity flags per listing_id |

---

## Dashboard Map (6 Pages)

| Page | Title | Focus |
|---|---|---|
| 1 | Market Overview | KPIs, composition, price bands |
| 2 | Location Intelligence | Geographic pricing, city/town hierarchy |
| 3 | Property Specs & Pricing | Bedrooms, area, amenities vs price |
| 4 | Agent & Listing Quality | Broker performance, quality flags |
| 5 | Temporal Trends & Seasonality | Volume and price over time, day/week/month patterns |
| 6 | Time Intelligence Deep Dive | YoY, QoQ, rolling windows, listing velocity, price momentum |

---

## Step 1 — Load Data in Power Query

### 1A. Load `egypt_re_clean.csv`

1. Open Power BI Desktop → **Home → Get Data → Text/CSV**.
2. Select `egypt_re_clean.csv`. Click **Transform Data** (do not load directly).
3. In Power Query Editor, rename the query to `Listings`.
4. Set column types explicitly — do not rely on auto-detection:

| Column | Type |
|---|---|
| `listing_id` | Text |
| `price_egp`, `price_per_sqm`, `area_value`, `lat`, `lon` | Decimal Number |
| `bedrooms`, `bathrooms`, `amenity_count`, `images_count`, `days_since_listed`, `listing_age_days`, `is_furnished_binary` | Whole Number |
| `listed_date` | Date |
| `scraped_at` | Date/Time |
| `is_premium`, `is_verified`, `is_featured`, `is_new_construction`, `is_direct_from_dev`, `is_exclusive`, `agent_is_super`, `has_view_360` | True/False |
| All `amen_*` columns | Whole Number |
| All remaining columns | Text |

5. Keep only these columns in the query (remove the rest):
   `listing_id, price_egp, price_per_sqm, area_value, bedrooms, bathrooms, amenity_count, days_since_listed, listing_age_days, images_count, is_furnished_binary, is_premium, is_verified, is_featured, is_new_construction, is_direct_from_dev, is_exclusive, listed_date, scraped_at, lat, lon, city, town, district, subdistrict, property_type, offering_type, price_period, furnished, agent_id, agent_name, agent_is_super, broker_name, category, completion_status`
6. Click **Close & Apply**.

### 1B. Load `egypt_re_amenities_binary.csv`

1. **Home → Get Data → Text/CSV** → select `egypt_re_amenities_binary.csv`.
2. Click **Transform Data**. Rename query to `Amenities`.
3. Confirm `listing_id` is Text type. Set all `amen_*` columns to Whole Number.
4. Click **Close & Apply**.

---

## Step 2 — Build the Star Schema

### 2A. Create Dimension Tables in Power Query

Re-open Power Query Editor (**Home → Transform Data**) for each step.

**DimLocation**
1. Right-click `Listings` → **Reference**. Rename to `DimLocation`.
2. Select only: `city, town, district, subdistrict, lat, lon`.
3. **Home → Remove Duplicates**. Add Index Column from 1 → rename `city_key`.

**DimPropertyType**
1. Reference `Listings` → rename `DimPropertyType`.
2. Select: `property_type, category`. Remove Duplicates. Index → `property_type_key`.

**DimOfferingType**
1. Reference `Listings` → rename `DimOfferingType`.
2. Select: `offering_type, price_period`. Remove Duplicates. Index → `offering_type_key`.

**DimAgent**
1. Reference `Listings` → rename `DimAgent`.
2. Select: `agent_id, agent_name, agent_is_super, broker_name`. Remove Duplicates on `agent_id`.

**DimFurnished**
1. Reference `Listings` → rename `DimFurnished`.
2. Select: `furnished`. Remove Duplicates. Index → `furnished_key`.

### 2B. Trim the Fact Table

1. Open `Listings` query → rename to `FactListings`.
2. Remove these columns (they live in dimensions now):
   `category, property_type, offering_type, price_period, furnished, agent_name, agent_is_super, broker_name, town, district, subdistrict`.
3. Retain `city`, `property_type`, `offering_type`, `furnished`, `agent_id` as FK text columns for joining.
4. Close & Apply.

### 2C. Create Relationships in Model View

Go to **Model view** (left sidebar).

| From (Many) | To (One) | Column |
|---|---|---|
| `FactListings[city]` | `DimLocation[city]` | city |
| `FactListings[property_type]` | `DimPropertyType[property_type]` | property_type |
| `FactListings[offering_type]` | `DimOfferingType[offering_type]` | offering_type |
| `FactListings[agent_id]` | `DimAgent[agent_id]` | agent_id |
| `FactListings[furnished]` | `DimFurnished[furnished]` | furnished |
| `FactListings[listing_id]` | `Amenities[listing_id]` | listing_id |
| `FactListings[listed_date]` | `Calendar[Date]` | date |

All cross-filter directions: **Single**. All cardinalities: **Many-to-One**.

---

## Step 3 — Calendar Table (DAX)

**Home → New Table**. Paste the full expression:

```dax
Calendar =
ADDCOLUMNS(
    CALENDAR(DATE(2024,1,1), DATE(2026,12,31)),
    "Year",             YEAR([Date]),
    "Quarter",          "Q" & FORMAT(QUARTER([Date]),"0"),
    "Quarter Number",   QUARTER([Date]),
    "Year-Quarter",     STR(YEAR([Date])) & "-Q" & FORMAT(QUARTER([Date]),"0"),
    "Month Number",     MONTH([Date]),
    "Month Name",       FORMAT([Date], "mmmm"),
    "Month Short",      FORMAT([Date], "mmm"),
    "Year-Month",       FORMAT([Date], "YYYY-MM"),
    "Year-Month Label", FORMAT([Date], "mmm YYYY"),
    "Day",              DAY([Date]),
    "Day Name",         FORMAT([Date], "dddd"),
    "Day Short",        FORMAT([Date], "ddd"),
    "Day of Week Number", WEEKDAY([Date], 2),
    "Is Weekend",       IF(WEEKDAY([Date], 2) >= 6, TRUE(), FALSE()),
    "Day Type",         IF(WEEKDAY([Date], 2) >= 6, "Weekend", "Weekday"),
    "Week Number",      WEEKNUM([Date], 2),
    "Month Period",
        SWITCH(
            TRUE(),
            DAY([Date]) <= 10, "Beginning",
            DAY([Date]) <= 20, "Middle",
            "End"
        ),
    "Season",
        SWITCH(
            TRUE(),
            MONTH([Date]) IN {12,1,2}, "Winter",
            MONTH([Date]) IN {3,4,5}, "Spring",
            MONTH([Date]) IN {6,7,8}, "Summer",
            "Autumn"
        )
)
```

After creating:
1. Select `Calendar` table → **Table tools → Mark as date table** → choose `Date` column.
2. In Model view, connect `FactListings[listed_date]` → `Calendar[Date]`.

---

## Step 4 — DAX Measures

Create an empty table: **Home → New Table** → `_Measures = {1}`. Delete the auto-generated column. All measures go here.

### 4A — Volume Counts

```dax
Total Listings =
COUNTROWS(FactListings)

Total Buy Listings =
CALCULATE([Total Listings], DimOfferingType[offering_type] = "Buy")

Total Rent Listings =
CALCULATE([Total Listings], DimOfferingType[offering_type] = "Rent")

Listings Last 7 Days =
CALCULATE(
    [Total Listings],
    DATESINPERIOD(Calendar[Date], LASTDATE(Calendar[Date]), -7, DAY)
)

Listings Last 30 Days =
CALCULATE(
    [Total Listings],
    DATESINPERIOD(Calendar[Date], LASTDATE(Calendar[Date]), -30, DAY)
)

Listings Last 90 Days =
CALCULATE(
    [Total Listings],
    DATESINPERIOD(Calendar[Date], LASTDATE(Calendar[Date]), -90, DAY)
)
```

### 4B — Price Measures

```dax
Median Price EGP =
MEDIAN(FactListings[price_egp])

Avg Price EGP =
AVERAGE(FactListings[price_egp])

Avg Price Per SQM =
AVERAGE(FactListings[price_per_sqm])

Median Price Per SQM =
MEDIAN(FactListings[price_per_sqm])

Avg Buy Price =
CALCULATE(MEDIAN(FactListings[price_egp]), DimOfferingType[offering_type] = "Buy")

Avg Buy Price Per SQM =
CALCULATE(AVERAGE(FactListings[price_per_sqm]), DimOfferingType[offering_type] = "Buy")

Avg Monthly Rent =
CALCULATE(MEDIAN(FactListings[price_egp]), DimOfferingType[offering_type] = "Rent")

Avg Rent Per SQM =
CALCULATE(AVERAGE(FactListings[price_per_sqm]), DimOfferingType[offering_type] = "Rent")
```

### 4C — Rental Yield

```dax
Annual Rental Yield % =
DIVIDE([Avg Monthly Rent] * 12, [Avg Buy Price], 0) * 100
```

### 4D — Property Specs

```dax
Avg Area SQM =       AVERAGE(FactListings[area_value])
Avg Bedrooms =       AVERAGE(FactListings[bedrooms])
Avg Bathrooms =      AVERAGE(FactListings[bathrooms])
Avg Amenity Count =  AVERAGE(FactListings[amenity_count])
Avg Days Since Listed = AVERAGE(FactListings[days_since_listed])
```

### 4E — Quality & Flag Measures

```dax
Premium Listings % =
DIVIDE(CALCULATE([Total Listings], FactListings[is_premium] = TRUE()), [Total Listings], 0) * 100

Verified Listings % =
DIVIDE(CALCULATE([Total Listings], FactListings[is_verified] = TRUE()), [Total Listings], 0) * 100

New Construction % =
DIVIDE(CALCULATE([Total Listings], FactListings[is_new_construction] = TRUE()), [Total Listings], 0) * 100

Direct From Dev % =
DIVIDE(CALCULATE([Total Listings], FactListings[is_direct_from_dev] = TRUE()), [Total Listings], 0) * 100

Featured Listings % =
DIVIDE(CALCULATE([Total Listings], FactListings[is_featured] = TRUE()), [Total Listings], 0) * 100

Furnished Listings % =
DIVIDE(CALCULATE([Total Listings], FactListings[is_furnished_binary] = 1), [Total Listings], 0) * 100
```

### 4F — Price Band Measures

```dax
Listings Under 5M =
CALCULATE([Total Listings], FactListings[price_egp] < 5000000, DimOfferingType[offering_type] = "Buy")

Listings 5M to 15M =
CALCULATE([Total Listings], FactListings[price_egp] >= 5000000, FactListings[price_egp] < 15000000, DimOfferingType[offering_type] = "Buy")

Listings Above 15M =
CALCULATE([Total Listings], FactListings[price_egp] >= 15000000, DimOfferingType[offering_type] = "Buy")

Listings Under 10K Rent =
CALCULATE([Total Listings], FactListings[price_egp] < 10000, DimOfferingType[offering_type] = "Rent")

Listings 10K to 50K Rent =
CALCULATE([Total Listings], FactListings[price_egp] >= 10000, FactListings[price_egp] < 50000, DimOfferingType[offering_type] = "Rent")

Listings Above 50K Rent =
CALCULATE([Total Listings], FactListings[price_egp] >= 50000, DimOfferingType[offering_type] = "Rent")
```

### 4G — Amenity Measures

```dax
Pool Listings =
CALCULATE([Total Listings], FILTER(Amenities, Amenities[amen_shared_pool] = 1))

Gym Listings =
CALCULATE([Total Listings], FILTER(Amenities, Amenities[amen_shared_gym] = 1))

Security Listings =
CALCULATE([Total Listings], FILTER(Amenities, Amenities[amen_security] = 1))

Avg Price With Pool =
CALCULATE(AVERAGE(FactListings[price_per_sqm]), FILTER(Amenities, Amenities[amen_shared_pool] = 1))

Avg Price Without Pool =
CALCULATE(AVERAGE(FactListings[price_per_sqm]), FILTER(Amenities, Amenities[amen_shared_pool] = 0))

Pool Price Premium % =
DIVIDE([Avg Price With Pool] - [Avg Price Without Pool], [Avg Price Without Pool], 0) * 100
```

### 4H — Time Intelligence Measures

```dax
Listings MoM Change % =
VAR CurrentMonth = [Total Listings]
VAR PrevMonth =
    CALCULATE([Total Listings], DATEADD(Calendar[Date], -1, MONTH))
RETURN
DIVIDE(CurrentMonth - PrevMonth, PrevMonth, 0) * 100

Price MoM Change % =
VAR CurrentMedian = [Median Price EGP]
VAR PrevMedian =
    CALCULATE([Median Price EGP], DATEADD(Calendar[Date], -1, MONTH))
RETURN
DIVIDE(CurrentMedian - PrevMedian, PrevMedian, 0) * 100

Listings QoQ Change % =
VAR CurrentQ = [Total Listings]
VAR PrevQ =
    CALCULATE([Total Listings], DATEADD(Calendar[Date], -1, QUARTER))
RETURN
DIVIDE(CurrentQ - PrevQ, PrevQ, 0) * 100

Listings YoY Change % =
VAR CurrentY = [Total Listings]
VAR PrevY =
    CALCULATE([Total Listings], DATEADD(Calendar[Date], -1, YEAR))
RETURN
DIVIDE(CurrentY - PrevY, PrevY, 0) * 100

Price YoY Change % =
VAR CurrentY = [Median Price EGP]
VAR PrevY =
    CALCULATE([Median Price EGP], DATEADD(Calendar[Date], -1, YEAR))
RETURN
DIVIDE(CurrentY - PrevY, PrevY, 0) * 100

Listings 3M Rolling Avg =
AVERAGEX(
    DATESINPERIOD(Calendar[Date], LASTDATE(Calendar[Date]), -3, MONTH),
    [Total Listings]
)

Price 6M Rolling Avg =
AVERAGEX(
    DATESINPERIOD(Calendar[Date], LASTDATE(Calendar[Date]), -6, MONTH),
    [Median Price EGP]
)

Listings YTD =
TOTALYTD([Total Listings], Calendar[Date])

Listings PYTD =
CALCULATE(
    [Listings YTD],
    SAMEPERIODLASTYEAR(Calendar[Date])
)

Listings YTD vs PYTD =
[Listings YTD] - [Listings PYTD]

Avg Listing Age Days =
AVERAGE(FactListings[listing_age_days])

Stale Listings (90+ days) =
CALCULATE(
    [Total Listings],
    FactListings[listing_age_days] > 90
)

Stale Listings % =
DIVIDE([Stale Listings (90+ days)], [Total Listings], 0) * 100

New Buy Listings This Month =
CALCULATE(
    [Total Buy Listings],
    DATESMTD(Calendar[Date])
)

New Rent Listings This Month =
CALCULATE(
    [Total Rent Listings],
    DATESMTD(Calendar[Date])
)
```

---

## Step 5 — Dashboard 1: Market Overview

**Page name:** `Market Overview` | **Canvas:** 1366 × 768

**Row 1 — KPI Cards (5 across)**
1. Insert → **Card** × 5. Assign:
   - `[Total Listings]` | `[Total Buy Listings]` | `[Total Rent Listings]` | `[Avg Buy Price]` (EGP, 0 decimals) | `[Avg Monthly Rent]` (EGP, 0 decimals)
2. Each card: Format → Callout value → 28pt bold. Add descriptive subtitle label.

**Donut — Buy vs Rent Split**
- Legend: `DimOfferingType[offering_type]` | Values: `[Total Listings]`
- Title: "Listing Type Split"

**Bar Chart — Listing Count by Property Type**
- Y-axis: `DimPropertyType[property_type]` | X-axis: `[Total Listings]` | Sort descending
- Title: "Listings by Property Type"

**Bar Chart — Listing Count by City**
- Y-axis: `DimLocation[city]` | X-axis: `[Total Listings]` | Sort descending
- Title: "Listings by City"

**Stacked Bar — Buy Price Band by City**
- Y-axis: `DimLocation[city]`
- Values: `[Listings Under 5M]`, `[Listings 5M to 15M]`, `[Listings Above 15M]`
- Title: "Buy Listings by Price Band & City"

**Slicers (right sidebar, Dropdown style):**
`DimLocation[city]` | `DimPropertyType[property_type]` | `DimOfferingType[offering_type]`

---

## Step 6 — Dashboard 2: Location Intelligence

**Page name:** `Location Intelligence` | **Canvas:** 1366 × 768

**Bubble Map — Avg Price Per SQM by District**
1. Insert → **Map** visual.
2. Location: `DimLocation[district]` | Latitude: `DimLocation[lat]` (Average) | Longitude: `DimLocation[lon]` (Average)
3. Bubble size: `[Avg Price Per SQM]` | Color saturation: `[Avg Price Per SQM]`
4. Tooltips: `[Total Listings]`, `[Avg Buy Price]`, `[Avg Monthly Rent]`
5. Title: "Avg Price Per SQM by District"

**Matrix — Location Hierarchy Drill-down**
1. Insert → **Matrix**.
2. Rows: `DimLocation[city]` → `DimLocation[town]` → `DimLocation[district]` (in order)
3. Values: `[Avg Price Per SQM]`, `[Total Listings]`, `[Avg Buy Price]`
4. Format → Stepped layout On. Enable drill-down arrows.
5. Title: "Price Intelligence — City → Town → District"

**Bar Chart — Top 15 Towns by Median Buy Price**
- Y-axis: `DimLocation[town]` | X-axis: `[Avg Buy Price]`
- Visual-level filter: `offering_type = "Buy"` | Top N: 15 by `[Avg Buy Price]`
- Title: "Top 15 Towns — Median Buy Price"

**Bar Chart — Top 15 Towns by Listing Volume**
- Duplicate above. X-axis: `[Total Listings]` | Top N: 15 by `[Total Listings]`
- Title: "Top 15 Towns by Listing Volume"

**Slicers:** `DimLocation[city]` | `DimOfferingType[offering_type]` | `DimPropertyType[property_type]`

---

## Step 7 — Dashboard 3: Property Specs & Pricing

**Page name:** `Specs & Pricing` | **Canvas:** 1366 × 768

**Column Chart — Median Price Per SQM by Bedroom Count**
- X-axis: `FactListings[bedrooms]` | Y-axis: `[Median Price Per SQM]`
- Visual-level filter: `offering_type = "Buy"` | Sort bedrooms ascending
- Title: "Price Per SQM by Bedroom Count (Buy)"

**Column Chart — Median Price Per SQM by Bathroom Count**
- X-axis: `FactListings[bathrooms]` | Y-axis: `[Median Price Per SQM]`
- Title: "Price Per SQM by Bathroom Count"

**Column Chart — Avg Price Per SQM by Furnished Status**
- X-axis: `DimFurnished[furnished]` | Y-axis: `[Avg Price Per SQM]`
- Title: "Price Per SQM by Furnished Status"

**Scatter Plot — Area vs Price (Log Scale)**
- X-axis: `FactListings[area_value]` (avg) | Y-axis: `FactListings[price_egp]` (avg)
- Legend: `DimPropertyType[property_type]` | Size: `[Total Listings]`
- Format → X-axis log scale On. Y-axis log scale On.
- Title: "Area vs Price by Property Type (Log Scale)"

**AmenityPriceSummary Calculated Table** (create via **Modeling → New Table**):

```dax
AmenityPriceSummary =
UNION(
    ROW("Amenity","Balcony",           "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_balcony]=1))),
    ROW("Amenity","Security",          "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_security]=1))),
    ROW("Amenity","Covered Parking",   "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_covered_parking]=1))),
    ROW("Amenity","Landmark View",     "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_view_of_landmark]=1))),
    ROW("Amenity","Shared Gym",        "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_shared_gym]=1))),
    ROW("Amenity","Lobby in Building", "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_lobby_in_building]=1))),
    ROW("Amenity","Central A/C",       "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_central_a_c]=1))),
    ROW("Amenity","Kitchen Appliances","Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_kitchen_appliances]=1))),
    ROW("Amenity","Built-in Wardrobes","Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_built_in_wardrobes]=1))),
    ROW("Amenity","Shared Spa",        "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_shared_spa]=1))),
    ROW("Amenity","Shared Pool",       "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_shared_pool]=1))),
    ROW("Amenity","Walk-in Closet",    "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_walk_in_closet]=1))),
    ROW("Amenity","Study",             "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_study]=1))),
    ROW("Amenity","Children's Pool",   "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_children_s_pool]=1))),
    ROW("Amenity","Water View",        "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_view_of_water]=1))),
    ROW("Amenity","Private Garden",    "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_private_garden]=1))),
    ROW("Amenity","Maids Room",        "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_maids_room]=1))),
    ROW("Amenity","Private Pool",      "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_private_pool]=1))),
    ROW("Amenity","Networked",         "Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_networked]=1))),
    ROW("Amenity","Dining in Building","Avg_SQM", CALCULATE(AVERAGE(FactListings[price_per_sqm]),FILTER(Amenities,Amenities[amen_dining_in_building]=1)))
)
```

**Bar Chart — Amenities by Avg Price Per SQM**
- Y-axis: `AmenityPriceSummary[Amenity]` | X-axis: `AmenityPriceSummary[Avg_SQM]`
- Sort descending. Title: "Amenities Ranked by Avg Price Per SQM"

**Line Chart — Amenity Count vs Avg Price**
- X-axis: `FactListings[amenity_count]` | Y-axis: `[Avg Price Per SQM]`
- Title: "Amenity Count vs Avg Price Per SQM"

**Slicers:** `DimLocation[city]` | `DimPropertyType[property_type]` | `FactListings[bedrooms]` (range)

---

## Step 8 — Dashboard 4: Agent & Listing Quality

**Page name:** `Agent & Quality` | **Canvas:** 1366 × 768

**KPI Cards — Two rows of three:**
- Row 1: `[Premium Listings %]` | `[Verified Listings %]` | `[New Construction %]`
- Row 2: `[Direct From Dev %]` | `[Featured Listings %]` | `[Furnished Listings %]`
- Format all as percentages, 1 decimal place.

**Bar Chart — Top 10 Brokers by Listing Count**
- Y-axis: `DimAgent[broker_name]` | X-axis: `[Total Listings]`
- Top N filter: 10 by `[Total Listings]` | Sort descending
- Title: "Top 10 Brokers by Listing Count"

**Bar Chart — Top 10 Brokers by Avg Price Per SQM**
- Duplicate above → X-axis: `[Avg Price Per SQM]` | Top N: 10 by `[Avg Price Per SQM]`
- Title: "Top 10 Brokers by Avg Price Per SQM"

**Donut — Direct from Developer vs Broker**
- Legend: `FactListings[is_direct_from_dev]` | Values: `[Total Listings]`
- Title: "Direct from Developer Distribution"

**Scatter — Agent Volume vs Avg Price**
- X-axis: `[Total Listings]` | Y-axis: `[Avg Price Per SQM]`
- Details: `DimAgent[agent_name]`
- Title: "Agent: Volume vs Avg Price"

**Table — Agent Performance Detail**
- Columns: `agent_name`, `broker_name`, `[Total Listings]`, `[Avg Price EGP]`, `[Avg Days Since Listed]`, `[Premium Listings %]`
- Conditional formatting on `[Avg Price EGP]` (color scale). Enable sort on all columns.
- Title: "Agent Performance Detail"

**Slicers:** `DimLocation[city]` | `DimOfferingType[offering_type]` | `FactListings[is_premium]`

---

## Step 9 — Dashboard 5: Temporal Trends & Seasonality

**Page name:** `Temporal Trends` | **Canvas:** 1366 × 768

**Purpose:** How listing volume and pricing evolve month by month, and which time periods drive peak activity.

**Line Chart — Monthly Listing Volume (Buy vs Rent)**
1. Insert → **Line chart**.
2. X-axis: `Calendar[Year-Month Label]` — sort by `Calendar[Year-Month]` (sort-by-column: right-click `Year-Month Label` in Data pane → Sort by Column → `Year-Month`).
3. Y-axis: `[Total Listings]`
4. Legend: `DimOfferingType[offering_type]`
5. Analytics pane → **Average line**: Value = `[Total Listings]`, label "Avg", dashed.
6. Title: "Monthly Listing Volume — Buy vs Rent"

**Line Chart — Monthly Median Price (Buy vs Rent)**
1. Insert → **Line chart**.
2. X-axis: `Calendar[Year-Month Label]` (sorted as above)
3. Y-axis: `[Median Price EGP]` — format as EGP currency.
4. Legend: `DimOfferingType[offering_type]`
5. Analytics pane → **Median line** for reference.
6. Title: "Monthly Median Price — Buy vs Rent"

**Column Chart — Listing Volume by Day of Week**
1. Insert → **Clustered column chart**.
2. X-axis: `Calendar[Day Name]` — sort by `Calendar[Day of Week Number]`.
   - In Data pane, right-click `Day Name` → **Sort by Column → Day of Week Number**.
3. Y-axis: `[Total Listings]`
4. Color: add `Calendar[Day Type]` to Legend (Weekend vs Weekday).
5. Title: "Listing Volume by Day of Week"

**Column Chart — Listing Seasonality by Month**
1. Insert → **Clustered column chart**.
2. X-axis: `Calendar[Month Short]` — sort by `Calendar[Month Number]`.
3. Y-axis: `[Total Listings]`
4. Legend: `DimOfferingType[offering_type]`
5. Title: "Listing Volume by Month (Seasonality)"

**Stacked Bar — Listings by Season**
1. Insert → **Stacked bar chart**.
2. Y-axis: `Calendar[Season]`
3. X-axis: `[Total Listings]`
4. Legend: `DimOfferingType[offering_type]`
5. Title: "Listing Volume by Season"

**Column Chart — Listings by Month Period**
1. Insert → **Clustered column chart**.
2. X-axis: `Calendar[Month Period]` (Beginning / Middle / End)
3. Y-axis: `[Total Listings]`
4. Sort X-axis manually: Beginning → Middle → End (add sort order column to Calendar if needed).
5. Title: "Listing Volume by Month Period"

**Slicers:** `Calendar[Year]` | `DimLocation[city]` | `DimPropertyType[property_type]` | `DimOfferingType[offering_type]`

**Layout suggestion:** 2-column grid
- Left column: Monthly Volume line chart (top), Seasonality by month (middle), Seasonality by season (bottom)
- Right column: Monthly Price line chart (top), Day of Week (middle), Month Period (bottom)

---

## Step 10 — Dashboard 6: Time Intelligence Deep Dive

**Page name:** `Time Intelligence` | **Canvas:** 1366 × 768

**Purpose:** Advanced time comparisons — YoY/QoQ/MoM changes, rolling averages, listing velocity, and price momentum over time. This dashboard requires the full Calendar table and the time intelligence measures from Step 4H.

### Row 1 — KPI Snapshot Cards (4 cards)

1. Insert → **Card** × 4:
   - `[Listings YTD]` — label "Listings Year-to-Date"
   - `[Listings PYTD]` — label "Same Period Last Year"
   - `[Listings YTD vs PYTD]` — label "YTD vs PYTD Variance" — add conditional formatting: positive = green, negative = red (Format → Callout value → Conditional formatting by rules)
   - `[Stale Listings %]` — label "Stale Listings (90+ Days) %"

### Row 2 — MoM / QoQ / YoY Change Cards (3 cards)

2. Insert → **Card** × 3:
   - `[Listings MoM Change %]` — label "MoM Volume Change"
   - `[Listings QoQ Change %]` — label "QoQ Volume Change"
   - `[Listings YoY Change %]` — label "YoY Volume Change"
3. Apply conditional formatting on all three: green for positive, red for negative.

### Visual 1 — YoY Comparison Column Chart

1. Insert → **Clustered column chart**.
2. X-axis: `Calendar[Month Short]` — sorted by `Calendar[Month Number]`.
3. Y-axis: `[Total Listings]`
4. Legend: `Calendar[Year]` — this creates side-by-side columns per year per month, showing year-over-year comparison directly.
5. Title: "Monthly Listing Volume — Year-over-Year Comparison"

### Visual 2 — QoQ Listing Volume (Bar Chart)

1. Insert → **Clustered bar chart**.
2. Y-axis: `Calendar[Year-Quarter]`
3. X-axis: `[Total Listings]`
4. Sort descending by `Calendar[Year-Quarter]` to show most recent first.
5. Add Data Labels.
6. Title: "Listing Volume by Quarter"

### Visual 3 — Rolling 3-Month Average vs Actual (Line Chart)

1. Insert → **Line chart**.
2. X-axis: `Calendar[Year-Month Label]` (sorted by `Year-Month`).
3. Y-axis (primary): `[Total Listings]` — line style solid, color `#1A3A5C`.
4. Y-axis (secondary): `[Listings 3M Rolling Avg]` — line style dashed, color `#C9A84C`.
5. Add both measures to the Values field and format them differently in Format → Lines → customize per series.
6. Title: "Actual vs 3-Month Rolling Average — Monthly Listings"

### Visual 4 — Price MoM Change Over Time (Line Chart)

1. Insert → **Line chart**.
2. X-axis: `Calendar[Year-Month Label]` (sorted by `Year-Month`).
3. Y-axis: `[Price MoM Change %]` — format as percentage.
4. Legend: `DimOfferingType[offering_type]` — separate lines for Buy and Rent price changes.
5. Analytics pane → **Constant line** at 0, dashed, labeled "No Change".
6. Title: "Month-over-Month Median Price Change % — Buy vs Rent"

### Visual 5 — Listing Velocity Scatter (Days Since Listed vs Price)

1. Insert → **Scatter chart**.
2. X-axis: `[Avg Days Since Listed]`
3. Y-axis: `[Avg Price Per SQM]`
4. Details: `DimLocation[city]` — one bubble per city.
5. Size: `[Total Listings]`
6. Legend: `DimOfferingType[offering_type]`
7. Title: "Listing Velocity — Days on Market vs Price Per SQM by City"
8. Add text box below chart: "Bubbles upper-left = high price, fast turnover. Bubbles lower-right = low price, slow movement."

### Visual 6 — Stale Listings Trend (Column Chart)

1. Insert → **Clustered column chart**.
2. X-axis: `Calendar[Year-Month Label]` (sorted by `Year-Month`).
3. Y-axis: `[Stale Listings %]` — format as percentage.
4. Analytics pane → **Average line** for reference.
5. Title: "Stale Listings % Over Time (Listings > 90 Days Old)"

### Visual 7 — YTD Cumulative Listings (Area Chart)

1. Insert → **Area chart**.
2. X-axis: `Calendar[Date]` — use Day level granularity within the current year filter.
3. Y-axis: `[Listings YTD]`
4. Legend: `Calendar[Year]` — shows current year vs prior year cumulative lines overlaid.
5. Format → Lines → enable markers at monthly intervals.
6. Title: "Cumulative Listings Year-to-Date vs Prior Year"

### Visual 8 — Listings by Week x Day (Matrix Heatmap)

Power BI does not have a native heatmap, so use a **Matrix** with conditional formatting:
1. Insert → **Matrix**.
2. Rows: `Calendar[Week Number]`
3. Columns: `Calendar[Day Short]` — sort columns by `Calendar[Day of Week Number]`.
4. Values: `[Total Listings]`
5. Format → Conditional formatting on Values → **Background color** → Diverging, lowest = white, highest = `#1A3A5C`.
6. Format → Row/column headers → reduce font to 9pt to fit all 52 weeks.
7. Title: "Listing Volume Heatmap — Week x Day of Week"

**Slicers for this page:**
- `Calendar[Date]` — **Between** style: Insert → Slicer → `Calendar[Date]` → Format → Slicer settings → Style: Between.
- `DimOfferingType[offering_type]` — Dropdown.
- `DimLocation[city]` — Dropdown.
- `DimPropertyType[property_type]` — Dropdown.

**Layout suggestion:**
- Row 1 (top strip): 4 KPI cards + 3 change cards, compact height
- Row 2: YoY comparison column chart (left 50%) | QoQ bar chart (right 50%)
- Row 3: Rolling avg line (left 50%) | Price MoM change line (right 50%)
- Row 4: Listing velocity scatter (left 33%) | Stale listings column (middle 33%) | Heatmap matrix (right 33%)
- Row 5 (full width): YTD cumulative area chart

---

## Step 11 — Formatting Rules (All Pages)

1. **Theme:** Home → Switch Theme → apply "Executive" or a custom JSON theme. Primary: `#1A3A5C`, Accent: `#C9A84C`, Background: white or `#F5F5F5`.
2. **Chart titles:** Format → Title → 12pt bold, consistent color.
3. **Hide surrogate keys:** Data pane → right-click `city_key`, `property_type_key`, etc. → **Hide in report view**.
4. **Slicer sync:** View → **Sync Slicers** → sync `city` and `offering_type` slicers across all 6 pages.
5. **Tooltips:** Format → Tooltip → add 2-3 additional measures per visual beyond defaults.
6. **Interactions:** Format → **Edit interactions** after selecting each slicer → prevent slicers from filtering each other.
7. **Page navigation:** Insert → **Buttons → Navigator → Page navigator** on each page for easy switching.
8. **Conditional formatting:** Apply to KPI change cards (MoM, QoQ, YoY) — positive = green, negative = red.

---

## Publish Checklist

- [ ] All 6 dashboard pages created and named correctly
- [ ] Star schema verified in Model view — no yellow warning triangles on relationships
- [ ] Calendar table marked as Date Table on `Date` column
- [ ] All time intelligence measures reference `Calendar[Date]` correctly
- [ ] `_Measures` table holds all measures — none floating in `FactListings`
- [ ] `AmenityPriceSummary` calculated table tested and returns 20 rows
- [ ] YoY column chart shows correct year grouping (2024, 2025, 2026 as separate series)
- [ ] Heatmap matrix conditional formatting applied and visible
- [ ] Stale listing measures use `listing_age_days` column (not recalculated)
- [ ] All slicers use Dropdown style
- [ ] `City` + `Offering Type` slicers synced across all pages via Sync Slicers
- [ ] Page navigator buttons added to all pages
- [ ] Tooltips customized on map, scatter, and matrix visuals
- [ ] All surrogate key columns hidden from report view
- [ ] File saved as `.pbip` (File → Save as → Power BI Project)
