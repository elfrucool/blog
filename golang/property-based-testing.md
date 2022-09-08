# PROPERTY BASED TESTING

On this article I'm going to show you a very concrete application regarding [State Monad](./state-monad.md) concept and one of its applications, a [Random Generator](./random-generator.md) into a way to implement tests that is named _Property-Based testing_[^1] [^2] [^3] [^4] (PBT for short), all using golang.

In short, given a program under test, we are going to assert invariants, i.e. _properties_ of it through passing constrained random data.

There are several golang libraries that implement PBT, let's list two of them:

* [testing/quick](https://pkg.go.dev/testing/quick) - Part of the standard golang libraries, but no longer maintained.
* [gopter](https://github.com/leanovate/gopter) - Latest commit was in 2021 (checked at 2022-09-07), much more robust and feature-rich

But in order to show the concepts we are discussing, we will take a minimalistic approach so we can have a feel of the power of PBT and random generators.

## PREPARING ENVIRONMENT

We need to do two actions to have our environment ready for the tests:

1. Add `github.com/stretchr/testify` to our project
2. Create a package with a test file

For \#1, open `go.mod` and add:

```go
// go.mod
module example.com/state-monad

go 1.18

require github.com/stretchr/testify v1.2.2
```

Then execute:

```sh
$ go mod tidy
```

This command will create a file named `go.sum` with some checksum and update `go.mod` with transitive dependenceies.

For \#2, do the following:

a) reate a new package named `pbt`:

```sh
$ mkdir pbt
$ echo 'package pbt' > pbt/pbt_test.go
```

b) Now, open `pbt/pbt_test.go` and import required packages:

```go
// pbt/pbt_test.go

package pbt

import (
	"testing"
	"time"

	"example.com/state-monad/rng"
	"github.com/stretchr/testify/require"
)
```

## TESTS WITH SINGLE INPUT

Let's say we want to test a function `func isOdd(n int) bool {...}`, for it, ideally we want to define two properties:
- `isOdd(n) == false` if `n` is even, we can generate even numbers for any random integer number `n` using `n * 2`
- `isOdd(n) == true` if `n` is odd, we can generate odd numbers for any random integer number `n` using `n * 2 + 1`

Let's rewrite our properties:

- `isOdd(n * 2) == false` for any int `n`
- `isOdd(n * 2 + 1) == true` for any int `n`

Now, ideally we should have two functions, each one of them to evaluate each property we already defined, let's write them without worrying how to put them on a test (yet).[^5]

```go
// pbt/pbt_test.go

// ...

// the function under test
func isOdd(n int) bool {
	panic("not yet implemented")
}

func isOdd_returnsFalseForEvenNumbers(t *testing.T, n int) {
    require.False(t, isOdd(n * 2))
}

func isOdd_returnsTrueForOddNumbers(t *testing.T, n int) {
    require.True()
}
```

Next thing is how can we assert these two properties against a bunch of random integers? let's create a function `_t1` for running 100 tests using 100 random values:

```go
// pbt/pbt_test.go

// ...

func _t1[A any](ga rng.RNG[A], seed uint64, t *testing.T, f func(*testing.T, A)) {
	for i := 0; i < 100; i++ {
		var a A
		seed, a = ga(seed)
		f(t, a)
	}
}
```

Now the test, on it we will assert our two properties we already defined:

```go
func TestIsOdd(t *testing.T) {
	t.Parallel()

	t.Run("should return false for even numbers", func(t *testing.T) {
		t.Parallel()
		seed := uint64(time.Now().UnixNano())
		_t1(rng.NextInt, seed, t, isOdd_returnsFalseForEvenNumbers)
	})

	t.Run("should return true for odd numbers", func(t *testing.T) {
		t.Parallel()
		seed := uint64(time.Now().UnixNano())
		_t1(rng.NextInt, seed, t, isOdd_returnsTrueForOddNumbers)
	})
}
```

If we run this test, it should not be surprise that it panics, becaunse our function `isOdd` is not yet implemented; let's try to fix it, e.g. returning `false` always

```go
// pbt/pbt_test.go

// ...

func isOdd(n int) bool {
	return false
}

// ...
```

No surprise our `isOdd_returnsFalseForEvenNumbers` property was verified but not `isOdd_returnsTrueForOddNumbers`. But also, our error is not very informative, let's fix that in **both** our properties:

```go
// pbt/pbt_test.go

// ...

func isOdd_returnsFalseForEvenNumbers(t *testing.T, n int) {
	number := n * 2
	require.Falsef(t, isOdd(number), "%d should return false", number)
}

func isOdd_returnsTrueForOddNumbers(t *testing.T, n int) {
	number := n*2 + 1
	require.Truef(t, isOdd(number), "%d should return true", number)
}

// ...
```

`isOdd_returnsTrueForOddNumbers` property reports error much better, e.g.: `6809761838893980321 should return true`

Let's test `isOdd_returnsFalseForEvenNumbers` property error messages:

```go
// pbt/pbt_test.go

// ...

func isOdd(n int) bool {
	return true
}

// ...
```

`isOdd_returnsFalseForEvenNumbers` has good error messages too: `-5525521985809462772 should return false`

Now, we can implement the function, but, can we cheat it? e.g. giving some known numbers such as: [^5]

```go
// pbt/pbt_test.go

// ...

func isOdd(n int) bool {
	switch n {
	case 0, 2, 4, 6, 8, 10:
		return false
	case 1, 3, 5, 7, 9:
		return true
	}
	return false
}

// ....
```

It simply failed with any random number not considered within the cases: `566507398597479755 should return true`.

Now it's time to correctly write the function:

```go
// pbt/pbt_test.go

// ...

func isOdd(n int) bool {
	return n%2 == 1
}

// ...
```

**IT FAILED**, specifically the `isOdd_returnsTrueForOddNumbers` property. Are you saying that there are odd numbers `n` for which `n%2 != 1`? Yes, example: `-2011454488991129831 should return true`. **Negative numbers return negative modulus**

One of the strengths of PBT is that it can uncover issues that were not considered when implementing the tests.

Now let's try to fix our function, e.g. what if we test that `n%2` is not zero?

```go
// pbt/pbt_test.go

// ...

func isOdd(n int) bool {
	return n%2 != 0
}

// ...
```

It's working now!

## TESTS WITH MORE THAN ONE INPUT

With our `_t1()` function we have all that we need, if we want to evaluate more data, we just have to define a complex `struct` with all the data that we need, and produce random values for it.

But for the sake of ergonomics, let's try defining a `_t2()` function that accepts two `rng`'s and executes 100 tests using them, and similar function `_t3()` for three input parameters.

```go
// pbt/pbt_test.go

// ...

func _t2[A, B any](ga rng.RNG[A], gb rng.RNG[B], seed uint64, t *testing.T, f func(*testing.T, A, B)) {
	for i := 0; i < 100; i++ {
		var a A
		var b B
		seed, a = ga(seed)
		seed, b = gb(seed)
		f(t, a, b)
	}
}

func _t3[A, B, C any](ga rng.RNG[A], gb rng.RNG[B], gc rng.RNG[C], seed uint64, t *testing.T, f func(*testing.T, A, B, C)) {
	for i := 0; i < 100; i++ {
		var a A
		var b B
		var c C
		seed, a = ga(seed)
		seed, b = gb(seed)
		seed, c = gc(seed)
		f(t, a, b, c)
	}
}

// ...
```

Now, let's choose adding two integers as our case:

```go
// pbt/pbt_test.go

// ...

func plus(a, b int) int {
	return a + b
}

// ...
```

What properties can we define for `plus` function?

We can start with something took from any math book:

1. For any number `n`, it is always true that `n + 0 = n` (zero is neutral element for addition) -- will use `_t()` function.
2. For any pair of numbers `m`, `n`, it is always true that `m + n = n + m` (commutativity) -- will use `_t2()` function.
3. For any three numbers `a`, `b`, `c`, it is always true that `a + (b + c) = (a + b) + c` (associativity) -- will use `_t3()` function.

Let's implement these three properties for addition:

```go
// pbt/pbt_test.go

// ...

func addZeroIsNeutral(t *testing.T, a int) {
	require.Equalf(t, a, plus(a, 0), "%d + 0 should be %d", a, a)
}

func addCommutativity(t *testing.T, a, b int) {
	require.Equalf(t, plus(a, b), plus(b, a), "%d + %d should be equal to %d + %d", a, b, b, a)
}

func addAssociativity(t *testing.T, a, b, c int) {
	require.Equalf(
		t, plus(a, plus(b, c)), plus(plus(a, b), c),
		"%d + (%d + %d) should be equal to (%d + %d) + %d",
		a, b, c, a, b, c,
	)
}

// ...
```

And the test function:

```go
// pbt/pbt_test.go

// ...

func TestAdd(t *testing.T) {
	t.Parallel()

	t.Run("neutral element", func(t *testing.T) {
		t.Parallel()
		seed := uint64(time.Now().UnixNano())
		_t1(rng.NextInt, seed, t, addZeroIsNeutral)
	})

	t.Run("commutativity", func(t *testing.T) {
		t.Parallel()
		seed := uint64(time.Now().UnixNano())
		_t2(rng.NextInt, rng.NextInt, seed, t, addCommutativity)
	})

	t.Run("associativity", func(t *testing.T) {
		t.Parallel()
		seed := uint64(time.Now().UnixNano())
		_t3(rng.NextInt, rng.NextInt, rng.NextInt, seed, t, addAssociativity)
	})
}

// ...
```

## DISCOVERING NEW THINGS ABOUT OUR PROGRAM

Let's say, we are not satisifed with our solution, that we suspect something can go wrong, let's add two more properties to our test:

1. For all two non-negative numbers `a` and `b`, the expression `a + b >= 0` is always true (adding two positive numbers gives us another positive number)
2. For all two non-positive numbers `a` and `b`, the expression `a + b <= 0` is always true (adding two negative numbers gives us another negative number)

```go
// pbt/pbt_test.go

// ...

func addPositiveNumbers(t *testing.T, a, b int) {
	// a small guard against bad input, we will ensure only giving valid input
	if a < 0 || b < 0 {
		panic("invalid input data")
	}

	result := plus(a, b)

	require.Truef(
		t, result >= 0,
		"%d + %d should be greater or equal than 0, but it was %d",
		a, b, result,
	)
}

func addNegativeNumbers(t *testing.T, a, b int) {
	// a small guard against bad input, we will ensure only giving valid input
	if a > 0 || b > 0 {
		panic("invalid input data")
	}

	result := plus(a, b)

	require.Truef(
		t, result <= 0,
		"%d + %d should be smaller or equal than 0, but it was %d",
		a, b, result,
	)
}

// ...
```

To make things easier, let's define an `abs()` function that converts a negative number into a positive one:

```go
// pbt/pbt_test.go

// ...

func abs(n int) int {
	if n < 0 {
		return -n
	}

	return n
}

// ...
```

And now the test, we will provide only valid input data using what we have in our `rng` library to transform values:

```go
// pbt/pbt_test.go

// ...

func TestAdd(t *testing.T) {
	t.Parallel()

	// ...

	t.Run("adding positive numbers result in positive number", func(t *testing.T) {
		t.Parallel()
		seed := uint64(time.Now().UnixNano())
		ga := rng.Map(rng.NextInt, abs)
		gb := rng.Map(rng.NextInt, abs)
		_t2(ga, gb, seed, t, addPositiveNumbers)
	})

	t.Run("adding negative numbers result in negative number", func(t *testing.T) {
		t.Parallel()
		nabs := func(n int) int { return -abs(n) } // negative abs
		seed := uint64(time.Now().UnixNano())
		ga := rng.Map(rng.NextInt, nabs)
		gb := rng.Map(rng.NextInt, nabs)
		_t2(ga, gb, seed, t, addNegativeNumbers)
	})
}

// ...
```

**BOTH PROPERTIES FAILED** why? let's see error messages:

- For `addPositiveNumbers` it was: `5039997529336360482 + 5416970325618956019 should be greater or equal than 0, but it was -7989776218754235115`

- For `addNegativeNumbers` it was: `-6207623438501387300 + -4864114072245600727 should be smaller or equal than 0, but it was 7375006562962563589`

It turns out that in math, above properties are true, but in computesr, int numbers have limited size, and if `a + b > biggest number`, then it results in some negative number, see how above messages are true in [playground](https://play.golang.com/p/cxgdbzRypXo)

What could we made is to restrict the size of the numbers such that `a + b <= biggest integer` and then properties will remain true. I'll left the exercise for the reader. ;)

## CONCLUSION

PBT is a powerful tool to uncover corner scenarios, potential bugs and hidden properties of our programs.

It does not substitute traditional unit testing (either simple test or using a table of data). It completes unit testing.

It challenges us to think on the invariants of our system, and its boundaries.

E.g. what if for an email we give non alphanumeric characers? what if for a name we give a string 10k characters length? ...

We have seen a concrete application of random-values generator, which is also an application of the state monad.

I hope this is helpful at least on thinking how we build programs.

## SEE:

- [State Monad in Golang](./state-monad.md)
- [Random-Generator using State monad](./random-generator.md)
- [Other Articles](../README.md)


[^1]: https://forallsecure.com/blog/what-is-property-based-testing

[^2]: https://hypothesis.works/articles/what-is-property-based-testing/

[^3]: https://hackernoon.com/what-is-property-based-testing-part-1\

[^4]: https://giedrius.blog/2019/07/08/property-based-testing-in-golang/

[^5]: The idea to "cheat" the function comes from providing a fixed number of inputs/expected outputs for testing, but with PBT, it's not possible, see: https://www.slideshare.net/ScottWlaschin/the-lazy-programmers-guide-to-writing-thousands-of-tests
