# Fibonacci benchmarks between js, wasm and server
<br>
Date: 2023-11-15
<br>

## Introduction

WebAssembly can't directly access the DOM, it has to call javascript and is known to incur a cost when doing so. What about raw computation, how does wasm compare to server-side computation or client-side javascript computation? And when is it favorable to use it?

The source code for the benchmark can be found [here](https://github.com/MoAlyousef/fib-bench), along with instructions on how to build it.

## Results
With an input value of 1:
- servertime: 6.831298828125 ms
- wasmtime: 0.008056640625 ms
- jstime: 0.004150390625 ms

With an input value of 45:
- servertime: 2983.470703125 ms
- wasmtime: 8184.0751953125 ms
- jstime: 15975.77490234375 ms

The results should appear in the browser's dev console. 

This was run on a windows machine running wsl2 x86_64 GNU/Linux.
Specs:
- Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz
- Speed: 3.40 GHz
- Cores: 4
- Logical processors: 8
- RAM: 16 GB
- HDD: ST1000LM035-1RK172

Rust version: 1.71 stable.
Google chrome: Version 119.0.6045.107 (Official Build) (64-bit)

wasm-opt -O3 didn't improve performance by much. It did reduce the generated wasm size by 30 percent however.

## Conclusion
- Performing server-side computations requires a network call and marshalling data to and from the server, which incurs an unnecessary cost when the computation is trivial. In such cases javascript and wasm offer a close enough computation cost. A wasm function call can be twice as slow as the javascript one when having to manipulate the DOM. 
- For intensive computations, the server cost can be considered negligible since native computation remains faster than both wasm and js. Even then, client-side javascript is only twice as slow as wasm!
- Wasm on the browser, to me, makes sense when wanting to target the web using a different language than javascript. Although I'm no fan of js, js browser engines do a good job at optimizing it. However other languages do bring other advantages to the table, either in language merit or ecosystem. It also makes sense if you're serving static web pages or SPA's and not handling (or can't handle) post requrests, or want to reduce server computations or avoid network issues. 
