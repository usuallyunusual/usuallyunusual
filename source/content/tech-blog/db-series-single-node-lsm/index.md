---
title : 1b - How a single node actually works - LSM tree based DB's.
date : 2026-06-24
summary : "How an LSM tree based single node DB works including compaction, MemTables, SSTables."
tags : ["databases", "data-engineering", "softwareengineering"]
---


LSM trees (Log Structured Merge trees), at a high level,  are append only log files that are compacted 
every now and then.

Assume you have a file and a process writing to it in append-only mode. This is your primitive DB. 
Write-Path :
The process is oblivious to the contents of the file, it just appends whatever you're asking of the write. 
Implying excellent write throughput. This also implies you cannot update the record in the traditional sense, you 
need to add it as a new record at the tail-end of the file. Which has imlications on the read-path (below)
Read-Path : 
When you ask it to read a particular record, the process needs to go through the entire file to find the right record with the ID
and then return it for you. Read throughput is horrible since we have no way of knowing whether a record even exists.

The write-path (above) implies that the file has to be read from the bottom to ensure that the latest version of the record
is fetched since updated records live in the bottom. The last record for a particular key is the most updated version of the record.
 Conclusion : Read throughput is horrible.
