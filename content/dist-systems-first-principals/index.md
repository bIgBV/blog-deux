+++
title = "Distributed systems from first principals"
date = 2024-06-20
draft = true

[taxonomies]
categories = ["programming", "papers", "projects"]
+++

* Long term project building different parts of a distributed system from first principals
* Core composed of a key value database
    * First implementation a hash table merge tree store
    * Possible LSM tree based implementation in the future.
* Building on that, distributed consensus.
    * Raft to start with
    * paxos/multi-paxos? That'll be harder, but also fun
* Finally a distributed membership layer to tie it all together.
* This is primarily a learning exercise, to better understand what it means to buid these services.
