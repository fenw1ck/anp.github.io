---
layout: post
title:  "cue: a little parallel pipeline"
date:   2016-07-01
categories: rust
---

I published a quick little crate for long-running "streaming" parallel tasks.

## Why?

Lately I've been working on a few CLI tools in Rust which do some parallel batch processing of many inputs, and write the results to a file. Putting a lock on a `BufWriter` and writing from worker threads is a bit too clumsy, and when I have ~32 threads, there's sometimes quite a bit of lock contention while waiting for the writes to complete.

So, I've found myself using a pattern like this:

![cue pipeline](/assets/img/cue-pipeline.svg)

Since I've been copy-pasting this everywhere and wasn't able to find anything comparable on [crates.io](http://crates.io), I decided to extract it into a library and publish it.

## What?

The [code is on GitHub](https://github.com/anp/cue). The `pipeline` method spawns up a couple of queues (one with a bounded buffer for incoming work, to limit RAM usage, and a lock free queue for results to reduce contention between worker threads), and then executes a pair of closures in parallel on all of the inputs from an iterator. Here's the (non-compiling) example from the repo:

```rust
extern crate cue;

fn main() {
    cue::pipeline(
        // naming the pipeline allows for better logging if multiple are running
        "demo",

        // number of worker threads needed, result thread will be spun up in addition
        8,

        // an iterator which yields items of the desired work type -- should be lazy
        // otherwise it doesn't make much sense to use a bounded work queue
        create_lazy_iterator_with_lots_of_items(),

        // item must match the Item type of the iterator above
        |item| { do_super_duper_expensive_task_which_returns_result(item) },

        // r here must match the return type of the worker closure
        |r| { write_result_to_disk_which_may_take_a_while(r); });

    println!("Done! The work has been processed in parallel.");
}
```

And there you go! I'm currently using this for a couple of tools for CLI tools which process 100M or more input items, and it's working nicely. I look forward to seeing other high-level tools for parallelism become available for Rust.
