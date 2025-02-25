+++
title = "Half measures and a story of testing"
date = 2024-10-14
draft = true

[taxonomies]
categories = ["programming", "rust"]
+++

Classically, when building systems, unit tests work the best within the core of
the system for testing business logic, while integration tests are better
suited to handle testing the functionality the interfaces treating the entire
system as a black box.

The issue with this though, is two fold:

1. Complex setup requirements.



```rust
struct TestConnection<THandler> {
    handlers: HashMap<&'static str, THandler>
}

impl<THandler> TestConnection<THandler> {
    pub fn new() -> Self {
        Self {
            handler: HashMap::new()
        }
    }

    pub fn register_handler(&self, handler: T) {
    }
}
```

* Redis protocol speaking client.
* RESP3: a binary protocol
* Builing complex workflows for managing cluster state
* Integration testing. But not really
    * We're at an edge of the system, so integration testing would realistically would be the full functional testing.
    * Complex infra
    * Needs real systems set up
    * Generally take a lot of time
* Instead use snapshot testing
    * [insta](https://insta.rs)
* Cjpture the state _on the wire_ and assert on it's entirety.
    * You're looking for whether or not the bits on the "wire" match the expected behavior
* To do this, we need a "wire" which we can open up, inspect and serialize it
    * In our case, it's a `TestConnection` which implements the `ConnectionLike` interface which we're using
    * Internally there's a `TestBuffer` type which is "wire" that the client is writing to.
        * Uses a lock to linearize everything, and therefore, we have deterministic execution.
            * Assumes that there are no other sources of non-determinism (which we currently do not have)
* But to do more complex things than just assert on what the code is doing in the happy case, we need to introduce hiccups.
* And `TestConnection` does this through handler stubs.
    * This is currently a pretty crude implementation of a stub, specialized to the particular methods that we care about
    * Can definitely be improved to (I've already run into a bunch of trouble maintaining the multiple locations of the bindings.)

Recently, I've been working on building a cluster controller for managing [Garnet][garnet] 

[garnet]: https://microsoft.github.io/garnet/docs
