+++
title = "A memory profiling journey"
description="I walk through the process of discovering the consequences of not having backpressure in a system"
date = 2025-06-21

[taxonomies]
categories = ["programming", "rust", "profiling"]

[extra]
+++

- What are we trying to do here?
  - Increased memory usage in C++ wrapper of our Rust library.
  - Wrapper is structured with main entry point which spawns background thread per producer instance
  - `SendMessage(producer_id, message)`
- Profiling anything requires a baseline.
  - How do you set up the baseline.
  - What tools do we have access to?
    - [`stats_alloc`](https://docs.rs/stats_alloc/latest/stats_alloc/)
    - [`dhat-rs`](https://docs.rs/dhat/latest/dhat/)
  - Setting up a baseline test
    - Different feature flags for different kind of profiling.
    - How to understand the output
      - Especially for the output of dhat-rs.
- Initial changes
  - Identified a possibly unnecessary clone
  - Turned out it did not make a difference.
- dhat viewer pointed out a majority of the allocations were alive for 50% of the runtime of the program.
- Looked into the lifetime of an allocations 
  - `Vec<u8>` created by copying contents of the payload when sending the message
  - Message sent to a background thread spawned per producer.
  - Background thread moves the message into the actual `Produce` call.
  - The thread function gets a handle to the runtime and `block_on`'s the `produce` call.
  - The produce call internally sends the message to an actor which buffers and encodes the request.
- Added `created_time` to the main `Message` struct.
  - Added print statements tracking timing at every step.
- Messages sitting in FFI layer queue immediately.
- Two things immediately stood out:
  - The FFI queue to process messages did not have a bound.
  - Using `block_on` made the processing of `Message`s serial, only one messages sent in a single batch.
- How do we fix this:
  - Adding a configurable bound to the FFI layer queue.
    - Use `recv_many` in the background process to ensure batches are sent.
    - This adds backpressure to the system as the thread calling `sendMessage` will block until there is space on the channel to send a message.
    - Consequently reduces memory usage as most of the allocations are hanging around waiting to be processed.
