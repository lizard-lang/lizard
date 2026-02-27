# Lizard Philosophy 🦎

*This document serves as the cultural and engineering baseline for the Lizard project.*

Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.

Lizard was built for modern developer happiness—but never at the expense of raw hardware physics. 

For thirty years, the software industry has forced developers into a false dichotomy: either write elegant, rapid-prototyping code in high-level languages (and suffer the GIL, the Garbage Collector, and the Virtual Machine), or write mathematically perfect, high-throughput code in C/C++ (and suffer decades of legacy syntax, ABI breakage, and manual memory fences).

Lizard destroys this dichotomy. We believe you can have the syntactic sugar and rapid iteration of Python, perfectly balanced with ruthless, bare-metal execution speed. 

In fact, Lizard looks at C++ and says: **"You aren't fast enough anymore. Your linker is slow, your struct padding is manual, your exceptions are a liability, and your syntax is from 1983."**

### 1. The Compiler is a Co-Pilot, Not a Nanny
Modern systems languages often adopt an adversarial relationship with the developer, enforcing strict borrow checkers to prevent you from hurting yourself, or hiding memory allocations entirely. 

Lizard assumes you are an expert. It provides massive quality-of-life improvements—like implicit hardware bus locking (`F0`) via Escape Analysis, so you don't have to manually write `std::atomic` barriers. But if you instruct the compiler to execute a raw, lock-free mutation on a shared memory pointer without a sequence counter (`-L3`), it will not warn you. It will compile the opcode, jump the instruction pointer, and let you deal with the catastrophic data race you requested. We give you the sugar, but you own the hardware.

### 2. The End of Religious Syntax Wars
Whitespace vs. Curly Braces is the most useless debate in computer science. Lizard ends it by supporting both. 
If you want to rapidly prototype a quantitative model using Pythonic indentation, do it. If you need to drop an explicit `{}` block in the middle of that exact same file to enforce deterministic RAII scope destruction for a 1ns spinlock, do it. The Dual-Parser Architecture normalizes it all into the exact same AST. We care about how fast the instructions execute, not what shape the text is on your screen.

### 3. We Do Not Participate in Security Theater (Obfuscation)
Lizard operates on a "Universal Recompilation Directive." Source code is distributed and JIT-compiled at runtime to guarantee perfect ABI alignment and dynamic microarchitecture targeting (AVX-512).

Because of this, we will never implement native source-code obfuscation or control-flow flattening. Obfuscation is an architectural anti-pattern. It pollutes the L1 instruction cache and actively sabotages the CPU's branch predictor. We will not compromise a 4ns execution pipeline to provide corporate management with the illusion of IP protection. 

If you must distribute proprietary algorithms without source code, compile it ahead-of-time to a standard C-ABI shared object (`.so`) and load it via our FFI. Let the community build obfuscators; the compiler remains pure.

### 4. The Standard Library is Not a Web Browser
Lizard will not ship with a 50-megabyte standard library containing JSON parsers, heavy HTTP clients, and bloated middleware. A massive standard library slows down compilation and encourages dependency rot.

We provide the mathematical primitives, the hardware IO topography tags (`[[writethrough]]`), fixed-point decimal scaling, lock-free ring buffers, and OS memory interfaces. The core stays ruthlessly lean. If you need to serve millions of HTTP requests, you use a dedicated, purpose-built engine like *Lizard on Tails*. 

### 5. Honest "Zero-Cost" Abstractions
We reject the legacy industry definition of "Zero-Cost," which often silently injects hidden virtual table pointers (`vptr`) or relies on catastrophic stack unwinding during exceptions. 

If a feature in Lizard claims to be zero-cost, it means it compiles down to exactly zero hardware cycles on the hot path. Lizard strictly bans asynchronous stack-unwinding exceptions. We utilize native monadic error handling (similar to `std::expected`). Yes, it introduces a conditional branch. But thanks to modern CPU branch predictors, the happy-path is functionally free, and the worst-case failure latency is mathematically capped. 

### 6. Drop the Tail
Software should fail cleanly and violently. 
When a thread encounters an unrecoverable state (a null pointer, a severed socket, a corrupted memory pool), it should not gracefully unwind its stack, execute 40 destructors, and write a polite XML crash report while the rest of the application hangs. 

It should sever its stack pointer (`XCHG EAX, ESP` 🔥) and die instantly. 
Latency is the only metric that matters. Do not apologize for crashing; just crash faster, reclaim a new stack from the pool, and process the next tick. 










Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.
