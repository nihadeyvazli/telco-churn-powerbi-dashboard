# Customer Churn Dashboard: Notes

## Tool and data

- **Tool:** Power BI Desktop on Windows
- **Dataset:** Telco Customer Churn (Kaggle), `WA_Fn-UseC_-Telco-Customer-Churn.csv`, 7,043 rows and 21 columns. No other dataset was used.
- **Files:** `churn_dashboard.pbix`, `images/dashboard_overview.png` (no filters), `images/dashboard_filtered_month_to_month.png` (Contract = Month-to-month), `images/whatif_scenario.png` (what-if page), `data/` (original CSV)

The report has two visible pages and one hidden page:
- **Page 1**: the main single-page dashboard
- **What-if Scenario**: a slider that estimates the revenue saved if churn drops in the riskiest segment
- **Tooltip - Segment Detail** (hidden): the small window that appears when hovering over a bar

All cleaning was done in Power Query and all calculations in DAX. No Python or Excel preprocessing.

---

## Power Query decisions

### Applied steps

| # | Step | What it does |
|---|---|---|
| 1 | Source | Loads the CSV |
| 2 | Promoted Headers | Uses the first row as column names |
| 3 | Set Data Types | Gives every column an explicit type |
| 4 | Replace Blank TotalCharges with 0 | Fills the 11 blank values with 0 |
| 5 | Set TotalCharges Type | Converts the column to Decimal Number (English US locale) |
| 6 | Add ChurnFlag | Yes = 1, No = 0 |
| 7 | Set ChurnFlag Type | Whole Number |

Data types: text for the categorical columns and `customerID`, whole number for `SeniorCitizen`, `tenure` and `ChurnFlag`, decimal number for `MonthlyCharges` and `TotalCharges`.

### TotalCharges

Power BI auto-detected `TotalCharges` as a decimal number, but type detection only looks at the first 200 rows. The blank values are further down, so they were silently turned into nulls. I set the column back to text first, replaced the blanks in a separate step, and only then converted it to a number, so the decision is visible in the applied steps.

**Decision: replace the blanks with 0.**

- All 11 blanks belong to customers with `tenure = 0`. They are new and haven't been billed yet, so 0 is the correct value, not a guess.
- Filtering them out would drop real customers (7,032 instead of 7,043) and remove the newest customers from the analysis.
- None of them churned, so removing them would also push the churn rate up slightly.

I used the English (US) locale for the conversion because the file uses a dot as the decimal separator (29.85). After this step the column showed 100% valid, 0 errors and exactly 11 zeros.

### ChurnFlag

`Churn = "Yes"` gives 1, otherwise 0. Quick check: the average of the column is 0.265, which matches the overall churn rate.

---

## Data model (star schema)

The dataset is a single flat table, so I added a small star schema around it. In Power Query I created three dimension tables by referencing the `Telco` query (right-click → Reference). Each one keeps only its own column with duplicates removed, so every value appears exactly once. Because they reference `Telco` instead of loading the CSV again, any cleaning step in `Telco` carries over to them automatically.

| Dimension | Column | Rows |
|---|---|---|
| `DimContract` | Contract | 3 |
| `DimInternetService` | InternetService | 3 |
| `DimPaymentMethod` | PaymentMethod | 4 |

```
DimContract (1)        ──┐
DimInternetService (1) ──┼──> (*) Telco
DimPaymentMethod (1)   ──┘
```

`Telco` is the fact table: one row per customer, and all measures are calculated from it. In Model view each dimension is linked to `Telco` on the matching column with a one-to-many relationship and single filter direction. Power BI detected the three relationships from the column names, and I checked the cardinality and direction.

The slicers, chart axes and the Contract × Internet Service matrix now use the dimension columns, so filters flow from the dimensions to the fact table. The numbers didn't change: Contract = Month-to-month still gives 3,875 customers and 42.7%, and Internet Service = Fiber optic gives 3,096 and 41.9%. Tenure has no dimension table because `TenureBand` is a calculated column on `Telco`, so the Tenure slicer and chart use it directly. The `Churn Reduction` table created by the what-if parameter stays disconnected on purpose.

---

## Calculated columns

I created the bands as DAX calculated columns: the task calls them "calculated columns", and the idea is to use columns for grouping and measures for aggregation. The cleaning itself stayed in Power Query.

```dax
TenureBand =
SWITCH(
    TRUE(),
    Telco[tenure] <= 12, "0-12",
    Telco[tenure] <= 24, "13-24",
    Telco[tenure] <= 48, "25-48",
    "49+"
)
```

```dax
TenureBandOrder =
SWITCH(
    TRUE(),
    Telco[tenure] <= 12, 1,
    Telco[tenure] <= 24, 2,
    Telco[tenure] <= 48, 3,
    4
)
```

`TenureBand` is sorted by `TenureBandOrder` (Sort by column), so the chart always shows 0-12 → 49+ instead of alphabetical order. The helper column is hidden from the report view.

```dax
ChargeBand =
VAR Q1 = PERCENTILE.INC(Telco[MonthlyCharges], 0.25)
VAR Q2 = PERCENTILE.INC(Telco[MonthlyCharges], 0.50)
VAR Q3 = PERCENTILE.INC(Telco[MonthlyCharges], 0.75)
RETURN
SWITCH(
    TRUE(),
    Telco[MonthlyCharges] <= Q1, "Q1 (Low)",
    Telco[MonthlyCharges] <= Q2, "Q2",
    Telco[MonthlyCharges] <= Q3, "Q3",
    "Q4 (High)"
)
```

The cutoffs (about $35.50, $70.35 and $89.85) are calculated from the data rather than typed in, so the bands update if the data changes. Each band holds roughly 25% of customers.

```dax
Segment =
Telco[Contract] & " · " & Telco[InternetService] & " · " & Telco[TenureBand] & " mo"
```

Used to find the highest-risk customer groups (see Key findings).

---

## DAX measures

All measures are explicit. No implicit drag-and-drop aggregations.

```dax
Total Customers = COUNTROWS(Telco)

Churned Customers = CALCULATE([Total Customers], Telco[Churn] = "Yes")

Churn Rate % = DIVIDE([Churned Customers], [Total Customers])

Avg Monthly Charges = AVERAGE(Telco[MonthlyCharges])

Monthly Revenue at Risk = CALCULATE(SUM(Telco[MonthlyCharges]), Telco[Churn] = "Yes")
```

### Why DIVIDE and not `/`

Some slicer combinations can leave no customers in the selection, so the denominator becomes 0. The `/` operator would return an error and break the visual. `DIVIDE` returns a blank and the report keeps working.

### Why churn rate is a measure, not a column

A column is calculated once per row and doesn't react to filters. A measure is recalculated for every bar and every slicer selection, always as churned customers divided by all customers in the current selection. That's why it stays correct when the report is sliced.

### Helper measures

```dax
Avg Churn Rate (Selection) =
CALCULATE([Churn Rate %], ALLSELECTED(Telco))
```
The overall churn rate for whatever is selected in the slicers. Used for the dashed average line and the colour rule.

```dax
Bar Color =
IF([Churn Rate %] > [Avg Churn Rate (Selection)], "#B5462F", "#2F5D50")
```
Red when a group is above the average, green when it's below (conditional formatting by field value). Because the average follows the slicers, the comparison is always made within the current selection.

```dax
Churn Rate (Min 100) =
IF([Total Customers] >= 100, [Churn Rate %])
```
Returns blank for segments with fewer than 100 customers, so they can't appear in the Top 2 table.

### Checks

With no filters: Total Customers = 7,043, Churn Rate % = 26.5%, Monthly Revenue at Risk = $139,130.85, Avg Monthly Charges = $64.76.

---

## Dashboard

- **Left panel:** title, three slicers (Contract, Internet Service, Tenure), three priority actions and a one-line definition of churn rate.
- **KPI cards:** Total Customers, Churn Rate %, Monthly Revenue at Risk, Avg Monthly Charges.
- **Charts:** churn rate by Contract, Internet Service and Payment Method (bar charts) and by Tenure Band (column chart, ordered 0-12 to 49+). All charts show churn **rate**, not counts.
- **Right column:** Contract × Internet Service matrix, Top 2 Risk Segments table and a short summary card.

**Interactivity:** I changed the default interaction in Report settings from cross-highlighting to cross-filtering, so clicking a bar or using a slicer filters every other visual. Test: with Contract = Month-to-month, Total Customers drops to 3,875, Churn Rate goes to 42.7%, and Fiber optic shows 54.6%, the same value as in the matrix. The second screenshot shows this state.

### Bonus features

| Bonus | Status |
|---|---|
| Churn rates above the average turn red | Done, plus an average reference line on every chart |
| Contract × InternetService matrix | Done, shown as a heatmap |
| Tooltip page | Done |
| What-if parameter | Done, on a separate page |
| Publish to Power BI Service | Not done (needs a work or school Microsoft account) |

### Tooltip page

A hidden page, **Tooltip - Segment Detail**, is set up as a tooltip (320 × 240) and linked to the four charts and the matrix. When you hover over a bar, a small 2 × 2 card shows Total Customers, Churned Customers, Churn Rate % and Monthly Revenue at Risk for that bar. For example, hovering over Month-to-month shows 3,875 customers and 42.7% churn. Power BI filters the tooltip page by the hovered bar automatically, so no extra DAX was needed. It also puts the segment size right next to the rate.

### What-if parameter

A numeric range parameter called `Churn Reduction` (0 to 50, step 5, default 10) drives the slider on the **What-if Scenario** page. The "worst segment" is the top risk segment from my analysis: Month-to-month · Fiber optic · 0-12 months.

```dax
Worst Segment Revenue at Risk =
CALCULATE(
    [Monthly Revenue at Risk],
    Telco[Contract] = "Month-to-month",
    Telco[InternetService] = "Fiber optic",
    Telco[TenureBand] = "0-12"
)

Worst Segment Churned =
CALCULATE(
    [Churned Customers],
    Telco[Contract] = "Month-to-month",
    Telco[InternetService] = "Fiber optic",
    Telco[TenureBand] = "0-12"
)

Monthly Revenue Saved =
[Worst Segment Revenue at Risk] * [Churn Reduction Value] / 100

Annual Revenue Saved = [Monthly Revenue Saved] * 12

Customers Retained =
ROUND([Worst Segment Churned] * [Churn Reduction Value] / 100, 0)
```

The segment loses $53,178 a month from about 643 churned customers.

| Churn reduction | Customers retained | Monthly revenue saved | Annual revenue saved |
|---|---|---|---|
| 10% | 64 | $5,318 | $63,814 |
| 20% | 129 | $10,636 | $127,628 |
| 50% | 322 | $26,589 | $319,070 |

**Assumptions:** the percentage is a relative reduction (10% means 10% of the customers who would have left stay instead), and the customers who stay keep paying their current monthly charge. The cost of the retention offer isn't included, so the real saving would be lower. The segment is fixed in the measures, so the slicers on the main page don't affect this page.

---

## Key findings

| Group | Churn rate |
|---|---|
| All customers | 26.5% (1,869 of 7,043) |
| Month-to-month / One year / Two year | 42.7% / 11.3% / 2.8% |
| Fiber optic / DSL / No internet | 41.9% / 19.0% / 7.4% |
| Electronic check | 45.3% (automatic payments: 15-17%) |
| Tenure 0-12 / 13-24 / 25-48 / 49+ | 47.4% / 28.7% / 20.4% / 9.5% |

Note: the reference notes show about 47.7% for 0-12 months, I get 47.4%. The difference is the 11 customers with tenure = 0 that I kept. None of them churned, so the rate is slightly lower.

### Why volume matters

When I ranked every Contract × Internet × Tenure combination by churn rate, the third row was `Month-to-month · No internet · 49+ mo` with 50% churn. It has **2 customers**, so 50% just means one of them left. One person behaving differently would turn it into 0% or 100%. A rate based on so few people is mostly chance, and the segment is worth about $19 a month. An even more extreme case: `Two year · Fiber optic · 13-24 mo` has a single customer, so if that person left it would show "100% churn".

So I only ranked segments with **at least 100 customers** (about 1.4% of the base) and always read the rate together with the customer count.

### The two segments that matter most

| Segment | Customers | Churn | Lost per month |
|---|---|---|---|
| Month-to-month · Fiber optic · 0-12 months | 916 | 70.2% | $53,178 |
| Month-to-month · Fiber optic · 13-24 months | 425 | 50.6% | $19,047 |

Together they are **1,341 customers (19% of the base)** but account for **$72.2K a month, about 52% of all revenue at risk**.

### Other observations

- **The contract matters more than the internet type.** In the matrix, Fiber optic churns at 54.6% on month-to-month contracts but only 7.2% on two-year contracts.
- Month-to-month customers are 55% of the base (3,875) but about 87% of the revenue at risk ($120.8K of $139.1K).
- `Month-to-month · Fiber optic · 25-48 months` (521 customers, 43.4%) ranks third on churn rate but loses slightly more money ($20.8K) than the second segment, so it shouldn't be ignored.
- Payment method is more likely a sign of low commitment than a cause of churn. That's my interpretation, not something the data proves.

### Limitation

The data shows **where** churn happens, not **why**. Price, service quality or competitor offers could all explain the Fiber result, and this dataset can't separate them.

---

## Recommendations (in priority order)

**1. Support new fiber customers through their first year.**
Target: month-to-month fiber customers in their first 12 months (916 customers, 70.2% churn). Proactive check-ins at around 30, 60 and 90 days and a service quality follow-up. This comes first because it's the riskiest large group and on its own accounts for about 38% of the revenue at risk ($53.2K a month). Even a small improvement here protects more revenue than anywhere else. The what-if page puts a number on it: cutting churn in this segment by 20% would keep about 129 customers and protect around $10.6K a month ($127.6K a year), before the cost of the program.

**2. Move month-to-month customers onto 1 or 2 year contracts.**
Target: 3,875 customers, about 87% of the revenue at risk. Offer a discount or a free add-on in exchange for a longer contract, especially in the first 24 months. Churn drops from 42.7% on monthly contracts to 11.3% and 2.8% on longer ones, and the matrix shows the same pattern within Fiber optic (54.6% vs 7.2%).

**3. Encourage Electronic check customers to switch to automatic payment.**
Electronic check churns at 45.3% compared with 15-17% for automatic payments. A small one-time credit for switching to bank transfer or card autopay. This comes third because payment method is probably a signal rather than a cause, so the effect is less certain. It's still the cheapest action to run.
