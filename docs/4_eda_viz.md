# 📊 Exploratory Data Analysis & Visualization

Exploratory Data Analysis (EDA) and Visualization are important steps in the data analysis process. This is the stage where a Data/Business Analyst explores the data to understand its structure, identify patterns, and gain insights that can inform our analysis and decision-making. We're not an analyst here, but hey, we can do EDA too!

In this section, we will perform EDA on the cleaned and conformed data in the Silver layer. Our goal in this analysis is to gain insights into the usage patterns of Cyclistic's bike-sharing service, identify trends, and (hopefully) help the business make informed decisions.


### Q1: Which station is the most popular among Cyclistic users?

```sql
SELECT
    start_station_name,
    COUNT(*) AS trip_count
FROM cyclistic.silver.trips_clean
WHERE start_station_name IS NOT NULL
GROUP BY start_station_name
ORDER BY trip_count DESC
LIMIT 1;
```

Output:

| start_station_name       | trip_count |
| ------------------------ | ---------- |
| Kingsbury St & Kinzie St | 40587      |


    The most popular station is **Kingsbury St & Kinzie St**, with a total of **40,587 trips**.


### Q2: At which period of the day and week is the bike activity the highest? Create a profile of bike activity across the day and week.

```sql
%sql
SELECT
    EXTRACT(DAYOFWEEK FROM ride_date) AS day_of_week, -- starts Sunday
    ride_hour,
    COUNT(*) AS trip_count
FROM cyclistic.silver.trips_clean
WHERE ride_hour IS NOT NULL
  AND ride_date IS NOT NULL
GROUP BY day_of_week, ride_hour
ORDER BY day_of_week, ride_hour;
```

[Chart here]

    Trip activity varies significantly by both hour and day of the week, with the highest volumes generally occurring in the afternoon and evening hours. Notable data points include:

    * Sunday (day 1): Peak at 3pm with 60,089 trips 
    * Monday (day 2): Peak at 5pm with 87,774 trips 
    * Tuesday (day 3): Peak at 5pm with 96,808 trips 
    * Early morning hours (midnight to 5am) consistently have the lowest trip counts across all days 
    * Activity rises sharply after 6am and remains high through the evening


### Q3: What's the distribution of classic and electric bike users in each month? Create a profile of bike activity across the year.

```sql
SELECT
    DATE_TRUNC('MONTH', started_at) AS month,
    rideable_type,
    member_casual,
    COUNT(*) AS trip_count
FROM cyclistic.silver.trips_clean
WHERE started_at IS NOT NULL
  AND rideable_type IS NOT NULL
GROUP BY month, rideable_type, member_casual
ORDER BY month, rideable_type, member_casual;
```

[Chart here]

    Trip activity by month shows a clear seasonal trend, with the highest usage in summer months and lower activity in winter. Electric bikes consistently have more trips than classic bikes each month. Examples include:

    January 2025: 89,049 electric bike trips, 49,541 classic bike trips June 2025: 424,748 electric bike trips, 253,092 classic bike trips August 2025: 509,267 electric bike trips, 280,343 classic bike trips November 2025: 241,091 electric bike trips, 115,050 classic bike trips December 2025: 95,667 electric bike trips, 44,736 classic bike trips

    Peak activity occurs from May to September, with electric bikes showing the largest growth during these months.


### Q4: Which type of user use the bikes longer? Determine the average ride duration in minutes.

```sql
SELECT
    member_casual,
    AVG(duration_min) AS avg_duration_min
FROM cyclistic.silver.trips_clean
WHERE member_casual IS NOT NULL
  AND duration_min IS NOT NULL
GROUP BY member_casual;
```

[Chart here]

    The average trip duration for members is 11.97 minutes, while for casual riders it is 19.21 minutes. Casual riders tend to take longer trips compared to members.


### Q5: How does the distance covered vary by ride type across different start stations?

```sql
SELECT
    rideable_type,
    start_station_name,
    SUM(distance_km) AS total_distance_km
FROM cyclistic.silver.trips_clean
WHERE rideable_type IS NOT NULL
  AND start_station_name IS NOT NULL
  AND distance_km IS NOT NULL
GROUP BY rideable_type, start_station_name
ORDER BY total_distance_km DESC
LIMIT 20;
```

[Chart here]

    The distance covered varies significantly by ride type and start station. Notable data points include:

    * Classic bikes at DuSable Lake Shore Dr & North Blvd: 52,178.5 km
    * Classic bikes at Michigan Ave & Oak St: 51,242.2 km
    * Electric bikes at Michigan Ave & Oak St: 44,005.9 km
    * Electric bikes at Navy Pier: 40,848.3 km
    * Classic bikes at Navy Pier: 47,128.3 km