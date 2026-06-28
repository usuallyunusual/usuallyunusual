---
title : 1b - How a single node actually works - LSM tree based DB's.
date : 2026-06-26
summary : "How an LSM tree based single node DB works including compaction, MemTables, SSTables,tombstoning, read and write amplification"
tags : ["databases", "data-engineering", "software-engineering","database-internals"]
---
LSM trees (Log-Structured Merge trees) are a write-optimized storage structure. 
Rather than finding and updating records in place, all writes go to an in-memory buffer and 
are flushed to immutable sorted files on disk. Compaction periodically merges these files to keep 
reads manageable.

Let's work with a simple file and a process writing to it in append-only mode. Append-only implies 
you don't have to read the entire file into memory. You just have to write to the tail-end of the file.
This is your primitive storage engine. 

### Write-Path
The process is oblivious to the contents of the file, it just appends whatever you're asking of it. 
Implying excellent write throughput. 
This also implies you cannot update the record in the traditional sense, because update mean reading the file, finding the
right record and rewriting it in place. 
To overcome this we add it as a new record at the tail-end of the file. 
The fact that updates are written as new records necessitates the compaction process.

Before we go into that, let's quickly check out the read path.

## Read-Path 

When you read a particular record, the process needs to go through the entire file to find the right record with the ID
and then return it for you. 
The write-path (above) implies that the file has to be read from the bottom to ensure that the latest version of the record
is fetched since updated records live in the bottom.
The last record for a particular key is the most updated version of the record (First if we're going bottom up).

Read throughput is not great since we have no way of knowing whether a record even exists.

## Memtables/SSTables
The fundamental problem here is an indexing problem. We don't have a structure so we have no choice 
but to scan everything.

The fix is two-fold, keep recent writes in memory and occasionally flush sorted data to disk.

### Memtable
The memtable is an in-memory data structure. Typically a red-black tree or a skip-list (See appendix)
The memtable needs to:

- Accept writes in any order but maintain sorted order (for the eventual SSTable flush, which requires sorted keys).
- Support point lookups.
- Be iterable in sorted order at flush time.

Both structures give you all three. The choice between them comes down to your concurrency model. 
Single-threaded or coarse-grained locking, red-black tree is fine. 
High-concurrency write path, skip list wins because you can get away with finer-grained 
or lock-free synchronization.

### Frozen Memtable
When the active memtable crosses a particular size threshold, you can't just stop accepting writes while you flush it to disk,
that would stall your entire write path, potentially for seconds.
So instead you "freeze" it. The memtable is marked read-only and a new empty memtable is immediately promoted to active.
Writes continue uninterrupted into the new memtable. The frozen one sits in memory, still serving reads, while a background thread 
flushes it to disk as an SSTable. Once the flush completes, the frozen memtable is dropped from memory.
"Frozen" just means "no more writes accepted, but still readable." It's a state transition, no data structure changes happen. 
At any point in time you might have one active memtable and one or two frozen ones in flight, 
depending on how fast flushes are completing relative to how fast new memtables are filling up.

### SSTable : (Sorted String Table)
The frozen memtable is flushed to disk as an SSTable which is a file where keys are stored 
in sorted order. Sorted order is the critical property here. It makes a few things possible:

- Binary search within a file. You can jump to roughly the right position using OS level disk seeks rather than scanning linearly.
- Sparse indexes. You don't need to index every key. Store one key per block (say, every 64KB), 
and for any lookup you binary-search the index, seek to the right block, then scan at most one block's worth 
of data. The index fits in memory and the bulk of data stays on disk.
- Efficient merges. Two sorted files can be merged in O(n) using the same algorithm as merge sort. 
This makes compaction cheap.
- Bloom filters. Since the SSTable is immutable after being written, you can build a bloom filter over its keys at flush time.
Before touching disk at all, the read path consults the bloom filter. 
If it says the key definitely doesn't exist in that file, you skip it entirely.

> Bloom filters have false positives but no false negatives, so this is safe.

### Sparse indexes
The SSTable is basically a large file on disk (Or many). The sparse index optimizes these reads by not reading the
entire file into memory, instead it reads only the relevant block.

As the SSTable is being written, once a block of data is written (say 64Kb), an entry is added into the index
with the first key and the starting byte-offset. This sparse index is small compared the large SSTable and the index is what is loaded
into memory and the byte-offset is used to read the right block. Since the SSTable is sorted 
(and naturally the index as well), finding the block which contains the key is a matter of binary searching the index.

One index per SSTable file. There could be many SSTables accumulated implying many indexes to read into memory,
which is another reason for compaction.

Now let's revisit the read path

## Revisited read path : 
Read now does the following : 
- Read active memtable
- Read any frozen memtables in flight
- Consult bloom-filter for each SSTable, newest to oldest, to check if key exists and short-circuit out.
- Read SSTable blocks that pass the bloom-filter using the sparse indexes for those SSTables.

Since SSTables are scanned newest to oldest, return on the first hit which would give us the latest version
of the key, which is the same semantics provided by our append-only log file approach. 

This is still more expensive than a B-tree point lookup on a cold key.
The more SSTable files accumulate, the more bloom filters and index lookups you pay per read. 
Which is another reason why compaction is so important.

## Revisited write-path
As discussed above, the write path would look like this
- Write into WAL
- Write into memtable (for inserts, updates and deletes)
- Eventually flushed into SSTable

#### Tombstoning (Deletes)
A delete in an LSM tree cannot work the way it does in a B-tree, In a B-tree you find the record and 
remove it in place. In an LSM tree the files are immutable, you can't reach into an SSTable and remove a key. 
The memtable is mutable, but even if the key lives there, deleting it from the memtable doesn't help 
because an older version of that key might already be sitting in an SSTable on disk. You'd return a ghost.
So instead of deleting, you write. A tombstone is a special write marker that says "this key is deleted." 
It has the same key as the record it's killing, but its value is a sentinel (a null, a flag byte, whatever the implementation chooses). 
It flows through the write path exactly like any other write which makes compaction even more critical.

## Compaction
SSTables accumulate. Every memtable flush produces a new one. 
Old versions of records pile up across multiple files. Without compaction, read latency degrades 
monotonically and disk usage balloons.
Compaction takes a set of SSTables, merges them (using that O(n log_2(k)) sorted merge for k files), 
and writes out a new, deduplicated SSTable. Superseded versions of keys are dropped. 
So are tombstones, the markers written when you delete a key, once the compaction is confident no older file contains that key.
The result: fewer but potentially larger files, clean(er) data, faster reads. 

At the cost of write amplification, 
bytes written to disk are significantly more than bytes written by the application, because the same data gets 
rewritten during each compaction round.
There are different strategies for which files to compact and when:

### Size-tiered compaction
Groups SSTables of similar sizes together. Cheap to run, but at any given time you may have multiple copies of the same 
key alive across tiers, which hurts read performance and temporarily spikes disk usage during a merge. Relatively lower write
amplification.

### Leveled compaction (used by RocksDB and LevelDB)
Organizes SSTables into levels with size caps. Each level is 10x larger than the one above.
Files at a given level are non-overlapping in key range, which means a read only needs to look at one SSTable per level.
Read amplification is bounded, but write amplification is higher because a file may be compacted repeatedly as it moves through levels.

### FIFO compaction 
Is rarely used outside time-series workloads, it just evicts old SSTables when total disk usage crosses a threshold. Simple but lossy.

### Time-Window Compaction (TWCS)
TWCS divides time into fixed buckets, one bucket per hour, or one per day, depending on your configuration. 
SSTables are assigned to a bucket based on the timestamp of the data they contain.
Within a bucket, it runs size-tiered compaction, merging SSTables of similar sizes together as they accumulate, 
exactly like STCS. But it never merges SSTables across bucket boundaries. 
An SSTable from the 2pm bucket never gets merged with an SSTable from the 3pm bucket.
Once a time bucket is old enough that no new data will arrive into it (the current time has moved past that window), 
that bucket is sealed. All remaining SSTables in it get compacted into one final SSTable for that window. 
Then it sits there untouched until TTL expiry.

> This only works if data arrives roughly in time-order. Out of order writes mess with compaction boundaries and break
> the model.

Useful for TTL workloads.

The right compaction strategy depends on your read/write ratio, your tolerance for write amplification, and whether your access pattern is mostly recent data (like time-series) or uniformly distributed.

### Tombstone compaction
Tombstones are where things get subtle. 
A tombstone consumes space and costs read latency for as long as it exists. 
The obvious fix is to drop it during compaction, once you merge the SSTable containing the tombstone 
with the SSTable containing the original record, both can be discarded together. Clean.

But you can only safely drop a tombstone when you're certain no older copy of that key exists below it.
In leveled compaction this is traceable because levels are non-overlapping, you know exactly which 
SSTables could contain an older version. 

In size-tiered compaction it's trickier because older data is spread across multiple tiers.

> In a distributed setting with many nodes, this causes zombie-resurrections which I'll tackle in a later post.

### Range deletes

We've discussed deleting point records. What if we delete a range of records together? 
This becomes expesnsive because it has to write many tombstones and compaction also becomes tricky.
We might not even know which keys exist in that range.

RocksDB solves this with range-tombstones, stored separately as it's own metadata block where entire range deletes
are represented as a single entry. This implies the read-path has to check this block for range deletes along with the
usual path we discussed above. This adds overhead but is cheaper than writing out all of the tombstones for range deletes and
dealing with the overhead of compaction.

## Crash Recovery
Crash recovery happens from the WAL.

### Sequence Numbers
Every write in an LSM system carries a monotonically increasing sequence number. 
This is how the system reasons about ordering across the memtable, frozen memtables, and SSTables simultaneously.
When you replay the WAL on recovery, you only replay entries with sequence numbers higher than the 
highest sequence number already represented in the on-disk SSTables. Everything below that watermark is 
already durable. This makes the recovery logic clean even if the WAL and SSTables have some overlap.

Examples for DB's that use LSM trees internally: RocksDB, LevelDB, Cassandra, BigTable and HBase.
# Appendix
## Red-Black trees
Red-Black Tree
A red-black tree is a self-balancing BST. Every node is colored red or black, and the tree 
enforces four invariants after every insert or delete:

- The root is black.
- No two adjacent nodes are both red.
- Every path from a node to a null leaf has the same number of black nodes.
- Null leaves are considered black.

These invariants together guarantee that the longest path from root to leaf is at most 2X
the shortest path. That's what keeps the tree "approximately balanced", not perfectly balanced like an 
AVL tree, but balanced enough that height stays O(log n).
When you insert a node, it's colored red initially (because inserting a red node can never violate rule 3,
it doesn't change the black-height of any path). If that causes a rule 2 violation (red parent),
you fix it through a combination of recoloring and rotations. 
A rotation is a local pointer rearrangement that changes the structure without breaking the BST 
ordering property (left child is smaller than root, right child is greater than root).
The practical consequence: red-black trees favor cheaper writes over perfect balance.

## Skip-List
A skip list is a completely different idea. It's a layered linked list.
Start with a base layer containing all your keys in sorted order, 
linked sequentially. Then build additional layers above it. Each higher layer contains a probabilistic subset of the keys 
from the layer below (typically each key is promoted to the next layer with 50% probability).
The top layer has very few keys. To search for a key, you start at the top-left. 
At each node you make a decision: 
- if the next node in this layer is ≤ your target, move right. 
- If it's > your target, drop down a layer. Repeat until you find the key or bottom out.
Because each layer skips roughly half the nodes of the layer below, 
you descend about log n layers and traverse some fixed small number of nodes per layer. 
Total: O(log n) expected, same as the BST.

Inserts work by finding the correct position at the base layer, 
then flipping coins to decide how many layers the new node should participate in, and splicing it into each.
The key insight is that no rebalancing is needed. 
The probabilistic promotion is what maintains the expected structure. 
There are no invariants to enforce after an insert, no rotations, no color fixes,just a coin flip and 
some pointer updates.

> I spent some time on this : 
> The probabilistic part affects performance (a bad run of coin flips could give you a lopsided structure), not correctness.

This is why skip lists dominate in concurrent settings. 
In a red-black tree, a single insert can trigger rotations that cascade up the tree, 
requiring locks on multiple nodes.

In a skip list, an insert touches only the immediate neighbors at each level, 
and those pointer updates can be made atomic with compare-and-swap operations. 
You can implement a lock-free skip list; a lock-free red-black tree is significantly harder.
RocksDB's memtable uses a skip list by default for exactly this reason, high-throughput concurrent writes
with minimal contention.

## References
- [Fay Chang, Jeffrey Dean, et al., "Bigtable: A Distributed Storage System for Structured Data"](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
- [LevelDB impl doc](https://github.com/google/leveldb/blob/main/doc/impl.md)
- [RocksDB: Evolution of Development Priorities in aKey-value Store Serving Large-scale Applications](https://dl.acm.org/doi/epdf/10.1145/3483840)
> This is a long but wonderful read, and it takes us through the different metrics they optimized for during development,
> SSD wear(write-amplification), CPU utilization, Space utilization etc.
