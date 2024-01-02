# Transaction Processing or Analytics?
## Online Transaction Process (OLTP)
A **transaction** is a **group of reads and writes** that form a **logical unit**. Transaction processing need not be ACID, but need to support **low-latency** reads and writes for clients. This is opposed to **batch processing** which only run **periodically**.

Although databases started being used for many different kinds of data, the basic **access pattern** remained similar to processing business transactions. An application typically looks up a small number of records by some key, using an index. Records are inserted or updated based on the user's input.

This access pattern is known as **online transaction processing**.

## Online Analytics Processing
Recently, databases have been increasingly used for **data analytics**. An analytics query typically scans over a huge number of records, reading only a few columns per record, and calculates aggregate statistics. 

These queries are often written by business analysts to help management make better decisions.

## Transaction Processing vs Analytic Systems Characteristics

|**Property**|**OLTP**|**OLAP**|
|----|----|----|
|Main read pattern|Small number of recrods per query, fetched by key.|Aggregate over large number of records.|
|Main write pattern|Random-access, low-latency writes from user input.|Bulk import (ETL) or event stream.|
|Primarily used by|End user/customer, via web application.|Internal analyst, for decision support.|
|What data represents|Latest state of data (current point in time).|History of events that happened over time.|
|Dataset size|Gigabytes to terabytes.|Terabytes to petabytes.|

## Data Warehousing
A **data warehouse** is a separate database to the OLTP systems that analysts can query **without affecting OLTP operations**. It contains a read-only copy of the data in all the various OLTP systems in a company. Data is extracted from OLTP databases, transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse. This process of getting data into the warehouse is known as **Extract-Transform-Load (ETL)**.

### Divergence Between OLTP Databases and Data Warehouses
The indexing algorithms that work well for OLTP are not very good for answering analytic queries. A big advantage of a separate data warehouse is that the data warehouse can be **optimised for analytic access patterns**.

Many data warehouses expose an **SQL interface**, because SQL is generally good for analytic queries. On the surface, data warehouses and relational OLTP databases look similar, however the internal implementation details are very different to optimise for different access patterns.

### Star and Snowflake Schemas
Analytics has much less diversity in data models compared to OLTP systems. Many data warehouses are used in a fairly formulaic style, known as a **star schema** (also known as dimensional modelling)**. 

![https://upload.wikimedia.org/wikipedia/commons/b/bb/Star-schema.png]

In the center of this schema is a **fact table** that represents an **event** that occurred at a particular time. This fact table can become extremel large as each individual event is captured.Some of the columns in the fact table are **attributes**, while other columns are **foreign key references** to other tables, called **dimension tables**. These dimensions represent the who, where, when, how, and why of the event.

A variation of the star schema is the **snowflake schema**, where dimensions are further broken down into **subdimensions**. Snowflake schemas are **more normalised** than star schemas, but star schemas are preferred because they are **simpler for analytics**.
