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


## 6. Attribute-Driven Micro-Optimization

While Lizard prioritizes high-throughput defaults, it recognizes that bare-metal programming requires explicit, surgical control over memory geometry and CPU pipeline behavior. To accommodate this within its Scope-Dominant Hybrid Lexer, Lizard introduces **C++-style Compiler Attributes (`[[...]]`) directly into Pythonic syntax.**

This allows developers to explicitly guide the JIT compiler's code generation without relying on implicit, potentially destructive optimizations.

### A. Deterministic Memory Layout (`[[packed]]`, `[[no_reorder]]`)
When defining data structures for network protocols, FFI (Foreign Function Interface) boundaries, or strict hardware payloads, developers must guarantee exact byte placement.
*   By default, Lizard analyzes structs for optimal cache-line packing.
*   Applying `[[no_reorder]]` strictly enforces the developer's declared field order.
*   Applying `[[packed]]` annihilates all compiler-injected padding bytes, forcing contiguous field alignment at the cost of potential unaligned memory access penalties.

### B. Hardware Alignment (`[[alignas(N)]]`)
Cache-line bouncing and false sharing are the absolute killers of multi-threaded throughput. Standard high-level languages abstract memory allocation, leaving alignment to the whims of the runtime's memory allocator (usually 8-byte or 16-byte boundaries). 

Lizard introduces the parameterized `[[alignas(N)]]` attribute directly into its object definition syntax. This forces the compiler and the underlying `mmap` allocator to guarantee that the starting memory address of the struct, variable, or buffer is perfectly divisible by `N`. 

This is structurally mandatory for:
*   **Preventing False Sharing:** Forcing independent thread variables onto separate 64-byte L1 cache lines.
*   **SIMD Vectorization:** Guaranteeing 32-byte or 64-byte boundaries for AVX2 and AVX-512 vector math loads.
*   **Lock-Free Queues:** Ensuring that SPMC/MPMC ring buffer nodes do not straddle cache-line boundaries (preventing hardware tearing).

### C. Branch Predictor Guidance (`[[likely]]`, `[[unlikely]]`)
Modern CPUs utilize highly advanced branch predictors (e.g., AMD's TAGE). In high-frequency paths, developers often possess empirical knowledge about the probability of a specific branch (e.g., an error check or an uncontended lock) that the compiler cannot infer statically.
*   Injecting `[[likely]]` or `[[unlikely]]` directly onto control flow statements (like `if` or `while`) instructs the Lizard compiler to emit optimal assembly layouts. It forces the unlikely path into a "cold" instruction section, preventing it from polluting the L1 instruction cache (L1i).

### Code Example: Pythonic Syntax with Hardware Attributes

```python
# The JIT compiler guarantees this object will always 
# be instantiated on a strict 64-byte L1 cache-line boundary,
# and will never inject padding bytes between the fields.
[[alignas(64)]]
[[packed]]
struct StockQuote:
    bid: decimal32
    ask: decimal32
    timestamp: int64

def exampleConcurrentLizardFunction():
    do_stuff();
    
    # Instructs the JIT to optimize the instruction cache 
    # assuming this lock is almost always uncontended.
    if sharedData.not_contested: [[likely]]
    {
        # The {} block overrides the whitespace parser.
        # The lock is deterministically acquired and released 
        # strictly within this physical boundary.
        ScopeGuard guard(sharedData.lock);
        print(f"suck it python");
    }

Lizard's C++ style syntax parser:

```

[[alignas(64)]]
[[packed]]
struct StockQuote {
    decimal32 bid;
    decimal32 ask;
    int64 timestamp;
};

void exampleConcurrentLizardFunction() {
    do_stuff();
    
    if (sharedData.not_contested) [[likely]] {
        // Standard C++ RAII scoping works exactly as expected.
        ScopeGuard guard(sharedData.lock);
        print(f"suck it c++");
    }
}

```


## 7. Strict Typology & The Thermal Execution Matrix

Lizard aggressively rejects legacy C/C++ type ambiguity (where the bit-width of an `int` or `long` is dependent on the target architecture or compiler implementation) while simultaneously supporting Python's arbitrary-precision integers. 

### A. Eradicating Type Ambiguity
In Lizard, memory geometry is mathematically explicit. 
Numeric types followed by a bit-width map directly to raw hardware registers. Numeric types without a bit-width dynamically allocate arbitrary-precision (BigInt) storage, trading bare-metal speed for mathematical boundlessness.

*   **Integers:** `int8`, `int16`, `int32`, `int64` (Hardware). `int` (Unlimited/Heap).
*   **Decimals (Fixed-Point):** `decimal4`, `decimal8`, `decimal16`, `decimal32`, `decimal64` (Hardware). `decimal` (Unlimited/Heap).
*   **Floating-Point:** `float4`, `float8`, `float16`, `float32`, `float64` (IEEE 754 Hardware). `float` (Unlimited/Heap). *(Note: `float64` completely replaces the legacy `double` keyword).*

### B. The Thermal Instruction Matrix (L1i Cache Control)
Modern CPUs execute instructions optimally when the hot path remains pinned in the L1 instruction cache (L1i). Lizard introduces a **Thermal Tagging System** to explicitly control the physical layout of opcodes within the compiled `memfd` Huge Page.

By applying thermal tags to control flow (`if`, `while`) or functions, developers command the JIT to physically group or banish opcodes in memory:

1.  **`[[hot]]` (Alias: `[[likely]]`):** Absolute L1i cache priority. The compiler enforces that these opcodes are physically contiguous with the primary entry point. Fall-through branch prediction is hardwired here.
2.  **`[[warm]]`:** Executed frequently, but not critical to the micro-second hot path. The compiler attempts to keep it on the same physical memory page as the hot loop, minimizing TLB strain, but will spill it adjacent if `[[hot]]` saturates the cache.
3.  ***(Untagged)*:** Standard compiler heuristic ordering.
4.  **`[[chill]]`:** Low-frequency execution (e.g., 1Hz telemetry loggers). The compiler physically segregates these instructions away from the hot loop to prevent L1i cache pollution.
5.  **`[[cold]]` (Alias: `[[unlikely]]`):** "Cold Storage." Reserved for catastrophic errors, fatal exceptions, or `tail-drop` panic routing. The compiler physically appends these instructions to the absolute farthest memory address of the 1GB page, ensuring they never load into the CPU prefetcher unless a disaster occurs.

### C. Advanced Hardware Control Tags (The Prior Art Arsenal)
To fully weaponize the compiler, Lizard supports the following bare-metal attributes:

*   **`[[force_inline]]` / `[[no_inline]]`:** Overrides compiler heuristics to forcibly inline (or prevent inlining) of a function, dictating exact instruction cache footprint.
*   **`[[pure]]`:** Informs the compiler that the function has zero side effects and only depends on its inputs. Allows the JIT to aggressively memoize results or entirely eliminate redundant calls.
*   **`[[tailcall]]`:** Forces strict Tail-Call Optimization (TCO). If the compiler cannot mathematically guarantee the function will execute without consuming a new stack frame, it throws a fatal compile-time error. Essential for lock-free state machines.
*   **`[[naked]]`:** Instructs the JIT to emit zero function prologue or epilogue (no stack frame setup/teardown). The developer assumes absolute manual control over hardware registers. Designed for raw system-call wrappers, custom interrupt handlers, or hyper-optimized inline assembly blocks.


## 8. Memory Topography & Compile-Time Monomorphization

Lizard’s primary directive is to eliminate the cognitive overhead of manual concurrency (`std::atomic`, memory fences). The compiler’s Escape Analysis automatically injects hardware bus locks (`F0`) for CPU-bound thread collisions. 

However, when interfacing directly with external hardware (PCIe, FPGAs, NIC Ring Buffers, or VRAM), developers must bypass the CPU's internal caching mechanisms.

### A. Hardware IO Fencing Attributes
To manage Memory-Mapped I/O (MMIO) without resorting to raw assembly, Lizard provides explicit cache-bypass attributes:

*   **`[[writethrough]]` (Non-Temporal Store):** Instructs the compiler to emit non-temporal streaming instructions (e.g., x86 `MOVNTI`). The data is written directly to main memory or the PCIe bus, entirely bypassing the CPU's L1/L2/L3 cache hierarchy. Essential for writing to GPU framebuffers or NIC transmission queues without polluting the local CPU cache.
*   **`[[volatile]]`:** (Strictly redefined from C). In Lizard, `[[volatile]]` does not imply atomicity. It mathematically guarantees that the compiler will *never* optimize away, reorder, or cache the read/write operation. Every access triggers a physical load/store to the memory address. Used exclusively for reading hardware status registers.
*   **`[[barrier]]`:** Enforces a hard compiler and hardware memory fence (e.g., `MFENCE`). Guarantees that all memory operations initiated before the barrier are globally visible to the entire system (including external PCIe devices) before any subsequent instructions are executed.

### B. Universal Template Monomorphization
Dynamic typing and Object-Oriented Polymorphism (OOP `vptr` injection) destroy L1 cache alignment and prevent aggressive inlining. Lizard rejects runtime polymorphism in favor of strict, C++-style Compile-Time Monomorphization.

Lizard supports full template metaprogramming seamlessly across both its Brace-Dominant and Pythonic syntax. 

*   **Zero Runtime Overhead:** When a template is invoked with a specific type (e.g., `<StockMarketQuote>`), the JIT compiler generates a mathematically distinct, perfectly optimized assembly path for that specific memory layout.
*   **Syntax Integrity:** Lizard intentionally retains the classic `<T>` bracket syntax for templates to cleanly separate compile-time monomorphization from runtime array indexing `[]`.

**Example: Pythonic Template Syntax**
```python
# The <T> syntax imports classic C++ template power into Pythonic defs
def process_exchange_buffer<T>(buffer_ptr: ptr<T>, size: int32):
    
    # Force a strict hardware read from the external NIC register
    [[volatile]]
    nic_status: int32 = read_hardware_register()
    
    if nic_status == 1: [[hot]]
    {
        # Stream the buffer directly to main memory, 
        # bypassing the CPU L1 cache entirely.
        [[writethrough]]
        write_to_memory(buffer_ptr, size)
        
        # Enforce a global hardware fence before continuing
        [[barrier]]
        notify_pipeline_ready()
    }
```

Lizard's C++ style syntax parser:

```

// Exact same AST, parsed via brace-and-semicolon dominance
void process_exchange_buffer<T>(ptr<T> buffer_ptr, int32 size) {
    
    [[volatile]]
    int32 nic_status = read_hardware_register();
    
    if (nic_status == 1) [[hot]] {
        [[writethrough]]
        write_to_memory(buffer_ptr, size);
        
        [[barrier]]
        notify_pipeline_ready();
    }
}

```

### C. Data-Oriented Typology (The "Struct" Standard)
Lizard abandons the legacy Object-Oriented `class` keyword. In high-frequency computing, Data-Oriented Design (DOD) is paramount. Developers construct mathematically aligned data payloads (`struct`) and feed them through pure transformation functions, rather than building state-heavy objects. 

*   **Public by Default:** All fields in a Lizard `struct` are public by default, mirroring Python's frictionless data access model. 
*   **Explicit Privacy:** C++ developers can still enforce strict memory encapsulation by explicitly opening a `private:` block within the struct definition.

**Example: Templated Pythonic Struct**
```python
# A generic lock-free ring buffer node, forced onto a 64-byte boundary
[[alignas(64)]]
struct RingBufferNode<T>:
    sequence_id: int64
    payload: T
    
    private:
        # Internal state hidden from the consumer
        _commit_flag: int32

# Instantiating the template generates a mathematically unique memory layout
node: RingBufferNode<StockMarketQuote>

// Valid Lizard Code (C++23 Style)
[[alignas(64)]]
struct RingBufferNode<T> {
    int64 sequence_id;
    T payload;
    
private:
    int32 _commit_flag;
};
```


























Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.
