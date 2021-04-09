# Semaphores in Rust
_November 27th, 2020 | #rust | #synchronization_primitives | #the_little_book_of_semaphores | #intermediate_

> Note: I recently stumbled upon Allen Downey’s [_The Little Book of Semaphores_](https://greenteapress.com/wp/semaphores), which is a short, sweet, and to-the-point textbook on the topic of synchronization. 
> Naturally, reading through the book inspired me to implement these synchronization primitives and problems in Rust.
> I hope others find this exploration interesting, insightful, and/or helpful! 🙂

Implementing synchronization primitives in Rust in a safe fashion is a bit strange in a circular kind of way. It’s similar to implementing a hash map in JavaScript: since everything in JS boils down to an Object, you’re essentially implementing a hash map using hash maps!

Similarly, for this semaphore implementation we’ll be using standard library types that are “more complex” from a conceptual and implementation standpoint than the primitive we’re actually implementing. 

I’m sure there’s a more low-level way to do this in Rust that better captures the primitive nature of these primitives, but that likely involves writing unsafe code and I’m just not comfortable enough yet with that. Maybe in the future, this will be something I’ll revisit! 

## What are Semaphores?

Semaphores are one of the simplest synchronization primitives upon which more complex tools (such as mutexes, barriers, etc.) can be built upon. 

Conceptually, a semaphore is just an integer that can only be incremented and decremented in an atomic fashion. You can imagine that the integer represents some number of resources that only a certain number of threads are able to access or acquire at the same time. 

> Note: The term _atomic_ means that even when multiple threads are all attempting to change the value of the semaphore, those changes occur in a sequential (and thus deterministic) order. 

There are different implications depending on whether the integer is in a positive or negative state. If it’s in a positive state `n`, this means that `n` threads are able to access the protected resource. If it’s in a negative state `-n`, this means that `n` threads are all blocked and waiting for access to the protected resource. 

## A Basic Semaphore Implementation

From the above description, our semaphore will contain an integer that can only be access in an atomic fashion. It will need a method that allows a thread to access the underlying critical section and increment the atomic integer in the process. It will also need a method that allows a thread to decrement the atomic integer and signal a waiting thread (assuming there is at least one) that they may acquire the semaphore. 

This is where what I said earlier about using more “advanced” synchronization primitives to implement a simpler one comes into play. In order to notify one of the threads waiting on the semaphore, we’ll need to make use of a [condition variable][condvar], which is the tool that facilitates threads waiting for their turn. It’s also what allows the semaphore to notify a waiting thread that they may not acquire the semaphore.  

In conjunction with a condition variable, we’ll also be using a `Mutex` to ensure that our integer is only ever accessed by one thread at a time. 

```rust
use std::sync::{Mutex, Condvar};

pub struct Semaphore {
    condvar: Condvar::new(),	
    counter: Mutex<isize>
}
```

Now let’s add our two methods. We’ll call the method that decrements the semaphore `acquire` to communicate the fact that when a thread calls this method, it’s acquiring one of the protected resources:

```rust
impl Semaphore {
    pub fn acquire(&self) {
        // gain access to the atomic integer 
        let mut count = self.counter.lock().unwrap();

        // wait so long as the value of the integer <= 0
        while *count <= 0 {
            count = self.condvar.wait(count).unwrap();
        }

        // decrement our count to indicate that we acquired
        // one of the resources
        *count -= 1;	
    }
}
```

We’ll call the second method that increments the semaphore `release` to indicate the fact that it’s releasing one of the protected resources so that another waiting thread may access it:

```rust
impl Semaphore {
    ...
    pub fn release(&self) {
        // gain access to the atomic integer
        let mut count = self.counter.lock().unwrap();

        // increment its value 
        *count += 1;

        // notify one of the waiting threads 
        self.condvar.notify_one();
    }
}
```

And that should do it for our (relatively) basic semaphore implementation! 

Let’s go ahead and wrap this up by adding a few tests:

```rust
#[cfg(test)]
mod tests {
    use super::Semaphore;
    use std::thread;
    use std::sync::Arc;
    use std::sync::mpsc::channel;
  
    #[test]
    fn test_sem_acquire_release() {
        let sem = Semaphore::new(1);

        sem.acquire();
        sem.release();
        sem.acquire();
    }
  
    #[test]
    fn test_child_waits_parent_signals() {
        let s1 = Arc::new(Semaphore::new(0));
        let s2 = s1.clone();

        let (tx, rx) = channel();

        let _t = thread::spawn(move || {
            s2.acquire();
            tx.send(()).unwrap();
        });

        s1.release();
        let _ = rx.recv();
    }
  
    #[test]
    fn test_parent_waits_child_signals() {
        let s1 = Arc::new(Semaphore::new(0));
        let s2 = s1.clone();

        let (tx, rx) = channel();

        let _t = thread::spawn(move || {
            s2.release();
            let _ = rx.recv();
        });

        s1.acquire();
        tx.send(()).unwrap();
    }
}
```
 
In the next post we’ll be using our homebrew semaphore to implement some other synchronization primitives!

[condvar]: https://doc.rust-lang.org/std/sync/struct.Condvar.html
