# Lizard Architecture 🦎

*This document serves as my architecture brainstorming establishing prior art.*

Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.

## 1. TLB Annihilation: Compiling into `memfd` Huge Pages

Traditional C/C++ compilers (GCC, Clang) and their associated linkers (LD) strictly isolate binary sections (`.text`, `.data`, `.rodata`, `.bss`) into standard 4KB memory pages. For a high-performance application, traversing these fragmented 4KB pages results in catastrophic Translation Lookaside Buffer (TLB) misses, forcing the CPU to repeatedly walk the page tables in main memory.

**Lizard bypasses the ELF loader entirely.** 

Lizard compiles the Abstract Syntax Tree (AST) directly into memory using Linux `memfd_create`. 
The execution environment explicitly requests memory in the following degradation order:
1. **1GB Huge Pages (`MAP_HUGETLB | MAP_HUGE_1GB`)**
2. **2MB Huge Pages (`MAP_HUGETLB | MAP_HUGE_2MB`)**
3. **Standard 4KB Pages (Fallback)**

### The Contiguous Execution Matrix
By fusing executable instructions and static data into a single 1-Gigabyte page, Lizard achieves a TLB footprint of exactly **one entry**. The CPU's Memory Management Unit (MMU) caches the page translation once, and the entire application executes without a single hardware page walk. 

Because modern linkers heavily resist fusing `.text` (executable) and `.data` (writable) into 1GB pages for security (W^X protections), Lizard acts as its own dynamic linker. It emits raw x86_64 opcodes directly into the `memfd` block, seals the memory descriptors, and executes a direct `JMP` to the entry point. 

## 2. Implicit Concurrency: Escape Analysis & Hardware Locking

Modern languages rely on developers manually placing `std::mutex` or `std::atomic` around shared memory. This is fundamentally flawed, as it requires human perfection to avoid race conditions and relies on the OS scheduler to manage thread contention. 

Lizard implements **Hardware-Level Implicit Concurrency** via AST Escape Analysis.

During the lexing and parsing phase, Lizard traces the lifespan of every memory pointer:
*   **Thread-Local Memory:** If a variable is mathematically proven to never leave the scope of its spawning thread, the compiler emits pure, raw x86_64 arithmetic instructions.
*   **Escaped Memory (Global/Shared):** If a variable crosses a thread boundary, Lizard automatically injects the x86 `LOCK` prefix (`F0`) into the instruction stream (e.g., `LOCK ADD`, `LOCK XCHG`). 

Under the `-L2` optimization flag, Lizard entirely bypasses the Linux kernel scheduler. It does not yield threads. It forces the physical AMD Infinity Fabric and memory controllers to arbitrate the collision directly on the silicon bus. 

## 3. Structural Geometry: The `-L1` Implicit Seqlock

Under the `-L1` (Strict Consistency) flag, Lizard automatically alters the structural geometry of user-defined data objects to guarantee tear-free reads across multiple cache lines.

When a multi-field struct is declared, Lizard silently prepends a 64-bit Sequence Counter (`seqlock`) to the memory layout, enforcing strict L1 cache-line alignment. 
*   **Writers:** The compiler automatically wraps writes by incrementing the sequence counter (odd = locked, even = unlocked).
*   **Readers:** The compiler automatically emits a spin-read loop. It reads the sequence counter, loads the struct memory, and verifies the sequence counter has not changed. 

This provides mathematically perfect 1-Writer / N-Reader synchronization without the developer writing a single line of lock-free logic.

## 4. Deterministic Error Handling: The Zero-Unwind Architecture

Traditional exception handling (C++ `throw`, Python tracebacks) relies on asynchronous stack unwinding. While often marketed as "zero-cost" on the happy path, throwing an exception requires the runtime to parse compressed stack-frame tables (DWARF) and execute reverse-destructors. This introduces catastrophic, non-deterministic latency spikes (often exceeding 5,000 microseconds). In a high-frequency pipeline, a 5ms stall is a fatal system failure. 

Lizard strictly bans asynchronous stack unwinding. 

Instead, Lizard implements **Native Monadic Error Handling** (architecturally similar to C++23 `std::expected` or Rust `Result`) directly at the compiler level.

### The Microstructural Trade-off
By returning union-based value/error types, Lizard forces error checking into the direct instruction stream via standard conditional branching (`CMP`, `Jcc`). 
* **The Cost:** We willingly trade the "free" happy-path of traditional exceptions for a potential 1-cycle branch evaluation.
* **The Gain:** We mathematically cap the worst-case execution time. 

Modern AMD/Intel branch predictors (TAGE) will natively learn the happy-path of a `std::expected` equivalent, reducing the branch cost to effectively zero cycles. When an error *does* occur, the pipeline simply takes the alternate branch. The latency of failure is identical to the latency of success. Execution remains 100% deterministic, lock-free, and contained within the L1 cache.

*(Note: In the event of an unrecoverable hardware fault, such as a null-pointer dereference or OOM, the runtime defaults to the "Tail Drop" protocol—instantly severing the thread's stack pointer `XCHG EAX, ESP` rather than attempting a safe unwind).*

## 5. Syntactic Duality: The Scope-Dominant Hybrid Lexer

Programming language syntax is traditionally strictly bifurcated: either whitespace-delimited (Python) or brace-delimited (C/C++/Rust). Lizard fundamentally rejects this dichotomy, implementing a **Scope-Dominant Hybrid Lexer**.

The compiler permits developers to mix Pythonic off-side rule indentation with explicit C-style scoping braces `{}` and statement terminators `;` within the exact same file, and even within the same function.

### The Lexical Resolution Protocol
To resolve parsing ambiguities without sacrificing compilation speed, the Lizard AST strictly prioritizes explicit tokens over implicit whitespace:

1. **Brace Dominance:** The `{` and `}` tokens dictate absolute Basic Block geometry. If the lexer encounters a `{`, it temporarily suspends whitespace-based `INDENT/DEDENT` tracking. The scope is mathematically locked until the matching `}` is consumed. 
2. **Deterministic RAII Scoping:** Because `{}` forces a hard scope boundary, developers can arbitrarily inject anonymous blocks into Pythonic code. This allows for hyper-precise object lifetime management and deterministic RAII (e.g., dropping a 1ns uncontested spinlock mid-function) without the overhead of Python's `with` context managers.
3. **Terminator Forgiveness:** The lexer treats the newline character `\n` as a soft statement terminator, but treats the semicolon `;` as a hard override. This allows C++ veterans to retain muscle memory without triggering syntax errors in Pythonic blocks.

### Code Example: Valid Lizard Syntax
```python
def exampleConcurrentLizardFunction():
     do_stuff();
     if sharedData.not_contested:
     {
          # The {} block overrides the whitespace parser.
          # The lock is deterministically acquired and released 
          # strictly within this physical boundary.
          ScopeGuard guard(sharedData.lock);
          print(f"suck it python");
     }
```
Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.
