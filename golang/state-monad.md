# State Monad in golang


[Monads](https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) abstract the context of a computation. Where "context" could be quantity (a list), availability (a thing that can be nil), fallibility (something that, when executed, can produce an error), asynchronous execution (something that is not yet present), etc. And **computation** is just applying a function to the value whithin the context, and the result will be in the same context. Here's the list of _monadic go_:

I strongly recommend you to read the following article if you still have not done it yet:

- [Functors, Applicatives, And Monads In Pictures](https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)

One of the most interesting monads is State[^1][^2] monad, it encapsulates stateful computations in a functional way. I.e. using functions without side effects.

[^1]: https://blog.ploeh.dk/2022/06/20/the-state-monad/
[^2]: https://www.infoq.com/articles/Understanding-Monads-guide-for-perplexed/

Here **in this article**, we are going to implement it in golang, the concept is relatively simple but powerful.

On a **second article**(coming soon), we are going to apply it to implement pseudo-random generators, not just numeric random generators, but to generate any kind of random data: numbers, chars, strings, lists, custom records, etc. And to do some transformations on them.

On a **third article**(coming soon), we will use our functional pseudo-random generator to a special kind of tests named [property-based testing](https://medium.com/criteo-engineering/introduction-to-property-based-testing-f5236229d237). This approach of testing is very powerful, and the implementation is not hard.

---

## REQUISITES

We will require go programming language --duh-- minimum version 1.18 because we want to use [generics](https://go.dev/doc/tutorial/generics).

## STARTING FROM SCRATCH

Let's start creating a new go project on an empty directory.

```sh
go mod init example.com/state-monad
```

So now we have a file named `go.mod` with contents:

```
module example.com/state-monad

go 1.18
```

Now let's create a directory for our package:

```sh
$ mkdir state
```

## DEFINITION OF STATE

A state of type `A` (generic) is just a function that, when evaluated, takes an initial state `s0` of type `S` and returns a tuple `(s1 S, a A)`:

```go
// state/state.go
package state

type State[S, A any] func(s0 S) (s1 S, a A)
```

So we can have a state that takes a number, produces a greet with it, and increments the number:

```go
// main.go
import "example.com/state-monad/state"

// ...

var sGreet state.State[int, string] = func (s0 int) (int, string) {
    return (s0 + 1, fmt.Sprintf("hello-%d", s0))
}
```

And we can evaluate it this way:

```go
// main.go
s1, greet := sGreet(10)
fmt.Printf("(%d, %s)\n", s1, greet) // will print "(11, hello-10)"
```

To make it more comfortable, we can have a function that ignores the returning state:

```go
// state/state.go
func RunState[S, A any](st State[S, A], s0 S) A {
    _, a := st(s0)
    return a
}
```

So going back to our example:

```go
msg := state.RunState(sGreet, 10)
fmt.Println(msg) // will print "hello-10"
```

Until this point, it doesn't seem to be very useful. Let's introduce some operations on top of this `State[S,A]` type.


## STATE AS A FUNCTOR

To transform what the state returned into another value we just have to implement a function `Map` on this way:

```go
// state/state.go

func Map[S, A, B any](sa State[S, A], f func(A) B) State[S, B] {
    return func(s0 S) (S, B) {
        s1, a := sa(s0)
        return s1, f(a)
    }
}
```

For example, if we took our original `sa` variable that increments the numeric state and produces greets, but we want to produce values that are the length of the greet, we can do this way:

```go
// main.go

//...

var sLength State[int, int] = state.Map(sGreet, func (s string) int {
    return len(s)
})

//...

l := state.RunState(sLength, 10)
fmt.Printf("%d\n", l) // will print "8"
```

Ok, now we can produce greets, and we can produce lengths of greets or any other transformation on the `A` value of a state evaluation.

But only one value at a time. Now, let's suppose we want to combine two stateful computations and produce a single stateful result, how can we do that? continue reading.

## APPLICATIVE STATE

Applicatives are functors that can be combined, in case of State, the way is this:

```go
// state/state.go

func Map2[S, A, B, C any](sa State[S, A], sb State[S, B], f func (A, B) C) State[S, C] {
    return func(s0 S) (S, C) {
        s1, a := sa(s0)
        s2, b := sb(s1)
        return s2, f(a, b)
    }
}
```

Applicatives also have a function to put values inside them:

```go
// state/state.go

func Pure[S, A any](a A) State[S, A] {
    return func(s0 S) (S, A) {
        return s0, a
    }
}
```

What can we do with them? we can combine the greeter and the greet length states to produce a combined result:

```go
// main.go

// ...

type GreentAndLength struct {
    Greet string
    Length int
}

var sGreetAndLength state.State[int, GreentAndLength] = state.Map2(
    sGreet,
    sLength,
    func (s string, n int) GreentAndLength {
        return GreentAndLength{
            Greet: s,
            Length: n,
        }
    },
)

// ...

gNl := state.RunState(sGreetAndLength, 10)

fmt.Printf("Greet: %s, Length: %d\n", gNl.Greet, gNl.Length)
// will print: "Greet: hello-10, Length: 8"
```

This way we can combine two independent stateful computatios and combine them to get a single result.

## MONADIC STATE

Let's convert `State[S, A]` to a monad:

```go
// state/state.go

func FlatMap[S, A, B any](sa State[S, A], func(A) State[S, B]) State[S, B] {
    return func(s0 S) (S, B) {
        s1, a := sa(s0)
        return f(a)(s1)
    }
}
```

It still seems to be abstract and not very useful, lets advance a bit defining a couple of functions


One interesting thing we can do is to get the state from a previous computation and make it visible to our chain of calls of `Map/Map2/FlatMap`:

```go
// state/state.go

func Get[S any]() State[S, S] {
	return func(s S) (S, S) {
		return s, s
	}
}
```

The other function does the opposite, it puts a value into the state:

```go
func Put[S any](s S) State[S, S] {
	return func(S) (S, S) {
		return s, s
	}
}
```

And another function to transform the state, note it will transform the value, not the type:

```go
func Modify[S any](f func(S) S) State[S, S] {
	return func(s0 S) (S, S) {
		s1 := f(s0)
		return s1, s1
	}
}
```

What can we do with them?

Coming soon...
