# Lizard Architecture 🦎

*This document serves as my architecture brainstorming establishing prior art.*

Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.

## 0. The Principle of Deterministic Performance

Lizard is a compiler, not a heuristic guessing engine. We fundamentally reject the paradigm where a compiler silently rearranges your memory geometry or ignores your caching instructions because an opaque, slow internal algorithm "knows better." 

If you want a massive static analyzer to profile your branch predictions and calculate optimal function inlining thresholds, you should build that as a separate, community-driven tool. That tool can then read your telemetry and inject explicit `[[force_inline]]` or `[[hot]]` tags directly into your Lizard source code. 

The compiler’s job is not to guess your intent; its job is to execute the tags you provide with absolute, unyielding hardware physics. Compilation must remain blistering fast, and performance must remain completely deterministic.

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

### C. Lexical Scope-Targeted Unwinding (`gtfo` / `go`)
While Lizard strictly bans dynamic, asynchronous stack unwinding (exceptions), it acknowledges that highly complex, multi-lock state machines require nonlinear control flow (e.g., jumping to a unified error block or restarting a hardware polling loop). Nested `if/else` logic for these states ruins branch prediction and pollutes the L1 instruction cache.

Lizard implements **Lexical Scope-Targeted Unwinding** via two explicit hardware jump commands: `go` and `gtfo` (Graceful Termination and Forward Operation).

*   **`go <label>`:** Emits a raw, unconditional x86 `JMP`. The compiler performs zero RAII analysis. It does not call destructors; it does not release locks. It is pure silicon violence for hyper-optimized state machines where the developer manually guarantees memory safety.
*   **`gtfo <label>` (Graceful Termination and Forward Operation):** Instructs the compiler to calculate the exact lexical delta between the call site and the destination label. The compiler statically determines which objects (locks, memory allocations) are alive at the `gtfo` call site but *out of scope* at the destination label. It physically injects only those specific destructors immediately before emitting the `JMP`.

**Constraints:** Both the jump call and the `label` must exist strictly within the same function boundary. Cross-function jumps are mathematically banned to preserve the System V ABI call stack.

### Code Example: Targeted Unwinding
This example demonstrates jumping backward to retry a corrupted hardware frame, and jumping forward to abort, perfectly managing RAII spinlocks at zero runtime cost.

```python
def process_trade_concurrently():
    do_stuff()

    if sharedData.not_contested: [[likely]]
    {
        ScopeGuard guard(sharedData.lock)
        
        # A jump destination label
        label corrupt_check:
        
        if packet.is_corrupted(): [[unlikely]]
        {
            ScopeGuard nic_guard(network_card.lock)
            
            # Attempt to fix the frame via raw hardware IO
            if network_card.repair_frame():
                # The compiler calculates the delta:
                # 1. nic_guard is out of scope at 'corrupt_check'.
                # 2. 'guard' is STILL in scope at 'corrupt_check'.
                # Action: Calls ~nic_guard, leaves 'guard' locked, and JMPs backward.
                gtfo corrupt_check 
        }

        if packet.is_fatal(): [[unlikely]]
            # The compiler calculates the delta:
            # 1. Both nic_guard (if alive) and 'guard' are out of scope at 'abort_sequence'.
            # Action: Calls all necessary destructors to release locks, then JMPs forward.
            gtfo abort_sequence 

        print("suck it python")
    }

    return true

    # The label is scoped strictly to the function frame.
    label abort_sequence:
        log_failure()
        return false
```

Lizard C++ like syntax example:

```
// Valid Lizard Code (C++23 Style Syntax)
bool process_trade_concurrently() {
    do_stuff();

    if (sharedData.not_contested) [[likely]] {
        ScopeGuard guard(sharedData.lock);
        
        corrupt_check: // Standard C-style label syntax supported
        
        if (packet.is_corrupted()) [[unlikely]] {
            ScopeGuard nic_guard(network_card.lock);
            
            if (network_card.repair_frame()) {
                // Destroys nic_guard, retains guard, JMPs backward.
                gtfo corrupt_check;
            }
        }

        if (packet.is_fatal()) [[unlikely]] {
            // Destroys all active RAII objects in the delta, JMPs forward.
            gtfo abort_sequence;
        }

        print("suck it c++");
    }

    return true;

abort_sequence:
    log_failure();
    return false;
}
```

As you can see Lizard solves the double memory leak problem.

### D. The Legacy Alias (`goto` -> `gtfo`)
Lizard natively supports C++ style `using` semantic aliasing to maintain compatibility and reduce friction for legacy systems engineers. 

To bridge the 40-year gap between unsafe C-style jumping and modern RAII memory safety, the standard Lizard environment implicitly defines the following alias:
`using goto = gtfo`

When a developer ports legacy C/C++ state machines into Lizard using the traditional `goto` keyword, the Lizard lexer intercepts it and automatically maps it to the `gtfo` AST node. The compiler silently upgrades the legacy unstructured jumps into mathematically safe, delta-calculated targeted unwinds. 

Lizard fixes legacy memory leaks behind the developer's back, without requiring them to alter a single line of their traditional control flow. It provides enterprise-friendly syntax wrapped around a hyper-violent, scope-aware execution engine.

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

### A. Deterministic Memory Layout (`[[packed]]`, `[[reorder]]`)
Lizard never silently reorders struct fields. The order declared in the source code is the exact mathematical geometry instantiated in RAM. When defining data structures for network protocols, FFI boundaries, or strict hardware payloads, developers must have absolute geometric authority without relying on `static_assert` paranoia.

*   **Default Behavior:** Struct fields are laid out exactly as written. The compiler injects standard hardware alignment padding between fields to prevent unaligned access penalties, but it will never change the field sequence. 
*   **`[[reorder]]`:** Explicitly commands the compiler to mathematically analyze the struct and automatically reorder the fields (typically largest-to-smallest) to perfectly pack them into the tightest possible L1 cache-line footprint, minimizing padding waste. This allows developers to group fields logically in the source code (e.g., `bid` next to `ask`) without paying a structural layout penalty.
*   **`[[packed]]`:** Annihilates all compiler-injected padding bytes, forcing contiguous byte-to-byte field alignment exactly as written. This is mandatory for raw network packet parsing, but introduces potential hardware penalties for unaligned memory access.

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
```


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

*   **`[[force_inline]]` (The Absolute Mandate):** In C/C++, `inline` is merely a suggestion that the compiler frequently ignores. In Lizard, `[[force_inline]]` is a mathematical mandate. It instructs the AST parser to physically copy and flatten the function's syntax nodes directly into the caller's scope *prior* to Label Resolution. 
    *   **Macros 3.0 & Zero-Cost Coroutines:** Because this attribute flattens the AST into a single System V ABI stack frame, developers can use `gtfo` and `go` to jump in and out of `[[force_inline]]` lambdas natively. This functionally creates zero-cost, type-safe hardware coroutines without the need for a macro preprocessor or a runtime scheduler. If the developer abuses this and blows out the L1i instruction cache, the compiler will happily let them.
*   **`[[no_inline]]`:** An absolute blockade. Prevents the JIT heuristic engine from ever inlining the function, dictating exact instruction cache isolation.
*   **`[[pure]]`:** Informs the compiler that the function has zero side effects and only depends on its inputs. Allows the JIT to aggressively memoize results or entirely eliminate redundant calls.
*   **`[[tailcall]]`:** Forces strict Tail-Call Optimization (TCO). If the compiler cannot mathematically guarantee the function will execute without consuming a new stack frame, it throws a fatal compile-time error.
*   **`[[naked]]`:** Instructs the JIT to emit zero function prologue or epilogue. The developer assumes absolute manual control over hardware registers (`RBP`/`RSP`). Designed for raw system-call wrappers or hyper-optimized inline assembly blocks.

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

### C. Data-Oriented Typology (The Eradication of Inheritance)
Lizard abandons the legacy Object-Oriented `class` keyword and explicitly bans classical Object-Oriented Inheritance. 

In high-frequency computing and modern systems architecture, Data-Oriented Design (DOD) is paramount. Deep inheritance trees (`Animal -> Dog -> Poodle`) and dynamic polymorphism physically destroy L1 cache alignment. When a developer relies on virtual functions, the compiler must silently inject an 8-byte virtual table pointer (`vptr`) into every struct, fracturing cache lines and forcing the CPU to execute a slow, branch-unpredictable memory lookup (dynamic dispatch) for every function call.

Lizard fundamentally rejects this architectural penalty. 

*   **Struct Composition Over Inheritance:** Developers construct mathematically aligned data payloads (`struct`) and use explicit Composition. If a struct needs the data of another struct, it physically embeds it as a field.
*   **Compile-Time Monomorphization:** If a developer needs a function to operate on multiple different types, they do not use a virtual base class. They use a Template (`<T>`). The Lizard compiler generates mathematically distinct, perfectly optimized assembly paths for each type at compile-time (Zero-Cost Static Dispatch).
*   **Public by Default:** All fields in a Lizard `struct` are public by default, mirroring Python's frictionless data access model. C++ developers can still enforce strict memory encapsulation by explicitly opening a `private:` block.

There is no `extends`, no `implements`, and no `vptr` injection. You pack the struct, and you transform the data.

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
```

Valid Lizard Code (C++23 Style):

```
// Valid Lizard Code (C++23 Style)
[[alignas(64)]]
struct RingBufferNode<T> {
    int64 sequence_id;
    T payload;
    
private:
    int32 _commit_flag;
};
```

## 9. The Universal Recompilation Directive (Zero ABI Breakage)

Traditional compiled languages (C, C++, Rust) rely heavily on pre-compiled shared libraries (`.so`, `.dll`, `.dylib`) and static archives (`.a`, `.lib`). This reliance introduces fragile Application Binary Interface (ABI) boundaries. If a pre-compiled library was built with a different struct padding, calling convention, or compiler version than the host application, passing a memory pointer across that boundary results in catastrophic, silent memory corruption.

**Lizard mathematically eliminates ABI breakage by refusing to participate in it.**

### A. Recursive Source-Level JIT Compilation
Lizard does not link against pre-compiled Lizard binaries. 

When you execute `lizard main.liz`, the compiler’s dependency resolver recursively crawls every `import` statement in the project tree. It ingests the raw source code of the entire dependency graph, flattens it into a single massive Abstract Syntax Tree (AST), and JIT-compiles the entire universe simultaneously. 

*   **Zero ABI Drift:** Because every dependency is compiled simultaneously by the exact same compiler instance, struct layouts, padding bytes, and calling conventions are mathematically guaranteed to perfectly align across all module boundaries.
*   **Global Optimization & Inlining:** Because the JIT has full visibility of the entire dependency graph's source code, it can aggressively inline functions across module boundaries—an optimization that standard linkers (without LTO) cannot achieve when crossing pre-compiled `.so` boundaries.

### B. Hardware-Specific Microarchitecture Targeting
Pre-compiled libraries are typically compiled against the lowest common denominator (e.g., standard `x86_64`) to maximize compatibility, abandoning massive performance gains available on modern silicon. 

Because Lizard inherently recompiles from source at execution time, it natively implements **Just-In-Time Microarchitecture Targeting**.
*   If the script is executed on an AMD Ryzen 9 9950X3D, the compiler automatically detects AVX-512, VPCLMULQDQ, and VAES extensions, and emits hyper-optimized vectorized instructions for the entire dependency graph.
*   The developer never has to manually specify `-march=native`. The execution environment inherently shapes the compiled geometry to perfectly match the physical silicon it is running on. 

### C. The Modular Syscall Architecture
Lizard does not ship with a bloated standard library attempting to encapsulate every OS networking or file system API. Baking abstractions like `std::net::socket` into the core compiler creates dogmatic bottlenecks and maintenance nightmares. 

Instead, Lizard provides native primitives for direct hardware-level system calls (e.g., executing the `SYSCALL` x86 instruction directly). 
*   **Decentralized Libraries:** Interfaces for Linux `epoll`, Windows `IOCP`, or `io_uring` are maintained as completely separate, community-driven or officially sponsored Lizard modules (e.g., managed via a `requirements.txt` style package graph). 
*   **Zero-Cost Kernel Boundaries:** Because Lizard generates raw opcodes, dropping into a kernel system call requires zero stack-switching overhead. The compiler loads the registers and executes the trap. 

The compiler remains mathematically pure; the ecosystem provides the adapters.

### D. Decentralized Dependency Management (Zero-Registry)
Lizard refuses to host or maintain a centralized package registry (e.g., `npm`, `crates.io`, `PyPI`). Centralized registries introduce supply-chain vulnerabilities, namespace squatting, and massive infrastructural maintenance burdens. 

Lizard implements **Direct Git-Link Resolution** via a brutally simple `lizard.req` manifest file. 

To import a module, the developer must explicitly define the namespace, the direct repository URL, and the immutable Git commit hash or release tag.

**Example `lizard.req` format:**
```text
# <namespace>  <git_repository_url>                     <commit_hash_or_tag>
sockets      github.com/lizard-lang/sockets           "v1.1.12"
dll-loader   github.com/lizard-lang/dll-loader        4630bb8f125146662d41aea1a10c5bbccc1bcc6f
math_lib     gitlab.com/quant-fund/lizard-math        "v2.0.1"
```

## 10. Memory Ergonomics: Rebindable References

To achieve the execution speed of C++ without a Garbage Collector, a language must expose memory addresses. However, legacy C++ memory ergonomics are fundamentally broken. C++ references (`T&`) suffer from strict initialization binding (they can never be reseated to point to a new object). To achieve dynamic reassignment, developers are forced to use raw pointers (`T*`), which introduces null-dereference anxiety and fractures syntax via the `->` operator. 

Lizard solves this by introducing **Rebindable References** as the default data-binding paradigm, perfectly mirroring Python's variable ergonomics on top of raw silicon. 

### A. The Mechanics of the Rebindable Reference
In Lizard, when you pass an object or assign a variable, it is handled as a Rebindable Reference.
*   **Syntax Fluidity:** Like Python, developers always use the dot operator (`object.field`) for memory access. The `->` pointer-dereference operator is entirely abolished from the high-level syntax.
*   **Dynamic Reseating:** Unlike C++ references, a Lizard reference can be dynamically reassigned to point to a different memory address mid-function without copying the underlying object payload. 
*   **Zero-Overhead:** Under the hood, the compiler evaluates a Rebindable Reference as a strict, non-null hardware pointer (`*`). There is zero reference-counting overhead and zero Garbage Collection pausing. It is simply a raw memory address with mathematically enforced syntactic sugar.

### B. Implicit Memory Escalation via Escape Analysis
In legacy systems languages, developers must manually decide whether a pointer is raw, uniquely owned (`std::unique_ptr`), or reference-counted (`std::shared_ptr`). If the architecture scales, the developer is forced to refactor thousands of types.

Lizard completely rejects manual pointer typing. The compiler’s Escape Analysis engine automatically handles memory lifecycle physics, allowing the developer to use simple, unified Rebindable References everywhere. 

The compiler mathematically proves the necessary memory abstraction at compile-time:

1.  **The Raw Pointer Path (Zero-Cost):**
    If the compiler proves a reference is strictly thread-local, or that its lifespan is guaranteed to be shorter than the object it points to (e.g., passing down the call stack), it compiles the reference as a raw, zero-overhead x86_64 pointer (`T*`).
2.  **The Cross-Thread Reference Path (Implicit Concurrency):**
    If the compiler proves a reference crosses a thread boundary, but is only *read/mutated* and never dynamically rebound to a new address, it remains a raw pointer. However, the compiler automatically injects the structural geometry dictated by your `-L` flags (e.g., `-L1` implicit `seqlock`, or `-L2` `F0` hardware locks) to guarantee thread-safe mutations.
3.  **The Ref-Count Escalation Path (Turing-Complete Survival):**
    If, and only if, the compiler detects that a reference crosses a thread boundary AND its lifespan cannot be statically determined (e.g., it is inserted into a global asynchronous hash map or dynamically rebound mid-flight), the compiler silently promotes the object to a heap allocation and attaches an implicitly locked reference counter (`LOCK XADD`). 

**The Developer Experience:**
The developer never types `rc_ptr` or `shared`. They simply declare a variable and use it. The Lizard JIT analyzes the entire dependency graph, guarantees the memory safety, and automatically applies the fastest physical abstraction the hardware allows.

### C. The Cyclic Reference Policy & `[[weak]]` Tags
Because Lizard fundamentally rejects tracing Garbage Collectors (mark-and-sweep), it is mathematically vulnerable to cyclic memory leaks when auto-escalated reference-counted objects point to each other (e.g., doubly-linked lists). 

Lizard enforces the philosophy that **Memory Topology is the Developer's Responsibility.**

The compiler will not pause your trading pipeline to walk memory graphs and break cyclic dependencies. If a developer engineers a cyclic graph using Rebindable References and allows it to escape the stack, it will leak memory until the system terminates. 

To safely construct bidirectional graphs without triggering cyclic memory leaks, Lizard provides a single, explicit memory tag:

*   **`[[weak]]`:** When applied to a reference definition, it explicitly instructs the compiler *not* to increment the target object's reference counter upon assignment. 
*   **Access Constraints:** Because a weak reference does not guarantee the survival of its target, developers must explicitly test its validity (e.g., `if node.parent:`) before accessing it. If a developer attempts to dereference a dead `[[weak]]` reference without checking, the compiler defaults to the standard hardware response: a zero-unwind segfault.

**Example: Bidirectional Graph Syntax**
```python
struct TreeNode:
    data: int64
    # Implicitly manages the child's lifespan (strong reference)
    left_child: TreeNode
    right_child: TreeNode
    
    # Breaks the cycle: does not increment the parent's ref-count
    [[weak]]
    parent: TreeNode 

def link_nodes(parent: TreeNode, child: TreeNode):
    parent.left_child = child
    child.parent = parent # Rebinds a weak reference safely

```

Lizard C++ style syntax example:

```
// Valid Lizard Code (C++23 Style Syntax)
struct TreeNode {
    int64 data;
    
    // Implicitly manages the child's lifespan (strong reference)
    TreeNode left_child;
    TreeNode right_child;
    
    // Breaks the cycle: does not increment the parent's ref-count
    [[weak]]
    TreeNode parent;
};

void link_nodes(TreeNode parent, TreeNode child) {
    parent.left_child = child;
    child.parent = parent; // Rebinds a weak reference safely
}

```

As you can see, Lizard solves the double ref-counting problem elegantly.

## 11. Bit Twiddling, Quantization, & Absolute Geometry

In low-level systems (HFT protocols, custom network framing) and modern AI inference engines, developers frequently require exact bit-width precision that does not align with standard 32 or 64-bit hardware boundaries. 

Lizard solves this natively at the syntax and AST level, explicitly supporting both high-frequency state packing and AI weight quantization.

### A. The `<Type>N` Arbitrary Precision Syntax
Lizard allows the developer to explicitly define numeric primitives of *any* arbitrary bit-width directly in the type signature (e.g., `int24`, `uint13`, `float4`, `dec7`). *(Note: `dec` is aliased to `decimal` via the standard library `using` directive).*

*Unlike legacy languages that must invent new keywords (`__int128`, `__m256i`) every time silicon manufacturers widen their hardware registers, Lizard's syntax is mathematically immune to the passage of time. If a future CPU introduces a native 1024-bit integer register, the developer simply types `int1024`, and the JIT maps it directly to the silicon without a single change to the language specification.*

*   **Integers (`intN`, `uintN`):** The compiler allocates the smallest hardware-addressable block and silently injects the necessary bit-masks (e.g., `& 0xFFFFFF` for `uint24`) on all operations.
*   **AI Quantization & Exotics (`float4`, `udec16`):** Lizard natively supports sub-byte and exotic numeric types. It introduces `ufloat` (Unsigned Float) and `udecimal` to support modern, sign-stripped AI weight packing. 
*   **Struct Compression:** This allows developers to easily pack highly specific data—such as compressing 4 distinct Option Greeks into a perfect 8-byte payload using `dec16`—without writing manual shift logic.

### B. Hardware Intrinsics vs. Emulation Fallback
Because Lizard is not a nanny, developers are free to define mathematically bizarre bit-widths (e.g., `float3`). 

If a user-defined numeric width directly maps to existing silicon hardware intrinsics (e.g., standard `fp8` AI formats, or native IEEE 754), the compiler executes them natively on the FPU or AVX registers at zero cost.

If a developer defines a bit-width that physical hardware does not natively support, the JIT compiler will not throw an error. It will automatically synthesize the exact mantissa/exponent logic via slow, multi-cycle software emulation. 

**The Edge Cases:**
If an engineer pushes the boundaries to absolute absurdity, the compiler implements rigid structural decay:
*   **3-Bit Float (`float3`):** Resolves as an implicit `ufloat` (no bits left for the sign).
*   **2-Bit Float (`float2`):** Devolves into raw state combinations (`0.0`, `0.1`, `1.0`, `1.1`).
*   **1-Bit Float (`float1`):** Devolves mathematically into an integer (`0` or `1`). You asked for it. 

### C. Contiguous Bitfield Auto-Packing
In Lizard, bitfields are treated as pure contiguous data streams across all numeric types. 

The compiler automatically applies the `[[packed]]` attribute to bitfield blocks, packing them sequentially into the nearest machine-addressable byte boundary, entirely ignoring legacy C/C++ base-type allocation rules. 

**Example: 9-Bit Contiguous Field**
```python
# The compiler analyzes the 9 requested bits and automatically 
# packs them into exactly 2 bytes (16 bits) of physical memory.
struct StatusHeader:
    a: uint : 1
    b: dec : 1
    c: uint : 1
    d: float : 1  # Compiles to 0.0 or 1.0 logic.
    e: uint : 1
    f: uint : 4
```

Example C++ style syntax:
```
// Valid Lizard Code (C++23 Style Syntax)
// The compiler guarantees the exact same contiguous 2-byte packing.
struct StatusHeader {
    uint a : 1;
    dec b : 1;
    uint c : 1;
    float d : 1; // Compiles to software-emulated 0.0 or 1.0.
    uint e : 1;
    uint f : 4;
};
```
**The Sub-Byte Hardware Boundary Rule:**
If a developer defines an independent sub-byte primitive (e.g., `float4` or `int7`) without using explicit bitfield syntax, the compiler prioritizes memory addressability over spatial compression. 

Because the smallest individually addressable unit of memory on x86_64 is 1 byte (8 bits), the compiler will allocate full, independent bytes for these primitives. This guarantees that developers can still extract the raw memory address pointer (`&`) of the field to pass across FFI boundaries.

**Example: Addressability vs. Wasted Space**
```python
# Because no bitfield syntax is used, the compiler allocates 1 physical 
# byte for each int7 primitive. This struct takes exactly 4 bytes in RAM,
# with 4 bits of completely wasted padding. 

# The compiler will not stop you from doing something mathematically obtuse, 
# but it will absolutely judge you for it. The Lizard sighs deeply. 
#(Expect a compiler warning).
struct dumbass:
    l: int7
    m: int7
    a: int7
    o: int7
```

If absolute spatial compression is required (e.g., for ML weights or network framing), the developer must explicitly invoke Contiguous Bitfield Auto-Packing (Section 11C), which mathematically strips the primitives of their independent memory addresses to achieve perfect sub-byte density.

Example: Perfect Spatial Compression

```
# The bitfield syntax explicitly destroys the memory addresses.
# The compiler perfectly packs all four 4-bit floats into exactly 2 bytes.
struct ai_data_packed:
    a: float : 4
    b: float : 4
    c: float : 4
    d: float : 4
```



Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.
