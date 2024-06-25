+++
title = "BitCask a hash table based merge tree"
date = 2024-06-21

[taxonomies]
categories = ["programming", "papers"]
+++

# [Overview]
[overview]: #overview

As a part of a exercise in learing distributed databases from first principals, I came across the [Bitcask paper][bitcask-paper] which describes the implementation of a log-structured Hash table. Here are some of my notes on the paper.

[bitcask-paper]: https://riak.com/assets/bitcask-intro.pdf

From the start, the authors focued on plugagable storage, allowing them to test different kinds of storage engines.

They laid out the following goals for the databse(paraphrasing):

* Low latency single item reads and write
* High throughput for streaming random writes
* Larger than RAM data sets without any performance degradation
* Crash recovery and durability
* Easy backup and reetore

The key insight they presented was to combine the amortized constant time semantics of Hash tables with the write performance characteristics of LSM trees.

# Working model
The model for a bitcask database is based off a single directory which is the `Bitcask` containing the data files. At any given point, only one of them is open for {% sidenote(number="1", title="writing" ) %}
The paper also calls out the fact that only one process can ever open the active file with read write permissions.
{% end %} 

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


