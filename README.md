# urban mobility data preprocessing and feature engineering pipeline

an end-to-end data preprocessing, cleaning, transformation, and feature engineering pipeline for urban mobility datasets using multi-source data ingestion.

---

## project overview

this project integrates multimodal urban transport data across tabular csv, relational sql, and hierarchical json formats to construct machine-learning-ready feature representations. the pipeline handles raw data ingestion, integrity verification, anomaly and outlier treatment, categorical encoding, non-linear transformations, feature scaling, domain-specific feature engineering, and relational merging.

---

## dataset architecture

the pipeline ingests and cleanses three distinct source datasets:

1. **riders demographic records (`riders - riders.csv.csv`)**
   - 300 records tracking registered riders.
   - attributes: rider id, name, age, gender, city, signup date, total rides, cancelled rides, and average rating.

2. **trip transactions (`trips.json`)**
   - 2000 recorded trip transactions.
   - attributes: trip id, rider id, city zone, distance, duration, fare amount, payment mode, ride date, and surge indicator.

3. **city zones spatial data (`city_zones.sql`)**
   - 10 urban operational zones queried via in-memory sqlite[cite: 10].
   - attributes: zone name, population density, traffic index, average speed, and zone classification type .

---

## pipeline workflow

### 1. data ingestion and integrity validation
- multi-format ingestion using `pandas` and an in-memory `sqlite3` execution engine .
- schema inspection via `.info()`, shape verification, and missing value checks .
- domain integrity rule enforcement:
  - rider age constraints: $age \ge 10$[cite: 10].
  - trip distance and financial validity: $distance > 0$ and $fare \ge 0$ .

### 2. anomaly and outlier treatment
- **z-score screening**: detects statistical anomalies beyond three standard deviations ($\vert{}z\vert{} > 3$) on `fare_amount` (16 outliers detected) and `distance_km` (3 outliers detected).
- **iqr boundary clipping**: applies upper and lower interquartile range fences ($q_1 - 1.5 \times iqr$, $q_3 + 1.5 \times iqr$) to cap extreme `duration_min` anomalies (18 detected).
- **percentile winsorization**: caps extreme tail distributions at the 1st and 99th percentiles.

### 3. temporal extraction and categorical encoding
- **datetime breakdown**: extracts temporal components (`hour`, `day_of_week`, `month`) from `ride_date`.
- **label encoding**: converts nominal demographic features like `gender` into zero-indexed numeric identifiers.
- **one-hot encoding**: dummy-encodes nominal attributes `payment_mode` and `zone`.
- **ordinal binning**: categorizes continuous `traffic_index` into structured hierarchical bins (`Low`: 0, `Medium`: 1, `High`: 2) based on domain thresholds.

### 4. distribution transformations and feature scaling
- **skewness reduction**:
  - applies natural log transform ($\ln$) to right-skewed `fare_amount` and `distance_km`.
  - applies square-root transform ($\sqrt{x}$) to normalize `duration_min` variance.
- **feature scaling**:
  - **standardscaler**: centers `total_rides`, `cancelled_rides`, and `avg_rating` around mean 0 with unit variance ($\sigma = 1$).
  - **minmaxscaler**: bounds spatial metrics `population_density`, `traffic_index`, and `avg_speed_kmph` strictly between 0 and 1.

### 5. feature construction
engineers behavioral, financial, and temporal indicators:
- `avg_ride_distance`: mean distance travelled per trip across rider history.
- `avg_ride_fare`: mean fare expense per trip across rider history.
- `is_peak_hour`: binary flag marking rush periods ($hour \in [7\text{--}9, 18\text{--}21]$).
- `days_since_signup`: user account tenure calculated relative to current date.
- `ride_cancellation_rate`: ratio of cancelled bookings to lifetime bookings ($cancelled\_rides / total\_rides$).
- `surge_flag`: flag identifying rides with dynamic surge pricing above per-kilometer rate thresholds.

### 6. relational merging and dataset consolidation
joins the enriched tables via foreign key associations (`trips.rider_id = riders.rider_id` and `trips.zone = city_zones.zone_name`) to produce a single feature table.

---

## processing summary
