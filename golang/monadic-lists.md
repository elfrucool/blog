# Monadic Lists in Golang


[Monads](https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) abstract the context of a computation. Where "context" could be quantity (a list), availability (a thing that can be nil), fallibility (something that, when executed, can produce an error), asynchronous execution (something that is not yet present), etc. And **computation** is just applying a function to the value whithin the context, and the result will be in the same context. Here's the list of _monadic go_:

List is the context that is easier to be used to explain the concept of a monad, so let's use it.

I strongly recommend you to read the following article if you still have not done it yet:

- [Functors, Applicatives, And Monads In Pictures](https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)

## Functor

Let's start with `Functor` category, don't be afraid of it, it just mean: _a "container" whose elements can be transformed inside it_.

Consider the following example:

```go
words := []string{"foo", "bar", "baz"}

upperWords := []string{}

for _, w := range words {
    upperWords = append(upperWords, strings.ToUpper(w)
}

fmt.Printf("%v\n", upperWords)
// will print FOO BAR BAZ
```

And now, let's define a functor over a list, it's just a function:

```go
package lists

func Map[A, B any](as []A, f func(A) B) []B {
	bs := make([]B, len(as))

	for i := range as {
		bs[i] = f(as[i])
	}

	return bs
}
```

And it could be used this way:

```go
words := []string{"foo", "bar", "baz"}

upperWords := lists.Map(words, func(s string) string {
    return strings.ToUpper(s)
})

fmt.Printf("%v -> %v\n", words, upperWords)
```

What we did is to abstract the list and its iteration away, and describe the transformation as _given a list, I want to map all its elements with this function_

It could be even shorter, as this:

```go
words := []string{"foo", "bar", "baz"}

upperWords := lists.Map(words, strings.ToUpper)

fmt.Printf("%v -> %v\n", words, upperWords)
```

Now it looks concise. The more complex our code is becoming, the more important os to express on _higher voice_ the intent of our operations and do not allow them to be overshadowed by implementation details.

To make it a real Functor, we also need to fulfill a few rules:

Let's `Id` be the following function:

```go
func Id[A any](a A) A {
    return a
}
```

Rules:

1. `lists.Map(as, Id) == as` for any list `as` regardless its type (identity rule)
2. `lists.Map(lists.Map(as, f), g)` == `lists.Map(as, func gof(a A) C { return g(f(a)) }` for any list `as` and any `func f[A, B any](A) B` and any `func g[B any](B) C` -- associativity


## Applicative

Applicatives are functors that can be merged. Also They have a function to put a value into the context. so:

```go
// lists.go

func Pure[A any](a A) []A {
    return []A{a}
}
```

We can use it very simple:

```
// main.go

singleValueInList := lists.Pure(10)

fmt.Println(singleValueInList) // will print [10]
```

And for combining two applicatives:

```go
func Map2[A, B, C any](as []A, bs []B, f func(A, B) C) []C {
	cs := []C{}

	for i := range as {
		a := as[i]
		for j := range bs {
			b := bs[j]
			cs = append(cs, f(a, b))
		}
	}

	return cs
}
```

And we can use it this way:

```go
odds := []int{1, 3, 5, 7, 9}
evens := []int{0, 2, 4, 6, 8}

multiplication := lists.Map2(odds, evens, func (i, j int) int {
    return i * j
})

fmt.Printf("%v x %v = %v\n", odds, evens, multiplication)
// will print:
// [1 3 5 7 9] x [0 2 4 6 8] = [0 2 4 6 8 0 6 12 18 24 0 10 20 30 40 0 14 28 42 56 0 18 36 54 72]
```

Interesting things come when we start chaining transformations over lists, example, given the following code with several transformations:

```go
odds := []int{1, 3, 5, 7, 9}
evens := []int{0, 2, 4, 6}

fmt.Printf("odds: %v\nevens: %v\n\n", odds, evens)

var pairs []T2[int, int] = lists.Map2(
	odds, evens, func(o, e int) T2[int, int] {
		return T2[int, int]{Fst: o, Snd: e}
	},
)

fmt.Printf("pairs: %v\n", pairs)

// Fst is the original pairs ([]T2[int, int]), Snd is the sum of each value
var additions []T2[T2[int, int], int] = lists.Map(
    pairs,
    func(t T2[int, int]) T2[T2[int, int], int] {
        return T2[T2[int, int], int]{t, t.Fst + t.Snd}
    },
)

// express each operation as a + b = c, e.g. 1 + 2 = 3
var operations []string = lists.Map(
	additions,
	func(t T2[T2[int, int], int]) string {
		return fmt.Sprintf("%d + %d = %d", t.Fst.Fst, t.Fst.Snd, t.Snd)
	},
)

for _, o := range operations {
	fmt.Println(o)
}
```

Now consider same code on a single expression:

```go
type T2[A, B any] struct {
    Fst A
    Snd B
} // tuple to hold two values

odds := []int{1, 3, 5, 7, 9}
evens := []int{0, 2, 4, 6}

var makePairs = func(o, e int) T2[int, int] {
    return T2[int, int]{Fst: o, Snd: e}
}

var sumPreservingOperands = func(t T2[int, int]) T2[T2[int, int], int] {
    return T2[T2[int, int], int]{Fst: t, Snd: t.Fst + t.Snd}
}

var sumOperationToString = func(t T2[T2[int, int], int]) string {
    return fmt.Sprintf("%d + %d = %d", t.Fst.Fst, t.Fst.Snd, t.Snd)
}

// express each operation as a + b = c, e.g. 1 + 2 = 3
var operations []string = lists.Map(
    lists.Map(
        lists.Map2(odds, evens, makePairs),
        sumPreservingOperands,
    ),
    sumOperationToString,
)

for _, o := range operations {
    fmt.Println(o)
}
```

The expression is very concise although it performs a series of complex operations:
1. It combines two lists using `makePairs` function to keep both elements on same structore.
2. It performs an addition operation and put the original data on a structure, so, both: operands and result are returned.
3. It converts each structore of operations and result into a human readable format.

Given the functor laws, we can make main expression even more compact, consider the following code:

```go
package function

func Pipe[A, B, C any](f func(A) B, g func(B) C) func(A) C {
	return func(a A) C {
		return g(f(a))
	}
}
```

We can do the following:

```go
type T2[A, B any] struct {
	Fst A
	Snd B
} // tuple to hold two values

odds := []int{1, 3, 5, 7, 9}
evens := []int{0, 2, 4, 6}

var makePairs = func(o, e int) T2[int, int] {
	return T2[int, int]{Fst: o, Snd: e}
}

var sumPreservingOperands = func(t T2[int, int]) T2[T2[int, int], int] {
	return T2[T2[int, int], int]{Fst: t, Snd: t.Fst + t.Snd}
}

var sumOperationToString = func(t T2[T2[int, int], int]) string {
	return fmt.Sprintf("%d + %d = %d", t.Fst.Fst, t.Fst.Snd, t.Snd)
}

var sumToString func(t T2[int, int]) string = function.Pipe(
    sumPreservingOperands, sumOperationToString,
)

var operations []string = lists.Map(
	lists.Map2(odds, evens, makePairs),
	sumToString,
)

for _, o := range operations {
	fmt.Println(o)
}
```

This is a small example of the power of lists as Functors/Applicatives. Unfortunately go is very verbose to write anonymous functions but ¯\\\_(ツ)\_/¯.

Applicatives have their own laws you can read about them here:
- https://medium.com/@jnkrtech/an-introduction-to-applicative-functors-aea966799b1d

## Monads

WIP
