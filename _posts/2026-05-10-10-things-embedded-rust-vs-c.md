---
title: 10 Things My Embedded Rust Has That Your Embedded C Doesn't
date: 2026-05-10 12:00:00 +0000
categories: [Rust, Embedded Systems]
tags: [rust, embedded, c, comparison, programming-languages]
---

Whenever someone brings up Rust in any discussion, it's always about "memory safety". Especially when talking about Rust vs C/C++, many people just refuse to see anything else beyond it.

After having suffered for quite some time with Embedded C myself, in this article I will show 10 things that make Embedded Rust my go-to choice, deliberately ignoring everything related to memory and its safety.

Without further ado, let's dive into compile-time magic!

## 1. Type Checking

Let's assume you're writing a DMA driver. Following the best coding practices, you introduced a new type to denote a DMA transfer type:

```c
typedef enum {
    SINGLE = 1,
    LINKED_LIST = 2,
} xfer_t;

void start_dma_transfer (
    uint8_t *src,
    uint8_t *dst,
    xfer_t xfer
);
```

Do you think a C compiler will prevent you from calling the function like that?

```c
start_dma_transfer (&my_src, &my_dst, 1234);
```

Nope, it will happily compile it. Rust, at the other hand, will give you a compile time error, which you fully deserve.

## 2. Core Library

The C standard library was created for UNIX programmers and hasn't changed much since 1970. It has lots of functions, but somehow it never has that very function you need.

The Rust core library is the best friend of an embedded programmer. It is a tiny subset of the standard library (or lack thereof, since it's enabled by the `no_std` attribute) and doesn't even include dynamic memory allocation, let alone collections like `HashMap`, file I/O, or multithreading.

But it offers most of the great features you would ever need as an embedded programmer, all in one place:

- All the primitive types and millions of functions defined for each one (e.g. see the list of methods for `u8`)
- Iterators over arrays (one of the best zero-cost inventions of mankind)
- Traits (even better than Iterators)
- Option and Result types (even better than Traits)
- Atomic operations
- Overloadable operations
- Inline assembly

...and lots of other things.

If you need dynamic memory, you can optionally add the `alloc` library (assuming that someone has ported it to your target machine).

## 3. Pattern Matching instead of Switch/Case Statement

This one is short. One of the most infuriating things in C for me has always been the need to explicitly add "break" in every "case" of a "switch".

Rust solved the problem radically: it doesn't have a "switch" statement at all, instead it uses `match`, which is so much better on any level.

## 4. Absence of Pre-/Post-Increment Operators (`++i` and `i++`)

Yesterday I encountered a LinkedIn quiz where the author asked what will be the result of the following C++ code:

```cpp
int i = 5;
int x = i++ / ++i;
```

After adopting Rust you won't be able to produce such entertaining quizzes anymore, and your code will be boring and predictable:

```rust
let mut i = 5;
i += 1; // No more i++
let x = i / i;
```

## 5. Known Size for Enums

The C language doesn't specify the size of enums. In the example below it is always up to a compiler to choose the actual representation for A and B (called discriminant). Needless to say, this doesn't help when you need to talk to code written in another language. I personally encountered this while writing C code which was talking to a library in OpenCL C.

```c
typedef enum {
    A = 1234,
    B
} my_great_enum_t;
```

In Rust you can simply do this:

```rust
#[repr(u16)]
enum MyGreatEnum {
    A = 1234,
    B,
}
```

And you can be sure that the discriminant of `MyGreatEnum` values is `u16` - a 16-bit unsigned integer.

## 6. Generics and Typestate Pattern

What can be more beautiful than the following code for defining an array of N elements of an arbitrary type T (and T doesn't even need to be Copy if you initialize the array with `from_fn`):

```rust
struct MyBuffer<T, const N: usize> {
    buf: [T; N]
}
```

Ask Rust to come into your heart and you won't need to define four different buffer types for `u8`, `u16`, `u32`, `u64` or, god forbid, switch to C++.

Also, generics are used in Rust for implementing the Typestate Pattern, which is very handy for people writing HALs for embedded systems.

## 7. Assignment and Comparison Are Different Again

If you like bugs due to the code using an operator for assignment when the intention was to perform a comparison (see CWE-481 for examples and this Quora question for the background), Rust is not for you.

```c
// What if you accidentally type "x = 42"?
if (x == 42) {
    ...
}
```

I remember seeing a "defensive programming" article where the author recommended to develop a habit of writing C code conditions backwards (for which some desperate C programmers coined the term Yoda conditions), hoping that if you accidentally type "42 = x" then even an all-forgiving C compiler will complain:

```c
// Very defensive!
// Protects you from typing "42 = x"
if (42 == x) {
    ...    
}
```

Alternatively, you can switch to Rust, where the result of "x = 42" is not a boolean and can't be used in places where a boolean is expected.

## 8. Absence of Preprocessor

When coding in C, which way of defining constants you prefer? All the examples below are from the real code:

```c
#define ONE 1
#define TWO (2)
#define volatile
#define true false
```

Or, perhaps, you like nested `#ifdefs`? Bad news: there's no pre-processor in Rust.

Instead Rust offers:

- Conditional compilation using predefined and user-defined features
- Macros which are expanded into abstract syntax trees, rather than string preprocessing
- Constant functions, which are evaluated at compile-time

## 9. Mandatory Brackets for One-Liner If/Else

This nice indentation bug from Apple would not be possible in Rust, because curly brackets for if/else are mandatory even for one-liners:

```rust
// brackets are always mandatory!
if a != b {
    a = b;
}
```

## 10. Private, Public and Re-exported Values. No Headers Anymore

Everything you define is by default private to your crate (a "crate" in Rust parlance is either a library or a binary). This includes variables, types, traits, functions, modules - everything.

Rust gives you fine-grain control over what you make visible to external users, to the other parts of your crate, or to a particular compile unit.

And with all that power Rust doesn't need header files anymore.

## 11. Methods for Structs

I have to confess, having no methods for structs in C was one of the reasons I almost succumbed to C++ back in the day.

Wait, I promised you 10 things, and I am already on 11th, and haven't even mentioned yet that in Rust:

- You can return more than one value from a function
- You have an integrated documentation system
- There's no need for makefiles to build your project

...and many other features accumulated over the 50 years of research since the times of C inception. And memory safety, of course!

## Afterword

Rust is not perfect. Lifetimes, macros, async, the dreadful borrow checker, the need to use raw pointers instead of references for volatile accesses, and simply the sheer complexity of the language (comparable to the complexity of C++) may not be for everyone.

Regarding the ecosystem, there is only one compiler (plus its automotive-grade version Ferrocene), limited support for less popular CPU architectures and platforms (even though Rust uses LLVM backend), and reluctance of many big players in the embedded world to pay any attention to the language.

The motto of Rust, which is "if it compiles - it works", is also not completely true. But somehow, it is still much more pleasant to use, faster to write, and easier to debug than C.

Give it a try.
