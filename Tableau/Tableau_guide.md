# Tableau — Egypt Real Estate 2026
## Role & Context

You are a senior Tableau developer and visual analytics specialist.
Build a Tableau workbook (`.twbx`) from the Egypt Real Estate 2026 base dataset only.
No ML, NLP, or forecast files are required.

**Division of work with Power BI:**
Power BI covers: Market Overview, Location pricing, Property Specs vs Price, Agent Performance, Monthly Trends, YoY/QoQ/Rolling time intelligence.
Tableau covers: Amenity intelligence, geospatial compound-level exploration, rental yield analysis, and **temporal patterns at granular levels** (day-of-week heatmaps, quarterly seasonality, listing velocity over time, price momentum by city over time). No overlap.

---

## Dashboard Map (3 Dashboards)

| Dashboard | Title | Focus |
|---|---|---|
| 1 | Amenity Intelligence | Co-occurrence, price premiums, furnished × amenity scatter |
| 2 | Geospatial & Yield Explorer | Point map, subdistrict ranking, buy vs rent by district, yield by town |
| 3 | Temporal Patterns & Price Momentum | Time heatmaps, quarterly price trends, listing velocity, city price trajectories |

---

## Source Files to Connect

| Data Source Name | File | Notes |
|---|---|---|
| `MainListings` | `egypt_re_clean.csv` | Primary — all listing columns |
| `Amenities` | `egypt_re_amenities_binary.csv` | Join on `listing_id` |

---

## Step 1 — Connect & Configure Data

### 1A. Connect MainListings

1. Open Tableau Desktop → **Connect → To a File → Text file** → select `egypt_re_clean.csv`.
2. In the Data Source tab, rename the connection to `MainListings`.
3. Drag the CSV sheet into the canvas area.

### 1B. Join Amenities

1. Click **Add** next to the connection name → **Text file** → select `egypt_re_amenities_binary.csv`.
2. Drag it into the canvas beside the main sheet.
3. Click the join circle → set type to **Left**. Join condition: `[listing_id] = [listing_id (Amenities)]`.
4. Verify: row count must equal `egypt_re_clean.csv` exactly. If it inflates, check for duplicate `listing_id` in the amenities file and deduplicate.

### 1C. Set Field Data Types

In the Data Source tab, click each column header to set type:

| Column | Type |
|---|---|
| `listing_id` | String |
| `price_egp`, `price_per_sqm`, `area_value`, `lat`, `lon` | Number (decimal) |
| `bedrooms`, `bathrooms`, `amenity_count`, `images_count`, `days_since_listed`, `listing_age_days` | Number (whole) |
| `listed_date` | Date |
| `scraped_at` | Date & Time |
| `is_premium`, `is_verified`, `is_furnished_binary`, all `amen_*` cols | Number (whole) |
| `city`, `town`, `district`, `subdistrict`, `property_type`, `offering_type`, `furnished`, `agent_name`, `broker_name` | String |

### 1D. Set Geographic Roles

1. Right-click `lat` → **Geographic Role → Latitude**.
2. Right-click `lon` → **Geographic Role → Longitude**.
3. Right-click `city` → **Geographic Role → City**.

---

## Step 2 — Custom Color Palette

1. Close Tableau Desktop.
2. Open `~/Documents/My Tableau Repository/Preferences.tps` in any text editor.
3. Add inside the `<preferences>` tag:

```xml
<color-palette name="EgyptRE PropertyType" type="regular">
  <color>#1A3A5C</color>
  <color>#C9A84C</color>
  <color>#2E8B6A</color>
  <color>#B03A3A</color>
  <color>#5A4A8A</color>
  <color>#2D8BBB</color>
  <color>#E06030</color>
  <color>#6B6B6B</color>
</color-palette>

<color-palette name="EgyptRE Yield" type="ordered-diverging">
  <color>#2166AC</color>
  <color>#92C5DE</color>
  <color>#F7F7F7</color>
  <color>#F4A582</color>
  <color>#D6604D</color>
</color-palette>

<color-palette name="EgyptRE Temporal" type="ordered-sequential">
  <color>#EFF3FF</color>
  <color>#BDD7E7</color>
  <color>#6BAED6</color>
  <color>#2171B5</color>
  <color>#084594</color>
</color-palette>
```

4. Save and reopen Tableau. All three palettes appear in the color picker under "Custom".

---

## Step 3 — All Calculated Fields

Create all fields via **Analysis → Create Calculated Field** before building any sheet.

### 3A — Core Numeric

```
Price Per SQM
[price_egp] / [area_value]

Log Price
LOG([price_egp])

Log Area
LOG([area_value])
```

### 3B — Date & Time Dimensions

```
Month Number
MONTH([listed_date])

Month Name
DATENAME('month', [listed_date])

Month-Year Label
DATENAME('month', [listed_date]) + " " + STR(YEAR([listed_date]))

Month-Year Sort Key
YEAR([listed_date]) * 100 + MONTH([listed_date])

Quarter Number
DATEPART('quarter', [listed_date])

Quarter-Year Label
"Q" + STR(DATEPART('quarter', [listed_date])) + " " + STR(YEAR([listed_date]))

Quarter-Year Sort Key
YEAR([listed_date]) * 10 + DATEPART('quarter', [listed_date])

Day of Week Name
DATENAME('weekday', [listed_date])

Day of Week Number
DATEPART('weekday', [listed_date])

Week Number
DATEPART('week', [listed_date])

Year
YEAR([listed_date])

Season
IF MONTH([listed_date]) IN (12, 1, 2) THEN "Winter"
ELSEIF MONTH([listed_date]) IN (3, 4, 5) THEN "Spring"
ELSEIF MONTH([listed_date]) IN (6, 7, 8) THEN "Summer"
ELSE "Autumn"
END

Month Period
IF DAY([listed_date]) <= 10 THEN "Beginning"
ELSEIF DAY([listed_date]) <= 20 THEN "Middle"
ELSE "End"
END
```

### 3C — Price Bands

```
Price Band Buy
IF [offering_type] = "Buy" THEN
  IF [price_egp] < 5000000 THEN "Under 5M"
  ELSEIF [price_egp] < 15000000 THEN "5M-15M"
  ELSE "Above 15M"
  END
ELSE NULL
END

Price Band Rent
IF [offering_type] = "Rent" THEN
  IF [price_egp] < 10000 THEN "Under 10K"
  ELSEIF [price_egp] < 50000 THEN "10K-50K"
  ELSE "Above 50K"
  END
ELSE NULL
END
```

### 3D — Rental Yield (LOD Expressions)

```
Buy Median Price by City
{ FIXED [city] : MEDIAN(IF [offering_type]="Buy" THEN [price_egp] END) }

Rent Median Price by City
{ FIXED [city] : MEDIAN(IF [offering_type]="Rent" THEN [price_egp] END) }

Annual Yield by City %
([Rent Median Price by City] * 12) / [Buy Median Price by City] * 100

Buy Median Price by Town
{ FIXED [city], [town] : MEDIAN(IF [offering_type]="Buy" THEN [price_egp] END) }

Rent Median Price by Town
{ FIXED [city], [town] : MEDIAN(IF [offering_type]="Rent" THEN [price_egp] END) }

Annual Yield by Town %
([Rent Median Price by Town] * 12) / [Buy Median Price by Town] * 100

Buy Count by Town
{ FIXED [city], [town] : SUM(IF [offering_type]="Buy" THEN 1 ELSE 0 END) }

Rent Count by Town
{ FIXED [city], [town] : SUM(IF [offering_type]="Rent" THEN 1 ELSE 0 END) }
```

### 3E — Temporal Analysis Fields

```
Listings This Month LOD
{ FIXED YEAR([listed_date]), MONTH([listed_date]) : COUNTD([listing_id]) }

Monthly Median Price SQM
{ FIXED YEAR([listed_date]), MONTH([listed_date]) : MEDIAN([price_per_sqm]) }

Quarterly Median Price SQM
{ FIXED YEAR([listed_date]), DATEPART('quarter',[listed_date]) : MEDIAN([price_per_sqm]) }

Is Stale Listing
[listing_age_days] > 90

Listing Age Band
IF [listing_age_days] <= 30 THEN "0-30 days"
ELSEIF [listing_age_days] <= 60 THEN "31-60 days"
ELSEIF [listing_age_days] <= 90 THEN "61-90 days"
ELSEIF [listing_age_days] <= 180 THEN "91-180 days"
ELSE "180+ days"
END

Buy Price SQM Chart
IF [offering_type] = "Buy" THEN [price_per_sqm] END

Rent Price SQM Chart
IF [offering_type] = "Rent" THEN [price_per_sqm] END
```

### 3F — Amenity Per-Listing Fields

```
Price With Pool
IF [amen_shared_pool] = 1 THEN [price_per_sqm] END

Price Without Pool
IF [amen_shared_pool] = 0 THEN [price_per_sqm] END
```

**Create these 20 amenity-vs-price calculated fields** (one per amenity) for Sheet 1B:

```
Avg SQM Balcony:           AVG(IF [amen_balcony]=1 THEN [price_per_sqm] END)
Avg SQM Security:          AVG(IF [amen_security]=1 THEN [price_per_sqm] END)
Avg SQM Covered Parking:   AVG(IF [amen_covered_parking]=1 THEN [price_per_sqm] END)
Avg SQM Landmark View:     AVG(IF [amen_view_of_landmark]=1 THEN [price_per_sqm] END)
Avg SQM Shared Gym:        AVG(IF [amen_shared_gym]=1 THEN [price_per_sqm] END)
Avg SQM Lobby:             AVG(IF [amen_lobby_in_building]=1 THEN [price_per_sqm] END)
Avg SQM Central AC:        AVG(IF [amen_central_a_c]=1 THEN [price_per_sqm] END)
Avg SQM Kitchen Appl:      AVG(IF [amen_kitchen_appliances]=1 THEN [price_per_sqm] END)
Avg SQM Wardrobes:         AVG(IF [amen_built_in_wardrobes]=1 THEN [price_per_sqm] END)
Avg SQM Shared Spa:        AVG(IF [amen_shared_spa]=1 THEN [price_per_sqm] END)
Avg SQM Shared Pool:       AVG(IF [amen_shared_pool]=1 THEN [price_per_sqm] END)
Avg SQM Walk-in Closet:    AVG(IF [amen_walk_in_closet]=1 THEN [price_per_sqm] END)
Avg SQM Study:             AVG(IF [amen_study]=1 THEN [price_per_sqm] END)
Avg SQM Childrens Pool:    AVG(IF [amen_children_s_pool]=1 THEN [price_per_sqm] END)
Avg SQM Water View:        AVG(IF [amen_view_of_water]=1 THEN [price_per_sqm] END)
Avg SQM Private Garden:    AVG(IF [amen_private_garden]=1 THEN [price_per_sqm] END)
Avg SQM Maids Room:        AVG(IF [amen_maids_room]=1 THEN [price_per_sqm] END)
Avg SQM Private Pool:      AVG(IF [amen_private_pool]=1 THEN [price_per_sqm] END)
Avg SQM Networked:         AVG(IF [amen_networked]=1 THEN [price_per_sqm] END)
Avg SQM Dining Building:   AVG(IF [amen_dining_in_building]=1 THEN [price_per_sqm] END)
```

---

## Step 4 — Dashboard 1: Amenity Intelligence

**Purpose:** Which amenities co-occur, which drive the highest price premiums, and how furnished status + amenity count interact with pricing.

### Sheet 1A — Amenity Presence vs Median Price (Horizontal Bar)

1. New sheet → rename `Amenity vs Price`.
2. Drag `Measure Names` to Rows.
3. Drag `Measure Values` to Columns.
4. In the Measure Values card, keep only the 20 `Avg SQM *` calculated fields. Remove all others.
5. Mark type: **Bar**.
6. Sort Rows: right-click `Measure Names` on Rows shelf → Sort → Descending by field → `Measure Values`.
7. Create a calculated field `Is Luxury Amenity` to color bars:
   ```
   [Measure Names] = "Avg SQM Private Pool"
   OR [Measure Names] = "Avg SQM Private Garden"
   OR [Measure Names] = "Avg SQM Shared Spa"
   OR [Measure Names] = "Avg SQM Walk-in Closet"
   OR [Measure Names] = "Avg SQM Study"
   OR [Measure Names] = "Avg SQM Water View"
   OR [Measure Names] = "Avg SQM Childrens Pool"
   ```
8. Drag `Is Luxury Amenity` to **Color**. True = `#C9A84C` (gold), False = `#2D8BBB` (blue).
9. Drag `Measure Values` to **Label**. Format as integer (#,###).
10. Clean up axis: right-click X-axis → Edit Axis → title "Avg Price Per SQM (EGP)".
11. Edit aliases for Measure Names: right-click in Data pane → Aliases → rename each to a clean label (e.g., "Private Pool", "Shared Spa").
12. Title: "Amenities Ranked by Avg Price Per SQM"

### Sheet 1B — Amenity Count vs Price Scatter

1. New sheet → rename `Amenity Count vs Price`.
2. Columns: `amenity_count` — right-click on shelf → **Dimension**.
3. Rows: `AVG([price_per_sqm])`.
4. Detail: drag `listing_id` to Detail, or use `city` for one point per city-amenity-count combination if too dense.
5. Color: `furnished` → Furnished = `#2E8B6A`, Unfurnished = `#B03A3A`, Semi-Furnished = `#C9A84C`.
6. Size: `AVG([area_value])`.
7. Mark type: **Circle**. Opacity: 70%.
8. Filters: Add `city` and `property_type` as quick filters.
9. Edit Tooltip:
   ```
   Amenity Count: <amenity_count>
   Avg Price/SQM: <AVG(price_per_sqm)>
   Furnished: <furnished>
   Avg Area: <AVG(area_value)> sqm
   City: <city>
   ```
10. Title: "Price Per SQM by Amenity Count & Furnished Status"

### Sheet 1C — Amenity Co-occurrence (Heatmap via Parameter)

**Create the parameter:**
1. Data pane → right-click → **Create Parameter**.
2. Name: `Select Base Amenity`. Type: String. Allowable values: List.
3. Add values: Balcony, Security, Covered Parking, Landmark View, Shared Gym, Lobby, Central AC, Kitchen Appliances, Wardrobes, Shared Spa, Shared Pool, Walk-in Closet, Study, Childrens Pool, Water View, Private Garden, Maids Room, Private Pool.

**Create the calculated field:**
```
Base Amenity Flag
IF [Select Base Amenity] = "Balcony" THEN [amen_balcony]
ELSEIF [Select Base Amenity] = "Security" THEN [amen_security]
ELSEIF [Select Base Amenity] = "Covered Parking" THEN [amen_covered_parking]
ELSEIF [Select Base Amenity] = "Landmark View" THEN [amen_view_of_landmark]
ELSEIF [Select Base Amenity] = "Shared Gym" THEN [amen_shared_gym]
ELSEIF [Select Base Amenity] = "Lobby" THEN [amen_lobby_in_building]
ELSEIF [Select Base Amenity] = "Central AC" THEN [amen_central_a_c]
ELSEIF [Select Base Amenity] = "Kitchen Appliances" THEN [amen_kitchen_appliances]
ELSEIF [Select Base Amenity] = "Wardrobes" THEN [amen_built_in_wardrobes]
ELSEIF [Select Base Amenity] = "Shared Spa" THEN [amen_shared_spa]
ELSEIF [Select Base Amenity] = "Shared Pool" THEN [amen_shared_pool]
ELSEIF [Select Base Amenity] = "Walk-in Closet" THEN [amen_walk_in_closet]
ELSEIF [Select Base Amenity] = "Study" THEN [amen_study]
ELSEIF [Select Base Amenity] = "Childrens Pool" THEN [amen_children_s_pool]
ELSEIF [Select Base Amenity] = "Water View" THEN [amen_view_of_water]
ELSEIF [Select Base Amenity] = "Private Garden" THEN [amen_private_garden]
ELSEIF [Select Base Amenity] = "Maids Room" THEN [amen_maids_room]
ELSEIF [Select Base Amenity] = "Private Pool" THEN [amen_private_pool]
ELSE 0
END
```

**Build the sheet:**
1. New sheet → rename `Amenity Co-occurrence`.
2. Rows: `Measure Names`. Keep only the 20 `Avg SQM *` measures in Measure Values.
3. Columns: `SUM(IF [Base Amenity Flag]=1 THEN 1 ELSE 0 END)` — shows how many listings with the selected base amenity also have each row amenity.
4. Mark type: **Square**.
5. Color: `Measure Values` → EgyptRE Temporal palette (more co-occurrences = darker blue).
6. Show the `Select Base Amenity` parameter control on the dashboard.
7. Title: "Co-occurrence — Listings Also Having [Select Base Amenity]" (use Insert → Parameter in title).

### Dashboard 1 Assembly — "Amenity Intelligence"

1. New Dashboard → Size: Fixed 1366 × 768.
2. Layout:
   - Left panel (40% width, full height): Sheet 1A (Amenity vs Price ranking bar).
   - Top right (60% width, 45% height): Sheet 1C (Co-occurrence heatmap) + parameter control.
   - Bottom right (60% width, 55% height): Sheet 1B (Amenity count scatter).
3. Title text object: "Amenity Intelligence & Lifestyle Analysis", 18pt bold, `#C9A84C`.
4. **Filter action:** Dashboard → Actions → Add Action → Filter.
   - Source: Sheet 1A. Target: Sheet 1B. Run on: Select. Clearing: Show all values.
5. **Global quick filters:** Add `city` and `property_type` from Sheet 1A, apply to all sheets.
6. Show `Select Base Amenity` parameter on dashboard.

---

## Step 5 — Dashboard 2: Geospatial & Yield Explorer

**Purpose:** Point-level listing map, compound/subdistrict price ranking, and rental yield by town.

### Sheet 2A — Listing Point Map

1. New sheet → rename `Listing Point Map`.
2. Double-click `lat` → Rows. Double-click `lon` → Columns.
3. Mark type: **Circle**.
4. Color: `AVG([price_per_sqm])` → Custom Diverging → low `#2E8B6A`, high `#B03A3A`. Fix range: 0 to 2000.
5. Size: `AVG([area_value])`.
6. Tooltip:
   ```
   Compound: <subdistrict>
   Price: EGP <price_egp>
   Price/SQM: <price_per_sqm>
   Type: <property_type> | <offering_type>
   Bedrooms: <bedrooms>
   Furnished: <furnished>
   Agent: <agent_name>
   City: <city>
   ```
7. Map background: Map menu → **Map Layers** → Style: Dark.
8. Filters: `city`, `offering_type`, `property_type` as quick filters. `bedrooms` as range filter.
9. Title: "Listings by Coordinate — Price Per SQM"

### Sheet 2B — Subdistrict Price Ranking (Horizontal Bar)

1. New sheet → rename `Compound Ranking`.
2. Rows: `subdistrict`. Columns: `MEDIAN([price_per_sqm])`.
3. Mark type: **Bar**.
4. Filter: `offering_type = "Buy"`.
5. Top N filter on `subdistrict`: Top 30 by `MEDIAN([price_per_sqm])`.
6. Sort: descending by `MEDIAN([price_per_sqm])`.
7. Color: `city` → EgyptRE PropertyType palette.
8. Drag `COUNTD([listing_id])` to Label. Edit label: "n=<value>".
9. Analytics pane → drag **Reference Line** → Table → Value: `MEDIAN([price_per_sqm])` → Label: "Market Median" → dashed, `#C9A84C`.
10. Title: "Top 30 Compounds by Median Price Per SQM (Buy)"

### Sheet 2C — Rental Yield by Town (Bubble Map)

1. New sheet → rename `Yield by Town`.
2. Columns: `AVG([lon])`. Rows: `AVG([lat])`.
3. Detail: drag `city` and `town` to Detail shelf.
4. Mark type: **Circle**.
5. Color: `AVG([Annual Yield by Town %])` → EgyptRE Yield palette. Center at 5.
6. Size: `COUNTD([listing_id])`.
7. Filter: `[Buy Count by Town] >= 10 AND [Rent Count by Town] >= 10`.
8. Tooltip:
   ```
   Town: <town> (<city>)
   Buy Median Price: EGP <Buy Median Price by Town>
   Rent Median/Month: EGP <Rent Median Price by Town>
   Annual Yield: <Annual Yield by Town %>%
   Listing Count: <COUNTD(listing_id)>
   ```
9. Title: "Rental Yield % by Town (min. 10 Buy + 10 Rent listings)"

### Sheet 2D — Buy vs Rent Price Per SQM by District (Dual-Axis Bar)

1. New sheet → rename `Buy vs Rent by District`.
2. Rows: `district`. Columns: `MEDIAN([Buy Price SQM Chart])` and `MEDIAN([Rent Price SQM Chart])`.
3. Right-click second pill on Columns → **Dual Axis**. Right-click right axis → **Synchronize Axis**.
4. Mark type for both axes: **Bar**.
5. Color Buy bars: `#1A3A5C`. Color Rent bars: `#C9A84C`.
6. Top N filter on `district`: Top 20 by `MEDIAN([Buy Price SQM Chart])`.
7. Sort `district` descending by Buy price.
8. Labels on: format as integer.
9. Title: "Buy vs Rent Price Per SQM by District (Top 20)"

### Dashboard 2 Assembly — "Geospatial & Yield Explorer"

1. New Dashboard → Size: Fixed 1366 × 768.
2. Layout:
   - Left panel (50% width, full height): Sheet 2A (Point map).
   - Right top (50% width, 30% height): Sheet 2C (Yield by town).
   - Right middle (50% width, 40% height): Sheet 2B (Compound ranking bar).
   - Right bottom (50% width, 30% height): Sheet 2D (Buy vs Rent by district).
3. Title: "Geospatial & Yield Explorer", 18pt bold, `#C9A84C`.
4. **Filter Action — Map to right panels:**
   - Source: Sheet 2A. Target: Sheets 2B, 2C, 2D. Run on: Select. Field: `city`.
5. **Highlight Action — Compound ranking to map:**
   - Source: Sheet 2B. Target: Sheet 2A. Fields: `subdistrict`. Run on: Select.
6. Global quick filters: `city`, `offering_type`, `property_type` — apply to all sheets.

---

## Step 6 — Dashboard 3: Temporal Patterns & Price Momentum

**Purpose:** Granular time-based analysis — heatmaps, quarterly seasonality, listing velocity, and city-level price trajectories. This is not covered in Power BI.

### Sheet 3A — Activity Heatmap: Month x Day of Week

1. New sheet → rename `Month x Day Heatmap`.
2. Columns: `Day of Week Name` — sort ascending by `Day of Week Number`.
3. Rows: `Month Name` — sort ascending by `Month Number`.
4. Mark type: **Square**.
5. Color: `COUNTD([listing_id])` → EgyptRE Temporal palette (more listings = darker).
6. Label: `COUNTD([listing_id])` — font 9pt, white.
7. Remove row/column dividers: Format → Borders → set all to None.
8. Tooltip:
   ```
   Month: <Month Name>
   Day: <Day of Week Name>
   Listings: <COUNTD(listing_id)>
   ```
9. Title: "Listing Activity Heatmap — Month x Day of Week"

### Sheet 3B — Quarterly Price Trend (Dual-Axis Line)

1. New sheet → rename `Quarterly Price Trend`.
2. Columns: `Quarter-Year Label` — sort ascending by `Quarter-Year Sort Key`.
3. Rows: `MEDIAN([Buy Price SQM Chart])` and `MEDIAN([Rent Price SQM Chart])`.
4. Right-click second pill → **Dual Axis**. Synchronize axes.
5. Mark type for both: **Line** with circle markers.
6. Color: Buy line = `#1A3A5C`, Rent line = `#C9A84C`.
7. Analytics pane → Reference Line → Table-scoped → Median → label "Buy Median" → dashed.
8. Tooltip:
   ```
   Quarter: <Quarter-Year Label>
   Buy Price/SQM: <MEDIAN(Buy Price SQM Chart)>
   Rent Price/SQM: <MEDIAN(Rent Price SQM Chart)>
   ```
9. Title: "Quarterly Median Price Per SQM — Buy vs Rent"

### Sheet 3C — City Price Trajectories Over Time (Multi-Line)

1. New sheet → rename `City Price Trajectories`.
2. Columns: `Month-Year Label` — sort ascending by `Month-Year Sort Key`.
3. Rows: `MEDIAN([price_per_sqm])`.
4. Color: `city` → EgyptRE PropertyType palette.
5. Mark type: **Line**.
6. Filter: `offering_type = "Buy"`. Add `city` as a multi-select quick filter.
7. Analytics pane → Reference Band → 25th to 75th percentile.
8. Label: End-of-line labels only (Format → Mark Labels → Last).
9. Tooltip:
   ```
   City: <city>
   Month: <Month-Year Label>
   Median Price/SQM: <MEDIAN(price_per_sqm)>
   ```
10. Title: "Monthly Buy Price Per SQM Trajectory by City"

### Sheet 3D — Listing Age Band Distribution Over Time (Stacked Bar)

1. New sheet → rename `Listing Age Distribution`.
2. Columns: `Quarter-Year Label` — sort by `Quarter-Year Sort Key`.
3. Rows: `COUNTD([listing_id])`.
4. Color: `Listing Age Band` → 0-30 days = `#2E8B6A`, 31-60 = `#C9A84C`, 61-90 = `#E06030`, 91-180 = `#B03A3A`, 180+ = `#5A4A8A`.
5. Mark type: **Bar** (stacked).
6. Label: percent of total per bar — right-click measure on Rows → Add Table Calculation → Percent of Total → Scope: Pane. Format as "X%".
7. Tooltip:
   ```
   Quarter: <Quarter-Year Label>
   Listing Age Band: <Listing Age Band>
   Count: <COUNTD(listing_id)>
   ```
8. Title: "Listing Age Distribution by Quarter (How Long Listings Stay Active)"

### Sheet 3E — Listing Velocity: Days Since Listed vs Price (Animated Scatter)

1. New sheet → rename `Listing Velocity`.
2. Columns: `AVG([days_since_listed])`.
3. Rows: `AVG([price_per_sqm])`.
4. Detail: `city` — one mark per city.
5. Color: `city` → EgyptRE PropertyType palette.
6. Size: `COUNTD([listing_id])`.
7. Add `Quarter-Year Label` to **Pages** shelf — this creates an animated chart. Click Play to animate city bubbles moving over time. Set speed: medium.
8. Mark type: **Circle**. Opacity: 80%.
9. Analytics pane → Reference Line on X-axis at `AVG([days_since_listed])` → label "Market Avg Days" → dashed.
10. Reference Line on Y-axis at `AVG([price_per_sqm])` → label "Market Avg Price" → dashed.
11. Tooltip:
    ```
    City: <city>
    Quarter: <Quarter-Year Label>
    Avg Days Since Listed: <AVG(days_since_listed)>
    Avg Price/SQM: <AVG(price_per_sqm)>
    Listing Count: <COUNTD(listing_id)>
    ```
12. Title: "Listing Velocity Over Time — Days on Market vs Price Per SQM"

### Sheet 3F — Season & Month Period Bar Chart (Grouped)

1. New sheet → rename `Season & Period Analysis`.
2. Columns: `Season` — manual sort: Winter, Spring, Summer, Autumn.
3. Rows: `COUNTD([listing_id])`.
4. Color: `Month Period` → Beginning = `#2166AC`, Middle = `#6BAED6`, End = `#BDD7E7`.
5. Mark type: **Bar** (grouped).
6. Label: count.
7. Add `offering_type` as quick filter.
8. Title: "Listing Volume by Season & Month Period"

### Dashboard 3 Assembly — "Temporal Patterns & Price Momentum"

1. New Dashboard → Size: Fixed 1366 × 768.
2. Layout:
   - Row 1 (top half): Sheet 3A left (50%) | Sheet 3B right (50%)
   - Row 2 (middle): Sheet 3C left (50%, include `city` quick filter) | Sheet 3D right (50%)
   - Row 3 (bottom): Sheet 3E left (50%, include Pages control) | Sheet 3F right (50%)
3. Title: "Temporal Patterns & Price Momentum", 18pt bold, `#C9A84C`.
4. **Filter Action — Heatmap to trajectories:**
   - Source: Sheet 3A. Target: Sheets 3C, 3D. Run on: Select. Field: `Month Name`.
5. **Filter Action — Quarterly trend to age distribution:**
   - Source: Sheet 3B. Target: Sheet 3D. Run on: Select. Field: `Quarter-Year Label`.
6. Global quick filters: `offering_type`, `city`, `property_type` — apply to all sheets.
7. Show the Pages control for Sheet 3E as a floating object near the scatter chart.

---

## Step 7 — Formatting Rules (All Dashboards)

### Per-Sheet Formatting
1. Format → **Shading** → Worksheet: `#F5F5F5` (light). Apply consistently across all sheets.
2. Format → **Font** → all text: 10pt, Tableau Book or Tableau Regular.
3. Format → **Lines** → Grid lines: light gray `#DDDDDD`. Zero line: `#999999`.
4. All axes: right-click → **Edit Axis** → write a clean human-readable title. Remove auto-generated field names.
5. Remove placeholder "Abc" marks: uncheck Show Mark Labels unless intentional.

### Tooltip Rules
- Never show raw field names. Prefix each value: "City:", "Price:", "Listings:", etc.
- Remove "Sheet" prefix from all tooltip titles.

### Dashboard Layout Rules
- Fixed size 1366 × 768 on all three dashboards.
- Use **Layout containers** (horizontal/vertical) for all sheet placement. Use floating only for the Pages control on Dashboard 3.
- Padding between sheets: 4px (Dashboard → Layout pane → Outer padding).
- Dashboard title: Text object, 18pt bold, color `#C9A84C`.
- All filter controls: right-click → Single Value Dropdown.
- Legend placement: bottom of dashboard or inside a tiled container on the right.

---

## Step 8 — Publish Checklist

- [ ] Data join verified: row count after join equals row count of `egypt_re_clean.csv`
- [ ] Geographic roles set on `lat` (Latitude) and `lon` (Longitude) — maps render correctly
- [ ] All LOD expressions tested: apply a city filter and verify yield measures recalculate
- [ ] All 20 `Avg SQM *` calculated fields created and appear in Sheet 1A
- [ ] `Select Base Amenity` parameter tested — co-occurrence updates for all 18 values
- [ ] Quarterly price dual-axis chart shows two synchronized axes without overlap
- [ ] Animated scatter (Sheet 3E) plays correctly through quarterly frames
- [ ] Month x Day heatmap displays all 7 days x 12 months in correct order
- [ ] City trajectories chart shows distinct lines per city
- [ ] Season & Month Period chart shows 4 seasons x 3 periods = 12 bar groups
- [ ] All tooltips formatted — no raw field names visible on any sheet
- [ ] Dashboard actions tested: click source → target filters correctly, clearing resets
- [ ] Custom color palettes loaded from `Preferences.tps` — EgyptRE palettes visible in color picker
- [ ] Pages control visible and functional on Dashboard 3
- [ ] No insights duplicated from Power BI (no market overview, no agent charts, no YoY column charts)
- [ ] Workbook saved as `.twbx` (File → Export Packaged Workbook) with embedded data extract
