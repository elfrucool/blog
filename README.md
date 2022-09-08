# ABOUT

This is just a collection of articles related to coding.

# ARTICLES

## Monads in go:

[Monads](https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) abstract the context of a computation. Where "context" could be quantity (a list), availability (a thing that can be nil), fallibility (something that, when executed, can produce an error), asynchronous execution (something that is not yet present), etc. And **computation** is just applying a function to the value whithin the context, and the result will be in the same context. Here's the list of _monadic go_:

- [monadic lists](./golang/monadic-lists.md) List is the context that is easier to be used to explain the concept of a monad, so let's use it.

- **monadic optional/nil**(coming soon) It's very common to use a pointer to represent a value that may not exist. In this article I suggest another approach to handle presense/absence of a value.

- **monadic fallibility**(coming soon) The way go handles errors is returning a value of `error` type. It's a common pattern to return `(something, error)` tuple. Unfortunately, tuples are not first-class citizen in go. You can't wrap them on a type. But we can work around this limitation and build a monadic abstraction on top of this pattern.

- **monadic channels**(coming soon)  In this example I'm working with channels + goroutines as asynchronous context. And abstracting them I can offer another way ot thinking in concurrency. I will work with the WebCrawler exercise of [a tour of go](https://go.dev/tour/concurrency/10)

- **state monad in go** This is a sequence of three articles that explain (1) [how to implement the state monad in go](./golang/state-monad.md), (2) an arbitrary [pseudo-random values generator](./golang/random-generator.md), and (3) [property-based testing](./golang/state-monad.md) using the pseudo-random values generator we implemented.
