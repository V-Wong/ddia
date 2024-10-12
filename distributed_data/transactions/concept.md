# The Slippery Concept of a Transaction
## Overview
Most relational databases support transactions. In the late 2000s, NoSQL databases arised and **sacrificed transactions** to offer **better scalability**.

Transactions have advantages and limitations. To understand these trade-offs, we must understand the details of the **guarantees** that transactions can provide.

## The Meaning of ACID
ACID describes the **safety guarantees** provided by transactions. These transactions typically involve modifying **several objects** at once.

### Atomicity
Transactions are often composed of **multiple statements**. Atomicity guarantees that each transaction is treated as a single **unit**, which either succeeds or fails completely. If any statement in the transction fails, the entire transction fails, and the database is **unchanged**.

### Consistency
Consistency ensures that a transaction brings the database **from one consistent state to another**, preserving all **invariants** (e.g. referential integrity).

Note that consistency depends on the **application's notion of invariants**. Not all invariants can be enforced at the database level. It is often the **application's responsibility** to define transactions correctly so that they preserve consistency.

### Isolation
Transactions guarantee that concurrently executing transactions are isolated from each other: they **cannot step on each other's toes**.

Isolation is often formalized as **serializability**: each transaction can pretend that it is the **only transaction running**, and that when all transactions are committed, the result is the same as if they ran **serially (one after another)**. 

However, serializable isolation is **rarely used in practice**, because it carries a **performance penalty**.

### Durability
Durability guarantees that once a transaction has been committed, it will **remain committed** even in the case of a system failure. This has different meanings across databases:
- **Single-node database**: typically means that the data has been written to **non-volatile storage**. It usually also involves a **write-ahead log** which allows **recovery** in the event of disk corruption.
- **Replicated database**: may mean that the data has been successfully **copied to some number of nodes**. 

In order to provide a durability guarantee, a database must **wait** until writes or replications are complete before reporting a transaction as successfully committed.

## Single-Object and Multi-Object Operations
### Single-Object Writes
Atomicity and isolation also apply when a **single object** is being changed (on a single node). For example, if a large document is being written to the database:
- If a network connection is interrupted after one half is sent, does the database store the unparseable first half?
- If the power fails while the database is overwriting the previous value, are the old and new values spliced together?
- If another client reads the document while the write is in progress, will it see a partially updated value?

Atomicity can be implemented using a **log for crash recovery**, and isolation can be implemented using a **lock on each object**.

### The Need for Multi-Object Transactions
Many distributed data stores have **abandoned multi-object transactions** because they are **difficult to implement across partitions**, and they can conflict with **high availability** or **performance requirements**.

There are some cases where single-object inserts, updates and deletes are sufficient. However, in most cases, writes to several different objects need to be coordinated:
- **Relational databases**: multi-object transactions ensure **foreign key references** remain valid when inserting several records that refer to one another.
- **Document databases**: fields that need to be updated together are often **within the same document (object)**, hence not needing multi-object transactions. However, document databases lacking joins also encourage **denormalization**. This often requires updating several related documents in one go, which makes multi-object transactions useful.
- **Databases with secondary indexes**: indexes need to be updated every time a value is changed. These indexes are **different database objects**, and hence also benefit from multi-object transactions.

### Handling Errors and Aborts
A key feature of a transaction is that it can be **aborted and safely retried** if an error occurred.

ACID databases are based on this philosophy:
- If the database is in danger of violating its guarantee of atomicity, isolation, or durability, it would rather abandon the transaction entirely than allow it to remain half-finished**. 

In contrast, leaderless replication data stores work on a **best effort basis**:
- The database will do as much as it can, and if it runs into an error, it won't undo something it has already done - so it's the application's responsibility to recover from errors.

Although retrying an aborted transaction is a simple and effective error handling mechanism, it **isn't perfect**:
- If the transaction succeeded, but the server **failed to acknowledge** the successful commit to the client (e.g. network error), then retrying causes the transaction to be performed twice.
- If the error is due to **overload**, retrying will make the problem worse due to feedback cycles.
- It is only worth retrying after **transient errors** (e.g. deadlock, isolation violation, network interruptions, and failover); after a permanent error (e.g. constraint violation) a retry would be pointless.
- If the transaction also has **side effects** outside of the database, those side effects may happen even if the transaction is aborted.
- If the **client process fails** while retrying, any data it was trying to write to the database is lost.
