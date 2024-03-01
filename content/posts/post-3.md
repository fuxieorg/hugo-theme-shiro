---
title: 'Why is esbuild fast?'
date: 2024-03-01
draft: false
---

> This article was originally published on the [esbuild documentation](https://esbuild.github.io/faq/#why-is-esbuild-fast).

Several reasons:

- It's written in [Go](https://go.dev/) and compiles to native code.

  Most other bundlers are written in JavaScript, but a command-line application is a worst-case performance situation for a JIT-compiled language. Every time you run your bundler, the JavaScript VM is seeing your bundler's code for the first time without any optimization hints. While esbuild is busy parsing your JavaScript, node is busy parsing your bundler's JavaScript. By the time node has finished parsing your bundler's code, esbuild might have already exited and your bundler hasn't even started bundling yet.

  In addition, Go is designed from the core for parallelism while JavaScript is not. Go has shared memory between threads while JavaScript has to serialize data between threads. Both Go and JavaScript have parallel garbage collectors, but Go's heap is shared between all threads while JavaScript has a separate heap per JavaScript thread. This seems to cut the amount of parallelism that's possible with JavaScript worker threads in half [according to my testing](https://github.com/evanw/esbuild/issues/111#issuecomment-719910381), presumably since half of your CPU cores are busy collecting garbage for the other half.

- Parallelism is used heavily.

  The algorithms inside esbuild are carefully designed to fully saturate all available CPU cores when possible. There are roughly three phases: parsing, linking, and code generation. Parsing and code generation are most of the work and are fully parallelizable (linking is an inherently serial task for the most part). Since all threads share memory, work can easily be shared when bundling different entry points that import the same JavaScript libraries. Most modern computers have many cores so parallelism is a big win.

- Everything in esbuild is written from scratch.

  There are a lot of performance benefits with writing everything yourself instead of using 3rd-party libraries. You can have performance in mind from the beginning, you can make sure everything uses consistent data structures to avoid expensive conversions, and you can make wide architectural changes whenever necessary. The drawback is of course that it's a lot of work.

  For example, many bundlers use the official TypeScript compiler as a parser. But it was built to serve the goals of the TypeScript compiler team and they do not have performance as a top priority. Their code makes pretty heavy use of [megamorphic object shapes](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html) and unnecessary [dynamic property accesses](https://github.com/microsoft/TypeScript/issues/39247) (both well-known JavaScript speed bumps). And the TypeScript parser appears to still run the type checker even when type checking is disabled. None of these are an issue with esbuild's custom TypeScript parser.

- Memory is used efficiently.

  Compilers are ideally mostly O(n) complexity in the length of the input. So if you are processing a lot of data, memory access speed is likely going to heavily affect performance. The fewer passes you have to make over your data (and also the fewer different representations you need to transform your data into), the faster your compiler will go.

  For example, esbuild only touches the whole JavaScript AST three times:

  1. A pass for lexing, parsing, scope setup, and declaring symbols
  2. A pass for binding symbols, minifying syntax, JSX/TS to JS, and ESNext-to-ES2015
  3. A pass for minifying identifiers, minifying whitespace, generating code, and generating source maps

  This maximizes reuse of AST data while it's still hot in the CPU cache. Other bundlers do these steps in separate passes instead of interleaving them. They may also convert between data representations to glue multiple libraries together (e.g. string→TS→JS→string, then string→JS→older JS→string, then string→JS→minified JS→string) which uses more memory and slows things down.

  Another benefit of Go is that it can store things compactly in memory, which enables it to use less memory and fit more in the CPU cache. All object fields have types and fields are packed tightly together so e.g. several boolean flags only take one byte each. Go also has value semantics and can embed one object directly in another so it comes "for free" without another allocation. JavaScript doesn't have these features and also has other drawbacks such as JIT overhead (e.g. hidden class slots) and inefficient representations (e.g. non-integer numbers are heap-allocated with pointers).

Each one of these factors is only a somewhat significant speedup, but together they can result in a bundler that is multiple orders of magnitude faster than other bundlers commonly in use today.