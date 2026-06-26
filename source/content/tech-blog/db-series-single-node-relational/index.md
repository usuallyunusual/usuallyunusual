---
title : 1a - How a single node actually works?
date : 2026-06-24
summary : "How a single node DB works like from page structure, on-disk storage, indexing, WAL."
tags : ["database", "data-engineering", "softwareengineering"]
math: true
---
{{< katex >}}
This is part-1 of a series walking through how databases work and the structures and processes that power them. 
It helps understand the different concepts and the problem they solve intuitively instead of just learning the feature.
Let's talk about a single node relational database.

## Single Node Relational DB
Intuitively for any database we need 2 things
- A place to store data (Where and how we store the actual data)
- A lookup to find specific records (The indexing which makes database queries actually fast)

We're ignoring query-planning and query-execution strategy, maybe I'll retroactively add those at a later time.

Let's first start with indexing.
## Indexing
An index is what offers quick lookups to keys in the database. You can't possibly look through all the data to find 
the record you're looking for, that would provide abysmal performance because the process would need to read
every file into memory and scan the entire thing. Indexes solve this problem by providing an efficient way to find records
on the table.

### How?

Let's take the binary search trees which have 2 child-nodes per parent which is very efficient for searches.
For a million records the height of the BST would be the \$ log_2(1000000) \$ which is 20.
So we've reduced the lookup complexity from scanning the entire data to scanning 20 nodes.
Every lookup would do 20 node visits. On disk where seek takes `ms` this could be catastrophic.

The bottleneck here is that the binary search tree has a branching factor of 2 which forces the tree to 
have a greater height, which, on-disk, translates to seeks.

### Enter B-Trees.

B-trees unlike BST's are allowed to have multiple keys (compared to one) and multiple branches (unlike 2 in BST's).
There are some invariants which ensure that the lookup is correct and efficient.
- Every node except the root must contain `[m/2]-1` keys which ensures 50% fill rate.
- A node with `k` keys contains `k+1` children
- All keys within a node are in sorted order : ascending. This makes binary search valid in the sub-tree.
- All leaf nodes are at the same depth.

All data is stored in the leaf nodes. 

#### Search
The search logic is similar to a BST, from thr root node traverse the tree based on whether the key you are looking 
for is greater than or lesser than the key in the node until you find the right key or hit a leaf node.

#### Insertion
Insertion would either add a key to an existing leaf with room. If there's no room, split the leaf and push the median
upwards to the parent. This might cause cascading splits upwards into the root. Which is what ensures the tree is balanced.

#### Deletion
Deletion is the most complex operation. There are three main cases:
- If the key is in a leaf with more than t-1 keys, simply remove it.
- If the key is in an internal node, replace it with either its in-order predecessor or successor (which is always in a leaf), then delete from the leaf. This reduces the problem to leaf deletion.
- If the leaf would fall below the minimum fill after deletion, you must rebalance.
  - First try to borrow a key from an adjacent sibling via the parent (a rotation). 
  - If both siblings are at minimum fill, merge the node with a sibling, pulling the 
  separator key down from the parent. The merge reduces the parent's key count by one, which may cascade upward.

Essentially B-Trees are short and fat trees which avoid as much disk IO. The height becomes %log_t(n)$ which becomes very small.

### B+ tree: the variant that actually runs your database
The B+ tree modifies one rule: key-value pairs are stored only in leaf nodes. Non-leaf nodes store only keys and child pointers.
This has profound consequences, higher fanout. Internal nodes, stripped of data payloads, can pack more keys per page. 
A page that previously held 100 full records might now hold 500 keys, reducing tree height.

This traversal assumes a single reader or writer.
Concurrent access introduces its own complexity, which we'll get to in part 1c.

### Linked leaf scan. 
All data is stored in leaf nodes, which are linked for fast range scans. 
Once you find the start of a range, you walk the leaf linked list without ever touching the internal nodes again. 
This makes range queries extremely efficient. 
In a plain B-tree, each key in the range might require a separate traversal from the root. 

Because all values live in leaves, every key in an internal node is also duplicated in a leaf.
This is a minor storage cost but acceptable given the benefits.

In practice, 99% of database management systems use B+ trees for indexing. 
This is because the B+ tree holds no data in the internal nodes. 
This maximizes the number of keys stored in a node thereby minimizing the number of levels needed in a tree. 
Smaller tree depth invariably means faster search.

## But where's the data stored?

### Page layout: how a row lives inside a page

In the simplest model, the database maintains a heap file for each table.
This is essentially an unordered collection of fixed-size pages (PostgreSQL uses 8KB pages by default).

This is not an arbitrary size, the database operates on entire pages, which means if the page size of the underlying OS 
does not match with the page size of the DB, it will cause the DB to spin off many more 
IO operations per update (The read-modify-write amplification). This causes disk wear out.

Records are written to pages roughly in insertion order. 
When you insert a row, the database finds a page with enough free space and writes the tuple there.

We might also have ClusteredIndexes which are basically BTrees which store entire records in the index itself,
so you avoid the second disk-IO, which is faster than the heap file approach.This makes range scans extremely efficient since the
order is maintained by the index and data is colocated with it on-top of which there is no second IO.

Regardless of whether it's a heap page or a B+ tree leaf page, the internal structure of a page is similar. 
A page is a fixed-size block (4KB, 8KB, 16KB are common). 
It contains:
- A page header: page type, free space pointer, number of tuples, LSN (log sequence number for recovery).
- A slot array (also called item pointer array): a compact array at the start of the page,
where each slot is a small record containing an offset + length pointing to a tuple within the page.
Slots are fixed-size and grow from the beginning of the page; tuple data grows from the end. 
Free space is in the middle. This indirection layer means tuples can be moved within a page (for compaction)
without updating the index, only the slot pointer changes, and the TID (page + slot number) stays stable.
- Tuple data: the actual row. Variable-length fields like TEXT or BYTEA may be stored inline if small, 
or TOASTed out-of-line (in PostgreSQL's terminology) if large, stored in a separate TOAST table and referenced by pointer 
from the main tuple. This is how databases handle rows that are logically large but physically decomposed.
- Null bitmaps: a compact bitmap at the tuple header indicating which columns are NULL, avoiding the need to 
store any bytes for NULL values.

Data is stored as tuples and collected into pages. Tuples are not allowed to span different pages.
Pages are usually 8kB and are configurable. These pages are located in a heap-file which postgres maintains.
 
> Read more about TOAT here https://www.postgresql.org/docs/current/storage-toast.html

So given an index (B-Tree or such) which provides us a tupleID (Which should contain which page and offset the 
tuple lives in) we can get the datum in an efficient manner. 
- Traverse index to find the key
- Read TID from leaf
- Fetch the correct page
- Fetch the right tuple ID

Step 3 and 4 become the heap fetch.
The heap is unordered with respect to columns and the index is what provides order. 
Which implies that range queries do random IO on disk since the order is defined in the index and the index may point
to random pages.

The flip side is also worth noting, sequential scans (no index) are sometimes faster for large-range queries 
on cold data precisely because sequential reads coalesce into larger OS read-aheads.

The escape hatch is covering indexes (Postgres calls them index-only scans). 
If all columns needed by a query are present in the index, the heap fetch is skipped entirely

Secondary indexes store the primary-key value of the row and retrieval looks like
- Traverse secondary-index for the key
- Read the primary-key of row
- Traverse primary-index for the primary-key
- Retrieve row data.

Basically traversing two indexes. Secondary indexes using Clustered Indexes become slower than the single hop heap fetch 
that is used in postgres.

The practical implication: if your primary key is a UUID (random 128-bit value),
new rows insert into random positions in the clustered B+ tree (because keys are sorted and UUID's are random),
causing page splits (which could potentially affect the entire tree and cause more disk wear),
write amplification, and poor cache locality. 

Sequential integer primary keys (auto-increment or ULIDs/UUIDv7)
insert at the rightmost leaf, which is much more efficient. This is why "don't use UUID as primary key in 
InnoDB" is a concrete, grounded recommendation, not a style opinion.

Every insert or update in a B+ tree is potentially a read-modify-write on a page 
(read the 8KB page into buffer pool, modify it, mark it dirty, eventually flush it). 
For write-heavy workloads, this is expensive. LSM trees invert the trade-off, 
which is the subject of part 1b.

## Write-Ahead Logging (WAL) and the relationship to disk
Before any change reaches a data page on disk, it's written to the WAL (also called the redo log). 
The WAL is an append-only sequential file, the cheapest possible write pattern. Data pages are dirty in the
buffer pool (RAM) and are flushed to disk lazily. If the process crashes, the WAL is replayed to reconstruct 
any changes that hadn't been flushed.

This means the on-disk representation of a table at any given moment may be slightly stale,
some committed changes are only in the WAL, not yet in the heap/clustered index pages. 
The storage engine handles this transparently. The WAL is also what enables replication,
ship the WAL to a standby, replay it there, and the standby is consistent.

## On WAL and fsync. 
The WAL provides durability only if it actually reaches stable storage before the ack goes back to the client.
Postgres calls fsync() on WAL writes before confirming a commit. If fsync is disabled, crashes can corrupt data
even with WAL in place. The write was acknowledged but never committed from the OS page cache. 
Never disable fsync in production.

### What's next?
ACID guarantees and how WAL, MVCC and transactions compose to provide these guarantees will be covered in another post.

