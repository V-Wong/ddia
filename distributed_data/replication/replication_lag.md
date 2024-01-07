# Replication Lag
## Overview
An application reading from an asynchronous follower may see **outdated information** if the follower has fallen behind the leader. This leads to **inconsistencies in the database**: running the same query on the leader and follower at the same time may **produce different results**. This inconsistency is a **temporary state** - if we stop writing and wait for a while, the followers will eventually catch up and become consistent with the leader. This effect is known as **eventual consistency**.

The delay between a write happening on the leader and being reflected on a follower is known as **replication lag**. This may range from less than a second to several minutes if the system is near capacity or if there is a networking issue. When this lag is large, the inconsistencies creates real **problems for applications**.

## Reading Your Own Writes
If a user views the data shortly after making a write, the read may be from a replica, but the write may not yet have reached it yet. To the user, it looks as though the submitted data was **lost**.

Here, we need **read-after-write consistency**. This guarantees a user will always see the updates they **submitted themselves**. It makes no promises about **other users**: other users' updates may not be visible until some later time.

There are various techniques to implement this:
- Read from the **leader** for any data the user may have **modified**. This requires knowing whether something might have been modified, without actually querying it. For example, user profile information on a social network is normally only editable by the owner of the profile. Thus, a simple rule is: always read the user's own profile from the leader.
- If most things are editable by the user, most reads would go to the leader, negating benefits of read scaling. **Other criteria** may be used to decide whether to read from the leader. For example, we could track the **time of last update** and, for some period after the last update, make all reads from the leader.
    - We could also monitor the replication log on followers and **prevent queries on any follower** that has fallen significantly behind the leader.
- The client can **remember the timestamp** of its **most recent write** - then the system can ensure that the replica serving any reads for that user reflects updates until that timestamp. If a replica is not sufficiently up to date, the read can be handled by **another replica** or the query can **wait** until the replica has caught up.

There are various complexities as well:
- If the replicas are geographically distributed, then any request that needs to be served by the leader must be **routed** to the datacenter that contains the leader.
- If the same user is accessing the service from **multiple devices**, we may want **cross-device read-after-write consistency**. This raises serveral additional issues to consider:
    - Timestamp metadata will need to be centralised as the code running on one device doesn't know what updates have happened on other devices.
    - If we require reading from the leader, we must first route requests from all of a user's devices to the same datacenter.

## Monotonic Reads
If a user makes a query to a follower with some lag, then to another follower with greater lag, the user may see data in the first query but not in the second query. To the user, it looks as though the system has **moved back in time**.

Here, we need **monotonic read consistency**. This guarantees that if one user makes several reads in sequence, they will not see time go backwards - i.e. they will not read older data after previously having read newer data.

One way of achieving monotonic reads it to make sure that each user always makes their reads from the **same replica** (different users can read from different replicas). For example, the replica can be chosen based on a hash of the user ID. However, if that replica fails, the user's queries will need to be rerouted to another replica.

## Consistent Prefix Reads
In a **partitioned database**, different partitions operate **independently** so there is **no global ordering of writes**: when a user reads from the database, they may see some parts of the database in an older state and some in a newer state.

Here, we need **consistent prefix reads consistency**. This guarantees that if a sequence of writes happens in a **certain order**, then anyone reading those writes will see them in the **same order**.
    - Note: monotonic reads applies to a single data item, while consistent prefix reads applies to a sequence of writes.

One solution to make any **causally related** writes to the **same parition**. But this can't always be done efficiently.
