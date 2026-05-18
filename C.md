# C Cheatsheet

A quick-reference guide covering C's core concepts, syntax, and common patterns.

---

## Table of Contents

1. [Variables and Types](#variables-and-types)
2. [Operators](#operators)
3. [Control Flow](#control-flow)
4. [Functions](#functions)
5. [Pointers](#pointers)
6. [Arrays](#arrays)
7. [Strings](#strings)
8. [Structs and Unions](#structs-and-unions)
9. [Enums and Typedefs](#enums-and-typedefs)
10. [Dynamic Memory](#dynamic-memory)
11. [Preprocessor](#preprocessor)
12. [Header Files and Compilation Units](#header-files-and-compilation-units)
13. [File I/O](#file-io)
14. [Formatted I/O](#formatted-io)
15. [Bitwise Operations](#bitwise-operations)
16. [Function Pointers](#function-pointers)
17. [Variadic Functions](#variadic-functions)
18. [Error Handling Patterns](#error-handling-patterns)
19. [Concurrency (pthreads)](#concurrency-pthreads)
20. [Common Standard Library Functions](#common-standard-library-functions)
21. [Build Tools and Compilation](#build-tools-and-compilation)

---

## Variables and Types

```c
// Declaration and initialization
int x = 5;
int y;          // indeterminate value (local), 0 (static/global)
int a = 0, b = 1, c = 2;

// Constants
const int MAX_SIZE = 100; // runtime constant (not a compile-time constant in C)
#define MAX_POINTS 100000 // preprocessor constant (compile-time, no type safety)

// Storage classes
static int count; // persists across calls (local) or limits to file scope (global)
extern int shared; // declaration only, defined elsewhere

// Type qualifiers
const int x = 10;       // read-only
volatile int flag;      // may change outside program control (hardware, signals)
restrict int *p;        // pointer is the only way to access the pointed-to memory (C99)
_Atomic int counter;    // atomic access (C11)
```

### Integer Types

| Type | Minimum Range | Typical Size |
|------|--------------|--------------|
| `char` | -128 to 127 (or 0 to 255) | 1 byte |
| `signed char` | -128 to 127 | 1 byte |
| `unsigned char` | 0 to 255 | 1 byte |
| `short` | -32768 to 32767 | 2 bytes |
| `int` | -32768 to 32767 | 4 bytes (usually) |
| `long` | -2^31 to 2^31-1 | 4 or 8 bytes |
| `long long` | -2^63 to 2^63-1 | 8 bytes |

All integer types have `unsigned` variants doubling the positive range.

### Fixed-Width Integers (C99)

```c
#include <stdint.h>

int8_t   i8;    
int16_t  i16;   
int32_t  i32;   
int64_t  i64;   
uint8_t  u8;    // unsigned variants
uint16_t u16;
uint32_t u32;
uint64_t u64;

size_t   sz;    // unsigned, result of sizeof, array indexing
ssize_t  ssz;   // signed size (POSIX, not standard C)
ptrdiff_t pd;   // signed difference between pointers
intptr_t ip;    // integer wide enough to hold a pointer
uintptr_t up;

// Limits
INT32_MIN, INT32_MAX, UINT64_MAX
SIZE_MAX
```

### Floating-Point Types

| Type | Precision | Typical Size |
|------|-----------|--------------|
| `float` | ~7 decimal digits | 4 bytes |
| `double` | ~15 decimal digits | 8 bytes |
| `long double` | ~18-33 decimal digits | 8, 12, or 16 bytes |

### Boolean (C99)

```c
#include <stdbool.h>

bool flag = true;
bool done = false;
// Under the hood: _Bool type, true = 1, false = 0
```

### Type Casting

```c
int x = 42;
double y = (double)x;
int z = (int)3.99;    // truncates to 3
void *p = &x;
int *ip = (int *)p;   // cast void pointer to typed pointer
```

### sizeof

```c
sizeof(int)          // size of type in bytes
sizeof(x)            // size of variable
sizeof(arr) / sizeof(arr[0])  // number of elements in a stack-allocated array
// WARNING: sizeof on a pointer gives pointer size, not array size
```

---

## Operators

### Arithmetic

```c
+  -  *  /  %    // add, sub, mul, div, modulo
++x  --x         // prefix increment/decrement (returns new value)
x++  x--         // postfix increment/decrement (returns old value)
```

### Comparison and Logical

```c
==  !=  <  >  <=  >=   // comparison (return 0 or 1)
&&  ||  !              // logical AND, OR, NOT (short-circuit)
```

### Assignment

```c
=  +=  -=  *=  /=  %=
&=  |=  ^=  <<=  >>=
```

### Ternary

```c
int max = (a > b) ? a : b;
```

### Comma Operator

```c
// Evaluates left to right, result is the rightmost expression
int x = (a = 1, b = 2, a + b); // x = 3
```

---

## Control Flow

```c
// if / else
if (x > 0) {
    printf("positive\n");
} else if (x == 0) {
    printf("zero\n");
} else {
    printf("negative\n");
}

// for
for (int i = 0; i < 10; i++) { }

// infinite loop
for (;;) { }

// while
while (condition) { }

// do-while (body executes at least once)
do {
    // ...
} while (condition);

// switch (falls through by default -- use break)
switch (value) {
case 1:
    printf("one\n");
    break;
case 2:
case 3:
    printf("two or three\n");
    break;
default:
    printf("other\n");
    break;
}

// Intentional fallthrough (document it)
switch (level) {
case 3:
    setup_level3();
    /* fallthrough */
case 2:
    setup_level2();
    /* fallthrough */
case 1:
    setup_level1();
    break;
}

// goto (use sparingly, mainly for cleanup in error paths)
if (error) {
    goto cleanup;
}
// ...
cleanup:
    free(buffer);
    return -1;

// break and continue
for (int i = 0; i < 100; i++) {
    if (i % 2 == 0) continue; // skip even numbers
    if (i > 50) break;        // exit loop
}
```

---

## Functions

```c
// Declaration (prototype) and definition
int add(int a, int b);           // prototype (usually in header)

int add(int a, int b) {          // definition
    return a + b;
}

// void return type
void greet(const char *name) {
    printf("Hello, %s\n", name);
    return
}

// Static functions (file-scoped, not visible to other translation units)
static int helper(int x) {
    return x * 2;
}

// Inline functions (hint to inline at call site, C99)
static inline int square(int x) {
    return x * x;
}

// _Noreturn (C11) -- function never returns (e.g., exit, abort)
_Noreturn void fatal(const char *msg) {
    fprintf(stderr, "%s\n", msg);
    exit(1);
}

// Passing arrays (decays to pointer, size is lost)
void process(int arr[], size_t len);     // arr is int*
void process(int *arr, size_t len);      // equivalent
void matrix(int rows, int cols, int m[rows][cols]); // VLA parameter (C99)

// Returning pointers (never return a pointer to a local variable)
int* allocate(size_t n) {
    int *p = malloc(n * sizeof(int));
    return p; // caller must free
}
```

---

## Pointers

```c
int x = 42;
int *p = &x;       // p stores the address of x
int val = *p;      // dereference: val = 42
*p = 100;          // x is now 100

// NULL pointer
int *p = NULL;      // always initialize unused pointers to NULL
if (p != NULL) { }
if (p) { }          // idiomatic null check

// Pointer arithmetic
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;       // arrays decay to pointers
p++;                // now points to arr[1]
p += 2;             // now points to arr[3]
int diff = p - arr; // 3 (element count, not byte count)
```

### Pointer to Pointer

```c
int x = 5;
int *p = &x;
int **pp = &p;     // pointer to pointer
**pp = 10;         // x is now 10

// Common use: function that modifies a pointer
void alloc(int **out, size_t n) {
    *out = malloc(n * sizeof(int));
}
int *data;
alloc(&data, 100);
```

### void Pointer (generic pointer)

```c
void *p;            // can hold any pointer type, no arithmetic
int x = 42;
p = &x;
int *ip = (int *)p; // must cast before dereferencing

// Used by malloc, qsort, memcpy, etc.
```

### const and Pointers

```c
const int *p;       // pointer to const int (can't modify *p)
int const *p;       // same as above
int *const p = &x;  // const pointer to int (can't modify p itself)
const int *const p = &x; // const pointer to const int (can't modify either)

// Rule of thumb: read right-to-left
// int *const p -> "p is a const pointer to int"
// const int *p -> "p is a pointer to const int"
```

---

## Arrays

```c
// Stack-allocated (fixed size, known at compile time)
int arr[5] = {1, 2, 3, 4, 5};
int zeros[100] = {0};         // all zeros
int partial[5] = {1, 2};      // rest are 0
int inferred[] = {1, 2, 3};   // size = 3, inferred from initializer

// Designated initializers (C99)
int arr[10] = {[0] = 1, [5] = 50, [9] = 99};

// Multidimensional arrays
int matrix[3][4] = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12},
};
int val = matrix[1][2]; // 7

// Array length (only for stack arrays, NOT pointers)
size_t len = sizeof(arr) / sizeof(arr[0]);

// Variable-length arrays (C99, optional in C11)
void func(size_t n) {
    int vla[n]; // allocated on the stack, size determined at runtime
    // WARNING: no bounds checking, stack overflow risk for large n
}

// Arrays decay to pointers when passed to functions
void print_array(int *arr, size_t len); // arr is a pointer, sizeof(arr) == pointer size
```

### Common Array Operations

```c
#include <string.h>

// Copy & move
memcpy(dst, src, n * sizeof(int));     // non-overlapping
memmove(dst, src, n * sizeof(int));    // safe for overlapping regions

// Fill
memset(arr, 0, sizeof(arr));           // zero-fill (works for 0 and -1)

// Compare
int equal = (memcmp(a, b, n * sizeof(int)) == 0);

// Sort
qsort(arr, len, sizeof(int), compare_ints);

// Binary search (array must be sorted)
int *found = bsearch(&key, arr, len, sizeof(int), compare_ints);
```

---

## Strings

Strings in C are null-terminated `char` arrays. The terminator `'\0'` marks the end.

```c
// String literals (stored in read-only memory)
const char *s = "hello";          // pointer to string literal
char s[] = "hello";               // mutable copy: {'h','e','l','l','o','\0'}
char s[10] = "hello";             // padded with '\0': 10 bytes total

// Character literals
char c = 'A';                     // integer value 65
char newline = '\n';
char null = '\0';                 // 0
```

### string.h Functions

```c
#include <string.h>

strlen(s)                          // length (excludes '\0')
strcmp(a, b)                       // 0 if equal, <0 if a<b, >0 if a>b
strncmp(a, b, n)                   // compare at most n characters

strcpy(dst, src)                   // copy (dst must be large enough, UNSAFE)
strncpy(dst, src, n)               // copy at most n chars (may not null-terminate!)

strcat(dst, src)                   // append src to dst (UNSAFE)
strncat(dst, src, n)               // append at most n chars, always null-terminates

strchr(s, c)                       // find first occurrence of c, returns pointer or NULL
strrchr(s, c)                      // find last occurrence of c
strstr(haystack, needle)           // find first occurrence of substring

// Tokenizing (modifies the original string!)
char str[] = "one,two,three";
char *token = strtok(str, ",");
while (token) {
    printf("%s\n", token);
    token = strtok(NULL, ",");
}

// Safer alternatives (POSIX / C11 Annex K)
strtok_r(str, delim, &saveptr)     // thread-safe strtok (POSIX)
snprintf(dst, size, fmt, ...)      // bounded formatted write
```

### String Conversions

```c
#include <stdlib.h>

int i = atoi("42");                // ASCII string to int (no error checking)
long l = atol("123456");
double d = atof("3.14");

// Safer alternatives
long l = strtol("42", &endptr, 10);   // base 10, endptr points past last parsed char
double d = strtod("3.14", &endptr);
unsigned long ul = strtoul("255", NULL, 16); // parse hex

// Number to string
char buf[32];
snprintf(buf, sizeof(buf), "%d", 42);
snprintf(buf, sizeof(buf), "%.2f", 3.14159);
```

---

## Structs and Unions

### Structs

```c
struct Point {
    int x;
    int y;
};

// Declaration and initialization
struct Point p1 = {10, 20};               // positional
struct Point p2 = {.x = 10, .y = 20};     // designated (C99)
struct Point p3 = {0};                    // zero-initialize all fields

// Access
p1.x = 30;

// Pointer to struct
struct Point *pp = &p1;
pp->x = 40;         // equivalent to (*pp).x

// Nested structs
struct Rectangle {
    struct Point origin;
    struct Point size;
};

struct Rectangle r = {.origin = {0, 0}, .size = {100, 50}};
int width = r.size.x;
```

### Struct Padding and Alignment

```c
// Compilers insert padding for alignment
struct Padded {
    char a;      // 1 byte + 3 bytes padding
    int b;       // 4 bytes
    char c;      // 1 byte + 3 bytes padding
};               // total: 12 bytes

// Reordering to minimize padding
struct Compact {
    int b;       // 4 bytes
    char a;      // 1 byte
    char c;      // 1 byte + 2 bytes padding
};               // total: 8 bytes

// Packed struct (compiler-specific, removes padding)
struct __attribute__((packed)) Wire {
    char a;
    int b;
    char c;
};               // total: 6 bytes (unaligned access, may be slower)

// Check layout
offsetof(struct Padded, b)  // #include <stddef.h>, byte offset of field b
sizeof(struct Padded)
```

### Unions

All members share the same memory. Only one member is valid at a time.

```c
union Value {
    int i;
    float f;
    char s[16];
};

union Value v;
v.i = 42;
printf("%d\n", v.i);

v.f = 3.14;        // now only v.f is valid, reading v.i is undefined

sizeof(union Value) // size of the largest member (16)
```

### Tagged Union (discriminated union)

```c
typedef enum { VAL_INT, VAL_FLOAT, VAL_STRING } ValueType;

typedef struct {
    ValueType type;
    union {
        int i;
        float f;
        char s[64];
    };  // anonymous union (C11)
} Value;

Value v = {.type = VAL_INT, .i = 42};

switch (v.type) {
case VAL_INT:    printf("%d\n", v.i); break;
case VAL_FLOAT:  printf("%f\n", v.f); break;
case VAL_STRING: printf("%s\n", v.s); break;
}
```

### Bit Fields

```c
struct Flags {
    unsigned int readable  : 1;  // 1 bit
    unsigned int writable  : 1;
    unsigned int executable: 1;
    unsigned int           : 5;  // 5-bit padding
    unsigned int mode      : 3;  // 3 bits (values 0-7)
};

struct Flags f = {.readable = 1, .writable = 1, .mode = 5};
```

---

## Enums and Typedefs

### Enums

```c
enum Color { RED, GREEN, BLUE };        // 0, 1, 2
enum Color c = RED;

enum Status {
    OK = 0,
    ERR_NOT_FOUND = -1,
    ERR_TIMEOUT = -2,
};

// Enums are ints; there is no type safety preventing out-of-range values.
```

### Typedefs

```c
typedef unsigned long ulong;
typedef int (*CompareFunc)(const void *, const void *);  // function pointer type

// Typedef with struct (avoids writing "struct" everywhere)
typedef struct {
    char name[64];
    int age;
} User;

User u = {.name = "Alice", .age = 30};

// Opaque types (hide implementation in .c, expose typedef in .h)
// In header:
typedef struct Database Database;
Database *db_open(const char *path);
void db_close(Database *db);
// In source:
struct Database {
    int fd;
    // internal fields...
};
```

---

## Dynamic Memory

```c
#include <stdlib.h>

// Allocate
int *p = malloc(10 * sizeof(int));       // uninitialized memory
int *p = calloc(10, sizeof(int));        // zero-initialized
if (p == NULL) { /* handle allocation failure */ }

// Resize
p = realloc(p, 20 * sizeof(int));        // may move the block
// If realloc fails, it returns NULL and the original block is still valid.
// Common bug: p = realloc(p, ...) leaks the original on failure.
int *tmp = realloc(p, new_size);
if (tmp == NULL) {
    free(p);
    return -1;
}
p = tmp;

// Free
free(p);
p = NULL;  // prevent use-after-free (dangling pointer)

// Common mistakes:
// - Double free: calling free(p) twice
// - Use-after-free: accessing memory after free
// - Memory leak: losing the pointer without freeing
// - Buffer overflow: writing past the allocated size
```

### Flexible Array Members (C99)

```c
// Last member can be an incomplete array (size determined at allocation)
typedef struct {
    size_t length;
    int data[];  // flexible array member, must be last
} IntArray;

IntArray *arr = malloc(sizeof(IntArray) + n * sizeof(int));
arr->length = n;
for (size_t i = 0; i < n; i++) {
    arr->data[i] = i;
}
free(arr);
```

---

## Preprocessor

### Macros

```c
// Object-like macros
#define PI 3.14159265358979
#define MAX_BUF 4096

// Function-like macros (watch out for side effects and missing parentheses)
#define MAX(a, b) ((a) > (b) ? (a) : (b))
#define SQUARE(x) ((x) * (x))
// Pharentesis are needed to avoid errors when we use an expression as input
// SQUARE(i++) is UB! Prefer inline functions for anything non-trivial.

// Multi-line macro
#define SWAP(a, b) do { \
    typeof(a) _tmp = (a); \
    (a) = (b);            \
    (b) = _tmp;           \
} while (0)

// Stringification
#define STR(x) #x                // STR(hello) -> "hello"

// Token pasting
#define CONCAT(a, b) a##b        // CONCAT(foo, bar) -> foobar

// Variadic macro
#define LOG(fmt, ...) fprintf(stderr, fmt "\n", ##__VA_ARGS__)

// Undef
#undef MAX
```

### Conditional Compilation

```c
#ifdef DEBUG
    printf("debug info\n");
#endif

#ifndef HEADER_H
#define HEADER_H
// header contents
#endif

#if defined(__linux__)
    // Linux-specific code
#elif defined(__APPLE__)
    // macOS-specific code
#elif defined(_WIN32)
    // Windows-specific code
#endif

#if __STDC_VERSION__ >= 201112L
    // C11 features
#endif
```

### Predefined Macros

```c
__FILE__           // current filename (string)
__LINE__           // current line number (int)
__func__           // current function name (C99, string)
__DATE__           // compilation date "Mmm dd yyyy"
__TIME__           // compilation time "hh:mm:ss"
__STDC__           // 1 if conforming to standard C
__STDC_VERSION__   // 199901L (C99), 201112L (C11), 201710L (C17)
```

### Pragmas

```c
#pragma once                     // non-standard but widely supported include guard
#pragma pack(push, 1)            // set struct packing alignment to 1
#pragma pack(pop)
_Pragma("once")                  // C99 _Pragma operator (usable in macros)
```

---

## Header Files and Compilation Units

### Header File Structure

```c
// utils.h
#ifndef UTILS_H
#define UTILS_H

#include <stddef.h>

// Public API declarations
typedef struct Config Config;

Config *config_load(const char *path);
void config_free(Config *cfg);
const char *config_get(const Config *cfg, const char *key);

#endif // UTILS_H
```

### Source File Structure

```c
// utils.c
#include "utils.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Private to this file
struct Config {
    char **keys;
    char **values;
    size_t count;
};

// Static helper (internal linkage, not visible outside this file)
static char *duplicate_string(const char *s) {
    char *copy = malloc(strlen(s) + 1);
    if (copy) strcpy(copy, s);
    return copy;
}

// Public implementation
Config *config_load(const char *path) {
    // ...
}
```

### Include Conventions

```c
#include <stdio.h>       // system/standard headers (searched in system paths)
#include "myheader.h"    // project headers (searched in local path first)

// Order convention:
// 1. Corresponding header (e.g., utils.c includes "utils.h" first)
// 2. Standard library headers
// 3. Third-party library headers
// 4. Project headers
```

### Compilation Model

```
source files (.c)  -->  preprocessor  -->  compiler  -->  object files (.o)
                                                              |
                                       linker  <--------------+
                                         |
                                      executable
```

```bash
# Compile to object file
gcc -c utils.c -o utils.o

# Link object files into executable
gcc main.o utils.o -o myapp

# Or all at once
gcc main.c utils.c -o myapp
```

---

## File I/O

```c
#include <stdio.h>

// Open / close
FILE *f = fopen("data.txt", "r");   // "r", "w", "a", "rb", "wb", "r+", etc.
if (f == NULL) {
    perror("fopen");
    return -1;
}
// ... use f ...
fclose(f);

// Reading
int ch = fgetc(f);                   // read one char, returns EOF on end
char *line = fgets(buf, sizeof(buf), f); // read line (includes '\n'), NULL on EOF
size_t n = fread(buf, sizeof(char), count, f); // binary read

// Writing
fputc('A', f);
fputs("hello\n", f);
fprintf(f, "value: %d\n", 42);
size_t n = fwrite(buf, sizeof(char), count, f); // binary write

// Seeking
fseek(f, 0, SEEK_SET);              // beginning
fseek(f, 0, SEEK_END);              // end
fseek(f, offset, SEEK_CUR);         // relative to current position
long pos = ftell(f);                 // current position
rewind(f);                           // equivalent to fseek(f, 0, SEEK_SET)

// Flushing
fflush(f);                           // write buffered data to file
fflush(stdout);                      // force stdout flush (useful before input)

// Check state
feof(f)                              // true if end-of-file reached
ferror(f)                            // true if an error occurred
clearerr(f)                          // reset error and EOF indicators

// Read entire file
fseek(f, 0, SEEK_END);
long size = ftell(f);               
rewind(f);                          // return to beginning
char *content = malloc(size + 1);
fread(content, 1, size, f);
content[size] = '\0';

// Standard streams
stdin   // standard input
stdout  // standard output (line-buffered)
stderr  // standard error (unbuffered)
```

### File Modes

| Mode | Description |
|------|-------------|
| `"r"` | Read (file must exist) |
| `"w"` | Write (creates or truncates) |
| `"a"` | Append (creates if needed) |
| `"r+"` | Read and write (file must exist) |
| `"w+"` | Read and write (creates or truncates) |
| `"a+"` | Read and append |
| `"rb"`, `"wb"` | Binary mode (no newline translation on Windows) |

---

## Formatted I/O

### printf / sprintf / snprintf

```c
printf("int: %d\n", 42);
printf("unsigned: %u\n", 42u);
printf("long: %ld\n", 42L);
printf("long long: %lld\n", 42LL);
printf("size_t: %zu\n", sizeof(int));
printf("float: %f\n", 3.14);
printf("scientific: %e\n", 3.14);
printf("string: %s\n", "hello");
printf("char: %c\n", 'A');
printf("pointer: %p\n", (void *)ptr);
printf("hex: 0x%x\n", 255);
printf("octal: %o\n", 255);
printf("percent: %%\n");

// Width and precision
printf("[%10d]\n", 42);        // right-align, width 10
printf("[%-10d]\n", 42);       // left-align
printf("[%010d]\n", 42);       // zero-padded
printf("[%.5s]\n", "hello world"); // truncate string to 5 chars
printf("[%.2f]\n", 3.14159);   // 2 decimal places
printf("[%*d]\n", width, 42);  // dynamic width

// To a buffer (always use snprintf, never sprintf)
char buf[64];
snprintf(buf, sizeof(buf), "User %s is %d years old", name, age);
```

### scanf

```c
int x;
scanf("%d", &x);                    // read int from stdin

char name[64];
scanf("%63s", name);                // read word (no spaces), limit to 63 chars

// fgets + sscanf is generally safer than scanf
char line[256];
fgets(line, sizeof(line), stdin);
sscanf(line, "%d %d", &a, &b);
```

---

## Bitwise Operations

```c
&    // AND
|    // OR
^    // XOR
~    // NOT (bitwise complement)
<<   // left shift
>>   // right shift

// Common patterns
flags |= MASK;          // set bits
flags &= ~MASK;         // clear bits
flags ^= MASK;          // toggle bits
if (flags & MASK) { }   // test bits

// Bit manipulation
#define BIT(n) (1U << (n))
#define SET_BIT(x, n)    ((x) |= BIT(n))
#define CLEAR_BIT(x, n)  ((x) &= ~BIT(n))
#define TOGGLE_BIT(x, n) ((x) ^= BIT(n))
#define TEST_BIT(x, n)   ((x) & BIT(n))

// Check if power of two
bool is_power_of_two(unsigned int x) {
    return x && !(x & (x - 1));
}

// Count trailing zeros, leading zeros, popcount (GCC/Clang builtins)
__builtin_ctz(x)      // count trailing zeros
__builtin_clz(x)      // count leading zeros
__builtin_popcount(x)  // count set bits
```

---

## Function Pointers

```c
// Declaration
int (*fp)(int, int);        // fp is a pointer to a function taking two ints, returning int

// Assignment
int add(int a, int b) { return a + b; }
fp = add;                   // or fp = &add (equivalent)

// Invocation
int result = fp(3, 4);      // or (*fp)(3, 4)

// As function parameter
void apply(int *arr, size_t len, int (*transform)(int)) {
    for (size_t i = 0; i < len; i++) {
        arr[i] = transform(arr[i]);
    }
}
int doubled(int x) { return x * 2; }
apply(arr, len, doubled);

// Typedef for readability
typedef int (*CompareFunc)(const void *, const void *);

int compare_ints(const void *a, const void *b) {
    return (*(const int *)a) - (*(const int *)b);
}

qsort(arr, len, sizeof(int), compare_ints);

// Array of function pointers (dispatch table)
typedef void (*Handler)(void);
Handler handlers[] = {handle_start, handle_stop, handle_pause};
handlers[action]();

// Returning a function pointer
typedef int (*Op)(int, int);
Op get_operation(char c) {
    switch (c) {
    case '+': return add;
    case '-': return subtract;
    default:  return NULL;
    }
}
```

---

## Variadic Functions

```c
#include <stdarg.h>

int sum(int count, ...) {
    va_list args;
    va_start(args, count);   // initialize, last named parameter

    int total = 0;
    for (int i = 0; i < count; i++) {
        total += va_arg(args, int);  // extract next argument by type
    }

    va_end(args);            // cleanup
    return total;
}

int result = sum(4, 10, 20, 30, 40); // 100

// Passing va_list to another function
void vlog(const char *fmt, va_list args) {
    vfprintf(stderr, fmt, args);
}

void log_error(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    vlog(fmt, args);
    va_end(args);
}
```

---

## Error Handling Patterns

C has no exceptions. Error handling is done through return values and errno.

### Return Codes

```c
// Convention: 0 = success, negative = error
int parse_config(const char *path, Config *out) {
    FILE *f = fopen(path, "r");
    if (!f) return -1;

    // ...
    if (bad_format) {
        fclose(f);
        return -2;
    }

    fclose(f);
    return 0;
}

// Convention: NULL = error for pointer-returning functions
User *user_find(int id) {
    // ...
    if (!found) return NULL;
    return user;
}
```

### errno

```c
#include <errno.h>
#include <string.h>

FILE *f = fopen("missing.txt", "r");
if (!f) {
    int err = errno;                     // save immediately (may be overwritten)
    fprintf(stderr, "Error %d: %s\n", err, strerror(err));
    perror("fopen");                     // prints "fopen: No such file or directory"
}

// Common errno values
ENOMEM    // out of memory
EINVAL    // invalid argument
ENOENT    // no such file or directory
EACCES    // permission denied
EAGAIN    // resource temporarily unavailable
EINTR     // interrupted system call
```

### Cleanup with goto

```c
// The goto-cleanup pattern is idiomatic C for multi-resource cleanup
int process_file(const char *path) {
    int ret = -1;
    FILE *f = NULL;
    char *buf = NULL;

    f = fopen(path, "r");
    if (!f) goto cleanup;

    buf = malloc(4096);
    if (!buf) goto cleanup;

    // ... do work ...
    ret = 0;

cleanup:
    free(buf);     // free(NULL) is safe
    if (f) fclose(f);
    return ret;
}
```

---

## Concurrency (pthreads)

POSIX threads. Link with `-lpthread`.

### Thread Creation and Joining

```c
#include <pthread.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d running\n", id);
    return NULL;
}

pthread_t thread;
int id = 42;
pthread_create(&thread, NULL, worker, &id);

void *result;
pthread_join(thread, &result); // block until thread finishes

// Detached thread (resources freed automatically on exit)
pthread_detach(thread);
```

### Mutex

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);
// critical section
counter++;
pthread_mutex_unlock(&lock);

// Dynamic initialization / destruction
pthread_mutex_t lock;
pthread_mutex_init(&lock, NULL);
// ... use ...
pthread_mutex_destroy(&lock);

// Trylock (non-blocking)
if (pthread_mutex_trylock(&lock) == 0) {
    // acquired
    pthread_mutex_unlock(&lock);
}
```

### Read-Write Lock

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

pthread_rwlock_rdlock(&rwlock);   // shared read lock (multiple readers)
// ... read ...
pthread_rwlock_unlock(&rwlock);

pthread_rwlock_wrlock(&rwlock);   // exclusive write lock
// ... write ...
pthread_rwlock_unlock(&rwlock);
```

### Condition Variables

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

// Waiting side
pthread_mutex_lock(&lock);
while (!ready) {                          // always use a while loop (spurious wakeups)
    pthread_cond_wait(&cond, &lock);      // atomically unlocks mutex and waits
}
// ... consume ...
pthread_mutex_unlock(&lock);

// Signaling side
pthread_mutex_lock(&lock);
ready = 1;
pthread_cond_signal(&cond);               // wake one waiter
// pthread_cond_broadcast(&cond);         // wake all waiters
pthread_mutex_unlock(&lock);
```

### C11 Atomics

```c
#include <stdatomic.h>

atomic_int counter = 0;

atomic_fetch_add(&counter, 1);
atomic_store(&counter, 0);
int val = atomic_load(&counter);
bool swapped = atomic_compare_exchange_strong(&counter, &expected, desired);

// Memory ordering (default is memory_order_seq_cst)
atomic_store_explicit(&counter, 0, memory_order_relaxed);
atomic_load_explicit(&counter, memory_order_acquire);
```

---

## Common Standard Library Functions

### stdlib.h

```c
// Process
exit(0);                  // terminate, run atexit handlers
_Exit(1);                 // terminate immediately, no cleanup
atexit(cleanup_func);     // register cleanup function
abort();                  // abnormal termination (raises SIGABRT)
system("ls -la");         // execute shell command

// Environment
char *val = getenv("HOME");

// Random numbers
srand(time(NULL));        // seed (call once)
int r = rand();           // random int in [0, RAND_MAX]
int r = rand() % 100;     // random int in [0, 99] (biased, fine for non-crypto)

// Integer arithmetic
int a = abs(-5);
long a = labs(-5L);
div_t d = div(17, 5);    // d.quot = 3, d.rem = 2
```

### string.h / memory

```c
memcpy(dst, src, n);      // copy n bytes (non-overlapping)
memmove(dst, src, n);     // copy n bytes (overlapping safe)
memset(ptr, 0, n);        // fill n bytes
memcmp(a, b, n);          // compare n bytes
memchr(ptr, ch, n);       // find byte in memory
```

### math.h (link with -lm)

```c
#include <math.h>

sqrt(x)   cbrt(x)
pow(x, y)
log(x)    log10(x)   log2(x)
exp(x)
fabs(x)                // absolute value (double)
ceil(x)   floor(x)    round(x)   trunc(x)
fmod(x, y)             // floating-point modulo
sin(x)    cos(x)      tan(x)
asin(x)   acos(x)     atan2(y, x)
INFINITY  NAN
isnan(x)  isinf(x)    isfinite(x)
```

### ctype.h

```c
#include <ctype.h>

isalpha(c)   isdigit(c)   isalnum(c)
isupper(c)   islower(c)   isspace(c)
toupper(c)   tolower(c)
isprint(c)   ispunct(c)
```

### assert.h

```c
#include <assert.h>

assert(ptr != NULL);          // aborts with message if false
assert(n > 0 && "n must be positive"); // trick: string is always truthy

// Compile with -DNDEBUG to disable all asserts in release builds
// Static assert (C11, evaluated at compile time)
_Static_assert(sizeof(int) == 4, "int must be 4 bytes");
```

### signal.h

```c
#include <signal.h>

volatile sig_atomic_t running = 1;

void handle_sigint(int sig) {
    running = 0;
}

signal(SIGINT, handle_sigint);    // register handler

while (running) {
    // main loop
}

// Common signals: SIGINT (Ctrl-C), SIGTERM, SIGSEGV, SIGABRT, SIGPIPE
```

---

## Build Tools and Compilation

### GCC / Clang Flags

```bash
# Basic compilation
gcc -o myapp main.c utils.c

# Warnings 
gcc -Wall -Wextra -Wpedantic -Werror -o myapp main.c

# Standards
gcc -std=c11 -o myapp main.c
gcc -std=c17 -o myapp main.c

# Optimization
gcc -O0 main.c    # no optimization (debug)
gcc -O2 main.c    # standard optimization
gcc -O3 main.c    # aggressive optimization
gcc -Os main.c    # optimize for size

# Debug symbols
gcc -g -o myapp main.c

# Sanitizers (invaluable for catching bugs)
gcc -fsanitize=address -g -o myapp main.c      # AddressSanitizer (buffer overflow, use-after-free)
gcc -fsanitize=undefined -g -o myapp main.c    # UBSan (undefined behavior)
gcc -fsanitize=thread -g -o myapp main.c       # ThreadSanitizer (data races)
gcc -fsanitize=address,undefined -g -o myapp main.c  # combine

# Link libraries
gcc -o myapp main.c -lm -lpthread -lssl

# Include and library paths
gcc -I/usr/local/include -L/usr/local/lib -o myapp main.c -lmylib

# Preprocessor defines
gcc -DDEBUG -DMAX_SIZE=1024 -o myapp main.c

# Generate dependency info (for Makefiles)
gcc -MM main.c
```

### Makefile (basic)

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c17 -g
LDFLAGS = -lm -lpthread
SRCS = main.c utils.c parser.c
OBJS = $(SRCS:.c=.o)
TARGET = myapp

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: clean
```

### CMake (basic)

```cmake
cmake_minimum_required(VERSION 3.20)
project(myapp C)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_compile_options(-Wall -Wextra -Wpedantic)

add_executable(myapp main.c utils.c parser.c)
target_link_libraries(myapp m pthread)

# Build:
# mkdir build && cd build
# cmake ..
# cmake --build .
```

### Useful Tools

| Tool | Purpose |
|------|---------|
| `gdb` / `lldb` | Debugger |
| `valgrind` | Memory leak / error detection |
| `strace` / `ltrace` | System call / library call tracing |
| `objdump -d` | Disassemble binary |
| `nm` | List symbols in object files |
| `ldd` | List shared library dependencies |
| `cppcheck` | Static analysis |
| `clang-format` | Code formatting |
| `clang-tidy` | Linter and static analysis |
| `gcov` / `lcov` | Code coverage |
| `perf` | Performance profiling (Linux) |
