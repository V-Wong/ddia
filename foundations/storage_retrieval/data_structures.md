# Data Structures That Power Your Database
## Logs
A log is an **append-only sequence of records**. In a log, old versions of values are not overwritten, instead searches look for the **last occurence of a key**. The performance characteristics are:
- **Fast sets**: append is relatively efficiently.
- **Slow gets**: every get requires an O(n) search through the database.

## Segmentation
To save space, we break a log into **segments** of a certain size, close a segment file when it reaches a certain size, and make subsequent writes to a new segment file. These segments are periodically **compacted** which **deduplicate keys**, and potentially **merges segments together**.

## Indexes
Indexes are **additional metadata** that **speed-up searches** by signposting the desired data. They are **derived from the primary data** and do not affect the contents of the database; only affecting the **performance** of queries. 

Every index **slows down writes**, because the index must also be updated on each write. It is up to the developer to choose appropriate indexes for their application.

## Hash Indexed Log Databases
The simplest index is an **in-memory hash map** where every key is **mapped to a byte offset** in its segment.
- On each **write (append)**: update hash map to reflect offset of the data just added.
- On each **read**: use hash map to find offset in the segment, seek to that location, and read the value. If key is not present, check the hashmap of the second-most-recent segment, and so on.

### Analysis
Hash indexed log databases have various advantages:
- Appending and segment merging are **sequential writes**, which are much **faster than random writes**.
- Concurrency and crash recovery are simple if segments files are append-only.
- Merging old segments avoids data files getting fragmented over time.

However it also has limitations:
- Hash map must fit in memory; can't handle large number of keys.
- Range queries are not efficient.

## Log Structured Merge Trees (LSM Trees)
### Sorted String Tables (SSTables)
A sorted string table is a segment file **sorted by key**. This has several advantages over log segments with hash indexes:
- Can use merge sort to **efficiently merge segments**.
- Only need **sparse index of some keys** as we can search in offset ranges between keys in the index.

### Developing a Storage Engine
LSM Trees uses a cascade of SSTables to build a **storage engine algorithm**:
- When write comes in: add to **in-memory** balanced tree data structure (e.g. red-black tree).
    - This in-memory tree is often called a **memtable**.
- When memtable size exceeds some threshold, write it out to disk as an SSTable file. The new SSTable file becomes the most recent segment of the database and a new empty memtable is created for use.
- To serve a read request, try to find the key in the memtable, then in the most recent on-disk segment, then the next-older segment, and so on.
- Periodically run merging and compaction to combine SSTable files and discard overwritten or deleted values.

If the database **crashes**, the **most recent writes are lost**. This is resolved by keeping a separate **log on disk** to which every write is immediately appended. This log is not sorted and is only used to restore the memtable after a crash.

### Performance Optimisations
#### Bloom Filters
LSM Trees can be slow when looking up **non-existent keys** as we have to check all segments. This can be optimised with **bloom filters**:
- Memory efficient probabalistic data structure that approximates the contents of a set.
- Membership tests can have false positives, but no false negatives.

Bloom filters are also called [signatures](https://vwong.dev/notes/COMP9315/indexing/signature_based_indexing.html).

#### Timing and Ordering of Compaction/Merging
There are 2 main strategies for compaction and merging:
- **Size-tiered**: newer and smaller SSTables are successively merged into older and larger SSTables.
- **Leveled**: key range is split up into smaller SStables and older data is moved into separate levels, which allows the compaction to proceed more incrementally and use less disk space.

## B-Trees
B-trees are the most **widely used** indexing structures in almost all relational databases and many non-relational databases. The basic operations are already covered in [COMP9315 notes](https://vwong.dev/notes/COMP9315/indexing/btrees.html).

### Making B-Trees Reliable
Writes in a B-tree **overwrites** a page on disk with new data. It is assumed that the overwrite does not change the location of the page; i.e., all references to that page remain intact when the page is overwritten.

Some B-tree operations require **multiple pages to be overwritten**. This is dangerous because crashes can lead to a **corrupted index**. To resolve this, B-tree implementations typically have a **write ahead log (WAL)** to which every B-tree modification must be written to before it can be applied to the pages of the tree itself.

Careful attention also needs to be placed if **multiple threads** are accessing the B-tree at the same time - otherwise a thread may see the tree in an **inconsistent state**. This is typically resolved using **latches (lightweight locks)**.
    - Log-structured approaches are simpler in this regard, as they do all merging in the background without interfering with incoming queries and atomically swap old segments for new segments from time to time.

### B-Tree Optimisations
B-trees have been optimised heavily over the years:
- Instead of overwriting pages and maintaining a WAL for crash recovery, some databses use a **copy-on-write scheme**. A modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location.
- Can save space by storing **abbreviations of keys** since we only need enough information to act as boundaries between key ranges.
- Many B-tree implementations try to lay out the tree so that leaf pages appear in sequential order on disk. However, it's difficult to maintain that order as the tree grows.
    - In contrast, LSM-trees rewrite large segments of the storage in one go during merging, so it is easier to keep sequential keys close to each other on disk.
- Additional pointers can be added to the tree. For example, each leaf page may have references to its sibling pages to the left and right, which allows scanning keys in order without jumping back to parent pages.

### Comparing B-Trees and LSM-Trees
The rule of thumb is:
- LSM-trees are **faster** for **writes**.
    - Typically have lower **write amplification** compared to B-trees.
        - Write amplification is the phenomena where 1 logical write to the database results in multiple physical writes to disk.
    - All SSTable writes are sequential.
    - Low fragmentation as SStables are not page-oriented and are periodically compacted to remove fragmentation.
- B-trees are **faster** for **reads**.
- B-trees have **simCOMP0pler transaction semantics**.
    - Each key exists in exactly one place in the index.

## Other Indexing Structures
### Secondary Indexes
In a secondary index, **keys are not unique**, and there may be multiple rows with the same key. This can be solved either by:
- Making each value in the index a list of matching row IDs.
- Making each key unique be appending a row identifier to it.

Either way, B-trees and LSM-trees can be used as secondary indexes.

### Storing Values Within Indexes
The value in an index can hold either:
- A **reference** to the row stored elsewhere.
    - The place where rows are stored is known as a **heap file**, and it stores data in no particular order (may be append-only, or may keep track of deleted rows in order to overwrite them with new data later). This **avoids duplicating data** when multiple secondary indexes are present.
    - Updating values without changing the key in a heap file is efficient if the data can be overwritten in place. If the updated data needs to be moved, then all indexes must be updated to point to the new heap location.
- The entire **row data** itself.
    - This is known as a **clustered index** and allows all queries to be answered using the index alone. 
    - This is most efficient for reads, but can incur penalties for writes, as the data is duplicated and must be updated multiple times.
    - Transactional guarantees are also more complex to maintain.
- Some of the row data.
    - This is known as a **covering index** and allows some queries to be answered using the index alone.
    - This is a compromise between clustered and non-clustered indexes.

### Multi-Column Indexes
Already covered in [COMP9315 notes](https://vwong.dev/notes/COMP9315/indexing/multi_dimensional_search_trees.html).

### Full-Text Search and Fuzzy Indexes
Full-text serach engines allow a search for a word to be expanded to include:
- Synonyms.
- Ignore grammatical variations.
- Search for ocurrences of words near each other in the same document.
- Support various linguistic analysis of the text.

In Lucene, the in-memory database is a **finite state automaton** over the characters in the keys. This automaton can be transformed into a **Levenshtein automaton**, which supports efficient search for words within a given edit distance.

### In-Memory Databases
#### RAM vs Disk
The data structures discussed so far have all been answers to the limitations of disks. As RAM becomes cheaper, it is feasible to **keep datasets entirely in memory** which is much faster than disk. These **in-memory databases** are faster than traditional databases because they don't need to read from disk and they avoid the overhead of encoding in-memory data structures in a form that can be written to disk.

#### Durability
Some in-memory key-value stores, such as Memcached, are only intended for caching, where it's acceptable for data to be lost if a machine is restarted. Other in-memory databases aim for **durability** by any of the following:
- Writing a log of changes to disk.
- Writing periodic snapshots to disk.
- Replicating the in-memory state to other machines.

When an in-memory database is restarted, it needs to reload its state, either from disk or over the network from a replica. These in-memory databases only use the disk as an append-only log for durability, and reads are served entirely from memory.

#### Other Advantages
Some in-memory databases provide data models that are difficult to implement with disk-based indexes. For example, Redis offers a database-like interface to various data structures such as sets and priority querues.

#### Larger Datasets
In-memory databases can be extended to support datasets **larger than the available memory** by **evicting least recently used data** from RAM to disk when there is not enough memory, and loading it back into memory when it is accessed again in the future. This is similiar to what OSs do with **virtual memory**, but the database can manage memory more efficiently, as it can work at the granularity of individual records rather than entire memory pages.
