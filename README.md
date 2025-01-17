# cosync

![docs.rs](https://img.shields.io/docsrs/cosync)
![Crates.io](https://img.shields.io/crates/v/cosync)
![Crates.io](https://img.shields.io/crates/l/cosync)

This crate provides a single-threaded, sequential, parameterized async runtime. In other words, this creates *coroutines*, specifically targeting video game logic, though `cosync` is suitable for creating any sequences of directions which take time.

Here's a basic `Cosync` example:

```rust
use cosync::{Cosync, CosyncInput};

fn main() {
    // the type parameter is the *value* which other functions will get.
    let mut cosync: Cosync<i32> = Cosync::new();
    let example_move = 20;

    // there are a few ways to queue tasks, but here's a simple one:
    cosync.queue(move |mut input: CosyncInput<i32>| async move {
        // set our input to `example_move`...
        *input.get() = example_move;
    });

    let mut value = 0;
    cosync.run_until_stall(&mut value);

    // okay, we ran our future, and since it has no awaits, we know
    // it will have completed!
    assert_eq!(value, example_move);
}
```

Additionally, `Cosync` can handle unsized Ts, including dynamic dispatch:

```rust
use cosync::Cosync;

// unsized type
let mut cosync: Cosync<str> = Cosync::new();
cosync.queue(|mut input| async move {
    let input_guard = input.get();
    let inner_str: &str = &input_guard;
    println!("inner str = {}", inner_str);
});

// dynamic dispatch
trait DynDispatch {
    fn test(&self);
}
let mut cosync_dyn: Cosync<dyn DynDispatch> = Cosync::new();
cosync_dyn.queue(|mut input| async move {
    let inner: &mut dyn DynDispatch = &mut *input.get();
});
```

`Cosync` is **not** multithreaded, nor parallel -- it works entirely sequentially. Think of it as a useful way of expressing code that is multistaged and takes *time* to complete, that you want to do *later*. Moving cameras, staging actors, and performing animations often work well with `Cosync`. Loading asset files, doing mathematical computations, or doing IO should be done by more easily multithreaded runtimes such as [switchyard](https://github.com/BVE-Reborn/switchyard).

This crate exposes two methods for driving the runtime: `run_until_stall` and `run_blocking`. You generally want `run_until_stall`, which attempts process as much of the queue as it can, until it cannot (ie, a future returns `Poll::Pending`), at which point control is returned to the caller.

There are three ways to make new tasks. First, the `Cosync` struct itself has a `queue` method on it. Secondly, each task gets a `CosyncInput<T>` as a parameter, which has `get` (to get access to your `&mut T`) and `queue` to queue another task (which is at the end of the queue, not necessarily after the task which added it). Lastly, you can create a `CosyncQueueHandle` with `Cosync::create_queue_handle` which is `Send` and can be given to other threads to create new tasks for the `Cosync`.

This crate depends on only `std`. It is in an early state of development, but is in production ready state right now.
