# 🏠 Egypt Real Estate 2026 — End-to-End Analytics Project

> A full-stack data analytics project on **39,713 property listings** scraped from PropertyFinder Egypt (March 2026), covering data engineering, exploratory analysis, NLP, machine learning, and BI dashboards in both Power BI and Tableau.

---

## 📑 Table of Contents

- [Project Overview](#project-overview)
- [Dataset](#dataset)
- [Repository Structure](#repository-structure)
- [Pipeline Phases](#pipeline-phases)
  - [Phase 1 — Data Cleaning](#phase-1--data-cleaning)
  - [Phase 2 — Exploratory Data Analysis](#phase-2--exploratory-data-analysis)
  - [Phase 3 — Outlier Detection, Correlation & Machine Learning](#phase-3--outlier-detection-correlation--machine-learning)
  - [Phase 4 — NLP Analysis](#phase-4--nlp-analysis)
- [BI Dashboards](#bi-dashboards)
  - [Power BI](#power-bi)
  - [Tableau](#tableau)
- [Output Files](#output-files)
- [Setup & Installation](#setup--installation)
- [Libraries & Dependencies](#libraries--dependencies)
- [Design Decisions](#design-decisions)

---

## Project Overview

This project builds a complete analytics pipeline on Egypt's real estate market, from raw CSV ingestion through to interactive BI dashboards. It answers questions like:

- Which cities and property types command the highest price per sqm?
- What amenities drive the most significant price premiums?
- Do listing descriptions have measurable sentiment patterns tied to price?
- Can we predict listing prices and classify Buy vs Rent reliably?

The work is split across **4 Jupyter notebooks** (Python pipeline) and **2 BI tools** (Power BI + Tableau), with a clean division of analytical responsibility between the tools.

---

## Dataset

| Attribute | Detail |
|---|---|
| **Source** | PropertyFinder Egypt (scraped March 2026) |
| **Size** | 39,713 rows × 53 columns |
| **Coverage** | Cairo, Giza, North Coast, Red Sea, Suez, Alexandria |
| **Offering Types** | Buy and Rent |
| **Key Columns** | `listing_id`, `price_egp`, `area_value`, `bedrooms`, `bathrooms`, `city`, `town`, `district`, `subdistrict`, `property_type`, `offering_type`, `furnished`, `amenities`, `description`, `lat`, `lon`, `listed_date`, `scraped_at`, `agent_id`, `broker_name` |

---

## Repository Structure

```
egypt-real-estate-2026/
│
├── data/
│   └── egypt_re_raw.csv                  # Original scraped dataset (not versioned)
│
├── notebooks/
│   ├── 01_cleaning.ipynb                 # Phase 1 — Data cleaning & feature engineering
│   ├── 02_eda.ipynb                      # Phase 2 — Exploratory data analysis
│   ├── 03_outlier_ml.ipynb               # Phase 3 — Outlier detection, correlation & ML
│   └── 04_nlp.ipynb                      # Phase 4 — NLP on descriptions
│
├── outputs/
│   ├── egypt_re_clean.csv
│   ├── egypt_re_no_outliers.csv
│   ├── egypt_re_amenities_binary.csv
│   ├── egypt_re_nlp.csv
│   ├── egypt_re_predictions.csv
│   ├── feature_importance.csv
│   └── eda_summary_stats.csv
│
├── eda_outputs/                          # All EDA charts saved as PNG
│   └── *.png
│
├── bi/
│   ├── egypt_re_2026.pbip                # Power BI Project file
│   └── egypt_re_2026.twbx               # Tableau Packaged Workbook
│
├── docs/
│   ├── 01_python_guide.md               # Python pipeline specification
│   ├── Powerbi_guide.md                 # Power BI build guide
│   └── Tableau_guide.md                 # Tableau build guide
│
├── requirements.txt
└── README.md
```

---

## Pipeline Phases

### Phase 1 — Data Cleaning

**Notebook:** `01_cleaning.ipynb`  
**Output:** `egypt_re_clean.csv`, `egypt_re_amenities_binary.csv`

The first notebook establishes the analysis-ready dataset:

- **Null audit** — columns with >40% nulls are documented and either dropped or imputed with justified decisions
- **Price outlier removal** — IQR method applied *separately* for Buy and Rent listings to avoid cross-contamination of distributions
- **Type coercion** — `bedrooms` and `bathrooms` are stored as object strings; parsed to numeric with special handling for "Studio" → 0, "10+" → 10, etc.
- **Area validation** — `area_value` confirmed in sqm via `area_unit`; flags raised for values <10 or >50,000 sqm
- **Date parsing** — `listed_date` and `scraped_at` converted to datetime; `year`, `month`, `week`, `day_of_week` extracted
- **Furnished normalization** — collapsed to exactly 3 canonical values: Furnished / Unfurnished / Semi-Furnished
- **Amenities parsing** — stored as string-encoded lists; parsed to real lists per row; top-20 amenities expanded to binary flag columns saved in `egypt_re_amenities_binary.csv`
- **Deduplication** — on `listing_id`, keeping the most recent `scraped_at`

**Engineered columns added:**

| Column | Formula |
|---|---|
| `price_per_sqm` | `price_egp / area_value` |
| `bedroom_bathroom_ratio` | `bedrooms / bathrooms` |
| `days_since_listed` | `scraped_at - listed_date` |
| `is_furnished_binary` | 1 if Furnished else 0 |
| `amenity_count` | count of amenities per listing |
| `listing_age_days` | age of listing in days at scrape time |

---

### Phase 2 — Exploratory Data Analysis

**Notebook:** `02_eda.ipynb`  
**Output:** `eda_outputs/*.png`, `eda_summary_stats.csv`

Full visual EDA across five dimensions:

- **Univariate** — price distributions on log scale (Buy vs Rent), property type counts, city distribution, bedroom and area distributions
- **Bivariate** — median price by city (Buy and Rent side-by-side), price vs area scatter (log axes, colored by property type), price per sqm by bedroom count, furnished status vs price boxplot, amenity count vs price
- **Multivariate** — Pearson correlation heatmap, pairplot (price_per_sqm, area, bedrooms, amenity_count; hue = offering_type)
- **Temporal** — listing volume by month, median price by month (Buy/Rent), day-of-week posting volume
- **Geographic** — lat/lon scatter colored by price_per_sqm, top 15 towns by listing count

All charts saved as PNG to `./eda_outputs/`. Every chart has a title and labelled axes.

---

### Phase 3 — Outlier Detection, Correlation & Machine Learning

**Notebook:** `03_outlier_ml.ipynb`  
**Output:** `egypt_re_no_outliers.csv`, `egypt_re_predictions.csv`, `feature_importance.csv`

This notebook combines formal statistical treatment with the full ML pipeline in a single end-to-end flow.

**Outlier Detection**

- **Z-score and IQR methods** applied to: `price_egp`, `area_value`, `price_per_sqm`, `bedrooms`, `bathrooms`
- Per-column report: outlier count, % of total, min/max of outlier values
- `is_outlier` boolean flag column added; rows flagged if *any* feature qualifies
- Before/after distribution comparisons plotted
- Clean dataset saved as `egypt_re_no_outliers.csv`

**Correlation Analysis**

- **Pearson and Spearman correlation matrices** computed and compared on the clean dataset
- **Point-biserial correlation** between `price_egp` and all 20 binary amenity columns — most price-correlated amenities annotated
- **VIF (Variance Inflation Factor)** check on candidate ML features to detect multicollinearity

**Machine Learning**

Two prediction targets trained on `egypt_re_no_outliers.csv` merged with NLP columns:

*Target 1 — Price Regression (Buy and Rent trained separately)*

| Model | Notes |
|---|---|
| Linear Regression | Baseline |
| Random Forest Regressor | Feature importance extracted |
| XGBoost Regressor | Feature importance extracted |

Evaluation: MAE, RMSE, R² on 80/20 split stratified by city. Residual plots included.

*Target 2 — Offering Type Classification (Buy vs Rent)*

| Model | Notes |
|---|---|
| Logistic Regression | Baseline |
| Random Forest Classifier | — |
| XGBoost Classifier | — |

Evaluation: Accuracy, F1, Confusion Matrix, ROC-AUC.

**Feature set used for both targets:**  
`area_value`, `bedrooms`, `bathrooms`, `amenity_count`, `is_furnished_binary`, `city` (encoded), `property_type` (encoded), `topic_label` (encoded), `sentiment_score`, `days_since_listed`

---

### Phase 4 — NLP Analysis

**Notebook:** `04_nlp.ipynb`  
**Output:** `egypt_re_nlp.csv`

Natural language processing applied to the `description` column:

- **Language detection** via `langdetect` — listings split into English and Arabic subsets, counts reported
- **English NLP pipeline** — lowercase, punctuation removal, stopword removal, lemmatization → word frequency bar chart (top 30) → TF-IDF vectorization (top 500 features) → WordCloud for Buy vs Rent
- **Topic modeling** — LDA with 5 topics on English descriptions; topics manually labelled based on top words; `topic_label` column added; topic distribution by city plotted
- **Sentiment analysis** via TextBlob/VADER — `sentiment_score` (polarity) and `sentiment_label` (Positive/Neutral/Negative) columns added; correlation with price_per_sqm analysed; boxplot of price_per_sqm by sentiment label

---

---

## BI Dashboards

### Power BI

**File:** `bi/egypt_re_2026.pbip`  
**Source tables:** `egypt_re_clean.csv` + `egypt_re_amenities_binary.csv`

The Power BI workbook implements a **star schema** with a DAX Calendar table and 6 dashboard pages:

| Page | Focus |
|---|---|
| Market Overview | KPI cards, Buy/Rent split, listing counts by city & type, price bands |
| Location Intelligence | Bubble map by district, drill-down matrix (City → Town → District), top town rankings |
| Specs & Pricing | Price per sqm by bedrooms/bathrooms/furnishing, area vs price scatter, amenity ranking |
| Agent & Quality | Broker performance, premium/verified/featured flag rates, agent scatter |
| Temporal Trends | Monthly volume and price lines, day-of-week bar, seasonality, month-period breakdown |
| Time Intelligence | YoY comparison, QoQ bar, rolling 3M average, MoM price change, velocity scatter, stale listing trend, YTD cumulative area chart, week×day heatmap matrix |

Key DAX measures include: rental yield %, MoM/QoQ/YoY volume and price changes, 3-month rolling average, YTD vs PYTD variance, stale listing %, pool price premium %, and 20 amenity-specific price measures.

> **Note:** The Temporal Trends and Time Intelligence pages are powered entirely by `egypt_re_clean.csv` — no forecast file is required.

---

### Tableau

**File:** `bi/egypt_re_2026.twbx`  
**Source tables:** Same two CSVs, joined on `listing_id` (Left join)

Tableau focuses on analysis **not covered in Power BI** — amenity intelligence, geospatial compound-level exploration, and granular temporal patterns:

| Dashboard | Focus |
|---|---|
| Amenity Intelligence | Amenity price ranking bar, amenity count vs price scatter, amenity co-occurrence heatmap (parameter-driven) |
| Geospatial & Yield Explorer | Point listing map (dark background), top-30 compound price ranking, rental yield % bubble map by town, Buy vs Rent dual-axis by district |
| Temporal Patterns & Price Momentum | Month × Day heatmap, quarterly dual-axis price trend, city price trajectories over time, listing age band stacked bar, animated velocity scatter (Pages shelf), season × month period grouped bar |

Custom color palettes (`EgyptRE PropertyType`, `EgyptRE Yield`, `EgyptRE Temporal`) are defined in `Preferences.tps`. LOD expressions handle city-level and town-level rental yield calculations. Sheet 3E features an animated scatter chart driven by a Pages shelf on `Quarter-Year Label`.

---

## Output Files

| File | Produced by | Consumed by |
|---|---|---|
| `egypt_re_clean.csv` | `01_cleaning.ipynb` | Power BI, Tableau, all downstream notebooks |
| `egypt_re_amenities_binary.csv` | `01_cleaning.ipynb` | Power BI, Tableau |
| `egypt_re_no_outliers.csv` | `03_outlier_ml.ipynb` | ML section within same notebook |
| `egypt_re_nlp.csv` | `04_nlp.ipynb` | `03_outlier_ml.ipynb` (ML section), Power BI |
| `egypt_re_predictions.csv` | `03_outlier_ml.ipynb` | Power BI |
| `feature_importance.csv` | `03_outlier_ml.ipynb` | Power BI |
| `eda_summary_stats.csv` | `02_eda.ipynb` | Reference |

---

## Setup & Installation

### 1. Clone the repository

```bash
git clone https://github.com/your-username/egypt-real-estate-2026.git
cd egypt-real-estate-2026
```

### 2. Create a virtual environment

```bash
python -m venv venv
source venv/bin/activate        # macOS/Linux
venv\Scripts\activate           # Windows
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Place the raw data file

Download or copy `egypt_re_raw.csv` into the `data/` directory. The notebooks reference it via relative path `../data/egypt_re_raw.csv`.

### 5. Run notebooks in order

```bash
jupyter notebook
```

Open and run notebooks in this sequence: `01_cleaning` → `02_eda` → `04_nlp` → `03_outlier_ml`. Note that the ML section inside `03_outlier_ml` requires the NLP output (`egypt_re_nlp.csv`) produced by `04_nlp`, so run `04` before `03`.

### 6. Open BI files

- **Power BI:** Open `bi/egypt_re_2026.pbip` in Power BI Desktop. Update the file path in Power Query if the `outputs/` folder location differs.
- **Tableau:** Open `bi/egypt_re_2026.twbx` in Tableau Desktop. Data is embedded in the packaged workbook.

---

## Libraries & Dependencies

```
pandas>=2.0
numpy>=1.24
matplotlib>=3.7
seaborn>=0.12
scikit-learn>=1.3
xgboost>=2.0
nltk>=3.8
textblob>=0.17
vaderSentiment>=3.3
wordcloud>=1.9
langdetect>=1.0
scipy>=1.11
jupyter>=1.0
ipykernel>=6.0
```

Install all at once:

```bash
pip install -r requirements.txt
```

---

## Design Decisions

**Why IQR per offering type for price outliers?**  
Buy and Rent prices occupy completely different numeric ranges (millions EGP vs thousands EGP/month). Pooling them before outlier removal would incorrectly flag normal rental prices as outliers relative to the Buy distribution.

**Why keep both Pearson and Spearman correlations?**  
Real estate price data is right-skewed. Spearman captures monotonic relationships that Pearson misses when distributions are non-normal, giving a more robust picture of which features co-move with price.

**Why merge outlier detection and ML into one notebook?**  
The two phases share a natural dependency — outlier removal directly produces the clean dataset that feeds the ML models. Keeping them in one notebook makes the data flow explicit and eliminates the need to reload and re-validate the dataset between phases.


**Power BI vs Tableau division of work:**  
Power BI handles structured market overviews, agent performance, and time-intelligence patterns (YoY, QoQ, rolling windows) where DAX measures provide clean, reusable logic. Tableau handles amenity co-occurrence (which requires parameter-driven interactivity), geospatial compound-level exploration (point maps with custom tooltips), and animated temporal patterns (Pages shelf) where Tableau's visual grammar is more expressive.

**No hardcoded paths:**  
All notebooks use relative paths from the project root. This ensures the project runs correctly regardless of where it is cloned.

---

## License

This project is for educational and portfolio purposes. The underlying dataset was scraped from a public real estate portal and is not redistributed here.
