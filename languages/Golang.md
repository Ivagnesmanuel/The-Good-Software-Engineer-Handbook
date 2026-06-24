# Go Cheatsheet

A quick-reference guide covering Go's core concepts, syntax, and common patterns.

---

## Table of Contents

1. [Variables and Types](#variables-and-types)
2. [Control Flow](#control-flow)
3. [Functions](#functions)
4. [Structs](#structs)
5. [Methods and Interfaces](#methods-and-interfaces)
6. [Error Handling](#error-handling)
7. [Pointers](#pointers)
8. [Arrays, Slices, and Maps](#arrays-slices-and-maps)
9. [Strings](#strings)
10. [Generics](#generics)
11. [Concurrency](#concurrency)
12. [Channels](#channels)
13. [Select](#select)
14. [Context](#context)
15. [Closures](#closures)
16. [Defer, Panic, and Recover](#defer-panic-and-recover)
17. [Type Assertions and Type Switches](#type-assertions-and-type-switches)
18. [Embedding](#embedding)
19. [Packages and Modules](#packages-and-modules)
20. [Testing](#testing)
21. [Common Standard Library Packages](#common-standard-library-packages)
22. [Go Toolchain](#go-toolchain)
23. [Build Tags and Conditional Compilation](#build-tags-and-conditional-compilation)
24. [CGo](#cgo)

---

## Variables and Types

```go
// Short declaration (type inferred, only inside functions)
x := 5

// Var declaration (can be at package level)
var y int = 10
var z int // zero value: 0

// Multiple declarations
var (
    name   string = "Alice"
    age    int    = 30
    active bool   = true
)

// Constants (evaluated at compile time, untyped until used)
const MaxPoints = 100_000
const Pi float64 = 3.14159

// Constant block with iota (auto-incrementing)
const (
    Sunday    = iota // 0
    Monday           // 1
    Tuesday          // 2
    Wednesday        // 3
)

// Iota patterns
const (
    _  = iota             // skip 0
    KB = 1 << (10 * iota) // 1024
    MB                    // 1048576
    GB                    // 1073741824
)
```

### Basic Types

| Type | Variants | Zero Value |
|------|----------|------------|
| Integer | `int`, `int8`, `int16`, `int32`, `int64` | `0` |
| Unsigned | `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr` | `0` |
| Float | `float32`, `float64` | `0.0` |
| Complex | `complex64`, `complex128` | `(0+0i)` |
| Boolean | `bool` | `false` |
| String | `string` | `""` |
| Byte | `byte` (alias for `uint8`) | `0` |
| Rune | `rune` (alias for `int32`, represents a Unicode code point) | `0` |

### Type Conversions

Go has no implicit conversions. All conversions are explicit.

```go
x := 42
y := float64(x)
z := int64(x)
s := string(rune(65)) // "A"

// strconv for string <-> number
import "strconv"
i, err := strconv.Atoi("42")       // string -> int
s := strconv.Itoa(42)               // int -> string
f, err := strconv.ParseFloat("3.14", 64)
s := strconv.FormatFloat(3.14, 'f', 2, 64)
```

### Type Aliases and Defined Types

```go
// Type alias (identical to the underlying type)
type Byte = uint8

// Defined type (new distinct type, has its own method set)
type UserID int64
type Celsius float64
```

---

## Control Flow

```go
// if / else (no parentheses around condition, braces required)
if x > 0 {
    fmt.Println("positive")
} else if x == 0 {
    fmt.Println("zero")
} else {
    fmt.Println("negative")
}

// if with init statement (scoped to the if/else chain)
if err := doSomething(); err != nil {
    return err
}

// for (the only loop keyword in Go)
for i := 0; i < 10; i++ { }

// while-style
for condition { }

// infinite loop
for { }

// for range over slice
for i, v := range slice { }      // index and value
for _, v := range slice { }      // value only
for i := range slice { }         // index only

// for range over map
for k, v := range m { }
for k := range m { }             // keys only

// for range over string (iterates by rune, not byte)
for i, r := range "hello" { }   // i = byte offset, r = rune

// for range over channel
for msg := range ch { }          // reads until channel is closed

// for range over integer (Go 1.22+)
for i := range 10 { }            // 0, 1, 2, ..., 9

// switch (no fallthrough by default, no break needed)
switch value {
case 1:
    fmt.Println("one")
case 2, 3:
    fmt.Println("two or three")
default:
    fmt.Println("other")
}

// switch with no condition (cleaner if/else chain)
switch {
case x > 0:
    fmt.Println("positive")
case x == 0:
    fmt.Println("zero")
default:
    fmt.Println("negative")
}

// switch with init statement
switch v := compute(); {
case v > 100:
    fmt.Println("large")
default:
    fmt.Println("small")
}

// Explicit fallthrough (falls into the next case body)
switch x {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("one or two")
}

// Labels and goto
Loop:
    for i := 0; i < 10; i++ {
        for j := 0; j < 10; j++ {
            if condition {
                break Loop    // break the outer loop
            }
        }
    }
```

---

## Functions

```go
func add(a, b int) int {
    return a + b
}

// Multiple return values
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Named return values (initialized to zero values, returned by bare return)
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // returns x, y
}

// Variadic functions
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
// Call: sum(1, 2, 3) or sum(slice...)

// Functions are first-class values
var fn func(int) int
fn = func(x int) int { return x * 2 }

// Function types
type Transformer func(string) string
```

---

## Structs

```go
type User struct {
    Name   string
    Age    int
    Active bool
}

// Initialization
u1 := User{Name: "Alice", Age: 30, Active: true}
u2 := User{"Bob", 25, true} // positional (fragile, avoid in public APIs)
var u3 User                  // zero value: {"", 0, false}

// Field access
u1.Name = "Alicia"

// Pointer to struct (field access auto-dereferences)
p := &User{Name: "Charlie", Age: 35}
p.Name = "Chuck" // equivalent to (*p).Name

// Anonymous struct (useful for one-off data, JSON, tests)
point := struct {
    X, Y int
}{10, 20}

// Struct tags (metadata for encoding, validation, etc.)
type Config struct {
    Host    string `json:"host" yaml:"host"`
    Port    int    `json:"port,omitempty"`
    Secret  string `json:"-"` // excluded from JSON
}
```

### Struct Comparison and Copying

```go
// Structs are comparable if all fields are comparable
a := User{Name: "Alice", Age: 30}
b := User{Name: "Alice", Age: 30}
fmt.Println(a == b) // true

// Structs are copied by value (no references, independent copies)
c := a
c.Name = "Bob" // a.Name is still "Alice"

// Do NOT copy a struct after first use if it contains a sync.Mutex, sync.WaitGroup,
// or sync/atomic type: the copy gets an independent, unsynchronized lock.
// Pass such structs by pointer. `go vet` flags this (copylocks).
```

---

## Methods and Interfaces

### Methods

Methods are functions with a receiver. The receiver binds the method to a type.

```go
type Rectangle struct {
    Width, Height float64
}

// Value receiver (operates on a copy)
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver (can modify the original, avoids copy for large structs)
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

r := Rectangle{Width: 10, Height: 5}
a := r.Area()   // value receiver: callable on both T and *T
r.Scale(2.0)    // pointer receiver: callable here only because r is addressable

// Method sets (govern interface satisfaction):
//   type T   -> value-receiver methods only
//   type *T  -> value- AND pointer-receiver methods
// A value of T in an interface does NOT satisfy an interface that requires
// a pointer-receiver method -- store &T instead.
```

### Interfaces

Interfaces define behavior. A type implements an interface implicitly -- no `implements` keyword.

```go
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Rectangle satisfies Shape because it has both methods
func PrintShape(s Shape) {
    fmt.Printf("Area: %.2f, Perimeter: %.2f\n", s.Area(), s.Perimeter())
}
```

### Interface Composition

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composed interface
type ReadWriter interface {
    Reader
    Writer
}
```

### The Empty Interface and `any`

```go
// any is an alias for interface{} (Go 1.18+)
func printAnything(v any) {
    fmt.Println(v)
}

// Useful for heterogeneous collections
items := []any{42, "hello", true, 3.14}
```

### Common Standard Interfaces

| Interface | Package | Method(s) |
|-----------|---------|-----------|
| `Stringer` | `fmt` | `String() string` |
| `Error` | `builtin` | `Error() string` |
| `Reader` | `io` | `Read([]byte) (int, error)` |
| `Writer` | `io` | `Write([]byte) (int, error)` |
| `Closer` | `io` | `Close() error` |
| `Handler` | `net/http` | `ServeHTTP(ResponseWriter, *Request)` |
| `Sort` | `sort` | `Len()`, `Less(i, j)`, `Swap(i, j)` |
| `Marshaler` | `encoding/json` | `MarshalJSON() ([]byte, error)` |
| `Unmarshaler` | `encoding/json` | `UnmarshalJSON([]byte) error` |

---

## Error Handling

Go uses explicit error returns instead of exceptions. The `error` interface is a single method: `Error() string`.

```go
// Checking errors (the fundamental pattern)
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err) // wrap with context
}

// Creating errors
import "errors"

err := errors.New("something went wrong")
err := fmt.Errorf("failed to process %s: %w", name, originalErr) // wrap
```

### Custom Error Types

```go
type NotFoundError struct {
    Resource string
    ID       int
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s with ID %d not found", e.Resource, e.ID)
}

// Return as error interface
func findUser(id int) (*User, error) {
    return nil, &NotFoundError{Resource: "user", ID: id}
}
```

```go
// Typed nil in an interface is NOT nil.
// An interface value is (type, value); it equals nil only when BOTH are nil.
func find() error {
    var e *NotFoundError // nil pointer
    return e             // returns a non-nil error: (type=*NotFoundError, value=nil)
}
if find() != nil { /* TRUE -- surprises callers */ }
// Fix: return a literal nil on the no-error path; never return a typed nil pointer as error.
```

### errors.Is and errors.As (Go 1.13+)

```go
// errors.Is -- checks if any error in the chain matches a target value
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("file not found")
}

// errors.As -- checks if any error in the chain matches a target type
var nfe *NotFoundError
if errors.As(err, &nfe) {
    fmt.Printf("resource: %s, id: %d\n", nfe.Resource, nfe.ID)
}

// errors.Join -- combine multiple errors (Go 1.20+)
err := errors.Join(err1, err2, err3)
```

### Sentinel Errors

```go
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
)

func fetch(id int) (*Item, error) {
    // ...
    return nil, fmt.Errorf("fetch item %d: %w", id, ErrNotFound)
}

// Caller:
if errors.Is(err, ErrNotFound) { }
```

---

## Pointers

Go has pointers but no pointer arithmetic.

```go
x := 42
p := &x      // p is *int, points to x
fmt.Println(*p) // 42, dereference
*p = 100     // x is now 100

// new allocates zeroed memory and returns a pointer
p := new(int) // *int, *p == 0

// nil is the zero value for pointers
var p *int    // p == nil
```

### When to Use Pointers

- Pointer receivers: when methods need to modify the receiver, or to avoid copying large structs.
- Shared state: when multiple parts of the code need to observe mutations.
- Optional values: `*T` can be `nil`, signaling "no value" (e.g., optional JSON fields).
- Maps, channels, and functions are reference types -- no need to pass pointers to them. Slices are *almost* one: the slice header (pointer, len, cap) is copied by value, so a callee can mutate existing elements but its `append`/reslice growth is invisible to the caller -- return the slice, or pass `*[]T`, when the length may change.

---

## Arrays, Slices, and Maps

### Arrays

```go
// Fixed size, values (rarely used directly -- slices are preferred)
var arr [5]int                // [0, 0, 0, 0, 0]
arr := [5]int{1, 2, 3, 4, 5}
arr := [...]int{1, 2, 3}     // length inferred: [3]int
```

### Slices

Slices are dynamic views over arrays. They are the workhorse collection type.

```go
// Creation
s := []int{1, 2, 3, 4, 5}
s := make([]int, 5)        // len=5, cap=5, all zeros
s := make([]int, 0, 10)    // len=0, cap=10

// Append (may allocate a new backing array)
s = append(s, 6)
s = append(s, 7, 8, 9)
s = append(s, other...)    // spread another slice

// Slicing (low:high, shares backing array)
sub := s[1:4]   // elements at index 1, 2, 3
sub := s[:3]    // 0, 1, 2
sub := s[2:]    // 2 to end
sub := s[:]     // full copy of the slice header

// Full slice expression (low:high:max, limits capacity)
sub := s[1:4:4] // len=3, cap=3 (append won't overwrite s)

// Aliasing: a sub-slice shares the backing array until a grow reallocates.
a := []int{1, 2, 3, 4}
b := a[:2]
b = append(b, 99) // overwrites a[2]: a is now [1 2 99 4]
// Force a private array with a full slice expression: b := a[:2:2]

// Length and capacity
len(s)  // number of elements
cap(s)  // capacity of the backing array

// Copy
dst := make([]int, len(src))
copy(dst, src) // returns number of elements copied

// Delete element at index i (preserving order).
// Leaves a stale reference in the vacated tail slot -- a leak if elements hold pointers.
s = append(s[:i], s[i+1:]...)

// Delete element at index i (Go 1.21+ slices package).
// Preferred: it zeroes the vacated tail, so removed elements can be garbage-collected.
import "slices"
s = slices.Delete(s, i, i+1)

// Nil vs empty slice
var s []int         // nil, len=0, cap=0
s := []int{}        // non-nil, len=0, cap=0
s := make([]int, 0) // non-nil, len=0, cap=0
// json.Marshal: nil -> null, empty -> []
```

### slices Package (Go 1.21+)

```go
import (
    "cmp"
    "slices"
)

slices.Sort(s)
slices.SortFunc(s, func(a, b int) int { return cmp.Compare(a, b) })
slices.Contains(s, 42)
slices.Index(s, 42)       // -1 if not found
slices.Equal(a, b)
slices.Reverse(s)
slices.Compact(s)          // remove consecutive duplicates
slices.Min(s)
slices.Max(s)
slices.Clone(s)
slices.BinarySearch(s, 42)
```

### Maps

```go
// Creation
m := map[string]int{"a": 1, "b": 2}
m := make(map[string]int)
m := make(map[string]int, 100) // capacity hint

// Access
v := m["key"]             // zero value if missing
v, ok := m["key"]         // comma-ok idiom
if v, ok := m["key"]; ok {
    fmt.Println(v)
}

// Set and delete
m["key"] = 42
delete(m, "key")

// Iteration (order is randomized)
for k, v := range m { }

// Length
len(m)

// Nil map: reads return zero values, writes panic
var m map[string]int // nil
_ = m["key"]         // ok, returns 0
// m["key"] = 1      // panic!

// Plain maps are NOT safe for concurrent access. A concurrent write
// (with another read or write) is a fatal runtime error -- not a recoverable
// panic. Guard with sync.Mutex/RWMutex or use sync.Map. See `Concurrency.md`.
```

### maps Package (Go 1.21+)

```go
import "maps"

maps.Keys(m)      // returns iter.Seq[K]
maps.Values(m)    // returns iter.Seq[V]
maps.Equal(a, b)
maps.Clone(m)
maps.Copy(dst, src)
maps.DeleteFunc(m, func(k string, v int) bool { return v < 0 })
```

---

## Strings

Strings are immutable byte slices. For Unicode-aware operations, work with runes.

```go
s := "hello, world"
r := 'A'                     // rune literal (int32)
raw := `no \n escapes here`  // raw string literal (backticks)
```

### Manipulation

```go
import "strings"

strings.Contains(s, "world")
strings.HasPrefix(s, "hello")
strings.HasSuffix(s, "world")
strings.Index(s, "world")        // -1 if not found
strings.Count(s, "l")

strings.ToUpper(s)
strings.ToLower(s)
strings.Title(s)                 // deprecated, use cases.Title
strings.TrimSpace(s)
strings.Trim(s, " \t")
strings.TrimPrefix(s, "hello")
strings.TrimSuffix(s, "world")

strings.Replace(s, "old", "new", -1) // -1 = replace all
strings.ReplaceAll(s, "old", "new")   // Go 1.12+

strings.Split(s, ",")
strings.SplitN(s, ",", 2)       // at most 2 parts
strings.Fields(s)                // split on whitespace
strings.Join(parts, ", ")

strings.Repeat("ha", 3)         // "hahaha"
strings.Map(unicode.ToUpper, s) // transform each rune

strings.EqualFold(a, b)         // case-insensitive comparison
```

### strings.Builder (efficient string concatenation)

```go
var b strings.Builder
b.WriteString("hello")
b.WriteString(", ")
b.WriteString("world")
result := b.String() // "hello, world"
```

### Rune Iteration

```go
s := "Hello, 世界"

// By byte
for i := 0; i < len(s); i++ {
    fmt.Printf("%x ", s[i])
}

// By rune (Unicode code point)
for i, r := range s {
    fmt.Printf("%d: %c\n", i, r) // i = byte offset
}

// Rune count
utf8.RuneCountInString(s) // 9 (not len(s) which is 13 bytes)

// Convert between string and runes/bytes
runes := []rune(s)
bytes := []byte(s)
s = string(runes)
s = string(bytes)
```

### fmt Formatting

| Verb | Meaning | Example |
|------|---------|---------|
| `%v` | Default format | `{Alice 30}` |
| `%+v` | With field names | `{Name:Alice Age:30}` |
| `%#v` | Go syntax | `main.User{Name:"Alice", Age:30}` |
| `%T` | Type | `main.User` |
| `%d` | Decimal integer | `42` |
| `%b` | Binary | `101010` |
| `%x` / `%X` | Hex lower / upper | `2a` / `2A` |
| `%o` | Octal | `52` |
| `%f` | Float | `3.140000` |
| `%.2f` | Float, 2 decimals | `3.14` |
| `%e` | Scientific | `3.14e+00` |
| `%s` | String | `hello` |
| `%q` | Quoted string | `"hello"` |
| `%c` | Character (rune) | `A` |
| `%p` | Pointer | `0xc0000b4000` |
| `%10d` | Right-align, width 10 | `"        42"` |
| `%-10d` | Left-align, width 10 | `"42        "` |
| `%010d` | Zero-padded | `0000000042` |

---

## Generics

Go 1.18 introduced type parameters for functions, types, and interfaces.

```go
// Generic function
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
// Call: Min(3, 5) or Min[int](3, 5)

// Generic type
type Pair[T, U any] struct {
    First  T
    Second U
}

func NewPair[T, U any](first T, second U) Pair[T, U] {
    return Pair[T, U]{First: first, Second: second}
}
```

### Type Constraints

```go
import "golang.org/x/exp/constraints" // since Go 1.21, cmp.Ordered (std "cmp") replaces constraints.Ordered

// Built-in constraints
// any          -- alias for interface{}, allows any type
// comparable   -- supports == and != (usable as map keys)

// Custom constraint
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~float32 | ~float64
}

// ~T means "any type whose underlying type is T" (includes defined types)
type Celsius float64 // satisfies ~float64

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// Interface with methods as constraint
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}
```

### cmp Package (Go 1.21+)

```go
import "cmp"

cmp.Less(3, 5)           // true
cmp.Compare(3, 5)        // -1, 0, or 1
cmp.Or(a, b, c)          // returns first non-zero value (Go 1.22+)
```

---

## Concurrency

For the underlying model — happens-before, the Go memory model, the M:N scheduler, deadlock conditions, and the race detector — see `Concurrency.md`.

### Goroutines

Goroutines are lightweight, multiplexed onto OS threads by the Go runtime.

```go
// Launch a goroutine
go doWork()

// With a closure
go func() {
    fmt.Println("running concurrently")
}()

// Passing the loop value as an argument (explicit; also safe to capture directly,
// since each iteration now has its own copy of the loop variable)
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

### sync.WaitGroup

```go
import "sync"

var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        fmt.Println(n)
    }(i)
}
wg.Wait() // block until all goroutines finish
```

### sync.Mutex and sync.RWMutex

```go
var (
    mu      sync.Mutex
    counter int
)

// Exclusive lock
mu.Lock()
counter++
mu.Unlock()

// Idiomatic: defer Unlock
mu.Lock()
defer mu.Unlock()
counter++

// RWMutex: multiple concurrent readers OR one exclusive writer
var rw sync.RWMutex
rw.RLock()          // shared read lock
defer rw.RUnlock()
// ...
rw.Lock()           // exclusive write lock
defer rw.Unlock()
```

### sync.Once

```go
var (
    once     sync.Once
    instance *Config
)

func GetConfig() *Config {
    once.Do(func() {
        instance = loadConfig()
    })
    return instance
}
```

### sync.Map (concurrent-safe map)

```go
var m sync.Map

m.Store("key", 42)
v, ok := m.Load("key")
m.Delete("key")
m.Range(func(key, value any) bool {
    fmt.Println(key, value)
    return true // continue iteration
})
```

### Atomic Operations

```go
import "sync/atomic"

var counter atomic.Int64

counter.Add(1)
counter.Store(0)
v := counter.Load()
swapped := counter.CompareAndSwap(oldVal, newVal)
```

---

## Channels

Channels are typed conduits for communication between goroutines.

```go
// Unbuffered channel (synchronous: send blocks until a receiver is ready)
ch := make(chan int)

// Buffered channel (asynchronous up to capacity)
ch := make(chan int, 100)

// Send and receive
ch <- 42      // send
v := <-ch     // receive
v, ok := <-ch // ok is false if channel is closed and drained

// Close (only sender should close, signals no more values)
close(ch)

// Channel runtime rules:
//   close(closedCh) -> panic: close of closed channel
//   close(nilCh)    -> panic: close of nil channel
//   ch <- v         -> panic if ch is already closed
//   <-closedCh      -> no block: zero value immediately (ok == false)
//   <-nilCh         -> blocks forever (a nil channel disables a select case)

// Range over channel (reads until closed)
for msg := range ch {
    fmt.Println(msg)
}

// Directional channel types (for function signatures)
func producer(out chan<- int) { } // send-only
func consumer(in <-chan int) { }  // receive-only
```

### Channel Patterns

```go
// Fan-out: distribute work across goroutines
for i := 0; i < numWorkers; i++ {
    go worker(jobs, results)
}

// Fan-in: merge multiple channels into one
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

// Done channel (signal cancellation)
done := make(chan struct{})
go func() {
    // work...
    close(done) // broadcast to all readers
}()
<-done // wait for signal

// Semaphore (limit concurrency)
sem := make(chan struct{}, maxConcurrent)
for _, task := range tasks {
    sem <- struct{}{} // acquire
    go func(t Task) {
        defer func() { <-sem }() // release
        process(t)
    }(task)
}
```

---

## Select

Select lets a goroutine wait on multiple channel operations.

```go
select {
case msg := <-ch1:
    fmt.Println("received from ch1:", msg)
case ch2 <- value:
    fmt.Println("sent to ch2")
case <-time.After(5 * time.Second):
    fmt.Println("timed out")
default:
    fmt.Println("no channel ready") // non-blocking select
}

// Common: select in a loop with a done/quit channel
for {
    select {
    case msg := <-msgCh:
        process(msg)
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Ticker (periodic events)
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
for {
    select {
    case <-ticker.C:
        fmt.Println("tick")
    case <-done:
        return
    }
}
```

---

## Context

The `context` package carries deadlines, cancellation signals, and request-scoped values across API boundaries and goroutines.

```go
import "context"

// Background context (root, never canceled)
ctx := context.Background()

// With cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// With timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// With deadline
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(1*time.Hour))
defer cancel()

// Checking cancellation
select {
case <-ctx.Done():
    return ctx.Err() // context.Canceled or context.DeadlineExceeded
default:
    // continue work
}

// With value (use sparingly, prefer explicit parameters)
type ctxKey string
ctx = context.WithValue(ctx, ctxKey("requestID"), "abc-123")
reqID := ctx.Value(ctxKey("requestID")).(string)

// Cause (Go 1.20+)
ctx, cancel := context.WithCancelCause(context.Background())
cancel(fmt.Errorf("shutdown requested"))
fmt.Println(context.Cause(ctx)) // "shutdown requested"

// AfterFunc (Go 1.21+)
stop := context.AfterFunc(ctx, func() {
    cleanup()
})
defer stop()

// Passing context: always the first parameter, named ctx
func DoWork(ctx context.Context, input string) error { }
```

---

## Closures

```go
// Basic closure
add := func(a, b int) int { return a + b }
fmt.Println(add(1, 2))

// Closures capture variables by reference
counter := 0
inc := func() int {
    counter++
    return counter
}
fmt.Println(inc()) // 1
fmt.Println(inc()) // 2

// Returning a closure
func makeMultiplier(factor int) func(int) int {
    return func(x int) int {
        return x * factor
    }
}
double := makeMultiplier(2)
fmt.Println(double(5)) // 10

// Immediately invoked
result := func(x int) int {
    return x * x
}(5) // 25
```

---

## Defer, Panic, and Recover

### Defer

Deferred calls execute in LIFO (last-in, first-out) order when the enclosing function returns.

```go
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close() // runs when readFile returns

    return io.ReadAll(f)
}

// Defer evaluates arguments immediately but delays execution
for i := 0; i < 3; i++ {
    defer fmt.Println(i) // prints 2, 1, 0
}

// Defers run at function return, not at end of loop iteration:
//   for _, p := range paths { f, _ := os.Open(p); defer f.Close() } // leaks until return
// Move the body into a helper func (or Close explicitly) to release per iteration.

// Common: defer with named return values for error annotation
func doWork() (err error) {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer func() {
        if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()
    // ... work with tx ...
    return nil
}
```

### Panic and Recover

```go
// Panic halts normal execution, runs deferred functions, then crashes
panic("something went terribly wrong")

// Recover catches a panic (only works inside a deferred function)
func safeCall(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    f()
    return nil
}

// When to panic: truly unrecoverable situations (programmer errors,
// corrupted state). Prefer returning errors for expected failures.
```

---

## Type Assertions and Type Switches

```go
// Type assertion (extract concrete type from interface)
var i any = "hello"

s := i.(string)     // panics if i is not a string
s, ok := i.(string) // ok is false if i is not a string

// Type switch
switch v := i.(type) {
case string:
    fmt.Println("string:", v)
case int:
    fmt.Println("int:", v)
case nil:
    fmt.Println("nil")
default:
    fmt.Printf("unknown type: %T\n", v)
}
```

---

## Embedding

Go uses composition instead of inheritance. Embedding promotes fields and methods.

```go
// Struct embedding
type Base struct {
    ID int
}

func (b Base) Describe() string {
    return fmt.Sprintf("ID: %d", b.ID)
}

type User struct {
    Base           // embedded (not a field name in usage)
    Name string
}

u := User{Base: Base{ID: 1}, Name: "Alice"}
u.Describe() // promoted from Base
u.ID          // promoted field access
u.Base.ID     // explicit access also works

// Interface embedding (composition of interfaces)
type ReadCloser interface {
    io.Reader
    io.Closer
}
```

---

## Packages and Modules

### Package Structure

```
myproject/
  go.mod
  go.sum
  main.go               // package main
  config/
    config.go            // package config
  internal/              // only importable by parent module
    auth/
      auth.go            // package auth
  pkg/                   // convention for public library code
    util/
      util.go            // package util
  cmd/
    server/
      main.go            // package main (separate binary)
    cli/
      main.go            // package main
```

### init Functions

```go
// init() runs automatically when the package is imported, before main.
// A package (or even a single file) can have multiple init functions.
func init() {
    // register drivers, validate config, etc.
}
```

### Visibility

```go
// Exported: starts with uppercase (accessible from other packages)
func PublicFunction() {}
type PublicType struct {
    PublicField  int
    privateField int // unexported, only accessible within the package
}

// Unexported: starts with lowercase
func privateHelper() {}
```

### Imports

```go
import "fmt"
import "os"

// Grouped imports
import (
    "fmt"
    "os"

    "github.com/user/repo/internal/config" // third-party or internal
)

// Import aliases
import (
    "fmt"
    mrand "math/rand" // alias to avoid collision
    _ "image/png"     // blank import: run init() only, don't use the package
    . "math"          // dot import: use Sqrt() instead of math.Sqrt() (avoid)
)
```

### Go Modules

```bash
go mod init github.com/user/project  # create go.mod
go mod tidy                           # add missing, remove unused deps
go mod download                       # download dependencies
go mod vendor                         # copy deps into vendor/
go mod graph                          # print dependency graph
go get github.com/pkg/errors@v0.9.1   # add/update a dependency
go get -u ./...                       # update all dependencies
```

```
// go.mod
module github.com/user/project

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    golang.org/x/sync v0.6.0
)
```

### Workspaces (Go 1.18+)

Workspaces let you work on multiple modules simultaneously without publishing or using `replace` directives in `go.mod`. Useful for developing a library and its consumer side-by-side.

```
myworkspace/
  go.work
  services/
    api/
      go.mod          // module github.com/user/api
      main.go
    worker/
      go.mod          // module github.com/user/worker
      main.go
  libs/
    shared/
      go.mod          // module github.com/user/shared
      shared.go
```

```bash
# Initialize a workspace (creates go.work in the current directory)
go work init ./services/api ./services/worker ./libs/shared

# Add a module to an existing workspace
go work use ./libs/newmodule

# Add all modules under a directory
go work use -r ./libs

# Sync workspace go.work with module go.mod files
go work sync
```

```
// go.work
go 1.22

use (
    ./services/api
    ./services/worker
    ./libs/shared
)
```

Now `services/api` can import `github.com/user/shared` and Go resolves it from the local `libs/shared` directory automatically -- no `replace` in `go.mod` needed.

```bash
# Build/test across the whole workspace
go build ./...
go test ./...

# Build a specific module
go build ./services/api/...

# Ignore the workspace (use go.mod only)
GOWORK=off go build ./...
```

Workspace rules:
- `go.work` should **not** be committed if the workspace is developer-local (e.g., local overrides). **Do** commit it for monorepo-style projects where all modules live together.
- `go.work.sum` tracks checksums for workspace dependencies (analogous to `go.sum`).
- `go work edit` provides programmatic editing of `go.work`, similar to `go mod edit`.


---

## Testing

### Test Files

Test files live alongside the code they test, suffixed with `_test.go`.

```go
// math.go
package math

func Add(a, b int) int { return a + b }
```

```go
// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    got := Add(2, 3)
    want := 5
    if got != want {
        t.Errorf("Add(2, 3) = %d, want %d", got, want)
    }
}

// Table-driven tests (idiomatic Go)
func TestAddTable(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positives", 2, 3, 5},
        {"negatives", -1, -2, -3},
        {"zero", 0, 0, 0},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

### Test Helpers

```go
// t.Helper() marks a function as a helper (errors report caller's line)
func assertEqual(t *testing.T, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("got %d, want %d", got, want)
    }
}

// t.Cleanup registers cleanup to run after the test
func TestWithTempDir(t *testing.T) {
    dir := t.TempDir() // auto-cleaned up
    t.Cleanup(func() {
        // additional cleanup if needed
    })
}

// t.Parallel() marks a test for parallel execution
func TestParallel(t *testing.T) {
    t.Parallel()
    // ...
}

// t.Skip
func TestLinuxOnly(t *testing.T) {
    if runtime.GOOS != "linux" {
        t.Skip("linux only")
    }
}
```

### Benchmarks

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

// Run: go test -bench=. -benchmem
```

### Fuzz Tests (Go 1.18+)

```go
func FuzzAdd(f *testing.F) {
    f.Add(1, 2) // seed corpus
    f.Fuzz(func(t *testing.T, a, b int) {
        result := Add(a, b)
        if result != a+b {
            t.Errorf("Add(%d, %d) = %d", a, b, result)
        }
    })
}

// Run: go test -fuzz=FuzzAdd
```

### TestMain

```go
// TestMain controls test execution for the package (setup/teardown)
func TestMain(m *testing.M) {
    setup()
    code := m.Run()
    teardown()
    os.Exit(code)
}
```

---

## Common Standard Library Packages

### encoding/json

```go
import "encoding/json"

// Marshal (struct -> JSON bytes)
data, err := json.Marshal(user)
data, err := json.MarshalIndent(user, "", "  ")

// Unmarshal (JSON bytes -> struct)
var user User
err := json.Unmarshal(data, &user)

// Decoder/Encoder (streaming, for io.Reader/Writer)
decoder := json.NewDecoder(r.Body)
err := decoder.Decode(&user)

encoder := json.NewEncoder(w)
err := encoder.Encode(user)

// Dynamic JSON (unknown structure)
var result map[string]any
err := json.Unmarshal(data, &result)
```

### net/http

```go
import "net/http"

// Simple server (Go 1.22+ enhanced routing)
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    fmt.Fprintf(w, "User %s", id)
})
http.ListenAndServe(":8080", mux)

// HTTP client
resp, err := http.Get("https://api.example.com/data")
if err != nil {
    return err
}
defer resp.Body.Close()
body, err := io.ReadAll(resp.Body)

// Custom client with timeout
client := &http.Client{Timeout: 10 * time.Second}
req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
req.Header.Set("Content-Type", "application/json")
resp, err := client.Do(req)
```

### os and io

```go
// Read file
data, err := os.ReadFile("config.json")

// Write file
err := os.WriteFile("output.txt", data, 0644)

// Open for streaming
f, err := os.Open("input.txt")       // read-only
f, err := os.Create("output.txt")    // create/truncate
f, err := os.OpenFile("log.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
defer f.Close()

// io helpers
data, err := io.ReadAll(reader)
n, err := io.Copy(dst, src)
reader := io.LimitReader(r, 1024*1024) // limit to 1MB
```

### time

```go
import "time"

now := time.Now()
t := time.Date(2024, time.March, 15, 10, 30, 0, 0, time.UTC)

d := 5 * time.Second
time.Sleep(d)

// Duration arithmetic
elapsed := time.Since(start)      // time.Now().Sub(start)
remaining := time.Until(deadline) // deadline.Sub(time.Now())

// Formatting (uses reference time: Mon Jan 2 15:04:05 MST 2006)
s := t.Format("2006-01-02 15:04:05")
s := t.Format(time.RFC3339)
t, err := time.Parse("2006-01-02", "2024-03-15")

// Timers and tickers
timer := time.NewTimer(5 * time.Second)
<-timer.C // fires once

ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
for range ticker.C { } // fires repeatedly
```

### log/slog (structured logging, Go 1.21+)

```go
import "log/slog"

slog.Info("user logged in", "userID", 42, "ip", "1.2.3.4")
slog.Error("request failed", "err", err, "method", r.Method)
slog.Warn("deprecated endpoint", "path", r.URL.Path)

// JSON handler
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))
slog.SetDefault(logger)

// With attributes
logger = logger.With("service", "api", "version", "1.0")
```

---

## Go Toolchain

| Command | Description |
|---------|-------------|
| `go run main.go` | Compile and run |
| `go build` | Compile binary |
| `go build -o myapp` | Compile with output name |
| `go install` | Compile and install to `$GOPATH/bin` |
| `go test ./...` | Run all tests |
| `go test -v` | Verbose test output |
| `go test -run TestName` | Run matching tests |
| `go test -cover` | Show coverage percentage |
| `go test -coverprofile=c.out` | Generate coverage file |
| `go tool cover -html=c.out` | View coverage in browser |
| `go test -race` | Enable race detector |
| `go test -bench=.` | Run benchmarks |
| `go test -fuzz=FuzzName` | Run fuzz tests |
| `go vet ./...` | Static analysis |
| `go fmt ./...` | Format code |
| `gofmt -s -w .` | Format with simplification |
| `go generate ./...` | Run go:generate directives |
| `go doc fmt.Println` | Show documentation |
| `go env` | Print Go environment |
| `go work init` | Create a workspace (multi-module) |
| `go tool pprof` | CPU/memory profiling |

### Build Flags

```bash
# Cross-compilation
GOOS=linux GOARCH=amd64 go build -o myapp
GOOS=darwin GOARCH=arm64 go build -o myapp

# Embed version info
go build -ldflags="-X main.version=1.2.3 -s -w" -o myapp
# -s: strip symbol table, -w: strip DWARF (the debug-info format), reduces binary size

# Static binary (no cgo)
CGO_ENABLED=0 go build -o myapp
```

---

## Build Tags and Conditional Compilation

```go
//go:build linux
// +build linux     (old syntax, still works)

//go:build linux && amd64
//go:build !windows
//go:build integration

package mypackage
```

```bash
# Run tests with a custom build tag
go test -tags=integration ./...
```

### go:embed (Go 1.16+)

```go
import "embed"

//go:embed templates/*
var templateFS embed.FS

//go:embed version.txt
var version string

//go:embed logo.png
var logo []byte
```

### go:generate

```go
//go:generate stringer -type=Color
//go:generate mockgen -source=service.go -destination=mock_service.go
```

---

## CGo

CGo lets Go call C code and vice versa. The pseudo-package `"C"` provides access to C types and functions. The comment block immediately before `import "C"` is the C preamble -- it is compiled as C code.

### Calling C from Go

```go
package main

/*
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

// Inline C function
int add(int a, int b) {
    return a + b;
}
*/
import "C" // must be on its own line, no blank line after the comment
import (
    "fmt"
    "unsafe"
)

func main() {
    // Call C functions
    result := C.add(3, 4)
    fmt.Println(result) // 7

    sqrtVal := C.sqrt(16.0)
    fmt.Println(sqrtVal) // 4

    // C.int, C.long, C.double, etc. are distinct from Go types -- cast explicitly
    var x C.int = 42
    goX := int(x)
}
```

### C Types and Go Equivalents

| C Type | CGo Type | Go Equivalent |
|--------|----------|---------------|
| `int` | `C.int` | `int32` (usually) |
| `long` | `C.long` | `int64` on 64-bit |
| `char` | `C.char` | `byte` |
| `float` | `C.float` | `float32` |
| `double` | `C.double` | `float64` |
| `void *` | `unsafe.Pointer` | `unsafe.Pointer` |
| `size_t` | `C.size_t` | `uint` |
| `char *` | `*C.char` | use `C.CString` / `C.GoString` |

### String Conversion

```go
// Go string -> C string (allocates with malloc, must be freed)
cs := C.CString("hello")
defer C.free(unsafe.Pointer(cs))

// C string -> Go string (copies the data)
goStr := C.GoString(cs)

// C data with explicit length -> Go string
goStr := C.GoStringN(cData, cLen)

// Go []byte -> C data
cBytes := C.CBytes(goSlice) // malloc'd, must be freed
defer C.free(cBytes)

// C data -> Go []byte
goSlice := C.GoBytes(unsafe.Pointer(cData), C.int(length))
```

### Linking to External C Libraries

```go
/*
#cgo LDFLAGS: -lsqlite3
#cgo CFLAGS: -I/usr/local/include
#cgo pkg-config: libpng
#cgo linux LDFLAGS: -lrt
#cgo darwin LDFLAGS: -framework CoreFoundation
#cgo darwin,amd64 CFLAGS: -DDARWIN_AMD64   // constraints combine as OS[,arch]

#include <sqlite3.h>
*/
import "C"
```

### Calling Go from C (Export)

```go
//export GoAdd
func GoAdd(a, b C.int) C.int {
    return a + b
}

// The C side can now call GoAdd() after including _cgo_export.h
```

### Passing Structs

```go
/*
typedef struct {
    int x;
    int y;
} Point;

Point make_point(int x, int y) {
    Point p = {x, y};
    return p;
}
*/
import "C"

func main() {
    p := C.make_point(10, 20)
    fmt.Println(p.x, p.y)

    // Allocate a C struct in Go
    var p2 C.Point
    p2.x = 5
    p2.y = 15
}
```

### CGo Caveats

- **Performance**: each Go-to-C call has overhead (~100-200ns). Avoid fine-grained calls in hot loops.
- **Memory**: C memory (`C.malloc`, `C.CString`) is invisible to Go's GC (garbage collector). Always `C.free` manually.
- **Pointer passing**: Go pointers passed to C must not be retained by C after the call returns. Go enforces this at runtime (`cgocheck`).
- **Cross-compilation**: CGo requires a C compiler for the target platform. `CGO_ENABLED=0` disables CGo entirely for pure-Go static builds.
- **Build time**: CGo packages compile slower than pure Go.

---

## Quick Reference: Zero Values

| Type | Zero Value |
|------|------------|
| `int`, `float64`, etc. | `0` |
| `bool` | `false` |
| `string` | `""` |
| `pointer`, `func`, `interface`, `slice`, `map`, `channel` | `nil` |
| `struct` | all fields set to their zero values |
| `array` | all elements set to their zero values |
