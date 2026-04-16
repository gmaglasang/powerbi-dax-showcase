# Power BI DAX Showcase

A curated collection of DAX measures from two Power BI models I built — a **SaaS Metrics Dashboard** and a **Vendor Quality Scorecard**. Each measure is annotated to explain the pattern, why it's non-trivial, and what DAX concept it demonstrates.

![DAX](https://img.shields.io/badge/Language-DAX-yellow) ![Power BI](https://img.shields.io/badge/Tool-Power%20BI-F2C811) ![Models](https://img.shields.io/badge/Models-2-blue)

---

## Why This Exists

Most DAX portfolios are tutorials or toy examples. These measures were written to solve real reporting problems — point-in-time churn tracking, weighted defect scoring, cohort retention, and disconnected slicer patterns. They reflect the kind of work a senior BI developer does day-to-day.

---

## Models

| Model | Measures | Domain |
|---|---|---|
| [SaaS Metrics Dashboard](#-saas-metrics-dashboard) | 41 | SaaS revenue, churn, cohort analysis |
| [Vendor Quality Scorecard](#-vendor-quality-scorecard) | 123 | QA testing, defect tracking, project success |

---

## 📈 SaaS Metrics Dashboard

### 1. Churned MRR
**Pattern:** `CALCULATETABLE` + `ADDCOLUMNS` + `SUMX` + `FILTER` for first-churn detection

```dax
Churned MRR =
VAR _curDate = MAX(Fact_Subscriptions[Date])
VAR _prevDate = EDATE(_curDate, -1)
VAR _churned_keys =
    CALCULATETABLE(
        VALUES(Fact_Subscriptions[Customer_Key]),
        Fact_Subscriptions[Status] = "Churned",
        Fact_Subscriptions[Date] = _curDate,
        ALL(Dim_Date)
    )
VAR _first_churn_month =
    CALCULATETABLE(
        ADDCOLUMNS(
            VALUES(Fact_Subscriptions[Customer_Key]),
            "FirstChurn", CALCULATE(MIN(Fact_Subscriptions[Date]),
                          Fact_Subscriptions[Status] = "Churned", ALL(Dim_Date))
        ),
        Fact_Subscriptions[Customer_Key] IN _churned_keys,
        ALL(Dim_Date)
    )
RETURN
SUMX(
    FILTER(_first_churn_month, [FirstChurn] = _curDate),
    VAR _ck = [Customer_Key]
    RETURN
    CALCULATE(
        SUM(Fact_Subscriptions[MRR]),
        Fact_Subscriptions[Status] = "Active",
        Fact_Subscriptions[Date] = _prevDate,
        Fact_Subscriptions[Customer_Key] = _ck,
        ALL(Dim_Date)
    )
)
```

> **Why it's non-trivial:** A naive churn MRR calculation double-counts customers who churned in previous months. This measure uses `ADDCOLUMNS` to tag each customer's *first* churn date, then filters to only first-time churns in the current period before looking up their prior MRR. Requires managing three separate filter contexts simultaneously.

---

### 2. Churn Rate
**Pattern:** Point-in-time count using `EDATE` + `SELECTEDVALUE` with `MAX` fallback

```dax
Churn Rate =
VAR _lastDate =
    DATE(
        SELECTEDVALUE(Dim_Date[Year], MAX(Dim_Date[Year])),
        SELECTEDVALUE(Dim_Date[Month_Number], MAX(Dim_Date[Month_Number])),
        1
    )
VAR _prevDate = EDATE(_lastDate, -1)
VAR _churned =
    CALCULATE(
        DISTINCTCOUNT(Fact_Subscriptions[Customer_Key]),
        Fact_Subscriptions[Status] = "Churned",
        Fact_Subscriptions[Date] = _lastDate,
        ALL(Dim_Date)
    )
VAR _prev_active =
    CALCULATE(
        DISTINCTCOUNT(Fact_Subscriptions[Customer_Key]),
        Fact_Subscriptions[Status] = "Active",
        Fact_Subscriptions[Date] = _prevDate,
        ALL(Dim_Date)
    )
RETURN DIVIDE(_churned, _prev_active, 0)
```

> **Why it's non-trivial:** Standard date context aggregates across all months in a selection. Using `SELECTEDVALUE` with a `MAX` fallback pins the calculation to a single point-in-time month, making it behave correctly whether one month or a range is selected.

---

### 3. Cohort Retention %
**Pattern:** Cross-filter cohort membership check against a global latest date

```dax
Cohort Retention % =
VAR _cohortCustomers =
    CALCULATETABLE(
        VALUES(Fact_Subscriptions[Customer_Key]),
        Fact_Subscriptions[Is_New] = TRUE()
    )
VAR _cohortSize = COUNTROWS(_cohortCustomers)
VAR _currentLatestDate = CALCULATE(MAX(Fact_Subscriptions[Date]), ALL(Dim_Date))
VAR _stillActive =
    CALCULATE(
        DISTINCTCOUNT(Fact_Subscriptions[Customer_Key]),
        Fact_Subscriptions[Status] = "Active",
        Fact_Subscriptions[Date] = _currentLatestDate,
        Fact_Subscriptions[Customer_Key] IN _cohortCustomers,
        ALL(Dim_Date)
    )
RETURN DIVIDE(_stillActive, _cohortSize, 0)
```

> **Why it's non-trivial:** Cohort analysis requires holding two separate date contexts — the cohort's *signup* period (filtered by slicer) and the *current* latest date (always global, regardless of filters). `ALL(Dim_Date)` in `_currentLatestDate` escapes the slicer context, while `IN _cohortCustomers` preserves cohort membership from the signup period.

---

### 4. Net Revenue Retention (NRR)
**Pattern:** Point-in-time MRR ratio using `EDATE` with explicit date pinning

```dax
Net Revenue Retention =
VAR _curDate = MAX(Fact_Subscriptions[Date])
VAR _prevDate = EDATE(_curDate, -1)
VAR _cur_mrr =
    CALCULATE(
        SUM(Fact_Subscriptions[MRR]),
        Fact_Subscriptions[Status] = "Active",
        Fact_Subscriptions[Date] = _curDate,
        ALL(Dim_Date)
    )
VAR _prev_mrr =
    CALCULATE(
        SUM(Fact_Subscriptions[MRR]),
        Fact_Subscriptions[Status] = "Active",
        Fact_Subscriptions[Date] = _prevDate,
        ALL(Dim_Date)
    )
RETURN DIVIDE(_cur_mrr, _prev_mrr, 0)
```

> **Why it's non-trivial:** `Fact_Subscriptions` is a monthly snapshot table — each row is a customer's state at a point in time. Both dates must override filters with `ALL(Dim_Date)` to pull exact-date snapshots rather than summing across a range. NRR > 100% signals expansion revenue exceeds churn — a key SaaS investor metric.

---

## 🔬 Vendor Quality Scorecard

### 5. Project Success Rate %
**Pattern:** `SUMMARIZE` + `FILTER` + `COUNTROWS` for project-level threshold evaluation

```dax
Project Success Rate % =
VAR Proj =
    SUMMARIZE(
        QualitestData, QualitestData[ProjectID],
        "Crit", SUM(QualitestData[CriticalDefects]),
        "Pass", DIVIDE(SUM(QualitestData[PassedTests]), SUM(QualitestData[TotalTests]), 0)
    )
VAR Success =
    COUNTROWS(FILTER(Proj, [Crit] <= 7 && [Pass] >= 0.95))
RETURN
DIVIDE(Success, COUNTROWS(Proj), 0)
```

> **Why it's non-trivial:** Each project spans multiple test rows. A simple `DIVIDE` evaluates at row level, not project level. `SUMMARIZE` collapses data to one row per project with aggregated metrics, then `FILTER` applies success thresholds at the correct project grain. Classic row context vs. filter context problem solved with table functions.

---

### 6. Successful Projects in Context
**Pattern:** `SUMX` over `VALUES` with per-project `VAR` + `CALCULATE` context transition

```dax
Successful Projects in Context =
SUMX(
    VALUES(QualitestData[ProjectID]),
    VAR crit = CALCULATE(SUM(QualitestData[CriticalDefects]))
    VAR passRate = CALCULATE(
        DIVIDE(SUM(QualitestData[PassedTests]), SUM(QualitestData[TotalTests]), 0)
    )
    RETURN IF(crit <= 5 && passRate >= 0.95, 1, 0)
)
```

> **Why it's non-trivial:** Iterating over `VALUES(ProjectID)` creates a row context per project. `CALCULATE` inside each `VAR` transitions that row context into a filter context, scoping both metrics to the current project's rows only. This correctly handles multi-row projects while fully respecting active report slicers.

---

### 7. Defects per 1000
**Pattern:** Volume-weighted average via `SUMX` instead of naive `AVERAGE`

```dax
Defects per 1000 =
VAR TotalTests = SUM(QualitestData[TotalTests])
VAR WeightedDefects =
    SUMX(QualitestData, QualitestData[DefectsPer1000] * QualitestData[TotalTests])
RETURN
DIVIDE(WeightedDefects, TotalTests, 0)
```

> **Why it's non-trivial:** A simple `AVERAGE(DefectsPer1000)` treats a 10-test project equally to a 10,000-test project — statistically wrong. This computes a proper test-volume-weighted average: each project's defect rate multiplied by its test count, summed, then divided by total tests. Correct defect density regardless of project size distribution.

---

### 8. Projects Marker (Disconnected Slicer)
**Pattern:** `ISINSCOPE` + `COALESCE` + `SELECTEDVALUE` for chart highlight via disconnected table

```dax
Projects Marker (Disconnected) =
VAR SelMonth =
    COALESCE(
        SELECTEDVALUE(MarkerMonth[MonthName]),
        MAX('Calendar'[MonthName])
    )
RETURN
IF(
    ISINSCOPE('Calendar'[MonthName]) &&
    MAX('Calendar'[MonthName]) = SelMonth,
    [Projects This Month]
)
```

> **Why it's non-trivial:** `MarkerMonth` is a disconnected table with no relationship to `Calendar` — used to drive a highlight marker on a line chart without disturbing the chart's date axis. `COALESCE` handles no-selection by defaulting to the latest month. `ISINSCOPE` ensures the measure only fires at the month grain, preventing incorrect aggregation at quarter or year level.

---

## DAX Concepts Covered

| Concept | Measures |
|---|---|
| Filter context manipulation | Churned MRR, Churn Rate, NRR |
| Row context → filter context transition | Successful Projects in Context |
| Table functions (`SUMMARIZE`, `CALCULATETABLE`, `ADDCOLUMNS`) | Project Success Rate %, Churned MRR, Cohort Retention % |
| Volume-weighted aggregation | Defects per 1000 |
| Point-in-time snapshots | Churn Rate, NRR, Churned MRR |
| Cohort analysis | Cohort Retention % |
| Disconnected slicer pattern | Projects Marker |
| `ISINSCOPE` grain detection | Projects Marker |

---

*Built by [Guilbert Maglasang](https://github.com/gmaglasang) · [Portfolio](https://gmaglasang.github.io)*
