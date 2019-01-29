--- 
title:  "Easily Migrate Postgres/MySQL Records to InfluxDB"
image: /images/influxdb.png
date:   2018-07-05 15:04:23
tags: [python, influxdb, time-series, postgreSQL, MySQL]
description: Computers have been collecting and storing data in relational/schema systems for many years. However, digital storage growth outpaces that of computing processing power by leaps and bounds. Additionally, the amount of unstructured that is collected greatly exceeds that of structured data, further limiting the utility of tradional database systems. For these reasons, today's data storage technqiues call for some new technical constructs required to break the boundaries of traditional transaction-processing databases.

excerpt_separator: <!--more-->
---
Computers have been collecting and storing data in relational/schema systems for many years. However, digital storage growth outpaces that of computing processing power by leaps and bounds. Additionally, the amount of unstructured that is collected greatly exceeds that of structured data, further limiting the utility of tradional database systems. For these reasons, today's data storage technqiues call for some new technical constructs required to break the boundaries of traditional transaction-processing databases.
<!--more-->

<img src="/images/data-explosion.jpg" alt="drawing" width="600px"/>

In an effort to garner meaning from rapidly growing data sources, an increasing amount of organizations find themselves having to move their data from a traditional relational database like PostgreSQL or MySQL to a non-relational database like MongoDB, or in some cases, a Time Series Database (TSDB). The scenario under consideration for this article is based on the need to analyze and visualize trends over large periods of time over large amounts of [time series](https://en.wikipedia.org/wiki/Time_series#Panel_data) data. There are a few reasons as to why typical relational databases are inefficient for storing time series data. First, at every measurement interval, tables are populated with fresh data that increases table cardinality. The consequence is that table cardinality as well the space taken on disk increases with the number of measurements. As soon as table indexes become large enough to prevent themselves to be cached, data retrieval becomes significantly slow thus jeopardizing the performance of applications sitting on top ofthe database. Therefore, a TSDB is much more attractive solution for the anaysis and storage of time series data for two main reasons:

#### Efficiency

A TSDB co-locates chucks of data within the same time range on the same physical section of the database cluster and hence enables quick access for faster, more efficient analysis. As a result, range queries, even over many records complete extremely quickly.  In many cases, regular databases produce an index out of memory error because of the sheer volume of time series dataand subsequently affect the performance of read and write operations.

#### Usability

TSDBs typically include functions and operations that are common to time series data analysis. First, they utilize data retention policies, continuous queries, flexible time aggregations,and range queries.  Additionally, they offer mathematical querying functionality that allows users to understand their data in ways common to the analysis of time series data. As a result, the overall user experience in dealing with time series data is significantly improved in comparison to database systems in which this functionality needs to be manually configured and developed.

### The Python Script

The following script transfers records from a PostgreSQL or MySQL database to InfluxDB - just fill out the relevant database connection information and schema design object as required. You need to also make sure you have the appropriate python libraries installed: `pip install psycopg2 mysqlclient indfluxdb`. Note, InfluxDB incorporates some [key concepts](https://docs.influxdata.com/influxdb/v1.5/concepts/key_concepts/) that might seem unfamiliar for those coming from a typical relation database background. These should be reviewed and understood before using the script below. 

<script src="https://gist.github.com/Zir0-93/b7d2cf47ae54e53100357e0cebae4a57.js"></script>
