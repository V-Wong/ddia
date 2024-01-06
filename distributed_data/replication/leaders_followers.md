# Leaders and Followers
## Context
**Replication** means keeping a copy of the same data on multiple machines connected via a network. This has various benefits:
- Keeps data **geographically close** to users (reduce latency).
- Allows system to **continue working** even if **some parts have failed** (increase availability).
- Scales number of machines that can **serve read queries** (increase read throughput).

In this chapter, the assumption is that each machine can hold a **copy of the entire dataset**. The diffculty of replication lies in **handling changes** to replicated data and ensuring it **reaches all replicas**. 

## Overview
The most common replication method is **leader-based replication**:
1. 1 replica is designated as **leader**. When clients want to write to the database, it must send requests to the leader, which first writes the changes locally.
2. The other replicas are known as **followers**. When the leader writes new data locally, it also sends the data change to all its followers as part of a **replication log** or **change stream**. Each follower takes the log from the leader and updates its local copy of the database accordingly, applying all writes in the same order as they were processed on the leader.
3. When a client wants to read from the database, it can query the leader or any of its followers.

This replication strategy is **built-in** to many relational databases such as PostgreSQL and some nonrelational databases such as MongoDB. It is also not restricted to only databases: highly available distributed message brokers such as Kafka also use it.

## Synchronous Versus Asynchronous Replication
Replication can happen in two modes (configured individually for each follower):
- **Synchronously**: leader waits for follower to confirm its write before reporting success to user.
- **Asynchronously**: leader does not wait for response from follower before reporting success to user.

Replication is usually fast, but a follower can fall behind the leader by several minutes or more because:
- Follower is recovering from a failure.
- Follower is operating near maximum capacity.
- There are network problems between the nodes.

The advantage of synchronous replication is that the follower is **guaranteed** to have an **up-to-date copy** of the data that is **consistent with the leader**. If the leader fails, we know the data is available on the follower. The disadvantage is that if the synchronous follower doesn't respond (crashes or otherwise), the **write cannot be processed**. The leader must **block all writes** and wait until the synchronous replica is available again.

It is thus impractical for all followers to be synchronous: any node outage would halt the whole system. A common practice is **semi-synchronous** which **makes 1 follower synchronous**, and the others asynchronous. If the synchronous follower is unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that **at least two nodes** have the up-to-date copy of the data.

Often, leader-based replication is configured to be **completely asynchronous**. If the leader fails and is not recoverable, any writes not yet replicated would be lost. Thus, a write is **not guaranteed to be durable**, even if it has been confirmed to the client. The advantage is that the leader can **continue processing writes**, even if **all followers have fallen behind**. Therefore, despite the tradeoff, this strategy is still commonly used.

## Setting Up New Followers
We may periodically **set up new followers**, to increase the number of replicas, or to replace failed nodes. We must ensure that the new follower has an **accurate copy** of the leader's data.

Simply copying data files from one node to another is typically not sufficient: clients are constantly writing to the database, so a copy would see different data at different times.

We could lock the database, but that would ruin high availability. The strategy to set up a new follower without downtime is as follows:
1. Take a consistent snapshot of the leader's database at some point in time.
2. Copy the snapshot to the new follower node.
3. The follower connects to the leader and requests all data changes that have happened since the snapshot was taken. 
    - This requires that the snapshot is associated with an exact position in the leader's replication log.
4. When the follower has processed the backlog of data changes since the snapshot, we say it has **caught up**. 
    - It can not continue to process data changes from the leader as they happen.

## Handling Node Outages
### Follower Failure: Catch-up Recovery
Each follower keeps locally a **log of data changes received from the leader**. If a follower crashs and is restarted, or if the network between leader and the follower is temporarily interrupted, the follower can recover easily:
1. From its log, follower knows that the last transaction that was processed before the fault occured.
2. The follower can connect to the leader and request all data changes that occurred during the time when the follower was disconnected.
3. The follower applies these changes until it has caught up to the leader and can continue receiving a stream of data changes as before.

### Leader Failure: Failover
Handling failure of the leader is trickier and usually consists of the following steps:
1. **Determining that the leader has failed**: nodes frequently bounce messages back and forth between each other, and if a node doesn't response after some period of time, it is assumed to be dead.
2. **Choosing a new leader**: the best candidate for leadership is usually the replica with the **most up-to-date data changes** from the **old leader**. Getting all nodes to agree on a new leader is a **concensus problem**.
3 **Reconfiguring the system to use the new leader**: clients now need to **send write requests to the new leader**. If the old leader comes back, the system needs to ensure that the old leader becomes a follower and recognises the new leader.

Failover is fraught with complexities:
- If asynchronous replication is used, the new leader may not have **received all writes** from the **old leader** before it failed. If the former leader rejoins the cluster after a new leader has been chosen, what should happen to those writes? The new leader may have received conflicting writes in the meantime. The most common solution is for the old leader's unreplicated writes to simply be discarded, which may vioalte clients' durability expectations.
- Discarding writes is especially dangerous if other storage systems needs to be **coordinated with the database contents**. It can cause **mismatches** between the storage systems, resulting in incorrect behaviour.
- In certain fault scenarios, two nodes could believe they are the leader. This is known as **split brain** and can lead to lost or corrupted data as there is no process for resolving conflicts if both leaders accept writes. Some systems will shut down one node if two leaders are detected.
- The **correct timeout** to determine a dead leader is a critical decision:
    - Too short of a timeout may cause **unnecessary failovers**.
    - Too long of a timeout means a **longer time to recovery**.

## Implementation of Replication Logs
### Statement-based Replication
In the simplest case, the leader **logs every write request** (statement) that it executes and sends that statement log to its followers. This can break down in various ways:
- Any statement that calls a **non-deterministic function** such as `NOW()` is likely to generate a different value on each replica.
- If statements use an **autoincrementing column**, or if they depend on existing data in the database (e.g. `UPDATE ... WHERE <some condition>`), they must be execueted in **exactly the same order** on each replica, or else they may have a different effect. This can be limiting when there are multiple concurrently executing transactions.
- Statements that have **side effects** (e.g. triggers, stored procedures, user-defined functions) may result in different side effects occurring on each replica.

There are workarounds for these issues, but other replication methods are now generally preferred.

### Write Ahead Log (WAL) Shipping
Recall that for certain storage engines, every write is appended to a log:
- For log-structured storage engines (e.g. LSM Trees), this log is the main place for storage.
- For B-Trees which overwrites individual disk blocks, every modification is first written to WAL so that the index can be restored to a consistent state after a crash.

This exact log can be used to build a replica on another node: besides writing the log to disk, the leader also sends it across the network to its followers. When the follower processes this log, it builds a copy of the **exact same data structures** as found on the leader.

This replication method is used in PostgreSQL and Oracle. Its main disadvantage is that the log describes the data on a **low level**: a WAL contains details of which bytes were changed in which disk blocks. This makes replication **closely coupled to the storage engine**. It is typically not possible to run different versions of the database software on the leader and followers. 

This can have a large operational impact: 
- If the replication protocol allows the follower to use newer software than the leader, we can perform a zero-downtime upgrade by first upgrading the followers and the performing a failover to make one of the upgraded nodes the new leader. 
- If the replication protocol does not allow version mismatch, as is common with WAL shipping, such upgrades require downtime.

### Logic (Row Based) Log Replication
An alternative is **different log formats** for replication and for the storage engine. This kind of replication log is called a **logical log** to distinguish it from the storage engine's (physical) data representation.

A logical log for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row:
- For an inserted row, the log contains the new values of all columns.
- For a deleted row, the log contains enough information to uniquely identify the row that was deleted. This is typically the primary key, if present, otherwise the old values of all columns.
- For an updated row, the log contains enough information to uniquely identify the updated row, and the new values of all columns (or at least the ones changed).

A transaction that modifies several rows generates **several log records**, followed by a record indicating that the transaction was committed.

Since a logical log is **decoupled from the storage engine internals**, it can be easily allow version mismatch between the leader and followers. It is also **easier for external applications to parse**. 

### Trigger Based Replication
If we want **more control** over replication, such as only replicating a subset of the data, then we may need to **move replication up to the application layer**.

A trigger lets us register **custom applicatino code** that is **automatically executed** when a data change occurs in a database system. This trigger can log this change to a separate table, which can then be read by an **external process**. That external process can then apply any necessary application logic and replicate the data change to another system. 

This has **greater overhead** compared to other methods, and is more prone to bugs and limitatinos than the database's built-in replication. However, it can be useful due to its flexibility.
