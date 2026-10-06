# FoodFlow — Architecture & Design Decisions

This document explains *why* FoodFlow is built the way it is, not just what it contains. It covers the Medallion layering, the responsibilities of each layer, and the key engineering decisions behind the cleaning, modeling, and ML stages.

---

## 1. Design Goals

1. **Replayable ingestion** — raw data lands untouched so the pipeline can be rerun from a known state.
2. **Clean once, consume many times** — all real-world messiness is resolved in a single trusted layer, never re-solved by every downstream consumer.
3. **Model for consumption** — a star schema so analysts and ML read from a stable, well-shaped model.
4. **Prove data quality** — reconciliation and integrity checks are part of the pipeline, not an afterthought.
5. **Serverless and reproducible** — runs on Databricks serverless compute with no cluster management, so anyone can reproduce it.

---

## 2. The Medallion Architecture

FoodFlow uses the three-tier Medallion pattern. Each tier is a set of Delta Lake tables in the `workspace.foodflow` schema.

### Bronze — Raw Landing Zone
**Responsibility:** capture source data exactly as delivered.

- Six source files are downloaded from Kaggle and written as-is to Delta: `bronze_train`, `bronze_stores`, `bronze_items`, `bronze_transactions`, `bronze_oil`, `bronze_holidays`.
- No type coercion beyond what the reader infers, no filtering, no business logic.
- **Why:** Bronze is the system of record for the raw truth. If a Silver transformation later proves wrong, we fix the logic and replay from Bronze without re-downloading 125M rows.

### Silver — Trusted / Cleaned Zone
**Responsibility:** resolve real-world data problems exactly once and validate the result.

Key transformations:

- **Returns vs demand (`silver_sales`)** — in the raw data, a negative `unit_sales` is a return, not negative demand. Mixing the two would corrupt any forecast. Silver adds an `is_return` boolean and derives `unit_sales_demand` (returns floored to 0) as the clean target for modeling, while preserving the original value for auditability.
- **Missing oil prices (`silver_oil`)** — oil is not quoted on weekends/holidays, leaving gaps. We **forward-fill** the last known price so every calendar date has a value, which is both economically reasonable (price doesn't change when the market is closed) and necessary for a gap-free feature.
- **Transferred holidays (`silver_holidays`)** — Ecuador transfers some public holidays to a different date. The nominal date is not the one that affects shopping behavior. An `is_effective_holiday` flag marks the date that actually behaves as the holiday.
- **Dimension standardization (`silver_stores`, `silver_items`)** — de-duplicated, typed, and cleaned ready for modeling.

**Validation in Silver:**
- **Reconciliation** — cleaned totals are checked against source totals so we can prove no data was silently dropped or double-counted.
- **Referential integrity** — every sales record must resolve to a valid store, item, and date; orphaned keys are surfaced.

### Gold — Business-Ready Zone
**Responsibility:** model data for analytics and ML consumption.

A **star schema** is chosen because it is the industry-standard shape for analytical workloads: a central fact surrounded by conformed dimensions, giving fast, intuitive slicing.

- **`gold_fact_sales`** — grain is one row per date × store × item, holding the demand measure, promotion flag, and foreign keys to the dimensions.
- **`gold_dim_date`** — conformed date dimension; enriched with calendar attributes, the `is_effective_holiday` flag (from Silver), and the forward-filled `oil_price`. Putting holiday and oil context on the date dimension means any fact joined to it inherits that context for free.
- **`gold_dim_store`** — city, state, type, cluster.
- **`gold_dim_item`** — family, class, perishable.
- **`gold_daily_store_sales`** — a pre-aggregated mart (daily totals per store) so dashboards don't rescan the full fact.
- **`gold_ml_features`** — the modeling table with engineered lag/rolling/calendar features.

---

## 3. Feature Engineering (for ML)

The forecasting target is daily demand. Features are built on top of Gold so the model trains on trusted data:

- **Calendar:** `month` (seasonality), `is_weekend`, `is_holiday` (effective-holiday flag).
- **Business:** `any_promo` (promotion active), `oil_price` (macro signal).
- **Autoregressive:** `lag_7`, `lag_14` (demand 7 and 14 days earlier), `roll_7` (7-day rolling average).

Lag and rolling features are what let a tree model capture recent momentum and weekly seasonality without an explicit time-series model.

---

## 4. Modeling & Validation Decisions

- **Model choice:** `GradientBoostingRegressor` — gradient-boosted trees handle non-linear interactions among calendar, promo, and lag features well, need little scaling, and give interpretable feature importances. Hyperparameters: `n_estimators=300, max_depth=3, learning_rate=0.05` (shallow trees + many rounds + low learning rate = a well-regularized ensemble).
- **Honest backtest:** the last **90 days** are held out as a test window the model never sees in training. This simulates forecasting the future rather than scoring data the model memorized.
- **Baseline comparison:** predictions are benchmarked against a naive **"same as last week" (`lag_7`)** baseline. A forecast is only worth deploying if it beats this baseline on **MAPE** and **RMSE**.
- **Explainability:** feature importances are reported so the forecast's drivers are transparent, not a black box.

---

## 5. Why Serverless Databricks

- No cluster creation or tuning — the notebook attaches to serverless compute, which keeps the project reproducible for anyone with a Databricks workspace.
- Delta Lake gives ACID writes and table versioning, so each layer is a durable, queryable, time-travel-capable table rather than a loose file.

---

## 6. Data Flow Summary

```
Bronze (raw, replayable)
   → Silver (clean once: returns, oil fill, holidays; reconcile + integrity)
      → Gold (star schema + marts + ML features)
         → BI dashboards  &  Gradient Boosted Trees forecast (backtested vs baseline)
```
