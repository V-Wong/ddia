# Multi-Leader Replication
## Overview
Leader-based replication has a major downside: if we can't connect to the leader, we can't write to the database. Thus, a natural extension is allow **more than one node** to accept writes. These nodes that process writes still must forward that data change to all other nodes. This configuration is called **multi-leader replication**. In this setup, each leader simultaneously acts as a **follower to the other leaders**.

## Use Cases for Multi-Leader Replication
### Multi Datacenter Operation
If we have **multiple datacenters**, we can have a leader in **each datacenter**. Within each datacenter, regular leader-follower replication is used; between datacenters, each datacenter's leader replicates its changes to the leaders in other datacenters.

Here is how single-leader and multi-leader configurations fare in a multi-datacenter deployment:
- **Performance**:
    - Single-leader: every write must go over the internet to the datacenter with the leader. This can add **significant latency** to writes.
    - Multi-leader: every write can be processed in the local datacenter and is replicated asynchronously to the other datacenters. This may mean **better perceived performance** for end users.
- **Tolerance of datacenter outages**:
    - Single-leader: if the datacenter with the leader fails, failover can promote a follower in another datacenter to be leader.
    - Multi-leader: each datacenter can continue operating independently of others, and replication catches up when the failed datacenter comes back online.
- **Tolerance of network problems**:
    - Single-leader: very sensitive to network problems because writes are made **synchronously** over inter-datacenter links.
    - Multi-leader: **asynchronous** replication can tolerate network problems better - temporary network interruption does not prevent writes being processed.

Although multi-leader replication has advantages, it has a big downside: the same data may be **concurrently modified** in two different datacenters, and those **write conflicts** must be **resolved**.

### Clients With Offline Operation
Multi-leader replication is also appropriate if an application needs to continue working while **disconnected from the internet**.

For example, consider calendar apps where we can see and create meetings at any time. If we make changes while offline, they need to be **synced** with a server and other devices when our device is next online. In this case, every device has a **local database** that acts as a **leader**, and there is an **asynchronous multi-leader replication** process (sync) between the replicas of our calendar on all devices.

From an architectural POV, this is essentially multi-leader replication between datacenters, where each device is a "datacenter" and the network connection between them is extremely unreliable.

### Collaborative Editing
Real-time collaborative editing applications allow several people to **edit a document simultaneously**, e.g. Google Docs. This has a lot in common with the offline operation problem. When one user edits a document, the changes are instantly applied to their **local replica** and **asynchronously replicated** to the server and any other users who are editing the same document.

If we want to guarantee no **editing conflicts**, the application must **obtain a lock** on the document before a user can edit it. If another user wants to edit the same document, they first have to **wait** until the first user has committed the changes and **released the lock**. This collaboration model is equivalent to single-leader replication with transactions on the leader.

However, for faster collaboration, we may want to make the **unit of change** very small (e.g. a single keystroke) and avoid locking. This allows multiple users to **edit simultaneously**, but also brings the challenges of multi-leader replication, including requiring conflict resolution.

## Handling Write Conflicts
The biggest problem with multi-leader replication is that **write conflicts** can occur, which means **conflict resolution** is required.

Consider the following example:
- A wiki page is **simultaneously** being edited by two users.
- User 1 changes the title from "A" to "B".
- User 2 changes the title from "B" to "C".

Here, each user's change is successfully applied to their local leader. However, when the changes are asynchronously replicated, a **conflict is detected**. This problem does not occur in a single-leader database.

### Synchronous vs Asynchronous Conflict Detection
In a single-leader database, the second write will either:
- Block and wait for the first write to complete.
- Abort the second write transaction, forcing the user to retry the write.

OTOH, in a multi-leader setup, both writes are successful, and the conflict is only detected asynchronously later on. At that time, it may be too late to ask the user to resolve the conflict.

We could make the conflict detection synchronous by waiting for the write to be replicated to all replicas before signifying success to the user. But then we would los the ability for each replica to accept writes independently; we might as well just use single-leader replication.

### Conflict Avoidance
The simplest strategy for dealing with conflicts is to avoid them: if the application can ensure all writes for a particular record go through the same leader, then conflicts cannot occur. 

For example, in an application where a user can edit their **own data**, we can ensure that requests from a particular user are always routed to the **same datacenter** and use the leader in that datacenter for reading and writing. Different users may have different "home" datacenters, but from any one user's POV, the configuration is essentially single leader.

However, sometimes we may want to **change the designated leader** for a record - perhaps because one datacenter has failed or the user has moved geographically. Here, conflict avoidance breaks down, and we have to deal with the possibility of concurrent writes on different leaders.

### Converging Towards a Consistent State
A single-leader database applies writes in a **sequential order**: if there are several updates to the same field, the last write determines the final value of the field. In a multi-leader configuration, there is **no defined ordering** of writes, so it's not clear what the final value should be.
