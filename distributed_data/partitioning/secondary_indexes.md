# Partitioning and Secondary Indexes
## Overview
Partitioning becomes more complicated once there are **secondary indexes**. A secondary index usually doesn't identify a record uniquely, but is rather a way of searching for occurrences of a particular value. This means secondary indexes don't map neatly to partitions and other strategies are required.

## Partitioning Secondary Indexes by Document
In this strategy, each partition is **completely separate**: each partition maintains its **own secondary indexes**, covering only the documents in that partition. It doesn't care what data is stored in other partitions. Whenever we write to the database, we only need to deal with the partition that contains the document we are writing. Hence, a document-partitioned index is called a **local index**.

Reading however requires sending our query to **all partitions** and **combine the results**. This is known as **scatter/gather**, and it can make read queries on secondary indexes expensive.

## Partitioning Secondary Indexes by Term
Rather than each partition having its own secondary index (a local index), we construct a **global index** that covers data in **all partitions**. This global index must **itself be partitioned**, but can be partitioned differently to the primary key index.

This kind of index is **term-partitioned** because the term we're looking for determines the partition of the index. As before, we can partition the index by the term itself, or using a hash of the term.

The advantage of a global index is that it can make **reads more efficient**. Rather than scatter/gather-ing over all partitions, a client only needs to make a request to the partition containing the term it wants. The downside is that **writes are slower** and more complicated, because a write to a single document may now affect multiple partitions of the index.

Furthermore, updates to global secondary indexes are often **asynchronous** so we don't have strong consistency guarantees.
