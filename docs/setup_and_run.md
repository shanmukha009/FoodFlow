# FoodFlow — Setup & Run Guide

How to reproduce FoodFlow from scratch on Databricks. The whole pipeline runs on **serverless compute** — no cluster to create.

---

## 1. Prerequisites

- A **Databricks** workspace (Free Edition works — it is serverless).
- A **Kaggle** account.
- Accept the competition rules for **Favorita Grocery Sales Forecasting** on Kaggle, and verify your phone number on Kaggle (required before the API will authorize downloads).

---

## 2. Attach to Serverless Compute

Databricks Free Edition has no "create cluster" step — it is serverless.

1. Open your workspace and create/import the FoodFlow notebooks.
2. At the top of a notebook, attach it to **Serverless / Default Interactive Compute**.

> Note: shell utilities like `apt-get` are **not** available on serverless compute. The Bronze notebook therefore extracts the `.7z` archive using the pure-Python `py7zr` package rather than a system tool.

---

## 3. Configure Kaggle API Access

This account uses Kaggle's newer single-token format (a token starting with `KGAT_`), not the older username + key pair.

In the first notebook cell:

```python
%pip install --upgrade kaggle py7zr
dbutils.library.restartPython()
```

Then write your Kaggle token so the API can find it:

```python
import os
os.makedirs(os.path.expanduser("~/.kaggle"), exist_ok=True)
with open(os.path.expanduser("~/.kaggle/access_token"), "w") as f:
    f.write("KGAT_...your token...")  # paste your token
```

> Security: treat this token like a password. If it has ever appeared in a screenshot or shared screen, regenerate it in your Kaggle account settings.

Download the competition files:

```python
import kaggle
kaggle.api.competition_download_files(
    "favorita-grocery-sales-forecasting",
    path="/tmp/favorita"
)
```

Extract any `.7z` files with `py7zr`:

```python
import py7zr, glob, os
for archive in glob.glob("/tmp/favorita/*.7z"):
    with py7zr.SevenZipFile(archive, mode="r") as z:
        z.extractall(path="/tmp/favorita")
```

---

## 4. Run the Pipeline

Open **`notebooks/FoodFlow_pipeline.ipynb`** and run it top to bottom. It executes all stages in sequence, writing every table into the `workspace.foodflow` schema.

| Stage | Produces |
|---|---|
| Bronze | `bronze_train`, `bronze_stores`, `bronze_items`, `bronze_transactions`, `bronze_oil`, `bronze_holidays` |
| Silver | `silver_sales`, `silver_oil`, `silver_holidays`, `silver_stores`, `silver_items` (+ reconciliation & integrity checks) |
| Gold | `gold_fact_sales`, `gold_dim_date`, `gold_dim_store`, `gold_dim_item`, `gold_daily_store_sales`, `gold_ml_features` |
| Analytics | Spark SQL analytics + dashboard charts |
| ML Forecast | Gradient Boosted Trees forecast, backtest vs baseline, MAPE/RMSE, feature importance, forecast plot |

---

## 5. Verify Each Layer

Quick checks after each stage:

```sql
-- Bronze landed?
SELECT count(*) FROM workspace.foodflow.bronze_train;

-- Silver reconciles (returns separated)?
SELECT is_return, count(*) FROM workspace.foodflow.silver_sales GROUP BY is_return;

-- Gold star schema populated?
SELECT count(*) FROM workspace.foodflow.gold_fact_sales;
SELECT count(*) FROM workspace.foodflow.gold_ml_features;
```

If counts are non-zero and the Silver reconciliation/integrity checks pass, the layer is good to build on.

---

## 6. Export the Notebook for GitHub

To publish the repo, export the Databricks notebook as `.ipynb` (File → Export → IPython Notebook) into the local `notebooks/` folder, then:

```bash
git init
git add .
git commit -m "FoodFlow: enterprise medallion data & AI platform"
git branch -M main
git remote add origin https://github.com/<your-username>/FoodFlow.git
git push -u origin main
```
