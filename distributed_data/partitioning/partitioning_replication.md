# Partitioning and Replication
## Overview
Partitioning is usually **combined** with **replication** so that **copies of each partition** are stored on **multiple nodes**. This means that, even though **each record belongs to exactly one partition**, it may still be stored on several different nodes for **fault tolerance**.

A node may store **more than one partition**. If a leader-follower replication model is used: each partition's leader is assigned to one node, and its followers are assigned to other nodes. Each node may be the leader for some partitions and a follower for other partitions.

The choice of partitioning scheme is mostly **independent** of the choice of replication scheme.
