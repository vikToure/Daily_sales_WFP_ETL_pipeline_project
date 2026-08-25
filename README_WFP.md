# WFP Food Prices in Nigeria

**3MTT NextGen Cohort — Data Science Track (DS-14)**
**Author:** Victor Egenkonye

## Project Overview

This project builds a complete ETL (Extract, Transform, Load) pipeline analyzing food commodity prices across Nigeria over more than two decades, using real World Food Programme (WFP) market monitoring data.

**Business questions:**
1. How have prices for key food commodities changed over time?
2. Which Nigerian states have the highest and most volatile food prices?
3. Did food prices in the insurgency-affected northeast (Borno/Yobe) diverge from the rest of the country after the Boko Haram insurgency escalated in 2009?

## Data Source

- **Dataset:** WFP Food Prices for Nigeria
- **Source:** World Food Programme (WFP), via the Humanitarian Data Exchange (HDX) — `data.humdata.org/dataset/wfp-food-prices-for-nigeria`
- **Size:** 87,963 raw price records, 16 columns, spanning January 2002 – April 2026
- **Coverage:** Multiple states, markets, and food commodities across Nigeria, with both Retail and Wholesale price types

## Tools & Libraries

| Tool | Purpose |
|---|---|
| Python 3 | Core language |
| pandas | Data cleaning, transformation, aggregation |
| numpy | Numerical operations, conditional column creation |
| matplotlib | Visualization |
| sqlite3 | Loading cleaned data into a persistent database |
| Jupyter Notebook | Development environment |

## Key Data Quality Decisions

This dataset required more careful handling than a typical clean CSV — several deliberate scoping decisions were made along the way, each documented here for transparency:

- **Price verification filter:** Only rows flagged `'actual'` in `priceflag` were kept (51,789 of 87,963 rows). Rows flagged `'aggregate'` or `'actual,aggregate'` were excluded, since these represent estimated/summarized values rather than directly observed prices. This disproportionately affects **Borno (14,574 excluded rows) and Yobe (9,586 excluded rows)** — a reflection of genuine data-collection challenges in Nigeria's conflict-affected northeast, not a flaw in the analysis.
- **Retail vs Wholesale split:** `pricetype` was split into two separate datasets (`df_retail`, `df_wholesale`) rather than averaged together, since Wholesale (trade-level) and Retail (consumer-level) prices for the same commodity are not comparable numbers. `df_retail` (2014–2026) is used as the primary dataset for commodity trends and regional comparisons. `df_wholesale` (2002–2021) is used specifically for the pre/post-insurgency comparison, since Retail data doesn't extend back before the insurgency began.
- **Unit normalization:** The `unit` column contained 20 distinct values mixing different quantities of the same measure (e.g. `KG`, `100 KG`, `50 KG`). Rather than parsing these with regex, a manual lookup dictionary (`unit_map`) was built — practical given the fixed, small (20-value) set — splitting each into a `quantity` and `base_unit`, then computing a normalized `price_per_unit` for fair comparison over time.
- **Commodity granularity preserved:** Distinct commodity labels (e.g. `Maize (white)`, `Maize (yellow)`, `Maize flour`) were kept separate rather than merged, since each has its own `commodity_id` in the source data and typically a genuinely different price point.
- **Administrative naming:** `admin1`/`admin2` were renamed to `state`/`lga` to match Nigeria's actual administrative terminology (State → Local Government Area).

## Pipeline Structure

The notebook (`wfp_food_prices_in_nigeria_project.ipynb`) follows an ETL flow, producing three analytical tables:

**1. Extract & QA**
Loads the raw CSV and runs a full data quality pass: nulls, duplicates, and — critically for this dataset — checking every categorical column (`currency`, `unit`, `priceflag`, `pricetype`) for hidden inconsistencies before any aggregation happens.

**2. Transform**
Filters to verified price records, splits by price type, normalizes units, renames columns for clarity, and drops columns that become redundant after filtering (`currency`, `priceflag`, `pricetype`, `market_id`, `commodity_id`).

**3. Analysis Tables**
- **`commodity_trend`** — average price per commodity per date, tracking individual commodities over time (Maize, Rice, Palm/Vegetable Oil charted)
- **`regional_summary`** — average price and volatility (standard deviation) per state per commodity, e.g. Palm Oil compared across all 13 states
- **`insurgency_comparison`** — average price per commodity, split by region (Borno/Yobe vs. Rest of Nigeria) and by period (Pre-insurgency vs. Insurgency-era, split at July 2009), built from Wholesale data

**4. Load**
All three analysis tables, plus the cleaned `df_retail` and `df_wholesale` datasets, are saved into a SQLite database (`food_prices.db`) for persistence and reproducibility.

## Key Findings

- **Scope:** 51,789 verified price records were analyzed out of 87,963 raw rows; 36,174 rows were excluded as aggregated/estimated, disproportionately from Borno and Yobe.
- **Commodity trends:** Maize flour rose ~361% (from ~₦177.55 to ~₦818.96 per unit) between 2014 and 2026. Imported rice rose even more sharply — from ~₦140.36 to ~₦2,002.47 — outpacing local rice's rise from ~₦112.59 to ~₦990.24, meaning imported rice has become proportionally more expensive relative to local rice over time.
- **Regional differences (Palm Oil):** Borno had the highest average price (₦691.07); Jigawa had the lowest (₦390.33). Abia showed the highest volatility (std ₦350.81) — nearly as large as its own average, meaning its price swings dramatically month to month.
- **Pre/post-insurgency comparison:** Results were mixed, not uniform, across commodities. Maize and Millet prices in Borno/Yobe stayed flat or fell (−2.6% and −26.7%) while the Rest of Nigeria rose (+16.9% and +13.5%) — consistent with a hypothesis that local demand weakened while supply was redirected elsewhere. Imported rice, however, fell in *both* regions, suggesting import/exchange-rate dynamics unrelated to the insurgency were also at play. This pattern should be read as suggestive, not conclusive — Borno/Yobe's pre-insurgency sample is thin (91 records vs. 3,220 in the insurgency era), limiting confidence in that earlier baseline.

## How to Run

1. Ensure `wfp_food_prices_nga.csv` is in the same folder as the notebook
2. Open `wfp_food_prices_in_nigeria_project.ipynb` in Jupyter
3. Run all cells top to bottom (**Kernel → Restart & Run All** recommended, to guarantee a clean run)
4. Output: a populated `food_prices.db` SQLite database containing five tables (`commodity_trend`, `regional_summary`, `insurgency_comparison`, `df_retail`, `df_wholesale`), with all charts rendered inline

## Files in This Submission

| File | Description |
|---|---|
| `wfp_food_prices_in_nigeria_project.ipynb` | Full ETL pipeline and analysis notebook |
| `wfp_food_prices_nga.csv` | Raw source data (WFP, via HDX) |
| `food_prices.db` | SQLite database produced by the pipeline (generated on run) |
| `README.md` | This file |

## Limitations & Future Improvements

- The pre-insurgency comparison relies on a relatively thin Wholesale data sample for Borno/Yobe (91 records) — a larger historical sample would strengthen confidence in that baseline
- The insurgency-era comparison currently stops at 2021 (the end of available Wholesale data); a complementary check using Retail data (which extends to 2026) could show whether patterns found here persist into more recent years
- Unit normalization used a manually-built lookup table, appropriate given the fixed 20-value set in this dataset — a more general solution (e.g. regex-based parsing) would be needed for a dataset with an unpredictable or growing set of unit labels
- Further analysis could incorporate `usdprice` alongside the Naira-denominated `price` to separate genuine local price movement from currency devaluation effects
