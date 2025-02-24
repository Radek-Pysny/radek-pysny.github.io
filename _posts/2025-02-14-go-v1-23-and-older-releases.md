---
title: 'Go v1.23 and older releases'
tags: [Go, GoReleaseNotes]
---

## Go v1.23 and older releases

 - [Go v1.23](#go-v123)
     - [Already used features](#already-used-features-of-go-v123)
     - [Not yet used features](#not-yet-used-interesting-features-of-go-v123)
 - [Go v1.22](#go-v122)
     - [Already used features](#already-used-features-of-go-v122)
     - [Not yet used features](#not-yet-used-interesting-features-of-go-v122)
 - [Go v1.21](#go-v121)
     - [Already used features](#already-used-features-of-go-v121)
     - [Not yet used features](#not-yet-used-interesting-features-of-go-v121)
 - [Go v1.20](#go-v120)
     - [Already used features](#already-used-features-of-go-v120)
     - [Not yet used features](#not-yet-used-interesting-features-of-go-v120)
 - [Go v1.19](#go-v119)
     - [Already used features](#already-used-features-of-go-v119)
     - [Not yet used features](#not-yet-used-interesting-features-of-go-v119)
 - [Go v1.18](#go-v118)

---

---

I was using Go v1.23 already for several months. However, I decided to go in the past and check
what new feature was added to Go and its ecosystem since I started as a Go developer.

To make it easier, I will mention just those features I have already used and those I would like to start using.


---

### Go v1.23

Released on 2024-08-13.


#### Already used features of Go v1.23

Introduction of iterators was a big deal for me. However, before using them, I read the 
[official documentation ot `iter` package](https://pkg.go.dev/iter@go1.23.0) and two articles
before I was able to understand it correctly. I have already used it and I really like it.
It is true that consuming already made iterator is more fun than implementing the brand-new one.
I was as well a bit curious about adding [`Preorder`](https://pkg.go.dev/go/ast@go1.23.0#Preorder)
iterator into package [`go/ast`](https://pkg.go.dev/go/ast@go1.23.0).
Here is an example of iterator I have implemented (map-based y/x position sorting):

```go
func (g guideMap) allSorted() iter.Seq[any] {
	return func(yield func(any) bool) {
		ys := slices.Collect(maps.Keys(g))
		sort.Ints(ys)

		for _, y := range ys {
			row := g[y]
			xs := slices.Collect(maps.Keys(row))
			sort.Ints(xs)

			for _, x := range xs {
				if !yield(row[x]) {
					return
				}
			}
		}
	}
}
```

Touches on [`time.Timer`](https://pkg.go.dev/time@go1.23.0#Timer) 
and [`time.Ticker`](https://pkg.go.dev/time@go1.23.0#Ticker) affected my work life. A possibility to garbage-collect 
these unstopped objects before they fire bring some good. But usage of unbuffered channel internally brought guarantee 
that for any call to a `Reset` or `Stop` method, no stale values prepared before that call will be sent or received 
after the call. TL;DR: no more optional receiving of ticks after stopping a ticker/timer.


#### Not yet used interesting features of Go v1.23

I did not opt in for Go telemetry (using `go telemetry` command). I felt like possibly exposing something
secret. Better safe than sorry situation for me.

I did not even check generic aliases as it was still an experimental feature.

I did not find any reason to try "canonicalizing" values via [`unique`](https://pkg.go.dev/unique@go1.23.0)
package. I know, Lua uses "interning" of strings, so there is always just one instance of each string
in the memory. So using [`Make[T]`](https://pkg.go.dev/unique@go1.23.0#Make) generic function and working with 
[`Handle[T]`](https://pkg.go.dev/unique@go1.23.0#Handle) type (two values of that 
generic type  are considered equal iff the values used for making them are equal) would reduce memory
footprint and comparison might be as efficient as simple pointer comparison.

The new [`structs`](https://pkg.go.dev/structs@go1.23.0) package provided types for struct fields that modify 
properties of the containing struct type such as memory layout. Not used yet.

The new [`Encode`](https://pkg.go.dev/encoding/binary@go1.23.0#Encode) and 
[`Decode`](https://pkg.go.dev/encoding/binary@go1.23.0#Decode) functions of 
[`encoding/binary`](https://pkg.go.dev/encoding/binary@go1.23.0) package became byte slice equivalents to 
`Read` and `Write`. Also `Append` allowed marshaling multiple data into the same byte slice.

Etc.

---

### Go v1.22

Released on 2024-02-06.


#### Already used features of Go v1.22

The most importantly, `for` loop creates a new variable for each iteration, so accidental sharing bugs are avoided
(aka trying to save pointers into slice for all iterated values will not produce a slice of pointers all pointing
to the only one variable with value from the very last iteration). Also `go vet` was aligned to this.

I almost immediately, started using `for` loops ranging over integers (e.g. `for range b.N { ... }`) not just
in my new benchmarks.

`go get` could not be used outside a module anymore.

Printing of coverage summaries for packages without own test files on `go test -cover` is something one might
notice (e.g. `mymod/mypack coverage: 0.0% of statements`).


#### Not yet used interesting features of Go v1.22

Enhanced routing patterns used by [`net/http.ServeMux`](https://pkg.go.dev/net/http@go1.22.0#ServeMux):
- registering `"POST /items/create"` to restrict invocation method
  (mind that `"GET"` method also registers for `"HEAD"` method)
- Wildcards in patterns, like `/items/{id}`, match segments of the URL path.
  The actual segment value may be accessed by calling the `Request.PathValue` method.
- A wildcard ending in `...`, like `/files/{path...}`, must occur at the end of a pattern and matches all the 
  remaining segments.
- A pattern that ends in `/` matches all paths that have it as a prefix, as always.
- To match the exact pattern including the trailing slash, end it with `{$}`, as in `/exact/match/{$}`.
- If two patterns overlap in the requests that they match, then the more specific pattern takes precedence. 
  If neither is more specific, the patterns conflict. This rule generalizes the original precedence rules
  and maintains the property that the order in which patterns are registered does not matter.

Vendor directory for workspaces was not used as I am not usually using Go workspaces.

I did not notice that `vet` complained on `slice = append(slice)`, nor `defer log.Println(time.Since(t))`
(equivalent of `tmp := time.Since(t); defer log.Println(tmp)`), nor warnings for mismatched key-value pairs 
in `log/slog` calls.

I did not use [`math/rand/v2`](https://pkg.go.dev/math/rand/v2@go1.22.0) even though it brought many changes. 
Perhaps, one might change this...

Also, I did not face a brand-new package of [`go/version`](https://pkg.go.dev/go/version@go1.22.0).

The [`Null[T]`](https://pkg.go.dev/database/sql@go1.22.0#Null) generic type of 
[`database/sql`](https://pkg.go.dev/database/sql@go1.22.0) provided a way to scan nullable columns 
for any column types.

Etc.


---

### Go v1.21

Released on 2023-08-08.


#### Already used features of Go v1.21

New built-ins `min` and `max` for searching minimum, resp. maximum, value of the given varargs.
Well, that one is not that much useful. Nevertheless, usage of `clear` is incomparably more
useful either for deletion of all key-value pairs from a map, or for zeroing all elements of a slice
(be careful about that much difference of semantics between the different supported argument types).

As far as I know, usage of `init` functions to initialize package requirements is discouraged,
as it brings something one might see as way to inject a kind of hidden behavior. Since v1.21,
package initialization order is specified in a more accurate way. The algorithm is following:
 1. Sort all packages by import path.
 2. If package list is empty, finish. Otherwise, continue.
 3. Find the first fully initialized package.  
 4. Initialize that package.
 5. Remove that package from the list.
 6. Skip back to the step 2.
This reminds me rules of Python's MRO (aka Method Resolution Order) algorithm.

Improvement in type inference — that hit more deeply also generics — are taken for granted.

Introduced [`log/slog`](https://pkg.go.dev/log/slog@go1.21.0) brought already sufficient logger. 
I am still missing some features, but I am trying
to use algorithms provided by [`slices`](https://pkg.go.dev/slices@go1.21.0) 
and [`maps`](https://pkg.go.dev/maps@go1.21.0) packages supporting the basic building blocks of Go.
From package [`cmp`](https://pkg.go.dev/cmp@go1.21.0), I have so far used just 
[`Ordered`](https://pkg.go.dev/cmp@go1.21.0#Ordered) type constraint for a few generic methods.

Just in my small training projects, I found use of functions [`OnceFunc`](https://pkg.go.dev/sync@go1.21.0#OnceFunc) 
and [`OnceValue`](https://pkg.go.dev/sync@go1.21.0#OnceValue) 
of [`sync`](https://pkg.go.dev/sync@go1.21.0) standard package. 
However, I might use them more often to help with common lazy initialization use cases.

Extension of [context](https://pkg.go.dev/context@go1.21.0) has some useful functions.
To get a context that does not get cancelled on cancelling its parent context might be used 
[WithoutCancel](https://pkg.go.dev/context@go1.21.0#WithoutCancel). One can register a function
to execute after a context has been cancelled via [AfterFunc](https://pkg.go.dev/context@go1.21.0#AfterFunc).
Other than that, `Background`-based and `TODO`-based contexts might be considered equal.


#### Not yet used interesting features of Go v1.21

Correct reported error of Go methods on receiver of C type in files that import `"C"`.

Package [`testing/slogtest`](https://pkg.go.dev/testing/slogtest@go1.21.0) for validation of 
[`slog.Handler`](https://pkg.go.dev/log/slog@go1.21.0#Handler) implementations is something I might give a try.

And to name a few interesting bits: 
[`errors.ErrUnsupported`](https://pkg.go.dev/errors@go1.21.0#ErrUnsupported) to denote unsupported operations,
[encoding/binary.NativeEndian](https://pkg.go.dev/encoding/binary@go1.21.0#NativeEndian) for smart conversions,
[math/big.Int.Float64](https://pkg.go.dev/math/big@go1.21.0#Int.Float64) to get the nearest FP value to the given
multi-precision integer together with indication of rounding error, etc.


---

### Go v1.20

Released on 2023-02-01.


#### Already used features of Go v1.20

Wrapping of multiple errors using [`fmt.Errorf`](https://pkg.go.dev/fmt@go1.20.0#Errorf) with `%w` verb.
Functions `errors.Is` and `errors.As` were extended accordingly. An error can wrap multiple errors 
by implementing `Unwrap() []error` method. Also [`errors.Join`](https://pkg.go.dev/errors@go1.20.0#Join)
appeared to return an error wrapping a list of errors.

Package [`runtime/metrics`](https://pkg.go.dev/runtime/metrics@go1.20.0) extended a list of supported
metrics. Time-based histogram metrics are since then less precise, but take up much less memory.


#### Not yet used interesting features of Go v1.20

Simple conversion from a slice to an array, so `[4]byte(slice)` (instead of `*(*[4]byte)(slice)`).

Extended [`unsafe`](https://pkg.go.dev/unsafe@go1.20.0) package with stuff like `SliceData`, `String`,
and `StringData`. 

Packages [`bytes`](https://pkg.go.dev/bytes@go1.20.0) and [`strings`](https://pkg.go.dev/strings@go1.20.0) 
provides new `CutPrefix` and `CutSuffix` functions similar to `TrimPrefix` and `TrimSuffix` but also report 
whether the value was trimmed.

And to name a few interesting bits:
[`context.WithCancelCause`](https://pkg.go.dev/context@go1.20.0#WithCancelCause) to cancel context with the given
error and [`context.Cause`](https://pkg.go.dev/context@go1.20.0#Cause) to retrieve that error,
a few new methods for atomic updates of [`sync.Map`](https://pkg.go.dev/sync@go1.20.0#Map) type

---

### Go v1.19

Released on 2022-08-02.


#### Already used features of Go v1.19

I used at least some of new atomics types like [`sync/atomic.Uint64`](https://pkg.go.dev/sync/atomic@go1.19.0#Uint64)
or [`sync/atomic.Pointer`](https://pkg.go.dev/sync/atomic@go1.19.0#Pointer).

Then [`time.Duration.Abs`](https://pkg.go.dev/time@go1.19.0#Duration.Abs) is one of methods I have already experience
with.

And I was definitely using _pattern-defeating quicksort_ algorithm introduced by 
the [`sort`](https://pkg.go.dev/sort@go1.19.0) package in Go v1.19.


#### Not yet used interesting features of Go v1.19

To be honest, nothing particular seemed important enough to deserve some study time of mine. 

---

### Go v1.18

Released on 2022-03-19.

My first production-grade experience with Go and I already might use generics (so to include type parameters
for function and type definitions). And slowly I used experimental packages `golang.org/x/exp/slices`
and `golang.org/x/exp/maps`.
