# 📊 SME Financial Health Dashboard — Power BI

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-217346?style=for-the-badge&logo=microsoft&logoColor=white)
![Power Query](https://img.shields.io/badge/Power%20Query-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Star Schema](https://img.shields.io/badge/Data%20Modeling-Star%20Schema-blue?style=for-the-badge)
![ETL](https://img.shields.io/badge/ETL-Data%20Cleaning-orange?style=for-the-badge)
![Forecasting](https://img.shields.io/badge/Forecasting-Time%20Series-9cf?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge)

A full Power BI dashboard that lets a small/medium business track its financial health (financial statements, profitability, budget forecasting) from a multi-company quarterly dataset.

---

## 🎯 Project goal

This project addresses a concrete business need: giving an SME owner a clear, actionable view of the company's financial position across **three angles**:

1. **Financial statements** — the core numbers (revenue, expenses, net income, cash flow) and how they evolve.
2. **Profitability & performance** — margins, ROE, ROA, and cross-company comparison.
3. **Forecasting & budgeting** — anticipating next quarter's revenue to support budget decisions.

## 🗂️ Dataset

- **2,000 rows** × **30 columns**
- **50 companies** (`Company_ID`), tracked **quarterly from 2015 to 2024** (`Year`, `Quarter`)
- No missing values

Beyond the standard financial fields, the source file also contains columns meant for an entirely different type of task (sentiment scores, social-media buzz, macro-economic indicators, fraud/anomaly flags). Part of this project was therefore about **critically identifying out-of-scope columns and justifying their removal**, rather than keeping everything by default.

A quick statistical check showed that `Inflation_Rate`, `Interest_Rate`, `Exchange_Rate` and `Global_Economic_Score` vary **randomly per company**, even within the same year/quarter (standard deviation ≈ 2 within a single quarter). These are not genuine shared macro-economic indicators but per-row noise — hence their exclusion.

## 🧹 Data cleaning (Power Query)

### Removed columns (13)
| Category | Columns |
|---|---|
| Stock market data | `Stock_Price`, `Volume_Traded` |
| Alt-data / sentiment | `News_Sentiment_Score`, `Social_Media_Buzz`, `Sector_Trend_Index` |
| Statistical noise (confirmed by analysis) | `Global_Economic_Score`, `Inflation_Rate`, `Exchange_Rate`, `Interest_Rate` |
| Out of scope (fraud/anomaly) | `Audit_Flag`, `Fraud_Flag`, `Market_Shock_Flag`, `Policy_Change_Flag`, `Target_Anomaly_Class` |

### Columns kept
`Year`, `Quarter`, `Company_ID`, `Revenue`, `Expenses`, `Operating_Income`, `Net_Income`, `Assets`, `Liabilities`, `Equity`, `Cash_Flow`, `EPS`, `ROE`, `ROA`, `Debt_to_Equity`, `Target_Revenue_Next_Qtr` (kept specifically for the forecasting tab).

### Type/formatting decisions
- `ROE` and `ROA` are **already** expressed as percentage points (`ROE = Net_Income / Equity × 100`). Applying Power BI's native *Percentage* type would have multiplied the value by 100 again → used **Decimal Number** with the custom format code `0.00"%"` instead.
- `Debt_to_Equity`: decimal number, 2 decimals, optional `x` suffix.
- Currency fields set as **Fixed Decimal Number**, Currency format.

## 🧩 Data modeling — Star schema

One fact table (`FactFinancials`) and two dimension tables (`DimDate`, `DimCompany`) — the minimal structure that still qualifies as a proper star schema, sized to what the dataset actually contains.

**Key steps:**
- Added a custom column `DateKey = #date(Year, (Quarter-1)*3+1, 1)` to turn `Year` + `Quarter` into a real date, enabling time intelligence and the native forecasting engine.
- `DimDate`: marked as the Date Table, with a `Quarter Label` (`"Q" & Quarter & " " & Year`) and `YearQuarterSort` for correct axis sorting.
- `DimCompany`: dedicated company dimension.
- Single-direction, one-to-many relationships between each dimension and the fact table.

**Key modeling note:** `Assets`, `Liabilities` and `Equity` are balance-sheet **snapshots**, not additive flows — summing them across quarters is meaningless. `LASTNONBLANK`-based measures were used to pull the **latest available quarter's** figures instead of a plain `SUM`.

## 🧮 Core DAX measures

```dax
Total Revenue = SUM(FactFinancials[Revenue])
Net Income = SUM(FactFinancials[Net_Income])
Net Profit Margin % = DIVIDE([Net Income], [Total Revenue])
Operating Margin % = DIVIDE(SUM(FactFinancials[Operating_Income]), [Total Revenue])

Latest Assets =
CALCULATE(SUM(FactFinancials[Assets]), LASTNONBLANK(DimDate[DateKey], [Total Revenue]))

Latest Equity =
CALCULATE(SUM(FactFinancials[Equity]), LASTNONBLANK(DimDate[DateKey], [Total Revenue]))

Forecast Revenue (3Q Avg) =
AVERAGEX(
    DATESINPERIOD(DimDate[DateKey], LASTDATE(DimDate[DateKey]), -2, QUARTER),
    [Total Revenue]
)

Forecast Variance % =
DIVIDE(
    SUM(FactFinancials[Target_Revenue_Next_Qtr]) - [Forecast Revenue (3Q Avg)],
    SUM(FactFinancials[Target_Revenue_Next_Qtr])
)
```

`Target_Revenue_Next_Qtr` was deliberately kept to build a genuine forecast-vs-actual comparison for next quarter, without having to invent an artificial target.

## 📑 Report layout — 3 tabs

### Tab 1 · Financial statements overview
KPI cards (Revenue, Expenses, Net Income, Cash Flow), Revenue → Net Income waterfall chart, Assets vs Liabilities/Equity stacked column chart, cash flow area chart with a zero-line, detailed table with data bars.

### Tab 2 · Profitability & performance trends
KPI cards (margin, operating margin, ROE, ROA), ROE vs ROA scatter/bubble chart (bubble = revenue, color = company), margin-vs-benchmark gauge, ranked bar chart of companies by net income, ROE heatmap matrix by company × year.

### Tab 3 · Forecasting & budget planning
KPI cards (forecast, actual, variance), revenue line chart with **Power BI's native forecasting feature** (Analytics pane → Forecast), actual-vs-`Target_Revenue_Next_Qtr` combo chart, forecast variance (%) bar chart, exportable table for budget tracking.

Each tab is complemented by **tooltips** (field-level and report-page) to add context without cluttering the main visuals.

## ⏱️ Time budget (≈ 12–15 h)

| Task | Time |
|---|---|
| Power Query cleanup + star schema + relationships | 2–3 h |
| Core DAX measures (margins, ratios, forecast, variance) | 2 h |
| Tab 1 (statements) | 2–3 h |
| Tab 2 (profitability) | 2–3 h |
| Tab 3 (forecasting) + native forecast setup | 2 h |
| Tooltips + formatting polish + review | 2 h |

## 🛠️ Skills demonstrated

- Data cleaning and critical column selection (Power Query / ETL)
- Dimensional modeling (star schema, relationships, date table)
- DAX measure writing (aggregations, `LASTNONBLANK`, `DATESINPERIOD`, `DIVIDE`)
- Choosing the right visual for the right message (waterfall, heatmap, gauge, combo chart)
- Multi-tab report UX design driven by business decisions
- Using Power BI's native forecasting feature

## 🚀 Usage

1. Open the `.pbix` file in Power BI Desktop.
2. Refresh the data if needed (Home → Refresh).
3. Navigate between the three tabs using the report's bottom navigation bar.

## 📄 License

Project built for demonstration / portfolio purposes.
