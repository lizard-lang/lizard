# Lizard Architecture 🦎

*This document serves as my architecture brainstorming establishing prior art.*

Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.

## 1. TLB Annihilation: Cross-Platform Contiguous Huge Pages

Traditional C/C++ compilers (GCC, Clang, MSVC) and their associated linkers strictly isolate binary sections (`.text`, `.data`, `.rodata`, `.bss`) into standard 4KB memory pages. For a high-performance application, traversing these fragmented 4KB pages results in catastrophic Translation Lookaside Buffer (TLB) misses, forcing the CPU to repeatedly walk the page tables in main memory.

**Lizard bypasses standard OS executable loaders entirely.** 

Lizard compiles the Abstract Syntax Tree (AST) directly into raw memory. To achieve a TLB footprint of exactly **one entry**, the execution environment aggressively negotiates with the host OS kernel for contiguous Large/Huge Page allocations before falling back to standard 4KB pages.

### OS-Specific Memory Negotiation:

*   **Linux (`memfd`, `/dev/shm`, & `hugetlbfs`):** 
    For strictly isolated processes, Lizard compiles directly into memory using `memfd_create` or anonymous `mmap`. For multi-process execution, zero-copy shared libraries, and bare-metal IPC, Lizard compiles directly into POSIX shared memory (`shm_open` via `/dev/shm`) or explicit `hugetlbfs` large-page filesystem mounts. It explicitly requests memory in the following degradation order:
    1. 1GB Huge Pages (`MAP_HUGETLB | MAP_HUGE_1GB` or via 1GB `hugetlbfs` mount)
    2. 2MB Huge Pages (`MAP_HUGETLB | MAP_HUGE_2MB` or via 2MB `hugetlbfs` mount)
    3. Standard 4KB Pages (Fallback via anonymous `mmap` or standard `/dev/shm`)

*   **Windows (`VirtualAlloc` API):** 
    Lizard utilizes `VirtualAlloc` or `VirtualAlloc2` with the `MEM_LARGE_PAGES` flag (requiring `SeLockMemoryPrivilege`). By allocating raw 2MB (or system-defined large page maximums) contiguous blocks, it completely bypasses the legacy PE (Portable Executable) loading overhead of the Windows memory manager. For multi-process sharing, it leverages `CreateFileMapping` backed by the system paging file.

*   **macOS / Apple Silicon (`MAP_JIT` & Superpages):** 
    To satisfy strict ARM64 W^X (Write XOR Execute) hardware protections on M-series chips, Lizard requests anonymous memory using `mmap` combined with `MAP_JIT` and `VM_FLAGS_SUPERPAGE_SIZE_2MB`. This allows the compiler to rapidly write the opcodes and instantly toggle the memory to an executable state without fracturing the TLB.

### Universal Architectural Claim (Prior Art)
The specific OS API is an implementation detail. The fundamental architectural prior art established here is **the intentional fusion of executable machine code (`.text`) and static/mutable data (`.data`) into the largest contiguous, hardware-supported memory pages available on the host system—whether memory-mapped privately or exposed globally via shared memory/IPC file systems—entirely bypassing traditional on-disk binary format loaders (ELF, PE, Mach-O) to intentionally eliminate TLB cache-miss latency during isolated or multi-process JIT execution.** 

This architecture applies to any current or future operating system, hypervisor, or bare-metal kernel implementing large, huge, or gigabyte-page contiguous memory mapping, both private and shared.

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
