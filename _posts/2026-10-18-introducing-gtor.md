---
layout: post
title: "Introducing gtor"
date: 2026-10-18
author: knonderful
---

* TOC
{:toc}

# Introduction

This article presents a case for using generators in Rust and offers the [`gtor`](https://crates.io/crates/gtor) crate as an ergonomic and
intuitive approach for both implementation and consumption of generators.

# Use case

Let's say we want have an application that searches for quotes in a book and presents them to the user.
The user can provide the name of a character that must be mentioned in the quote and for each match the user is asked whether to keep that
quote, discard the quote or abort the search.
A CLI version of this application could look like something like this:

```
$ quote_find --in lotr.txt --mentioning Frodo

Bilbo:
  "You had better come and live here, Frodo my lad, and then we can celebrate
   our birthday-parties comfortably together."
[K]eep / [D]iscard / [A]bort? _

Gaffer:
  "Small wonder that trouble came of it, I say. But be that as it may, Mr.
   Frodo is as nice a young hobbit as you could wish to meet."
[K]eep / [D]iscard / [A]bort? _
```

Let's say that in our application code we have a section where we have the input stream of the book on the one hand and a to-be-written
function that provides quotes from that input stream to the caller.
The caller then presents each quote to the user and either stores the quote, continues to the next quote or aborts the search.
In Rust this could look something like this:

```rust
struct Quote {
    speaker: String,
    content: String,
}

struct InputStream; // -> omitted as implementation detail

struct QuoteProvider<'a> {
    input: InputStream,
    mentioning: &'a str,
}

impl QuoteProvider<'_> {
    fn next_quote(&mut self) -> Option<Quote> {
        // Read next quote from input stream and match with `self.mentioning` -> omitted as implementation detail
        // -> omitted as implementation detail
    }
}

// The signature of this function depends on the approach we take (see the following sections)
fn find_quotes(quote_provider: QuoteProvider /*, ??? */) /* -> ??? */ {
    // To be implemented...
}

fn process_quotes(quote_provider: QuoteProvider) {
    // 1. Use find_quotes() to get quotes
    // 2. Pass to each quote to handle_quote()
    // 3. Continue on `true`, return on `false`
}

#[must_use] // <- because this controls the application flow; it may not be ignored
fn handle_quote(quote: Quote) -> bool {
    // 1. Print quote
    // 2. Ask for decision
    // 3. Store quote, if desired
    // 4. Return `true` to continue and `false` for abort
    // -> omitted as implementation detail
}

```

So we have `process_quotes()` as the _caller_ and `find_quotes()` as the _callee_.
We'll ignore the other code fragments; they are not relevant for the purpose of this article.
The callee produces the quotes for the caller to consume.
The number of quotes that will be produced can vary from zero to many, depending on the size of the book and the provided character name.
In Rust we have several ways of doing this.
The following sections will look into each approach.

## Callee returns a collection

This will often be the go-to solution for a "return any number of" requirement in most programming languages.
Let's look at the implementation:

```rust
fn find_quotes(mut quote_provider: QuoteProvider) -> impl Iterator<Item=Quote> {
    let mut quotes = Vec::new();
    while let Some(quote) = quote_provider.next_quote() {
        quotes.push(quote);
    }
    quotes.into_iter()
}

fn process_quotes(quote_provider: QuoteProvider) {
    for quote in find_quotes(quote_provider) {
        if !handle_quote(quote) {
            return;
        }
    }
}
```

Cool, that works.
We're done.

But wait a minute...
What if the user provides a giant book where the target character is mentioned all over the place?
The result could be a collection of virtually endless dimensions, causing out-of-memory issues.
We could enforce a limit on the number of quotes somehow, but that means that we have an (artificial) cap that hampers the application's
usage.
Ultimately, collecting all items in a collection doesn't scale well.

And then there's a second problem with this approach, which is that all quotes get processed from the input stream _before_ the user is
queried.
If the user decides to abort on the 3rd quote and we have 1000 quotes, then that means we unnecessarily processed 997 quotes.
When reading from a local SSD this will be likely be unnoticable but if the book is streamed from a slow network connection the user will
end up waiting pointlessly.

And there are more problems with this approach that are not applicable to this example, but are mention-worthy:

- This approach per-definition does not work with an infinite stream.
- All code paths must return the _same_ `Iterator` implementation.
  - `-> impl Iterator` only _hides_ the implementation from the caller; it does not serve as an abstraction for monomorphization.
    So if the function wants to return a `Vec` in one code branch and a `HashSet` in another, we'll need either dynamic dispatch
    (`Box<dyn Iterator>`) or some kind of static dispatch, for example with an `enum` that can hold any variant and implements `Iterator`
    itself.
- If this function internally calls another function that uses the same approach, we end up with a lot of temporary collections being
  collected into other collection, causing a cascade of the same problem down the code path.

So we don't want to collect everything, but instead pass the `Quote` objects to the caller as they become available (and discard them when
they are no longer needed).

## Using Callbacks

We could say the callee can call the caller back when it has something to be called back about...
Hey, we could use a callback!

```rust
fn find_quotes(mut quote_provider: QuoteProvider, callback: impl Fn(Quote)) {
    while let Some(quote) = quote_provider.next_quote() {
        callback(quote);
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    find_quotes(quote_provider, |quote| {
        handle_quote(quote);
    });
}
```

Excellent!
Push it to production.

Oh wait, what does the CI pipeline say?

```
warning: unused return value of `handle_quote` that must be used
  --> src/callback.rs:44:9
   |
44 |         handle_quote(quote);
   |         ^^^^^^^^^^^^^^^^^^^
```

How cruel.
But now that I think of it, we _did_ mark `handle_quote()` with `#[must_use]` for a reason, since we must abort processing when the return
value is `false`, so we must not ignore the return value.
So let's just do that then with a swift edit:

```rust
fn process_quotes(quote_provider: QuoteProvider) {
    find_quotes(quote_provider, |quote| {
        if !handle_quote(quote) {
            return; // Ehhhh, nope. This just returns from the callback; `find_quotes()` will still be running.
        }
    });
}
```

That doesn't work, so let's try again:

```rust
fn find_quotes(mut quote_provider: QuoteProvider, mut callback: impl FnMut(Quote)) {
    while let Some(quote) = quote_provider.next_quote() {
        callback(quote);
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    let mut do_handle = false;
    find_quotes(quote_provider, |quote| {
        if !do_handle {
            return;
        }
        do_handle = handle_quote(quote);
    });
}
```

That kind of works, but now when the user tells us to abort at quote number 3, but he still has to wait for the remaining 997 quotes to be
processed.
And the code also has introduced some readability problems.

Let's just use our trusty callback to inform `find_quotes()` when we want to abort:

```rust
fn find_quotes(mut quote_provider: QuoteProvider, callback: impl Fn(Quote) -> bool) {
    while let Some(quote) = quote_provider.next_quote() {
        if !callback(quote) {
            return;
        }
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    find_quotes(quote_provider, |quote| {
        handle_quote(quote)
    });
}
```

While this does work efficiently and without processing anything we don't need to, it has some architectural flaws.

Firstly, our `callback` now has gone from one responsibility (or concern) to two: _handle quotes_ and _control the application flow_.
While this may be clear when you write the code, it will quickly get confusing when you read this kind of code in the context of an entire
program, especially if there are multiple such constructs at work at the same time.
It doesn't help that the return type doesn't express the _intent_ and it can easily be misused if you flip the meaning
(`continue_processing: bool` vs `abort_processing: bool`).
We could introduce a dedicated enum for this, but in practice we often see developers "doing the `bool`".

It turns out that callbacks work well when the caller produces something for the callee (e.g. a lazy-constructed object, a mapped type or a
filtering decision).
But when the callee produces something for the caller via a callback things start to fall apart[^1].

Another problem with this approach is when the application logic gets more complicated and we add a function "between" `process_quotes()`
and `find_quotes()`.
Let's call this function `search_for_quotes()`.
`process_quotes()` calls `search_for_quotes()`, passing its callback function in.
Then `search_for_quotes()` has to take that callback and pass it into `find_quotes()`, possibly wrapped with its own lambda expression that
adds more logic and thus complexity.
The problem here is that there is not clear visual clue that `find_quotes()` is calling back up to `process_quotes()`.
We are used to looking at parameters passed in a "downward" manner: a function call gets a parameter passed and it may use that "thing".
We don't naturally consider that that "thing" influences the code in the current function from 2 or three layers deeper in the call stack.

And things get even worse when callbacks get passed that are not expressed as lambdas, but instead call into a completely separate piece of
code.
In that case the reader has no immediate indication of what happens when and reasoning about the code becomes a nightmare.

One thing to take away from all this (for any programming language that supports callbacks): use callbacks only to pass information
from the caller to the callee; never the other way around.

So we're concluding that we want the caller to be in control of the program flow.
In other words: the caller wants to _pull_ the quotes, instead of them being _pushed_.
But we don't want to collect everything first.
Doesn't Rust provide this trait for that?
What was it, again...

## Implementing Iterator

Ah yes, we've seen it before.
Our good friend `Iterator`.
We can just take our state, stick it in a `struct` and implement `Iterator<Item=Quote>`.
And instead of that iterator being based on a collection, we dynamically pull the next quote and return it.
Maximum efficiency.
Let's do it:

```rust
struct QuoteIter<'a> {
    quote_provider: QuoteProvider<'a>,
}

impl Iterator for QuoteIter<'_> {
    type Item = Quote;

    fn next(&mut self) -> Option<Self::Item> {
        self.quote_provider.next_quote()
    }
}

fn find_quotes(quote_provider: QuoteProvider<'_>) -> impl Iterator<Item=Quote> + '_ {
    QuoteIter { quote_provider }
}

fn process_quotes(quote_provider: QuoteProvider) {
    for quote in find_quotes(quote_provider) {
        if !handle_quote(quote) {
            return;
        }
    }
}
```

Wow, that was unexpectedly easy, albeit rather verbose.
We have found the holy grail!
Surely, the time has come to frolic joyfully in the yard, at last relieved.

Well, hang on there, cowboy/cowgirl/cowhand/cowpoke/buckaroo.
This works, it is true.
But it would be apt to note that it really only works this well because `QuoteProvider` already provides `next_quote()`, which is a very
`Iterator`-compatible function and other than that we have nothing much else to do.

So let's push a little harder with a more involved example:

```rust
fn get_internet_quotes() -> Vec<Quote> {
    unimplemented!() // -> omitted as implementation detail
}

struct QuoteIter<'a> {
    internet_quotes: IntoIter<Quote>,
    quote_provider: QuoteProvider<'a>,
    quote_provider_consumed: bool,
}

impl Iterator for QuoteIter<'_> {
    type Item = Quote;

    fn next(&mut self) -> Option<Self::Item> {
        if !self.quote_provider_consumed {
            if let Some(item) = self.quote_provider.next_quote() {
                return Some(item);
            }
            self.quote_provider_consumed = true;
        }
        self.internet_quotes.next()
    }
}

struct QuoteProviderIter<'a> {
    quote_provider: QuoteProvider<'a>,
}

fn find_quotes<'a>(
    quote_provider: QuoteProvider<'a>,
    speaker_filter: impl Fn(&str) -> bool + 'a,
) -> impl Iterator<Item=Quote> + 'a {
    // Let's say our implementation must (in order):
    // - Pull some quotes off "the internet".
    //   - Let's say this has to happen _before_ the next step because the internet closes at 4 P.M.
    // - Return all quotes from the QuoteProvider.
    //   - When None is returned the iteration is done.
    //   - Added bonus quirk: QuoteProvider specifies that next_quote() may not be called after None
    //     has been returned or it will panic. <3
    // - Return all internet quotes.
    // - For all quotes: only return those for which `speaker_filter` returns true
    QuoteIter {
        quote_provider,
        internet_quotes: get_internet_quotes().into_iter(),
        quote_provider_consumed: false,
    }
        .filter_map(move |quote| {
            if !speaker_filter(&quote.speaker) {
                return None;
            }
            Some(quote)
        })
}

fn process_quotes(quote_provider: QuoteProvider) {
    for quote in find_quotes(quote_provider, |speaker| speaker.starts_with("Darth")) {
        if !handle_quote(quote) {
            return;
        }
    }
}
```

Now, when looking at the `Iterator::next()` of `QuoteIter` it becomes a lot less intuitive what this code actually does (or _should_ do).
And then there's the `filter_map()` on that iterator, which in itself is surprisingly ergonomic (hurray for Rust iterators).
But still it splits the logic in two.
We could pass the `speaker_filter` into `QuoteIter`, but then we need a generic parameter to capture the type and we need trait bounds on
`Iterator::next()` just to get started with the code.
I doubt we're getting closer to readable code with such changes.

And even with our "convenient" example above, one could argue that the implementation involves a lot of boiler plate code.
If a developer has to apply this kind of of practice in multiple places in the code base, they will surely be tempted to reach for the
callback or collection approach before long.

And one more nitpick about going through `Iterator` is that it gets awkward with fallible operations.
Say that our `InputStream` can fail (which it should actually do in the real world if we want to support streaming).
Then that means that `QuoteProvider::next_quote()` is also fallible.
But if we want to implement `Iterator` around a fallible operation, we end up with `type Item = Result<Quote>`, which means the iterator
returns `Option<Result<Quote>>`.
A more appropriate type would be `Result<Option<Quote>>`, since we can only evaluate the "optionality" once we've managed to read the input
without failure.
But this is more a shortcoming of the `Iterator` trait and Rust's commitment to this trait when it comes to `for`-loop ergonomics. (Try to
implement an iterator in C++ and you'll quickly stop complaining about these kind of details with any kind of commitment.)

Maybe we'll have to look beyond our regular Rust capabilities, then.

## Using a Generator

Some programming languages have this feature called "co-routines" or "resumable functions" or "generators".
The key characteristic that sets them apart from normal functions is that they can be suspended at a point of choosing (by the callee) and
can then later be resumed (by the caller).

```python
def count_up_to_5():
    for i in range(1, 6):
        yield i

for number in count_up_to_5():
    print(number)
```

In this Python example we see `count_up_to_5()` being called in a `for`-loop.
The Python run-time will run the code of `count_up_to_5()` up to `yield i`, then take the `i` and apply it to `number`.
Then it will execute the iteration of the bottom `for`-loop (in this case printing the number) and then resume the state of
`count_up_to_5()` after the `yield i` point, which will cause another iteration and then yield the value `2`, which then gets appled to the
`number` variable, etc.
This continues until `count_up_to_5()` reaches the end of its block, causing the bottom `for` loop to terminate.

This makes it considerably more ergonomic to express more complicated work flows than the `Iterator` approach earlier.
So let's all switch to Python.

Don't.

The grass is not greener.
It is actually infested with constricting snakes.

It turns out that Rust also has generators (renamed to `coroutines` at some point).
Let's use one for our problem:

```rust
use std::ops::{Coroutine, CoroutineState};
use std::pin::pin;

fn find_quotes(mut quote_provider: QuoteProvider<'_>) -> impl Coroutine<Yield=Quote, Return=()> + use < '_ > {
    #[coroutine]
    move || {
        while let Some(quote) = quote_provider.next_quote() {
            yield quote;
        }
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    let mut generator = pin!(find_quotes(quote_provider));
    loop {
        match generator.as_mut().resume(()) {
            CoroutineState::Yielded(quote) => {
                if !handle_quote(quote) {
                    break;
                }
            }
            CoroutineState::Complete(()) => break,
        }
    }
}
```

That is not too far from the ergonomics of the collection solution, but we can do better.
The `Coroutine` trait can actually be made to fit into `Iterator` if it doesn't produce a result:

```rust
use std::ops::{Coroutine, CoroutineState, Deref, DerefMut};
use std::pin::{pin, Pin};

struct CoroIter<T>(T);

impl<C> Iterator for CoroIter<Pin<C>>
where
    C: Deref + DerefMut,
    <C as Deref>::Target: Coroutine<(), Return=()>,
{
    type Item = <<C as Deref>::Target as Coroutine>::Yield;

    fn next(&mut self) -> Option<Self::Item> {
        match self.0.as_mut().resume(()) {
            CoroutineState::Yielded(value) => Some(value),
            CoroutineState::Complete(()) => None,
        }
    }
}

fn find_quotes(mut quote_provider: QuoteProvider<'_>) -> impl Coroutine<Yield=Quote, Return=()> + use < '_ > {
    #[coroutine]
    move || {
        while let Some(quote) = quote_provider.next_quote() {
            yield quote;
        }
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    for quote in CoroIter(pin!(find_quotes(quote_provider))) {
        if !handle_quote(quote) {
            break;
        }
    }
}
```

As opposed to our earlier `Iterator` implementation attempts, `CoroIter` actually is general-purpose and can be used for any coroutine that
produces `()` as a result.

There is only one tiny litte caveat: generators are is not available in `stable` Rust, so we can practically not use them.

Before we move on, what is with the `pin!()` and how Rust actually implements these co-routines?
I'm glad you asked both these questions at the same time, because they are related.
You see, Rust generates a state machine out of the expression following `#[coroutine]`.
The state machine gets polled via `Coroutine::resume()`, which effectively advances the state and delivers results to the poller.
If that sounds familiar, it is probably because that essentially identical to how `async` and `Future::poll()` work.
We'll get back to that.

As for the `pin!()`: the generated state machine contains references (pointers) to itself.
That implies that if the developer would be allowed to move the future or co-routine around in memory, advancing the state after such a move
is effectively a use-after-move, which is undefined behavior.
Which is bad.
Since Rust does not provide a direct way to prohibit an instance of a type from being moved, the `Pin<T>` type was introduced.
Once an object is pinned it can not be moved anymore.
The final key to the puzzle is both `Future` and `Coroutine` requiring a `Pin<&mut Self>` for their state-advancing operations, rather than
a `&mut self`.
The developer can pin an object either on the stack with `pin!()` or on the heap with `Box::pin()`.

Let's move on then.

# Gtor

Enter `gtor`.
The aim of `gtor` is to provide ergonomic generators (both for implementor and consumer) that have a sufficiently small footprint that they
can be used without too much concern in terms of performance.

Let's see what that looks like:

```rust
use gtor::generator;
use std::pin::pin;

#[generator(yield_type = Quote)]
fn find_quotes<'a>(mut quote_provider: QuoteProvider<'a>) {
    while let Some(quote) = quote_provider.next_quote() {
        yield_value!(quote);
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    for quote in pin!(find_quotes(quote_provider)) {
        if !handle_quote(quote) {
            break;
        }
    }
}
```

That looks good.
There is basically no noise in that code, except for the `#[generator]` annotation.
We need to pin the generator for the same reason that we needed to do it with the co-routine code.

Note that we explicitly declared the lifetime `'a` here, even though Rust can technically elide that lifetime.
This is due to limitations in the `#[generator]` macro implementation.
See the related documentation for more information (if you care).

Let's go a step beyond and test it with our (contrived) example from the "Implement Iterator" approach:

```rust
#[generator(yield_type = Quote)]
fn find_quotes<'a, F>(mut quote_provider: QuoteProvider<'a>, speaker_filter: F)
where
    F: Fn(&str) -> bool,
{
    // - Pull some quotes off "the internet".
    //   - Let's say this has to happen _before_ the next step because the internet closes at 4 P.M.
    let internet_quotes = get_internet_quotes();

    // - Return all quotes from the QuoteProvider.
    //   - When None is returned the iteration is done.
    //   - Added bonus quirk: QuoteProvider specifies that next_quote() may not be called after None
    //     has been returned or it will panic. <3
    while let Some(quote) = quote_provider.next_quote() {
        if speaker_filter(quote.speaker.as_str()) {
            yield_value!(quote);
        }
    }

    // - Return all internet quotes.
    // - For all quotes: only return those for which `speaker_filter` returns true
    for quote in internet_quotes {
        if speaker_filter(quote.speaker.as_str()) {
            yield_value!(quote);
        }
    }
}

fn process_quotes(quote_provider: QuoteProvider) {
    for quote in pin!(find_quotes(quote_provider, |speaker| speaker.starts_with("Darth"))) {
        if !handle_quote(quote) {
            break;
        }
    }
}
```

This is much nicer an more intuitive than `QuoteIter`.

Note that we can't use `impl Trait` in `#[generator]` for the same reason that we have to use explicit lifetimes for this macro.

## Features

`gtor` creates generators from any standard Rust function (not `const`/`unsafe`/`async`/`extern`).
Every generator yields a user-defined type (let's call this `Y`).
Additionally, a generator can specify the return type.

| Return Type    | Boundedness | Iterator Item Type     | Remark                                            |
|----------------|-------------|------------------------|---------------------------------------------------|
| `()`           | Finite      | `Y`                    |                                                   |
| `T` (not `()`) | Finite      | `Yield(Y)`/`Return(T)` | Use `loop` syntax if the result must be retained. |
| `!`            | Infinite    | (not available)        | Call `next_value()` for iteration.                |

We've already seen examples of generators that return `()`.
Let's take a look at the other two options.

```rust
#[generator(yield_type = u32)]
fn with_result() -> String {
    for i in 1..8 {
        yield_value!(i);
    }
    "We are done.".to_string()
}

fn consume() {
    let mut generator = pin!(with_result());
    let message = loop {
        match generator.as_mut().next_item() {
            GeneratorItem::Yield(yielded) => println!("Yielded {yielded}"),
            GeneratorItem::Return(message) => break message,
        }
    };
    println!("Message: {message}");
}
```

In the example above we use Rust's `break` with return value for `loop` to capture the result without the need for something like
`Option<T>` (accompanied with the usual unwrapping).
This is useful when the code after the generator consumption needs the result value.
But often the result will be something like `Result<(), E>` and the caller just wants to act on the error, if it exists.
For such cases a `for`-loop is more convenient:

```rust
#[generator(yield_type = u32)]
fn with_fallible(until: u32) -> Result<(), String> {
    for i in 1..until {
        yield_value!(i);
        if i > 100 {
            return Err("You can't do that".to_string());
        }
    }
    Ok(())
}

fn consume() -> Result<(), String> {
    for item in pin!(with_fallible(101)) {
        match item {
            GeneratorItem::Yield(yielded) => println!("Yielded {yielded}"),
            GeneratorItem::Return(result) => return result.map_err(|e| format!("Could not consume: {e}")),
        }
    }

    Ok(())
}
```

Now for the return type `!`.
In Rust this means that there is no return value for the function.
In this case it means that the generator is infinite; it will always yield another value and is never finished.
Let's look at an example:

```rust
#[generator(yield_type=usize)]
fn id_generator() -> ! {
    let mut current = 0;
    loop {
        current += 1;
        yield_value!(current);
    }
}

fn consume() {
    let mut generator = pin!(id_generator());
    println!("Player #1 gets ID {}", generator.as_mut().next_value());
    println!("Player #2 gets ID {}", generator.as_mut().next_value());
    println!("Player #3 gets ID {}", generator.as_mut().next_value());
    println!("Player #4 gets ID {}", generator.as_mut().next_value());
}
```

First, this type of generator does not implement `Iterator`.
While this would be technically possible, `gtor` explicitly opts out of this because developers will intuitively assume that a `for` loop
will eventually end, which would not be the case.
Additionally, this is supported by this idea that infinite generators will probably be useful for generation of some counter value, unique
IDs or some randomized number.
Most of these use cases will not want to loop on the generator, but rather have some other mechanism like a game loop or an input stream
that drives the pulling of the value.

Secondly, `Iterator` returns `Option<T>`, but we already know that our values are not optional; if the generator is polled, there will be
value.
This alleviates the caller from having to do the unwrapping and decreases complexity on the calling side.

## Performance

`gtor` generators are _stackless_, which means that they don't store any data beyond what they need for their internal state machine.
Generators don't use any run-time, threads or interaction with the operating system; it is all pure Rust.
Performance has not been tested yet, but it should be pretty snappy.

## Design

The main feature that is required for a generator to work is that it can suspend, return a value, and then be resumed from that point.
Crates like  [`genawaiter2`](https://crates.io/crates/genawaiter2) (also stackless) also support this.
Unlike `gtor` these crates base generators on lambda expression rather than functions.

The reasoning behind `gtor` is that a generator is more a "behavioral characteristic" than an object.
An object is often driven by the data that it encapsules and then we add some methods to interact with that data.
But for generators we don't really care about the data inside the generator; it is the behavior that we are interested in.
The same kind of reasoning can be applied to `async` functions.
Therefor, it makes sense to apply the "generatorism" on a function rather than a local object, since functions also have this
behavior-focus.
An example of this is the use case described in this article: "the callee produces the quotes for the caller to consume".
We start with a callee (a function), since that is how software expresses behavior.

### Implemented on Async

`gtor` leverages the Rust compiler's ability to generate a state machine for an `async` function.
Each `#[generator]` function is transformed into an `async` block with the resulting `Future` being contained in `Generator`.
When the generator is advanced the future is polled.
The future receives a context object which can be used to store the yield.
Every `yield_value!()` gets transformed into a call to the context object with the yield.
This call is then `.await`ed, causing the `Future` to be suspended.
The machinery around the future pulls the value from the context and passes it to the calling code.

The method for yielding a value to the context is marked `unsafe`, since the caller must ensure that non of its constraints are violated,
which could result in various kinds of undefined behavior.
When just working with the intended mechanics of `#[generator]` the user is not exposed to this and no `unsafe` code has to be called.

### No Sending of Values

Many co-routine/generator crates support passing values into the generator, based on which the generator can base its response.
`gtor` (currently) explicitly does not support this.
This kind of thing can be implemented in different ways, none of which are a fit for every use case and it all complicates the solution.
Additionally, it seems like this kind of "need" takes a step in the direction of state machine programming for which a function (or a lambda
expression, for that matter) is not a good option, since it tends to end up with a long function with a lot of branching, the disadvantages
quickly outweighing the advantages.
Things really bad if the state machine has arrows that point back to a previous state.
In such cases it is often better to express the different states and their transitions explicitly, rather than them being implicit as part
of your function.
Generators can still be used to ergonomically return any number of items "on top" of this state machine machinery.

------

[^1]: It looks like the reason for this is that the recipient of the produced value will often want to perform some kind of follow-up action
with
the result (in this case even possibly aborting the application), which a callback naturally prevents the caller from carrying out, since
the control flow is on the callee's side: the caller can not "escape" from the callback.
