# DAX Measures & Metric Definitions

Notes
- Flow metrics (revenue, orders, reviews) follow the year slicer.
- Snapshot metrics (churn, repeat purchase, forecast) ignore it via `REMOVEFILTERS(Dim_Date)`.
- The like-for-like (LFL) measures below are reconstructed from the logic described in the README. If your model has slightly different names, keep your own version.

## 1. Date table and helper column

```DAX
Order_Date =
DATE(
    YEAR(cleaned_orders[order_purchase_timestamp]),
    MONTH(cleaned_orders[order_purchase_timestamp]),
    DAY(cleaned_orders[order_purchase_timestamp])
)
```
Calculated column on `cleaned_orders`. Relationship: `Dim_Date[Date]` 1 -> * `cleaned_orders[Order_Date]`.

```DAX
Dim_Date =
VAR MinDate = DATE(YEAR(MIN(cleaned_orders[Order_Date])), 1, 1)
VAR MaxDate = DATE(YEAR(MAX(cleaned_orders[Order_Date])), 12, 31)
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMMM"),
    "Month Year", FORMAT([Date], "MMM YYYY"),
    "Month Year Sort", YEAR([Date]) * 100 + MONTH([Date]),
    "Quarter", "Q" & QUARTER([Date]),
    "Day of Week", FORMAT([Date], "dddd"),
    "Day Number", WEEKDAY([Date], 2)
)
```
Mark as date table, and sort `Month Name` by `Month Number`, `Month Year` by `Month Year Sort`.

## 2. Core KPIs

```DAX
Total Revenue = SUM(cleaned_order_payments[payment_value])

Total Orders = DISTINCTCOUNT(cleaned_orders[order_id])

Average Order Value = DIVIDE([Total Revenue], [Total Orders])
```

## 3. Like-for-like growth

2018 only has data until August, so 2018 is compared with Jan-Aug 2017.

```DAX
Revenue PY (Like for Like) =
VAR _CurrYear = SELECTEDVALUE(Dim_Date[Year])
VAR _Cutoff = IF(_CurrYear = 2018, 8, 12)
RETURN
CALCULATE(
    [Total Revenue],
    REMOVEFILTERS(Dim_Date),
    Dim_Date[Year] = _CurrYear - 1,
    Dim_Date[Month Number] <= _Cutoff
)

Revenue Growth % (LFL) =
VAR _CurrYear = SELECTEDVALUE(Dim_Date[Year])
VAR _Cutoff = IF(_CurrYear = 2018, 8, 12)
VAR _Curr =
    CALCULATE(
        [Total Revenue],
        REMOVEFILTERS(Dim_Date),
        Dim_Date[Year] = _CurrYear,
        Dim_Date[Month Number] <= _Cutoff
    )
VAR _Prev = [Revenue PY (Like for Like)]
RETURN
IF(_CurrYear <= 2017, BLANK(), DIVIDE(_Curr - _Prev, _Prev))
```
Growth is blank for 2016 and 2017 because 2016 only has Sep-Dec.

```DAX
Revenue Growth Note =
VAR y = SELECTEDVALUE(Dim_Date[Year])
RETURN
IF(
    ISBLANK(y) || y <= 2017,
    "Tidak dihitung · 2016 hanya Sep-Des",
    FORMAT([Revenue Growth % (LFL)], "+0.0%;-0.0%") & " vs Jan-Agu 2017"
)
```

## 4. Forecast

```DAX
Forecast Next 6M =
VAR _LastActual =
    CALCULATE(
        MAXX(
            FILTER(revenue_forecast, NOT ISBLANK(revenue_forecast[actual_revenue])),
            revenue_forecast[date]
        ),
        REMOVEFILTERS(Dim_Date)
    )
RETURN
CALCULATE(
    SUM(revenue_forecast[predicted_revenue]),
    REMOVEFILTERS(Dim_Date),
    revenue_forecast[date] > _LastActual,
    revenue_forecast[date] <= EDATE(_LastActual, 6)
)
```
Result: R$ 7.52M for Sep 2018 - Feb 2019 (not bound to the year slicer).

## 5. Risk metrics

```DAX
Churn Risk Rate =
CALCULATE(
    DIVIDE(
        COUNTROWS(FILTER(customer_churn_predictions, customer_churn_predictions[predicted_churn] = 1)),
        COUNTROWS(customer_churn_predictions),
        0
    ),
    REMOVEFILTERS(Dim_Date)
)
```
Snapshot at 3 Sep 2018.

```DAX
Review Risk Rate =
DIVIDE(
    CALCULATE(COUNTROWS(review_score_predictions), review_score_predictions[predicted_bad_review] = 1),
    COUNTROWS(review_score_predictions),
    0
)

Actual Bad Review Rate =
DIVIDE(
    CALCULATE(COUNTROWS(review_score_predictions), review_score_predictions[is_bad_review] = 1),
    COUNTROWS(review_score_predictions)
)

Late Delivery Rate =
DIVIDE(
    CALCULATE(COUNTROWS(review_score_predictions), review_score_predictions[is_late] = 1),
    COUNTROWS(review_score_predictions)
)

Bad Review Rate (min 100) =
VAR n = COUNTROWS(review_score_predictions)
RETURN IF(n >= 100, [Actual Bad Review Rate])
```

```DAX
Repeat Purchase Rate =
CALCULATE(
    VAR Cust =
        ADDCOLUMNS(
            VALUES(cleaned_customers[customer_unique_id]),
            "n", CALCULATE(DISTINCTCOUNT(cleaned_orders[order_id]))
        )
    RETURN
        DIVIDE(COUNTROWS(FILTER(Cust, [n] > 1)), COUNTROWS(Cust)),
    REMOVEFILTERS(Dim_Date)
)

Product Revenue = SUM(cleaned_order_items[price])
```

## 6. Calculated columns (not measures)

On `review_score_predictions`:

```DAX
Delay Bucket =
SWITCH(
    TRUE(),
    review_score_predictions[delay_days] <= 0, "On time",
    review_score_predictions[delay_days] <= 3, "Late ≤ 3 days",
    review_score_predictions[delay_days] <= 7, "Late 3-7 days",
    "Late > 7 days"
)

Delay Bucket Sort =
SWITCH(
    TRUE(),
    review_score_predictions[delay_days] <= 0, 1,
    review_score_predictions[delay_days] <= 3, 2,
    review_score_predictions[delay_days] <= 7, 3,
    4
)
```
Set `Delay Bucket` to sort by `Delay Bucket Sort`. These must be calculated columns: as measures they raise "A single value for column 'delay_days' cannot be determined".

## 7. Risk levels

```DAX
Churn Level =
VAR s = AVERAGE(customer_churn_predictions[churn_risk_score])
RETURN SWITCH(TRUE(), s >= 0.60, "High", s >= 0.40, "Medium", "Low")

Review Level =
VAR r = AVERAGE(customer_churn_predictions[avg_review_score])
RETURN SWITCH(TRUE(), r < 3, "High", r < 4, "Medium", "Low")
```
The Low / Medium / High pills and category icons in the table are SVG strings returned by measures (Data category = Image URL).

## 8. Metric definitions

| Metric | Definition | Follows year slicer |
|---|---|---|
| Total Revenue | Sum of `payment_value` (includes freight) | Yes |
| Total Orders | Distinct order count | Yes |
| Average Order Value | Total Revenue / Total Orders | Yes |
| Revenue Growth (LFL) | Revenue of the selected year vs the same months of the previous year; blank for 2016-2017 | Yes |
| Prophet Revenue Forecast | Predicted revenue for the 6 months after the last actual month (Sep 2018 - Feb 2019) | No |
| Review Risk Rate | Share of orders predicted to receive a bad review (model score >= 0.5). Actual = review 1-2 stars | Yes |
| Churn Risk Rate | Share of customers with churn score >= 0.5, snapshot 3 Sep 2018 | No |
| Repeat Purchase Rate | Share of customers with more than one order (all time) | No |
| Late Delivery Rate | Share of orders delivered after the estimated date | Yes |
| Product Revenue | Sum of item `price` (used by category charts) | Yes |
| Avg Rating | Average review stars per category. Low >= 4, Medium 3-4, High risk < 3 | Yes |
