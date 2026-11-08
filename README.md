# azure-databricks-finops
FinOps analysis on simulated Azure Databricks billing data: medallion pipeline in SQL, metric view, AI/BI dashboard and Genie Agent
# Azure Databricks FinOps – Cost Optimization

An end-to-end FinOps analysis in Databricks on **simulated** Azure Databricks billing data for a fictional airline data platform (July–September 2026). The project finds where compute spend is wasted, estimates what fixing it would save, and makes the results available through an AI/BI dashboard and a Genie Agent.

> The data is simulated and prices are illustrative, not official Azure prices.

## Key findings

- **$15.4k (32%) of potential savings** in $48.8k of three-month spend; idle compute alone made up **33%** of cost.
- Most of the waste came from **two resources with auto-termination turned off**: a dev cluster busy only **9%** of the hours it ran, and a BI SQL warehouse busy **26%**. Turning on 30-minute auto-termination and warehouse auto-stop would save **$13.9k**.
- Idle cost share rose from **19% in July to 37–38%** in August and September, after the new dev cluster was created.
- Moving scheduled jobs from all-purpose to job compute would save **24%** on those jobs (45% lower DBU price).
- **19% of spend ($9.4k) had no team owner** (untagged), so it could not be charged back.
- **Anomaly:** on 19 August a nightly ETL job (`flight_ops_ingest`) ran for **22 hours instead of ~4.5**, making it the most expensive day ($969).

## Architecture

```
CSV files ──► Unity Catalog volume
                 │
                 ▼
Bronze   raw Delta tables (read_files)
                 │
                 ▼
Silver   cleaned data: duplicates removed, names standardized,
         invalid rows quarantined
                 │
                 ▼
Gold     cost per resource per day, idle cost, savings scenarios
                 │
       ┌─────────┴──────────┐
       ▼                    ▼
Metric view            Materialized view
(KPI definitions)      (daily cost summary)
       │                    │
       └─────────┬──────────┘
                 ▼
AI/BI dashboard + Genie Agent
```

## What I built

| Layer | Work |
| --- | --- |
| Bronze | Loaded 3 CSV files from a Unity Catalog volume into Delta tables with SQL |
| Silver | Data profiling and quality checks: 831 raw rows → 8 duplicates removed, 15 workspace names standardized, 3 invalid rows quarantined → 820 clean rows |
| Gold | Joined usage, resources and prices; calculated total cost, idle cost and two savings scenarios |
| Metric view | Defined Total Cost, Idle Cost, Idle Cost Share, Cost per DBU and Potential Savings once, so the dashboard and Genie use the same KPIs |
| Materialized view | Precomputed daily cost per workspace for the dashboard |
| Governance | RBAC (GRANT), a column mask, and an ABAC column-mask policy driven by a governed tag |
| AI/BI dashboard | KPI counters, monthly cost, idle cost by resource, daily cost by workspace, cost by team |
| Genie Agent | Column descriptions, instructions, example SQL and a benchmark of 8 questions with ground-truth SQL: 8/8 in two consecutive runs |

## Working with AI assistance

I used Genie Code to draft the SQL and reviewed every result. Reviews caught missing columns, a non-existent join column, inaccurate AI-generated column descriptions, and a comparison between data at different levels of detail. Benchmark failures were split into agent errors (fixed with instructions and example SQL) and benchmark errors such as rounding or result shape (fixed in the ground truth after verifying the result).

## Data

| File | Rows | Description |
| --- | --- | --- |
| `billing_usage.csv` | 831 | Usage per resource per day: uptime, active hours, DBUs, team tag |
| `resources.csv` | 10 | Clusters and SQL warehouses: type, size, auto-termination setting |
| `sku_prices.csv` | 3 | Illustrative price per DBU by compute type |

## Repository structure

```
notebooks/   01_bronze, 02_silver, 03_gold
data/        the three CSV files
images/      dashboard screenshot
```

## Tools

Databricks (Free Edition), Unity Catalog, Delta Lake, SQL, metric views, materialized views, AI/BI Dashboards, Genie, Git/GitHub.
