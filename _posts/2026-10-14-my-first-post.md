---
layout: post
title: "Introducing gtor"
date: 2026-10-15
---

This article presents a case for using generators in Rust as well as a crate that provides the user with the means to implement and consume
them in an intuitive way.

# Use case

Let's say we want have an application that finds quotes in a book and presents them to the user.
The user can provide the name of a character that must be mentioned inside the quote and for each quote that is found the user can decide
whether to keep that quote, discard the quote or abort the search.
A CLI version of this application could look like this:

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

In our application code we have a section where we have the input stream of the book on the one hand and a to-be-written function that
provides quotes from that input stream to the caller.
The caller then presents them to the user and either stores the quote, continues to the next quote or aborts the search.
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
        unimplemented!()
    }
}

// The signature of this function depends on the approach (see the following sections)
fn find_quotes(quote_provider: QuoteProvider /*, ??? */) /* -> ??? */ {
    // ...
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
    unimplemented!()
}

```

So we have `process_quotes()` as the _caller_ and `find_quotes()` as the _callee_.
We'll ignore the other code fragments; they are implementation details for the sake of this article.
The callee has to produce the quotes for the caller to consume.
The number of quotes that will be produced can vary from zero to many, depending on the book that the user provides and the character he
wants to match.
In Rust we have several ways of doing this.
The following sections will look into each approach.

## Callee returns a collection

This will often be the go-to solution for a "return any number of" requirement.
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
Even though we're not storing the whole book in memory and we're doing our best to only select the quotes that mention the target, we still
end up with a huge amount of quotes in that collection.
The application has no way of knowing how many quotes are going to be matched, so this ends up being a potential out-of-memory case, unless
we limit it somehow, but that means that we have an (artificial) cap on the number of quotes, even though the current hardware might still
have space to handle more.
So the problem that we have induced with this approached is that the application has scalability problems, or: collecting all items in a
collection doesn't scale well.

And then there's a second problem with this approach, which is that all quotes get processed from the input stream _before_ the user is
queried.
If the user decides to abort on the 3rd quote and we have 1000 quotes, then that means we processed 997 quotes too many.
When reading from a local SSD this will be likely be unnoticable but if the book is streamed from a slow network connection the user will
end up waiting pointlessly.

And there are more problems with this approach that are not applicable to this example, but are mention-worthy:

- This approach per-definition does not work with an infinite stream.
- All code paths must return the _same_ `Iterator` implementation.
  - `-> impl Iterator` only _hides_ the implementation from the caller; it does not serve as an abstraction.
    So if the function wants to return a `Vec` in one code branch and a `HashSet` in another, we'll need either dynamic dispatch
    (`Box<dyn Iterator>`) or some kind of static dispatch, for example with an `enum` that can hold any variant and implements `Iterator`
    itself.
- If this function internally calls another function that uses the same approach, we end up with a lot of temporary collections being
  collected into other collection, causing a cascade of the same problem down the code path.

## Using Callbacks

So we don't want to collect everything, but instead pass the `Quote` objects to the caller as they become available.
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
value is `false`.
So let's just do that then quickly:

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
And the code just has some serious readability problems.

Let's tell just `find_quotes()` when we want to abort via our trusty callback, instead:

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
program.
It doesn't help that the return type doesn't express the _intent_ and it can easily be misused if you flip the meaning
(`continue_processing: bool` vs `abort_processing: bool`).

It turns out that callbacks work well when the caller produces something for the callee (e.g. a lazy-constructed object, a mapped type or a
filtering decision).
But when the callee produces something for the caller via a callback things start to fall apart.
It looks like the reason for this is that the recipient of the produced value will often want to perform some kind of follow-up action with
the result (in this case even possibly aborting the application), which a callback naturally prevents the caller from carrying out, since
the control flow is on the callee's side: the caller can not "escape" from the callback.

Another problem with this approach is when the application logic gets more complicated and we add a function "between" `process_quotes()`
and `find_quotes()`.
Let's call this function `search_for_quotes()`.
`search_for_quotes()` would get called by `process_quotes()`, which would call its callback function in.
Then `search_for_quotes()` has to take that callback and pass it into `find_quotes()`, possibly wrapped with its own lambda expression that
adds more logic and thus complexity.

And things get even worse when callbacks get passed that are not expressed as lambdas, but instead call into a completely separate piece of
code.
In that case the reader has no immediate indication of what happens when and reasoning about the code becomes a nightmare.

If you take one thing away from this article (for any programming language that supports callbacks): callbacks only topass information from
the caller to the callee, never the other way around.

So we're concluding that we want the caller to be in control of the program flow.
In other words: the caller wants to _pull_ the quotes, instead of them being _pushed_.
But we don't want to collect everything first.
Doesn't Rust provide this trait for that?
What was it, again...

## Implementing Iterator

Ah yes, we've seen it before.
Our good friend `Iterator`.
We can just take out state and implement `Iterator<Item=Quote>`.
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

Wow, that was unexpectedly easy.
We have found the holy grail!
Surely, the time has come to frolic joyfully in the yard, at last relieved.

Well, hang on there, cowboy/cowgirl/cowhand/cowpoke/buckaroo.
This works, it is true.
But it would be wise to note that it really only works this well because `QuoteProvider` already provides `next_quote()`, which is very
`Iterator`-compatible out-of-the-box and we're not doing much else.
Here is a more involved example:

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

Now, when looking at the `Iterator::next()` of `QuoteIter` it becomes a lot less intuitive what this code actually does.
And then there's the `filter_map()` on that iterator, which in itself is surprisingly ergonomic (hurray for Rust iterators).
But still it splits the logic in two.
We could pass the `speaker_filter` into `QuoteIter`, but then we need a generic parameter to capture the type and we need trait bounds on
`Iterator::next()` just to get started with the code.
I doubt we're getting closer to readable code with such changes.

One side note on the the splitting of the function (which is supposed to entail the entire implementation) and the iterator
implementation...
Say we have a struct that has a bunch of logic implemented over different methods.
This bunch can be thought of as a "unit".
In Rust (and many other languages) methods are grouped together with the type to emphasize this unity.
But if one method has to provide items based on the state in the type and we implement that in a separate `Iterator` type, we break this
natural unity.
Sometimes this is OK or even natural, but oftentimes it is not.

And even with our "convenient" example above, one could argue that the implementation involves a lot of boiler plate code.
If a developer has to do this kind of thing a lot in the code base, they will surely be tempted to reach for the callback or collection
approach after having jumped through this hoop a few times.

And one more nitpick about going through `Iterator` is that it gets awkward with fallible operations.
Say that our `InputStream` can fail (which it should actually do in the real world if we want to support streaming).
Then that means that `QuoteProvider::next_quote()` is also fallible.
But if we want to implement `Iterator` around a fallible operation, we end up with `type Item = Result<Quote>`, which means the iterator
returns `Option<Result<Quote>>`.
A more appropriate type would be `Result<Option<Quote>>`, since we can only evaluate the "optionality" once we've managed to read the input
without failure.
But this is more a shortcoming of the `Iterator` trait and Rust's commitment to this trait when it comes to `for`-loop ergonomics. (Try to
implement an iterator in C++ and you'll quickly stop complaining about these kind of details with any kind of commitment.)

Maybe we'll have to look beyond what we known as `stable`...

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
Then it will execute the iteration of the `for`-loop (in this case printing the number) and then resume the state of `count_up_to_5()` after
the `yield i` point, which will cause another iteration and then yield the value `2`, which then gets appled to the `number` variable, etc.
This continues until `count_up_to_5()` reaches the end of its block, causing the lower `for` loop to terminate.

This makes it considerably more ergonomic to express more complicated work flows than the `Iterator` approach earlier.
So let's all switch to Python.

Don't.

The grass is not greener.
It is actually infested with constricting snakes.

It turns out that Rust also has generators.
Let's use one for our problem:

(TODO)

One tiny litte caveat: generators are is not available in `stable` Rust, so we can practically not use them.

- Not supported in stable Rust.
- The input feature is a bit yank.

# Gtor

- Observation: `async` already generates a state machine from normal imperative code.
- Solution: leverage the compiler's ability to generate a state machine and build utility around it to pass values between caller and
  callee.

## Usage

- `#[generator]`
- Pinning.
- Different types of generators (table with finite-no-result/finite-with-result/infinite + iteration modes).

## Under the hood

(not sure about this section... might not be too useful... instead just explain briefly during `#[generator]`)

## Why no input

- Complicates the `gtor` design.
- It is usually better to model an explicit state machine
  - One generator tends to grow into a large function where the disadvantages outweigh the benefits.
  - Each state can return multiple items using `#[generator]`.
- TODO: Some kind of language that models a state machine and generates code (first Rust, then C++).
