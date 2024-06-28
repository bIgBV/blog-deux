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

# [Working model]
[working model]: #Working-model

The model for a bitcask database is based off a single directory which is the `Bitcask` containing the data files. At any given point, only one of them is open for 
{% sidenote(input_name="write-semantics", inline_segment="writing" ) %}
    The paper also calls out the fact that only one process can ever open the active file with read write permissions.
{% end %}. Once the size of the active file reaches a certain threshold, a new file is opened for writing and the previous file is considered immutable. 

An interesting note here is by ensuring that the active file is only append only, when using spinning rust, you don't need a disk seek to write the latest entry. The old, data files are considered immutable and only ever read from. In addition to that, deletions are tombstone values which are appended. This could be a problem for the `get` path, but the authors handle this via using a `KeyDir` structure which I'll go into soon.

# [Entry format]
[entry format]: #entry-format

Each entry on disk is of variable length, using a length prefix encoding for the keys and values. The paper does not prescribe good defaults for these values. 

![file-disk-entry](../entry-header.png)

And as you can see, the structure includes and internal only timestamp along with a CRC to wrap the whole thing for data integrity.

# [Keydir]
[keydir]: #keydir

This is where the hash table part of the "Hash table merge tree" comes in. All the active keys and their latest location on disk is stored in memory in the `KeyDir` structure. Both the keys and values are fixed size, which I suspect allows for optimized hash table impelentations.

> The resizing behaviour of most hashtables might not be what we want for an in memory structure which which more likely than not would reach a stable size. I wonder what kind of a data structure would fit this use case better.

There's a "KeyDir" structure created to map every key to it's location for the latest entry in a given file. This is a fixed size structure, which I suspect can be held in memory.

![[Pasted image 20240616215543.png]]

When a write occurs, the `KeyDir` is automatically updated with the latest location, so a lookup goes through the following steps

- Get `file_id` from key dir
- Get `file_id` file from cask
- Seek to `value_pos` within file
- Read `value_sz` bytes.

According to the authors, the file system cache does a good job of keeping the hot files cached.

In order to garbage collect, since the data files are only ever appended to, leaving old entries still present, a compaction process referred to as "merging" is done. It iterates over all the entries outputting a file containing only the live version of the entries. 


