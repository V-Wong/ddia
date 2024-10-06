# Rebalancing Partitions
## Overview
In a database, over time:
- **Query throughput increases**: requiring more CPUs.
- **Dataset size increases**: requiring more disks and RAM.
- **Machine fails**: requiring other machines to take responsibility.

These require data and requests to be **moved across nodes**, a process known as **rebalancing**.

Minimum requirements for rebalancing include:
- Load should be **shared fairly between nodes** after rebalancing.
- Database should **continue accepting reads and writes** during rebalancing.
- Data moved between nodes should be **minimal**.

## Strategies for Rebalancing
### Hash Modulo N (Bad Idea)
We can assign partitions to nodes with `hash(key) % N` where `N` is the number of nodes. This however requires **most keys to move** when `N` changes, making rebalancing very expensive.

### Fixed Number of Partitions
We can create **more partitions than nodes**, and assign several partitions to each node. We can then rebalance as follows:
- **Node is added to cluster**: node can **steal** a few partitions from every existing node, until partitions are fairly distributed again. 
- **Node is removed from cluster**: node **gives** all its partitions to existing nodes.

Only **entire partitions** are moved between nodes. The number of partitions does not change, nor does assignment of keys to partitions.

The number of partitions is usually **fixed** on database setup. It is configured to the **maximum number of nodes** we can have, so must be chosen high enough to accomodate future growth.

Choosing the right number of partitions is difficult:
- **Too few partitions**: partitions grow too large, making rebalancing and recovery from node failures very expensive.
- **Too many partitions**: too much management overhead.

## Dynamic Partitioning
Key range-partitioned databases create partitions **dynamically**:
- **Partitions grows beyond configured size**: partition is split in two, so that approximately half the data ends up in each half.
- **Partition shrinks below configured threshold**: partition is merged with adjacent partition.

Each partition is assigned to one node, and each node can have multiple partitions. After a partition has been split, one half can be transferred to another node.

An advantage of dynamic partitioning is that the number of partitions **adapts to the total data volume**:
- **Small amount of data**: few partitions, overhead is small.
- **Large amount of data**: size of each partition is limited to a configurable maximum.

Dynamic partitioning is also suitable for hash-partitioned data.

### Partitioning Proportionally to Nodes
Recall that:
- **Dynamic partitioning**: number of partitions is proportional to size of datset.
- **Fixed partitioning**: size of partitions is proportional to size of data set.

Another option is the number of partitions **proportional to number of nodes** - having a **fixed number of partitions per node**. In this case:
- **Number of nodes unchanged**: size of each partition grows proportionally to dataset size.
- **Number of nodes increased**: size of each partition shrinks.

Since a larger data volume generally requires more nodes, this approach keeps the **size of each partition stable**.

When a new node joins the cluster, it randomly chooses a fixed number of existing partitions to split, and takes ownership of one half of each split partition, while leaving the other half in place.

### Operations: Automation or Manual Rebalancing
There is a gradient between **fully automatic rebalancing** and **fully manual rebalancing**. Some databases generate a suggested partition assignment automatically, but require an administrator to commit it before it takes effect.

Fully automated rebalancing can be convenient, but it can be **unpredictable**. If not done carefully, this process can overload the network or the nodes and harm the performance of other requests while rebalancing is in progress.

Thus, it is often good to have a **human in the loop** for rebalancing.
