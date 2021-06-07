# Caching Recursive Functions with the `cached` Crate

_#rust | #python | #caching_

During a recent YouTube derping session, I ran across [this][caching-decorator-vid] video on the `@cache` decorator from Python's `functools` library. This decorator augments the decorated function with memoization capabilities, providing significant speedups with just a one-liner!

I was curious then if there was an analogous Rust crate, which led me to find [cached][cached-crate]. 


caching-decorator-vid: https://www.youtube.com/watch?v=DnKxKFXB4NQ
cached-crate: https://crates.io/crates/cached
