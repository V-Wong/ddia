# Request Routing
## Overview
**Request routing** concerns **which node(s)** clients connect to when making a request. This is part of the more general problem of **service discovery**, which affects any software accessible over a network.

Different approaches to this problem influence how clients make requests:
- Contact **any node** (e.g. via round-robin load balancer). If that node owns the partition, it can handle the request directly; otherwise, it forwards the request to the appropriate node, receives the reply, and passes the reply to the client.
- Send all requests to a **routing tier** first, which determines the node that should handle each request and forwards it accordingly.
- Require that clients be **aware of the partitioning and assignment of partitions to nodes**. The client can then directly connect to the appropriate node.

The key problem in all approaches is: how does the component making routing decisions **learn about changes in the assignment of partitions to nodes**.

## Coordination Service
Many distributed data sytems rely on a **separate coordination service** such as **ZooKeeper** to keep track of cluster metadata. Each node registers itself in ZooKeeper, and ZooKeeper maintains the **authoritative mapping** of partitions to nodes. 

The routing tier or partitioning-aware client can subscribe to this information in ZooKeeper. When a partition changes ownership, or a node is added or removed, ZooKeeper notifes the routing tier so that it can keep its routing information up to date.

## Gossip Protocol
Cassandra and Riak use a **gossip protocol** among the nodes to disseiminate any changes in cluster state. Requests can be sent to any node, and that node forwards them to the approriate node for the requested partition. This model puts **more complexity on the database nodes** but **avoids the dependency on an external coordination service**.

## Parallel Query Execution
**Massively parallel processing (MPP)** relational databases are often used for **analytics**, and are much more sophisticated in the queries supported. The MPP query optimizer can break complex queries into a number of execution stages and partitions, many of which can be executed in parallel on different nodes of the database cluster.
