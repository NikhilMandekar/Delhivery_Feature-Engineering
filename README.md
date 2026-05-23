# Delhivery Logistics — Feature Engineering & Delivery Time Analytics

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=flat&logo=pandas&logoColor=white)
![SciPy](https://img.shields.io/badge/SciPy-8CAAE6?style=flat&logo=scipy&logoColor=white)
![Scikit-learn](https://img.shields.io/badge/Scikit--learn-F7931E?style=flat&logo=scikit-learn&logoColor=white)

---

##  Business Problem

Delhivery's raw shipment data represents each delivery as multiple rows one per leg of the journey (like connecting flights). The goal is to reconstruct complete end-to-end delivery records, engineer meaningful features, validate routing algorithm estimates through hypothesis testing, and surface actionable insights to reduce delivery times and improve fleet utilisation.

---

##  Dataset Overview

- **144,867 rows × 24 columns** - raw multi-leg shipment records
- Key fields: `trip_uuid`, `source_name`, `destination_name`, `od_start_time`, `od_end_time`, `actual_time`, `osrm_time`, `osrm_distance`, `segment_actual_time`, `route_type`

---

## Approach

### 1. Data Cleaning
- Filled missing `source_name` and `destination_name` using mode imputation
- Converted 4 datetime columns (`trip_creation_time`, `od_start_time`, `od_end_time`, `cutoff_timestamp`) to proper datetime format using Pandas

### 2. Row Aggregation (Reconstructing Trips)
- Used Pandas `groupby` on `trip_uuid + source_center + destination_center` with `sum()` aggregation to reconstruct per-leg records
- Further aggregated on `trip_uuid` alone to get complete end-to-end trip metrics
- Applied `first()` / `last()` for fields where aggregation didn't make semantic sense (e.g. route type, timestamps)

### 3. Feature Engineering (10+ features)
- **Temporal features** from `trip_creation_time`: `trip_year`, `trip_month`, `trip_day`, `trip_hour`, `trip_weekday`
- **Geographic features** from source/destination names: `city`, `place_code`, `state` via `str.rsplit()`
- **Transit time**: computed `od_time_diff` = (`od_end_time` − `od_start_time`) in minutes; original columns dropped

### 4. Hypothesis Testing (4 paired tests via SciPy + visual QQ-plots)

| Test | Variables | Finding |
|---|---|---|
| T-test 1 | `od_time_diff` vs `start_scan_to_end_scan` | Significant difference — scan time ≠ actual OD time |
| T-test 2 | `actual_time` vs `osrm_time` (trip-level) | OSRM systematically **underestimates** actual delivery time |
| T-test 3 | `actual_time` vs `segment_actual_time` | Significant - aggregation method affects time totals |
| T-test 4 | `osrm_distance` vs `segment_osrm_distance` | Distance estimates diverge at trip vs segment level |

### 5. Outlier Treatment & Preprocessing
- Detected outliers in all numerical fields using IQR method (visualised with Seaborn boxplots)
- Capped outliers using `clip(lower, upper)` in Pandas
- Applied one-hot encoding to `route_type` using Pandas `get_dummies()`
- Standardised numerical features using Scikit-learn's `MinMaxScaler`

---

## Key Business Insights & Recommendations

1. **OSRM underestimates real delivery time** - routing model needs recalibration with ground-truth data before use in SLA commitments
2. **Busiest corridors** concentrated between Tier-1 cities - resource allocation should prioritise these routes for faster turnaround
3. **Weekday and hourly patterns** in trip creation reveal optimal dispatch windows to reduce idle fleet time
4. **Last-mile delay** is the primary source of deviation from OSRM estimates - targeted operational fixes at destination zones will have the highest ROI
