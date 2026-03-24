# retail-retention-cohort-analysis
End-to-end customer retention analysis using Python and SQL for data engineering, featuring an interactive Power BI dashboard for cohort behavior tracking.


# 🛒 End-to-End Retail Cohort & Retention Analysis

> **Can we predict which customers will churn — and when?**  
> This project answers that question by building a full data pipeline from raw transactional records to an interactive Power BI dashboard, using SQL Window Functions to track customer cohorts over time.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Dataset](#dataset)
- [Project Architecture](#project-architecture)
- [Key Findings](#key-findings)
- [Repository Structure](#repository-structure)
- [How to Run](#how-to-run)
- [Power BI Dashboard](#power-bi-dashboard)
- [Skills Demonstrated](#skills-demonstrated)

---

## Project Overview

Most e-commerce analytics stop at total revenue or total orders — **vanity metrics** that don't tell you whether your customers are actually coming back. This project goes deeper by answering:

- What percentage of customers return after their first purchase?
- At what month do most customers churn?
- Are there seasonal patterns in long-term retention?
- How does retention differ across customer groups?

The output is a **Retention Heatmap** — a matrix where each row is a customer cohort (defined by when they first purchased) and each column is how many months later they were still active.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Data Engineering | Python, Pandas, NumPy |
| SQL Logic | `pandasql` (SQLite Window Functions) |
| Visualisation | Power BI Desktop |
| Data Source | UCI Online Retail Dataset |

---

## Dataset

**Source:** [UCI Machine Learning Repository — Online Retail](https://archive.ics.uci.edu/ml/datasets/Online+Retail)

| Property | Value |
|----------|-------|
| Raw records | 541,909 rows |
| Time range | December 2010 – December 2011 |
| Geography | Primarily United Kingdom (+ international) |
| Columns | InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country |

### Data Quality Findings

| Issue | Count | % of Total | Action Taken |
|-------|-------|-----------|--------------|
| Missing `CustomerID` | 135,080 | 24.93% | Dropped — can't track without an ID |
| Negative Quantity (returns) | 10,624 | ~2% | Filtered out |
| Zero / negative `UnitPrice` | 2,517 | ~0.5% | Filtered out |
| **Clean records retained** | **397,884** | **73.4%** | Used for analysis |

---

## Project Architecture

```
Raw Data (541k rows)
        │
        ▼
┌─────────────────────┐
│  Phase A: Audit     │  ← Identify missing IDs, returns, anomalies
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Phase B: Cleaning  │  ← Drop nulls, filter bad rows, normalise dates to month
└─────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│  Phase C: SQL Window Logic   │  ← Assign CohortMonth (birth month) per customer
│                              │     Calculate CohortIndex (months since birth)
│  MIN(InvoiceMonth)           │
│  OVER(PARTITION BY           │
│       CustomerID)            │
└──────────────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Phase D: Matrix    │  ← Pivot → normalise → retention % per cohort/age
└─────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│  Phase E: BI Export          │  ← Save Retail_Cohort_Data_Final.csv
│  (Retail_Cohort_Data_Final   │     for Power BI consumption
│   .csv)                      │
└──────────────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Power BI Dashboard │  ← Heatmap, Line charts, Country slicer
└─────────────────────┘
```

### The SQL Cohort Logic (Core Technique)

Rather than using loops or multi-step merges, the cohort assignment uses **SQL Window Functions** via `pandasql`:

```sql
WITH cohort_lookup AS (
    -- Stamp every transaction row with the customer's first-ever purchase month
    SELECT
        CustomerID,
        InvoiceMonth,
        MIN(InvoiceMonth) OVER(PARTITION BY CustomerID) AS CohortMonth
    FROM df_clean
),
age_calculation AS (
    -- Calculate months elapsed since the customer's "birth month"
    SELECT
        *,
        (strftime('%Y', InvoiceMonth) - strftime('%Y', CohortMonth)) * 12 +
        (strftime('%m', InvoiceMonth) - strftime('%m', CohortMonth)) + 1 AS CohortIndex
    FROM cohort_lookup
)
SELECT
    CohortMonth,
    CohortIndex,
    COUNT(DISTINCT CustomerID) AS TotalUsers
FROM age_calculation
GROUP BY 1, 2
ORDER BY 1, 2
```

- **`CohortMonth`** — the month a customer made their very first purchase (their "birth month")
- **`CohortIndex`** — `1` = first month active, `6` = still buying 5 months later, etc.

---

## Key Findings

### Retention Matrix (% of original cohort still active)

| Cohort | Month 1 | Month 2 | Month 3 | Month 6 | Month 12 |
|--------|---------|---------|---------|---------|----------|
| Dec 2010 | 100% | 36.6% | 32.3% | 39.8% | **50.3%** |
| Jan 2011 | 100% | 22.1% | 26.6% | 28.8% | 11.8% |
| Feb 2011 | 100% | 18.7% | 18.7% | 24.7% | — |
| Mar 2011 | 100% | 15.0% | 25.2% | 16.8% | — |

### 🔍 Insight 1 — The Dec 2010 "Anniversary Effect"
The very first cohort shows a striking **50.3% retention at Month 12**, far above any other cohort. This suggests a strong seasonal pattern: customers acquired during the 2010 holiday season returned for the 2011 holiday season. This is the business equivalent of "year-one loyalists."

**Business implication:** Design re-engagement campaigns targeting the 11-month mark for all cohorts.

### 🔍 Insight 2 — The "Month 1 Cliff"
Every cohort drops sharply from 100% to **~15–22% by Month 2**, indicating the majority of customers are one-time buyers. The Dec 2010 cohort is the exception (~37%), suggesting that the type of customer acquired matters as much as the quantity.

**Business implication:** The first 30 days post-purchase are critical. Improved onboarding emails or a loyalty incentive at the first purchase could move this number significantly.

### 🔍 Insight 3 — Stabilisation After Month 3
Customers who survive past the 3-month mark tend to plateau around **20–35% retention** and remain relatively stable. If a customer buys three months in a row, they are likely to become a long-term loyalist.

**Business implication:** A "3-month milestone" reward (e.g., loyalty tier upgrade) could lock in this at-risk group.

---

## Repository Structure

```
📦 retail-cohort-analysis
 ┣ 📓 User_Retention_Cohort_Heatmap.ipynb   # Main analysis notebook (documented)
 ┣ 📊 Project_2_BI.pbix                     # Power BI dashboard file
 ┣ 📄 README.md                             # This file
 ┗ 📂 data/
    ┗ Retail_Cohort_Data_Final.csv          # Cleaned + enriched output (generated by notebook)
```

> **Note:** `Retail_Cohort_Data_Final.csv` is generated by running the notebook. The raw dataset is loaded directly from the UCI repository URL inside the notebook — no manual download needed.

---

## How to Run

### Prerequisites

```bash
pip install pandas numpy pandasql openpyxl
```

### Steps

1. Clone this repository
2. Open `User_Retention_Cohort_Heatmap.ipynb` in Jupyter or Google Colab
3. Run all cells — the notebook will:
   - Download the UCI dataset automatically
   - Perform the data audit and cleaning
   - Execute the SQL cohort logic
   - Generate the retention matrix
   - Export `Retail_Cohort_Data_Final.csv`
4. Open `Project_2_BI.pbix` in Power BI Desktop
5. Update the data source path to point to your generated CSV

---

## Power BI Dashboard

The `.pbix` file contains a 3-page interactive dashboard built on `Retail_Cohort_Data_Final.csv`:

| Page | Contents |
|------|----------|
| **Page 1** | Cohort pivot table, retention line chart by cohort age, Country slicer |
| **Page 2** | Detailed retention line chart — customer count vs. CohortIndex |
| **Page 3** | Country filter slicer for cross-segment comparison |

**Key interactive features:**
- **Country slicer** — filter all visuals by geography (e.g., UK-only vs. international)
- **Line charts** — visualise retention decay curves per cohort over time
- **Pivot table** — raw customer counts by CohortMonth and date hierarchy

---

## Skills Demonstrated

- **Data Engineering** — Auditing, cleaning, and transforming 541k raw records with documented reasoning
- **SQL Window Functions** — `PARTITION BY`, CTEs, date arithmetic in `pandasql`
- **Cohort Analysis** — Defining cohorts, computing retention matrices, interpreting heatmaps
- **Business Thinking** — Translating % numbers into actionable retention strategy recommendations
- **BI Dashboarding** — Building multi-page, filterable Power BI reports from a structured CSV export

---

*Dataset: Daqing Chen, Sai Liang Sain, and Kun Guo, "Data mining for the online retail industry", Journal of Database Marketing and Customer Strategy Management, 2012.*
