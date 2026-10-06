# FoodFlow — Data Dictionary

Table-by-table and column-by-column reference for every layer. All tables live in the `workspace.foodflow` schema on Databricks.

> Column names reflect the pipeline logic described in the README and architecture docs. Where the raw Kaggle schema carries additional columns, they are preserved through Bronze and surfaced in Silver/Gold as needed.

---

## BRONZE — Raw Landing Zone

Raw source data, landed as-is from Kaggle. No cleaning applied.

### `bronze_train`
Raw sales fact (~125M rows).

| Column | Type | Description |
|---|---|---|
| `id` | long | Row identifier from source |
| `date` | date | Sales date |
| `store_nbr` | int | Store number |
| `item_nbr` | int | Item number |
| `unit_sales` | double | Units sold; **negative = return** |
| `onpromotion` | boolean | Item on promotion that day (nullable in source) |

### `bronze_stores`
| Column | Type | Description |
|---|---|---|
| `store_nbr` | int | Store number (key) |
| `city` | string | Store city |
| `state` | string | Store state |
| `type` | string | Store type |
| `cluster` | int | Store cluster grouping |

### `bronze_items`
| Column | Type | Description |
|---|---|---|
| `item_nbr` | int | Item number (key) |
| `family` | string | Product family |
| `class` | int | Product class |
| `perishable` | int | 1 if perishable, else 0 |

### `bronze_transactions`
| Column | Type | Description |
|---|---|---|
| `date` | date | Date |
| `store_nbr` | int | Store number |
| `transactions` | int | Transaction count for that store-day |

### `bronze_oil`
| Column | Type | Description |
|---|---|---|
| `date` | date | Date |
| `dcoilwtico` | double | Daily oil price; **null on weekends/holidays** |

### `bronze_holidays`
| Column | Type | Description |
|---|---|---|
| `date` | date | Holiday/event date |
| `type` | string | Holiday, Event, Transfer, Bridge, Work Day, Additional |
| `locale` | string | National / Regional / Local |
| `locale_name` | string | City/state the holiday applies to |
| `description` | string | Holiday name |
| `transferred` | boolean | True if the holiday was officially moved to another date |

---

## SILVER — Trusted / Cleaned Zone

Cleaned, reconciled, integrity-checked.

### `silver_sales`
Cleaned sales with returns separated from demand.

| Column | Type | Description |
|---|---|---|
| `date` | date | Sales date |
| `store_nbr` | int | Store number |
| `item_nbr` | int | Item number |
| `unit_sales` | double | Original units sold (preserved for audit) |
| `is_return` | boolean | True when `unit_sales` < 0 |
| `unit_sales_demand` | double | Clean demand target; returns floored to 0 |
| `onpromotion` | boolean | Promotion flag, null coerced to false |

### `silver_oil`
Gap-free oil prices.

| Column | Type | Description |
|---|---|---|
| `date` | date | Date |
| `oil_price` | double | Forward-filled daily oil price (no nulls) |

### `silver_holidays`
Holidays with effective-date logic resolved.

| Column | Type | Description |
|---|---|---|
| `date` | date | Date |
| `description` | string | Holiday name |
| `locale` | string | National / Regional / Local |
| `is_effective_holiday` | boolean | True on the date that actually behaves as the holiday (transfers resolved) |

### `silver_stores`
Standardized, de-duplicated store dimension input.

| Column | Type | Description |
|---|---|---|
| `store_nbr` | int | Store number (key) |
| `city` | string | City |
| `state` | string | State |
| `type` | string | Store type |
| `cluster` | int | Cluster |

### `silver_items`
Standardized, de-duplicated item dimension input.

| Column | Type | Description |
|---|---|---|
| `item_nbr` | int | Item number (key) |
| `family` | string | Product family |
| `class` | int | Product class |
| `perishable` | int | 1 if perishable, else 0 |

---

## GOLD — Business-Ready Zone

Star schema, aggregated marts, and the ML feature table.

### `gold_fact_sales`
Central fact. Grain: one row per **date × store × item**.

| Column | Type | Description |
|---|---|---|
| `date` | date | FK → `gold_dim_date` |
| `store_nbr` | int | FK → `gold_dim_store` |
| `item_nbr` | int | FK → `gold_dim_item` |
| `unit_sales_demand` | double | Clean demand measure |
| `onpromotion` | boolean | Promotion flag |

### `gold_dim_date`
Conformed date dimension, enriched with holiday and oil context.

| Column | Type | Description |
|---|---|---|
| `date` | date | Date key |
| `year` | int | Calendar year |
| `month` | int | Calendar month (1–12) |
| `day` | int | Day of month |
| `day_of_week` | int | Day of week |
| `is_weekend` | boolean | Saturday/Sunday flag |
| `is_holiday` | boolean | Effective-holiday flag (from `silver_holidays`) |
| `oil_price` | double | Forward-filled oil price (from `silver_oil`) |

### `gold_dim_store`
| Column | Type | Description |
|---|---|---|
| `store_nbr` | int | Store key |
| `city` | string | City |
| `state` | string | State |
| `type` | string | Store type |
| `cluster` | int | Cluster |

### `gold_dim_item`
| Column | Type | Description |
|---|---|---|
| `item_nbr` | int | Item key |
| `family` | string | Product family |
| `class` | int | Product class |
| `perishable` | int | Perishable flag |

### `gold_daily_store_sales` (mart)
Pre-aggregated daily sales per store for dashboards.

| Column | Type | Description |
|---|---|---|
| `date` | date | Date |
| `store_nbr` | int | Store number |
| `total_demand` | double | Sum of `unit_sales_demand` for the store-day |

### `gold_ml_features`
Modeling table for the demand forecast.

| Column | Type | Description |
|---|---|---|
| `date` | date | Date |
| `store_nbr` | int | Store number |
| `item_nbr` | int | Item number |
| `unit_sales_demand` | double | Target: daily demand |
| `month` | int | Calendar month (seasonality) |
| `is_weekend` | boolean | Weekend flag |
| `is_holiday` | boolean | Effective-holiday flag |
| `any_promo` | boolean | Promotion active that day |
| `oil_price` | double | Forward-filled oil price |
| `lag_7` | double | Demand 7 days earlier |
| `lag_14` | double | Demand 14 days earlier |
| `roll_7` | double | 7-day rolling-average demand |
