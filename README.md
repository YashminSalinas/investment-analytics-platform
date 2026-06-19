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

> **Data Lake (Raw) → Data Warehouse (Star Schema) → BI Layer (Power BI)**

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
| Exchange rate (USD/PEN) | **Real data** | Sourced from the BCRP (Banco Central de Reserva del Perú) public API. Used to normalize all financial figures to a single reporting currency. |
| Investments | Simulated | Generated with realistic business patterns (not random/uniform) |
| Projects | Simulated | 20+ projects across multiple business areas |
| Budgets | Simulated | Allocated per project/period |
| Expense execution | Simulated | Transaction-level granularity, with intentional variance patterns |

> Simulated data is intentionally generated with realistic "messiness" — inconsistent date formats, unstandardized status labels, missing fields — to mirror how raw enterprise data actually looks before cleaning. This is documented in `/docs/data_quality_rules.md`.

---

## 🏗️ Architecture

### 🔵 Data Lake (Raw Layer)
Raw, unprocessed files as they would arrive from source systems — inconsistent formatting, no validation applied. Single entry point of the pipeline.

### 🟠 Data Warehouse (Structured Layer)
Cleaned, validated, and modeled using a **star schema**:

- `fact_investments` — transaction-level grain (one row per expense execution event)
- `dim_project`
- `dim_area`
- `dim_time`
- `dim_currency`
- `dim_investment`

All monetary fields are normalized to **PEN** (reporting base currency) using real BCRP exchange rate data, while preserving the original currency and amount for traceability.

### 📊 BI Layer
Interactive Power BI dashboard covering financial KPIs, execution tracking, and risk indicators (see KPIs section).

---

## ⚙️ Data Pipeline Flow

```
Raw Data (Data Lake)
        ↓
Python (pandas) — Cleaning, validation, currency normalization
        ↓
Structured Star Schema (Data Warehouse)
        ↓
Power Query — Final refinement layer in Power BI
        ↓
Power BI Dashboard
```

---

## 📊 KPIs & Metrics

**Core financial KPIs**
- Total Budget Allocated (PEN)
- Total Actual Spend (PEN)
- Budget Variance (absolute and %)
- Execution Rate (%)

**Risk & control KPIs**
- Projects Over Budget (execution > 110%)
- Projects Delayed (past estimated end date, not marked complete)
- **Projects At Risk** — composite indicator combining time elapsed vs. budget executed, not just a single threshold (see `/docs/kpi_definitions.md` for full logic)

**Distribution & trend analysis**
- Spend distribution by business area
- Spend distribution by investment type
- Monthly spend trend and cumulative execution curve

---

## 🛠️ Tech Stack

- **Python (pandas)** — data generation, cleaning, transformation, currency normalization, DWH load
- **BCRP public API** — real exchange rate data
- **Power Query** — final-mile transformation inside Power BI
- **Power BI** — star schema modeling (relationships), DAX measures, dashboard

---

## 📂 Repository Structure

```
data_lake/
  raw/              # Unprocessed source files
  processed/        # Cleaned, validated outputs

data_warehouse/
  fact/             # fact_investments
  dim/              # dim_project, dim_area, dim_time, dim_currency, dim_investment

etl/
  generate_raw_data.py
  fetch_exchange_rate_bcrp.py
  data_cleaning.py
  load_warehouse.py
  etl_log.txt        # Run log: rows processed, errors, rows discarded

bi/
  investment_dashboard.pbix

docs/
  architecture_diagram.png
  data_dictionary.md
  data_quality_rules.md
  kpi_definitions.md
```

---

## 🧠 Design Decisions Worth Noting

- **Transaction-level fact table grain**, not pre-aggregated — preserves analytical flexibility (e.g., trend analysis, anomaly detection) at the cost of more rows. A deliberate trade-off, not a default.
- **Investments modeled as a dimension, not part of the fact grain** — separates "capital allocation decisions" from "operational budget execution," which are conceptually different business events.
- **Real exchange rate data** instead of a static/simulated rate — avoids an unrealistic assumption that would undermine the credibility of every PEN-denominated KPI.
- **Intentional data quality issues in the raw layer** — designed to make the cleaning/transformation step demonstrate actual judgment (handling missing values, standardizing categorical fields, resolving currency formats), rather than operating on already-clean data.

---

## 🔄 Roadmap

**v1.0 (current)**
- Data Lake + Data Warehouse structure
- Python-based ETL with logging
- Core Power BI dashboard with risk KPIs

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
