# 📊 Investment Analytics Platform

![Status](https://img.shields.io/badge/status-active--development-success)
![Architecture](https://img.shields.io/badge/architecture-data%20lake%20%2B%20warehouse-blue)
![BI](https://img.shields.io/badge/BI-Power%20BI-yellow)
![Data](https://img.shields.io/badge/exchange%20rate-BCRP%20real%20data-orange)
![Focus](https://img.shields.io/badge/focus-financial%20analytics-purple)

---

## 🚀 Overview

**Investment Analytics Platform** is an end-to-end analytics system that simulates how an investment/finance organization consolidates, models, and reports on investment and budget execution data.

It replicates a realistic enterprise data flow:

> **Data Lake (Raw) → ETL (Python) → Data Warehouse (Star Schema) → BI Layer (Power BI)**

The goal is to demonstrate practical data engineering and financial analytics skills applied to a domain-relevant problem: **budget control and investment performance tracking**, not a generic tutorial dataset.

---

## 🎯 Business Problem

Finance and investment teams typically face:

- Investment, project, and budget data scattered across disconnected files/systems
- Inconsistent formats (currencies, dates, status labels) that block reliable reporting
- No single source of truth to answer: *"How much have we actually executed vs. budgeted, and which projects are at risk?"*
- Manual, error-prone reporting cycles that delay decision-making

This project builds a structured, analytics-ready pipeline that answers those questions reliably and reproducibly.

---

## 🧩 Data Sources

| Source | Type | Notes |
|---|---|---|
| Exchange rate (USD/PEN) | **Real data** | Sourced from the BCRP (Banco Central de Reserva del Perú) public API, series `PD04640PD`. Used to normalize all financial figures to a single reporting currency. |
| Projects | Simulated | 24 projects across 6 business areas, generated with intentional risk profiles (healthy / over-budget / delayed / stalled) |
| Investments | Simulated | 1–2 funding investments per project |
| Budgets | Simulated | 1–4 budget lines per project, allocated by period |
| Expense execution | Simulated | Transaction-level granularity (498 transactions), with risk-profile-driven variance patterns |

All simulated data comes from a single reproducible generator (`generate_raw_data.py`, fixed `random.seed(42)`) and is intentionally generated with realistic "messiness" — inconsistent date formats, unstandardized status labels, mixed currency symbols, missing fields — to mirror how raw enterprise data actually looks before cleaning. The exchange rate is the only non-simulated dataset (`fetch_exchange_rate_bcrp.py`), pulled directly from the BCRP's public API.

---

## 🏗️ Architecture

### 🔵 Data Lake (Raw Layer)
Raw, unprocessed files as they would arrive from source systems — inconsistent formatting, no validation applied. Single entry point of the pipeline.

```
data_lake/raw/
  proyectos_raw.csv
  inversiones_raw.csv
  presupuestos_raw.csv
  ejecucion_gasto_raw.csv
  tipo_cambio_raw.csv
```

### 🟠 Data Warehouse (Structured Layer)
Cleaned, validated, and modeled using a **star schema** with two transactional fact tables (separated by grain) plus one derived, pre-aggregated fact table:

**Dimensions**
- `dim_project` — one row per project
- `dim_area` — canonical business areas (6)
- `dim_time` — daily calendar grain (`id_time` as `YYYYMMDD`), covering the full date range observed across projects and transactions
- `dim_currency` — static currency catalog (PEN, USD)
- `dim_investment` — funding/capital allocation catalog

**Facts**
- `fact_budget` — grain: **one row per project × budget period**. Budgeted amount, normalized to PEN.
- `fact_expense_execution` — grain: **one row per individual expense transaction** (the most granular table in the model). Actual amount executed, normalized to PEN, including ~2% legitimate negative transactions (credit notes/reversals), kept as net spend rather than forced positive.
- `project_metrics` — **derived**, not loaded from raw data: one row per project, pre-calculated from the two fact tables above. Holds the aggregated KPIs (execution %, variance, risk flags) so Power BI doesn't need to recompute them from transactional grain on every visual.

**Support table (not a dimension)**
- `bridge_exchange_rate` — daily USD→PEN rate, forward-filled to cover every calendar day (the BCRP only publishes rates on business days). Used internally by the ETL to convert amounts to PEN; **not exposed as a navigable table in Power BI**.

> `fact_budget` and `fact_expense_execution` are kept as two separate fact tables rather than one, because they have genuinely different grains (budget period vs. individual transaction) — merging them would mix granularities in a single table, a classic star-schema modeling mistake.

All monetary fields are normalized to **PEN** (reporting base currency) using real BCRP exchange rate data, while preserving the original currency and amount for traceability.

### 📊 BI Layer
Interactive Power BI dashboard covering financial KPIs, execution tracking, and risk indicators (see KPIs section).

---

## ⚙️ Data Pipeline Flow

```
Raw Data (Data Lake)
        ↓
etl_utils.py              — reusable cleaning functions (dates, amounts, text/category normalization)
        ↓
build_bridge_exchange_rate.py  — [1/4] forward-filled USD/PEN rate, daily grain
        ↓
build_dimensions.py            — [2/4] dim_project, dim_area, dim_time, dim_currency, dim_investment
        ↓
build_facts.py                 — [3/4] fact_budget, fact_expense_execution, project_metrics
        ↓
run_pipeline.py                — [4/4] orchestrates the 3 steps above + referential integrity
                                  and business-rule validation + writes etl_log.txt
        ↓
Structured Star Schema (Data Warehouse)
        ↓
Power Query — Final refinement layer in Power BI
        ↓
Power BI Dashboard
```

`run_pipeline.py` is a thin orchestrator: it calls the cleaning/loading modules in order, consolidates their logs, runs post-load validations (referential integrity, no orphan foreign keys, no unexpected negative amounts, 1:1 coverage in `project_metrics`), and writes everything to a single `etl_log.txt`.

---

## 📊 KPIs & Metrics

**Core financial KPIs**
- Total Budget Allocated (PEN)
- Total Actual Spend (PEN)
- Budget Variance (absolute and %)
- Execution Rate (%)

**Risk & control KPIs** (pre-calculated in `project_metrics`)
- `is_over_budget` — execution > 100% (flagged "at risk" above 110%)
- `is_delayed` — past estimated end date, not marked complete
- `is_stalled` — % of time elapsed exceeds % of budget executed by a meaningful margin
- `is_at_risk` — composite indicator: true if over-budget (>110%), delayed, or stalled (see `/docs/kpi_definitions.md` for full logic)

**Distribution & trend analysis**
- Spend distribution by business area
- Spend distribution by investment type
- Monthly spend trend and cumulative execution curve

---

## 🛠️ Tech Stack

- **Python (csv, datetime, re)** — data generation, cleaning, transformation, currency normalization, DWH load (no pandas dependency in the ETL itself — pure standard library for portability)
- **BCRP public API** — real exchange rate data
- **Power Query** — final-mile transformation inside Power BI
- **Power BI** — star schema modeling (relationships), DAX measures, dashboard

---

## 📂 Repository Structure

```
data_lake/
  raw/              # Unprocessed source files
  processed/        # etl_log.txt — consolidated run log

data_warehouse/
  dim/              # dim_project, dim_area, dim_time, dim_currency,
                     # dim_investment, bridge_exchange_rate (support table)
  fact/             # fact_budget, fact_expense_execution, project_metrics

etl/
  generate_raw_data.py
  fetch_exchange_rate_bcrp.py
  etl_utils.py                    # shared cleaning functions
  build_bridge_exchange_rate.py
  build_dimensions.py
  build_facts.py
  run_pipeline.py                 # single entry point

bi/
  investment_dashboard.pbix

docs/
  architecture_diagram.png
  data_dictionary.md
  data_quality_rules.md
  kpi_definitions.md
```

---

## 🧹 Data Quality Rules (key decisions)

A few non-obvious cleaning decisions, documented here because they reflect real business judgment rather than default library behavior:

- **Ambiguous dates (`NN-NN-AAAA` with both components ≤12)**: genuinely irresolvable from the numbers alone. Day-month is prioritized over month-day, consistent with the regional (Peru) convention used elsewhere in the source data and documented as an explicit assumption in `etl_log.txt`.
- **Mixed budget period granularity** (`presupuestos_raw.csv` mixes ISO month, text month, and quarter): periods are normalized to the first day of the period rather than forced into a single granularity — avoids fabricating a monthly breakdown from a quarterly figure that was never actually budgeted that way.
- **Exchange rate gaps** (BCRP only publishes business days): forward-fill — the last known published rate is carried forward, matching standard accounting practice (LOCF), since no real exchange rate exists for non-business days.
- **Negative expense transactions** (~2% of rows): kept as legitimate credit notes/reversals rather than discarded or forced positive. `monto_ejecutado_pen` represents **net** spend.
- **Unrecoverable rows** (invalid project reference, unparseable amount): rejected and counted in `etl_log.txt`, never silently dropped or fabricated.

Full detail in `/docs/data_quality_rules.md`.

---

## 🧠 Design Decisions Worth Noting

- **Transaction-level fact table grain**, not pre-aggregated — preserves analytical flexibility (e.g., trend analysis, anomaly detection) at the cost of more rows. A deliberate trade-off, not a default.
- **`fact_budget` and `fact_expense_execution` kept separate** — different grains (period vs. transaction) shouldn't share a fact table.
- **Investments modeled as a dimension, not part of the fact grain** — separates "capital allocation decisions" from "operational budget execution," which are conceptually different business events.
- **`project_metrics` as a derived, pre-aggregated table** — risk KPIs are computed once in Python and loaded as a 1-row-per-project table, instead of being recalculated by DAX from transactional grain on every dashboard interaction.
- **Real exchange rate data** instead of a static/simulated rate — avoids an unrealistic assumption that would undermine the credibility of every PEN-denominated KPI.
- **Intentional data quality issues in the raw layer** — designed to make the cleaning/transformation step demonstrate actual judgment (handling missing values, standardizing categorical fields, resolving currency formats, resolving genuinely ambiguous dates), rather than operating on already-clean data.

---

## 🔄 Roadmap

**v1.0 (current)**
- Data Lake + Data Warehouse structure
- Python-based ETL with logging (`etl_utils.py` + 3 build modules + orchestrator), validated end-to-end against all 5 raw sources
- Core Power BI dashboard with risk KPIs (in progress)

**v2.0 (planned)**
- Automated/scheduled pipeline execution
- Expanded data quality validation layer
- Drill-down and what-if analysis in Power BI
- Cloud migration path (Azure / AWS) for portfolio scalability demonstration

---

## 👤 Author Intent

Built as a portfolio project to demonstrate practical, end-to-end analytics capability relevant to financial/investment data roles:

- Data modeling (star schema design with deliberate grain decisions)
- ETL development in Python, including real external API integration
- Financial KPI design and risk logic
- BI dashboard development and data storytelling
- Documentation practices used in real data teams (data dictionary, data quality rules, KPI definitions)

---
