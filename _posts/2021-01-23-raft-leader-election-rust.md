---
layout:     post
title:      Implementing Raft's Leader Election in Rust
date:       2021-01-23 00:00:00
summary:
categories: Programming
---

Consensus algorithms is a topic that always caught my attention: it is complex and hard and needs a precise and safe solution. In other words: We have a couple of machines forming a cluster, and they operate on identical copies of the same data and can continue operating even in the scenario of some servers being down. This approach is used to solve a bunch of problems in distributed systems.

To give a bit of background of where we currently stand, we have to talk about [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)). Over the past decade or more, Paxos was almost synonymous with consensus, as it is the protocol taught in most computer science courses, and most implementations of consensus make use of it. The only problem is that Paxos is really difficult to understand, thus making it very hard to be implemented correctly. With that, Diego Ongaro and John Ousterhout [designed Raft](https://raft.github.io/raft.pdf), with the most important goal of making it understandable.

Given that Raft focuses so much on understandability, it breaks down the consensus into three relatively independent subproblems (quoting the Raft paper):

* Leader election: a new leader must be chosen when an existing leader fails.
* Log replication: the leader must accept log entries from clients and replicate them across the cluster, forcing the other logs to agree with their own
* Safety: if any server has applied a given log entry to its state machine, then no other server may apply a different command for the same log index.

In this blog post, I will cover only the Leader election. The other two topics I will cover in the future in other blog posts!

### The basics of Raft and leader election

A Raft cluster is composed of several servers and five is a typical number, therefore allowing the system to have two failures. A server in this cluster is in one of three possible states:

* Leader: Handles all client requests.
* Followers: are passive, and only respond to requests from leaders.
* Candidate: a state used to elect a new leader.

Raft divides time into terms of arbitrary length, and they are numbered with consecutive integers. Each one of these terms starts with an election, where one or more candidates attempt to become the leader of the cluster. When a candidate wins the election, it serves as the leader for the rest of the term. To explain this process a bit better, we can illustrate these transitions.

![raft-states](/images/raft-states-2.png)

Raft uses a heartbeat mechanism to trigger elections. When a server joins the cluster, they begin as followers, and they remain in this state as long as they receive valid heartbeats from the current leader. The leaders send periodic heartbeats to all followers in order to notify them of the existence of it and maintain its authority. When a follower does not receive a heartbeat over a period of time, named in Raft "election timeout", then it proceeds to start a new election.

A new election follows these steps:

1. Changing the state from follower to candidate
1. Increase the current term by 1
1. Votes for itself
1. Requests votes from all servers in the cluster
1. If the server gets the majority of the votes, it assumes the role of leader
1. Starts sending heartbeats to all followers, including the new term

This is a very simplistic and short explanation for the leader election process, and for more details do not hesitate in [reading the paper](https://raft.github.io/raft.pdf)! It's a very pleasant read.

### Implementation in Rust

[Rust](https://www.rust-lang.org/) is a programming language designed for performance, safety, and safe concurrency. It is strongly typed, compiled, has no garbage collector, and has no runtime (a.k.a a very minimal runtime).

I got interested in Rust at the end of 2020, by reading a couple of blog posts. With that trigger, I decided to read [The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html) book and started to code very small projects. Only that very small projects do not tell you much about the language in the real world. So I decided to implement something that is more related to what I work on on my daily basis: dealing with distributed systems.

Note: if you are not familiar with the language, you can quickly check [the Gentle intro to Rust](https://stevedonovan.github.io/rust-gentle-intro/).

#### The implementation

I decided to use the most simple and basic constructs of the language. This means no big runtimes, libs that do most of the job and etc. So I ended up using only threads for concurrency, and TCP for the RPCs.

##### Reproducing the states and types in the code

My initial approach was to find out which types were the most important ones, and reproduced them in the code.

```rust
// Examples of types.
enum State {
  FOLLOWER,
  LEADER,
  CANDIDATE,
}

enum LogEntry {
    Heartbeat { term: u64, peer_id: String },
}

struct Server {
    id: String,
    address: SocketAddrV4,
    state: State,
    term: u64,
    log_entries: Vec<LogEntry>,
    voted_for: Option<Peer>,
    next_timeout: Option<Instant>,
    config: ServerConfig,
    current_leader: Option<Leader>,
    number_of_peers: usize,
}

// Visit the file "types.rs" in the repo to see the all type definition.
```

And once my types were defined, I started to implement the state transitions step by step.

##### A server starts as follower

In order to reproduce this behavior, I implemented a `new` method inside `Server`, to ensure that all servers are started with this state.

```rust
impl Server {
    pub fn new(
        config: ServerConfig,
        number_of_peers: usize,
        address: SocketAddrV4,
        id: String,
    ) -> Self {
        Server {
            id: id,
            state: State::FOLLOWER,
            term: 0,
            log_entries: Vec::new(),
            voted_for: None,
            next_timeout: None,
            config: config,
            current_leader: None,
            number_of_peers: number_of_peers,
            address: address,
        }
    }
}
```

Now we have a server that starts as follower! Let's go and implement the rest of the state transitions.

##### Starting an election when a timeout occurs

As mentioned before, a server in Raft times out when it does not receive a heartbeat from a leader in a predetermined amount of time. Each server has a randomized timeout setting, to ensure that servers time out at different times.

To implement that, I decided to do the following:

``` rust
// The final implementation is not exactly like this,
// but the idea is the same.
thread::spawn(|| {
    loop {
        if server.has_timed_out() {
            new_election(&server);
        }
    }
});
```
So we have a thread that runs an infinite loop, always checking for the server's timeout. Whenever it occurs, a new election is triggered.

The election process, in the happy path, occurs as follow:

```rust
// 1. Changing the state from follower to candidate
server.state = State::CANDIDATE;

// 2. Increase the current term by one
server.term = server.term + 1;
server.refresh_timeout();

// 3. Vote for itself
server.voted_for = Some(Peer {
    id: server.id.to_string(),
    address: server.address,
});

// 4. Prepare vote requests and make the RPC request
let request_vote_rpc = Some(VoteRequest {
    term: new_term,
    candidate_id: id,
})

let rpc_response = rpc_client.request_vote(request_vote_rpc);

// 5. if the server gets the majority of votes, becomes leader
if has_won_the_election(&server, rpc_response) {

    // 6. sends heartbeats to all followers
    let log_entry = LogEntry::Heartbeat {
        term: server.term,
        peer_id: server.id.to_string(),
    };

    rpc_client.broadcast_log_entry(log_entry);
}
```

##### Election timeout and discovering another leader

In case a timeout occurs or the server does not get the majority of the votes, it will start the election once again, increasing the term once more. In Raft, it is totally fine to have terms without an leader.

Another interesting case can occur during the election process: another server in the cluster is elected, and starts sending heartbeats. When this occurs, our server changes back the state from `CANDIDATE` to `FOLLOWER`, and set the terms to the current leader's term.

```rust
// Example of how a server behaves when receiving heartbeats
// It becomes a follower if the received heartbeat has a higher
// term than itself.

if term > server.term {
    info!(
        "Server {} becoming follower. The new leader is: {}",
        server.id, peer_id
    );

    server.term = term;
    server.state = State::FOLLOWER;
    server.voted_for = None;
    server.current_leader = Some(Leader {
        id: peer_id.to_string(),
        term: term,
    })
}
```

With that, we have covered all the steps for the state transitions in Raft's leader election algorithm!

##### More info about the full implementation

The core logic for the leader election is implemented in the `raft::core` package, where functions to handle log entries (heartbeats), elections, and timeouts are placed. Please visit the [full implementation](https://github.com/laurocaetano/rsraft/blob/fda7e677627577e452fef5858d58bce3ed8d74f6/src/raft/core.rs) for a more detailed look into the code.

*Important notes*

RPCs are done through TCP connections. Each server starts up with a TPC listener, and create client connections to all other servers in the cluster.

The implementation is not making usage of persistent storage, so all operations are done in memory. For this reason, the `Server` instance is shared across many functions, by making usage of Rust's `Arc` and `Mutex` combination. It took me a while to understand the concepts, but once you get the basics, it turns out very simple!

There is a [demo](https://github.com/laurocaetano/rsraft/blob/fda7e677627577e452fef5858d58bce3ed8d74f6/src/raft/demo.rs), which runs and demonstrates the process of electing a leader in a new cluster. By running it, we get the following output:

```
22:46:10 [INFO] Starting server at: 127.0.0.1:3300...
22:46:10 [INFO] Starting server at: 127.0.0.1:3301...
22:46:10 [INFO] Starting server at: 127.0.0.1:3302...
22:46:11 [INFO] The server server_1, has a timeout of 3 seconds.
22:46:11 [INFO] The server server_3, has a timeout of 6 seconds.
22:46:11 [INFO] The server server_2, has a timeout of 5 seconds.
22:46:14 [INFO] Server server_1 has timed out.
22:46:14 [INFO] Server server_1, with term 1, started the election process.
22:46:14 [INFO] Server server_1 has won the election! The new term is: 1
22:46:14 [INFO] Server server_3 with term 0, received heartbeat from server_1 with term 1
22:46:14 [INFO] Server server_3 becoming follower. The new leader is: server_1
22:46:14 [INFO] Server server_2 with term 0, received heartbeat from server_1 with term 1
22:46:14 [INFO] Server server_2 becoming follower. The new leader is: server_1
22:46:14 [INFO] Server server_3 with term 1, received heartbeat from server_1 with term 1
22:46:14 [INFO] Server server_2 with term 1, received heartbeat from server_1 with term 1
22:46:16 [INFO] Server server_3 with term 1, received heartbeat from server_1 with term 1
22:46:16 [INFO] Server server_2 with term 1, received heartbeat from server_1 with term 1
```

Please check out the [full implementation in my GitHub repo](https://github.com/laurocaetano/rsraft)!

### Conclusions

Writing code in Rust is very pleasant. The syntax is very nice, it has very good support for emacs, Cargo is super simple and easy, and the compiler really helps you out in finding out your mistakes.

The downside is the learning curve, at least for me who is used to languages with GCs. Understanding the borrow checker takes time, and mistakes will be made on the way - although it is a powerful part of the language.

All in all, I learned a lot during the two weeks of implementing Raft's leader election, and I highly recommend you doing something like that too! It's much fun and you end up learning a lot on the way.
