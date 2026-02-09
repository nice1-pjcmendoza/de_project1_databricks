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



### 


## Build the Catalog & Schemas



## Build Bronze Layer

### Analyze Source Systems

### Create DDL for Tables

### Develop SQL Load Scripts



## Build Silver Layer

### Explore & Understand The Data

### Create DDL for Tables

### Clean & Load ___



## Build Gold Layer

### Create the Fact and Dimension Tables

### Create the KPIs and Metrics



## Wrap Up



## 🌟 About Me

Hi there! I'm **Paul Joseph Mendoza**. I’m an IT professional and passionate YouTuber on a mission to share knowledge and make working with data enjoyable and engaging!

Let's stay in touch! Feel free to connect with me on the following platforms:

[![LinkedIn]](https://www.linkedin.com/in/paul-joseph-mendoza/)
[![Website]](https://www.datawithbaraa.com)