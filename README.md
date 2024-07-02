# Iceberg Lab

## Summary
This workshop will take you through the new capabilities that have been added to CDP Public Cloud Lakehouse. 

In this workshop you will learn how to take advantage of Iceberg to support Data Lakehouse initiatives.

**Value Propositions**:  

Take advantage of Iceberg - **CDP Open Data Lakehouse**, to experience:  
- Better performance  
- Lower maintenance responsibilities  
- Multi-function analytics without having many copies of data  
- Greater control  

It will also give on overview of **Cloudera Data Flow** to give hands on experience of:  
- Real-time data streaming  
- Out of the box connection to various data sources and integration with Iceberg  
- Easy deployment of data pipelines using no code and accelerators  


*Note to admins: Refer to the Setup file containing the recommendations to setup the lab*


## TABLE OF CONTENT
  * [1. Introduction to the workshop](#1-introduction-to-the-workshop)  
    * [1.1. Logging in](##11-logging-in)  
    * [1.2. Data Set](##11-data-set)   
    * [1.3. Access the data set in Cloudera Data Warehouse](##13-access-the-data-set-in-cloudera-data-warehouse)  
    * [1.4. Generating the Iceberg tables](##14-generating-the-iceberg-tables)   
  * [2. Table Maintenance in Iceberg](#2-table-maintenance-in-Iceberg)  
    * [2.1. Loading data](##21-loading-data)  
    * [2.2. Partition evolution](22-partition-evolution)  
    * [2.3. Snapshots](##23-snapshots)
    * [2.4. Acid-V2](##24-Acid-V2)



### 1. Introduction to the workshop  
**Goal of the section:   
Check the dataset made available in a database in a csv format and store it all as Iceberg.** 

#### 1.1. Logging in

Access the url indicated by the presenter and indicate your credentials.

_Make a note of your username, in a CDP Public Cloud workshop, it should be a account from user01 to user50, assigned by your workshop presenter. It will be useful during the lab._

#### 1.2. Data Set description

Data set for this workshop is the publicly available Airlines data set, which consists of c.80million row of flight information across the United States.  
For additional information : [Data Set Description](./documentation/IcebergLab-Documentation.md#data-set)

#### 1.3. Access the data set in Cloudera Data Warehouse
In this section, we will check that the airlines data was ingested for you: you should be able to query the master database: `airlines_csv`.   

Each participant will then create their own Iceberg databases out of the shared master database.


Navigate to Data Warehouse service:
  
![Home_CDW](./images/home_cdw.png)

Then choose an **Impala** Virtual Warehouse and open the SQL Authoring tool HUE. There are two types of virtual warehouses you can create in CDW, here we'll be using the type that leverages **Impala** as an engine:  

![Typesofvirtualwarehouses.png](./images/ImpalaVW.png)  


Execute the following in **HUE** Impala Editor to test that data has loaded correctly and that you have the appropriate access.   
  
```SQL
SELECT COUNT(*) FROM airlines_csv.flights_csv; 
```

**Expected output**  
![Hue Editor](./images/Hue_editor.png)


  
#### 1.4. Generating the Iceberg tables

In this section, we will generate the Iceberg database from the pre-ingested csv tables.   

**Run the below queries to create your own databases and ingest data from the master database**  
  

```SQL
-- CREATE DATABASES
-- EACH USER RUNS TO CREATE DATABASES
CREATE DATABASE ${user_id}_airlines;
CREATE DATABASE ${user_id}_airlines_maint;
```

*Note: These queries use variables in Hue

To set the variable value with your username, fill in the field as below:  

![Setqueryvaribale](./images/Set_variable_hue.png)  


To run several queries in a row in Hue, make sure you select all the queries:*  

![Hue_runqueries.png](./images/Hue_runqueries.png)  



  
**Once the database is created, create the Hive tables first**  



```SQL
-- CREATE HIVE TABLE FORMAT TO CONVERT TO ICEBERG LATER
drop table if exists ${user_id}_airlines.planes;

CREATE TABLE ${user_id}_airlines.planes (
  tailnum STRING, owner_type STRING, manufacturer STRING, issue_date STRING,
  model STRING, status STRING, aircraft_type STRING,  engine_type STRING, year INT 
) 
TBLPROPERTIES ( 'transactional'='false' )
;

INSERT INTO ${user_id}_airlines.planes
  SELECT * FROM airlines_csv.planes_csv;

-- HIVE TABLE FORMAT TO USE CTAS TO CONVERT TO ICEBERG
drop table if exists ${user_id}_airlines.airports_hive;

CREATE TABLE ${user_id}_airlines.airports_hive
   AS SELECT * FROM airlines_csv.airports_csv;

-- HIVE TABLE FORMAT
drop table if exists ${user_id}_airlines.unique_tickets;

CREATE TABLE ${user_id}_airlines.unique_tickets (
  ticketnumber BIGINT, leg1flightnum BIGINT, leg1uniquecarrier STRING,
  leg1origin STRING,   leg1dest STRING, leg1month BIGINT,
  leg1dayofmonth BIGINT, leg1dayofweek BIGINT, leg1deptime BIGINT,
  leg1arrtime BIGINT, leg2flightnum BIGINT, leg2uniquecarrier STRING,
  leg2origin STRING, leg2dest STRING, leg2month BIGINT, leg2dayofmonth BIGINT,
  leg2dayofweek BIGINT, leg2deptime BIGINT, leg2arrtime BIGINT 
);

INSERT INTO ${user_id}_airlines.unique_tickets
  SELECT * FROM airlines_csv.unique_tickets_csv;
  
```
Copy & paste the SQL below into HUE

```SQL
-- TEST PLANES PROPERTIES
DESCRIBE FORMATTED ${user_id}_airlines.planes;
```  

Pay attention to the following properties: 
- Table Type: `EXTERNAL`
- SerDe Library: `org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe`
- Location: `warehouse/tablespace/external/hive/`

```SQL
-- CREATE ICEBERG TABLE FORMAT STORED AS PARQUET
drop table if exists ${user_id}_airlines.planes_iceberg;

CREATE TABLE ${user_id}_airlines.planes_iceberg
   STORED AS ICEBERG AS
   SELECT * FROM airlines_csv.planes_csv;

-- CREATE ICEBERG TABLE FORMAT STORED AS PARQUET
drop table if exists ${user_id}_airlines.flights_iceberg;

CREATE TABLE ${user_id}_airlines.flights_iceberg (
 month int, dayofmonth int, 
 dayofweek int, deptime int, crsdeptime int, arrtime int, 
 crsarrtime int, uniquecarrier string, flightnum int, tailnum string, 
 actualelapsedtime int, crselapsedtime int, airtime int, arrdelay int, 
 depdelay int, origin string, dest string, distance int, taxiin int, 
 taxiout int, cancelled int, cancellationcode string, diverted string, 
 carrierdelay int, weatherdelay int, nasdelay int, securitydelay int, 
 lateaircraftdelay int
) 
PARTITIONED BY (year int)
STORED AS ICEBERG 
;
```


Copy & paste the SQL below into HUE

```SQL
-- TEST PLANES PROPERTIES
DESCRIBE FORMATTED ${user_id}_airlines.planes_iceberg;
```

Pay attention to the following properties for this Iceberg table, compared to the same table in Hive format:
- Table Type: `	EXTERNAL_TABLE`
- SerDe Library: `org.apache.iceberg.mr.hive.HiveIcebergSerDe`
- Location: `/warehouse/tablespace/external/hive/user020_airlines.db/planes_iceberg	`


You now have your own database you can run the below queries over:  

![AirlinesDB](./images/AirlinesDB.png)


### 2. Table Maintenance in Iceberg

In this section, we will load data in Iceberg format and demonstrate a few key maintenance features of Iceberg.

#### 2.1. Loading data

Under the 'maintenance' database, let's load the flight table partitioned by year.  


```SQL
-- [TABLE MAINTENANCE] CREATE FLIGHTS TABLE IN ICEBERG TABLE FORMAT
drop table if exists ${user_id}_airlines_maint.flights;

CREATE TABLE ${user_id}_airlines_maint.flights (
 month int, dayofmonth int, 
 dayofweek int, deptime int, crsdeptime int, arrtime int, 
 crsarrtime int, uniquecarrier string, flightnum int, tailnum string, 
 actualelapsedtime int, crselapsedtime int, airtime int, arrdelay int, 
 depdelay int, origin string, dest string, distance int, taxiin int, 
 taxiout int, cancelled int, cancellationcode string, diverted string, 
 carrierdelay int, weatherdelay int, nasdelay int, securitydelay int, 
 lateaircraftdelay int
) 
PARTITIONED BY (year int)
STORED AS ICEBERG 
;
```

**Partition evolution**: the insert queries below are designed to demonstrate partition evolution
and snapshot feature for time travel 


```SQL
-- LOAD DATA TO SIMULATE SMALL FILES
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 1;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 2;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 3;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 4;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 5;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 6;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 7;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 8;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 9;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 10;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 11;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 12;

INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 1;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 2;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 3;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 4;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 5;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 6;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 7;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 8;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 9;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 10;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 11;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 12;
```

#### 2.2. Partition evolution  
  
  
Let's look at the file size. 

**In Impala**  
  
```SQL
SHOW FILES in ${user_id}_airlines_maint.flights;
```  

**For reference only**, you can check out other ways to run that command: [pyspark](documentation/IcebergLab-Documentation.md#pyspark)

Make a note of the average file size which should be around 5MB.
Also note the path and folder structure: a folder is a partition, a file is an ingest as we performed them above.

Now, let's alter the table, adding a partition on the month on top of the year.  

```SQL
ALTER TABLE ${user_id}_airlines_maint.flights SET PARTITION SPEC (year, month);
```  
  
Check the partition fields in the table properties
```SQL
DESCRIBE EXTENDED  ${user_id}_airlines_maint.flights
```

![Partitionkeys](./images/Partitionkeys.png)  


Ingest a month worth of data.  
  
```SQL
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv
 WHERE year <= 1996 AND month <= 1
```  

Let's have another look:   
  
```SQL
SHOW FILES in ${user_id}_airlines_maint.flights;
```  

Will show the newly ingested data, note the path, folder breakdown is different from before, with the additional partitioning over month taking place.


#### 2.3. Snapshots

From the INGEST queries earlier, snapshots was created and allow the time travel feature in Iceberg 
and this section will demonstrate the kind of operation you can run using that feature.

```SQL
DESCRIBE HISTORY ${user_id}_airlines_maint.flights;  
```  

Make a note of the timestamps for 2 different snapshots, as well as the snapshot id for one, you can then run:

```SQL
DESCRIBE HISTORY ${user_id}_airlines_maint.flights  BETWEEN '${Timestamp1}' AND '${Timestamp2}';
```
Timestamp format looks like this:  
`2024-04-11 09:48:07.654000000`  

You can run queries on the content of a snapshot refering its SnapshotID or timestamp:    
  
```SQL
SELECT COUNT(*) FROM ${user_id}_airlines_maint.flights FOR SYSTEM_VERSION AS OF ${snapshotid}
```
 or for example:  
 
```SQL
SELECT * FROM ${user_id}_airlines_maint.flights FOR SYSTEM_VERSION AS OF ${snapshotid}

```
Snapshot id format looks like:
`3916175409400584430` **with no quotes**

#### 2.4 ACID V2

Here we'll show the commands that could be run concomitantly thanks to [ACID](./documentation/IcebergLab-Documentation.md#acid) in Iceberg
Let's update a row.  


First, let's find a row:  

```SQL
SELECT year, month, tailnum, deptime, uniquecarrier  FROM  ${user_id}_airlines_maint.flights LIMIT 1
```
Save the values for year, month, tailnum and deptime to be able to identify that row after update.
Example:

![Savearow_hue-ACID](./images/Savearow_hue-ACID.png)  

You can bring back that row with a SELECT on a few unique characteristics and we'll update the uniquecarrier value to something else.

```SQL
SELECT * FROM ${user_id}_airlines_maint.flights
WHERE year = ${year}
  AND month = ${month}
  AND tailnum = '${tailnum}'
  AND deptime = ${deptime};
```


As Iceberg table are created as V1 by default, 
you will be able to migrate the table from Iceberg V1 to V2 using the below query:
```SQL
ALTER TABLE ${user_id}_airlines_maint.flights
SET TBLPROPERTIES('format-version'= '2')
```
Then try the UPDATE:

```SQL

UPDATE ${user_id}_airlines_maint.flights
SET uniquecarrier = 'BB' 
WHERE year = ${year}
  AND month = ${month}
  AND tailnum = '${tailnum}'
  AND deptime = ${deptime};

--- Check that the update worked:
SELECT * FROM ${user_id}_airlines_maint.flights
WHERE year = ${year}
  AND month = ${month}
  AND tailnum = '${tailnum}'
  AND deptime = ${deptime};
```

CDP Public Cloud now includes support for Apache Iceberg in the following services: Impala, Hive, Flink, Sql Stream Builder (SSB), Spark 3, NiFi, and Replication Manager (BDR).
  
Cloudera the only vendor to support Iceberg in a multi-hybrid cloud environment. 
Users can develop an Iceberg application once and deploy anywhere.  


