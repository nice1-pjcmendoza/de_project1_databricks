I'm thinking of a Junior Data Engineering Project on Databricks that will showcase ETL and Medallion Architecture on Databricks. 
All raw data and tables will be stored within the Azure service. 

Attached is a sample dataset for January 2025.
Do the following:
1. Analyze the attached dataset. 
2. Provide the structure of schema for a batch ingestion process using the Cyclistic bike share dataset, as attached. 
2. Provide both SQL and Python codes on each step, including creating the schemas/tables, data ingestion, data cleaning, and so on.

3. Suggest best practices and optimizations we can apply in this project.
4. Suggest 3 concise and professional project titles that will impress the interviewers.

---

Awesome idea, Paul! A Cyclistic/Divvy ETL on Databricks is perfect for showcasing the Medallion Architecture (Bronze → Silver → Gold), governance with Unity Catalog, and both SQL + PySpark skills—all inside Azure (ADLS Gen2 + Databricks). Below is a complete, interview‑ready blueprint with:

1.  an analysis of your January 2025 sample,
2.  a clear, production‑grade batch ingestion schema,
3.  end‑to‑end SQL & Python (PySpark) code for every medallion step,
4.  best‑practice/optimization notes, and
5.  three concise project titles.

***

## 1) Quick analysis of your January 2025 dataset

**File:** `202501-divvy-tripdata.csv` (Cyclistic/Divvy schema)

**Shape & structure**

*   **Rows × columns:** **138,689 × 13**. Columns: `ride_id, rideable_type, started_at, ended_at, start_station_name, start_station_id, end_station_name, end_station_id, start_lat, start_lng, end_lat, end_lng, member_casual`. 

**Types (inferred)**

*   `ride_id, rideable_type, start_station_name, start_station_id, end_station_name, end_station_id, member_casual` → string
*   `started_at, ended_at` → timestamp
*   `start_lat, start_lng, end_lat, end_lng` → double 

**Data quality highlights**

*   **Duplicate `ride_id`:** **0** (good). 
*   **Nulls:** `start_station_*` missing **22,852** rows; `end_station_*` missing **24,073** rows; `end_lat/end_lng` missing **61** rows. 
*   **Durations:** computed duration\_secs > 0 for all; **0** negative or zero durations in this sample. 
*   **Geo validation:** **61** rows have out‑of‑range/missing lat/lng on at least one end. 
*   **Self‑loops:** \~**2,704** same station id; \~**5,275** same coordinates. Useful for QA or business rules. 
*   **Haversine distance (km):** median \~**1.29 km**, 90th **3.65 km**, 99th **7.90 km**, max **32.83 km**; >50 km **0** rows. 
*   **Member mix:** **114,536** member (82.6%) vs **24,153** casual (17.4%). 
*   **Rideable types:** **89,071** electric vs **49,618** classic. 
*   **Temporal:** peaks around **16:00–18:00**; midweek busier; **53** trips start in Dec 2024 (cross‑month edge case). 
*   **Top stations (sample):** “Kingsbury St & Kinzie St”, “Canal St & Madison St”, “Clinton St & Washington Blvd” rank high for start/end. 

**Implications for ETL**

*   Keep Bronze **lossless** (ingest everything as‑is).
*   In Silver, apply rules: drop/route rows with **missing end lat/lng**, invalid geo, non‑positive durations, etc.
*   Enrich: add `ride_date`, `ride_hour`, `duration_sec/min`, and `distance_km` for analytics.

***

## 2) Batch ingestion schema (for the attached Cyclistic CSV)

Below is a **logical schema** for batch loads (Bronze) + **Silver extensions** used downstream.

### Source/Raw CSV → Bronze table schema

```text
ride_id            STRING        -- unique trip id (no duplicates in sample)
rideable_type      STRING        -- classic_bike | electric_bike
started_at         TIMESTAMP     -- trip start time (UTC)
ended_at           TIMESTAMP     -- trip end time (UTC)
start_station_name STRING        -- may be NULL
start_station_id   STRING        -- may be NULL
end_station_name   STRING        -- may be NULL
end_station_id     STRING        -- may be NULL
start_lat          DOUBLE        -- required in raw file
start_lng          DOUBLE        -- required in raw file
end_lat            DOUBLE        -- can be NULL (61 rows in sample)
end_lng            DOUBLE        -- can be NULL (61 rows in sample)
member_casual      STRING        -- member | casual
_ingest_file       STRING        -- (added) source file name
_ingest_ts         TIMESTAMP     -- (added) ingestion timestamp
```

**Bronze rules:** Keep it **append‑only**, no filters, minimal type coercions, add lineage columns (`_ingest_*`). (We’ll use CHECK constraints in Silver/Gold, not Bronze.)

### Silver (cleansed) schema (adds derived + constraints)

```text
ride_id            STRING        NOT NULL
rideable_type      STRING        NOT NULL
started_at         TIMESTAMP     NOT NULL
ended_at           TIMESTAMP     NOT NULL
ride_date          DATE          NOT NULL  -- generated from started_at
ride_hour          TINYINT       NOT NULL  -- 0..23
duration_sec       INT           NOT NULL  -- ended_at - started_at (must be >0)
duration_min       DECIMAL(9,2)  NOT NULL
start_station_id   STRING        -- nullable but prefer NOT NULL when available
start_station_name STRING
end_station_id     STRING
end_station_name   STRING
start_lat          DOUBLE        NOT NULL  -- enforce valid range
start_lng          DOUBLE        NOT NULL
end_lat            DOUBLE        NOT NULL
end_lng            DOUBLE        NOT NULL
distance_km        DECIMAL(9,3)  NOT NULL  -- haversine
is_self_loop       BOOLEAN       NOT NULL
member_casual      STRING        NOT NULL  -- CHECK (IN ('member','casual'))
_src_file          STRING        NOT NULL
_load_date         DATE          NOT NULL  -- for partitioning
```

**Silver quality gates (examples):**

*   `ended_at > started_at`
*   latitude in \[-90, 90], longitude in \[-180, 180] for **both** ends
*   `duration_sec > 0`
*   `member_casual IN ('member','casual')`

**Partitioning recommendation:** Partition **Silver** & **Gold** by `ride_date` or `_load_date` (more stable for incremental loads). Use **Z‑ORDER** on `started_at`, `member_casual`, and popular station columns for query skipping.

***

## 3) Databricks implementation (SQL **and** Python)

Assumptions to keep this Azure‑native:

*   Storage: **Azure Data Lake Storage Gen2** (ADLS Gen2)
*   Mountless approach with **Unity Catalog** external locations & storage credentials (no dbutils mount).
*   Batch ingestion via **COPY INTO** (idempotent, tracks loaded files), with optional PySpark alternative.
*   Directory layout in ADLS Gen2:
    *   `abfss://raw@<storage_account>.dfs.core.windows.net/divvy/2025/01/*.csv` (landing)
    *   We’ll write Delta tables to UC‑managed storage (or an external location for each layer).

> Replace placeholders like `<storage_account>`, `<container>`, `<sp_client_id>` etc. with your actual values/secrets.

***

### 3.1. Prereqs: Unity Catalog + storage bindings

**SQL (admin once per workspace)**

```sql
-- 1) Create storage credential (Service Principal or Managed Identity)
CREATE STORAGE CREDENTIAL sc_divvy
  WITH AZURE_SERVICE_PRINCIPAL (
    CLIENT_ID     = '${secrets.azure.sp_client_id}',
    CLIENT_SECRET = '${secrets.azure.sp_client_secret}'
  )
  AZURE_TENANT_ID = '${secrets.azure.tenant_id}';

-- 2) Create external locations for raw and curated
CREATE EXTERNAL LOCATION loc_divvy_raw
  URL 'abfss://raw@<storage_account>.dfs.core.windows.net/divvy'
  WITH (STORAGE CREDENTIAL sc_divvy);

CREATE EXTERNAL LOCATION loc_divvy_curated
  URL 'abfss://curated@<storage_account>.dfs.core.windows.net/divvy'
  WITH (STORAGE CREDENTIAL sc_divvy);

-- Optional: volumes if you prefer UC Volumes
```

***

### 3.2. Create catalog & schemas (Medallion)

**SQL**

```sql
CREATE CATALOG IF NOT EXISTS cyclistic COMMENT 'Cyclistic/Divvy demo';
USE CATALOG cyclistic;

CREATE SCHEMA IF NOT EXISTS bronze COMMENT 'Raw ingested (lossless)';
CREATE SCHEMA IF NOT EXISTS silver COMMENT 'Cleansed & conformed';
CREATE SCHEMA IF NOT EXISTS gold   COMMENT 'Modeled analytics (facts/aggregations)';
```

**Python (PySpark)**

```python
spark.sql("CREATE CATALOG IF NOT EXISTS cyclistic")
spark.sql("USE CATALOG cyclistic")
for s in ["bronze","silver","gold"]:
    spark.sql(f"CREATE SCHEMA IF NOT EXISTS {s}")
```

***

### 3.3. **Bronze** – Batch ingestion (idempotent)

**Approach A: COPY INTO (SQL)**

```sql
USE CATALOG cyclistic;
USE SCHEMA bronze;

-- Create an empty Delta table with the raw schema
CREATE TABLE IF NOT EXISTS trips_raw (
  ride_id            STRING,
  rideable_type      STRING,
  started_at         TIMESTAMP,
  ended_at           TIMESTAMP,
  start_station_name STRING,
  start_station_id   STRING,
  end_station_name   STRING,
  end_station_id     STRING,
  start_lat          DOUBLE,
  start_lng          DOUBLE,
  end_lat            DOUBLE,
  end_lng            DOUBLE,
  member_casual      STRING,
  _ingest_file       STRING,
  _ingest_ts         TIMESTAMP
)
COMMENT 'Raw Cyclistic trips (as-is) stored as Delta';

-- Load January files. COPY INTO is idempotent: it tracks which files have been loaded.
COPY INTO trips_raw
FROM (
  SELECT
    ride_id,
    rideable_type,
    to_timestamp(started_at)  AS started_at,
    to_timestamp(ended_at)    AS ended_at,
    start_station_name,
    start_station_id,
    end_station_name,
    end_station_id,
    CAST(start_lat AS DOUBLE) AS start_lat,
    CAST(start_lng AS DOUBLE) AS start_lng,
    CAST(end_lat   AS DOUBLE) AS end_lat,
    CAST(end_lng   AS DOUBLE) AS end_lng,
    member_casual,
    _metadata.file_name       AS _ingest_file,
    current_timestamp()       AS _ingest_ts
  FROM 'abfss://raw@<storage_account>.dfs.core.windows.net/divvy/2025/01/'
)
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true', 'multiLine'='false')
COPY_OPTIONS ('mergeSchema'='true');  -- safe for extra cols in future
```

**Approach B: PySpark (explicit schema + write Delta)**

```python
from pyspark.sql.types import *
from pyspark.sql.functions import input_file_name, current_timestamp

schema = StructType([
    StructField("ride_id", StringType()),
    StructField("rideable_type", StringType()),
    StructField("started_at", TimestampType()),
    StructField("ended_at", TimestampType()),
    StructField("start_station_name", StringType()),
    StructField("start_station_id", StringType()),
    StructField("end_station_name", StringType()),
    StructField("end_station_id", StringType()),
    StructField("start_lat", DoubleType()),
    StructField("start_lng", DoubleType()),
    StructField("end_lat", DoubleType()),
    StructField("end_lng", DoubleType()),
    StructField("member_casual", StringType())
])

src = "abfss://raw@<storage_account>.dfs.core.windows.net/divvy/2025/01/"
df_bronze = (spark.read
    .format("csv").option("header","true")
    .schema(schema)
    .load(src)
    .withColumn("_ingest_file", input_file_name())
    .withColumn("_ingest_ts", current_timestamp())
)

(df_bronze.write
  .format("delta")
  .mode("append")
  .saveAsTable("cyclistic.bronze.trips_raw"))
```

***

### 3.4. **Silver** – Cleansing, standardization & enrichment

**Business rules applied**

*   keep only rows with non‑null `started_at`, `ended_at`, and **ended\_at > started\_at**
*   validate geo ranges for both start and end points
*   compute `duration_sec`, `duration_min`, `ride_date`, `ride_hour`, `distance_km` (Haversine)
*   mark `is_self_loop` if same station\_id or same coordinates
*   deduplicate by `ride_id` (Bronze has none now, but good to show the skill)
*   partition by `_load_date` ( = date(`_ingest_ts`)) or by `ride_date`

**SQL**

```sql
USE CATALOG cyclistic;
USE SCHEMA silver;

-- Haversine helper (inline for clarity)
CREATE OR REPLACE FUNCTION silver.haversine_km(
  lat1 DOUBLE, lon1 DOUBLE, lat2 DOUBLE, lon2 DOUBLE
) RETURNS DOUBLE
RETURN 2*6371*asin(sqrt(
  pow(sin(radians(lat2-lat1)/2),2) +
  cos(radians(lat1))*cos(radians(lat2))*pow(sin(radians(lon2-lon1)/2),2)
));

-- Create Silver table
CREATE TABLE IF NOT EXISTS trips_clean
(
  ride_id            STRING NOT NULL,
  rideable_type      STRING NOT NULL,
  started_at         TIMESTAMP NOT NULL,
  ended_at           TIMESTAMP NOT NULL,
  ride_date          DATE NOT NULL,
  ride_hour          TINYINT NOT NULL,
  duration_sec       INT NOT NULL,
  duration_min       DECIMAL(9,2) NOT NULL,
  start_station_id   STRING,
  start_station_name STRING,
  end_station_id     STRING,
  end_station_name   STRING,
  start_lat          DOUBLE NOT NULL,
  start_lng          DOUBLE NOT NULL,
  end_lat            DOUBLE NOT NULL,
  end_lng            DOUBLE NOT NULL,
  distance_km        DECIMAL(9,3) NOT NULL,
  is_self_loop       BOOLEAN NOT NULL,
  member_casual      STRING NOT NULL,
  _src_file          STRING NOT NULL,
  _load_date         DATE NOT NULL
)
USING DELTA
PARTITIONED BY (_load_date)
TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite'='true',
  'delta.autoOptimize.autoCompact'='true'
);

-- Upsert into Silver
MERGE INTO trips_clean t
USING (
  SELECT
    br.ride_id,
    br.rideable_type,
    br.started_at,
    br.ended_at,
    DATE(br.started_at)              AS ride_date,
    HOUR(br.started_at)              AS ride_hour,
    CAST((unix_timestamp(br.ended_at)-unix_timestamp(br.started_at)) AS INT) AS duration_sec,
    ROUND((unix_timestamp(br.ended_at)-unix_timestamp(br.started_at))/60.0,2) AS duration_min,
    br.start_station_id,
    br.start_station_name,
    br.end_station_id,
    br.end_station_name,
    br.start_lat, br.start_lng, br.end_lat, br.end_lng,
    CAST(silver.haversine_km(br.start_lat, br.start_lng, br.end_lat, br.end_lng) AS DECIMAL(9,3)) AS distance_km,
    (br.start_station_id = br.end_station_id
     OR (round(br.start_lat,6)=round(br.end_lat,6) AND round(br.start_lng,6)=round(br.end_lng,6))) AS is_self_loop,
    br.member_casual,
    br._ingest_file AS _src_file,
    DATE(br._ingest_ts) AS _load_date
  FROM cyclistic.bronze.trips_raw br
  WHERE br.started_at IS NOT NULL
    AND br.ended_at   IS NOT NULL
    AND br.ended_at   >  br.started_at
    AND br.start_lat BETWEEN -90 AND 90
    AND br.end_lat   BETWEEN -90 AND 90
    AND br.start_lng BETWEEN -180 AND 180
    AND br.end_lng   BETWEEN -180 AND 180
) s
ON t.ride_id   = s.ride_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- Add Silver-level data quality constraints (fail-fast or informative)
ALTER TABLE trips_clean SET TBLPROPERTIES (
  'quality.duration_pos' = 'ended_at > started_at',
  'quality.geo_valid'    = 'lat/lng within range'
);

-- Optional performance tune
OPTIMIZE trips_clean ZORDER BY (ride_date, member_casual, start_station_id);
```

**Python (PySpark)**

```python
from pyspark.sql import functions as F, types as T

spark.sql("USE CATALOG cyclistic")
spark.sql("USE SCHEMA silver")

br = spark.table("cyclistic.bronze.trips_raw")

haversine = F.udf(
    lambda lat1,lon1,lat2,lon2: float(
        2*6371*np.arcsin(np.sqrt(
            np.sin(np.radians(lat2-lat1)/2)**2 +
            np.cos(np.radians(lat1))*np.cos(np.radians(lat2))*np.sin(np.radians(lon2-lon1)/2)**2
        ))
    ), T.DoubleType()
)

slv = (br
  .filter(
      F.col("started_at").isNotNull() &
      F.col("ended_at").isNotNull()   &
      (F.col("ended_at") > F.col("started_at")) &
      (F.col("start_lat").between(-90,90)) &
      (F.col("end_lat").between(-90,90))   &
      (F.col("start_lng").between(-180,180)) &
      (F.col("end_lng").between(-180,180))
  )
  .withColumn("ride_date", F.to_date("started_at"))
  .withColumn("ride_hour", F.hour("started_at").cast("tinyint"))
  .withColumn("duration_sec", (F.col("ended_at").cast("long") - F.col("started_at").cast("long")).cast("int"))
  .withColumn("duration_min", F.round(F.col("duration_sec")/60.0, 2))
  .withColumn("distance_km", F.round(
      haversine("start_lat","start_lng","end_lat","end_lng"), 3))
  .withColumn("is_self_loop",
      (F.col("start_station_id") == F.col("end_station_id")) |
      ((F.round("start_lat",6) == F.round("end_lat",6)) & (F.round("start_lng",6) == F.round("end_lng",6)))
  )
  .withColumn("_src_file", F.col("_ingest_file"))
  .withColumn("_load_date", F.to_date(F.col("_ingest_ts")))
  .drop("_ingest_ts","_ingest_file")
  .dropDuplicates(["ride_id"])
)

(slv.write
 .format("delta")
 .mode("append")
 .partitionBy("_load_date")
 .option("mergeSchema","true")
 .saveAsTable("cyclistic.silver.trips_clean"))

spark.sql("OPTIMIZE cyclistic.silver.trips_clean ZORDER BY (ride_date, member_casual, start_station_id)")
```

***

### 3.5. **Gold** – Dim/Facts & aggregates for analytics

**Star-ish model** (simple & interview‑friendly):

*   `gold.dim_station` (distinct station attributes from Silver)
*   `gold.fact_trips` (duration, distance, ride type, member type, denormalized station ids)
*   `gold.daily_kpis` (aggregations by date, member type, rideable type)
*   `gold.station_hourly` (traffic by station & hour)

**SQL**

```sql
USE CATALOG cyclistic;
USE SCHEMA gold;

-- Dimensions
CREATE OR REPLACE TABLE dim_station AS
SELECT DISTINCT
  COALESCE(start_station_id, end_station_id) AS station_id,
  COALESCE(start_station_name, end_station_name) AS station_name
FROM cyclistic.silver.trips_clean
WHERE station_id IS NOT NULL;

-- Fact (can be kept narrow; we already have Silver with clean columns)
CREATE TABLE IF NOT EXISTS fact_trips
USING DELTA
PARTITIONED BY (ride_date)
AS
SELECT
  ride_id, ride_date, ride_hour,
  member_casual, rideable_type,
  start_station_id, end_station_id,
  duration_sec, duration_min, distance_km, is_self_loop
FROM cyclistic.silver.trips_clean;

-- Aggregates
CREATE OR REPLACE TABLE daily_kpis AS
SELECT
  ride_date,
  member_casual,
  rideable_type,
  COUNT(*)                      AS trips,
  ROUND(AVG(duration_min),2)    AS avg_duration_min,
  ROUND(AVG(distance_km),3)     AS avg_distance_km
FROM cyclistic.silver.trips_clean
GROUP BY ride_date, member_casual, rideable_type;

CREATE OR REPLACE TABLE station_hourly AS
SELECT
  ride_date,
  ride_hour,
  start_station_id,
  COUNT(*) AS departures
FROM cyclistic.silver.trips_clean
GROUP BY ride_date, ride_hour, start_station_id;

-- Perf tune
OPTIMIZE fact_trips ZORDER BY (member_casual, rideable_type);
OPTIMIZE daily_kpis ZORDER BY (ride_date);
```

**Python (PySpark)**

```python
spark.sql("USE CATALOG cyclistic"); spark.sql("USE SCHEMA gold")

slv = spark.table("cyclistic.silver.trips_clean")

# dim_station
dim = (slv.select(
        F.coalesce("start_station_id","end_station_id").alias("station_id"),
        F.coalesce("start_station_name","end_station_name").alias("station_name"))
      .where(F.col("station_id").isNotNull())
      .dropDuplicates(["station_id"]))
dim.write.mode("overwrite").saveAsTable("cyclistic.gold.dim_station")

# fact_trips
fact = slv.select(
    "ride_id","ride_date","ride_hour","member_casual","rideable_type",
    "start_station_id","end_station_id","duration_sec","duration_min","distance_km","is_self_loop"
)
(fact.write
  .format("delta").mode("overwrite")
  .partitionBy("ride_date")
  .saveAsTable("cyclistic.gold.fact_trips"))

# daily_kpis
daily = (slv.groupBy("ride_date","member_casual","rideable_type")
          .agg(F.count("*").alias("trips"),
               F.round(F.avg("duration_min"),2).alias("avg_duration_min"),
               F.round(F.avg("distance_km"),3).alias("avg_distance_km")))
daily.write.mode("overwrite").saveAsTable("cyclistic.gold.daily_kpis")
```

***

### 3.6. Optional: Operational touches

*   **Schedules:** Job to run **daily** for prior day’s files, or weekly backfills.
*   **Idempotency:** `COPY INTO` ensures the same file won’t be re‑loaded; maintain file naming convention (`YYYY/MM/*.csv`) for atomic batches.
*   **Observability:** Add simple QA counts (rows in/out) and a “quarantine” table for rejected records.

***

## 4) Best practices & optimizations (Azure + Databricks + Delta)

*   **Governance with Unity Catalog**: centralize permissions (catalog → schema → table), use **storage credentials** + **external locations**; avoid ad‑hoc mounts.
*   **Batch ingestion:** Prefer **COPY INTO** for simple, idempotent batch loads; for continuous incremental later, use **Auto Loader** with schema evolution and file notification.
*   **Schema management:** Pin the Bronze schema explicitly; allow `mergeSchema` only where safe (e.g., new nullable cols). Add **column comments** for documentation.
*   **Quality gates:** Keep **Bronze lossless**; enforce constraints in **Silver** (e.g., `duration_sec > 0`, geo ranges). Route failed rows to a **quarantine** table for inspection.
*   **Partitioning:** Partition by a **stable** column (`_load_date` or `ride_date`). Avoid too many small partitions.
*   **File size & small files:** Enable **Optimize Write + Auto‑Compact**; schedule `OPTIMIZE` and `VACUUM` (e.g., weekly) to compact data and clean old files.
*   **Z‑ORDER clustering:** Z‑ORDER frequently filtered columns (`ride_date`, `member_casual`, `start_station_id`) to improve data skipping.
*   **Photon + AQE:** Run jobs on **Photon‑enabled** clusters; keep **Adaptive Query Execution** on for better join strategies and skew handling.
*   **Secrets & least privilege:** Store SP credentials in **Databricks Secrets**; scope storage credential permissions to required containers.
*   **Cost containment:** Autoscaling with sensible min/max; prefer **job clusters** for batch; cache only what’s reused; prune columns early.
*   **Reproducibility:** Tag each load with `_src_file`, `_load_date`; keep job runs versioned (notebook revision or repos).

***

## 5) Three concise, professional project titles

1.  **“Cyclistic on the Lake: A Medallion ETL with Delta & Unity Catalog”**
2.  **“RideWise: Azure Databricks Batch Pipelines for Cyclistic”**
3.  **“Pedal to Insights: Bronze–Silver–Gold on Delta Lake”**

***

## What you’ll be able to demo in interviews

*   Clean **Medallion** story (governed catalog, batch ingestion, cleansing, modeling).
*   **SQL & PySpark** proficiency (same pipeline expressed both ways).
*   Quality rules, partitioning, Z‑ORDER, and operational patterns (idempotency, optimize, vacuum).
*   Practical EDA of a real month of data—outliers, station nulls, self‑loops, member mix—grounded in the file you supplied. 

***

If you want, I can package this into a **single Databricks notebook** (with markdown cells + runnable SQL/Python) or generate a **README.md** you can push to GitHub for your portfolio. Also—do you want me to swap `COPY INTO` for **Auto Loader** so you can talk about incremental ingestion too, or keep it strictly batch for now?
