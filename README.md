# Olist Predictive Sales & Risk Analytics

End-to-end analytics project on the Olist Brazilian E-Commerce dataset: data cleaning, SQL ingestion, customer segmentation (RFM), revenue forecasting (Prophet), churn and bad-review risk prediction, and a 3-page Power BI dashboard.

![Executive Overview](docs/screenshots/page1_overview.png)

## Business questions

1. How fast is revenue growing, and what does the next 6 months look like?
2. Which factors drive bad customer reviews?
3. Which customers are at risk of never coming back, and where should retention effort go?
4. How much can the data and models be trusted? (limitations are documented explicitly)

## Key findings

| # | Finding | Evidence |
|---|---|---|
| 1 | Revenue grew strongly, but from a small base | Jan-Aug 2018 revenue +137% vs Jan-Aug 2017 (like-for-like). 2018: R$ 8.70M, 54K orders, AOV R$ 161 |
| 2 | Late delivery is the main trigger of bad reviews | Late orders: ~54% bad reviews vs 9.2% for on-time orders. In 2018: 9.25% (on time), 17.48% (late <= 3 days), 61.73% (3-7 days), 79.66% (> 7 days) |
| 3 | Service quality worsened in 2018 | Late orders 6.5% (2017) to 9.3% (2018); actual bad reviews 12.1% to 13.4% |
| 4 | Retention is weak | ~3% of customers order more than once; 67.7% of customers are predicted as churn risk (snapshot 3 Sep 2018) |
| 5 | Churn risk is concentrated in inactive customers | At Risk 84.0%, Hibernating 83.6%, Recent/Active 51.6%, Champions 43.4% |
| 6 | Revenue forecast | Prophet: R$ 7.52M for Sep 2018 - Feb 2019 |

## Dashboard

| Page | Purpose |
|---|---|
| **Executive Overview** | KPI cards (Revenue, Orders, AOV, Forecast, Review Risk, Churn Risk), monthly trend and forecast, revenue by state, category performance table |
| **Risk Deep Dive** | Churn by RFM segment, bad review rate by delivery delay, top categories by revenue and by bad review rate |
| **Insights & Data Notes** | Findings with evidence, interpretation and recommendation; data limitations; metric definitions |

![Risk Deep Dive](docs/screenshots/page2_risk.png)
![Insights & Data Notes](docs/screenshots/page3_insights.png)

## Methodology

**Pipeline**

| Notebook | Content |
|---|---|
| `01_data_cleaning.ipynb` | Cleaning and standardising the raw Olist tables |
| `02_sql_ingestion.ipynb` | Loading cleaned tables into a SQL database |
| `03_advanced_customer_analytics.ipynb` | RFM segmentation and customer analytics |
| `04_model_revenue_forecasting.ipynb` | Monthly revenue forecast with Prophet |
| `05_model_review_score_risk.ipynb` | Bad-review risk model |
| `06_model_customer_churn.ipynb` | Customer churn model |

**Data model (Power BI).** Star schema: `Dim_Date` -> `cleaned_orders` -> `cleaned_order_items`, `cleaned_order_payments`, `cleaned_customers`, with `cleaned_products` linked through order items. Prediction tables (`customer_churn_predictions`, `rfm_segmentation`, `review_score_predictions`, `revenue_forecast`) are joined by customer or order ID.

**Key modelling decisions**

- **Like-for-like growth.** Olist data effectively ends in August 2018 and 2016 only has Sep-Dec, so 2018 is compared with the same Jan-Aug window of 2017. A full-year comparison would be misleading. Growth is not calculated for 2016 and 2017.
- **Snapshot vs flow metrics.** Revenue, orders and review metrics are flows and follow the year slicer. Churn, RFM segments, repeat purchase rate and the forecast are snapshots and intentionally ignore the slicer.
- **Reconciliation.** Dashboard figures were checked against raw order-level counts (2018: 52,463 reviewed orders, 4,854 late, 7,010 bad reviews).

Full metric definitions and DAX: [`docs/dax_measures.md`](docs/dax_measures.md)

## Data limitations

1. **Limited time range.** Data covers Sep 2016 - Oct 2018. 2016 has only Sep-Dec (329 orders) and 2018 is practically complete only until August.
2. **Mostly one-time buyers.** ~97% of customers ordered once, so churn and retention analysis is limited and churn scores are strongly driven by time since last purchase.
3. **Short forecasting history.** The forecast uses about 20 months of clean data. Yearly seasonality cannot be learned, so the Nov 2017 peak is not captured.
4. **Model evaluation.** Review model: precision ~43%, recall ~56% (threshold 0.5), calculated on the full dataset rather than a held-out test set. A proper held-out evaluation is a next step.
5. **Churn label.** There is no ground-truth churn label; the dashboard shows predicted probabilities. The label definition and its time window should be read together with the notebook (`06_model_customer_churn.ipynb`).
6. **Revenue definitions.** Total Revenue uses `payment_value` (includes freight). Category-level revenue uses item `price`, so the two are not directly comparable.

## Repository structure

```
olist-predictive-analytics/
├── README.md
├── dashboard/                 Power BI file and theme (olist_indigo_theme.json)
├── notebooks/                 6 notebooks: cleaning -> SQL -> RFM -> 3 models
├── data/
│   ├── cleaned/               cleaned tables
│   └── processed/             model outputs used by the dashboard
├── docs/
│   ├── dax_measures.md
│   └── screenshots/
└── .gitignore
```

The raw Olist files (`data/raw/`) are not included. Download them from Kaggle: [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) and place them in `data/raw/`.

## How to run

1. Create a virtual environment and install the dependencies used in the notebooks (pandas, scikit-learn, prophet, SQLAlchemy, Jupyter).
2. Download the raw dataset into `data/raw/`.
3. Run notebooks `01` to `06` in order.
4. Open the Power BI file in `dashboard/` with Power BI Desktop, apply the theme (View -> Themes -> Browse for themes), and point the data sources to `data/cleaned/` and `data/processed/`.

## Tech stack

Python (pandas, scikit-learn, Prophet), SQL, Power BI (DAX, star schema, SVG-based conditional visuals), Git.

## Next steps

- Evaluate the review and churn models on a held-out test set and report ROC-AUC, precision and recall from it.
- Test whether the churn label leaks recency information.
- Add the 2016-2018 year-over-year growth per category with a minimum order threshold.
- Test a retention campaign (60-120 days after first purchase) as the data suggests; this is a hypothesis, not a proven result.

## Author

Adam Huzaini - Informatics Engineering graduate focused on data analysis.
GitHub: [adamhuzaini](https://github.com/adamhuzaini)
