---
title: 'Go caveats vol. 1'
tags: [Go]
---

## Go caveats vol. 1

- [Handling nil](#handling-nil)
    - [Nil pointers](#nil-pointers)
    - [Nil slices](#nil-slices)
    - [Nil maps](#nil-maps)
    - [Nil functions](#nil-functions)
    - [Nil method receivers](#nil-method-receivers)
    - [Overview of channel operations](#overview-of-channel-operations)
    - [Nil channels](#nil-channels)
    - [Nil interfaces](#nil-interfaces)
- [Method sets and method receivers](#method-sets-and-method-receivers)
- [Bonus: Unit tests as a proof](#method-sets-and-method-receivers)

---

---

### Handling nil

Go language has something called `nil`, but it might be tricky as it might appear in more context. This section
will take a look on one important aspect connected with `nil`. Go has pointers, but also pointer method receivers,
and a handful of pointer-based types (we will check together slices, maps, functions, channels, but also interfaces).
Let us check each of those in matter of what is still valid and where a panic might occur.

Go is using so-called **fat pointers** that enables not just efficient implementation of slices. Next to pointer
to raw data, one can find additional information like type information or length of slice. One might be always 
aware of existence of fat pointers.


#### Nil pointers

Basic pointers are used in any Go code base. They are used e.g. to:
 - reference bigger data structures;
 - model optional fields (`nil` meaning a missing struct field);
 - implement some of advanced abstract data structures (like linked lists or trees);
 - sharing data and functionality (e.g. logger), etc.

It is easy to dereference a pointer and cause a panic. In the following example is defined variable `numPtr`,
but it is not assigned, so it has default value of pointer type that is `nil` value:

```go
var nilPtr *int

fmt.Printlm(*nilPtr)      // ❌ PANIC ❌ 
```

Issue is not just when you dereference explicitly using `*` unary operator. Go allow us to write simplified
code thanks to some syntax sugars. Ano one of those is hiding the difference between accessing a field of struct
value and accessing a field of struct pointer. As one might see in the following code snippet:

```go
type coord struct {
  X int
  Y int
}

var nilPtr *coord

nilPtr.X = 12             // ❌ PANIC ❌
newY := nilPtr.Y + 1      // ❌ PANIC ❌
```

One might always check the pointer for being non-`nil` before accessing the underlying data:

```go
if nilPtr != nil {
  fmt.Printlm(*nilPtr)
}

// ...

if nilPtr != nil {
  nilPtr.X = 12
  newY := nilPtr.Y + 1
}
```


#### Nil slices

Slice is a fat pointer, so its capacity and length is stored next to a data pointer. This allows us to do many
things with nil slice without causing any panic at all:

```go
var slice []int

fmt.Println(len(slice), cap(slice))   // prints zero twice
for i, x := range slice {
  // ...
}
slice = append(slice, 1, 2, 3)  // after slice is not nil anymore
```

As one might see built-in functions like `cap`, `len`, and `append` are ready for operating on a `nil` slice. 
Similarly, processing of slice in a `for` loop is OK as it internally relies on `len` built-in function.
But one should ensure that slice is not nil as soon as indexing operation is used:

```go
var nilSlice []int

nilSlice[0] = 1           // ❌ PANIC ❌
fmt.Println(nilSlice[5])  // ❌ PANIC ❌
```

It is true, that before slice indexing, checking equality against `nil` might not be enough.
Also, one might notice that panic holds message like `runtime error: index out of range [0] with length 0`
that is different one to all other `nil`-related panics 
(namely `runtime error: invalid memory address or nil pointer dereference`).
One should ensure that all the needed indices are in the valid range of the slice:

```go
var nilSlice []int

if len(nilSlice) > 5 {
  nilSlice[0] = 1
  fmt.Println(nilSlice[5])
} 
```


#### Nil maps

Regarding maps, the rule is — any read operation is safe. So all the following operations are safe:

```go
var nilMap map[int]int

len(nilMap)               // → 0 (containes no pair)
x := nilMap[key]          // → 0 (zero value)
x, found := nilMap[key]   // → 0 and false (zero value and not found)

for k, v := range nilMap {
  // loop body is not executed at all
}
```

Only updating a `nil` map will always panic:

```go
var nilMap map[string]bool

nilMap["true"] = true     // ❌ PANIC ❌
```


#### Nil functions

In Go, one might face with anonymous function (aka λ), closures (proper subset of anonymous functions,
that access variables available in scope of the function where it was instantiated), and regular function pointers.
The issue comes when you call function that is `nil`. 

```go
var nilFunc func()        // similar to `f := (func ())(nil)`

nilFunc()                 // ❌ PANIC ❌
```

One has to check if the given function is not nil before calling it:

```go
func apply(x int, f func(int) int) int {
	if f == nil {
		return x
	}

	return f(x)
}
```


#### Nil method receivers

If the method receiver is a pointer value, one might check it for being not-`nil` before interacting
with receiver in any dangerous way:

```go
type coord struct {
  X int
  Y int
}

func (c *coord) DangerousIsOrigin() bool {
  return c.X == 0 && c.Y == 0   // ❌ PANIC ❌
}

func (c *coord) SafeIsOrigin() bool {
  if c == nil {
    return false
  }
  
  return c.X == 0 && c.Y == 0
}

// ...
  var nilReceiver *coord
  
  if nilReceiver.DangerousIsOrigin() {   // ❌ PANIC is propagated here ❌
    // ...
  }
  
  _ = (*coord)(nil).DangerousIsOrigin()  // ❌ PANIC is propagated here ❌
```

The approach to adopt is present in the `SafeIsStart` in the previous code snippet. Better safe than sorry.

There can be one more caveat connected with method receiver and nil, but this one is might be a bit less obvious.
Thanks to syntax sugar, one might call value receiver method on a nil pointer.

```go
func (coord) ValueReceiver() {}

// ...
  (*coord)(nil).ValueReceiver()    // ❌ PANIC ❌
```


#### Overview of channel operations

Channels are a bit more complex topic, so before hitting .
At first, we will look on closing a channel using `close(c)`:

| Channel state  | Outcome               |
|----------------|-----------------------|
| `nil` channel  | ❌ PANICS ❌            |
| closed channel | ❌ PANICS ❌            |
| open channel   | Closes a channel `c`. |

All scenarios for sending into channel using `ch <- v` are captured by the following table:

| Channel state         | Outcome                    |
|-----------------------|----------------------------|
| `nil` channel         | Blocks forever!            |
| closed channel        | ❌ PANICS ❌                 |
| unbuffered channel    | Blocking until rendezvous. |
| full buffered channel | Blocking until it is full. |
| buffered channel      | Enqueue value in buffer.   |

All the basic scenarios of receiving value from channel can be found in the following table.
We are using full receive statement of `v, o = <-ch`.
However, it is almost the same as "single-result" receive statement `v = <- ch`, just one does not get "open flag".

| Channel state          | Outcome                                       |
|------------------------|-----------------------------------------------|
| `nil` channel          | Blocks forever!                               |
| closed channel         | Always, zero value into `v` and `o == false`. |
| unbuffered channel     | Blocking until rendezvous.                    |
| empty buffered channel | Blocking until rendezvous.                    |
| buffered channel       | Value from buffer into `v` and `o == true`.   |

As one might see, 
**all panic-causing operations are against already closed channel or at least happens during closing a channel**.
Namely:
 - closing of `nil` channel;
 - closing of already closed channel (aka double close);
 - and sending into already closed channel.

All other actions are safe, but some of those might block the current goroutine forever.
Just keep that possible issue in your mind.

Just to note, already closed channel might be also tricky due to immediately receiving zero value.
However, one idiomatic use case rely on receiving from closed channel.
The following code snippet presents that principle:

```go
signalChannel := make(chan struct{})

// producer side - single-shot signalization
close(signalChannel)

// consumer side - repeatable receive of signal by any count of consumers
<-signalChannel
```


#### Nil channels

One should be careful only when trying to close `nil` channel.

```go
var nilChannel chan int

close(nilChannel)   // ❌ PANIC ❌ 
```

Communication via `nil` channel blocks forever.
This is not as dangerous as it might look at the first sight.
If you face `select` with actions on more channels or with `default` branch,
each `select` branch represented by sending/receiving to/from any `nil` channel is just skipped
(not loosing any performance by polling etc.).

```go
func merge(out chan<- int, a, b <-chan int) {
  for a != nil || b != nil {
    select {
    case x, open := <-a:
      if !open {
        log.Println("a set to nil")
        a = nil
        continue
      } else {
        out <- x
      }

    case x, open := <-b:
      if !open {
        log.Println("b set to nil")
        b = nil
        continue
      } else {
        out <- x
      }

    }
  }
  
  close(out)
}

// ...

	go merge(out, a, b)
  for x := range out {
	  fmt.Print(x, ";")
  }
```


#### Nil interfaces

There are two caveats connected with `nil` interface that I faced so far. Let us first check the obvious one.
As one might know, the majority of interfaces provide a method set. And calling any method on `nil` value
usually cause a panic:

```go
var nilInterface error

nilInterface.Error()   // ❌ PANIC ❌
```

The second one is a bit harder for understanding.
One has to know that every variable of interface type is a fat pointer that consists from pointer to held value
(that implements that interface) and pointer to underlying type.

```go
var ptr *int = nil;

if ptr == nil {
  // ✅ will execute
}

var wrapped any = ptr

if wrapped == nil {
  // ❌ will NEVER execute
}
```

What is wrong with that code snippet above? There equality check of `wrapped` against `nil` value that should
be true. It feels to natural. But this comparison fails due to comparing `nil` value typecast into `any` interface.
How does `wrapped` look like? It is of interface type `any`. Any interface value is fat pointer.
The following illustration should give the basic idea why equality check failed:

```text
    wrapped:
         |------|
         | nil  | value pointer
         |------|
         | *int | underlying type
         |------|
         
    nil:
         |------|
         | nil  | value pointer
         |------|
         | nil  | underlying type
         |------|
```


---

### Method sets and method receivers

One does not have to be aware of this tricky part of Go.
In Go specification, _method set_ of a type is defined as methods callable on a operand of that type.
Every type has a possibly empty method set associated with it.


Method set on every defined type `T` depends on the receiver type of caller.
The difference between value receiver and pointer receiver is descibed by the following table.

| Receiver type | Description      | Available method types     |
|---------------|------------------|----------------------------|
| `T`           | value receiver   | `(t T)`                    |
| `*T`          | pointer receiver | both `(t T)` and  `(t *T)` |

That table might be rewritten into a different point of view.
Mind that every method on pointer receiver can be used only on pointer receiver.

| Method type | Description             | Available receivers |
|-------------|-------------------------|---------------------|
| `(t T)`     | value receiver method   | both `T` and `*T`   |
| `(t *T)`    | pointer receiver method | only `*T`           |


#### Trouble-free example

That is true, but Go is using syntax sugar hiding this constraint as far as receiver is accessed directly.
Check the following code snippet. One might expect compilation error on one line calling `sound` method,
but that it not that case. Thanks to syntax sugar, compiler is able to swap type of receiver.


```go
package main

import (
	"fmt"
)

type door struct{}

// Door has a value receiver method.
func (d door) sound() string {
	return "creak"
}

type doorbell struct{}

// Doorbell has a pointer received method.
func (b *doorbell) sound() string {
	return "ding"
}

func main() {
	d := door{}
	db := doorbell{}

	d.sound()
	(&d).sound()
	db.sound()
	(&db).sound()
}
```


#### Troublesome example

The tricky part appears as soon as an instance of defined type is wrapped by interface.
Each call of `loudSound` with direct value as argument will transform that argument's type into interface.
In that case, wrapping of value (not pointer) into interface will fail on compilation due to not available
pointer method of that defined type.


```go
package main

import (
	"fmt"
	"strings"
)

type sounder interface {
	sound() string
}

func loudSound(x sounder) {
	fmt.Println(strings.ToUpper(x.sound()))
}

type door struct{}

// Door has a value receiver method.
func (d door) sound() string {
	return "creak"
}

type doorbell struct{}

// Doorbell has a pointer received method.
func (b *doorbell) sound() string {
	return "ding"
}

func main() {
	d := door{}
	db := doorbell{}
	
	loudSound(d)
	loudSound(&d)
	loudSound(&db)
	
	loudSound(db)   // ❌ point of failure ❌
	// COMPILATION ERROR: cannot use db
	// (variable of struct type doorbell)
	// as sounder value in argument to loudSound:
	// doorbell does not implement sounder
	// (method sound has pointer receiver)
}
```


--- 

### Bonus: Unit tests as a proof

Here is a complete file with unit tests.
You might check yourself if statements about `nil` and panics are still true.

```go
package nilhandling

import (
	"fmt"
	"testing"
	"time"
)

// expectPanic might be used to test if the given function cause a panic.
func expectPanic(t *testing.T, f func()) {
	t.Helper()

	defer func() {
		p := recover()
		t.Logf("caught panic: %v", p)
	}()

	f()

	t.Errorf("expected panic was not caught")
}

// expectDeadlock is simple helper function to test if the given function possible cause a deadlock.
func expectDeadlock(t *testing.T, f func()) {
	t.Helper()

	ch := make(chan struct{})

	go func() {
		f()
		close(ch)
	}()

	select {
	case <-ch:
		t.Errorf("expected deadlock was not caught")

	case <-time.After(1 * time.Second):
		t.Log("caught deadlock")
	}
}

func Test_panic_nilPointer_explicit(t *testing.T) {
	var nilPtr *int

	expectPanic(t, func() {
		fmt.Println(*nilPtr)
	})
}

func Test_panic_nilPointer_hidden(t *testing.T) {
	var nilPtr *coord

	expectPanic(t, func() {
		nilPtr.X = 12
	})

	expectPanic(t, func() {
		newY := nilPtr.Y + 1
		_ = newY
	})
}

func Test_panic_nilSlice_indexing(t *testing.T) {
	var nilSlice []int

	expectPanic(t, func() {
		nilSlice[0] = 1
	})

	expectPanic(t, func() {
		fmt.Println(nilSlice[5])
	})
}

func Test_panic_nilMap_update(t *testing.T) {
	var nilMap map[string]bool

	expectPanic(t, func() {
		nilMap["true"] = true
	})
}

func Test_panic_nilFunction_call(t *testing.T) {
	var nilFunc1 func()
	nilFunc2 := (func())(nil)

	expectPanic(t, func() {
		nilFunc1()
	})

	expectPanic(t, func() {
		nilFunc2()
	})
}

func Test_panic_nilReceiver_call(t *testing.T) {
	var nilReceiver *coord

	expectPanic(t, func() {
		_ = nilReceiver.DangerousIsOrigin()
	})

	expectPanic(t, func() {
		_ = (*coord)(nil).DangerousIsOrigin()
	})
}

func Test_panic_nilReceiver_methodOfValueReceiver(t *testing.T) {
	expectPanic(t, func() {
		(*coord)(nil).ValueReceiver()
	})
}

func Test_deadlock_nilChannel_send(t *testing.T) {
	var nilChannel chan struct{}

	expectDeadlock(t, func() {
		nilChannel <- struct{}{}
	})
}

func Test_deadlock_nilChannel_receive(t *testing.T) {
	var nilChannel chan struct{}

	expectDeadlock(t, func() {
		<-nilChannel
	})
}

func Test_panic_nilChannel_close(t *testing.T) {
	var nilChannel chan struct{}

	expectPanic(t, func() {
		close(nilChannel)
	})
}

func Test_panic_closedChannel_close(t *testing.T) {
	ch := make(chan struct{})
	close(ch)
	closedChannel := ch

	expectPanic(t, func() {
		close(closedChannel)
	})
}

func Test_panic_closedChannel_send(t *testing.T) {
	ch := make(chan struct{})
	close(ch)
	closedChannel := ch

	expectPanic(t, func() {
		closedChannel <- struct{}{}
	})
}

func Test_panic_nilInterface_call(t *testing.T) {
	var nilInterface error

	expectPanic(t, func() {
		nilInterface.Error()
	})
}

func Test_invalidNilComparison_nilInterface(t *testing.T) {
	var ptr *int

	if ptr != nil {
		t.Errorf("expected to be nil")
	}

	var wrapped any = ptr

	if wrapped == nil {
		t.Errorf("expected to be non-nil")
	}
}

// coord is just a sample object used by unit tests.
type coord struct {
	X int
	Y int
}

func (c *coord) DangerousIsOrigin() bool {
	return c.X == 0 && c.Y == 0 // ❌ PANIC ❌
}

func (c *coord) SafeIsOrigin() bool {
	if c == nil {
		return false
	}

	return c.X == 0 && c.Y == 0
}

func (coord) ValueReceiver() {}

```
