# FoodFlow — Enterprise Data & AI Platform on Databricks

An end-to-end, production-style data platform built on **real grocery point-of-sale data** (Corporación Favorita, ~125M rows). FoodFlow ingests raw, messy retail data and turns it into a governed, business-ready lakehouse using a **Medallion architecture** (Bronze → Silver → Gold) on Databricks, then layers analytics dashboards and a machine-learning demand-forecasting model on top.

This is the same shape of system an enterprise data team delivers for a client: reliable ingestion, real-world data cleaning with reconciliation, dimensional modeling, BI, and ML on a trusted foundation, with data quality treated as a first-class concern at every stage.

---

## Table of Contents
1. [Problem & Motivation](#1-problem--motivation)
2. [Dataset](#2-dataset)
3. [Architecture](#3-architecture)
4. [Tech Stack](#4-tech-stack)
5. [Pipeline — Layer by Layer](#5-pipeline--layer-by-layer)
6. [Analytics / BI](#6-analytics--bi)
7. [Machine Learning — Demand Forecast](#7-machine-learning--demand-forecast)
8. [Data-Quality Engineering](#8-data-quality-engineering)
9. [Results](#9-results)
10. [Repo Structure](#10-repo-structure)
11. [How to Run](#11-how-to-run)
12. [Key Learnings](#12-key-learnings)

---

## 1. Problem & Motivation

Grocery retailers live or die by demand forecasting: over-stock perishable goods and they rot, under-stock and shelves go empty. The raw data behind these decisions is messy — negative sales from returns, missing price feeds on weekends, public holidays that get transferred to different dates, and hundreds of millions of transaction rows.

FoodFlow answers a realistic business question end to end: **given years of historical sales, promotions, holidays, and oil prices, can we build a trusted data foundation and forecast daily demand well enough to beat a naive baseline?** The project demonstrates the full lifecycle — ingest, clean, reconcile, model, analyze, and predict — the way a real data/AI team would deliver it.

---

## 2. Dataset

**Source:** [Corporación Favorita Grocery Sales Forecasting](https://www.kaggle.com/competitions/favorita-grocery-sales-forecasting) (Kaggle), a real Ecuadorian grocery chain.

| Property | Value |
|---|---|
| Total sales rows | ~125 million |
| Stores | 54 |
| Items | 4,036 |
| Date range | 2013-01-01 → 2017-08-15 |
| Source tables | 6 (sales, stores, items, transactions, oil, holidays) |

The six raw source files:

| File | Grain | Description |
|---|---|---|
| `train.csv` | date × store × item | Unit sales, with an `onpromotion` flag. ~125M rows. |
| `stores.csv` | store | Store metadata: city, state, type, cluster. |
| `items.csv` | item | Item metadata: family, class, perishable flag. |
| `transactions.csv` | date × store | Transaction counts per store per day. |
| `oil.csv` | date | Daily oil price (Ecuador's economy is oil-dependent). |
| `holidays_events.csv` | date | National/regional/local holidays and events, including transferred holidays. |

---

## 3. Architecture

FoodFlow implements the **Medallion architecture**: data flows through three progressively refined layers, each stored as **Delta Lake** tables in the `workspace.foodflow` schema.

```
Kaggle (Favorita CSVs, ~125M rows)
        │  programmatic ingest (Kaggle API)
        ▼
   ┌──────────────────────────────────────────────┐
   │  BRONZE  — raw, as-is Delta tables            │
   │  bronze_train, bronze_stores, bronze_items,   │
   │  bronze_transactions, bronze_oil,             │
   │  bronze_holidays                              │
   └──────────────────────────────────────────────┘
        │  clean • reconcile • standardize
        ▼
   ┌──────────────────────────────────────────────┐
   │  SILVER  — trusted, cleaned Delta tables      │
   │  silver_sales (returns vs demand split),      │
   │  silver_oil (forward-filled prices),          │
   │  silver_holidays (effective-holiday logic),   │
   │  silver_stores, silver_items                  │
   └──────────────────────────────────────────────┘
        │  model • aggregate • feature-engineer
        ▼
   ┌──────────────────────────────────────────────┐
   │  GOLD  — business-ready star schema + marts   │
   │  gold_fact_sales,                             │
   │  gold_dim_date, gold_dim_store, gold_dim_item,│
   │  gold_daily_store_sales (mart),               │
   │  gold_ml_features (ML feature table)          │
   └──────────────────────────────────────────────┘
        │
        ├──────────────┬────────────────────┐
        ▼              ▼                    ▼
   Dashboards     ML Forecast         (Streamlit app — planned)
   (Spark SQL)    (Gradient Boosted
                   Trees)
```

Why Medallion: each layer has a single clear responsibility. Bronze preserves the raw truth so ingestion is replayable; Silver is where all real-world messiness is resolved exactly once; Gold is modeled for consumption so analysts and ML never re-clean data. See [`docs/architecture.md`](docs/architecture.md) for the full design rationale.

---

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Platform | Databricks Lakehouse (serverless compute) |
| Storage / format | Delta Lake (ACID transactions, versioned tables) |
| Transformation | PySpark + Spark SQL |
| Ingestion | Kaggle API (programmatic download) |
| Machine learning | scikit-learn — `GradientBoostingRegressor` |
| Analysis / viz | pandas, matplotlib, Databricks dashboards |
| Languages | Python, SQL |

---

## 5. Pipeline — Layer by Layer

### Bronze — raw ingestion
Downloads the six Favorita files from Kaggle via the API, extracts them, and lands each one **as-is** into a Delta table (`bronze_train`, `bronze_stores`, `bronze_items`, `bronze_transactions`, `bronze_oil`, `bronze_holidays`). No cleaning happens here by design — Bronze is the immutable landing zone, so the pipeline can be rerun from a known raw state.

### Silver — cleaning & reconciliation
This is where real-world mess is resolved:

- **`silver_sales`** — splits `unit_sales` into true demand vs returns. Negative `unit_sales` values are returns, so an `is_return` flag is added and a clean `unit_sales_demand` column (returns floored to 0) is derived for forecasting.
- **`silver_oil`** — oil prices are missing on weekends/holidays; prices are **forward-filled** so every date has a usable value.
- **`silver_holidays`** — Favorita's calendar includes *transferred* holidays (a holiday officially moved to another day). An `is_effective_holiday` flag resolves which date actually behaves as the holiday.
- **`silver_stores`, `silver_items`** — standardized, de-duplicated dimension inputs.

A **reconciliation check** confirms cleaned totals tie back to the source, and **referential-integrity checks** confirm every sales key resolves to a valid store, item, and date.

### Gold — dimensional model + marts
A classic **star schema** built for consumption:

- **`gold_fact_sales`** — the central fact at date × store × item grain, with demand, promotion flag, and foreign keys.
- **`gold_dim_date`** — conformed date dimension enriched with calendar attributes, the effective-holiday flag, and the forward-filled oil price.
- **`gold_dim_store`** — store attributes (city, state, type, cluster).
- **`gold_dim_item`** — item attributes (family, class, perishable).
- **`gold_daily_store_sales`** — an aggregated mart (daily sales per store) for fast dashboarding.
- **`gold_ml_features`** — the modeling table with engineered features (see §7).

---

## 6. Analytics / BI

Spark SQL across the Gold star schema answers core business questions, rendered as Databricks dashboard charts:

- **Monthly sales trend** across the full 2013–2017 window.
- **Top item families** by total demand.
- **Sales by state** to see geographic concentration.
- **Promotion impact** — demand on promoted vs non-promoted item-days.
- **Holiday vs normal-day demand** using the effective-holiday flag.

---

## 7. Machine Learning — Demand Forecast

A **Gradient Boosted Trees** regressor (`scikit-learn`) forecasts daily demand.

**Feature table (`gold_ml_features`):**

| Feature | Meaning |
|---|---|
| `month` | Calendar month (seasonality) |
| `is_weekend` | Weekend flag |
| `is_holiday` | Effective-holiday flag |
| `any_promo` | Promotion active that day |
| `oil_price` | Forward-filled daily oil price |
| `lag_7` | Demand 7 days earlier |
| `lag_14` | Demand 14 days earlier |
| `roll_7` | 7-day rolling-average demand |

**Model:** `GradientBoostingRegressor(n_estimators=300, max_depth=3, learning_rate=0.05)`.

**Validation (honest backtest):** the model is trained on history and tested on the **last 90 days**, which it never saw during training, and compared against a **naive baseline** that simply predicts "same as 7 days ago" (`lag_7`). Metrics reported are **MAPE** and **RMSE**, plus a **feature-importance** breakdown for explainability and a forecast-vs-actual plot. Beating the naive baseline is the bar for a forecast being genuinely useful.

---

## 8. Data-Quality Engineering

Real messiness handled explicitly, not ignored:

| Issue in raw data | How FoodFlow resolves it |
|---|---|
| Negative sales (returns) mixed into demand | `is_return` flag + clean `unit_sales_demand` column |
| Missing oil prices on weekends/holidays | Forward-fill in `silver_oil` |
| Transferred public holidays | `is_effective_holiday` logic in `silver_holidays` |
| Source-to-warehouse drift | Revenue/row reconciliation check (cleaned ties to source) |
| Orphaned keys | Referential-integrity checks across fact → dimensions |

Data quality is validated **in the pipeline**, so downstream analytics and ML run on a foundation that is proven to tie back to source.

---

## 9. Results

- A governed lakehouse of **6 Bronze → 5 Silver → 6 Gold** Delta tables over ~125M source rows.
- A star schema serving both BI dashboards and ML from a single trusted model.
- A demand-forecasting model **backtested on an unseen 90-day window** and benchmarked against a naive baseline, with MAPE/RMSE and feature importance reported.

> Exact metric values come from the latest run of the ML Forecast stage in `FoodFlow_pipeline.ipynb`; see the notebook output for the current MAPE/RMSE and the forecast plot.

---

## 10. Repo Structure

```
FoodFlow/
├── notebooks/
│   └── FoodFlow_pipeline.ipynb    # full end-to-end pipeline (Bronze→Silver→Gold→analytics→ML)
├── docs/
│   ├── architecture.md            # design decisions & medallion rationale
│   ├── data_dictionary.md         # table + column definitions, all layers
│   └── setup_and_run.md           # environment setup + run order
└── README.md
```

---

## 11. How to Run

FoodFlow runs on **Databricks (serverless compute)**. Full step-by-step instructions, including Kaggle API setup, are in [`docs/setup_and_run.md`](docs/setup_and_run.md). In brief:

1. Open a Databricks workspace and attach a notebook to **Serverless** compute.
2. Configure Kaggle API access and accept the competition rules.
3. Open `notebooks/FoodFlow_pipeline.ipynb` and run it top to bottom (Bronze → Silver → Gold → analytics → ML).

All tables are created in the `workspace.foodflow` schema.

---

## 12. Key Learnings

Building FoodFlow reinforced the discipline of an enterprise data pipeline: land raw data reliably and keep it replayable, resolve real-world mess (returns, missing prices, transferred holidays) exactly once in a trusted layer, **reconcile** cleaned data back to source, model a star schema for consumption, and only then layer analytics and ML on top. Treating data quality as a first-class, validated step at every stage is what separates a demo from a system a business could actually trust.

---

*Built by Shanmukha Raj — [LinkedIn](https://www.linkedin.com/in/shanmukh-raj) · [GitHub](https://github.com/shanmukha009)*
