# Traffic Surveillance Dashboard

An interactive dashboard for a traffic and surveillance monitoring system, built for three users:

| User | Page | Question it answers |
|---|---|---|
| Traffic-control operators | Operations | What is happening on the roads, and where is congestion? |
| Enforcement officers | Enforcement | Which violations and junctions need attention, and are challans being paid? |
| Management | Management | Are we improving, is revenue being collected, and what comes next? |

**Links**
- Dashboard (Looker Studio, public): https://datastudio.google.com/reporting/6b282f9f-027a-4a6f-9b0d-07e019246107
- Dataset and code (GitHub): https://github.com/abiya-pardeshi/traffic-surveillance-dashboard
- Colab notebook (data generation and forecasting): https://colab.research.google.com/drive/1QjAWt4f3teLWihePISwAlS3Ns-EXp1wb?usp=sharing
- Video walkthrough : https://www.loom.com/share/94dc454661a54a1792978012d1bf8fc0

**Repository structure**

```
traffic-surveillance-dashboard/
├── README.md
├── Traffic_Surveillance.ipynb   (data generation and forecasting)
└── data/                        (all CSV files)
```

All data is **synthetic**. It was generated with Python to behave like a mid-sized Indian city. Nothing in it describes a real place.

---

## 1. Setup instructions

**View the dashboard:** open the Looker Studio link above. No login is needed.

**Regenerate the dataset and the forecast (optional):**
1. Open `Traffic_Surveillance.ipynb` in Google Colab (use the "Open in Colab" button at the top of the notebook).
2. Click **Runtime, then Run all**. The notebook creates the CSV files listed in section 2 with a fixed random seed (`42`), so the output is identical every time, then runs the forecast and writes `forecast.csv` and `forecast_accuracy.csv`.
3. Download the generated files from the Colab session. The same files are already in the `data/` folder of this repository.

**Rebuild the dashboard from the CSVs:**
1. In Looker Studio, create one CSV File Upload dataset per file (a dataset holds one file layout).
2. Set `timestamp` columns to Text, and create the date fields shown in section 4.
3. Create the calculated fields listed in section 4.

Python packages: `numpy`, `pandas`, `statsmodels` (all pre-installed in Colab).

---

## 2. Dataset

All files are in the `data/` folder.

| File | Rows | One row is | Main columns |
|---|---|---|---|
| `junctions_cameras.csv` | 10 | One junction and its camera | `junction_id`, `name`, `zone`, `lat`, `lon`, `camera_id`, `camera_status`, `uptime_pct`, `free_flow_speed` |
| `traffic_flow.csv` | 86,880 | One junction in one hour | `timestamp`, `junction_id`, `vehicle_count`, `avg_speed_kmph`, `congestion_index`, `weather`, plus junction details |
| `violations.csv` | 32,000 | One violation | `violation_id`, `timestamp`, `junction_id`, `violation_type`, `vehicle_type`, `vehicle_number`, `fine_amount`, `speed_limit`, `speed_recorded`, `camera_id` |
| `challans.csv` | 32,000 | One challan (one per violation) | `challan_id`, `violation_id`, `issue_date`, `due_date`, `status`, `payment_date`, `amount_paid` |
| `violations_enriched.csv` | 32,000 | Violations joined with challans and junction details | all of the above, flattened for the dashboard |
| `forecast.csv` | long format | One day for one forecast metric | `date`, `metric`, `actual`, `forecast`, `lower`, `upper` |
| `forecast_accuracy.csv` | 4 | One method for one metric | `metric`, `method`, `MAPE_%`, `MAE` |

**Period:** 1 November 2025 to 28 October 2026 (362 days, 10 junctions).

### Assumptions built into the data

**Traffic**
- Two daily peaks: a morning rush near 09:00 and an evening rush near 18:30, plus a small lunch bump.
- Weekends carry about 20% less traffic than weekdays.
- Tourist season (Nov to Jan) is 15% busier. The monsoon (Jun to Sep) is 10% lighter, and speeds are about 4 km/h slower.
- Junctions differ in size. Three of them (Market Circle, Bus Stand Junction, River Bridge) are the busiest.
- Speed falls as load rises. `congestion_index = 1 - avg_speed / free_flow_speed`, with a free-flow speed of 50 km/h.

**Violations**
- Violations are sampled in proportion to traffic volume and junction weight, so busy junctions and rush hours produce more.
- Seven violation types, with fixed fines: No Helmet 500, Wrong Parking 500, and Over Speeding, Signal Jump, No Seatbelt, Triple Riding, Mobile Use 1,000 each.
- Helmet and triple-riding violations are two-wheelers only. Seatbelt violations are cars only.
- Speeding cases are recorded 10 to 40 km/h over a 50 km/h limit.
- About 9,000 vehicle numbers exist, with a skewed (Pareto) frequency, so some vehicles repeat many times.

**Challans**
- One challan per violation, issued 1 to 3 days after the violation, due 30 days after issue.
- 78% of challans are eventually paid. Of those, 20% are paid late (31 to 90 days).
- A challan is **Paid** if its payment date is on or before 28 October 2026. Otherwise it is **Contested** (6% of unpaid), **Overdue** (past due date) or **Pending** (not yet due).

### Known limitations
- All patterns were designed by me, so findings describe the dataset, not a real city.
- Violations from the two junctions whose cameras are Offline or in Maintenance (about 9%) are still recorded. A real system would flag this as a data-quality issue.
- Total vehicles counts each pass at each junction, so one vehicle crossing three junctions is counted three times.
- The linear growth I coded into violations is cancelled by the seasonal pattern, so daily violations are flat.

---

## 3. Key metrics: definitions and formulas

| Metric | Definition | Formula |
|---|---|---|
| Total vehicles | Vehicle detections across all junctions | `SUM(vehicle_count)` |
| Average speed | Mean hourly speed | `AVG(avg_speed_kmph)` |
| Congestion index | Share of free-flow speed lost, 0 to 1 | `1 - avg_speed / free_flow_speed`, then `AVG` |
| Camera uptime % | Mean uptime across cameras | `AVG(uptime_pct)` |
| Violations | Number of violations | `COUNT(violation_id)` |
| Violation rate | Violations per 1,000 vehicles | `violations / vehicles * 1000` |
| Payment rate | Share of challans paid | `paid challans / all challans` |
| Overdue amount | Fines on challans past due and unpaid | `SUM(fine_amount)` where `status = "Overdue"` |
| Revenue collected | Fines actually paid | `SUM(amount_paid)` |
| Outstanding amount | Billed but not collected | `SUM(fine_amount) - SUM(amount_paid)` |
| Collection efficiency | Share of billed fines collected | `SUM(amount_paid) / SUM(fine_amount)` |
| Average days to pay | Mean days from issue to payment | `AVG(payment_date - issue_date)`, paid challans only |
| Repeat offender rate | Vehicles with 3 or more violations | `vehicles with >= 3 violations / unique vehicles` |
| Hotspot rank | Junctions ordered by violations | `COUNT(violation_id)` by junction, descending |

---

## 4. Looker Studio calculated fields

```
violation_datetime = PARSE_DATETIME("%Y-%m-%d %H:%M:%S", timestamp)
traffic_datetime   = PARSE_DATETIME("%Y-%m-%d %H:%M:%S", timestamp)
hour_of_day        = FORMAT_DATETIME("%H", traffic_datetime)
weekday            = FORMAT_DATETIME("%u-%a", traffic_datetime)
location           = CONCAT(CAST(lat AS TEXT), ",", CAST(lon AS TEXT))
Payment Rate       = SUM(CASE WHEN status = "Paid" THEN 1 ELSE 0 END) / COUNT(challan_id)
Overdue Amount     = SUM(CASE WHEN status = "Overdue" THEN fine_amount ELSE 0 END)
Outstanding Amount = SUM(fine_amount) - SUM(amount_paid)
Collection Efficiency = SUM(amount_paid) / SUM(fine_amount)
Avg Days To Pay    = AVG(CASE WHEN status = "Paid"
                      THEN DATE_DIFF(PARSE_DATE("%Y-%m-%d", payment_date),
                                     PARSE_DATE("%Y-%m-%d", issue_date)) END)
```

---

## 5. Forecasting

**What is forecast and why**

| Metric | Horizon | Who uses it |
|---|---|---|
| Daily violations | 30 days | Enforcement: staffing and patrol planning |
| Daily vehicle volume | 14 days | Operators: capacity and signal planning |

**Method:** Holt-Winters exponential smoothing (`statsmodels`), with an additive damped trend and additive weekly seasonality (7-day cycle). The model was chosen because it is simple, explainable, and produces uncertainty ranges.

**Baseline:** seasonal naive, which repeats the value from the same weekday one week earlier.

**Evaluation:** the last 30 days were held out. Models were fit on earlier data and scored on the held-out days using MAPE and MAE. The final model is then refit on all data to forecast forward.

**Uncertainty:** the 95% interval is the forecast plus or minus 1.96 times the standard deviation of the model's in-sample residuals, with the lower bound clipped at zero.

**Results (30-day holdout)**

| Metric | Method | MAPE % | MAE |
|---|---|---|---|
| Daily violations | Seasonal naive | 11.31 | 11.60 |
| Daily violations | Holt-Winters | 11.30 | 11.89 |
| Daily vehicle volume | Seasonal naive | 9.32 | 11,040 |
| Daily vehicle volume | Holt-Winters | 10.30 | 12,183 |

**Reading the results**
- For violations the two methods tie. With about 100 events a day, random variation alone causes roughly 10% error, so about 11% is near the best any model can do.
- For vehicle volume the baseline is slightly better. The monsoon ended on 1 October, a step change that falls inside the test window, and a model trained on earlier data cannot anticipate it.
- Holt-Winters is used for the dashboard because it adds trend and prediction intervals.

**Assumptions**
- The weekly pattern continues, with no change in policy, enforcement intensity or camera coverage.
- No unusual events (festivals, strikes, road closures) fall in the forecast window.
- Only one year of history is available, so annual seasonality (for example the winter peak) cannot be learned. The violations forecast (about 102 a day) is above the last 90 days (about 88 a day) and should be read as an upper estimate.

**Next steps:** add monsoon and holiday indicators, test SARIMA or Prophet, and use a rolling-origin backtest.

---

## 6. What the data shows

1. **Hotspots:** three junctions produce 58% of violations. Per 1,000 vehicles, Market Circle is about 3.6 times Airport Road, so the effect is not just traffic volume.
2. **Timing:** the seven rush hours hold 54% of violations. Weekdays run about 25% above weekends.
3. **Repeat offenders:** 28% of violating vehicles have five or more violations and account for 53% of all violations. The worst vehicles appear at many junctions, so vehicle-based escalation is needed alongside junction patrols.
4. **Collections:** 72% of billed fines are collected, the same in every zone and violation type, so the gap is a process problem. Each percentage point is worth about Rs 2.4 lakh. Challans older than 120 days are 78% paid, against 25% for those under 30 days, so recent months must not be judged on raw payment rate.
5. **Overdue money ages:** 69% of overdue challans are more than 90 days past due. Over Speeding and Signal Jump make up 43% of the overdue amount.
6. **Fine design:** 36% of speeding cases are 30 km/h or more over the limit, but the fine is a flat Rs 1,000. A tiered fine would target the most dangerous cases.

---

## 7. Limitations and next steps

- Replace the synthetic data with real camera and challan exports, and re-check every metric definition.
- Add real incident and signal-timing data for operators.
- Automate the pipeline (Python to Google Sheets or BigQuery on a schedule).
- Improve the forecast with holiday and weather features, and track accuracy over time.
