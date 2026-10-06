# Notebooks

**`FoodFlow_pipeline.ipynb`** — the complete end-to-end Databricks notebook.

It runs the full Medallion pipeline in one notebook, in order:

1. **Bronze** — Kaggle API ingest → raw Delta tables
2. **Silver** — cleaning, returns-vs-demand split, oil forward-fill, holiday logic, reconciliation & integrity checks
3. **Gold** — star schema, aggregated mart, ML feature table
4. **Analytics** — Spark SQL business queries + dashboard charts
5. **ML Forecast** — Gradient Boosted Trees demand forecast, backtested vs a naive baseline (MAPE / RMSE) with feature importance

Run it top to bottom on Databricks serverless compute. See [`../docs/setup_and_run.md`](../docs/setup_and_run.md) for setup.
