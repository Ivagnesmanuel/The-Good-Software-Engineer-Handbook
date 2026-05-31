# Rust Cheatsheet

A quick-reference guide covering Rust's core concepts, syntax, and common patterns.

---

## Table of Contents

1. [Variables and Types](#variables-and-types)
2. [Ownership and Borrowing](#ownership-and-borrowing)
3. [Control Flow](#control-flow)
4. [Functions](#functions)
5. [Structs and Enums](#structs-and-enums)
6. [Pattern Matching](#pattern-matching) 
7. [Generics](#generics)
8. [Traits](#traits)
9. [Option and Result](#option-and-result)
10. [Error Handling](#error-handling)
11. [Collections](#collections)
12. [Iterators](#iterators)
13. [Closures](#closures)
14. [Strings](#strings)
15. [Lifetimes](#lifetimes)
16. [Smart Pointers](#smart-pointers)
17. [Concurrency](#concurrency)
18. [Async/Await](#asyncawait)
19. [Unsafe Rust](#unsafe-rust)
20. [Macros](#macros)
21. [Modules and Crates](#modules-and-crates)
22. [Cargo](#cargo)
23. [Useful Attributes](#useful-attributes)

---

## Variables and Types

```rust
// Immutable by default
let x = 5;

// Mutable
let mut y = 10;
y = 20;

// Type annotation
let z: i32 = 42;

// Constants (must have type annotation, evaluated at compile time)
const MAX_POINTS: u32 = 100_000;

// Static (fixed memory address for entire program lifetime)
static LANGUAGE: &str = "Rust";

// Shadowing (re-declare with let, can change type)
let s = "hello";
let s = s.len(); // s is now usize
```

### Scalar Types

| Type | Variants | Example |
|------|----------|---------|
| Integer | `i8`, `i16`, `i32`, `i64`, `i128`, `isize` | `let x: i64 = -42;` |
| Unsigned | `u8`, `u16`, `u32`, `u64`, `u128`, `usize` | `let x: u8 = 255;` |
| Float | `f32`, `f64` | `let x: f64 = 3.14;` |
| Boolean | `bool` | `let t: bool = true;` |
| Character | `char` | `let c: char = 'z';` |

### Compound Types

```rust
// Tuple (fixed length, mixed types)
let tup: (i32, f64, u8) = (500, 6.4, 1);
let (a, b, c) = tup;       // destructuring
let first = tup.0;          // index access

// Array (fixed length, same type)
let arr: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 10];        // [0, 0, 0, ...] (10 elements)
let first = arr[0];

// Slice (view into a contiguous sequence)
let slice: &[i32] = &arr[1..3]; // [2, 3]
```

### Type Aliases and Casting

```rust
type Kilometers = i32;

let x: i32 = 42;
let y: f64 = x as f64;
let c: u8 = 65;
let letter = c as char; // 'A'
```

---

## Ownership and Borrowing

Rust has no garbage collector. Memory is managed through a set of ownership rules checked at compile time.

### Ownership Rules

1. Each value has exactly one owner.
2. When the owner goes out of scope, the value is dropped.
3. Assignment moves ownership (for heap types). The original binding becomes invalid.

```rust
let s1 = String::from("hello");
let s2 = s1;          // s1 is MOVED into s2, s1 is no longer valid
let s3 = s2.clone();  // explicit deep copy, both s2 and s3 are valid

// Copy types live on the stack and are duplicated implicitly (no move)
// Includes: i32, f64, bool, char, tuples of Copy types
let a = 5;
let b = a; // copy, both valid
```

### Borrowing

Instead of transferring ownership, a reference to a value can be lent.

```rust
// Immutable borrow (&T) -- unlimited simultaneous borrows allowed
fn calculate_length(s: &String) -> usize {
    s.len()
}

let s = String::from("hello");
let len = calculate_length(&s); // s is still valid after the call

// Mutable borrow (&mut T) -- only ONE at a time, no concurrent immutable borrows
fn push_world(s: &mut String) {
    s.push_str(", world");
}

let mut s = String::from("hello");
push_world(&mut s);
```

### Borrowing Rules

1. You can have either **one** `&mut T` **or** any number of `&T` at the same time -- never both.
2. References must always be valid (no dangling pointers).

---

## Control Flow

```rust
// if / else (it's an expression, returns a value)
let num = if condition { 5 } else { 6 };

// if let (match a single pattern, ignore the rest)
if let Some(value) = optional {
    println!("{value}");
}

// let-else (match or diverge)
let Some(value) = optional else {
    return Err("missing value");
};

// loop (infinite, breakable with a value)
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;
    }
};

// Named loops
'outer: loop {
    loop {
        break 'outer; // breaks the outer loop
    }
};

// while
while n < 100 {
    n += 1;
}

// while let (loop until pattern stops matching)
while let Some(val) = stack.pop() {
    println!("{val}");
}

// for
for i in 0..5 { }           // 0, 1, 2, 3, 4
for i in 0..=5 { }          // 0, 1, 2, 3, 4, 5
for item in &collection { }     // borrow
for item in &mut collection { } // mutable borrow
for item in collection { }      // consume
```


---

## Functions

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b // no semicolon = implicit return
}

fn print_value(val: i32) {
    println!("Value: {val}");
}

// Early return
fn check(x: i32) -> &'static str {
    if x > 0 {
        return "positive";
    }
    "non-positive"
}

// Diverging function (never returns)
fn forever() -> ! {
    loop {}
}
```

---

## Structs and Enums

### Structs

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

let user = User {
    name: String::from("Alice"),
    age: 30,
    active: true,
};

// Shorthand (when variable name matches field name)
let name = String::from("Bob");
let user2 = User { name, age: 25, ..user }; // struct update syntax

// Tuple struct
struct Color(u8, u8, u8);
let red = Color(255, 0, 0);

// Unit struct (no fields, useful as marker types for traits)
struct Marker;
```

### Methods and Associated Functions

Methods are defined in `impl` blocks and are tied to a specific type.

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // Associated function (no self, called with ::)
    fn new(w: f64, h: f64) -> Self {
        Self { width: w, height: h }
    }

    // Method (borrows self immutably)
    fn area(&self) -> f64 {
        self.width * self.height
    }

    // Mutable method
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }

    // Consuming method (takes ownership of self)
    fn into_square(self) -> Self {
        let side = self.width.max(self.height);
        Self { width: side, height: side }
    }
}

let mut r = Rectangle::new(10.0, 5.0); // associated function
let a = r.area();                       // method call
r.scale(2.0);                           // mutable method
let sq = r.into_square();               // r is consumed, no longer valid
```

### Enums

Enum variants can hold data of different types and amounts.

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);

// Variants with named fields
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
}

// Enums can have impl blocks too
impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
        }
    }
}
```

---

## Pattern Matching

```rust
// match (must be exhaustive)
match value {
    0 => println!("zero"),
    1 | 2 => println!("one or two"),
    3..=9 => println!("three to nine"),
    n if n < 0 => println!("negative: {n}"), // match guard
    n => println!("other: {n}"),              // catch-all binding
}

// Destructuring structs
let Point { x, y } = point;

match point {
    Point { x: 0, y } => println!("on y-axis at {y}"),
    Point { x, y: 0 } => println!("on x-axis at {x}"),
    Point { x, y } => println!("({x}, {y})"),
}

// Destructuring enums
match shape {
    Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
    Shape::Rectangle { width, height } => width * height,
}

// @ bindings (bind a value while also testing it against a pattern)
match age {
    n @ 0..=12 => println!("child aged {n}"),
    n @ 13..=19 => println!("teen aged {n}"),
    n => println!("adult aged {n}"),
}

// matches! macro (returns bool)
let is_small = matches!(value, 0..=10);
```


---

## Generics

Generics allow code that works across multiple types without duplication. The compiler generates concrete implementations for each type used (monomorphization), so there is no runtime cost.

```rust
// Generic function
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut max = &list[0];
    for item in &list[1..] {
        if item > max {
            max = item;
        }
    }
    max
}

// Generic struct
struct Pair<T, U> {
    first: T,
    second: U,
}

impl<T, U> Pair<T, U> {
    fn new(first: T, second: U) -> Self {
        Self { first, second }
    }
}

// Conditional impl (methods available only when T satisfies bounds)
impl<T: Display + PartialOrd> Pair<T, T> {
    fn cmp_display(&self) {
        if self.first >= self.second {
            println!("first is larger: {}", self.first);
        }
    }
}

// Generic enum
enum Either<L, R> {
    Left(L),
    Right(R),
}
```

---

## Traits

Traits define shared behavior. They are similar to interfaces in other languages, but can include default implementations and work closely with generics.

```rust
// Define a trait
trait Summary {
    // Required method (implementors must provide)
    fn summarize(&self) -> String;

    // Default implementation (implementors can override)
    fn preview(&self) -> String {
        format!("Read more: {}", self.summarize())
    }
}

// Implement a trait for a type
impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{} by {}", self.title, self.author)
    }
}
```

### Trait Bounds

Trait bounds constrain generics so they only accept types that implement the required traits.

```rust
// impl Trait syntax (shorthand)
fn notify(item: &impl Summary) {
    println!("Breaking: {}", item.summarize());
}

// Equivalent generic syntax
fn notify<T: Summary>(item: &T) { }

// Multiple bounds
fn process<T: Summary + Display>(item: &T) { }

// where clause (cleaner when bounds get complex)
fn some_fn<T, U>(t: &T, u: &U) -> String
where
    T: Summary + Clone,
    U: Display + Debug,
{
    format!("{} {}", t.summarize(), u)
}

// Return a type that implements a trait (concrete type decided at compile time)
fn make_summary() -> impl Summary { }
```

### Trait Objects and `dyn`

When different concrete types are needed behind the same interface at runtime, use trait objects. The `dyn` keyword marks dynamic dispatch: method calls go through a vtable (a lookup table of function pointers) instead of being resolved at compile time.

```rust
// &dyn Trait or Box<dyn Trait>
fn log_all(items: &[&dyn Summary]) {
    for item in items {
        println!("{}", item.summarize());
    }
}

// Box<dyn Trait> for owned trait objects (e.g. heterogeneous collections)
let shapes: Vec<Box<dyn Shape>> = vec![
    Box::new(Circle { radius: 1.0 }),
    Box::new(Rect { w: 3.0, h: 4.0 }),
];

// Trade-offs:
// impl Trait / generics  -> static dispatch, monomorphized, zero-cost, one concrete type per call site
// dyn Trait               -> dynamic dispatch, vtable lookup, allows mixed concrete types at runtime
```

### Supertraits

```rust
// PrettyPrint can only be implemented by types that also implement Display + Debug
trait PrettyPrint: Display + Debug {
    fn pretty(&self) -> String;
}
```

### Common Derivable Traits

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default, PartialOrd, Ord)]
struct Point {
    x: i32,
    y: i32,
}
```

---

## Option and Result

`Option` and `Result` are enums at the core of Rust's approach to nullable values and error handling. They force the absent/error case to be handled explicitly.

```rust
// Option<T> = Some(T) | None
// Used when a value might be absent.
let found = vec![10, 20, 30].iter().position(|&x| x == 20); // Some(1)
let missing = vec![10, 20, 30].iter().position(|&x| x == 99); // None

// Result<T, E> = Ok(T) | Err(E)
// Used when an operation can fail.
let content: Result<String, io::Error> = fs::read_to_string("config.toml");
```

### Working with Option

```rust
// Unwrapping (use sparingly -- panics on None)
val.unwrap()
val.expect("reason for panic")

// Safe fallbacks
val.unwrap_or(default)
val.unwrap_or_default()          // uses T::default()
val.unwrap_or_else(|| compute()) // lazy fallback

// Transforming
val.map(|v| v + 1)              // Some(v) -> Some(v+1), None -> None
val.and_then(|v| other_fn(v))   // chain operations that themselves return Option
val.filter(|v| *v > 5)          // None if predicate fails

// Checking
val.is_some()
val.is_none()
```

### if let, while let, and let-else

```rust
// if let -- concise alternative to match when you only care about one variant
if let Some(value) = maybe_value {
    println!("got {value}");
}

// with else
if let Some(value) = maybe_value {
    println!("got {value}");
} else {
    println!("was None");
}

// works with Result too
if let Ok(content) = fs::read_to_string("config.toml") {
    println!("{content}");
}

// while let -- loop as long as the pattern matches
let mut stack = vec![1, 2, 3];
while let Some(top) = stack.pop() {
    println!("{top}");
}

// let-else (Rust 1.65+) -- bind or diverge (return, break, panic, etc.)
let Some(value) = maybe_value else {
    return Err("missing value".into());
};
// value is usable here, no nesting needed

let Ok(config) = fs::read_to_string("config.toml") else {
    panic!("failed to read config");
};
```

### Working with Result

```rust
// Same family of methods as Option, plus:
result.map_err(|e| CustomError::from(e)) // transform the error type
result.ok()    // Result<T,E> -> Option<T>, discards error
result.err()   // Result<T,E> -> Option<E>, discards success

// Matching
match fs::read_to_string("config.toml") {
    Ok(content) => println!("{content}"),
    Err(e) => eprintln!("Error: {e}"),
}

// Combinators
let upper = fs::read_to_string("file.txt")
    .map(|s| s.to_uppercase())
    .unwrap_or_else(|_| String::from("DEFAULT"));
```

---

## Error Handling

```rust
// Unrecoverable errors: panic! halts the current thread
panic!("something went wrong");

// Propagating errors with ?
// The ? operator returns early with Err if the expression is Err,
// otherwise unwraps the Ok value. Usable in functions returning Result (or Option).
fn first_line(path: &str) -> Result<String, io::Error> {
    let content = fs::read_to_string(path)?;
    let line = content
        .lines()
        .next()
        .ok_or(io::Error::new(io::ErrorKind::InvalidData, "empty file"))?;
    Ok(line.to_string())
}
```

### Custom Error Types

```rust
#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(std::num::ParseIntError),
    Custom(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO error: {e}"),
            AppError::Parse(e) => write!(f, "Parse error: {e}"),
            AppError::Custom(msg) => write!(f, "{msg}"),
        }
    }
}

// Implementing From enables automatic conversion via ?
impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}

// Now ? converts io::Error into AppError automatically
fn read_config(path: &str) -> Result<String, AppError> {
    let content = fs::read_to_string(path)?;
    Ok(content)
}
```

---

## Collections

### Vec\<T\>

```rust
let mut v: Vec<i32> = Vec::new();
let v = vec![1, 2, 3];

v.push(4);
v.pop();                // returns Option<T>
v.len();
v.is_empty();
v.contains(&3);
v.insert(1, 99);       // insert at index
v.remove(0);            // remove at index, shifts elements left
v.retain(|&x| x > 2);  // keep only elements matching the predicate
v.sort();
v.dedup();              // remove consecutive duplicates
v.extend([5, 6, 7]);

let slice = &v[1..3];
let first = v.get(0);  // Option<&T>, bounds-safe (no panic)
```

### HashMap\<K, V\>

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("key", 10);
map.get("key");          // Option<&V>
map.contains_key("key");
map.remove("key");

// Entry API (insert if absent, modify if present)
map.entry("key").or_insert(0);
*map.entry("key").or_insert(0) += 1;
map.entry("key").or_insert_with(|| expensive_computation());

for (key, value) in &map { }
```

### HashSet\<T\>

```rust
use std::collections::HashSet;

let mut set = HashSet::new();
set.insert("a");
set.contains("a");
set.remove("a");

// Set operations
let union = &set_a | &set_b;
let intersection = &set_a & &set_b;
let difference = &set_a - &set_b;
let symmetric_diff = &set_a ^ &set_b;
```

### VecDeque, BTreeMap, BTreeSet

```rust
use std::collections::VecDeque;

// VecDeque<T> -- double-ended queue, O(1) push/pop at both ends
let mut dq = VecDeque::new();
dq.push_back(1);
dq.push_front(0);
dq.pop_back();
dq.pop_front();

// BTreeMap<K, V> / BTreeSet<T> -- sorted by key, O(log n) operations
use std::collections::{BTreeMap, BTreeSet};
```

---

## Iterators

```rust
// Creating iterators from collections
let v = vec![1, 2, 3, 4, 5];
v.iter()       // yields &T
v.iter_mut()   // yields &mut T
v.into_iter()  // yields T, consumes the collection
```

### Adapters (lazy, must be consumed to execute)

```rust
iter
    .map(|x| x * 2)
    .filter(|x| *x > 4)
    .enumerate()               // yields (index, value)
    .skip(1)
    .take(3)
    .zip(other.iter())         // pairs elements from two iterators
    .chain(other.iter())       // concatenates two iterators
    .flatten()                 // flattens nested iterators
    .inspect(|x| println!("{x}")) // side-effect for debugging, passes value through
    .peekable()                // allows peeking at next element without consuming
    .rev()                     // reverses (requires DoubleEndedIterator)
    .cloned()                  // &T -> T (requires T: Clone)
    .copied()                  // &T -> T (requires T: Copy)
```

### Consumers (trigger evaluation)

```rust
    .collect::<Vec<_>>()
    .collect::<HashMap<_, _>>()
    .collect::<String>()
    .count()
    .sum::<i32>()
    .product::<i32>()
    .min()                     // Option<T>
    .max()                     // Option<T>
    .min_by_key(|x| x.len())
    .max_by_key(|x| x.len())
    .find(|x| **x > 3)        // Option<&T>
    .position(|x| *x > 3)     // Option<usize>
    .any(|x| *x > 3)          // bool
    .all(|x| *x > 0)          // bool
    .fold(0, |acc, x| acc + x)
    .reduce(|a, b| a + b)     // Option<T>
    .for_each(|x| println!("{x}"))
    .partition::<Vec<_>, _>(|x| *x > 3) // splits into (Vec, Vec)
```

### Implementing Iterator

```rust
struct Counter { count: u32, max: u32 }

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

---

## Closures

```rust
// Type-inferred
let add = |a, b| a + b;

// Explicit types
let add = |a: i32, b: i32| -> i32 { a + b };
```

### Capture Modes

The compiler infers the least restrictive capture mode automatically.

```rust
let name = String::from("Alice");
let greet = || println!("Hello, {name}");   // captures &name (Fn)

let mut count = 0;
let mut inc = || count += 1;                // captures &mut count (FnMut)

let name = String::from("Alice");
let consume = || drop(name);                // takes ownership (FnOnce)

// Force move (transfer ownership into closure regardless of usage)
let name = String::from("Alice");
let greet = move || println!("Hello, {name}");
// name is no longer valid here
```

### Closure Traits

The three closure traits form a hierarchy: every `Fn` is also `FnMut`, and every `FnMut` is also `FnOnce`.

```rust
fn apply_fn(f: impl Fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn apply_fn_mut(mut f: impl FnMut()) { f(); }
fn apply_fn_once(f: impl FnOnce() -> String) -> String { f() }

// Returning closures
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y
}
```

---

## Strings

```rust
// &str -- immutable string slice (string literals, borrowed views)
let s: &str = "hello";

// String -- heap-allocated, growable, owned
let mut s = String::new();
let s = String::from("hello");
let s = "hello".to_string();

// Conversions
let slice: &str = &s;
let owned: String = slice.to_owned();
```

### Manipulation

```rust
s.push('!');
s.push_str(" world");
s.insert(0, 'H');
s.insert_str(5, ", dear");

let combined = format!("{} {}", first, second); // does not take ownership

s.replace("old", "new");       // returns new String
s.replacen("old", "new", 1);   // replace first N occurrences
s.trim();
s.trim_start();
s.trim_end();
s.to_uppercase();
s.to_lowercase();
```

### Querying

```rust
s.starts_with("he");
s.ends_with("lo");
s.contains("ell");
s.is_empty();
s.len();  // byte length, NOT character count
```

### Iteration

Strings are UTF-8 encoded, so direct indexing (`s[0]`) is not allowed.

```rust
for c in s.chars() { }           // iterate by char
for b in s.bytes() { }           // iterate by byte
for (i, c) in s.char_indices() { } // char with byte offset

s.split(',');
s.split_whitespace();
s.lines();

// Substring via byte range (panics if not on a char boundary)
let sub = &s[0..5];
```

---

## Lifetimes

Lifetimes are the compiler's way of tracking how long references are valid. Most of the time the compiler infers them automatically (lifetime elision). You only need explicit annotations when the compiler can't determine the relationship between input and output references.

```rust
// This tells the compiler: the returned reference lives at least as long
// as the shorter of x and y.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Struct holding a reference must annotate the lifetime
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    // Elision applies: &self lifetime is assigned to the return reference
    fn content(&self) -> &str {
        self.text
    }
}

// 'static -- the reference is valid for the entire program duration
let s: &'static str = "I live forever";
```

### Lifetime Elision Rules

The compiler applies these rules automatically. If they fully determine output lifetimes, no annotation is needed.

1. Each reference parameter gets its own lifetime.
2. If there is exactly one input lifetime, it is assigned to all output references.
3. If `&self` or `&mut self` is a parameter, its lifetime is assigned to all output references.

---

## Smart Pointers

### Box\<T\>

Heap-allocates a value. Single owner, no runtime overhead beyond the allocation.

```rust
let b = Box::new(5);

// Required for recursive types (size must be known at compile time)
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

### Rc\<T\> and Arc\<T\>

Reference-counted pointers for multiple ownership.

```rust
// Rc<T> -- single-threaded only
use std::rc::Rc;

let a = Rc::new(String::from("shared"));
let b = Rc::clone(&a); // increments reference count (cheap, no deep copy)
println!("refs: {}", Rc::strong_count(&a)); // 2

// Arc<T> -- thread-safe version of Rc (uses atomic operations)
use std::sync::Arc;
let shared = Arc::new(vec![1, 2, 3]);
```

### RefCell\<T\>

Interior mutability: allows mutation through an immutable reference, with borrow rules checked at runtime instead of compile time.

```rust
use std::cell::RefCell;

let data = RefCell::new(5);
*data.borrow_mut() += 1;     // mutable borrow at runtime
println!("{}", data.borrow()); // immutable borrow at runtime

// Common combo: Rc<RefCell<T>> for shared mutable state (single-threaded)
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
```

### Cow\<T\>

Clone-on-write: borrows data when possible, clones only when mutation is needed.

```rust
use std::borrow::Cow;

fn sanitize(input: &str) -> Cow<str> {
    if input.contains("bad") {
        Cow::Owned(input.replace("bad", "good")) // allocation happens
    } else {
        Cow::Borrowed(input) // zero-cost, just a reference
    }
}
```

---

## Concurrency

### Threads

```rust
use std::thread;

let handle = thread::spawn(|| {
    println!("from spawned thread");
});
handle.join().unwrap(); // block until the thread finishes

// Move data into a thread
let data = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("{data:?}");
});

// Scoped threads (borrow from parent scope without move)
thread::scope(|s| {
    s.spawn(|| {
        println!("can borrow from parent: {data:?}");
    });
}); // all scoped threads are joined here automatically
```

### Channels (message passing)

```rust
use std::sync::mpsc; // multiple producer, single consumer

let (tx, rx) = mpsc::channel();
let tx2 = tx.clone(); // multiple producers

thread::spawn(move || {
    tx.send("hello").unwrap();
});

let msg = rx.recv().unwrap();  // blocking
let msg = rx.try_recv();       // non-blocking, returns Result
for received in rx { }         // iterate until all senders are dropped
```

### Shared State

```rust
// Mutex<T> -- mutual exclusion, one thread accesses data at a time
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

// RwLock<T> -- multiple concurrent readers OR one exclusive writer
use std::sync::RwLock;

let lock = RwLock::new(5);
{ let r = lock.read().unwrap(); }            // shared read lock
{ let mut w = lock.write().unwrap(); *w += 1; } // exclusive write lock
```

---

## Async/Await

Rust's async model is based on futures (lazy, poll-based). The language provides `async`/`await` syntax, but an executor (runtime) is required to drive futures to completion. Tokio is the most widely used runtime.

### Basics

```rust
// async fn returns an impl Future<Output = T>
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}

// Equivalent desugared form
fn fetch_data(url: &str) -> impl Future<Output = Result<String, reqwest::Error>> + '_ {
    async move {
        let body = reqwest::get(url).await?.text().await?;
        Ok(body)
    }
}

// Async blocks (create a future inline)
let fut = async {
    let result = some_async_fn().await;
    result + 1
};
```

### Tokio Runtime

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
// Main entry point
#[tokio::main]
async fn main() {
    let result = fetch_data("https://example.com").await.unwrap();
    println!("{result}");
}

// Equivalent manual setup
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // async code here
    });
}

// Custom runtime configuration
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .enable_all()
    .build()
    .unwrap();
```

### Spawning Tasks

```rust
// Spawn a concurrent task (like a lightweight thread)
let handle = tokio::spawn(async {
    expensive_async_work().await
});
let result = handle.await.unwrap(); // JoinHandle<T>

// Spawn blocking work on a dedicated thread pool
// (for CPU-heavy or synchronous I/O that would block the executor)
let result = tokio::task::spawn_blocking(|| {
    std::fs::read_to_string("large_file.txt")
}).await.unwrap();
```

### Concurrency Patterns

```rust
// Run futures concurrently, wait for all to complete
let (a, b, c) = tokio::join!(
    fetch("url1"),
    fetch("url2"),
    fetch("url3"),
);

// Race futures, return the first to complete
tokio::select! {
    val = future_a => println!("a finished first: {val:?}"),
    val = future_b => println!("b finished first: {val:?}"),
}

// Timeout
use tokio::time::{timeout, Duration};
match timeout(Duration::from_secs(5), slow_operation()).await {
    Ok(result) => println!("completed: {result:?}"),
    Err(_) => println!("timed out"),
}

// Sleep (non-blocking)
tokio::time::sleep(Duration::from_millis(100)).await;
```

### Channels (async)

```rust
use tokio::sync::{mpsc, oneshot, broadcast, watch};

// mpsc -- multiple producer, single consumer (bounded)
let (tx, mut rx) = mpsc::channel(32); // buffer size
tx.send("hello").await.unwrap();
let msg = rx.recv().await; // Option<T>, None when all senders dropped

// oneshot -- single value, single use
let (tx, rx) = oneshot::channel();
tx.send("done").unwrap(); // no .await, completes immediately
let msg = rx.await.unwrap();

// broadcast -- multiple producers, multiple consumers (all receivers get every message)
let (tx, _) = broadcast::channel(16);
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// watch -- single producer, multiple consumers (receivers see latest value)
let (tx, mut rx) = watch::channel("initial");
tx.send("updated").unwrap();
rx.changed().await.unwrap();
println!("{}", *rx.borrow());
```

### Async Traits

```rust
// Since Rust 1.75, async fn in traits works natively
trait HttpClient {
    async fn get(&self, url: &str) -> Result<String, Error>;
}

// For older Rust or when you need dyn dispatch, use the async-trait crate
use async_trait::async_trait;

#[async_trait]
trait HttpClient: Send + Sync {
    async fn get(&self, url: &str) -> Result<String, Error>;
}
```

### Stream (async iterator)

```rust
use tokio_stream::{self as stream, StreamExt};

let mut s = stream::iter(vec![1, 2, 3]);
while let Some(value) = s.next().await {
    println!("{value}");
}

// Common adapters: map, filter, take, merge, throttle, timeout
```

---

## Unsafe Rust

Unsafe Rust allows opting out of certain compiler guarantees. An `unsafe` block asserts to the compiler that the relevant invariants have been manually verified. The five operations permitted only in unsafe code:

1. Dereference raw pointers
2. Call unsafe functions
3. Access or modify mutable statics
4. Implement unsafe traits
5. Access fields of unions

### Raw Pointers

```rust
let mut x = 42;

// Creating raw pointers is safe
let r1 = &x as *const i32;     // immutable raw pointer
let r2 = &mut x as *mut i32;   // mutable raw pointer

// Dereferencing requires unsafe
unsafe {
    println!("r1 = {}", *r1);
    *r2 = 100;
}

// Pointers from arbitrary addresses (no guarantee of validity)
let addr = 0x012345usize;
let _p = addr as *const i32;

// Null pointers
let p: *const i32 = std::ptr::null();
let p_mut: *mut i32 = std::ptr::null_mut();
assert!(p.is_null());
```

### Unsafe Functions and Blocks

```rust
// Declaring an unsafe function (caller must uphold invariants)
unsafe fn dangerous(ptr: *const i32) -> i32 {
    *ptr // caller guarantees ptr is valid and aligned
}

unsafe {
    let val = dangerous(&42 as *const i32);
}

// Safe wrapper around unsafe code (common pattern)
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let ptr = values.as_mut_ptr();
    assert!(mid <= len);

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

### Mutable Statics

```rust
// Accessing/modifying mutable statics is unsafe because of potential data races
static mut COUNTER: u32 = 0;

unsafe {
    COUNTER += 1;
    println!("COUNTER: {COUNTER}");
}
```

### Unsafe Traits

```rust
// An unsafe trait has invariants the compiler can't verify
// Implementor promises to uphold them
unsafe trait RawSendable {
    fn raw_ptr(&self) -> *const u8;
}

unsafe impl RawSendable for MyType {
    fn raw_ptr(&self) -> *const u8 {
        &self.data as *const u8
    }
}

// Send and Sync are the most common unsafe traits
// The compiler auto-implements them when safe; manual impl requires unsafe
unsafe impl Send for MyWrapper {}
unsafe impl Sync for MyWrapper {}
```

### FFI (Foreign Function Interface)

```rust
// Calling C functions from Rust
extern "C" {
    fn abs(input: i32) -> i32;
    fn strlen(s: *const std::ffi::c_char) -> usize;
}

unsafe {
    println!("abs(-3) = {}", abs(-3));
}

// Exposing Rust functions to C
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

// C-compatible types
use std::ffi::{CStr, CString, c_char, c_int, c_void};

// Rust string -> C string
let c_string = CString::new("hello").unwrap();
let ptr: *const c_char = c_string.as_ptr();

// C string -> Rust string
unsafe {
    let c_str = CStr::from_ptr(ptr);
    let rust_str: &str = c_str.to_str().unwrap();
}

// Opaque types (representing C structs you don't need to inspect)
#[repr(C)]
struct CBuffer {
    data: *mut u8,
    len: usize,
}
```

### #[repr(...)] -- Controlling Memory Layout

```rust
#[repr(C)]          // C-compatible layout (required for FFI structs)
struct Point { x: f64, y: f64 }

#[repr(transparent)] // same layout as the single field (newtype FFI safety)
struct Wrapper(i32);

#[repr(u8)]          // set the discriminant type for enums
enum Color { Red = 0, Green = 1, Blue = 2 }
```

---

## Macros

### Common Standard Macros

```rust
println!("formatted: {} and {:?}", val, debug_val);
format!("returns a {}", "String");
eprintln!("prints to stderr");
dbg!(&value);              // prints file:line = value, returns the value
todo!("not implemented");  // compiles but panics at runtime
unimplemented!();
unreachable!();
assert!(condition);
assert_eq!(a, b);
assert_ne!(a, b);
vec![1, 2, 3];
include_str!("file.txt"); // embed file contents as &str at compile time
include_bytes!("img.png");
cfg!(target_os = "linux"); // bool, evaluated at compile time
env!("CARGO_PKG_VERSION"); // env var at compile time, panics if missing
option_env!("MY_VAR");     // Option<&str>
```

### Format Specifiers

| Specifier | Meaning | Output |
|-----------|---------|--------|
| `{}` | Display | `42` |
| `{:?}` | Debug | `[1, 2]` |
| `{:#?}` | Pretty Debug | multi-line |
| `{:04}` | Zero-padded | `0042` |
| `{:.2}` | 2 decimal places | `3.14` |
| `{:>10}` | Right-align, width 10 | `"        hi"` |
| `{:<10}` | Left-align | `"hi        "` |
| `{:^10}` | Center | `"    hi    "` |
| `{:b}` | Binary | `101010` |
| `{:x}` / `{:X}` | Hex lower / upper | `2a` / `2A` |
| `{:e}` | Scientific | `4.2e1` |
| `{val}` | Named binding | `println!("{val}")` |

### Declarative Macros

```rust
macro_rules! my_vec {
    () => { Vec::new() };
    ( $( $x:expr ),+ $(,)? ) => {
        {
            let mut v = Vec::new();
            $( v.push($x); )+
            v
        }
    };
}

let v = my_vec![1, 2, 3];
```

---

## Modules and Crates

### Inline Modules

```rust
mod math {
    pub fn add(a: i32, b: i32) -> i32 { a + b }

    pub mod advanced {
        pub fn factorial(n: u64) -> u64 {
            (1..=n).product()
        }
    }
}
```

### File-based Modules

Each module is either a single file (`routes.rs`) or a directory with a same-named parent file (`routes.rs` + `routes/`). This is the modern layout preferred since Rust 2018; the older `mod.rs` convention still compiles but is discouraged.

```
my_project/
  src/
    main.rs
    lib.rs
    config.rs              -> declared as `mod config;` in lib.rs
    routes.rs              -> declared as `mod routes;` in lib.rs
    routes/
      auth.rs              -> declared as `pub mod auth;` in routes.rs
      api.rs               -> declared as `pub mod api;`  in routes.rs
  tests/
    integration.rs
  benches/
    benchmark.rs
  examples/
    demo.rs
```

### Visibility

```rust
pub fn public() { }            // accessible from outside the module
pub(crate) fn crate_only() { } // accessible anywhere within this crate
pub(super) fn parent_only() { } // accessible to the parent module
fn private() { }                // default, only within the current module
```

### Imports and Re-exports

```rust
use std::collections::HashMap;
use std::io::{self, Read, Write};        // multiple items from one path
use crate::math::advanced::factorial;    // absolute path from crate root
pub use crate::math::add;               // re-export: consumers can use your_crate::add
```

---

## Cargo

### Commands

| Command | Description |
|---------|-------------|
| `cargo new project_name` | Create a new binary project |
| `cargo new --lib lib_name` | Create a new library project |
| `cargo init` | Initialize in an existing directory |
| `cargo build` | Compile (debug) |
| `cargo build --release` | Compile (optimized) |
| `cargo run` | Build and run |
| `cargo run -- args` | Run with arguments |
| `cargo test` | Run all tests |
| `cargo test test_name` | Run matching tests |
| `cargo test -- --nocapture` | Show `println!` output in tests |
| `cargo bench` | Run benchmarks |
| `cargo check` | Type-check without full compilation (fast feedback) |
| `cargo clippy` | Linter |
| `cargo fmt` | Format code |
| `cargo doc --open` | Generate and open documentation |
| `cargo add serde` | Add a dependency |
| `cargo remove serde` | Remove a dependency |
| `cargo update` | Update dependencies to latest compatible versions |
| `cargo tree` | Show dependency tree |
| `cargo fix` | Auto-fix compiler warnings |
| `cargo publish` | Publish crate to crates.io |

### Cargo.toml

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"          # Rust edition (2015, 2018, 2021, 2024)
authors = ["You <you@example.com>"]
description = "A short description"
license = "MIT"

[dependencies]
serde = "1.0"                                      # latest compatible with 1.x
serde_json = { version = "1.0", optional = true }   # feature-gated dependency
tokio = { version = "1", features = ["full"] }       # with specific features
my_lib = { path = "../my_lib" }                      # local path dependency
my_lib = { git = "https://github.com/user/repo" }   # git dependency

[dev-dependencies]
# Only compiled for tests, examples, and benchmarks
criterion = "0.5"

[build-dependencies]
# Only used by build.rs
cc = "1.0"

[features]
default = ["json"]        # features enabled by default
json = ["serde_json"]     # enabling "json" pulls in the optional serde_json dep
full = ["json"]           # composite feature

[[bin]]
name = "my_binary"
path = "src/main.rs"

[[example]]
name = "demo"
path = "examples/demo.rs"
```

### Workspaces

Workspaces manage multiple related crates in a single repository. They share a single `Cargo.lock` and output directory (`target/`), which speeds up builds and keeps dependency versions consistent.

```toml
# Root Cargo.toml
[workspace]
members = [
    "crates/core",
    "crates/cli",
    "crates/api",
]
resolver = "2"

# Shared dependency versions across workspace members
[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

Member crates reference workspace dependencies with `workspace = true`:

```toml
# crates/cli/Cargo.toml
[package]
name = "my_cli"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { workspace = true }       # inherits version and features from root
my_core = { path = "../core" }     # depend on a sibling crate
```

```bash
cargo build                         # builds all workspace members
cargo test -p my_cli                # test a specific member
cargo run -p my_cli                 # run a specific binary
```

---

## Useful Attributes

```rust
#[allow(dead_code)]              // suppress specific warnings
#[allow(unused_variables)]

#[cfg(target_os = "linux")]      // conditional compilation
#[cfg(test)]                     // compile only in test builds
#[cfg(feature = "serde")]        // compile only when feature is enabled

#[inline]                        // hint to inline the function
#[must_use]                      // warn if return value is ignored
#[deprecated(since = "2.0", note = "use new_fn instead")]
#[non_exhaustive]                // prevent exhaustive matching by external code
```

### Testing Attributes

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(add(2, 2), 4);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_panic() {
        risky_function();
    }

    #[test]
    fn test_result() -> Result<(), String> {
        if add(2, 2) == 4 { Ok(()) } else { Err("math is broken".into()) }
    }

    #[ignore] // skipped by default, run with: cargo test -- --ignored
    #[test]
    fn expensive_test() { }
}
```

---

## Quick Reference: Commonly Used Traits

| Trait | Purpose | Method(s) |
|-------|---------|-----------|
| `Display` | User-facing formatting | `fmt(&self, f)` |
| `Debug` | Developer formatting | derivable |
| `Clone` | Explicit deep copy | `clone(&self)` |
| `Copy` | Implicit bitwise copy | marker trait |
| `Default` | Default value | `default()` |
| `PartialEq` / `Eq` | Equality comparison | `eq(&self, &other)` |
| `PartialOrd` / `Ord` | Ordering | `partial_cmp`, `cmp` |
| `Hash` | Hashing | `hash(&self, state)` |
| `From` / `Into` | Type conversion | `from(val)` / `into(self)` |
| `TryFrom` / `TryInto` | Fallible conversion | returns `Result` |
| `AsRef` / `AsMut` | Cheap reference conversion | `as_ref(&self)` |
| `Deref` / `DerefMut` | Smart pointer dereference | `deref(&self)` |
| `Drop` | Custom destructor | `drop(&mut self)` |
| `Iterator` | Iteration | `next(&mut self)` |
| `IntoIterator` | Convert to iterator | `into_iter(self)` |
| `FromIterator` | Build from iterator | `from_iter(iter)` |
| `Send` | Safe to transfer across threads | auto trait |
| `Sync` | Safe to share references across threads | auto trait |
