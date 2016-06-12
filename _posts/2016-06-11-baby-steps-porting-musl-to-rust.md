---
layout: post
title:  "Baby Steps: Slowly Porting musl to Rust"
date:   2016-06-11
categories: rust
---

**TLDR: I'm toying with writing a C standard library in Rust by porting musl-libc over function-by-function. The work is in progress at [https://github.com/dikaiosune/rusl](https://github.com/dikaiosune/rusl).**

## Are you mad?

Let's say that one is hypothetically crazy enough to rewrite part or all of a C project into Rust. Every time a new vulnerability in a C-based program or library hits Hacker News, there are a bunch of calls to just rewrite the world in Rust, but even given unlimited resources, is it actually a reasonable thing to do? How well can Rust actually replace C in C's native domains? What are best practices for managing the risks inherent in a project like that (injecting new bugs/vulnerabilities, missing important edge case functionality, etc.)?

I don't really know good answers to these questions, but I have been tinkering with this project recently and I thought it was time to put down some of my thoughts about the advantages and pitfalls of a C-to-Rust rewrite project. Also, I recently read [these great talk slides](https://github.com/carols10cents/rust-out-your-c-talk) and thought it was about time to write about the somewhat-related stuff I've been toying with.

## Show me some code!

Ok, ok. So I'm a pretty verbose guy, I get it. If you're actually interested, I encourage you to stick around, but if not here are some examples of C -> Rust ports. You can definitely look at the [commits on the repo](https://github.com/dikaiosune/rusl/commits/master), where I've tried hard and occasionally succeeded for each new function and its previous C code to be side-by-side. [Here](https://github.com/dikaiosune/rusl/commit/8cac70db2ad065e06e1320f6a232d9e3c6563490) [are](https://github.com/dikaiosune/rusl/commit/7161d26b7414e123e226d63672705daeac7476c1) a [few](https://github.com/dikaiosune/rusl/commit/0e93e5fb122101f8fb7da2bed40ef1b37727293e) [examples](https://github.com/dikaiosune/rusl/commit/deb822aa93586206ed3b33661d2bd0231dc2daac). And here's a random one which I am not in particular attached to.

C version of musl's `strlen`:

```c
#include <string.h>
#include <stdint.h>
#include <limits.h>

#define ALIGN (sizeof(size_t))
#define ONES ((size_t)-1/UCHAR_MAX)
#define HIGHS (ONES * (UCHAR_MAX/2+1))
#define HASZERO(x) ((x)-ONES & ~(x) & HIGHS)

size_t strlen(const char *s)
{
	const char *a = s;
	const size_t *w;
	for (; (uintptr_t)s % ALIGN; s++) if (!*s) return s-a;
	for (w = (const void *)s; !HASZERO(*w); w++);
	for (s = (const void *)w; *s; s++);
	return s-a;
}
```

Rust version of rusl's `strlen`:

```rust
use core::usize;

use c_types::*;

#[no_mangle]
pub unsafe extern "C" fn strlen(s: *const c_schar) -> size_t {
    // TODO(adam) convert to checking word-size chunks
    for i in 0.. {
        if *s.offset(i) == 0 { return i as usize; }
    }
    return usize::MAX;
}
```

Yeah, I get it, I get it. Don't you see the TODO? It passes the tests, and I am happy. I know it isn't as efficient if it doesn't get autovectorized (and I honestly haven't checked), and I know that returning `usize::MAX` is not semantically correct (even if it *gasp* "shouldn't ever happen" because `0..` returns a really really giant iterator range). I also know that because I don't know C very well I can read the second one in about 10% of the time. *shrug*

#### Update

So after some comments on HN about needing to return a `usize::MAX` to satisfy Rust's control flow requirements, here's an updated version that still just checks one byte at a time, but reflects the fact that it probably shouldn't return at all unless a null byte is found:

```rust
#[no_mangle]
pub unsafe extern "C" fn strlen(s: *const c_schar) -> size_t {
    // TODO(adam) convert to checking word-size chunks
    let mut i = 0;
    loop {
        if *s.offset(i) == 0 { return i as usize; }
        i += 1;
    }
}
```

It's still less efficient because it doesn't autovectorize, but I can live with that.

## What does Rust offer here?

Because Rust's standard library relies on a C standard library to provide dynamic allocation, I/O, string manipulation, etc, `rusl` has to use `#![no_std]`, disabling the standard libary and reducing the usable types to those things which can run on bare metal. However, while Rust is widely known for memory safety, it still has a lot to offer. All of these could be useful Rust features for this project:

1. cargo + crates makes some things easier. For example, I use code from [syscall](https://github.com/kmcallister/syscall.rs) to generate Linux syscalls without having to write and debug it myself. In the future, I'm hoping to pull in work from [pnet](https://github.com/libpnet/libpnet) and [math](https://github.com/nagisa/math.rs) to reuse community-generated code. I'm also using crates for a few other things, like retrieving vararg parameters when C calls the function, even though Rust doesn't technically provide support for this yet.
2. Stricter type casting semantics than C, along with clearly defined bit-widths for primitive types.
3. Sum types (Rust enums) for representing all possible values of a type (like errors).
4. Immutability-by-default and references which enforce that.
5. Slice types which know their length.
6. Array bounds checking.
7. Non-text-substitution macros.

Granted, because right now I'm just doing straight source-to-source translation, only the first two items are really proving useful for the time being. I have been playing around with using RAII-style locks in place of the explicit unlocking that the musl C code requires, and have promising results so far, but nothing yet which passes all of the tests.

Some things which are difficult because they're not yet in Rust:

1. Stable inline assembly support (can be used on nightly for the time being).
2. `const_fn` support is limited, although will be improving soon.
3. Untagged union support ([RFC accepted, not yet implemented](https://github.com/rust-lang/rust/issues/32836)).
4. Weak linker arguments for function symbols (unstable feature, but lacking documentation, precludes a C program from overriding the rusl standard library functions at link time, functionality which musl supports AFAICT).
5. Variable arguments (there's no way to make them safe, although IIRC discussion is underway for implementing them for `extern` functions).

## How am I doing this?

My goal was to build musl libc, build a Rust static library of the functions I'd implemented, and then stick them into the musl archive before it gets linked against the test suite executables. And it works! Mostly. Using a nasty bash script instead of the musl Makefile, because make is ugly and I was already in way over my head by the time I figured out how to do the bash version. [Here's the bash script](https://github.com/dikaiosune/rusl/blob/master/build_and_test.sh), but please don't revoke my membership card just because it's not the one true way.

Once the two static archives have been munged together, the libc-test suite is built and executed, using the local `build/usr/[include,lib]` directories as the header/lib search paths.

Since musl doesn't yet pass quite all of its own tests (*tsk, tsk*), I recorded the test failures from a clean unadulterated test run in a file, and on each new test run I check to make sure they haven't further regressed (see [this part](https://github.com/dikaiosune/rusl/blob/master/build_and_test.sh#L63) of the bash script).

Whenever I replace a musl C function with a rusl Rust function, the scripts clean the build directory, build the two projects from scratch, munge their archives together, and run the test suite. I fix any regressions, and once they're all fixed I commit. This is more or less in keeping with the common-in-Rust philosophy (which I quite like) of "automatically maintain a repository of code that always passes all the tests" (I first read this [here](https://graydon2.dreamwidth.org/1597.html) but I'm unfamiliar with its provenance).

Pretty simple, right? Haha yeah. Until you're in a debugger stepping back and forth between C, Rust and assembly. It's thrilling in many ways, but it's also introduced me to a whole new category of problems and fun things!

## How much progress is there?

Not a lot. GitHub stats report about 0.5% or 1500 lines of the code is Rust (which is hilariously less than Assembly, at 0.9%). But, I think I've already shown myself that it's possible to do this kind of incremental porting work from C to Rust. What's done:

* `malloc`/`free` are in pure Rust right now. Wildly unsafe, non-idiomatic Rust, but they work.
* musl's assembly atomic operations are now implemented using inline assembly in Rust (so yes, musl itself calls into the Rust code a lot, not just the test suite).
* `errno` is set using the pthreads TLS which the rest of the library uses (it seems).
* musl's `mmap` syscall wrapper and helpers are in Rust, and called from the pure C musl code, the pure Rust rusl code, and the libc-test code.
* `stpcpy`/`strcmp`/`strcpy`/`strlen` are in Rust, and pass tests.
* `__wake`/`__wait`/`__vm_wait`/`__vm_lock`/`__vm_unlock` syscall wrappers are all in Rust, and called from a variety of musl pthread functions.
* A smattering of `time.h` and `unistd.h` functions are also ported, although test coverage for these does not appear to be great.

All of these are built with generally pretty unsafe code (not just in unsafe blocks, but using lots of pointer offsets and completely opting out of Rust's ownership system). That said, Rust has already forced me to clarify a bunch of ambiguous type casts and other interesting things. I like the explicitness it has over the C with regard to integer and pointer types and operations, although I'm sure that is at least in part because I'm not very familiar with C.

## What have I learned?

A whole bunch. Although, perhaps not as much as I'd hoped. It's very easy to get tunnel vision when porting a single 50-line C function without understanding where it fits in with the overall architecture. I think one thing I'm learning about learning-by-porting is that it requires extra work above and beyond the actual porting work in this case -- a mechanical translation doesn't provide a lot of insight about how something is supposed to work. So I'm learning a lot more about what the allocator does by trying out and throwing away various Rustic changes to it.

I'm also learning **just how monumental** an uphill battle the "rewrite everything in Rust" camp has. musl is a pretty small C project by current \*nix infrastructure standards, and it's a giant undertaking which I will probably never ever finish myself.

However, I do think that rewriting targeted portions of a system in Rust is definitely possible (hell, if I can do it...), and in some cases may be desirable. The talk slides I linked above talk about using a "Golden Master" set of files for testing regressions during rewriting, but I think that I personally wouldn't be very comfortable with doing a C-to-Rust rewrite without either having or building a pretty big regression test suite. Which is of course a catch-22, because if you have a great test suite you may not be writing C code that would benefit as much from a Rust rewrite as other projects. Alas...

## What's down the road?

My next goal (having been occupied with other things for a few weeks) is to improve the memory allocator using more Rustic and safe code. This means using RAII-style locks, less pointer magic, and hopefully building a somewhat typesafe linked list interface instead of the bag of functions that currently operate simultaneously on a single static data structure.

I'm hoping to make use of the growing `#![no_std]` crate ecosystem to replace a bunch of C code without having to write much myself (specifically thinking of the Rust libm and libpnet here, but I'm sure there are others).

As I get around to replacing more of the library, I'd like to experiment more with using internally type-safe interfaces to provide "safer unsafe" interfaces to C code.

## Comments?

Contact links are in the footer below. Would love to hear from anyone about this project or other C-to-Rust efforts, or just stuff in general. I'm usually in `#rust` on irc.mozilla.net as `dikaiosune`.

*Update*: [I posted this to /r/rust](https://www.reddit.com/r/rust/comments/4nqwfg/baby_steps_slowly_porting_musl_to_rust/), and it looks like someone also [submitted it to Hacker News](https://news.ycombinator.com/item?id=11888885), so conversations may be occurring there.
