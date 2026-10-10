---
title: "Rust's New Types"
description: "Zero-cost abstraction with a cost"
pubDate: "Oct 10 2026"
heroImage: "../../assets/blog-placeholder-4.jpg"
---

Rust is the only usable language that gives you tools for "perfect" type safety but those tools increase your compilation times, hit your runtime performance and have awful DX with verbose boilerplate. 

The issue here is that the term "zero-cost abstraction" can mean both 
- abstraction that costs nothing compared to a bespoke implementation  
- abstraction that you couldn't have implemented better yourself 

One may get confused about the difference between those and many do, thinking it means the former when more often than not it means the latter. 

The issue comes from not understanding the term "abstraction". In this context "using abstraction" means abstracting away the actual implementation. You're not working with details of the implementation and instead a packaged "abstraction". 

For example, `String` is an abstraction. Let's say you get some string data that you know is between 0 and 4 ASCII characters. You can write your own implementation with `[u8; 4]` handling (or not handling) invalid data, implement parsing and display handlers. Or you can use the standard `String` type that abstracts it for you. 

You may think that because `String` is a zero-cost abstraction as it is advertized as it would mean the compiler would be smart enough to optimize its usage and infer it only ever handles ASCII strings of length 4 or less. The unfortunate reality is that it simply isn't. Even Rust's compiler doesn't dare make such aggressive optimizations. 

"You don't pay for what you don't use" also means that you used the whole `String` abstraction and you're paying for the entire `String` abstraction, all 24 bytes on the stack and every heap allocation/deallocation. 

<!-- 
HERE I would actually talk about how Rust obviously has compilation time penatlies for using new types. 
Then how at runtime the new types kill LLVM specializaiton, then how the new types generate duplicate wrapper code negatively impacting runtime cache
Then about how actually writing new types for every little thing is very rare among real projects. 
Then point to the "Use Arc<str> over String" video saying how it didn't make sense to me when I first learned Rust but coming back to it after 2 years it opened my eyes, how most programmers don't think about even String type too hard as it is usually presented to them as a primitive like int or float
then talk about crates like `secrecy` paying for the String when they could be using Box<str>
 -->

There is only two ways out of zero-cost abstractions being neither here nor there: pick a side and either commit to being here to go elsewhere: 
- Give up and accept the cost of abstractions for their convenience (what most people are doing)
- Make the compilers smarter

If the written code can give more guarantees, give more precise instructions to the compiler it will be able to figure out what you want to do and optimize accordingly, eliminating leaky abstractions. 

And that means only one thing: more pure functional programming. 
Unfortunately at the moment of writing neither the developers nor the environment these developers operate in is receptive to functional paradigm. Not only is it inherently difficult as it requires one to thing about processes using new concepts the bare metal we all work with is built using Von Neumann architecture meaning you have to map Lambda calculus on top of it rather than working with it. 

As decades have shown hardware is the bane of software people so I expect in the coming years the shift towards the opposite: acceptance of inefficiency as inevitable and abandonment of performance techniques in every micro-inconvenience. 
