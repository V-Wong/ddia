# Leaderless Replication
## Overview
In leader-based replication, a client sends a write request to 1 node (the leader), and the database system copies that write to other replicas. A leader determines the order in which writes should be processed, and followers apply the leader's writes in the same order.

Some databases **abandon the concept of leaders** and allow **any replica** to **directly accept writes** from clients. This strategy is used by DynamoDB, Riak, Cassandra, and Voldemort.

In some leaderless implementations, the client directly sends its writes to **several replicas**, while in others, a **coordinator node** does this on behalf of the client. However, unlike a leader-based database, that coordinator **does not enforce a particular ordering of writes**. 

## Writing to the Database When a Node Is Down
In a leaderless configuration, **failover does not exist**. A client can send a request to all replicas in parallel, and it is sufficient for only **some of the replicas** to acknowledge the write. The client ignores the fact that one replica missed the write.

Once the unavailable node comes back online, any writes while the node was down are missing from that node. Thus, a client reading from that node may get **stale** values as responses. To solve this, a client's read requests are **sent to several nodes in parallel**. The client may get **different responses** from different nodes. **Version numbers** are used to determine which value is newer.

### Read Repair and Anti-Entropy
The replication scheme should ensure that eventually all data is copied to every replica. After an unavailable node comes back online, there are various approaches to catch up on missed writes:
- **Read repair**: when a client makes a read from several nodes in parallel, it can **detect stale responses** and write back the newer value to that replica.
- **Anti-entropy process**: a background process can continuously look for differences in data between replicas and copy and missing data across replicas. Unlike replication logs in leader-based replication, this process does not copy writes in any particular order, and there may be **significant delay** before data is copied.

### Quorum For Reading and Writing
Suppose we have:
- n replicas.
- Every write must be confirmed by w nodes to be considered successful.
- Each read queries at least r nodes. 

Then as long as **w + r > n**, we expect to get an **up-to-date value** when reading, because at least one of the r nodes must be up to date. Reads and writes that satisfy this condition are known as **quorum** read and writes.

These parameters n, w, and r are typically **configurable** and tuneable to **fit our workload**. For example, a heavy read and low write workload may nenefit from setting w = n and r = 1. This makes reads faster, but just 1 failed node causes all writes to fail.

The quorum condition allows the system **tolerate unavailable nodes** as follows:
- If w < n, we can still process writes if a node is unavailable.
- If r < n, we can still process reads if a node is unavailable.
- With n = 3, w = 2, r = 2, we can tolerate 1 unavailable node.
- With n = 5, w = 3, r = 3, we can tolerate 2 unavailable nodes.
- Normally, reads and writes are always sent to **all n replicas** in parallel. The parameters w and r determine how many nodes we **wait** for. If fewer than the required w or r nodes are available, writes or reads return an error.

Note that there may be **more than n nodes** in the cluster, but **any given value** is stored only on n nodes. This allows the dataset to be **partitioned**.

## Limitations of Quorum Consistency
Sometimes we may set w and r to smaller numbers that **don't satisfy the quorum condition**. In this case, reads and writes are still sent to n nodes, but a smaller number of successful responses is required for the operation to succeed. This increases the likelihood of reading **stale values**, but allows **lower latency** and **higher availability**. 

However, even with w + r > n, there could be edge cases where stale values are returned:
- If a **sloppy quorum** is used: the w writes may end up on **different nodes** than the r reads, so there is no longer a guaranteed overlap between the r nodes and the w nodes.
- If two writes occur **concurrently**: it is not clear which one happened first.
- If a write happens concurrently with a read: the write may be reflected on only some replicas. It's undetermined whether the read returns the old or new value.
- If a write succeeded on some replicas but failed on others, and overall succeeded on fewer than w replicas: it is **not rolled back** on the replicas where it succeeded. If a write was reported as failed, subsequent reads may or may not return the values from that write.
- If a node carrying a new value fails, and its data is restored from a replica carrying an old value: the number of replicas storing the new value may fall below w, breaking the quorum condition.
- Even if everything is working correctly: there are edge cases in which we can get unlucky with timing.

In general, quorum-based databases are generally optimised for use cases that can **tolerate eventual consistency**.

### Monitoring Staleness
From an **operational perspective**, we want to monitor whether our databases are returning up-to-date results even if we tolerate stale reads. This indicates **overall health** of the replication. If it falls behind significantly, it should alert us so that we can **investigate the cause**.

For leader-based replication:
- Writes are applied to the leader and to followers in the **same order**.
- Each node has a **position** in the replication log (the number of writes it has applied locally).
- By subtracting a follower's current position from the leader's current position, we can **measure the replication lag**.
- Thus, leader-based databases typically expose **metrics** for the replication lag, which can be fed into monitoring systems.

However, in systems with leaderless replication, there is no fixed order in which writes are applied, making monitoring more difficult.

## Sloppy Quorums and Hinted Handoff
In a large cluster (>> n nodes), it's likely that the client can connect to **some** database nodes during network interruption, but not the nodes it needs to a assemble a quorum for a **particular value**. In this case, database designers face a tradeoff:
- Is it better to return errors to **all requests** for which we cannot reach a quorum.
- Or should we **accept writes anyway**, and write them to some nodes that are reachable but **aren't among the n nodes** on which the value usually lives.

The latter is known as **sloppy quorum**: writes and reads still require w and r successful responses, but those may include nodes that are not among the designated n "home" nodes for a value. By analogy, if we lock ourself out of our house, we may knock on the neighbour's door and ask whether we can stay on their couch temporarily.

Once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate "home" nodes. This is called **hinted handoff**. By analogy, once we find the keys to our house again, our neighbour asks us to get off their couch and go home.

Sloppy quorums **increase write availability**: as long as **any** w nodes are available, the database can accept writes. However, this means that even when w + r > n, we cannot be sure to read the latest value for a key, because the latest value may have been temporarily written to some nodes outside of n.

Sloppy qurorum **isn't actually a quorum**. It's only an assurance of durability and is optional in all common Dynamo implementations.

#### Multi-Datacenter Operation
Leaderless replication is suitable for **multi-datacenter operations**, since it can tolerate conflicting concurrent writes, network interruptions and latency spikes.

Cassandra and Voledmort implement multi-datacenter support within the **normal leaderless model**:
- The number of replicas n includes nodes in **all datacenters**, and we specify how many of the n replicas we want in **each datacenter**.
- Each write is sent to **all replicas**, regardless of datacenter, but the client only waits for acknowledgement of quorum **within its local datacenter** so that it is unaffected by delays and interruptions in cross-datacenter links.
- The higher-latency writes to other datacenters are then transferred asynchronously (though can be configured).

Riak is more similar to **multi-leader replication**:
- All communication between client and databases nodes are **local to one datacenter**, so n describes the number of replicas within **one datacenter**.
- Cross-datacenter replication between database clusters happens asynchronously, in a style similar to multi-leader replication.

## Detecting Concurrent Writes
Dynamo-style databases allow several clients to **concurrently write** to the **same key**, which means that conflicts will occur even if strict quorum was used. This situation is similar to multi-leader replication, although it can also arise during read repair or hinted handoff.

The problem is that events may arrive in a **different order** at **different nodes**, due to variable network delays and partial failures.

If each node simply overwrote the value for a key whenever it received a write request from a client, the nodes would become **permanently inconsistent**.

In order to become eventually consistent, replicas should **converge towards the same value**. Unfortunately, most replicated databases are poor at handling this automatically: if we want to avoid losing data, as application developers, we must know the internals of our database's **conflict handling**.

### Last Write Wins (LWW)
We can declare that each replica need only store the most **recent value** and allow **older values** to be **overwritten** and **discarded**. Then, eventually all replicas will eventually converge to the same value.

Unfortunately, clients are not aware of other clients when sending write requests to database nodes. So it may not be clear which write happened first, and the **order is undefined**.

Even though these writes don't have a natural ordering, we can **enforce an arbitrary order**. For example, we can attach a **timestamp** to each write, and pick the biggest timestamp as the most recent, and discard any writes with an earlier timestamp. This conflict resolution algorith is called **last write wins**.

LWW achieves eventual convergence, but **at the cost of durability**: if there are several concurrent writes to the same key and all were reported successful to the client, only one of the writes will survive and the others will be **silently discarded**.

In some situations, such as caching, lost writes may be acceptable. If losing data is not acceptable, LWW is a poor choice for conflict resolution.

The only safe way of using a database with LWW is to ensure that a key is **only written once** and thereafter treated as **immutable**, thus **avoiding any concurrent updates** to the same key. For example, we can use a UUID as the key, thus giving each write operation a unique key.

### Happens-Before Relationship and Concurrency
An operation A **happens before** another operation B if any of the following applies:
- B knows about A.
- B depends on A.
- B builds upon A.

If one operation happened before another, the later operation should **overwrite** the earlier operation.

Two operations are **concurrent** if neither happens before the other.

### Capturing Happens-Before Relationships
A server can determine whether two operations are concurrent, or one happened before another as follows:
- Server maintains a **version number** for every key:
    - **Increments** it every time **key is written**
    - **Stores** new version number along with value written.
- When a client reads a key:
    - Server returns **all values** not overwritten, as well as latest version number.
- When a client writes a key:
    - Include version number from the prior read. 
    - **Merge** together values that it received from prior read.
- When server receives a write with a particular version number:
    - **Overwrite** all values with that version number or below (these have been merged into a new value).
    - **Keep** all values with a higher version number (these are concurrent with the incoming write).

When a write includes the version number from a prior read, that tells us which previous state the write is based on. If you make a write without including a version number, it is concurrent with all other writes, so it will not overwite anything - it will just be returned as one of the values on subsequent reads.

### Merging Concurrently Written Values
This algorithms ensures **no data is silently dropped**, but requires **clients to do extra work**: if several operations happen concurrenly, clients have to clean up afterward by merging the concurrently written values. Riak calls these concurrent values **siblings**.

Merging sibling values is essentially the same problem as **conflict resolution** in **multi-leader replication**. Various approaches include:
- **Last write wins**: pick one of the values based on version number or timestamp; but this implies **losing data**.
- **Union of values**: simply combine the values from each concurrent write. However this is made complex when **removing items**. If we merge 2 sibling records and an item has been only removed in 1 of them, then the removed item will **reappear** in the union of the siblings. To prevent this, the system must leave a marker with an appropriate versino number to indicate that the item has been removed when merging siblings. This is known as a **tombstone**.

As merging siblings in application code is complex and error-prone, there are efforts in designing **data structures** that automatically merge siblings in sensible ways.

### Version Vectors
When there are multiple replicas, we need a version number **per replica** as well as per key. Each replica increments its **own version number** when processing a write, and also keeps track of the version number it has seen from each of the other replicas.

The collection of version numbers from all replicas is called a **version vector**. Like regular version numbers, version vectors are sent from the database replicas to clients when values are read, and need to be sent back to the database when a value is subsequently written. The version vector allows the database to distinguish between overwrites and concurrent writes.

The version vector ensures that it is safe to read from one replica and subsequently write back to another replica. Doing so may result in siblings being created, but **no data is lost** as long as siblings are merged correctly.
