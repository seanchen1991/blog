# Takeaways from My Initial Exposure to Rust
_May 27th, 2020 | #rust | #introspection_

My journey into learning Rust and getting into its wonderful open source community has been, on the whole, pretty stop-and-go.

## First Encounter

I was introduced to Rust way back in 2014, probably around the time when it was around version 0.8 or so. I was a student in Hack Reactor’s sixteenth cohort, learning JavaScript, Angular, and Node.js in preparation for taking on a role as a web developer. One of the TAs (or Hackers-In-Residence as they were called back then) who mentored my cohort was an early adopter of Rust and was very active in the open source community.

He was (and still is?) a rather ardent advocate for Rust, citing its algebraic data types and its functional idioms. While my memories of that time are rather hazy, one thing I can say with absolute certainty is that I had zero idea of what any of it meant; I just nodded my head and pretended I understood.

To his credit, this TA, perhaps noticing my wide-eyed curiosity, mentored me in writing a tiny Rust library that I published on my then-fledgling GitHub account. I had very little idea what any of the code he helped me write meant, though it was apparently a useful utility for him that he pulled in as a dependency on whatever crazy project he was working on at the time.

## First Public Rust Code

Here’s the code for that early project, in all of its glorious version 0.8 Rust goodness:

```rust
//! A reader + writer stream backed by an in-memory buffer.

use std::io;
use std::slice;
use std::cmp::min;
use std::io::IoResult;

/// `MemStream` is a reader + writer stream backed by an in-memory buffer
pub struct MemStream {
    buf: Vec<u8>,
    pos: uint    
}

#[deriving(PartialOrd)]
impl MemStream {
    /// Creates a new `MemStream` which can be read and written to 
    pub fn new() -> MemStream {
        MemStream {
            buf: vec![],
            pos: 0 
        }
    }
    /// Tests whether this stream has read all bytes in its ring buffer
    /// If `true`, then this will no longer return bytes from `read`
    pub fn eof(&self) -> bool { self.pos >= self.buf.len() }
    /// Acquires an immutable reference to the underlying buffer of 
    /// this `MemStream`
    pub fn as_slice<'a>(&'a self) -> &'a [u8] { self.buf.as_slice() }
    /// Unwraps this `MemStream`, returning the underlying buffer
    pub fn unwrap(self) -> Vec<u8> { self.buf }
}

impl Reader for MemStream {
    fn read(&mut self, buf: &mut [u8]) -> IoResult<uint> {
        if self.eof() { return Err(io::standard_error(io::EndOfFile)) }
        let write_len = min(buf.len(), self.buf.len() - self.pos);
        {   
            let input = self.buf.slice(self.pos, self.pos + write_len);
            let output = buf.slice_mut(0, write_len);
            assert_eq!(input.len(), output.len());
            slice::bytes::copy_memory(output, input);
        }
        self.pos += write_len;
        assert!(self.pos <= self.buf.len());

        return Ok(write_len);
    }
}

impl Writer for MemStream {
    fn write(&mut self, buf: &[u8]) -> IoResult<()> {
        self.buf.push_all(buf);
        Ok(())
    }
}
```

It’s essentially a wrapper around a `Vec<u8>` that implements the `std::io::Reader` and `std::io::Writer` traits. `Reader`s are objects that can be read from, while `Writer`s are objects that can be written to. Files would be one prime example of something that requires these traits. 

> Note: Since those early pre-1.0 days, the trait names have been shorted from `Reader` and `Writer` to just `Read` and `Write`. Evidently, back in 0.8, the `Write` trait only required a `write` method to fulfill the trait. Nowadays, a `flush` method is also required as a way to completely drain all of the contents of the `Write`r.

Looking back at it now, it’s pretty cool to see that, even in its pre-1.0 days, Rust code really doesn’t look all that different from its current incarnation.

Context is important here. I was not a seasoned or confident programmer at the time by any means. I’d been learning how to assemble simple web front ends and back-end APIs for the past few months. I most certainly couldn’t tell you the difference between the stack and the heap; apparently it just wasn’t necessary to know about in order to implement web applications. I was missing so much foundational knowledge and background that I had no inkling why Rust’s philosophy and its combination of features were (and are) game-changing for the wider programming community.

To be honest, I think I just wanted some of this TA’s genius to rub off on me.

## Goodbye For Now

The months following Hack Reactor were a whirlwind of interviews and first-job jitters that consumed my waking days. I wouldn’t revisit Rust for almost two years, but the seed had been planted.

Timing can be everything when it comes to learning. At the outset of my programming journey, I was certainly too immature to understand Rust. That being said, I also think that due to the Rust seed having been planted in my mind early on, my subsequent learnings were shaped by that early exposure.

Indeed I would come to find out in the next few years that I really did not find web development all that interesting. I was drawn much more to concepts and ideas belonging to the realms of operating systems, systems programming, basically “lower level stuff”. To be fair, perhaps I was interested in these sorts of things just 'cause, and not actually because I was exposed to a low-level systems programming language early on. But the end result was that I naturally found my way back to Rust, and this time, much better equipped to really dig into the language and start learning it in earnest.
