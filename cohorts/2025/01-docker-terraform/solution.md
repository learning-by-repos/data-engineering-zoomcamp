# 1
```shell
docker run --rm python:3.12.8 pip --version
# pip 24.3.1 from /usr/local/lib/python3.12/site-packages/pip (python 3.12)
```

# 2 
`db:5433`

# 3
ingest data 
```shell
cd cohorts/2025/01-docker-terraform/
pip install pgcli mycli sqlalchemy psycopg2 pandas 
#cp ../../../01-docker-terraform/2_docker_sql/ingest_data.py .
#vim ingest_data.py
docker compose up -d
python ingest_data.py \
  --user=postgres \
  --password=postgres \
  --host=localhost \
  --port=5433 \
  --db=ny_taxi \
  --table_name=green_taxi_trips \
  --url=https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-10.csv.gz

python ingest_data.py \
  --user=postgres \
  --password=postgres \
  --host=localhost \
  --port=5433 \
  --db=ny_taxi \
  --table_name=taxi_zone_lookup \
  --url=https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv

```
run query

```sql
SELECT
     SUM(CASE WHEN trip_distance <= 1 THEN 1 ELSE 0 END) AS "Up to 1 mile",
     SUM(CASE WHEN trip_distance > 1 AND trip_distance <= 3 THEN 1 ELSE 0 END) AS "1 to 3 miles",
     SUM(CASE WHEN trip_distance > 3 AND trip_distance <= 7 THEN 1 ELSE 0 END) AS "3 to 7 miles",
     SUM(CASE WHEN trip_distance > 7 AND trip_distance <= 10 THEN 1 ELSE 0 END) AS "7 to 10 miles",
     SUM(CASE WHEN trip_distance > 10 THEN 1 ELSE 0 END) AS "Over 10 miles"
 FROM
     green_taxi_trips
 WHERE
     1=1
     AND lpep_pickup_datetime >= '2019-10-01' 
     AND lpep_pickup_datetime < '2019-11-01'
     AND trip_distance IS NOT NULL  -- Ensure valid trip distances;
```

result
```
+--------------+--------------+--------------+---------------+---------------+
| Up to 1 mile | 1 to 3 miles | 3 to 7 miles | 7 to 10 miles | Over 10 miles |
|--------------+--------------+--------------+---------------+---------------|
| 104830       | 198995       | 109642       | 27686         | 35201         |
+--------------+--------------+--------------+---------------+---------------+
```
**104,838; 199,013; 109,645; 27,688; 35,202**

# 4 

```sql
SELECT 
     DATE(lpep_pickup_datetime) AS trip_date,
     MAX(trip_distance) AS longest_trip_distance
 FROM 
     green_taxi_trips
 GROUP BY 
     DATE(lpep_pickup_datetime)
 ORDER BY 
     longest_trip_distance DESC
 LIMIT 1;
 ```

 result
 ```
+------------+-----------------------+
| trip_date  | longest_trip_distance |
|------------+-----------------------|
| 2019-10-31 | 515.89                |
+------------+-----------------------+
```
**2019-10-31**

# 5 
```sql

SELECT 
    taxi_zone_lookup."Zone" AS pickup_location, 
    SUM(green_taxi_trips.total_amount) AS total_amount
FROM green_taxi_trips
JOIN 
 taxi_zone_lookup ON green_taxi_trips."PULocationID" = taxi_zone_lookup."LocationID" 
WHERE 
    DATE(green_taxi_trips.lpep_pickup_datetime) = '2019-10-18'
    AND green_taxi_trips.total_amount IS NOT NULL  -- Ensure valid total_amount
GROUP BY green_taxi_trips."PULocationID", taxi_zone_lookup."Zone"
HAVING SUM(green_taxi_trips.total_amount) > 13000
ORDER BY total_amount DESC;
```
```
+---------------------+--------------------+
| pickup_location     | total_amount       |
|---------------------+--------------------|
| East Harlem North   | 18686.680000000073 |
| East Harlem South   | 16797.260000000075 |
| Morningside Heights | 13029.790000000028 |
+---------------------+--------------------+
```
**East Harlem North, East Harlem South, Morningside Heights**

# 6
```sql
SELECT 
    t2."Zone" AS dropoff_zone_name,
    MAX(trips.tip_amount) AS largest_tip
FROM green_taxi_trips trips
JOIN taxi_zone_lookup t1 ON trips."PULocationID" = t1."LocationID" 
JOIN taxi_zone_lookup t2 ON trips."DOLocationID" = t2."LocationID"  -- Drop-off location join
WHERE 
    trips.tip_amount IS NOT NULL
    AND t1."Zone" = 'East Harlem North'
    AND DATE(trips.lpep_pickup_datetime) BETWEEN '2019-10-01' AND '2019-11-01'
    
GROUP BY t2."Zone"
ORDER BY largest_tip DESC
LIMIT 1;
```

```
+-------------------+-------------+
| dropoff_zone_name | largest_tip |
|-------------------+-------------|
| JFK Airport       | 87.3        |
+-------------------+-------------+
```
**JFK Airport**

# 7

**terraform init, terraform apply -auto-approve, terraform destroy**