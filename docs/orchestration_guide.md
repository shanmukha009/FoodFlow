# FoodFlow — Orchestration (Databricks Workflows)

The pipeline runs as a scheduled **Databricks Workflow** (`FoodFlow_Pipeline`) rather than a notebook executed by hand. This is what makes it a pipeline that *operates* instead of a script someone runs manually.

The job definition is versioned in [`../workflows/foodflow_job.json`](../workflows/foodflow_job.json) (orchestration as code).

---

## What the job does

| Setting | Value |
|---|---|
| Job | `FoodFlow_Pipeline` |
| Task | `run_pipeline` — runs the end-to-end notebook on **serverless** compute |
| Schedule | Daily at 02:03 America/Los_Angeles |
| Retries | 2 automatic retries, 5-minute interval |
| Alerts | Email on success **and** on failure |
| Concurrency | 1 run at a time (no overlapping runs) |

The notebook executes all stages in order — Bronze → Silver → Gold → Analytics → ML — in a single task. Databricks also auto-captures **data lineage** for the job (upstream source tables → downstream Gold/feature tables).

---

## Why this matters

- **Scheduling** — the pipeline refreshes on its own; the forecast stays current as new data lands. No one runs it by hand.
- **Retries** — a transient failure (network, timeout during Kaggle download) is retried automatically instead of silently dying.
- **Alerting** — an email fires the moment a run fails, so breakage is noticed immediately.
- **Monitoring** — the Runs page keeps a full history: what ran, how long, pass/fail, logs per run.
- **Orchestration as code** — the job is exportable JSON that can be version-controlled and recreated, not click-ops that lives only in the UI.

This is the operational maturity that separates "I ran a notebook" from "I built a pipeline that runs itself."

---

## How to recreate the job

1. **Workflows / Jobs & Pipelines → Create → Job.**
2. Name it `FoodFlow_Pipeline`.
3. Add a task `run_pipeline`: type **Notebook**, point it at the pipeline notebook, compute **Serverless**.
4. Set **2 retries** (5-minute interval).
5. **Schedules & Triggers → Add schedule** → daily, 02:03 (pause if you don't want it to consume compute nightly).
6. **Notifications → Add** → your email → on success and on failure.
7. **Run now** to confirm a green run.

Or import `workflows/foodflow_job.json` directly (replace the `<your-email>` and `<your-user>` placeholders with your own values first).

---

## Possible next step (optional)

Split the notebook into per-stage tasks (Bronze → Silver → Gold → Analytics → ML) to get a multi-node dependency graph where each stage only starts after the previous one succeeds. The current single-task job already provides scheduling, retries, alerting, and monitoring; the multi-task version mainly adds finer-grained dependency visibility.
