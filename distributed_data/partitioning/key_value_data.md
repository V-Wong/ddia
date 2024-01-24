# Partitioning of Key-Value Data
## Overview
Our goal with partitioning is to spread data and query load **evenly across nodes**. If the partitioning is unfair, so that some partitions have more data or queries than others, we call it **skewed**. A partition with disproportionately high load is called a **hot spot**.

Assigning records to nodes **randomly** will distribute the data evenly, but when we read a particular item, we have no way of knowing which node it is on, and so have to **query all nodes** in parallel. We can do better in the simple case where we always access a record by its **primary key**.

## Partitioning by Key Range
A **continuous range of keys** (from some minimum to some maximum) can be assigned to each partition. If we know the **boundaries** between the ranges, we can easily determine which partition contains a given key. If we also know which partition is assigned to which node, we can make a request directly to the appropriate node.

The range of keys need **not be evenly spaced**, because the data may **not be evenly distributed**. These boundaries may be chosen manually by an administrator, or automatically by the database.

Within each partition, we can keep keys in **sorted order** (see [SSTables and LSM-Trees](foundations/storage_retrieval/data_structures.md#sorted-string-tables-sstables)). This allows **easy range scans**, and we can treat the key as a **concatenated index** in order to fetch several related records in one query (see [Multi-column indexes](foundations/storage_retrieval/data_structures.md#sorted-string-tables-sstables)).

The downside of key-range partitioning is that certain access patterns can **lead to hot spots**. If the key is timestamp, then the partitions correspond to ranges of time - e.g. one partition per day. Because data is written as the measurements occur, all the writes end up going to the same partition (the one for today), so that partition can be overloaded with writes while others sit idle.

To avoid this problem, we could use something else as the first element of the key. For example:
- We could prefix each timestamp with a client name.
- The partition is first by client name and then by time. 
- When we want to fetch the values of multiple clients within a time range, we perform a separate range query for each client name.

## Partitioning by Hash of Key
Because of the risk of skew and hot spots, many distributed datastores use a **hash function** to determine the partition for a given key. A good hash function takes skewed data and makes it **uniformly distributed** even if keys are very similar. 

Once we have a suitable hash function we assign each partition a **range of hashes**. The partition boundaries can be evenly spaced, or they can be chosen **pseudorandomly**.

The downside of hash-based partitioning is that we **can't efficiently perform range queries**. Keys that were once adjacent are now scattered across all the partitions, so their sort order is lost.

Cassandra achieves a **compromise** between the two partitioning strategies:
- A table can be declared with a **compound primary key** consistent of several keys.
- The **first part** of the compound key is hashed to determine the partition.
- The other columns are used as a concatenated index for sorting the data in Cassandra's SSTables.
- Queries therefore cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it can perform an efficient range scan over the other columns of the key.

This concatenated index approach enables an elegant data model for **one-to-many relationships**. If the primary key for updates is chosen to be `(user_id, update_timestamp)`, we can effeciently retrieve all updates by a particular user within some time interval, sorted by timestamp. Different users may be stored on different partitions, but within each user, the updates are stored ordered by timestamp on a single partition.

## Skewed Workloads and Relieving Hot Spots
Hash-based partitioning can help reduce hot spots. But it can't eliminate them: if all reads and writes are for the same key, all requests will be routed to the same partition.

Most data systems can't automatically compensate for such highly skewed workloads, so it is the **application's resposibility** to reduce the skew. For example, if one key is very hot, a **random number** can be prepended or appended to the key. This would split the writes to the key evenly across different keys, and to different partitions.

This however requires reads to do more work. Each read to the key now needs to read the data from all the split keys and combine it. Additionally, it requires **extra bookkeeping** as only some keys will benefit from being split. Thus, we need to also keep track of which keys are being split
