---
title: Apache Hive and Apache Druid performance testing for MIND Foods HUB Data Lake
author: Gabriele D'Arrigo
subject: Evaluating performances of enterprise data lake solutions for MIND Foods HUB project
keywords: [Big Data, Data Lake, Performance Testing, Apache Hive, Apache Druid]
---

<img src="C:\Users\Gabriele\AppData\Roaming\Typora\typora-user-images\image-20220202212948471.png" alt="image-20220202212948471" />



<h2 style="text-align:center;">Università degli Studi di Milano</h2>

<h3 style="text-align:center;">Facoltà di Scienze e Tecnologie</h3>

<h3  style="text-align:center;">Corso di Laurea in Sicurezza dei sistemi e delle reti informatiche</h3>

<h1  style="text-align:center;">Apache Hive and Apache Druid performance testing for MIND Foods HUB Data Lake</h1>





**Relatore**: Prof. Paolo Ceravolo, <paolo.ceravolo@unimi.it>

**Tesi di**: Gabriele D'Arrigo, <gabriele.darrigo@studenti.unimi.it>

**Matricola**: 909953

**Anno Accademico**: 2021-2022

<div style="page-break-after: always; visibility: hidden;"></div>

## Table of contents

0. Abstract
1. Introduction
    1.1 Big Data and Data Lake systems
    1.2 MIND Foods HUB
    1.3 Apache Druid
    1.4 Apache JMeter
2. Performance Testing and methodology
    2.1 Provision of each solution
    2.2 Data generation
    2.3 Data ingestion
    2.4 Queries
    2.5 Performance testing using Apache JMeter
3. Test results
4. Conclusions
4. References

<div style="page-break-after: always; visibility: hidden;"></div>

## 0. Abstract

A concise summary of the thesis.

## 1. Introduction

### 1.1 Big Data and Data Lake systems

A short paragraph on Big Data and Data Lake systems: what is a Data Lake, its purpose, mention schema-on-read vs schema-on-write concepts, mention the single table pattern.

### 1.2 MIND Foods HUB

Introduce the MIND Foods HUB project and describe the current Data Lake architecture with a focus on: 

#### Hadoop

Map/Reduce and HDFS

#### Apache Hive

what is Apache Hive, its key concepts

### 1.3 Apache Druid

Introduce Apache Druid with a short explanation of its key concepts.

### 1.4 Apache JMeter

A super short introduction to Apache JMeter.

<div style="page-break-after: always; visibility: hidden;"></div>

## 2. Performance Testing and methodology

To evaluate the performance of Apache Hive and Apache Druid and to achieve a reproducible test, I followed five significant steps for each database:

1. Provision of each solution
2. Generate syntetic data
3. Ingest data with specific schema optimization
4. Prepare the test queries
5. Run the performance testing using Apache JMeter

### 2.1 Provision each solution

To provision both solutions, I used a Vmware virtual machine hosted on the SESAR Lab cluster with the following configuration:

- 12 vCPU
- 48 GiB of memory

- 512 GiB storage
- Debian GNU/Linux 11 (bullseye) operative system.

With the machine adequately provisioned, I had to replicate the MIND Foods HUB environment; 
Provisioning was achieved using a dockerized multi-node Hadoop cluster configured as close to the one running in production.
The Hadoop cluster is composed by:

- A single NameNode
- Three DataNode
- A ResourceManager
- A NodeManager
- An HistoryServer
- A Zookeeper instance to handle the cluster high availability

#### Apache Hive provisioning

Apache Hive was configured to run on the same Hadoop dockerized cluster and consists in:

- A single HiveServer
- A Metastore server
- A MySQL 8.2 instance to persists Metastore's metadata

During the test execution of queries involving a big data set, I encountered various timeouts and `java.lang.OutOfMemoryError: Java heap space` errors caused by the default Hive's heap size of 1024 MiB.
After some research and various attempts, I increased the maximum heap size of the Hive Server to 4096 MiB. 
I also configured the Hive CLI JVM heap size to 8192 MiB.

#### Apache Druid provisioning

I configured Apache Druid to run as a clustered deployment and consists of the following servers:

- A Coordinator, responsible for handling the metadata and coordination of the various processes of the cluster
- A Broker that receives and forwards queries to the data servers
- A Router that acts as an API gateway, receives queries from external clients  and routes them to the Broker
- An Historical server, that handles the data storage and runs queries on historical, segmented data
- A MiddleManager, that handles ingestion of new data into the cluster

As previously mentioned, Apache Druid can ingest batch data natively or by loading data files from a Hadoop cluster.
Since one of the requirements of MIND Foods HUB Data Lake is to take advantage of the Hadoop infrastructure already in place, I configured the Druid cluster to work with HDFS by using [druid-hdfs-storage extensions](https://druid.apache.org/docs/latest/development/extensions-core/hdfs.html);
Instead of using a local mount, Apache Druid directly used HDFS to store data segments.

### 2.2 Data generation

To test both Druid and Hive, I generated 50 million rows (approximately 15 GB) of random, synthetic test data that can be downloaded from:

At the time of this writing MIND Foods HUB project was starting to ingest data into the Apache Hive cluster, so I had a relatively small dataset that was not suited to test the performances of Big Data systems like Hive and Druid.
To solve this problem, I wrote a Node.js application to generate random synthetic data that still respect the logical and semantic constraints of MIND Foods HUB Data Lake schema and to provide a realistic dataset to work with.
The application code is hosted on SESAR Lab Github's organization: : [https://github.com/SESARLab/mfh-measurements-generator]( https://github.com/SESARLab/mfh-measurements-generator).

MIND Foods HUB data are stored in a single table, named `dl_measurements`, that follows a denormalized data model to avoid expensive join operations.
Each row of the table can have (`NULL`) values, depending on the type of measurement.
`dl_measurements` table schema is the following:

```sql
CREATE TABLE dl_measurements
(
    id                        string,
    double_value              double,
    str_value                 string,
    unit_of_measure           string,
    sensor_id                 string,
    sensor_type               string,
    sensor_desc_name          string,
    location_id               string,
    location_name             string,
    location_description      string,
    location_botanic_name     string,
    location_cultivation_name string,
    location_latitude         double,
    location_longitude        double,
    location_altitude         double,
    measure_timestamp         timestamp,
    start_timestamp           timestamp,
    end_timestamp             timestamp,
    insertion_agent           string,
    insertion_timestamp       timestamp,
    CONSTRAINT dl_measurements_pk
        PRIMARY KEY (id) DISABLE NOVALIDATE
)
```

MIND Foods HUB *sensors* are of three types:

- Measurements, that register discrete, floating-point values (for example, temperature, humidity, wind speed, and others).
This type of measurement is stored in the `double_value` column, while the measurement time is stored in the `measure_timestamp` column.

- Phase sensors, that register a range of floating-point values in a given period.
This type of measurement is stored in the `str_value` column, while the time start and end of the measure are stored respectively in `start_timestamp` and `end_timestamp` columns.

- Tag sensors, that register string-based values. 
This type of measurement is stored in the `double_value` column, while the measurement time is stored in the `measure_timestamp` column.

Each *sensor* provides measurements for a given *location*, or rather cultivation.

To randomly generate data for `dl_measurements`, I needed to mock this relation between a sensor type and its measurement and guarantee these logical constraints:

- `double_value` is only populated for float-based measurements while `str_value` is `NULL`.
`measure_timestamp` is calculated, while `start_timestamp` and `end_timestamp` are `NULL`

- For phase-based measurement `str_value` is populated, while `double_value` is `NULL`.
Both `start_timestamp` and `end_timestamp` times are calculated, while `measure_timestamp` is `NULL`

- For tag based measurement `str_value` is populated, while `double_value` is `NULL`.
`measure_timestamp` is calculated, while `start_timestamp` and `end_timestamp` are `NULL`

While measurements values were randomly generated, *sensor* related data (`sensor_id`, `sensor_type`, `sensor_desc_name`, `unit_of_measure` ) were dumped from the MIND Foods HUB Hive production cluster and randomly picked for each generated row.
With [Mockaroo](https://www.mockaroo.com/), an online service that allows generating synthetic data comprehensive of commons and scientific plant names, I produced a set of 100 locations, randomly picked for each generated row of the dataset.

Finally, to simulate a dataset of a Data Lake in operation, all rows were generated computing the `insertion_timestamp` in two years.

### 2.3 Data ingestion

Before querying data, I created tables for both databases and ingested the synthetic generated CSV data.
The first step consisted in loading the generated dataset in a temporary HDFS folder on the Hadoop Namenode server to serve as the primary source for the ingestion process.

Unlike the original Hive `dl_measurements` table that holds MIND Foods HUB data, I decided to apply some table optimizations to test each database at the top of their performance capability.
Using Hive partitions and Druid segments, each *measurement* is stored depending on its `insertion_timestamp`, allowing each platform to efficiently retrieve data based on temporal criteria.

#### Apache Hive table optimization

On Apache Hive, `dl_measurements` has the following schema:

```sql
CREATE TABLE dl_measurements
(
    id                        string,
    sensor_id                 string,
    sensor_type               string,
    sensor_desc_name          string,
    double_value              double,
    str_value                 string,
    unit_of_measure           string,
    location_id               string,
    location_name             string,
    location_description      string,
    location_botanic_name     string,
    location_cultivation_name string,
    location_latitude         double,
    location_longitude        double,
    location_altitude         double,
    insertion_agent           string,
    measure_timestamp         timestamp,
    start_timestamp           timestamp,
    end_timestamp             timestamp,
    insertion_timestamp       timestamp,
    CONSTRAINT dl_measurements_pk
        PRIMARY KEY (id) DISABLE NOVALIDATE
)
    PARTITIONED BY (insertion_date string)
    STORED AS TEXTFILE
    LOCATION 'hdfs://namenode:9000/user/hive/warehouse/mfh.db/dl_measurements'
    TBLPROPERTIES ('bucketing_version' = '2');
```

As we can see, it defines an `insertion_date` partition, which determines how to store data into the table; *measurements* with the same `insertion_date`, are held together into the same partition, allowing Hive to efficiently retrieve data that satisfy specified criteria based on the `insertion_date`.

#### Apache Hive ingestion

To ingest data into Hive, I first loaded the CSV data from the temporary HDFS folder on the NameNode (i.e.: `/tmp/mfh`) into an external table:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS dl_measurements_external
(
    id                        string,
    sensor_id                 string,
    sensor_type               string,
    sensor_desc_name          string,
    double_value              double,
    str_value                 string,
    unit_of_measure           string,
    location_id               string,
    location_name             string,
    location_description      string,
    location_botanic_name     string,
    location_cultivation_name string,
    location_latitude         double,
    location_longitude        double,
    location_altitude         double,
    insertion_agent           string,
    measure_timestamp         timestamp,
    start_timestamp           timestamp,
    end_timestamp             timestamp,
    insertion_timestamp       timestamp,
    CONSTRAINT dl_measurements_external_pk
        PRIMARY KEY (id) DISABLE NOVALIDATE
)
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS TEXTFILE
    LOCATION 'hdfs://namenode:9000/tmp/mfh'
    TBLPROPERTIES ("skip.header.line.count"="1");
```

Then I loaded the data from the external table into `dl_measurements`, computing the partition_date from  with the following SQL statement:

```sql
FROM dl_measurements_external
INSERT OVERWRITE TABLE dl_measurements PARTITION(insertion_date)
SELECT *, date_format(insertion_timestamp, 'YYYY-MM-dd') AS insertion_date
DISTRIBUTE BY insertion_date;
```

#### Apache Druid datasource optimization

On Apache Druid, data was ingested using *insertion_timestamp* as the primary datasource timestamp, with a *month* granularity;
*measurements* registered within the same month are stored into the same *time chunks* (with a *time chunks* containing one or more segments).

#### Apache Druid ingestion

Druid ingestion is configured by submitting an *[ingestion task](https://druid.apache.org/docs/latest/ingestion/ingestion-spec.html)* spec to the Druid Coordinator; the overall process was more accessible since I can achieve it by using Druid's [web console](https://druid.apache.org/docs/latest/operations/druid-console.html).
Data was loaded from the temporary HDFS folder on the NameNode using the following spec:

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "hdfs",
        "paths": "hdfs://namenode:9000/tmp/mfh/dl-measurements_1642410970616.csv"
      },
      "inputFormat": {
        "type": "csv",
        "findColumnsFromHeader": true
      }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "dynamic"
      },
      "logParseExceptions": true
    },
    "dataSchema": {
      "timestampSpec": {
        "column": "insertion_timestamp",
        "format": "auto"
      },
      "dimensionsSpec": {
        "dimensions": [
          "id",
          {
            "type": "double",
            "name": "double_value"
          },
          "str_value",
          "unit_of_measure",
          "sensor_id",
          "sensor_type",
          "sensor_desc_name",
          "location_id",
          "location_name",
          "location_botanic_name",
          "location_cultivation_name",
          "location_description",
          {
            "type": "double",
            "name": "location_latitude"
          },
          {
            "type": "double",
            "name": "location_longitude"
          },
          {
            "type": "double",
            "name": "location_altitude"
          },
          "insertion_agent",
          "measure_timestamp",
          "start_timestamp",
          "end_timestamp",
          "insertion_timestamp"
        ]
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false,
        "segmentGranularity": "month"
      },
      "dataSource": "dl_measurements",
      "transformSpec": {
        "transforms": [
          {
            "type": "expression",
            "name": "insertion_timestamp",
            "expression": "timestamp_format(__time)"
          }
        ]
      }
    }
  }
}
```

#### Ingestion performance

| Database     | Partitions    | Ingestion time |
| ------------ | ------------- | -------------- |
| Apache Hive  | 1462 partions | 02:55:8        |
| Apache Druid | 51 segments   | 01:17:15       |
<sub>Table 1: Ingestion numbers</sub>

Table 1 reports the number of partitions and the computed ingestion time for each database.
Apache Druid was 55,8% faster than Apache Hive to import 50 million rows.


### 2.4 Queries

To benchmark Apache Hive and Apache Druid, I used the most common query executed by MIND Foods Hub front-end;
In this way, the benchmark is as close as possible to the actual scenario use cases.
The queries are, where feasible, slightly optimized for each database to make use of Apache Hive table partitions and Apache Druid time segments.

#### Query 1

Query the first 100 measurements of `measurements` sensors in a month time range.  

Apache Hive

```sql
SELECT *
FROM mfh.dl_measurements
WHERE insertion_date >= '2021-12-01'
AND insertion_date <= '2021-12-31'
AND double_value IS NOT NULL
LIMIT 100;
```

Apache Druid

```sql
SELECT *
FROM dl_measurements 
WHERE __time >= '2021-12-01'
AND __time <= '2021-12-31'
AND double_value IS NOT NULL
LIMIT 100;
```

#### Query 2

Query the first 100 measurements of `tag` sensors in a month's time range.  

Apache Hive:

```sql
SELECT *
FROM mfh.dl_measurements
WHERE insertion_date >= '2021-12-01'
AND insertion_date <= '2021-12-02'
AND str_value IS NOT NULL
AND start_timestamp IS NULL
LIMIT 100;
```

Apache Druid

```sql
SELECT *
FROM dl_measurements 
WHERE __time >= '2021-12-01'
AND __time <= '2021-12-02'
AND str_value IS NOT NULL
AND start_timestamp IS NULL
LIMIT 100;
```

#### Query 3

Query the first 100 measurements of `phase` sensors in a month time range.  

Apache Hive:

```sql
SELECT *
FROM mfh.dl_measurements
WHERE insertion_date >= '2021-12-01'
AND insertion_date <= '2021-12-02'
AND str_value IS NOT NULL
AND start_timestamp IS NOT NULL
LIMIT 100;
```

Apache Druid

```sql
SELECT *
FROM dl_measurements
WHERE __time >= '2021-12-01'
AND __time <= '2021-12-02'
AND str_value IS NOT NULL
AND start_timestamp IS NOT NULL
LIMIT 100;
```

#### Query 4

Group and count all locations that are cultivated in "cassoni sx".  

```sql
SELECT location_id, location_name, location_botanic_name, location_cultivation_name, COUNT(*) AS number_of_measurements
FROM dl_measurements
WHERE location_id = 'cassoni_sx'
GROUP BY location_id, location_name, location_botanic_name, location_cultivation_name;
```

#### Query 5

Group and count all available sensors.  

```sql
SELECT sensor_id, sensor_type, sensor_desc_name, COUNT(*) AS number_of_measurements
FROM dl_measurements
GROUP BY sensor_id, sensor_type, sensor_desc_name;
```

#### Query 6

Calculate the average of all measurements of a specific sensor.  

```sql
SELECT sensor_id, location_cultivation_name, AVG(double_value) AS average
FROM dl_measurements
WHERE sensor_id = 'TS_0310B473-depth_soiltemperature'
AND location_id = 'cassoni_sx'
AND location_cultivation_name = 'Rubiaceae'
GROUP BY sensor_id, location_id, location_cultivation_name;
```

### 2.5 Performance testing using Apache JMeter

I used JMeter to assess single-user query performance; only one thread was employed to run the test plan.
The motivation under this choice is that Data Lake platforms are very different from application databases that need to support hundreds or thousands of concurrent connections under heavy load.
Especially for small to mid organizations, Data Lake platforms run data extraction for analytical and business purposes, consisting of a few concurrent requests, often triggering batch jobs that run from hours to days.

The performance testing was intended to run against each database HTTP API.
Sadly, [Apache Hive](https://hive.apache.org/) doesn't expose a set of REST API to interact with, similarly to other more recent platforms. Instead, a client is forced to use Hive JDBC drivers or Hive Thrift drivers to perform TCP connections. 
To work around this problem, I decided to write an application that works as an HTTP layer on top of [Javascript Hive driver](https://www.npmjs.com/package/hive-driver), so that a client can execute Hive SQL statements via HTTP.
The application code is hosted on SESAR Lab Github's organization: [https://github.com/SESARLab/hive-http-proxy](https://github.com/SESARLab/hive-http-proxy).

JMeter ran with the following conditions:

- The query cache for each database was disabled
- Single user testing
- Each query was configured to run 10 times (to have 10 samples per query) in a separate Thread Group 
- Each Thread Group ran consecutively (one at a time) to avoid side effects on other requests

All tests were executed in [CLI Mode](https://jmeter.apache.org/usermanual/get-started.html#non_gui), to [reduce resource usages](https://jmeter.apache.org/usermanual/best-practices.html#lean_mean), on a MacBook Pro Mid 2015 with these specs:

- CPU 2,8 GHz Intel Core i7, 4 core/8 threads
- 16 GB 1600 MHz DDR3 memory
- 512 SSD storage
- macOS 12.1 Monterey operative system 

The reason behind the choice to run performance testing on a laptop instead of running JMeter on the same network of the cluster (to minimize request latency) is simple;
I decided to simulate an everyday use case, or rather a stakeholder of an organization that uses its client machine to extrapolate and analyze data and business insights from its Data Lake.

<div style="page-break-after: always; visibility: hidden;"></div>

## 3. Test results

Each JMeter execution produced a CSV dataset containing the test results for each platform (one for Hive, the other for Druid);
I imported test results in JMeter to calculate, for each sample: Average response time, Minimum response time, Maximum response time, and Average response time Standard Deviation.

Also, I compared each query Average response time with [ResultsComparator](https://github.com/rbourga/jmeter-plugins-2/blob/main/tools/resultscomparator/src/site/dat/wiki/ResultsComparator.wiki) plugin, to quantify the performance difference between Apache Hive and Apache Druid executions by calculating  [Cohen's *d*](https://en.wikipedia.org/wiki/Effect_size#Cohen's_d) of the samplers, which is one of the most popular measures of the effect size.
Cohen's *d* is defined as the difference between two means divided by a standard deviation for the data; 
The magnitude of *d*, namely the difference between the means, is described by the table below:

| d           | Effect size |
| ----------- | ----------- |
| 0           | Similar     |
| < 0.01      | Negligible  |
| [0.01-0.19] | Very small  |
| [0.20-0.49] | Small       |
| [0.50-0.79] | Medium      |
| [0.80-1.19] | Large       |
| [1.20-1.99] | Very large  |
| >= 2.0      | Huge        |

JMeter test results are downloadable from this repository hosted on SESAR Lab Github's organization:

[https://github.com/SESARLab/mfh-dl-performace-testing](https://github.com/SESARLab/mfh-dl-performace-testing)

<div style="page-break-after: always; visibility: hidden;"></div>

### 3.1 Query 1

![](C:\Users\Gabriele\code\mind\mfh-dl-performace-testing\content\Average response time - Query 1.png)
<sub>Figure 1: Average response time for Query 1</sub>


| Query           | Average | Min  | Max  | Std. Dev. |
| --------------- | ------- | ---- | ---- | --------- |
| Hive - Query 1  | 382     | 346  | 457  | 39,50     |
| Druid - Query 1 | 160     | 148  | 229  | 23,22     |
<sub>Table 2: Query 1  numbers</sub>

| Performance | Cohen's d | Average Difference |
| ------- | --------- | ------------------ |
| Query 1 | 6.87      | Huge decrease      |
<sub>Table 3: Performance evaluation for Query 1</sub>

Query 1 execution is fast on both platforms, staying under a 500 milliseconds threshold since it takes advantage of time partitions on Apache Hive and time segmentations on Apache Druid.
Using Apache Druid, we can see a considerable decrease in Average response time value, making this platform greatly performant than Hive.

<div style="page-break-after: always; visibility: hidden;"></div>

### 3.2 Query 2

![](C:\Users\Gabriele\code\mind\mfh-dl-performace-testing\content\Average response time - Query 2.png)
<sub>Figure 1: Average response time for Query 2</sub>


| Query           | Average | Min  | Max  | Std. Dev. |
| --------------- | ------- | ---- | ---- | --------- |
| Hive - Query 2  | 376     | 359  | 401  | 13,55     |
| Druid - Query 2 | 152     | 148  | 163  | 4,17      |
<sub>Table 4: Query 2  numbers</sub>

| Performance | Cohen's d | Average Difference |
| ----------- | --------- | ------------------ |
| Query 2     | 22.37     | Huge decrease      |
<sub>Table 5: Performance evaluation for Query 2</sub>

Like Query 1, Query 2 makes use of time partitions on Apache Hive and time segmentations on Apache Druid, making its execution fast on both platforms. But, again, using Apache Druid, we can see a vast decrease in the Average response time value.

<div style="page-break-after: always; visibility: hidden;"></div>

### 3.3 Query 3

![](C:\Users\Gabriele\code\mind\mfh-dl-performace-testing\content\Average response time - Query 3.png)
<sub>Figure 1: Average response time for Query 3</sub>


| Query           | Average | Min  | Max  | Std. Dev. |
| --------------- | ------- | ---- | ---- | --------- |
| Hive - Query 3  | 363     | 350  | 383  | 12,67     |
| Druid - Query 3 | 154     | 148  | 168  | 5,99      |
<sub>Table 6: Query 3 numbers</sub>

| Performance | Cohen's d | Average Difference |
| ----------- | --------- | ------------------ |
| Query 3     | 21.15     | Huge decrease      |
<sub>Table 7: Performance evaluation for Query 3</sub>

Query 3 follows the same trends, achieving a massive decrease in Average response time value for its execution on Apache Druid.
We can notice how the behaviour of all time queries is the same on both platforms, with a similar Average response time between Query 1, Query 2 and Query 3 per database and minor Standard Deviation.

<div style="page-break-after: always; visibility: hidden;"></div>

### 3.4 Query 4

![](C:\Users\Gabriele\code\mind\mfh-dl-performace-testing\content\Average response time - Query 4.png)
<sub>Figure 1: Average response time for Query 4</sub>


| Query           | Average | Min    | Max    | Std. Dev. |
| --------------- | ------- | ------ | ------ | --------- |
| Hive - Query 4  | 521931  | 518235 | 526918 | 3157,38   |
| Druid - Query 4 | 2027    | 2020   | 2038   | 6,53      |
<sub>Table 8: Query 1 numbers</sub>

| Performance | Cohen's d | Average Difference |
| ----------- | --------- | ------------------ |
| Query 4     | 232.87    | Huge decrease      |
<sub>Table 9: Performance evaluation for Query 4</sub>

Query 4 requires grouping four different columns and showing another behaviour between the platforms; 
While Apache Druid execution for Query 4 is sensibly slower than time queries, with an Average response time of 2 seconds circa, Apache Hive is enormously slower, requesting 8,69 minutes to query the data. 
Two factors essentially cause this: 

1. The query doesn't use any partitions, forcing Hive to do a full table scan to select and group the requested values 
2. The query uses 61 Mapper and 62 reducers for its execution, so it requires a discrete amount of I/O synchronization between the Mappers and Reducers, resulting in different disk writing operations that are notoriously slow

On the other side, Apache Druid uses an in [memory algorithm](https://druid.apache.org/docs/latest/querying/groupbyquery.html#strategies) to aggregate data from the segments, streaming the results back directly from the Broker to the client, resulting in high-speed query execution.

<div style="page-break-after: always; visibility: hidden;"></div>

### 3.5 Query 5

![](C:\Users\Gabriele\code\mind\mfh-dl-performace-testing\content\Average response time - Query 5.png)
<sub>Figure 1: Average response time for Query 5</sub>


| Query           | Average | Min    | Max    | Std. Dev. |
| --------------- | ------- | ------ | ------ | --------- |
| Hive - Query 5  | 511250  | 503896 | 518260 | 3898,01   |
| Druid - Query 5 | 1417    | 1401   | 1449   | 12,75     |
<sub>Table 10:  Query  5 numbers</sub>

| Performance | Cohen's d | Average Difference |
| ----------- | --------- | ------------------ |
| Query 5     | 184.97    | Huge decrease      |
<sub>Table 11: Performance evaluation for Query 5</sub>

Query 5 shows the exact behaviour of Query 6, with Hive, forced to do a full table scan to aggregates sensor's related data, resulting in 
an Average response time of 8,52 minutes circa per query execution.
Apache Druid is exceedingly performant, remaining under the 2 seconds threshold for Query 5 completion.

<div style="page-break-after: always; visibility: hidden;"></div>

### 3.6 Query 6

![](C:\Users\Gabriele\code\mind\mfh-dl-performace-testing\content\Average response time - Query 6.png)
<sub>Figure 1: Average response time for Query 6</sub>


| Query           | Average | Min    | Max    | Std. Dev. |
| --------------- | ------- | ------ | ------ | --------- |
| Hive - Query 6  | 580362  | 575520 | 585806 | 2952,75   |
| Druid - Query 6 | 266     | 264    | 271    | 2,01      |
<sub>Table 12: Query 6 numbers</sub>

| Performance | Cohen's d | Average Difference |
| ----------- | --------- | ------------------ |
| Query 6     | 277.84    | Huge decrease      |
<sub>Table 13: Performance evaluation for Query 6</sub>

Query 6 behave the same in Apache Hive, bound to a full table scan that requests 9,67 minutes to accomplish its execution.
Apache Druid acts differently from  Query 4 and Query 5, with Query 6 Average response time of only 264 milliseconds.

<div style="page-break-after: always; visibility: hidden;"></div>

## 4. Conclusions

Thesis conclusions

## 5. References
