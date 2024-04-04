Initialized by Azure Synapse Workspace!

# NYC-Taxi-Data Analysis Using Azure Synapse Analytics 

## Overview
Implemented a real-world project using Azure Synapse Analytics, utilizing NYC Taxi Trips data for practical learning, including creating SQL scripts and Spark notebooks. 
Demonstrated proficiency in setting up dedicated SQL pools and Spark pools, enabling Synapse Link and analytic store in Cosmos DB. 
Ingested and transformed data using Serverless SQL Pool and Spark Pool, and loaded data into dedicated SQL Pool. 
Presented data to Power BI from Serverless SQL Pool and Dedicated SQL Pool. 
Executed scripts and notebooks efficiently using Synapse Pipelines and Triggers. 
Conducted operational reporting from Cosmos DB data using Azure Synapse Analytics, and built insightful reports in Power BI tailored to the data stored in Azure Synapse Analytics.

## Data Overview 

The dataset that we are going to use for our project is the trip data from New York City Taxi Services.
There are three different taxi-hailing services in New York.
The first one is yellow taxis, which are only allowed to pick up passengers from the inner city.
The next one is green taxis. They, on the other hand, are allowed to only pick up passengers from the outer boroughs, but they can drop them off anywhere in the city.
Then we have the farm vehicles which operate throughout the city.
There is a further classification of the For Hire Vehicles called High Volume For-Hire Vehicles. <br> <br>

### Data Files Overview 
There are a lot of flies as shown below : 

1. Taxi Zone (CSV) <br>
2. Calendar (CSV)  <br>
3. Trip Type (TSV)  <br>
4. Rate Code (JSON)  <br> 
5. Payment Type (JSON)  <br>
6. Vendor (CSV Quoted )  <br>
7. Trip Data (Parquet,CSV,Delta)  <br>

![image](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/Data_Overview.png)


## Architecture
- Using Serverless SQL For Data Discovery
- Made Data Virtualization by using External Data Sources and External Files Format For more Organization and make ETL less Complex
- Made a Data Transformation to create USP , CETAS, and View to access Data, Remove unwanted columns, and Store pre-aggregated data <br>

### Solution Architecture
![Solution Architecture](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/solution-architecture.png)

### Dedicated SQL Pool Architecture
![Dedicated-SQL-pool-architecture](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/Dedicated-SQL-pool-architecture.png)

### SQL Server Pool Architecture
![SQL-server-pool-architecture](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/SQL-server-pool-architecture.png)

### Spark Server Pool Architecture
![Spark-server-pool-architecture](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/Spark-server-pool-architecture.png)


## Getting Started
1. Clone the repository: 
    ```bash
    git clone https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics.git
     ```
2. Set up Azure Synapse Analytics instance and necessary resources in your Azure subscription.
3. Configure Azure Synapse Analytics according to your data sources and desired analysis.
4. Visualize the analyzed data using Power BI or other preferred visualization tools.

## Project Requirements
### 1- Data Discovery 

<ol>
<li> Identify duplicates in data </li>
<li> Check for missing data values </li>
<li> Invalid/ Unexpected data in columns </li>
<li> Join data from multiple files </li>
<li> Summarize/ Aggregate data </li>
<li> Apply some transforms </li>
</ol>

```
The Assignment : 
Identify the percentage of cash and credit card trips by borough 
```
~~~~sql

WITH cte_payment_type AS
(
    SELECT 
        CAST(JSON_VALUE(jsonDoc,'$.payment_type') AS SMALLINT) AS payment_type,
        CAST(JSON_VALUE(jsonDoc,'$.payment_type_desc') AS VARCHAR(15)) AS payment_type_desc
    FROM
        OPENROWSET(
            BULK 'payment_type.json',
            DATA_SOURCE = 'nyc_taxi_data_raw',
            FORMAT = 'CSV',
            PARSER_VERSION = '1.0',
            FIELDQUOTE = '0x0b',
            FIELDTERMINATOR ='0x0b',
            ROWTERMINATOR = '0x0a'
        )
        WITH 
        (
            jsonDoc NVARCHAR(MAX)
        ) AS [result]
),
cte_taxi_zone AS
(
    SELECT * 
    FROM 
        OPENROWSET(
            BULK 'abfss://nyc-taxi-data@synapsecoursedl1968.dfs.core.windows.net/raw/taxi_zone.csv',
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0',
            FIRSTROW = 2
        ) 
        WITH (
            location_id SMALLINT,
            borough VARCHAR(15),
            zone VARCHAR(50),
            service_zone VARCHAR(15)
        ) AS [result]
),
cte_trip_data AS
(
    SELECT *
    FROM
        OPENROWSET(
            BULK 'trip_data_green_parquet/**',
            DATA_SOURCE = 'nyc_taxi_data_raw',
            FORMAT = 'PARQUET'
        ) AS [result]
)
SELECT 
    borough, 
    COUNT(1) total_trips,
    SUM(CASE WHEN payment_type_desc = 'Cash' THEN 1 ELSE 0 END) cash_trips,
    SUM(CASE WHEN payment_type_desc = 'Credit card' THEN 1 ELSE 0 END) card_trips,
    CAST((SUM(CASE WHEN payment_type_desc = 'Cash' THEN 1 ELSE 0 END)/ CAST(COUNT(1) AS DECIMAL)) * 100 AS DECIMAL(5,2)) AS cash_trips_perc,
    CAST((SUM(CASE WHEN payment_type_desc = 'Credit card' THEN 1 ELSE 0 END)/ CAST(COUNT(1) AS DECIMAL))* 100 AS DECIMAL(5,2)) AS card_trips_perc
FROM cte_trip_data
JOIN cte_payment_type
ON cte_trip_data.payment_type=cte_payment_type.payment_type
JOIN cte_taxi_zone
ON cte_trip_data.PULocationId = cte_taxi_zone.location_id
WHERE cte_trip_data.payment_type IN (1,2)
GROUP BY borough
ORDER BY borough

~~~~
Output : 

![image](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/sample_data.png)


### 2- Data Virtualization  

Data virtualization is a logical data layer that allows us to combine data from multiple <br>
sources at query time without having to write complex ETL pipelines to load the data. <br>



1. Create an External Data Source
~~~~sql
IF NOT EXISTS (SELECT * FROM sys.external_data_sources WHERE name = 'nyc_taxi_src')
    CREATE EXTERNAL DATA SOURCE nyc_taxi_src
    WITH
    (    LOCATION         = 'Path'
    );
~~~~
2. Create an External File Format 

~~~~sql
-- Create file format csv_file_format for parser version 2.0
IF NOT EXISTS (SELECT * FROM sys.external_file_formats WHERE name ='csv_file_format')
  CREATE EXTERNAL FILE FORMAT csv_file_format  
  WITH (  
      FORMAT_TYPE = DELIMITEDTEXT,
      FORMAT_OPTIONS (  
        FIELD_TERMINATOR = ','  
      , STRING_DELIMITER = '"'
      , First_Row = 2
      , USE_TYPE_DEFAULT = FALSE 
      , Encoding = 'UTF8'
      , PARSER_VERSION = '2.0' )   
      );  

~~~~

### 3- Data Ingestion  

```
As Show Below, we have Partitioned Files in different Locations so we need to Compain all these files and make a view 
We will use an External Tables and Stored Procedure then Create a view with partitioned columns.
```  
<br>

![image](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/transform.png)

##### a Simple Query to create a view for trip_data_green from Multiple Files and parquet files : 

~~~~sql

-- Create a view for trip_data_green
DROP VIEW IF EXISTS silver.vw_trip_data_green
GO

CREATE VIEW silver.vw_trip_data_green
AS
SELECT
    result.filepath(1) AS year,
    result.filepath(2) AS month,
    result.*
FROM
    OPENROWSET(
        BULK 'silver/trip_data_green/year=*/month=*/*.parquet',
        DATA_SOURCE = 'nyc_taxi_src',
        FORMAT = 'PARQUET'
    )
    WITH (
        vendor_id INT,
        lpep_pickup_datetime datetime2(7),
        lpep_dropoff_datetime datetime2(7),
        store_and_fwd_flag CHAR(1),
        rate_code_id INT,
        pu_location_id INT,
        do_location_id INT,
        passenger_count INT,
        trip_distance FLOAT,
        fare_amount FLOAT,
        extra FLOAT,
        mta_tax FLOAT,
        tip_amount FLOAT,
        tolls_amount FLOAT,
        ehail_fee INT,
        improvement_surcharge FLOAT,
        total_amount FLOAT,
        payment_type INT,
        trip_type INT,
        congestion_surcharge FLOAT
  ) AS [result]
GO

SELECT TOP(100) *
  FROM silver.vw_trip_data_green
GO
~~~~

### 3- Data Transformation

<ol>
<li> Join the key information required for reporting to create a new table. </li>
<li> Join the key information required for Analysis to create a new table.</li>
<li> Ability to query the ingested data using SQL</li>
<li> Must be able to analyze the transformed data via T-SQL </li>
<li> Transformed data must be stored in columnar format (i.e., Parquet) </li>
</ol>

### Business Requirements (1) :

Campaign to encourage credit card payments : 
1. trips made using credit card/ cash payments
2. Payment behavior during days of the week/ weekend
3. Payment behavior between boroughs


Solution : 
~~~~sql
SELECT
    td.year, 
    td.month,
    tz.borough,
    CONVERT(DATE,td.lpep_pickup_datetime) trip_date,
    c.day_name trip_day,
    CASE WHEN c.day_name IN ('Saturday','Sunday') THEN 'Y' ELSE 'N' END AS trip_day_weekend_ind,
    SUM(CASE WHEN pt.description = 'Credit card' THEN 1 ELSE 0 END) AS card_trip_count,
    SUM(CASE WHEN pt.description = 'Cash' THEN 1 ELSE 0 END) AS cash_trip_count
FROM silver.vw_trip_data_green td
JOIN silver.taxi_zone tz
    ON td.pu_location_id = tz.location_id
JOIN silver.calendar c
    ON c.date = CONVERT(DATE,td.lpep_pickup_datetime)
JOIN silver.payment_type pt
    ON td.payment_type = pt.payment_type
WHERE td.year = '2020' AND td.month = '01'
GROUP BY td.year, td.month,tz.borough, CONVERT(DATE,td.lpep_pickup_datetime), c.day_name
~~~~

### Business Requirements (2) 
<ol>
<li> Identify taxi demand  </li>
<li> Demand based on the borough </li>
<li> Demand based on day of the week/ weekend </li>
<li> Demand based on trip type (i.e., Street hail/ Despatch) </li>
<li> Trip distance, trip duration, total fare amount, etc per day/ borough </li>


</ol>

Solution : 
~~~~sql
SELECT 
    td.year, 
    td.month,
    tz.borough,
    CONVERT(DATE,td.lpep_pickup_datetime) trip_date,
    c.day_name trip_day,
    CASE WHEN c.day_name IN (''Saturday'',''Sunday'') THEN ''Y'' ELSE ''N'' END AS trip_day_weekend_ind,
    SUM(CASE WHEN pt.description = ''Credit card'' THEN 1 ELSE 0 END) AS card_trip_count,
    SUM(CASE WHEN pt.description = ''Cash'' THEN 1 ELSE 0 END) AS cash_trip_count,
    SUM(CASE WHEN tt.trip_type_desc = ''Street-hail'' THEN 1 ELSE 0 END) AS street_hail_trip_count,
    SUM(CASE WHEN tt.trip_type_desc = ''Dispatch'' THEN 1 ELSE 0 END) AS dispatch_trip_count,
    SUM(td.trip_distance) trip_distance,
    SUM(DATEDIFF(MINUTE, td.lpep_pickup_datetime, td.lpep_dropoff_datetime)) as trip_duration,
    SUM(td.fare_amount) fare_amount
FROM silver.vw_trip_data_green td
JOIN silver.taxi_zone tz
    ON td.pu_location_id = tz.location_id
JOIN silver.calendar c
    ON c.date = CONVERT(DATE,td.lpep_pickup_datetime)
JOIN silver.payment_type pt
    ON td.payment_type = pt.payment_type
JOIN silver.trip_type tt
ON td.trip_type = tt.trip_type
WHERE td.year = ''' + @year + '''
    AND td.month = ''' + @month + '''
GROUP BY td.year, td.month, tz.borough, CONVERT(DATE, td.lpep_pickup_datetime), c.day_name'
~~~~

### 4- Reporting Requirements

<ol>
<li> Taxi Demand </li>
<li> Join the key information required for Analysis to create a new table.</li>
<li> Credit Card Campaign</li>
<li> Must be able to analyze the transformed data via T-SQL </li>
<li> Operational Reporting</li>

</ol>


### Sample :
![image](https://github.com/ahmedashraffcih/NYC-Taxi-Data-Analysis-using-Synapse-Analytics/blob/main/imgs/sample.png)

 
## Contributing
Contributions to enhance and expand the capabilities of this project are welcome! Please follow these guidelines:

- Fork the repository.
- Create a new branch for your feature or enhancement.
- Commit your changes with descriptive messages.
- Submit a pull request for review.

## Acknowledgements
- Special thanks to the contributors of Azure Synapse Analytics and related Azure services.
- Credits to the providers of NYC-Taxi-Trips datasets and resources used for analysis.

## Contact
For any inquiries or feedback, feel free to contact the project maintainer:

Email - ahmedashraffcih@gmail.com <br>
LinkedIn - [ahmedashraffcih](https://www.linkedin.com/in/ahmedashraffcih/)

Feel free to customize and expand upon this README to better suit the specifics of your project.