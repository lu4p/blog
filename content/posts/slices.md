---
title: "Inner workings of allocating slices with Go (golang)"
date: 2020-12-02T22:23:20+01:00
tags: ["golang"]
---

I wasn't sure what the performance impact of preallocating a slice with `make` vs. 
just letting the slice grow via `append` is, so I investigated a little.

## Arrays
In order to understand what a slice is one needs to first understand how arrays work in go.

An array's type definition specifies a length and an element type. 
For example, the type `[3]int` represents an array of three integers. 

An array's size is fixed, its length is part of the type, so `[2]int` and `[3]int` are incompatible, 
e.g. a variable of type `[2]int` cannot be assigned to a variable of type `[3]int`.

```go
a := [2]int{1, 2}
var b [3]int
b = a
// Output: cannot use a (type [2]int) as type [3]int in assignment
```

Arrays are initialized right away, the zero value of an array is an array whose elements have their zero value.
```go
var b [2]int
fmt.Println(b[1])
// Output: 0
```
In-memory a variable of type `[3]int` is just 3 integers laid out sequentially.

An array is a value, an `[3]int` variable are just 3 integers, being passed around.

## Slices

First of all what is a slice exactly, a slice can be thought of a struct with the following values:
```go
type slice struct {
	len int            // the length of the slice
	cap int            // the capacity of the slice i.e. the max possible len without allocating new memory
	ptr unsafe.Pointer // pointer to the first Element of the underlying array
}
```

### append

`append` needs to allocate a new underlying array, if the elements to append exceed the capacity of the underlying array,
in this case a new bigger array is allocated, all elements copied from the old to the new array, 
and the new elements are copied after the last element of the copied slice. 

- `cap` is set to the length of the newly allocated array, 
- `len` is to the index of the last element added via `append` (+ 1)
- `ptr` is set to point to the underlying array

For performance it's important how often the values are copied around in memory.

This is what I came up with, to determine how often the slice is reallocated and copied,
I also print some useful information about the allocated capacity of the slice.

```go
package main

import (
	"fmt"
)

func main() {
	elmCount := 100_000

	fmt.Printf("%8v | %8v | %8v | %v\n", "len", "prev cap", "new cap", "ratio added cap")

	var foo []int
	for i := 0; i < elmCount; i++ {
		if i == 0 {
			foo = append(foo, i)
			continue
		}

		first := &foo[0]

		beforeCap := cap(foo)

		foo = append(foo, i)

		firstNew := &foo[0]

		// if the start of the underlying array changed
		if firstNew != first {
			afterCap, afterLen := cap(foo), len(foo)

			capRatio := float64(afterCap) / float64(beforeCap)

			fmt.Printf("%8v | %8v | %8v | %.2fx\n", afterLen, beforeCap, afterCap, capRatio)
		}
	}
}
```

In the above code a slice of integers with 100 000 elements is created, by using only `append`.

The above code outputs:
```
     len | prev cap |  new cap | ratio added cap
       2 |        1 |        2 | 2.00x
       3 |        2 |        4 | 2.00x
       5 |        4 |        8 | 2.00x
       9 |        8 |       16 | 2.00x
      17 |       16 |       32 | 2.00x
      33 |       32 |       64 | 2.00x
      65 |       64 |      128 | 2.00x
     129 |      128 |      256 | 2.00x
     257 |      256 |      512 | 2.00x
     513 |      512 |     1024 | 2.00x
    1025 |     1024 |     1280 | 1.25x
    1281 |     1280 |     1696 | 1.32x
    1697 |     1696 |     2304 | 1.36x
    2305 |     2304 |     3072 | 1.33x
    3073 |     3072 |     4096 | 1.33x
    4097 |     4096 |     5120 | 1.25x
    5121 |     5120 |     7168 | 1.40x
    7169 |     7168 |     9216 | 1.29x
    9217 |     9216 |    12288 | 1.33x
   12289 |    12288 |    15360 | 1.25x
   15361 |    15360 |    19456 | 1.27x
   19457 |    19456 |    24576 | 1.26x
   24577 |    24576 |    30720 | 1.25x
   30721 |    30720 |    38912 | 1.27x
   38913 |    38912 |    49152 | 1.26x
   49153 |    49152 |    61440 | 1.25x
   61441 |    61440 |    76800 | 1.25x
   76801 |    76800 |    96256 | 1.25x
   96257 |    96256 |   120832 | 1.26x
```

Interesting it looks like append is smart and doesn't only grow the slice to the currently needed capacity, 
but doubles the capacity for smaller capacities (under 1024) and grows it by ~1.25x for larger capacities (over 1024), 
to reduce the overhead of reallocating arrays.

In the runtime this is implemented as follows ([full implementation](https://github.com/golang/go/blob/c53315d6cf1b4bfea6ff356b4a1524778c683bb9/src/runtime/slice.go#L125)):
```go
func growslice(et *_type, old slice, cap int) slice {
...
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            newcap = doublecap
        } else {
            // Check 0 < newcap to detect overflow
            // and prevent an infinite loop.
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // Set newcap to the requested cap when
            // the newcap calculation overflowed.
            if newcap <= 0 {
                newcap = cap
            }
        }
    }
...
}
```

## Performance
I benchmarked what the performance impact of preallocating a slice with `make` vs auto growing via `append` is, the benchmark populates a slice with N integers them via a for loop. 

```go
func plainAppend(elms int) {
	var foo []int
	for i := 0; i < elms; i++ {
		foo = append(foo, i)
	}
}

func withMake(elms int) {
	foo := make([]int, 0, elms)
	for i := 0; i < elms; i++ {
		foo = append(foo, i)
	}
}
```

Full Benchmark: https://gist.github.com/lu4p/4906f5d1b88f23a1f740bb8895f25e40
```sh
$ go test -bench=. -count=10 -benchtime=100x > bench.out

# https://godoc.org/golang.org/x/perf/cmd/benchstat
$ benchstat bench.out

name                  time/op
10/append-8            729ns ±62%
10/with_make-8         119ns ±84%
100/append-8          1.46µs ±84%
100/with_make-8        454ns ±99%
1k/append-8           5.28µs ±33%
1k/with_make-8        2.61µs ±54%
10k/append-8           111µs ±23%
10k/with_make-8       31.4µs ±11%
100k/append-8         1.16ms ± 8%
100k/with_make-8       248µs ±11%
1Million/append-8     8.57ms ± 3%
1Million/with_make-8  2.83ms ± 1%
```
So in a simple use case preallocating the slice with `make` seems to be about 2-4 times faster. 
Although this is significant, in real-world scenarios it often doesn't really matter because appending is fast enough regardless.

If you don't know the number of elements a to which a slice will grow, 
it often makes little sense to allocate memory with make beforehand.
Of course there are exceptions from this, 
sometimes you know at least the order of magnitude to which a slice will grow,
then it can make sense to allocate a slice with a cap similar to the ballpark of expected elements.

Thanks to [Hu1buerger](https://github.com/Hu1buerger), for helping with the investigation. 