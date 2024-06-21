+++
title = "BitCask a hash table based merge tree"
date = 2018-01-22

[taxonomies]
categories = ["programming", "papers"]
+++

- Interesting to call out pluggable local storage out of the door.
- Allowed them to test various storage engines easily.

## Goals
* Load latency single item reads and write
* High throughput for streaming random writes
* Larger than RAM data sets without any performance degradation
* Crash recovery and durability
* Easy backup and reetore

- Key insight: Combining hash tables with the semantics of a LSM tree merging. 

## Model
- A single instance is a directory of files.
- At a given time, only one file is ever open for writing.
- Once it reaches a certain size threshold, a new file is opened for writing, and the previous file is considered immutable.
- Immutable files are only ever read from

The active file is only ever appended to, allowing for consistent performance on spinning rust.

## Entry format
![[Pasted image 20240616214202.png]]

Deletions are special tombstone entries which get removed during merging.

There's a "KeyDir" structure created to map every key to it's location for the latest entry in a given file. This is a fixed size structure, which I suspect can be held in memory.

![[Pasted image 20240616215543.png]]

When a write occurs, the `KeyDir` is automatically updated with the latest location, so a lookup goes through the following steps

- Get `file_id` from key dir
- Get `file_id` file from cask
- Seek to `value_pos` within file
- Read `value_sz` bytes.

According to the authors, the file system cache does a good job of keeping the hot files cached.

In order to garbage collect, since the data files are only ever appended to, leaving old entries still present, a compaction process referred to as "merging" is done. It iterates over all the entries outputting a file containing only the live version of the entries. 


