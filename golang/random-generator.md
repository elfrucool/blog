# Random Generator in GO using State Monad

See previous article [State Monad in Golang](./state-monad.md).

In this article we are going to build a pseudo-random values generator using the state monad, it will not just generate pseudo-random nubers, but any kind of data (strings, numbers, lists, etc); this library will contain several functions to combine all these values.

On a third article (coming soon) we will use this library to perform unit testing based on random values, this approach is named [property-based testing](https://medium.com/criteo-engineering/introduction-to-property-based-testing-f5236229d237)

## STARTING POINT

Let's start with previous article code and module, now let's create another package & file:

```sh
mkdir rng

echo 'package rng' > rng/rng.go
```

## RANDOM NUMBER GENERATOR DEFINITION

Our random generator (RNG for short) will be defined as:


```go
// rng/rng.go

import (
    "example.com/state-monad/state"
)

type RNG[A any] state.State[uint64, A]
```

Given some oddities of go, let's create some functions that will make the code easier to write/read, those functions will help to translate things between `rng` and `state` package:

```go
// rng/rng.go

// this function will convert any RNG into its corresponding State type
func _s[A any](r RNG[A]) state.State[uint64, A] {
    return state[uint64, A)(r
}

// this function will convert any State[uint64,A] into a RNG[A] type
func _r[A any](s state.State[uint64, A]) RNG[A] {
    return RNG[A](s)
}
```

Next function will run a RNG returning its value:

```go
// rng/rng.go

func Run[A any](r RNG[A], seed uint64) A {
	return state.RunState(_s(r), seed)
}
```

Let's start veryfying our code with something simple:


```go
// main.go

import "example.com/state-monad/rng"

func main() {
    var foo rng.RNG[string] = func(u uint64) (uint64, string) {
        return u + 1, "hello"
    }

    result := rng.Run(foo, 0)

    fmt.Printf("result: %s\n", result) // will print "hello"
}
```

It dosen't seem to be any improvement over our original [state monad](./state-monad.md); now let's add some randomness to it:


```go
// rng/rng.go

var NextSeed RNG[uint64] = func (seed uint64) (uint64, uint64) {
    // these numbers have nothing in particular, except they are "big"
	seed2 := seed*1318971987123988123 + 1928371293
	return seed2, seed2
}
```

And we can play generating some numbers with it:

```go
// main.go

// inside main()...

seed := uint64(0)
v, seed := rng.NextSeed(seed)
fmt.Printf("(v1, seed) = (%d, %d)\n", v, seed)
v, seed = rng.NextSeed(seed)
fmt.Printf("(v1, seed) = (%d, %d)\n", v, seed)
v, seed = rng.NextSeed(seed)
fmt.Printf("(v1, seed) = (%d, %d)\n", v, seed)
// will print some numbers that seem to be random
```

We can use `uint64(time.Now().UnixNano())` to generate different values each time the program is executed.


## CONVERTING RNG INTO A STATE MONAD

Although we have everything we need in `state/state.go` code, it will be very uncomfortable to cast back and forth between `state.State[uint64, A]` and `RNG[A]` when applying `Map/Map2/FlatMap/etc` functions, so let's add some wrapper functions into `rng` package for them, review [previous article](./state-monad.md) for their explanation:


```go
// rng.go

func Pure[A any](a A) RNG[A] {
	return _r(state.Pure[uint64](a))
}

func Get() RNG[uint64] {
	return _r(state.Get[uint64]())
}

func Map[A, B any](ra RNG[A], f func(A) B) RNG[B] {
	return _r(state.Map(_s(ra), f))
}

func Map2[A, B, C any](ra RNG[A], rb RNG[B], f func(A, B) C) RNG[C] {
	return _r(state.Map2(_s(ra), _s(rb), f))
}

func FlatMap[A, B any](ra RNG[A], f func(A) RNG[B]) RNG[B] {
	return _r(state.FlatMap(_s(ra), func(a A) state.State[uint64, B] {
		return _s(f(a))
	}))
}
```

With the above functions we can do things like these:

```go
// main.go

// inside main() function

var r1 rng.RNG[string] = rng.Pure("Hello")

var r2 rng.RNG[int] = rng.Map(r1, func (s string) int {
    return len(s)
})

var r3 rng.RNG[string] = rng.Map2(rng.NextSeed, r1, func (u uint64, s string) {
    return fmt.Sprintf("%d -> %s", u, s)
})

// combining several random operations
var r4 rng.RNG[string] = rng.FlatMap(
    rng.NextSeed, // take a seed
    func (u uint64) string {
        x := rng.NextSeed // take another seed
        y := rng.Map(x, func (u2 uint64) string) {
            return fmt.Sprintf("<%d>", u2) // second seed as string
        }
        return rng.Map2(u, y, func (u3 uint64, s string) string {
            return fmt.Sprintf("[%d] -> [%s]", u3, s) // combining both seeds
        })
    },
)
```

## BASIC GENERATORS

This library is to generate any random value, let's start building some primitive value generator functions.

The very basic generator is already defined, it is `NextSeed`, it takes a seed and generates a new one, returning itself as new state and new value.

Next is to generate some other values based on it: numbers of different sizes and chars, we are going to use `Map` function to generate them:

```go
// rng/rng.go

var NextInt64 RNG[int64] = Map(NextSeed, func(u uint64) int64 { return int64(u) })

var NextInt RNG[int] = Map(NextSeed, func(u uint64) int { return int(u) })

var NextRune RNG[rune] = Map(NextSeed, func(u uint64) rune { return rune(u) })

var NextUint RNG[uint] = Map(NextSeed, func(u uint64) uint { return uint(u) })

var NextUint8 RNG[uint8] = Map(NextSeed, func(u uint64) uint8 { return uint8(u) })

var NextInt8 RNG[int8] = Map(NextSeed, func(u uint64) int8 { return int8(u) })

var NextBool RNG[bool] = Map(NextSeed, func(u uint64) bool { return u%2 == 0 })
```

Not so bad, we can generate almost any kind of integer type AND booleans with this approach. Note the `NextBool` generator it uses modulus operator, same approach can be used to generate any enum random value that is based on numbers. (This is a challenge I left to the reader ;) )

To generate floating-point values, we are going to take two random numbers and combine them, note thatn we can generate positive/negative/zero/Inf/NaN values:

```go
// rng/rng.go

var NextFloat64 RNG[float64] = Map2(NextInt64, NextInt64, func(i1, i2 int64) float64 {
    return float64(i1) / float64(i2)
})

var NextFloat32 RNG[float32] = Map2(NextInt, NextInt, func(i1, i2 int) float32 {
    return float32(i1) / float32(i2)
})
```

## RANGED GENERATORS

Next challenge is to generate values within a range, we can think in two possible scenarios: min/max and just min, let's implement some functions to get them:

```go
// rng/rng.go

// this type is just to encapsulate all integer types and use generics
type Integer interface {
	int8 | uint8 | int16 | uint16 | rune | int | uint | int64 | uint64
}

func NextMax[A Integer](max A, r RNG[A]) RNG[A] {
	if max < 1 {
		panic("max must be positive number")
	}

	return Map(
		r,
		func(n A) A {
			n2 := n
			if n < 0 {
				n2 = -n
			}

			return n2 % max
		},
	)
}

func NextRanged[A Integer](min, max A, r RNG[A]) RNG[A] {
	if min > max {
		panic("min must not be greater than max")
	}

	if min == max {
		return Pure(min)
	}

	delta := max - min

	g1 := NextMax(delta, r)

	return Map(g1, func(n A) A {
		return n + min
	})
}

func Rune(min, max rune) RNG[rune] {
	return Map(
		NextRune,
		func(r rune) rune {
			r2 := r % (max - min)
			return r2 + min
		},
	)
}
```

So if we want `int` random numbers between 10 and 20, we just call `rng.NextRanged(10, 20, rng.NextInt)`, same approach if we want random letters between `a` and `z`: `rng.Rune('a', 'z')`.

# SIZED GENERATORS

Now we are going to generate size-based generators, with them we can produce lists and strings, the base of all of them is the `Times()` generator:


```go
// rng/rng.go

func Times[A any](times int, g RNG[A]) RNG[[]A] {
	var asg RNG[[]A] = Pure(make([]A, 0, times))
	for i := 0; i < times; i++ {
		asg = Map2(asg, g, func(as []A, a A) []A {
			return append(as, a)
		})
	}
	return asg
}
```

Now let's implement lists and strings of any random size within a range:

```go
// rng/rng.go

func NextList[A any](minL, maxL uint, ra RNG[A]) RNG[[]A] {
	rLength := NextRanged(minL, maxL, NextUint)
	return FlatMap(rLength, func(n uint) RNG[[]A] {
		n2 := int(n)
		if n2 < 0 {
			n2 = 0
		}

		return Times(n2, ra)
	})
}

func NextString(minL, maxL uint, runeGen RNG[rune]) RNG[string] {
	var runes RNG[[]rune] = NextList(minL, maxL, runeGen)
	return Map(runes, func(rs []rune) string { return string(rs) })
}
```

## APPLICATION

We have a complete pseudo-random value generator library with the most important types, to generate more complex types, we can use `Map/Map2/FlatMap` to map these values into structs/etc.

Now I'm going to show the power of above library in action into a small program that generates random directories, some of them will contain files and some do not, this could be useful for performance testing over a complex scenario.

```go
// main.go
package main

import (
	"fmt"
	"os"
	"path"
	"time"

    "example.com/state-monad/rng"
)

// this structure is for determining when to insert a txt file
type PathAndShouldCreateFile struct {
	Path             string
	ShouldCreateFile bool
}

// this structure will keep stats over the process
// note current implementation does not evaluate repeated entries
type Stats struct {
	TotalEntries int
	EmptyDirs    int
	Files        int
}

// handling stats as appendable (in fp. it is called monoid)
// soon, I'll add a dedicated article
func (s1 Stats) Append(s2 Stats) Stats {
	return Stats{
		TotalEntries: s1.TotalEntries + s2.TotalEntries,
		EmptyDirs:    s1.EmptyDirs + s2.EmptyDirs,
		Files:        s1.Files + s2.Files,
	}
}

// emptyDirStats() generates stats for single empty dir created
func emptyDirStats() Stats {
	return Stats{
		TotalEntries: 1,
		EmptyDirs:    1,
		Files:        0,
	}
}

// fileStats() generates stats for directory + file created
func fileStats() Stats {
	return Stats{
		TotalEntries: 1,
		EmptyDirs:    0,
		Files:        1,
	}
}

// this is the program, expressions r0 to r4 describe
// the possible random values to be generated
func main() {
    // letters from 'A' to 'Z'
	r0 := rng.NextRanged('A', 'Z', rng.NextRune)
    // strings between 1 and 8 characters of 'A-Z` letters
	r1 := rng.NextString(1, 8, r0)
    // lists of size between 1 and 20 elements of above strings
	r2 := rng.NextList(1, 20, r1)
    // converting the list []string{"a", "b", "c"} into string "a/b/c"
    // that represents a path
	r3 := rng.Map(r2, func(ss []string) string {
		return path.Join("tmp", path.Join(ss...))
	})
    // creates PathAndShouldCreateFile structore
    // if a random number between 0 and 10 is less than 3, then create a file (flag is true)
	r4 := rng.Map2(r3, rng.NextRanged(0, 10, rng.NextUint8), func(s string, n uint8) PathAndShouldCreateFile {
		return PathAndShouldCreateFile{
			Path:             s,
			ShouldCreateFile: n < 3,
		}
	})

    // let's create this number of directories
	maxExecutions := 100

    // seed will change everytime the program is run
	state := uint64(time.Now().UnixNano())

	stats := Stats{} // empty stats

    // iterate until get all executions
	for i := 1; i <= maxExecutions; i++ {
		var t PathAndShouldCreateFile
		state, t = r4(state)

		p := t.Path
        // create directory and all its parents
		err := os.MkdirAll(p, 0755)
		if err != nil {
            // on error, print and continue
			fmt.Fprintf(os.Stderr, "error creating dir: %s -> %v\n", p, err)
			continue
		}

        // we are going to create file 1.txt in the directory if the flag is set
        // we are going to update stats accordingly to the flag
		if t.ShouldCreateFile {
			f := path.Join(p, "1.txt")

			h, err := os.Create(f)
			if err != nil {
                // on error, print and continue
				fmt.Fprintf(os.Stderr, "error creating file: %s -> %v\n", f, err)
				continue
			}

			stats = stats.Append(fileStats())

			// maybe is not relevant if the file could not be closed
			if err = h.Close(); err != nil {
				fmt.Fprintf(os.Stderr, "error closing file: %s -> %v\n", f, err)
			}
		} else {
			stats = stats.Append(emptyDirStats())
		}

        // print something from time to time to give the user some info
		if i%10 == 0 {
			fmt.Printf("Built %d paths, (emptyDirs: %d, files: %d)\n", i, stats.EmptyDirs, stats.Files)
		}
	}

    // print some stats at the end
	fmt.Printf(
		"\nDone.\n---[stats]---\nTotal Directories     : %d\nEmpty Directories     : %d\nDirectories with files: %d\n",
		stats.TotalEntries,
		stats.EmptyDirs,
		stats.Files,
	)
}
```

## See:
- Previous article: [State Monad in Golang](./state-monad.md)
