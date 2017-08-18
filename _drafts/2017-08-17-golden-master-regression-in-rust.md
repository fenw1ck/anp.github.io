---
layout: post
title:  "Snapshots: Automating Golden Master Regression Tests in Rust"
date:   2017-08-17
categories: rust
---

**TLDR**: _I've been playing around with [snapshot](https://github.com/dikaiosune/snapshot-rs), a crate for automating golden master regression tests in Rust. It's experimental and unstable, but I think it's a cool example of how easy it is to build procedural macro helpers with the newer Rust APIs._

---

A few months ago, I was at [React Europe](https://www.react-europe.org/), watching a [talk about JavaScript testing tools](https://www.youtube.com/watch?v=qAZ3_xCHe48). Much of the time I'm at non-Rust programming events, I find myself thinking "Ha! We don't have these problems when using Rust!" But in this particular case, I was quite jealous of a couple of features in Jest that enable JS developers to iterate very quickly with constant feedback from their tests:

1. Jest's [watch mode](http://facebook.github.io/jest/docs/cli.html#watch) which runs tests every time a file in your project is changed, and only runs tests affected by the changed files. This is kind of like Rust's incremental compilation, but for tests. It also comes with a nice TUI that lets you specify filters for test names, manually run particular tests, etc.
2. Jest's [snapshot testing](https://facebook.github.io/jest/docs/snapshot-testing.html) which serializes the output of some value and checks that against a previously-generated value that was written to disk.

While I think someone (maybe me in a far off future) should definitely build #1 for Rust, I've been slowly building a crate for #2 using the new procedural macro APIs in nightly.

## How does it work in this crate?

You define a test that returns a value, like this:

```rust
// tests/simple.rs
#![feature(proc_macro)] // because proc macros aren't stable yet
extern crate snapshot;

mod test {
    use snapshot::snapshot;

    #[snapshot]
    fn simple_snapshot() -> i32 {
        let x = 1;
        x
    }
}
```

The returned value must implement the `Snap` trait, which at the moment has a blanket impl for any type with `serde::Deserialize + serde::Serialize`.

When you run this test while the `UPDATE_SNAPSHOTS` environment variable is set, it will write a file to `PATH_OF_TEST_FILE/__snapshots__/FILENAME_OF_TEST_FILE.snap`. In the above example, it would be written to `tests/__snapshots__/simple.rs.snap`. The contents of that file will be something like this (right now, subject to change):

```json
{
  "simple::test::simple_snapshot": {
    "file": "tests/simple.rs",
    "module_path": "simple::test",
    "test_function": "simple_snapshot",
    "recorded_value": 1
  }
}
```

If you were to change this test function to return `2` instead, and you ran the test again without setting `UPDATE_SNAPSHOTS`, you'd see an error like this:

```
test test::simple_snapshot ... FAILED

failures:

---- test::simple_snapshot stdout ----
	thread 'test::simple_snapshot' panicked at 'assertion failed: `(left == right)`: Test output doesn't match recorded snapshot!

left:  `2`
right: `1`
diff:  ` 2 1 `
```

_(the diff line shows red and green colors in your terminal to show the change)_

This error message definitely needs work! For one thing, it's not clear whether `left` is the recorded value or the newly returned value. But you can see the idea!

## What's left?

The error messages need improvement. I don't think `#[should_panic]` works, but I'm also not sure it's important for it to.

I've started a command line tool to automate updating snapshots interactively. I think that's a really important part of this being useful. To update snapshots right now you have to very carefully only invoke specific tests using the limited filtering capabilities of the Rust test runner. This should definitely be automated, but it's slightly harder because Rust doesn't have machine-readable test harness output, and there's no API (yet) for plugging in your own harness.

Also, it would be very useful to allow having unstructured String values for snapshots in addition to the structured JSON the crate currently writes. I'm not exactly sure how this will work in practice, but suggestions are welcome.

## Snapshot testing sounds dumb

Uh, maybe? It's definitely not very useful for lots of types of testing, but it's really useful for simply guaranteeing that your values-under-test don't change after they've been manually verified by a human. Or when you're refactoring some not-very-easily-tested code and don't want to break anything.

Jest's snapshot tests are what's sometimes called "golden master tests" or "[characterization testing](https://en.wikipedia.org/wiki/Characterization_test)." Obligatory Wikipedia quote:

> The goal of characterization tests is to help developers verify that the modifications made to a reference version of a software system did not modify its behavior in unwanted or undesirable ways. They enable, and provide a safety net for, extending and refactoring code that does not have adequate unit tests.

In the context of UI development, they often fill a slightly different role: requiring conscious sign-off from developers when the output changes from a common but untested component of a system. This can have nice side effects for tooling: you can fail CI if snapshots changed without being committed, and as a result you can also ensure that reviewers see changes to the output of these components without having to spin up a full manual QA environment.

These benefits are predicated on the assumption that you don't already have extensive behavioral or acceptance tests in place for the code in question. This assumption holds pretty often in my experience, and it's not always because of lack of attention or being rushed. My memory is that both rustc and Servo have custom systems for checking test output against files committed in the repository in situations where behavioral tests aren't ideal.

## Snapshot testing sounds less dumb

Cool. If you think you'd use this crate after `$FOO` feature was added, let me know!

## Wait, how did you make the procedural macro?

I followed along with Alex Crichton's work on the [async/await](https://github.com/alexcrichton/futures-await) procedural macros, basically. The `snapshot` crate [exports a separately-defined procedural macro](https://github.com/dikaiosune/snapshot-rs/blob/master/src/lib.rs#L8-L14) and also [defines a bunch of functions](https://github.com/dikaiosune/snapshot-rs/blob/master/src/lib.rs#L30) that we call to in the code that's written out to wrap the test function the user provides.

The actual [procedural macro code](https://github.com/dikaiosune/snapshot-rs/blob/master/snapshot-proc-macro/src/lib.rs) defines an item attribute which panics on anything other than a function definition, parses an AST using [syn](https://github.com/dtolnay/syn), and then writes out a test wrapper with [quote](https://github.com/dtolnay/quote).

In other words, there's nothing super original here, just a nice hack shamelessly adapted from [Alex](https://github.com/alexcrichton)'s work, and enabled by fantastic crates from [dtolnay](https://github.com/dtolnay)!

## Couldn't you have done this with `macro_rules`?

I don't think so, because `concat_idents!` can't actually create new identifiers, which is necessary to have correct backtraces. Also, even if I did, I wouldn't have learned a new Rust API!

## Bye

Thanks for reading! If this seems like something you'd like to use or contribute to, the [GitHub repository](https://github.com/dikaiosune/snapshot-rs) is a good place for that. You can find me on [Twitter](https://twitter.com/dika10sune), or on Mozilla IRC as `dikaiosune`.

I've also posted this to the [Rust subreddit](TODO) for discussion.

