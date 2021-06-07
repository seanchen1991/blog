# Implementing Cyclic Data Structures in Safe Stable Rust

In an episode of the [*Building with Rust*][bwr] podcast, I spoke with [Ralf Jung][ralf] about some research he's been working on with others: [GhostCell][ghostcell], which is a technique for loosening some of Rust's ownership restrictions. As a result, GhostCell partly enables what Ralf and his colleagues refer to as "data structures with in-degree > 1", e.g., linked list and graphs, to be implemented in an efficient manner using completely-safe Rust. 

Such data structures don't typically play well with Rust's ownership rules because, fundamentally, they require that data be owned and mutatable by multiple owners. 

To be clear, there are other strategies for implementing in-degree > 1 data structures in Rust in a safe manner. These strategies include:
 - Utilizing a `Vec` to act as an adjacency list for storing nodes as well as the node's neighbors.
 - Making use of an arena allocator to mitigate the possible hazards that arise when working with self-referential data in Rust.
 - Using `Rc` in combination with `RefCell` to punt the Rust compiler's checks that each node is mutated by at most one owner from compile-time to runtime.  

However, each of these strategies exhibit significant drawbacks in one way or another, to the point where Rust just wasn't really a great language to use if one had to work with these data structures in any non-trivial way. 



bwr: https://anchor.fm/building-with-rust/
ralf: https://www.ralfj.de/blog/
ghostcell: http://plv.mpi-sws.org/rustbelt/ghostcell/

