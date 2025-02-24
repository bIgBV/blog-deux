+++
title = "Deterministic simulation test -- the next step"
description = "Distributed systems are a hard problem, and in this article I try to combine principles of building sans-io state machines with Deterministic Simulation Testing to try and make my life a little easier."

date = 2025-02-23

[taxonomies]
categories = ["programming", "rust", "testing"]
+++

Distributed systems are a hard problem. Networks, hardware, and even the
universe all conspire to render your system in an inconsistent state which you
will have the express joy of debugging in the middle of the night. Generally,
when faced with the notion of trying to constrain how systems can fail,
engineers turn to testing, with a focus on integration tests to ensure that the
system as a whole works as expected even in the most dire of circumstances. 

While a good idea, in practice integration tests are seldom as comprehensive as
they set out to be. They are the first to get afflicted by bit rot and tech
debt due to their complex setup requirements. Debugging them tends to be
just as difficult as debugging the actual failures themselves.

When designing a new cluster controller for a new data store, I was faced with
solving this problem. I wanted to build a robust system, with a focus on
correctness. Plus, I wanted to validate this by running the system under a
multitude of scenarios and asserting on the expected behavior. Lucky for me,
this is a problem that engineers have been thinking of for a long time. 

This article is a exploration of how combining various techniques of building
state machines using sans-io practices, and deterministic simulation testing
leads to a lot of joy.

# The state machine

The main focus of this controller is to be highly available with a focus on
correctness, as this service will be the sole arbitrar for keeping the data
store instances in a consistent state. The best way to ensure that the
controller does not end up in a inconsistent state is to ensure that those
states are not reachable (representable), which can be done by using state
machines. There are many ways in Rust to design state machines, from complex
type states to using match statements. Ana Hoboden has an [excellent
article][ana-state-machine] on the various ways of implementing state machines
in Rust. 

[ana-state-machine]: https://hoverbear.org/blog/rust-state-machine-pattern/

For our usecase, our state machine only transitions from one state to the next
when an input is provided, so we have a core `step` method which is a function
of the form `(CurrentState, Input) -> NewState`, which internally is
implemented via a `match` statement.

```rust
struct Machine {
    state: State
}

enum State {
    Initialized,
    //...
    Failure
}

enum Input {
    Initialized,
    // ...

    ClusterEnabledState(bool),
}

enum Message {
    CheckClusterEnabled(Endpoint { uri: String })
}

fn step(self, input: Input) -> State {
    match(self.state, input) => {
        // All possible transitions here
        _ => State::Failure
    }
}
```


# Sans-IO driver

The `match` statement allows us to handle all possible states and their
transitions, plus gives us an escape hatch in the form of the `Failure` state
which is returned when an unexpected state transition occurs. State transitions
can have side effects, such as querying the cluster state or issuing a cluster
command. These side effects are not generated for every transition, so state
machine provides an API via a `poll_next_message` method to check whether or
not such a request exists. This is why the `step` method is designed to be
driven externally, via a method which might look something like this:

```rust
fn run_state_machine() {
    let machine = Machine {
        state: State::Initialized
    };

    let mut current_input = Input::Initialized;
    loop {
        let new_state = machine.step(current_input);

        if new_state.is_actionable() {
            // Perform action
        }

        if let Some(message) = machine.poll_next_message() {
            current_input = handle_message_and_get_input(message);
        } 
    }
}

```

This notion of querying the state machine for any messages that can cause
side-effects comes from the principles of building a sans-io based system. See
[this article from FireZone][firezone-sans-io] to learn more about it, but by
employing this pattern we get the following advantages:

[firezone-sans-io]: https://www.firezone.dev/blog/sans-io#user-content-fnref-2

- Synchronous core: The core state machine itself is not performing any actions
that might require asynchronous operations (think network, FS, time, etc.).
- Deterministic execution: We will expand on this later, but by inverting
control of external interactions, the code driving the state machine has
complete control over when the action can be performed.

In order to map the state machine `Input` and `Output` types to specific
actions, and their results, I have the `IoCore` type. By making this type
generic over a simulator implementation, we can can get the desired behavior we
want. The `Simulator` trait itself will be covered in the next section.

```rust
struct IoCore<S> {
    simulator: S,
}

impl<S> IoCore<S>
where
    S: Simulator,
{
    pub fn handle_message_and_get_input(message: Output) -> Result<Input, IoCoreError> {
        match input {
            // Handle various message types, for example and election message
            Output::CheckClusterEnabled(endpoint) => self
                .simulator
                .check_cluster_enabled(&endpoint.uri)
                .map(|enabled| Input::ClusterEnabledState(enabled))
                .map_err(Into::into),
        }
    }
}
```

This type is then passed to the driver, which can use it to drive the state machine.

# Deterministic Simulation Testing

We have the following pieces so far:

- A state machine.
- A driver to run the machine.
- An abstraction to call simulator APIs.

In order to close the loop, we bring in the concepts of deterministic
simulation testing(DST). DST doesn't necessarily prescribe specific steps to
take when implementing a system to get it's benefits. Rather, it's a set of
guidelines and techniques of building systems, at the end of which we should be
able to fully simulate the external systems that our system under test
interacts with.

There's a [lot][foundation-db-talk] [of][tigerbeetle]
[literature][predrag-eurorust] [on this][foundationdb] topic, and they provide
excellent context about the problem space and the general guidelines. I'm
someone who can internalize a concept only after building so it took me some
time to wrap my head around all of the pieces that come together to make for a
system that is self describing and replay-able.

[foundation-db-talk]: https://www.youtube.com/watch?v=4fFDFbi3toc
[predrag-eurorust]: https://www.youtube.com/watch?v=3EFue8PDyic
[tigerbeetle]: https://docs.tigerbeetle.com/about/vopr/
[foundationdb]: https://apple.github.io/foundationdb/testing.html


The neat thing is, this DST systems work extremely well with state machines and
systems built in sans-IO in mind, primarily because the core of the system has
been abstracted away from any source of non-determinism. And that's the main
principle of DST -- non-determinism. We want to ensure that all possible
interactions which can introduce non-determinism have been removed, such that
there is only one logical series of steps that leads to a particular outcome.
In distributed systems parlance, this is serializability. The simplest way of
achieving this is by providing your state machine, or the state machine driver
a single interface for all non-deterministic interactions. 

This interface in my case was the `Simulator` trait, which housed all external
systems methods

```rust
pub trait Simulator {
    // Random value generation
    fn gen_node_id(&self, prefix: Option<String>) -> ElectionNodeId;

    // Cluster operations
    fn is_cluster_enaled(&self, endpoint: String) -> bool;

    fn failover(&self, leader: String, replica: String) -> Result<(), Error>;

    // Election operations
    // Can be make async fns if required
    fn create_node(&self node: ElectionNodeId) -> impl Future<Output = Result<CreateState, Error>> + Send;

    // and so much more
}
```

By abstracting these operations behind the `Simulator` interface, we can have
multiple implementations of them for specialized purposes. In my case I have
two simulators `ConcreteSimulator` and `TestSimulator`. 

- **`ConcreteSimulator`**

This as the name suggests contains the concrete implementations of making the
required calls into the external systems. Internally, it is a fairly thin shim
over the API calls themselves, with the addition of the tracing functionality.
This is what makes the system self describing. The `ConcreteSimulaotor` has
functionality to serialize the input and outputs of every method that opt into
it. This prescribes certain requirements the these values, specifically the
requirement of them being serializable, _including_ the error types, as we want
to capture the full values. This is a hard requirement in order to be of use in
the `TestSimulator`

The values are written to a file on a background task started when a
`ConcreteSimulator` instance is initialized. The file name, and whether or not
recording is enabled can be easily configurable. While there are many ways of
handling the serialization format, I went for simplicity, and have
[ron][ron-github] encoded inputs and output values along with some metadata
like the method name. A single method invocation has the metadata along with
the inputs and outputs serialized into a string, which is then sent to the
worker thread to be written to the file. While it is possible to move the
serialization to the worker thread as well, this system will be only running
locally as well as specific test clusters, so I'm fine with the additional cost
of serialization on the main task, at the benefit of simplicity.

[ron-github]: https://github.com/ron-rs/ron?tab=readme-ov-file

- **`TestSimulator`**

The `TestSimulator` takes the files written created by `ConcreteSimulator` and
provides the replay-ability for the testing aspect of DST. The implementation
of `TestSimulator` contains facilities for parsing the scenario files created
by the `ConcreteSimlator`. The implementation provides a core method
`next_operation` which is responsible for deserializing the method invocation
metadata as well as inputs and output values of the method invocation.

```rust
fn next_operation<I, O>(&self, method_name: &'static str) -> Result<(I, O), SimulatorError>
where
    for<'a> I: serde::Deserialize<'a>,
    for<'b> O: serde::Deserialize<'b>,
{
    // Read next chunk and deserialize the values

    let expected_method_name = ron::from_str(scenario_file);

    if method_name != expected_method_name {
        panic!(
            "Test failure because unexpected method called: {}, expected: {}",
            method_name, expected_method_name
        );
    }

    // deserialize input and output
    (input, output)
}
```

Every method in the `Simulator` implementation of `TestSimulator` simply calls
the `next_operation` method with the expected input and output types, 

```rust
impl Simulator for TestSimulator {
    fn is_cluster_enaled(&self, endpoint: String) -> bool {
        let (input, output) = self
            .next_operation::<String, bool>("is_cluster_enabled")
            .unwrap();

        if input != endpoint {
            panic!(
                "is_cluster_enabled called with unexpected endpoint: {}, expected: {}",
                endpint, input
            );
        }

        output
    }

    // And so on
}
```

And with this, we get replayability along with behavior validation. If the
order of method calls does not match the order present in the scenario files,
then it is either a bug or a behavior issue, and therefore needs to be
investigated. 

Authoring tests now becomes as simple as instantiating the controller with the
`TestSimulator` which is given a scenario file to run through, and driving the
state machine until an expected state is reached.

And the best part is, this abstraction is not just limited to these two
implementations, we could just as easily implement a concrete simulator that
inject random failures driven by an seeded PRNG. The controller is continuously
run with the simulator recording every method invocation, and if a failure
occurs, we can simply run it through the `TestSimulator` to identify the exact
sequence of steps _and the resulting state_ that caused the failure to occur.


# Conclusion

This technique is extremely flexible and extensible, allowing for integration
tests to be run based on scenarios and abstracting the entire world out for the
core system. This allows for quicker feedback time, plus, doesn't require the
tests to set up the environment for each scenario individually. On top of this,
the techniques are extensible, and can be built on top to automatically create
new scenarios to test.

## An aside

One particularly helpful note I found when implementing this system was that
the functionally pure core is better architected as a pull based system instead
of a push based one. This is because it allows for the core (in my case the
state machine, and it's driver) to decide when to check external systems,
instead of being notified of changes which can happen at any point of time. By
inverting the control, it not only makes the core logic simpler, it also makes
the simulator implementations a lot simpler.

A concrete (pun only slightly not intended) example of this is checking for
elections state and cluster state notifications. The controller needs to act on
various notifications such as:

- Leader node disconnected
- Data store cluster node addition/failure
- Data store node primary failover
- A config file on the file system changing
- etc.

The sources of these events are disparate, and provide different mechanisms to
get the actual notifications. By inverting control, the core system dictates
when it wants to check on these notifications and acts on it, and by extension
makes it deterministic. The `Simulator` interface now only needs to have
methods like `fn check_cluster_notifications(&self) -> ClusterEvent` and `fn
check_election_notification(&self) -> ElectionEvent`, and therefore, the
implementors only need to care about serializing and de-serializing the actual
values, and not the entire mechanism of pushing the notification to the
controller.
