
# Data Engineering Zoomcamp 2025
## Homework Solutions for Module 3: Data Warehousing

The data used for this homework was loaded to a GCS bucket. For this, a GCS Service Account was created with a Storage Admin role. The parquet files containing the yellow taxi trip records from january to june where downloaded from: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page and loaded into the bucket.
The following query creates an external table named "external_yellow_tripdata" from the parquet files contained inside the bucket for a dataset named nytaxi.

```SQL
CREATE OR REPLACE EXTERNAL TABLE `PROJECT_ID.nytaxi.external_yellow_tripdata`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://nytaxiyellow/*.parquet']
);
```


### Q1. What is count of records for the 2024 Yellow Taxi Data?
- Run the following query to get the total count fo records for the 2024 Yellow Taxi Data:


```SQL
SELECT COUNT(1) FROM `nytaxi.external_yellow_tripdata`
```
<b>Answer</b>: 20,332,093


### Q2. Write a query to count the distinct number of PULocationIDs for the entire dataset on both the tables. What is the estimated amount of data that will be read when this query is executed on the External Table and the Table?

- To answer this question we need to create a non-external (or materialized) table using the following query:

```SQL
CREATE OR REPLACE TABLE `PROJECT_ID.nytaxi.yellow_tripdata_non_partinioned` AS SELECT * FROM `PROJECT_ID.nytaxi.external_yellow_tripdata`
```

- Once the table is created we can check the estimated amount of data that will be read when we query a count of the distinct number of PULocationIDs for each table. This can be done by typing the following queries <b>without executing</b>:

```SQL
SELECT 
        COUNT(DISTINCT(PULocationID)) 
        FROM `nytaxi.external_yellow_tripdata`;

SELECT 
        COUNT(DISTINCT(PULocationID))
        FROM `nytaxi.yellow_tripdata_non_partinioned`;
```

- Then we can highlight each individual SELECT query and look at the right hand side to consult the estimated data to be processed.

<b>Answer</b>: 0 MB for the External Table and 155.12 MB for the materialized Table.


### Q3. Write a query to retrieve the PULocationID from the table (not the external table) in BigQuery. Now write a query to retrieve the PULocationID and DOLocationID on the same table. Why are the estimated number of Bytes different?

- Write the following queries, then consult the estimated data to be processed:

```SQL
SELECT 
        PULocationID
        FROM `nytaxi.yellow_tripdata_non_partinioned`;

SELECT 
         PULocationID, DOLocationID
         FROM `nytaxi.yellow_tripdata_non_partinioned`;
```

<b>Answer</b>: BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.

### Q4. How many records have a fare_amount of 0?

- Run the following query:
```SQL
SELECT 
        COUNT(1)
        FROM `nytaxi.yellow_tripdata_non_partinioned`
        WHERE fare_amount=0;

```

<b>Answer</b>: 8,333 

### Q5. What is the best strategy to make an optimized table in Big Query if your query will always filter based on tpep_dropoff_datetime and order the results by VendorID (Create a new table with this strategy)

- Partitioning by tpep_dropoff_datetime divides the table by date/time, so that queries that filter on date ranges will only scan relevant partitions instead of the whole table, while clustering by VendorID would help optimize queries that filter or group by VendorID.
The following query creates a table partitioned by tpep_dropoff_datetime and clustered by VendorID:

```SQL
    SELECT 
    CREATE OR REPLACE TABLE `nytaxi.yellow_tripdata_partitioned_clustered`
    PARTITION BY DATE(tpep_dropoff_datetime)
    CLUSTER BY VendorID
    AS
    SELECT * FROM `nytaxi.yellow_tripdata_non_partinioned`;
```

<b>Answer</b>: Partition by tpep_dropoff_datetime and Cluster on VendorID

### Q6 Write a query to retrieve the distinct VendorIDs between tpep_dropoff_datetime 2024-03-01 and 2024-03-15 (inclusive)

Use the materialized table you created earlier in your from clause and note the estimated bytes. Now change the table in the from clause to the partitioned table you created for question 5 and note the estimated bytes processed. What are these values?

Choose the answer which most closely matches.

- The data processing sizes where compared for the following queries:

```SQL
SELECT distinct(VendorID) FROM `nytaxi.yellow_tripdata_non_partinioned`
WHERE tpep_dropoff_datetime >= "2024-03-01" AND tpep_dropoff_datetime <= "2024-03-15";

SELECT distinct(VendorID) FROM `nytaxi.yellow_tripdata_partitioned_clustered`
WHERE tpep_dropoff_datetime >= "2024-03-01" AND tpep_dropoff_datetime <= "2024-03-15";
```

<b>Answer</b>: 310.24 MB for non-partitioned table and 26.84 MB for the partitioned table

### Q7. Where is the data stored in the External Table you created?
- The data remains in its original location when creating an external table, in this case being the GCP Bucket.

<b>Answer</b>: GCP Bucket

### Q8. It is best practice in Big Query to always cluster your data:
- Clustering is not always beneficial when working with small tables (data size < 1GB), when columns have very few unique values, or queries dont filter on clustered columns. This could lead to increased storage costs through metadata overhead or slower insert performance, since bigquery would have to reorder and sort rows based on the clustering column.

<b>Answer</b>: False