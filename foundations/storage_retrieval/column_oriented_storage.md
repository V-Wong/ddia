# Column-Oriented Storage
## Overview
Fact tables often **contain many columns**, but typical data warehouse queries only **access a few of them at one time**.

In most OLTP databases, storage is laid out in a **row-oriented** fashion where all values from one row are stored next to each other. Similarly, document databases typically store an entire document as a contiguous sequence of bytes.

In contrast, **column-oriented** storage store all values from **each column together**. If each column is stored in a **separate file**, a query only needs to read and parse the columns actually used, saving a lot of work. This layout relies on each column file containing **rows in the same order**.

## Column Compression
We can further reduce the demands on disk throughput by **compressing data**.

### Bitmap Encoding
Often, the number of **distinct values** in a column is **small compared to the number of rows**. We can take a column with n distinct values and turn it into n separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has that value, and 0 if not.

If n is very small, those bitmaps can be stored with one bit per row. But if n is bigger, most of the bitmaps will be sparse. We can additionaly **run-length encode** the bitmaps.

These bitmaps make logical operator (OR, AND, etc) queries efficient as we can perform **bitwise operations** on the bitmaps.

### Memory Bandwidth and Vectorised Processing
To make efficient data warehouse queries, developers of analytical databases worry about:
- Bandwidth of getting data from disk into memory.
- Bandwidth from main memory into the CPU cache.
- Branch mispredictions.
- Bubbles in the CPU instruction processing pipeline.
- Usage of single-instruction-multi-data (SIMD) instructions in modern CPUs.

Besides reducing volume of data loaded from disk, column-oriented storage layouts also make **efficient use of CPU cycles**. For example, the query engine can take a chunk of compressed column data that fits in the CPU's L1 cache and iterate through it in a tight loop. This compression allows more rows from a column to fit in the same amount of L1 cache. 

Additionally, the bitwise operators described previously can be designed to operate on such chunks of comrpessed column data directly. This is known as **vectorised processing**.

## Sort Order in Column Storage
The rows in a column store do not have to be ordered in any specific way. But it can be useful to impose an order for performance optimisations. When sorting a column store, it does not make sense to sort each column independently, as we no longer know which items in the columns belong in the same row. Thus, the data must be **sorted an entire row at a time**.

The administrator of the database can choose the sort columns to optimise common queries. For example, if queries often target date ranges, such as the last month, it makes sense to first sort on the date key. Then the query optimiser can scan only the rows from the last month, instead of all rows. A second column can then be used to break ties in the first column.

Another advantage of sorted order is that it can help compression by creating **long sequences of the same value**. A run-length encoding would then drastically decrease the size of the data. This compression effect is strongest on the first column, and decreasingly weak on each column after that.

### Several Different Sort Orders
**Different queries** benefit from **different sort orders**, so it may be beneficial to store the same data sorted in **several different ways** (possibly in different replicas). This is similar to having **multiple secondary indexes** in a row-oriented store. But row-oriented stores keeps every row in one place (in the heap file or a clustered index), and secondary indexes just contain pointers to the matching rows. In a column store, there aren't usually pointers to data elsewhere, only columns containing values.

## Writing to Column-Oriented Storage
Column-oriented storages help make read queries faster, but also **makes writes more difficult**. An update-in-place approach such as B-trees is impossible as inserting a row may require rewriting all column files. Instead LSM-trees can be used:
- All writes first go to an in-memory store, where they are added to a sorted structure and prepared for writing to disk.
    - It doesn't matter if this in-memory store is row-oriented or column-oriented.
- When enough writes have accumulated, they are merged with the column files on disk and written to new files in bulk.

## Aggregation Optimisation
### Materialised Views
Data warehouse queries often involve **aggregate functions**. If the same aggregates are used by many different queries, it can be beneficial to **cache the aggregate values**. One way of creating such a cache is a **materialised view**. This is like standard (virtual) views in the relational data model. However, a materialized view is an **actual copy of the query results**, whereas a virtual view is just a **shortcut for writing queries**. When the underlying data changes, a materialized view also needs to be updated. This makes writes more expensive, which is why materialised views are typically used in OLAP and not OLTP databases.

### Data Cubes (or OLAP Cubes)
A **data cube** is a special case of a materialised view. It is a **grid of aggregates grouped by different dimensions**. This allows certain queries to become very fast, because they have effectively been **precomputed**. It however has **less flexibility** than querying the raw data if the dimension of interest is not in the data cube. Most data warehouses hence try to keep as much raw data as possible, and use data cubes only as performance boost for certain queries.
