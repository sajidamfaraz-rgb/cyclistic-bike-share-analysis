![Cyclistic-Logo](images/Header.png)

## **Ask**

Introduction and Business Objective: Cyclistic, a bike-share company, launched in 2016, operates 5,824 bicycles across 692 stations in Chicago, offering flexible pricing plans. Customers are categorized as casual riders (single-ride and day passes) and annual members. Cyclistic believes that Annual members generate higher profits than casual riders, making membership growth essential to Cyclistic's future success.

**Stake Holders:**

1.  Lily Moreno: The Marketing Director, overseeing the planning and execution of promotional campaigns and initiatives across email, social media, and other marketing platforms.

2.  Cyclistic Executive Team: The senior leadership responsible for evaluating and approving the proposed marketing strategy.

3.  Cyclistic Marketing Analytics Team: A group of data analysts tasked with gathering, examining, and presenting data to support informed marketing decisions and shape Cyclistic's overall marketing strategy.

**Business Goal:**

Moreno aims to convert existing casual riders into annual members by understanding their riding patterns, membership motivations, and digital marketing opportunities. To achieve this goal, her team will
develop targeted marketing strategies based on historical bike-trip data to identify usage trends, explore membership motivations, and assess the role of digital media in converting casual riders into members. The assigned task is to analyze _how annual members and casual riders use Cyclistic bikes differently_.

**Deliverables:**

A report with the folowing deliverables:

1.  Statement of the business task
2.  A description of all data sources used
3.  Documentation of any cleaning or manipulation of data
4.  Summary of analysis
5.  Supporting visualizations and key findings
6.  Top three recommendations based on analysis
    <br><br>

## **Prepare**

Since Cyclistic is a fictional company, this analysis will use publicly available data provided by Motivate International Inc. (acquired by Lyft in 2018) under the terms of its data license agreement.

**Historical Trip Data:**

1. The data is publicly accessible through cloud storage in ZIP format, containing CSV files.
2. Divvy's first-party trip data is owned by the City of Chicago and published monthly.
3. The dataset excludes trips conducted by staff for system maintenance and inspections.
4. All ride records are anonymized to protect user privacy.
5. The data is structured and contains the following attributes:
   - ride_id: Unique identifier assigned to each ride.
   - rideable_type: Type of bicycle, categorized as classic, docked, or electric.
   - started_at: Date and time when the ride began.
   - ended_at: Date and time when the ride ended.
   - start_station_name: Name of the departure station.
   - start_station_id: Unique identifier of the departure station.
   - end_station_name: Name of the arrival station.
   - end_station_id: Unique identifier of the arrival station.
   - start_lat: Latitude of the starting location.
   - start_lng: Longitude of the starting location.
   - end_lat: Latitude of the ending location.
   - end_lng: Longitude of the ending location.
   - member_casual: Indicates whether the rider is an annual member or a casual rider.

**ROCCC Assessment:**

ROCCC represents five criteria: Reliable, Original, Comprehensive, Current, and Cited. This framework helps assess the credibility, quality, and suitability of a dataset for analysis.

✓ Reliable (MODERATE): The dataset provides monthly trip records and station details, offering substantial information about bike usage. However, the presence of missing values may affect data accuracy and completeness.
✓ Original (HIGH): The information originates directly from Divvy's bike-sharing system, making it a primary data source.
✓ Comprehensive (MODERATE): The dataset includes key trip-related information but lacks demographic details and unique rider identifiers, restricting deeper analysis of individual riding behavior.
✓ Current (HIGH): Trip records are released regularly through monthly data updates.
✓ Cited (HIGH): The dataset is officially published by Divvy and maintained under the ownership of the City of Chicago, supporting its initiative to promote cycling as an alternative mode of transportation.

**Data Quality and Integrity:**

To maintain data accuracy and consistency, the datasets will undergo a series of preparation steps. These will include examining the records, eliminating duplicate entries, addressing missing and null values, standardizing data types, correcting formatting inconsistencies, resolving structural issues, and filtering out invalid records.

**Relevance of the Data to the Business Objective:**

The selected datasets are sufficient for addressing the business
objective, as they provide detailed trip information for both casual riders and annual members. Analyzing this information will help identify differences in their riding habits and usage patterns, providing the insights needed to answer the central business question.

**Data Constraints and Limitations:**

_Lack of Individual Rider Identification:_ Since each record only
identifies the rider as either a casual user or an annual member, we cannot determine individual riders' usage patterns, which could otherwise help Cyclistic target potential members with personalized membership offers.

_Privacy Restrictions:_ The dataset does not include personally
identifiable information (PII). Consequently, the absence of payment and customer identification details prevents us from tracking purchase history, verifying riders' location, or determining whether they have bought multiple single-ride passes.

_Limited Analysis Period:_ The scope of this analysis is restricted to historical trip records covering August 2025 through July 2026, which may not reflect longer-term usage patterns.

_Data Quality Issues:_ The dataset contains missing values, date
inconsistencies, and other irregularities. Additionally, the absence of demographic details and unique rider identifiers limits the depth of analysis and the ability to explore individual customer behavior.
<br><br>

## **Data Processing**

**Data Gathering:**

1. Transfer data from CSV files to PostgreSql tables using python:

```python
import os
import pandas as pd
from sqlalchemy import create_engine

# 2. Connect to PostgreSQL

engine = create_engine('postgresql://postgres:cyclistic@localhost:5432/cyclisticTripDB')

file_list = ['202508-divvy-tripdata', '202509-divvy-tripdata',
'202510-divvy-tripdata', '202511-divvy-tripdata','202512-divvy-tripdata', '202601-divvy-tripdata','202602-divvy-tripdata','202603-divvy-tripdata', '202604-divvy-tripdata', '202605-divvy-tripdata', '202606-divvy-tripdata','202607-divvy-tripdata']

for f_name in file_list:
    # Loading data
    file_path = os.path.join(f_name,'.csv')
    df = pd.read_csv(file_path)
    # Push the dataframe to a new table in postgres
    df.to_sql(f_name, engine, if_exists='replace', index=False)

    print(f'Data from file {f_name} successfully imported into PostgreSQL!')
```

2. Check for table structures before merging:

```postgresql
WITH table_structures AS (
    SELECT
        table_name,
        STRING_AGG(
            column_name || ':' || data_type,
            ', ' ORDER BY ordinal_position
        ) AS structure
    FROM information_schema.columns
    WHERE table_schema = 'public' and table_name like '%-divvy-tripdata'
    GROUP BY table_name
)
SELECT
    COUNT(*) AS total_tables,
    STRING_AGG(table_name, ', ') AS tables,
    structure
FROM table_structures
GROUP BY structure;
```

Result:<br>
All 12 tables follow the same structure.

3. Data Merging and backup:
   Step 1: merge 12 tables data into 1 table

```postgresql
CREATE OR REPLACE PROCEDURE merge_cyclistic_tables()
LANGUAGE plpgsql
AS $$
DECLARE
    tbl RECORD;
    inserted_rows BIGINT;
    total_rows BIGINT := 0;
BEGIN

    -- Clear existing data before refreshing
    TRUNCATE TABLE cyclisticsTripData;

    -- Read all monthly Cyclistic tables
    FOR tbl IN
        SELECT tablename
        FROM pg_tables
        WHERE schemaname = 'public' AND tablename LIKE '%-divvy-tripdata'
        ORDER BY tablename
    LOOP
        -- Inserting
        EXECUTE FORMAT(
            'INSERT INTO cyclisticsTripData
             SELECT * FROM public.%I',
            tbl.tablename
        );

        GET DIAGNOSTICS inserted_rows = ROW_COUNT;

        total_rows := total_rows + inserted_rows;

        RAISE NOTICE 'Table: % | Rows inserted: %',             tbl.tablename, inserted_rows;
    END LOOP;

    RAISE NOTICE 'Total records inserted: %', total_rows;

END;
$$;
```

Output:

```
NOTICE: Table: 202508-divvy-tripdata | Rows inserted: 790177
NOTICE: Table: 202509-divvy-tripdata | Rows inserted: 714759
NOTICE: Table: 202510-divvy-tripdata | Rows inserted: 646039
NOTICE: Table: 202511-divvy-tripdata | Rows inserted: 356628
NOTICE: Table: 202512-divvy-tripdata | Rows inserted: 140534
NOTICE: Table: 202601-divvy-tripdata | Rows inserted: 137787
NOTICE: Table: 202602-divvy-tripdata | Rows inserted: 201450
NOTICE: Table: 202603-divvy-tripdata | Rows inserted: 317037
NOTICE: Table: 202604-divvy-tripdata | Rows inserted: 448252
NOTICE: Table: 202605-divvy-tripdata | Rows inserted: 653704
NOTICE: Table: 202606-divvy-tripdata | Rows inserted: 762550
NOTICE: Table: 202607-divvy-tripdata | Rows inserted: 869051
NOTICE: Total records inserted: 6037968
CALL

Query returned successfully in 25 secs 608 msec.
```

Step 2: Create backup before making any changes to our database table (using power shell)

```
pg_dump -h localhost -p 5432 -U postgres -d cyclisticTripDB -t
cyclisticstripdata -F c -f
"E:`\Data`{=tex}\_Analysis`\Cyclisticstripdata`{=tex}.backup"
```

**Data standardization:**

Remove the leading and trailing spaces for fields with string data types.

```postgresql
update cyclisticstripdata set ride_id = trim(ride_id),
    rideable_type =  trim(rideable_type),
    start_station_name= trim(start_station_name),
    start_station_id =trim(start_station_id),
    end_station_name = trim(end_station_name),
    end_station_id = trim(end_station_id),
    member_casual = trim (member_casual);
```

Checked for ride Id length:

```postgresql
select distinct length(ride_id) as len_ride_id from cyclisticstripdata
```

Output:

<img src="images/image-1.png" width="120">
<br>

Check for station Id length:

```postgresql
select distinct length(start_station_id) as len_start_station_id from
cyclisticstripdata

select distinct length(end_station_id) as len_end_station_id from
cyclisticstripdata
```

Output:

<img src="images/image-2.png" width="130">
<img src="images/image-3.png" width="130">

<br>
Checking extra lengthy ids:

```postgresql
select distinct start_station_id
from cyclisticstripdata
where length(start_station_id)>8

select distinct end_station_id
from cyclisticstripdata
where length(end_station_id)>8
```

Output:

<img src="images/image-4.png" width="140">
<img src="images/image-5.png" width="140">

<br>
Correct ids and remove obvious typos.

```postgresql
update cyclisticstripdata
set start_station_id =SUBSTRING(start_station_id FROM POSITION('CHI0' IN start_station_id))
where length(start_station_id)>8

update cyclisticstripdata
set end_station_id =SUBSTRING(end_station_id FROM POSITION('CHI0' IN end_station_id))
where length(end_station_id)>8
```

**Duplicate Detection:**

Check for duplicate records using same ride_id.

```postgresql
SELECT *
FROM (
    SELECT *,
           COUNT(*) OVER (PARTITION BY ride_id) AS duplicate_count
    FROM cyclisticstripdata
) sub
WHERE duplicate_count > 1
ORDER BY ride_id;
```

Result: 70 duplicate rows found.

<img src="images/image-6.png">

Deduplicate records:

```postgresql
DELETE FROM cyclisticsTripData a
USING cyclisticsTripData b
WHERE a.ctid > b.ctid
  AND a.ride_id = b.ride_id;
```

**Station Name Standardization**

<u>Objective:</u> Ensure that each station ID is associated with a single standardized station name to prevent station usage from being split across multiple labels during analysis.

<u>Methodology:</u> After deduplication, 56 station IDs were identified with two different station names. To standardize these records, one name was selected for each station ID based on trip count and recent usage.

The station name with the highest trip count was prioritized, with recency considered to identify potential station renaming. A 60-day inactivity threshold was applied to determine whether an older station name had become obsolete. If the older name had not been used in the 60 days preceding the latest recorded usage, the newer name was selected as the standardized name. Otherwise, the older name was retained, assuming it remained the primary label.

<u>Outcome:</u> Each of the 56 identified station IDs was assigned a single standardized name, ensuring consistent station identification and more reliable station-level usage analysis.

```postgresql
CREATE TEMP TABLE temp_station AS

with dup_station as (
	select start_station_id as station_id
	from cyclisticstripdata
	WHERE start_station_id IS NOT NULL
    group by start_station_id
	having count(distinct start_station_name)>1
),
stations_usage as (
	select start_station_id as station_id, start_station_name as station_name,
		count(*)as trip_count, max(started_at) as last_used
	from cyclisticstripdata t
	where t.start_station_name IS NOT NULL
      AND Exists
		(select 1
		from dup_station d
		where d.station_id = t.start_station_id)
	group by start_station_id, start_station_name
),
station_rank as (
	select station_id, station_name,trip_count, last_used,
		row_number() over (partition by station_id order by trip_count desc) rn_trips,
		row_number() over (partition by station_id order by last_used desc) rn_recent
	from stations_usage
)

select a.station_id,
	(case when a.rn_recent=1 or (b.last_used-a.last_used)<INTERVAL '60 days'
		  then a.station_name
		  else b.station_name
	end) as station_name
from station_rank a
left join station_rank b on a.station_id = b.station_id and b.rn_trips>1 and b.rn_recent=1
where a.rn_trips = 1;

update cyclisticstripdata as c
set start_station_name = station_name
from temp_station as t
where t.station_id = c.start_station_id
	and t.station_name is distinct from c.start_station_name;

update cyclisticstripdata as c
set end_station_name = station_name
from temp_station as t
where t.station_id = c.end_station_id
	and t.station_name is distinct from c.end_station_name;

```

Standardized stations records:
<img src="images/image-7.png">

**Check for invalid values**

```postgresql
select distinct rideable_type
from cyclisticstripdata

select distinct member_casual
from cyclisticstripdata
```

No Invalid Values were found.

<img src="images/image-8.png" width="150">
<img src="images/image-9.png" width="150">
<br><br>

**Check for NULL values:**

```postgresql
SELECT
    COUNT(*) AS total_records,
	COUNT(*) FILTER (WHERE ride_id IS NULL) AS ride_id,
    COUNT(*) FILTER (WHERE started_at IS NULL) AS started_at,
    COUNT(*) FILTER (WHERE ended_at IS NULL) AS ended_at,
	COUNT(*) FILTER (WHERE start_station_name IS NULL) AS start_station_name,
    COUNT(*) FILTER (WHERE start_station_id IS NULL) AS start_station_id,
    COUNT(*) FILTER (WHERE end_station_name IS NULL) AS end_station_name,
    COUNT(*) FILTER (WHERE end_station_id IS NULL) AS end_station_id,
    COUNT(*) FILTER (WHERE start_lat IS NULL) AS start_lat,
    COUNT(*) FILTER (WHERE start_lng IS NULL) AS start_lng,
    COUNT(*) FILTER (WHERE end_lat IS NULL) AS end_lat,
    COUNT(*) FILTER (WHERE end_lng IS NULL) AS end_lng
FROM cyclisticstripdata;

```

<img src="images/image-10.png">

<br>

**Finding and Filling missing values for station-related columns**

As we can see there are null values in station-related columns.

```postgresql
SELECT
    COUNT(*) AS total_trips,
    COUNT(*) FILTER (WHERE start_station_name IS NULL) AS missing_start_name,
    COUNT(*) FILTER (WHERE start_station_id IS NULL) AS missing_start_id,
    COUNT(*) FILTER (WHERE end_station_name IS NULL) AS missing_end_name,
    COUNT(*) FILTER (WHERE end_station_id IS NULL) AS missing_end_id
FROM cyclisticstripdata;
```

<img src="images/image-11.png">

Now, checking records with missing station details but valid
coordinates.

Start Stations:

```postgresql
SELECT
    COUNT(*) AS missing_start_details_with_coordinates
FROM cyclisticstripdata
WHERE start_station_name IS NULL
  AND start_station_id IS NULL
  AND start_lat IS NOT NULL
  AND start_lng IS NOT NULL;
```

<img src="images/image-12.png">

<br>
End Stations:

```postgresql
SELECT
    COUNT(*) AS missing_end_details_with_coordinates
FROM cyclisticstripdata
WHERE end_station_name IS NULL
  AND end_station_id IS NULL
  AND end_lat IS NOT NULL
  AND end_lng IS NOT NULL;
```

<img src="images/image-13.png">

**Solution:**
With the given coordinate values of null staions, we will try
to find nearest known station. For this perpose we created tblStations table contains all known stations with id, name , lat, lng values:

```postgresql
create table tblStations
(station_id varchar(16), station_name varchar(100), lat numeric(9,6), lng numeric(9,6));

insert into tblStations
select distinct
	start_station_id as station_id,
	start_station_name as station_name,
	start_lat as lat,
	start_lng as lng
from cyclisticstripdata
where start_station_name IS NOT NULL
	AND start_station_id IS NOT NULL
	AND start_lat IS NOT NULL
	AND start_lng IS NOT NULL

Union
select distinct
	end_station_id as station_id,
	end_station_name as station_name,
	end_lat as lat,
	end_lng as lng
from cyclisticstripdata
where end_station_name IS NOT NULL
	AND end_station_id IS NOT NULL
	AND end_lat IS NOT NULL
	AND end_lng IS NOT NULL;
```

Now, calculating nearest stations list on lat and lng basis for null start station values in trip data, with the Haversine formula, which is a suitable approach for estimating the distance between two geographic coordinates. The nearest station name will be designated as the start station name and end station name.

First for Start stations:

```postgresql
with nearest_start_station as
(
	SELECT
	    t.ride_id,
	    t.start_lat,
	    t.start_lng,

	    s.station_id AS nearest_station_id,
	    s.station_name AS nearest_station_name,
	    s.lat AS nearest_latitude,
	    s.lng AS nearest_longitude,
	    s.distance_m
	FROM cyclisticstripdata AS t
	CROSS JOIN LATERAL (
	    SELECT
	        st.station_id,
	        st.station_name,
	        st.lat,
	        st.lng,
	        6371000 * ACOS(
	            LEAST(1.0, GREATEST(-1.0,
	                COS(RADIANS(t.start_lat))
	                * COS(RADIANS(st.lat))
	                * COS(RADIANS(st.lng - t.start_lng))
	                + SIN(RADIANS(t.start_lat))
	                * SIN(RADIANS(st.lat))
	            ))
	        ) AS distance_m
	    FROM tblStations AS st
	    ORDER BY distance_m ASC
	    LIMIT 1
	) AS s
	WHERE t.start_station_name IS NULL
		AND t.start_lat IS NOT NULL
		AND t.start_lng IS NOT NULL
)
update cyclisticstripdata
set start_station_id = s.nearest_station_id,
    start_station_name = nearest_station_name
from nearest_start_station s
where s.ride_id = cyclisticstripdata.ride_id;
```

Same for end stations null values:

```postgresql
with nearest_end_station as
(
	SELECT
	    t.ride_id,
	    t.end_lat,
	    t.end_lng,

	    s.station_id AS nearest_station_id,
	    s.station_name AS nearest_station_name,
	    s.lat AS nearest_latitude,
	    s.lng AS nearest_longitude,
	    s.distance_m
	FROM cyclisticstripdata AS t
	CROSS JOIN LATERAL (
	    SELECT
	        st.station_id,
	        st.station_name,
	        st.lat,
	        st.lng,
	        6371000 * ACOS(
	            LEAST(1.0, GREATEST(-1.0,
	                COS(RADIANS(t.end_lat))
	                * COS(RADIANS(st.lat))
	                * COS(RADIANS(st.lng - t.end_lng))
	                + SIN(RADIANS(t.end_lat))
	                * SIN(RADIANS(st.lat))
	            ))
	        ) AS distance_m
	    FROM tblStations AS st
	    ORDER BY distance_m ASC
	    LIMIT 1
	) AS s
	WHERE t.end_station_name IS NULL
		AND t.end_lat IS NOT NULL
		AND t.end_lng IS NOT NULL
)

update cyclisticstripdata
set end_station_id = s.nearest_station_id,
    end_station_name = nearest_station_name
from nearest_end_station s
where s.ride_id = cyclisticstripdata.ride_id;
```

Now, there are 278 more records which are still with null, end station id and name columns because lat and lng for those end stations are not given. So, we will fill those end station names with 'End Station Name Not Provided'

```postgresql
update cyclisticstripdata
set end_station_name='End Station Name Not Provided'
where end_station_name is null
```

Now, there is no trip with null station name and id.

**Removing Outliers**

Now, we will check for outliers in ride duration claculaion.

We, first create column ride_duration_minutes which will hold difference of end and start ride time

```postgresql
ALTER TABLE cyclisticstripdata
ADD COLUMN ride_duration_minutes DECIMAL(9,4);

update cyclisticstripdata
set ride_duration_minutes =(EXTRACT(EPOCH FROM (ended_at - started_at)) / 60) ;
```

The next step is to identify and remove ride-duration outliers. Rides lasting less than 1 minute will be classified as lower-bound outliers, as they may represent failed ride attempts or instances where users re-docked bikes to ensure they were securely locked. Similarly, rides exceeding 24 hours (1,440 minutes) will be classified as upper-bound outliers, as these unusually long durations may indicate lost or stolen
bikes. We will keep a copy of those deleted rides in an other table.

```postgresql
Create table deleted_trips as
select *
from cyclisticstripdata
where ride_duration_minutes < 1 or -- less than a minute OR negative durations
(start_time > end_time)
	ride_duration_minutes>= 1440	-- greater than 24 hrs

delete from cyclisticstripdata
where ride_duration_minutes < 1 or 	-- less than a minute OR negative
	ride_duration_minutes>= 1440	-- greater than 24 hrs
```

**Creating Derived Columns for Analysis:**

To support time-based analysis of bike usage, several date and time attributes were extracted from the started_at timestamp, including Year, Month, Month Name, Season, Day of Month, Day of Week, Hour, and Part of Day (Morning, Afternoon, Evening, and Night). These additional fields will help identify usage trends and compare riding patterns across different hours, days, months, and seasons.

```postgresql
ALTER TABLE cyclisticstripdata
    ADD COLUMN ride_year INT,
    ADD COLUMN ride_month INT,
    ADD COLUMN month_name VARCHAR(10),
    ADD COLUMN season VARCHAR(10),
    ADD COLUMN ride_day INT,
    ADD COLUMN day_of_week INT,
    ADD COLUMN day_name VARCHAR(10),
    ADD COLUMN ride_hour INT,
    ADD COLUMN hour_24_format VARCHAR(15);
    ADD COLUMN part_of_day VARCHAR(10);
```

```postgresql
UPDATE cyclisticstripdata
SET
    ride_year = EXTRACT(YEAR FROM started_at),
    ride_month = EXTRACT(MONTH FROM started_at),
    month_name = TO_CHAR(started_at, 'FMMonth'),

    season = CASE
        WHEN EXTRACT(MONTH FROM started_at) IN (3, 4, 5)
            THEN 'Spring'
        WHEN EXTRACT(MONTH FROM started_at) IN (6, 7, 8)
            THEN 'Summer'
        WHEN EXTRACT(MONTH FROM started_at) IN (9, 10, 11)
            THEN 'Autumn'
        WHEN EXTRACT(MONTH FROM started_at) IN (12, 1, 2)
            THEN 'Winter'
    END,

    ride_day = EXTRACT(DAY FROM started_at),
    day_of_week = EXTRACT(DOW FROM started_at),
    day_name = TO_CHAR(started_at, 'FMDay'),

    ride_hour = EXTRACT(HOUR FROM started_at),
    hour_24_format= TO_CHAR(started_at, 'FMHH12 AM')

    part_of_day = CASE
        WHEN EXTRACT(HOUR FROM started_at) >= 5
         AND EXTRACT(HOUR FROM started_at) < 12
            THEN 'Morning'
        WHEN EXTRACT(HOUR FROM started_at) >= 12
         AND EXTRACT(HOUR FROM started_at) < 17
            THEN 'Afternoon'
        WHEN EXTRACT(HOUR FROM started_at) >= 17
         AND EXTRACT(HOUR FROM started_at) < 21
            THEN 'Evening'
        ELSE 'Night'
    END
```

The data cleaning and processing stage is complete, and the dataset is now ready for analysis.
<br><br>

## **Data Analysis:**

In this phase, we will summarize the dataset using descriptive
statistics, carry out relevant computations, and examine the data to uncover meaningful patterns, correlations, and trends.

**Descriptive Statistics:**

we will first observe the most frequently occurring values for rider types, bike types, stations, and time-related variables.

```postgresql
SELECT
	COUNT(*) AS no_of_rows,
	COUNT(DISTINCT ride_id) AS total_rides,
	(SELECT member_casual from cyclisticstripdata
		GROUP  BY member_casual ORDER BY COUNT(*) DESC limit 1
	) AS mode_member_casual,
	(SELECT rideable_type FROM cyclisticstripdata
		GROUP  BY rideable_type ORDER  BY COUNT(*) DESC limit 1
	) AS mode_rideable_type,
	(SELECT start_station_name FROM cyclisticstripdata
		GROUP BY start_station_name ORDER  BY COUNT(*) DESC limit 1
	) AS mode_start_station_name,
	(SELECT end_station_name FROM cyclisticstripdata
		GROUP  BY end_station_name ORDER BY COUNT(*) DESC limit 1
	) AS mode_end_station_name,
	(SELECT season FROM cyclisticstripdata
		GROUP  BY season ORDER BY COUNT(*) DESC limit 1
	) AS mode_season,
	(SELECT ride_month FROM cyclisticstripdata
		GROUP  BY ride_month ORDER BY COUNT(*) DESC limit 1
	) AS mode_month,
	(SELECT day_name FROM cyclisticstripdata
		GROUP  BY day_name ORDER BY COUNT(*) DESC limit 1
	) AS mode_day_of_week,
	(SELECT part_of_day FROM cyclisticstripdata
		GROUP  BY part_of_day ORDER BY COUNT(*) DESC limit 1
	) AS mode_part_of_day,
	(SELECT ride_hour FROM cyclisticstripdata
		GROUP BY ride_hour ORDER BY COUNT(*) DESC limit 1
	) AS mode_hour
FROM cyclisticstripdata;
```

Result:

<img src="images/image-14.png">

<u>Key Findings:</u>

1. Rider Type: Members account for more rides than casual riders.
2. Bike Type: Electric bikes are the most frequently used bike type.
3. Station Preference: Navy Pier Station is the most frequently recorded starting and ending station.
4. Seasonal Demand: Summer is the busiest season for Cyclistic rides.
5. Monthly Demand: July records the highest number of rides.
6. Day of the Week: Saturday is the busiest day for Cyclistic rides.
7. Time of Day: Afternoon is the most active period for bike usage.
8. Peak Hour: 5:00 PM is the busiest hour for Cyclistic rides.

<br>

**Descritive analysis of Ride Duration:**

```postgresql
SELECT
    COUNT(ride_duration_minutes) AS count,
    AVG(ride_duration_minutes) AS average,
    STDDEV(ride_duration_minutes) AS std,
    MIN(ride_duration_minutes) AS min,
    MAX(ride_duration_minutes) AS max,

    PERCENTILE_CONT(0.25)
        WITHIN GROUP (ORDER BY ride_duration_minutes) AS "25th_percentile",

    PERCENTILE_CONT(0.50)
        WITHIN GROUP (ORDER BY ride_duration_minutes) AS "50th_percentile",

    PERCENTILE_CONT(0.75)
        WITHIN GROUP (ORDER BY ride_duration_minutes) AS "75th_percentile"

FROM cyclisticstripdata;
```

Result:

<img src="images/image-15.png">

<u>Key Findings:</u>

1. The average ride duration is 14.62 minutes, indicating that Cyclistic riders generally use the service for relatively short trips.
2. The shortest ride lasts 1 minute, while the longest lasts 1,439.97 minutes, equivalent to almost 24 hours.
3. The percentile analysis shows that most rides in this annual dataset last less than 20 minutes, further supporting the finding that Cyclistic is primarily used for short-duration trips.
   <br><br>

**Rider wise Trip Details:**

Observing Total number of member and casual rides and their respective average ride length

```postgresql
select member_casual,
	count(ride_id) as total_ride_count,
	AVG(ride_duration_minutes)as avg_ride_duration
from cyclisticstripdata
group by member_casual
```

Result:<br>
<img src="images/image-16.png">

<u>Key Findings:</u>

1. Members account for more rides: <br>
   Annual members recorded approximately 3.80 million rides, compared with 2.07 million rides by casual riders. Members account for about 64.8% of all rides in these two groups.
2. Casual riders spend longer on each ride: <br>
   The average casual ride lasts 18.93 minutes, whereas the average member ride lasts 12.26 minutes. This suggests that casual riders tend to use the bikes for longer journeys or activities.
3. The two groups may have different usage patterns: <br>
   The difference could be consistent with members using bikes for shorter, more routine trips and casual riders using them for longer recreational trips. However, average duration alone cannot confirm the purpose of their rides.
4. Casual riders may be an important target for membership conversion: <br>
   Cyclistic could investigate whether casual riders who take frequent or longer rides might benefit from annual membership. Before recommending conversion campaigns, ride frequency, seasonal behaviour and membership pricing must be compared to determine whether annual membership would actually offer them better value.

**Relevance to the business task:** Comparing ride patterns between members and casual riders can help the company develop targeted membership offers to convert casual riders into members. Casual riders who use the service frequently could be encouraged to purchase memberships by highlighting the potential cost savings and benefits of regular use. For
less frequent casual riders, the company could consider introducing lower-cost or flexible membership options that better suit their usage patterns and encourage them to become members.

**Bike type Detail:**

```postgresql
select member_casual, rideable_type,
	count(ride_id) as total_ride_count,
	AVG(ride_duration_minutes)as avg_ride_duration
from cyclisticstripdata
group by member_casual,rideable_type
```

Result:

<img src="images/image-17.png">

<u>Key Findings:</u>

1. Electric bikes are the most frequently used bike type.<br>
   Electric bikes account for approximately 69.8% of casual rides and 66.4% of member rides. This suggests that electric bikes are popular with both groups.

2. Casual riders take longer rides on both bike types.<br>
   - On classic bikes, casual riders average 29.02 minutes, compared with 13.91 minutes for members.
   - On electric bikes, casual riders average 14.74 minutes, compared with 11.43 minutes for members. This suggests that the difference between rider groups is not limited to one bike type.
3. Casual riders' classic-bike rides are particularly long.<br>
   Casual riders' average classic-bike ride is approximately 2.09 times as long as members' average classic-bike ride. This is the most striking difference in your results. It may indicate that casual riders use classic bikes for longer recreational journeys, but duration alone cannot confirm the reason.

4. Electric-bike rides have shorter average durations for both groups.<br>
   Casual riders average 14.74 minutes on electric bikes versus 29.02 minutes on classic bikes. Members average 11.43 minutes on electric bikes versus 13.91 minutes on classic bikes. One possible explanation is that electric bikes allow riders to cover distances more quickly or comfortably. However, you need distance data and further analysis to
   test that explanation.

5. Members account for more rides across both bike types. <br>Members record approximately 2.53 million electric-bike rides and 1.28 million classic-bike rides. Casual riders record approximately 1.46 million electric-bike rides and 608,000 classic-bike rides.

**Relevance to the business task:** Evaluating bike-type preferences among casual riders and members can help the company optimize its bike fleet and allocate the most popular bike types to stations based on demand. Maintaining the right mix of bikes at high-traffic locations can improve
the riding experience and encourage casual riders to become members. These insights can also guide targeted marketing campaigns that emphasize the benefits of different bike types and support the development of membership packages aligned with customers' riding preferences.

**Ride Patterns by Day and Hour:**

```postgresql
select member_casual, day_name, hour_24_format as ride_hour,
	count(ride_id) as total_ride_count
from cyclisticstripdata
where member_casual ='member'
group by member_casual, day_name, hour_24_format
order by total_ride_count desc

select member_casual, day_name, hour_24_format as ride_hour,
	count(ride_id) as total_ride_count
from cyclisticstripdata
where member_casual ='casual'
group by member_casual, day_name, hour_24_format
order by total_ride_count desc
```

Result:

<table>
    <tr>
    <td><img src="images/image-18.png"></td>
    <td><img src="images/image-19.png"></td>
    </tr>
</table>

Key Findings:

Cyclistic Members:

Cyclistic members demonstrate a clear weekday usage pattern, with the highest ride count recorded on Tuesday at 5 PM (75,694 rides). Weekday evening demand is particularly strong between 4 PM and 6 PM, while another noticeable peak occurs around 7 AM to 8 AM. These morning and evening peaks may reflect commuting or other routine travel activities.

Weekend usage follows a different pattern, with ride counts peaking during the afternoon. Saturday reaches its highest listed count at 3 PM (37,404 rides), while Sunday peaks at 4 PM (33,275 rides). Ride activity is lowest during the early morning hours, particularly between 1 AM and 5 AM.

Overall, the results suggest that members' usage is more concentrated around weekday morning and evening periods, whereas weekend activity is more prominent in the afternoon. These patterns can help Cyclistic plan service availability and targeted marketing, although further analysis
is needed to confirm riders' trip purposes.

Casual Riders:

Casual riders show their highest demand on Saturday afternoons, with the peak occurring at 3 PM (38,177 rides). High ride counts are also observed on Saturday and Sunday between midday and late afternoon. On weekdays, demand is particularly strong around 5 PM, especially on Friday. In contrast, ride counts are generally lower during the
early-morning hours.

These patterns suggest that casual riders' usage is concentrated in weekend afternoons, while weekday activity is more prominent in the late afternoon and early evening. This behaviour may reflect a combination of recreational and practical travel. Cyclistic could use these findings to
explore targeted marketing opportunities, particularly by comparing casual riders' usage patterns with those of annual members.

**Riding behaviour across the week:**

```postgresql
select member_casual, day_name,
	count(ride_id) as total_ride_count,
	avg(ride_duration_minutes) as avg_ride_duration_min
from cyclisticstripdata
where member_casual ='member'
group by member_casual, day_name
order by total_ride_count desc

select member_casual, day_name,
	count(ride_id) as total_ride_count,
	avg(ride_duration_minutes) as avg_ride_duration_min
from cyclisticstripdata
where member_casual ='casual'
group by member_casual, day_name
order by total_ride_count desc
```

Result:

<table>
    <tr>
    <td><img src="images/image-20.png"></td>
    <td><img src="images/image-21.png"></td>
    </tr>
</table>

<u>Key Findings:</u>

Cyclistic Members:

Annual members record the highest number of rides on Wednesday
(610,099), followed by Thursday and Tuesday, while Sunday has the lowest ride count (406,232). Approximately 76.6% of member rides occur on weekdays, suggesting that members use Cyclistic more frequently during the working week.

However, average ride duration is higher on weekends, reaching 13.53 minutes on Saturday and 13.45 minutes on Sunday, compared with approximately 11.75--12.18 minutes on weekdays. This suggests that members take more frequent but shorter rides during the week and fewer but longer rides on weekends. The pattern may reflect differences between routine weekday travel and weekend activities, although ride purpose cannot be determined from these statistics alone.

Casual Riders:

Casual riders record their highest ride volume on Saturday, with 438,723 rides, followed by Sunday with 342,482 rides. Friday also shows relatively high activity, with 327,122 rides, while Tuesday records the lowest number of rides at 229,817. This indicates that casual rider demand is strongest on weekends and increases toward Friday.

Average ride duration is also longer on weekends, reaching 22.06 minutes on Sunday and 21.54 minutes on Saturday. In comparison, Wednesday has the shortest average duration at 15.70 minutes. These findings suggest that casual riders may use Cyclistic bikes for longer recreational or personal trips on weekends, while weekday rides tend to be shorter. However, trip purpose cannot be confirmed from ride counts and duration
alone.

**Marketing and Pricing Opportunities:** Examining when casual riders and members are most active highlights the difference of service usage which can help the company schedule promotional campaigns and membership offers during periods when they are most likely to influence customer decisions. Tailoring these initiatives to different usage patterns may
encourage more casual riders to purchase memberships. The company could also evaluate time-based pricing strategies as a potential incentive for membership sign-ups, while first assessing customers' price sensitivity to ensure that such changes do not discourage ridership.

**Monthly Ride Patterns:**

```postgresql
select ride_month, member_casual, month_name,
	count(ride_id) as total_ride_count,
	avg(ride_duration_minutes) as avg_ride_duration_min
from cyclisticstripdata
where member_casual ='member'
group by ride_month, member_casual, month_name
order by ride_month

select ride_month, member_casual, month_name,
	count(ride_id) as total_ride_count,
	avg(ride_duration_minutes) as avg_ride_duration_min
from cyclisticstripdata
where member_casual ='casual'
group by ride_month, member_casual, month_name
order by ride_month
```

<table>
    <tr>
    <td><img src="images/image-22.png"></td>
    <td><img src="images/image-23.png"></td>
    </tr>
</table>

<u>Key Findings:</u>

Cyclistic Members:

Cyclistic member ride activity demonstrates a strong seasonal pattern. July records the highest number of rides, with 500,465 trips, followed by June with 444,192 and August with 443,169. In contrast, December and January have the lowest ride volumes, with approximately 109,000 rides each. This suggests that member demand is substantially higher during
the summer months and lower during winter.

Average ride duration also varies throughout the year. August has the longest average ride duration at approximately 13.08 minutes, while February has the shortest at 10.75 minutes. July has the highest ride volume but not the longest average ride duration, indicating that the busiest month does not necessarily have the longest trips.

Overall, the results suggest that Cyclistic member usage is seasonal, with stronger demand during summer and lower activity during winter. Weather conditions may contribute to this pattern, but additional weather data would be required to confirm the relationship.

Casual Riders:

Casual rider activity shows a pronounced seasonal trend. Ride volume increases from spring and reaches its highest level in July, with 345,581 rides, followed by August with 323,566 rides. Activity declines during autumn and reaches much lower levels in winter, particularly in December and January.

Average ride duration also varies seasonally. August records the longest average ride at approximately 21.71 minutes, while December has the shortest average at 12.87 minutes. May through September generally show higher average durations than the winter months.

These findings suggest that casual riders are more active and tend to take longer rides during the warmer part of the year. This pattern may be consistent with recreational or leisure use, although the purpose of individual trips cannot be established from ride counts and duration alone. Cyclistic could use these seasonal trends to plan summer promotions, maintain bike availability during peak months, and explore
membership offers aimed at converting frequent casual riders into members.

**Most Popular Start & End Stations**

Among Members:

```postgresql
select member_casual, start_station_name,
	count(ride_id) as total_ride_count
from cyclisticstripdata
where member_casual ='member'
group by member_casual, start_station_name
order by  total_ride_count desc

select member_casual, end_station_name,
	count(ride_id) as total_ride_count
from cyclisticstripdata
where member_casual ='member'
group by member_casual, end_station_name
order by  total_ride_count desc
```

<table>
    <tr>
    <td><img src="images/image-24.png"></td>
    <td><img src="images/image-25.png"></td>
    </tr>
</table>

The top 10 lists of start and end stations are almost identical, although their rankings and ride counts vary slightly. This consistency indicates that these stations are important hubs in members' travel patterns. Several are situated in or near Chicago's central business district, which contains major office buildings, government facilities,
healthcare services, and other employment centers. Their high activity may therefore partly reflect employees commuting to and from work, alongside other everyday journeys

Among Casual Riders:

```postgresql
select member_casual, start_station_name,
	count(ride_id) as total_ride_count
from cyclisticstripdata
where member_casual ='casual'
group by member_casual, start_station_name
order by  total_ride_count desc

select member_casual, end_station_name,
	count(ride_id) as total_ride_count
from cyclisticstripdata
where member_casual ='casual'
group by member_casual, end_station_name
order by  total_ride_count desc
```

<table>
    <tr>
    <td><img src="images/image-26.png"></td>
    <td><img src="images/image-27.png"></td>
    </tr>
</table>

The analysis of casual riders' start and end station usage reveals a strong concentration of rides around Chicago's lakefront, major tourist attractions, parks, and recreational areas.

Several stations, including Ritchie Ct & Banks St, Clark St & Schiller St, New St & Illinois St, and Dearborn Pkwy & Delaware Pl, also appear prominently in both casual and member riders lists. These findings suggest that casual riders may use the bike-sharing service for sightseeing, leisure, waterfront activities, and travel between popular
city destinations. However, the presence of residential and commercial locations throughout the results indicates that casual usage extends beyond tourist areas.

**Relevance to the business task:** Analyzing the most frequently used start and end stations for both casual riders and members can help the company identify high-demand locations and strategically target its marketing campaigns, advertising placements, and promotional initiatives to encourage casual riders to become annual members.

<br>

**Transfer Data to CSV:**

SQL Shell query:

```code
\copy public.cyclisticstripdata TO 'C:/DataAnalytics/cyclistic.csv' WITH (FORMAT CSV, HEADER TRUE, ENCODING 'UTF8')
```

<br><br>

## **SHARE**

In this phase, we will create visualizations and share our key findings.

<p style="vertical-align: middle;">For Cyclistic dashboard, go to &nbsp;
    <a href="https://public.tableau.com/app/profile/sajida.faraz/viz/Cyclistic_17924787487710/d_TimeAna?publish=yes"> <img src="images/Tableau.png" width="90"></a></p>

### **Key Insights:**

1. **Rider Composition:** Annual rides are predominantly generated by members, who account for 64.76% of total rides, while casual riders contribute the remaining 35.24%.

2. **Members' Riding Behavior:** Members primarily use the bike-sharing service for daily commuting, such as travelling to work or educational institutions. Their activity is highest during weekday commuting hours,
   particularly between 7 AM--8 AM and 4 PM--6 PM.

3. **Casual Riders' Riding Behavior:** Casual riders tend to use the service more for recreational activities, leisure, and sightseeing. Their usage is highest on weekends, especially between 12 PM and 6 PM. However, a noticeable increase in weekday rides is also observed between 4 PM and 6
   PM.

4. **Seasonal Trends:** Both membership groups exhibit a clear seasonal pattern in their riding behavior. Bike usage reaches its highest levels during summer and declines to its lowest during winter.

5. **Monthly Ride Patterns:** July records the highest number of rides for both members and casual riders, making it the busiest month across both user categories.

6. **Ride Duration Comparison:** Although members consistently account for a larger share of total rides than casual users, their average ride duration is comparatively shorter. Casual riders tend to spend more time on each trip.

7. **Weekday vs. Weekend Duration:** Both members and casual riders have longer average ride durations on weekends than on weekdays, indicating a greater tendency towards extended rides during leisure days.

8. **Bike Type Preference:** Electric bikes are the preferred choice for both rider categories, with members and casual users recording higher usage of electric bikes than classic bikes.

9. **Popular Station Locations:** The top 10 starting and ending stations are identical for both rider categories. These stations are also located relatively close to one another, suggesting shared preferences for popular routes and destinations.

## **ACT**

### **Top 3 Recommendations:**

1. **Tiered Transitional Membership Architecture:**

   To effectively capture and convert high-potential casual riders who are hesitant to commit to a full 365-day annual membership, Cyclistic should deploy a structured, tiered transitional membership model. Instead of relying on a binary choice between single-ride tickets and a rigid annual subscription, introducing intermediate plans will lower the
   psychological barrier to entry, cultivate regular riding habits, and systematically drive long-term conversion.

   **Acquisition:** A casual rider purchases
   <div style="margin-left: 40px;">
   <i>a Weekend Plus Pass</i> (weekend paired with a fixed allowance of weekday rides) OR <br>
   <i>a 10-Ride Pack</i> (with extended validity) OR<br>
   <i>a semi-annual unlimited access pass</i> (covering peak riding months)
   </div>
   without needing an annual membership. This lowers the barrier to entry and captures revenue from people who think annual memberships are too expensive.

   **Engagement:** Because they have the pass, they ride more
   frequently -- shifting from weekend-only users to regular weekly riders.

   **Conversion:** Utilize in-app analytics to track that a Flex Pass holder has used their rides heavily or saved a certain amount of money, the app triggers a targeted conversion offer: "You saved \$XX on your Weekend Plus Pass this month! For just \$X more, upgrade to an Annual
   Membership and ride unlimited all year."

2. **The Loyalty / Retention Play (Member-Only Perks):**

   Reserve exclusive perks for current annual members (e.g., free guest passes, priority bike reservations during rush hour, or discounts on partner merchandise). This rewards your core user base for staying subscribed, but keeps the introductory flex passes open to casual riders
   to drive top-of-funnel conversion.

3. **Targeted location-based marketing:**

   Focus promotional campaigns, advertisements, and membership offers around popular starting and ending stations to reach more casual riders and encourage them to become annual members.
