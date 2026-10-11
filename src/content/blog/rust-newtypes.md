---
title: "Encoding Program Logic In Strong Types Is A Faustian Bargain"
description: "Better the devil that's compile time checked"
pubDate: "Oct 10 2026"
heroImage: "../../assets/blog-placeholder-4.jpg"
---

Rust is the only usable language that gives you tools for "perfect" type safety but those tools increase your compilation times, hit your runtime performance and have awful DX with verbose boilerplate. 

## "Zero-Cost" Means Two Different Things

The issue here is that the term "zero-cost abstraction" can mean both 
- abstraction that costs nothing compared to a bespoke implementation  
- abstraction that you couldn't have implemented better yourself 

One may get confused about the difference between those and many do, thinking it means the former when more often than not it means the latter. 

The issue comes from not understanding the term "abstraction". In this context "using abstraction" means abstracting away the actual implementation. You're not working with details of the implementation and instead a packaged "abstraction". 

For example, `String` is an abstraction. Let's say you get some string data that you know is between 0 and 4 ASCII characters. You can write your own implementation with `[u8; 4]` handling (or not handling) invalid data, implement parsing and display handlers. Or you can use the standard `String` type that abstracts it for you. 

You may think that because `String` is a zero-cost abstraction as it is advertised as it would mean the compiler would be smart enough to optimize its usage and infer it only ever handles ASCII strings of length 4 or less. The unfortunate reality is that it simply isn't. Even Rust's compiler doesn't dare make such aggressive optimizations. 

"You don't pay for what you don't use" also means that you used the whole `String` abstraction and you're paying for the entire `String` abstraction, all 24 bytes on the stack and every heap allocation/deallocation. 

## Encoding Program Correctness in Types

Any _statically-typed_ language will advertise to you this kind of pattern: you have 2 functions doing `A` and `B` and you need `A` done before doing `B`. 

```python
def main() -> None:
    do_a()
    do_b()
```

Technically nothing in this kind of code stops you from calling `do_b` before `do_a` resulting in a critical uncaught error. So the statically-typed language offers a solution, return a zero-sized type from `do_a` as proof it was called before and have `do_b` accept it as a parameter

```rust
fn main() {
    let proof: ProofAWasDone = do_a();
    db_b(proof);
}
```

It is a nice demonstration of encoding program correctness in the type system and is also free (when advertised as such) with the compiler optimizing away the zero-sized variable. 

However, real-life codebases rarely follow this pattern due to the complexity of real-life software. While this particular minimalistic example is zero cost you may find yourself seeing performance regressions unexpectedly trying to actually encode the program lifetime state machine with types. 

I have a couple of anecdotal examples to present but the main point you ought to take away is this: in modern software development you're not safe from pitfalls even in the safest places. 

## When "Zero-Cost" Abstractions Aren't

There was a popular runtime regression over a newtype going around, the wrapper broke specialization leading to a performance regression of several magnitudes. 

```rust
fn main() {
    let t = Instant::now();
    let v = vec![0u8; 1<<34];
    println!("{:?}", t.elapsed());
}
```

```text
$ cargo run --release
4.291µs
```

```rust
fn main() {
    let t = Instant::now();
    let v = vec![WrappedByte(0); 1<<34];
    println!("{:?}", t.elapsed());
}
```

```text
$ cargo run --release
5.299196942s
```

I won't go into details as you can read the [original blog post](https://blog.polybdenum.com/2021/08/09/when-zero-cost-abstractions-aren-t-zero-cost.html) about it, and I am honestly not sure whether it's been addressed since, but that's beside the point. The point is that it's possible and there are likely more that you don't know about. 

## Compile Times and Duplicated Code

The most important problem, especially for Rust, is compilation time. Many new types cause longer compiliation times. It's simple: more code = longer compilation. 

It gets worse. Extremely common - the new type just a wrapper over some primitive and it needs to replicate some but not all behaviours of the primitive. But because of Rust compiler not being able to ensure all the methods on all the wrappers are truly identical it may fail to optimize them all into one. But Rust does not compile with identical code folding (ICF) enabled by default, it's an opt-in linker optimization. So identical methods duplicated across all your newtypes stay duplicated in the final binary, that means a larger binary and instruction-cache evictions at runtime. 

## You're Paying for Features You Don't Use

When I first started learning Rust YouTube recommended me [this great video](https://youtu.be/A4cKi7PTJSs) by Logan Smith about the inefficiency of the commonly used `Vec<T>` type (and by extension the `String` type). 

```rust
/// simplified
/// https://doc.rust-lang.org/src/alloc/vec/mod.rs.html#436
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}
/// https://doc.rust-lang.org/src/alloc/raw_vec/mod.rs.html#84
struct RawVecInner {
    ptr: Unique<u8>,
    cap: usize,
}

/// https://doc.rust-lang.org/beta/src/alloc/string.rs.html#353
pub struct String {
    vec: Vec<u8>,
}
```

At the time it frankly went over my head because all this time, coming from JavaScript and Python I learned to treat `str` type just as any other primitive like `int`, why not use the string type which is the most convenient like everyone else. And I wouldn't fault anyone else for not thinking about it too hard the same way, even after working with Rust or any other performance-oriented code for years. 

Only after 2 years with Rust the video was recommended to me again and I really understood the point. 
Most software developers don't ever think that they're paying for mutability of their arrays with the separate `length` and `capacity` values, both taking up 8 bytes each on 64-bit platforms. 

### Real World Code Example

There is a crate I've seen used that illustrates the problem - [`aliri_braid`](https://crates.io/crates/aliri_braid) (about 450k recent downloads at the time of writing). 
It exists to solve a very common problem - a string can be many things, an ID, a username, a token, seems like a perfect use case for newtypes - to not mess up your strings. 

But if you look at the implementation you will find the default underlying string type in use is `String`. How often do you have a database name or a user id which you may mutate the length over that you need an extra 8 bytes for capacity? Would you think such a default is justified? 

> Braids of custom string types are also supported, so long as they implement a set of expected traits. If not specified, the type named String in the current namespace will be used.

```rust
/// https://docs.rs/aliri_braid/0.4.0/aliri_braid/#usage
use aliri_braid::braid;
use compact_str::CompactString as String;

#[braid]
pub struct UserId;
```

Unfortunately it also highlights another problem - creating a new type is bothersome and boilerplate-y that you would dedicate a whole crate to it and delegate everything to it. And while the crate is hardly unused it's not exactly popular either meaning most developers choose to not type-ensure their strings at all than use a dedicated crate that makes things as simple as possible. 

### Box<[T]> Is Usually Enough

But I don't agree with the video completely, I think we ought to go further. As the title and the thumbnail of the video suggest Logan Smith proposes usage of standard library `Arc<[T]>` over `Vec<T>` and only briefly mentions `Rc<[T]>` and `Box<[T]>`. In my personal experience 99% of array and string usage in a program is plenty satisfied by `Box<[T]>` and `Box<str>` with immutable references to it. 
You oughtn't pay for what you don't use and you don't use a lot of features `Arc` provides: truly large amount of possible references - how often do you really need 5 billion references to the exact same string in a program's lifetime, all those references being thread-safe - not just the string data itself because `Box<str>` by itself is `Send + Sync` and most importantly asynchronous string lifetime - is the lifetime of a single string really that complex that an immutable reference `&str` won't cut it. 

Ultimately the inefficiencies in developer choices don't affect just themselves, they also affect the users of their code. If a crate you're using has a `String` in there somewhere for what could've been a `TinyAsciiStr` on the stack your only options are to eat the allocation or go nuclear and fork. 

## Two Ways Out

There are only two ways out of zero-cost abstractions being stuck fence-sitting: pick a side and either commit to go elsewhere: 
- Give up and accept the cost of abstractions for their convenience (what most people are doing)
- Make the compilers smarter

If the written code can give more guarantees, give more precise instructions to the compiler it will be able to figure out what you want to do and optimize accordingly, eliminating leaky abstractions. 

And that means only one thing: more pure functional programming. 
Unfortunately at the moment of writing neither the developers nor the environment these developers operate in is receptive to functional paradigm. Not only is it inherently difficult as it requires one to think about processes using new concepts the bare metal we all work with is built using Von Neumann architecture meaning you have to map Lambda calculus on top of it rather than working with it. 

As decades have shown hardware is the bane of software people so I expect in the coming years the shift towards the opposite: acceptance of inefficiency as inevitable and abandonment of performance techniques in every micro-inconvenience. 
