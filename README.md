# Databricks Data Warehousing Project

## Introduction

Welcome to the ...


## Understand the Business Requirements

The business wanted to understand the following:
- Who are the most valuable customers?
- What are the most popular bike routes?
- What are the most popular bike types?


## The Cyclistic Dataset

### Quick Overview of the Dataset

**File:** `202501-divvy-tripdata.csv` (Cyclistic/Divvy schema)
...

## Build the Azure Databricks Infrastructure

### Create the Databricks Workspace

![1770637481388](image/README/1770637481388.png)

### Create an Access Connector

![1770637584703](image/README/1770637584703.png)

![1770637630239](image/README/1770637630239.png)

### Create the Storage Account

![1770637685309](image/README/1770637685309.png)

### Enable Access to the Storage Account

![1770637797073](image/README/1770637797073.png)

### Create A Storage Credential in Databricks

![1770637904308](image/README/1770637904308.png)

### Create A Container

![1770638361588](image/README/1770638361588.png)

### Create An External Location

![1770637951731](image/README/1770637951731.png)

### Upload the Datasets

![1770638062683](image/README/1770638062683.png)






## Build the Catalog & Schemas

```sql
CREATE CATALOG IF NOT EXISTS cyclistic 
MANAGED LOCATION 'abfss://deprojectcontainer@deprojectextdatalake.dfs.core.windows.net/'
COMMENT 'Catalog for Cyclistic/Divvy';
```

```sql
DESCRIBE CATALOG EXTENDED cyclistic;
```

```sql
USE CATALOG cyclistic;

CREATE SCHEMA IF NOT EXISTS landing COMMENT 'Raw CSV files';
CREATE SCHEMA IF NOT EXISTS bronze COMMENT 'Raw ingested (lossless)';
CREATE SCHEMA IF NOT EXISTS silver COMMENT 'Cleansed & conformed';
CREATE SCHEMA IF NOT EXISTS gold   COMMENT 'Modeled analytics (facts/aggregations)';
```

## Build the Landing Schema

```sql
USE CATALOG cyclistic;
USE SCHEMA landing;

CREATE EXTERNAL VOLUME IF NOT EXISTS divvy_trip_data
LOCATION 'abfss://deprojectcontainer@deprojectextdatalake.dfs.core.windows.net/divvy_trip_data/'
COMMENT 'External volume for raw data';
```

## Build Bronze Layer

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
```

### Analyze Source Systems


### Create DDL for Tables


### Develop SQL Load Scripts

```sql
-- Load all files. COPY INTO is idempotent: it tracks which files have been loaded.
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
  FROM '/Volumes/cyclistic/landing/divvy_trip_data'
)
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true', 'multiLine'='false')
COPY_OPTIONS ('mergeSchema'='true');  -- safe for extra cols in future
```

## Build Silver Layer

```sql
USE CATALOG cyclistic;
USE SCHEMA silver;
```

```sql
-- Haversine function
CREATE OR REPLACE FUNCTION silver.haversine_km(
  lat1 DOUBLE, lon1 DOUBLE, lat2 DOUBLE, lon2 DOUBLE
) RETURNS DOUBLE
RETURN 2*6371*asin(sqrt(
  pow(sin(radians(lat2-lat1)/2),2) +
  cos(radians(lat1))*cos(radians(lat2))*pow(sin(radians(lon2-lon1)/2),2)
));
```

```sql

```

```sql

```



```sql

```

### Explore & Understand The Data

### Create DDL for Tables

```sql
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
```

### Clean & Load ___

```sql
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
    CAST((unix_timestamp(br.ended_at)-unix_timestamp(br.started_at)) AS INT)      AS duration_sec,
    ROUND((unix_timestamp(br.ended_at)-unix_timestamp(br.started_at))/60.0,2)     AS duration_min,
    br.start_station_id,
    br.start_station_name,
    br.end_station_id,
    br.end_station_name,
    br.start_lat, br.start_lng, br.end_lat, br.end_lng,
    CAST(silver.haversine_km(br.start_lat, br.start_lng, br.end_lat, br.end_lng) AS DECIMAL(9,3))       AS distance_km,
    (br.start_station_id <=> br.end_station_id 
     OR (round(br.start_lat,6) <=> round(br.end_lat,6) AND round(br.start_lng,6) <=> round(br.end_lng,6)))      AS is_self_loop,
    br.member_casual,
    br._ingest_file       AS _src_file,
    DATE(br._ingest_ts)   AS _load_date
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
```

```sql
-- Add Silver-level data quality constraints (fail-fast or informative)
ALTER TABLE trips_clean SET TBLPROPERTIES (
  'quality.duration_pos' = 'ended_at > started_at',
  'quality.geo_valid'    = 'lat/lng within range'
);
```

```sql
-- Optional performance tuning using ZORDER
OPTIMIZE trips_clean ZORDER BY (ride_date, member_casual, start_station_id);
```

## Build Gold Layer

```sql
USE CATALOG cyclistic;
USE SCHEMA gold;
```

### Create the Fact and Dimension Tables

```sql
-- Dimensions: List of Stations
CREATE OR REPLACE TABLE dim_station AS
SELECT DISTINCT
  COALESCE(start_station_id, end_station_id) AS station_id,
  COALESCE(start_station_name, end_station_name) AS station_name
FROM cyclistic.silver.trips_clean
WHERE COALESCE(start_station_id, end_station_id) IS NOT NULL;
```

```sql
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

-- Perf tune
OPTIMIZE fact_trips ZORDER BY (member_casual, rideable_type);
```


### Create the KPIs and Metrics

```sql
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
OPTIMIZE daily_kpis ZORDER BY (ride_date);
```

## Wrap Up



## 🌟 About Me

Hi there! I'm **Paul Joseph Mendoza**. I’m an IT professional and passionate YouTuber on a mission to share knowledge and make working with data enjoyable and engaging!

Let's stay in touch! Feel free to connect with me on the following platforms:

[![LinkedIn]](https://www.linkedin.com/in/paul-joseph-mendoza/)
[![Website]](https://www.datawithbaraa.com)