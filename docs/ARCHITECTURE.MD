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

## 2. Memory Topology: Thread-Local by Default & The `[[shared]]` Boundary

Modern languages attempt to be "smart" by using complex Escape Analysis to guess when a variable crosses a thread boundary, silently injecting hidden locks or promoting memory to the heap. This introduces "spooky action at a distance"—if a developer accidentally passes a reference to a background thread, the compiler silently wraps the entire data structure in atomic locks, inexplicably tanking pipeline throughput.

Lizard refuses to play guessing games with your memory topology. 

Lizard enforces a strict **Thread-Local by Default** architecture for all module-scoped variables. 

*   **Function Scope:** Variables declared inside a function execute exactly as expected—they are bound to the raw hardware stack & registers (`RSP`).
*   **Module/Global Scope:** Variables declared at the file or module level (which in Python or C would default to unprotected global state) are inherently compiled as **Thread-Local Storage (TLS)** (syntactically equivalent to C++11 `thread_local`). 

Every thread gets its own perfectly isolated, lock-free instance of the global state. They execute with zero locks, zero atomic instructions, and absolute bare-metal ALU speed. 

### The `[[shared]]` Attribute
To orchestrate multi-threaded data, developers must explicitly cross the concurrency boundary using the `[[shared]]` attribute. 

When a module-level variable or array is tagged with `[[shared]]`, the JIT compiler allocates the payload into the globally visible `mmap` / HugePage execution block. *Only then* does the compiler apply the hardware synchronization geometry dictated by your Optimization Flags (e.g., `-L1` implicit Seqlocks, or `-L2` hardware `F0` locks). 

**The Pragmatic Advantage:**
1. **Zero Spooky Slowdowns:** 99% of your state logic runs at lock-free, single-thread physics. The compiler never silently penalizes your throughput.
2. **Explicit Intent:** The developer acknowledges and accepts the physical cost of a memory barrier because they explicitly requested it. 
3. **Trivial Debugging:** If a developer asks, *"Why isn't my global array updating across threads?"*, the answer is always, *"Because you mutated your thread-local copy. Tag it `[[shared]]`."*

### Code Example: Explicit Concurrency
This syntax provides the frictionless joy of Python while natively driving hardware-level memory boundaries.

```python
struct secret_agent:
    name: string
    codename: string
    license: string
    nemesis: string
    mission: string

# Allocated in globally visible shared memory.
# The JIT automatically wraps this array in the requested -L synchronization.
[[shared]] secret_agents = []

# Module-level variable. In Lizard, this is inherently Thread-Local.
# Every spawned thread gets its own isolated, lock-free copy of this counter.
total_agents_dispatched: int32 = 0 

secret_agents.add(bond_template)
secret_agents.add(alec_template)

thread MoneyPenny:
    # Safely mutates the shared memory state across the thread boundary
    secret_agents.name = "Pierce Brosnan"
    
    # Mutates MoneyPenny's isolated TLS counter (Executes in 1 clock cycle)
    total_agents_dispatched += 1

thread AgentM:
    # Safely reads and mutates the shared state
    secret_agents.mission = "kill 006"
    
    # Mutates AgentM's isolated TLS counter. Does NOT affect MoneyPenny's count.
    total_agents_dispatched += 1

def execute_mission():
    # Stack-local variable. Bound to the RSP register.
    local_kill_count: int32 = 0 
```

## 3. First-Class Threads & OS Telemetry

In legacy languages, concurrency is treated as an afterthought—relegated to standard library imports (`std::thread`, `import threading`) where OS-level thread management requires verbose, platform-specific boilerplate.

Lizard elevates the thread to a **First-Class AST Node**. 

By defining threads natively within the syntax, the compiler guarantees deterministic execution lifecycles and provides unparalleled bare-metal telemetry out of the box.

### Mandatory Naming & POSIX Telemetry
When a developer defines a thread in Lizard, they are forced to name it. The compiler takes this AST identifier and automatically binds it to the underlying OS thread name (e.g., via `pthread_setname_np` on Linux, which enforces a 15-character physical limit. If a thread name exceeds the limit lizard will sigh at you (issue a warning) and truncate the thread name. It will issue an error if two threads truncate to the same name.). 

If your pipeline crashes and you open `gdb`, `htop`, or a Linux core dump, you will not see 50 anonymous processes. You will see exactly which thread faulted directly in the OS telemetry.

### The Execution Object
The declared thread immediately becomes a first-class execution object in the parent scope, exposing native lifecycle and hardware-affinity methods without requiring external OS libraries.

**Example: Native Threading & Core Pinning**
```python
# The compiler inherently recognizes this as a thread entry point.
# It automatically maps the AST identifier to the OS thread name.
thread FeedHandler_CTA:
    while running:
        parse_market_data()

def system_boot():
    # The thread exists as a manageable object in the parent scope.
    
    # 1. Hardware Affinity: Pin the thread directly to CPU Core 2
    # Bypasses the need for complex `taskset` or `numactl` bash scripts.
    FeedHandler_CTA.pin_core(2)
    
    # 2. Execution Lifecycle
    FeedHandler_CTA.start()
    
    # ... runtime logic ...
    
    FeedHandler_CTA.join()
```

### A. Structural Geometry: The `-L` Hardware Synchronization Matrix

When a developer crosses the concurrency boundary using the `[[shared]]` attribute, Lizard maps optimization flags (`-L`) directly to physical hardware cache-line synchronization. This allows developers to dynamically trade strict data integrity for raw throughput or domain-specific access patterns.

*   **`-L1` (Default) — Strict Consistency (The Implicit Seqlock):** 
    Lizard automatically alters the physical geometry of `[[shared]]` objects. The compiler silently prepends a 64-bit Sequence Counter to the memory layout, enforcing strict L1 cache-line alignment. 
    *   *Writers* automatically wrap mutations by incrementing the sequence counter. 
    *   *Readers* automatically emit a spin-read loop to verify the sequence counter has not changed mid-read. 
    *   *Best for:* High-Frequency Trading (HFT), deterministic state machines, and structs < 128 bytes where tear-free reads are mathematically mandatory.

*   **`-L2` — Eventual Consistency (The Atomic Wall):** 
    Strips all sequence counters and brute-forces the AMD Infinity Fabric. The compiler injects the x86 `F0` (`LOCK`) prefix exclusively on the exact primitive being mutated, completely ignoring multi-field struct boundaries. 
    *   *Trade-off:* Accepts torn reads across complex structs in exchange for continuous, non-blocking asynchronous throughput. 
    *   *Best for:* Biotechnology (e.g., massive multi-megabyte genomic structs where locking the entire object for a single scalar mutation is impossible), stochastic simulations, and macro-probabilistic modeling.

*   **`-L3` — Bare-Metal Anarchy:** 
    Zero locks. Zero sequence counters. Zero `F0` prefixes. The compiler emits pure, raw x86_64 arithmetic instructions directly against the `[[shared]]` memory block. 
    *   *Trade-off:* Guarantees catastrophic data races and visual tearing if not managed manually by the developer. 
    *   *Best for:* Single-Producer/Single-Consumer (SPSC) lock-free ring buffers where the developer mathematically guarantees thread safety via their own custom padding and atomic memory fences.

### B. Hardware Topology & Semantic Scheduling

The OS scheduler's logical core numbering (e.g., Linux mapping Core 0 and Core 16 as hyperthread siblings) is a notorious trap. Accidentally manually pinning a 100% CPU spin-polling loop to the same physical silicon as a UDP network feed handler will result in catastrophic packet latency.

Lizard solves this by providing native hardware topology introspection and Semantic Thread Scheduling.

#### Topology Introspection
Lizard abstracts OS-specific CPU topology (e.g., Linux `/proc/cpuinfo`, Windows `GetLogicalProcessorInformation`) into native, array-based hardware helpers:
*   `CPU.get_physical_cores()`: Returns an array of all true physical silicon core IDs, bypassing OS logical fragmentation.
*   `CPU.get_virtual_cores()`: Returns an array of all secondary SMT/hyperthread logical core IDs.
*   `CPU.get_core_pairs()`: Returns a nested array mapping physical cores to their associated virtual threads (e.g., `[[0,1], [2,3]]` on Windows, or `[[0,16], [1,17]]` on Linux). This is mathematically future-proofed; if future silicon architectures introduce tri-core SMT or beyond, the arrays will dynamically scale to reflect the true physical clusters (e.g., `[[0,1,2], [3,4,5]]`). This allows systems engineers to intentionally pair high-bandwidth Producer/Consumer threads onto the exact same physical silicon to achieve instantaneous L1/L2 cache sharing without crossing the Infinity Fabric.
*   `CPU.get_smt_sibling(id)`: A bidirectional mapping function. Pass a physical core ID to get its corresponding SMT sibling, or pass a virtual ID to resolve its physical host core.

#### Semantic Thread Tags
For developers who want optimized CPU placement without the risk of manually hardcoding core collisions, Lizard provides semantic scheduling tags. The Lizard runtime evaluates these tags at `.start()` to optimally distribute threads across the physical silicon topology.

*   **`[[heavy]]` (The ALU Monopolizer):** 
    Designed for math-intensive operations, 100% active spin-polling, or complex ML inference. The scheduler aggressively hunts for an unoccupied physical core. If the environment permits, it will auto-pin the thread to guarantee uninterrupted hardware access and maximum L1/L2 cache ownership.
*   **`[[medium]]` (The Bursty IO):** 
    Designed for network feed handlers and event-driven architectures. Frequently sleeps waiting for IO (`epoll`, `IOCP`), but requires high compute capacity when awakened. The scheduler prefers open physical cores, but will safely map them to SMT siblings of other `[[medium]]` or `[[light]]` threads if silicon is constrained.
*   **`[[light]]` (The Background Worker):** 
    Designed for telemetry, logging, and asynchronous state-saving. The scheduler actively prioritizes dumping these threads onto SMT/hyperthread siblings to intentionally keep the primary physical cores completely uncontended for `[[heavy]]` workloads.

**Example: Semantic Scheduling**
```python
# Lizard ensures the logger runs on an SMT sibling, 
# keeping physical cores free for the execution engine.
[[light]]
thread TelemetryLogger:
    flush_logs()

# Lizard automatically isolates this thread on a dedicated physical core.
[[heavy]]
thread AlphaExecutionEngine:
    while running:
        calculate_vwap_and_fire()
```

### C. Execution Mechanics & The Affinity Sandbox

Lizard rejects the opaque, cache-destroying multiplexing of goroutines and async/await thread pools. Lizard threads map 1:1 with OS threads, but their placement is governed by Semantic Tags and the developer's declared **Affinity Sandbox**.

By default, calling `thread.start()` with semantic tags allows the Lizard runtime to orchestrate placement across the entire host CPU. However, in enterprise environments where Lizard coexists with legacy C++ infrastructure or dedicated NIC interrupt cores, developers can restrict Lizard's physical domain.

#### 1. The Global Affinity Pool
Developers can define exactly which cores the Lizard runtime is allowed to touch. Once defined, all semantic tags (`[[heavy]]`, `[[light]]`) are evaluated strictly within this restricted silicon sandbox.

```python
# Reserve cores 0-3 for a legacy C++ engine.
# Hand cores 4-15 (and their SMT siblings) exclusively to Lizard.
available_phys = CPU.get_physical_cores()[4:15]
available_virt = CPU.get_virtual_cores()[4:15]

# Lock the runtime sandbox. 
# Lizard will now only schedule threads across this specific subset.
CPU.set_affinity_pool(available_phys + available_virt)

[[heavy]]
thread AlphaEngine:
    do_math()

# The Lizard runtime sees [[heavy]] and auto-pins it to Core 4, 
# mathematically guaranteeing it never touches Cores 0-3.
AlphaEngine.start() 
```

#### 2. The Absolute Override (`.start(core_id)`)
Semantic tags are a quality-of-life feature for macro-orchestration. When absolute deterministic geometry is required, the developer can bypass the tag heuristics and assert physical dominance by passing a core ID directly to the start method.

```python
thread DirectNetworkFeed:
    parse_udp()

# Bypasses all runtime heuristics and semantic tags.
# Violently pins the thread to physical Core 2 before the instruction pointer jumps.
DirectNetworkFeed.start(core_id=2)
```

#### 3. Custom Thread Managers
Because threads are First-Class AST nodes and hardware introspection (`CPU.get_core_pairs()`) is native, developers are not forced to use Lizard's default scheduler. Quantitative engineering teams can effortlessly write their own domain-specific `ThreadManager` classes in pure Lizard syntax to handle hyper-specific SPSC (Single-Producer/Single-Consumer) core pairing or dynamic load balancing.

#### 4. OS-Level Inheritance (Respecting `taskset` and `cgroups`)
Lizard does not suffer from the arrogant "God Complex" of legacy market data providers. 

If a systems engineer launches a Lizard binary via a strict OS-level mask (e.g., `taskset -c 4-15 lizard bot.liz` on Linux, or via Docker `cgroups`), the Lizard runtime inherently respects that boundary at boot. 

The hardware introspection APIs (`CPU.get_physical_cores()`) and the semantic scheduler will automatically truncate their available topology maps to match the OS-enforced affinity mask. You will never have to write an `LD_PRELOAD` hack to intercept rogue `pthread_setaffinity_np` calls just to stop the runtime from breaking out of its designated hardware partition.


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

Developers can command the JIT to physically group or banish opcodes using qualitative tags, or take absolute mathematical control over the binary layout using explicit sequencing. 

By applying thermal tags to control flow (`if`, `while`) or functions, developers command the JIT to physically group or banish opcodes in memory:
1.  **`[[layout(n)]]` (Absolute Physical Authority):** This tag forces the compiler to arrange the physical opcodes sequentially based on the `n` integer value, exactly mirroring memory offsets. `[[layout(0)]]` commands the compiler to place the function at the absolute top of the executable page (the entry point; maximum L1i cache priority). Priority decreases as `n` increases. Layout tags strictly override all qualitative thermal tags. 
2.  **`[[hot]]` (Alias: `[[likely]]`):** Absolute L1i cache priority. Mapped internally as a low-integer `layout(n)` offset. 
3.  **`[[warm]]`:** Executed frequently. Kept physically adjacent to the hot loops to minimize TLB strain.
4.  ***(Untagged)*:** Standard compiler heuristic ordering.
5.  **`[[chill]]`:** Low-frequency execution. Segregated to prevent L1i cache pollution.
6.  **`[[cold]]` (Alias: `[[unlikely]]`):** "Cold Storage." Mapped internally to `[[layout(inf)]]`. Appended to the absolute farthest, highest memory address of the 1GB page. 

*(Note: The compiler will emit a warning (a deep sigh) if a developer writes contradictory physical layouts, such as tagging a function `[[cold]] [[layout(0)]]`, before ultimately obeying the explicit integer `layout` command).*

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
    If the compiler sees a reference crosses a thread boundary ([[shared]] tag), but is only *read/mutated* and never dynamically rebound to a new address, it remains a raw pointer. However, the compiler automatically injects the structural geometry dictated by your `-L` flags (e.g., `-L1` implicit `seqlock`, or `-L2` `F0` hardware locks) to guarantee thread-safe mutations.
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

### D. Deterministic Collections (`[]` Arrays)
Lizard provides the syntactic frictionless joy of Python lists (`[]`), but completely rejects Python's underlying architecture (which scatters pointers randomly across the heap). 

When a developer initializes an array `quote_list = []`, the compiler intrinsically treats it as a dynamically resizing, **contiguous memory arena** (similar to a highly optimized C++ `std::vector`, but tightly integrated into the JIT).

**The Allocation Physics:**
*   **Contiguous by Default:** Objects appended to the list (`quote_list.push(quote)`) are physically packed adjacent to each other in memory, guaranteeing perfect L1/L2 cache prefetching during iteration. 
*   **Hidden `malloc`, Explicit Control:** The compiler automatically handles the underlying `mmap` / `malloc` arena expansion. However, to prevent the latency spikes associated with dynamic reallocation (e.g., the array doubling in size and copying memory mid-loop), developers can explicitly pre-allocate the hardware arena using a native method.

**Example: Zero-Allocation Hot Pathing**
```python
# The compiler inherently knows this is an array of contiguous StockQuotes.
quote_list = []

# Developer reads the required size from an IO source
quotes_to_allocate: int64 = file.read_size()

# EXPLICIT CONTROL: The developer pre-allocates the exact memory block required.
# This mathematically guarantees that the subsequent loop will execute 
# exactly zero `malloc` or `mmap` system calls.
quote_list.reserve(quotes_to_allocate)

[[unroll(4)]]
for i in range(quotes_to_allocate):
    # Pure O(1) memory pointer math. Zero allocation overhead.
    quote_list.push(parse_quote())
```


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

### D. Strict Typology & Ergonomic Casting
Implicit type coercion (e.g., silently casting a `float` to an `int`, or an `int32` to an `int16`) is a systemic source of precision-loss bugs and hardware faults in legacy systems. By default, Lizard strictly bans implicit type casting. The compiler demands mathematical exactness. 

However, because Lizard is built for *modern developer happiness*, it rejects the verbose, unreadable bloat of C++ casting (`static_cast<int16>(a)`). Lizard implements a highly ergonomic, multi-tiered casting architecture:

1.  **Explicit C-Style Casting:** Developers use clean, bracketed prefixes to enforce a cast. The compiler statically verifies that the cast is mathematically valid (preventing unsafe reinterpret-casts) before emitting the hardware conversion instructions.
    *   *Standard:* `(int16)a`
    *   *Scoped (for complex expressions):* `(int16)(a + b)`
2.  **The "Figure It Out" Cast:** If the destination type is mathematically obvious from the assignment or function signature, the developer can simply write `(cast)`. The compiler infers the exact required bit-width and generates the promotion/truncation opcodes.
    *   *Example:* `my_int64 = (cast)my_int32`
3.  **Statement-Level `[[implicit_cast]]`:** For single equations where hardware up-casting is guaranteed to be safe, tagging the statement temporarily relaxes strict-typing for that exact line. Lizard syntax allows the tag to be placed at the beginning or the end of the statement.
4.  **Block-Level `[[implicit_cast]]`:** When writing dense mathematical logic, tagging an entire `{}` block instructs the AST to silently auto-promote types across all equations within that physical boundary.

**Example: Ergonomic Type Coercion**
```python
def calculate_metrics(raw_size: int16, multiplier: int32, offset: int64):
    
    # 1. Explicit C-Style Cast
    expanded_size: int64 = (int64)raw_size
    
    # 2. The Auto-Infer Cast (Compiler knows destination is int64)
    expanded_size = (cast)raw_size 
    
    # 3. Statement-Level Implicit Casting (Prefix or Postfix)
    # The int16 and int32 are silently promoted to match the int64 destination.
    [[implicit_cast]] total_value: int64 = raw_size * multiplier + offset
    
    adjusted_value: int64 = raw_size * multiplier [[implicit_cast]]

    # 4. Block-Level Implicit Casting
    # All equations within the block automatically handle cross-width math.
    [[implicit_cast]]
    {
        volume: int64 = raw_size * multiplier + offset
        surface_area: int64 = raw_size * raw_size * multiplier
    }
```


## 12. Zero-Cost Static Reflection

Modern high-level languages (Java, C#, Python) achieve reflection by polluting the runtime environment with bloated metadata dictionaries and executing slow, dynamic lookups. Conversely, legacy systems languages (C, C++) completely abandon reflection to maintain memory purity, forcing developers to rely on horrific macro preprocessors or external code generators to serialize structs.

Lizard introduces **Zero-Cost Static Reflection** by exposing the compiler's Abstract Syntax Tree (AST) directly to the execution syntax.

### A. The `_members` Iterator
Because Lizard executes a Universal Recompilation Directive, the JIT compiler possesses perfect, real-time knowledge of every data structure's geometry. 

Developers can natively iterate over the structural metadata of any object using the `_members` keyword. 

**Example: Automated Serialization (Pythonic Syntax)**
```python
def serialize_struct(data: ptr<StockQuote>):
    # The compiler intercepts this loop at JIT-time
    for member in data._members:
        # The compiler statically unrolls this logic, emitting 
        # sequential hardware instructions for each field.
        network_buffer.write(member.value)
```
Example: Automated Serialization (C++23 Style Syntax)
```
void serialize_struct(ptr<StockQuote> data) {
    // Exact same AST evaluation, mapped to C++ style range-based iteration
    for (auto member : data._members) {
        network_buffer.write(member.value);
    }
}
```

### B. Compile-Time Loop Unrolling
When the JIT compiler encounters a reflection iteration (`for member in object._members`), it does **not** generate a runtime `while` loop. It does not allocate reflection metadata into the executable binary.

Instead, the compiler treats the loop as an AST generation macro. It statically evaluates the loop during compilation, physically unrolling the logic and emitting raw, sequential hardware instructions for every matching field. 

This provides the ultimate dynamic ergonomic power of Python's `getattr()` reflection, while mathematically guaranteeing the exact zero-cost execution latency of a manually unrolled C struct.

## 13. Unrestricted Compile-Time Execution (`constexpr`)

In legacy languages like C++, the `constexpr` (compile-time evaluation) keyword is structurally restricted. Developers cannot perform system I/O, make network requests, or read environmental states during compilation because the compiler is generating an artifact (`.exe`) intended to run on a different machine at a later date. This limitation frequently forces high-frequency developers to push mathematically static calculations (like calculating the UTC-to-EST epoch delta) into the runtime hot path.

Because Lizard enforces a Universal Recompilation Directive (JIT compiling directly on the host silicon at execution time), **the execution environment and the compilation environment are exactly the same.** 

Lizard fundamentally unshackles `constexpr`. 

### A. The Universal `constexpr` Mandate
In Lizard, *any* function, from *any* library, can be invoked as `constexpr`. 

When the JIT compiler encounters a `constexpr` variable assignment, it pauses AST traversal, literally executes the requested function natively in the compiler's own memory space, captures the returned payload, and permanently burns that result into the `memfd` instruction stream as a raw, static hardware constant. 

### B. Arbitrary Compile-Time I/O
The Lizard compiler does not sandbox `constexpr` evaluation. 
*   **System Telemetry:** You can execute a syscall to fetch the exact `RDTSC` (Time Stamp Counter) CPU frequency at the microsecond of compilation and bake it in as a constant multiplier, eliminating runtime cycle division.
*   **File I/O:** You can read a 500MB JSON configuration file into memory during compilation, extract the required thresholds, and bake them into the binary, entirely eliminating the JSON parsing library from the final executable.
*   **Network Requests:** If a developer wants to execute an HTTP `GET` request to a vendor API to fetch the daily trading limits and burn them into the binary as immutable static constants, the compiler will execute the network call and lock the value.

**Example: Unrestricted Compile-Time Physics**
```python
# The JIT compiler executes this function during compilation.
# The network request occurs BEFORE the binary is finalized.
def fetch_daily_strike_limit() -> int64:
    response = http.get("https://api.exchange.com/limits")
    return response.read_int()

# The result is burned directly into the x86 instruction stream 
# as a raw literal constant. Zero runtime overhead.
constexpr DAILY_LIMIT = fetch_daily_strike_limit()

def process_trade(size: int64):
    # Compiles to a raw hardware comparison: CMP RDI, [Constant]
    if size > DAILY_LIMIT: [[unlikely]]
        gtfo reject_trade
```

This absolute freedom allows systems engineers to strip massive amounts of initialization logic, environmental polling, and parsing overhead out of their runtime architecture, mathematically ensuring that the hot path only executes pure alpha.

### C. AST Introspection & Native Compile-Time Linting

In legacy ecosystems, enforcing architectural standards requires third-party static analyzers (Clang-Tidy, ESLint, Ruff). These external linters are bolted-on bureaucratic nightmares. They require active participation to maintain, rely on complex configuration files, break during toolchain updates, and frequently suffer from false-positive troubleshooting loops. 

Lizard fundamentally believes that outsourcing code-auditing to a third-party tool is a failure of the compiler. The compiler's job is to assist the engineer, and that includes giving them the power to mathematically enforce their organization's specific architectural laws.

Because Lizard's `constexpr` phase has unrestricted access to the internal environment, it exposes the compiler's own Abstract Syntax Tree (AST) metadata to the developer. You can write native `constexpr` functions to iterate over your codebase, inspect metadata tags, and execute custom `static_assert` logic before the binary is ever emitted. 

**Example: The Native HFT Linter**
```python
# This function executes entirely during the compilation phase.
[[constexpr]]
def audit_hft_pipeline():
    # Iterate over all functions registered in the project's AST
    for func in _project.functions:
        
        # Enforce organizational rules: 
        # "If a quant marks a function as hot, it MUST be branchless."
        if "hot" in func._tags and "branchless" not in func._tags:
            throw CompileError(
                f"Architecture Violation: Function '{func.name}' is tagged "
                f"[[hot]] but is missing the [[branchless]] mandate. "
                f"Fix your control flow."
            )

# Execute the audit before generating the binary
audit_hft_pipeline()
```

By exposing AST tags to `constexpr` evaluation, developers can build their own unbreakable CI/CD gatekeepers directly into the source code using the exact same syntax they use to write their trading algorithms. Zero external dependencies. Zero YAML files. Complete architectural authority.

## 14. Deterministic Optimization Passes
Lizard fundamentally rejects "heuristic optimizations" (where the compiler guesses branch probabilities or silently rearranges struct geometry). However, Lizard aggressively embraces **Deterministic Optimizations**—mathematical passes that provably reduce hardware execution cycles without altering the developer’s explicit structural intent. 

These passes exist to ensure developer happiness. Developers are encouraged to write rapid, iterative logic without worrying about the raw x86 register penalty of reusing a variable. 

### A. The Static Single Assignment (SSA) Matrix
Immediately following AST generation, Lizard transforms the logical tree into a strict Static Single Assignment (SSA) intermediate representation. This is the core physics engine of the compiler, responsible for mathematical proofing:
*   **Register Allocation Graph Coloring:** The SSA pass perfectly collapses temporary variables, iterative reassignments, and local aliases, mapping the final mathematically necessary values directly to raw x86 hardware registers (`RAX`, `RDI`, `XMM0`).
*   **Escape Analysis Validation:** The SSA graph provides the absolute mathematical proof of a memory pointer's lifespan, which the compiler uses to seamlessly enforce `-L2` hardware locking or `gtfo` Delta Destruction.

### B. Constant Folding & Propagation
Lizard performs aggressive, multi-pass constant folding at compile-time. 
If an expression evaluates strictly to constants (e.g., `(1024 * 1024) / 4`), the compiler executes the math internally and emits the final literal directly into the `memfd` instruction stream. This eliminates unnecessary ALU (Arithmetic Logic Unit) cycles for mathematically static values.

### C. Dead-Code Elimination (DCE)
Because Lizard enforces a Universal Recompilation Directive across the entire dependency graph, it has perfect visibility into which functions, templates, and code paths are actually invoked. 
Any function, branch, or structural logic that the SSA graph proves cannot be physically reached during execution is mathematically purged before opcode emission. This prevents the L1 instruction cache from being polluted by unused dependencies or dormant FFI templates.

The JIT compiler optimizes your logic, but it never overrides your physical commands. 

### D. Explicit Hardware Mandates (`[[unroll]]`, `[[vectorize]]`)
Legacy C++ compilers (like MSVC) treat `#pragma unroll` as a polite suggestion, often refusing to unroll loops based on opaque, internal cache-heuristic equations. This forces systems engineers into the miserable practice of manually copy-pasting loop logic 50 times in the source code.

Lizard treats optimization tags as absolute physical mandates. If the compiler cannot execute the mandate exactly as requested, it will generate the closest physical equivalent, emit a compiler warning (a sigh), and proceed. 

*   **`[[unroll(n)]]`:** Instructs the AST to physically unroll the loop exactly `n` times. If the developer explicitly requests `[[unroll(50)]]`, the compiler will emit 50 sequential, branchless blocks of instructions into the L1 cache. 
    *   **The Pragmatic Fallback:** If the JIT's `constexpr` folding or SSA analysis mathematically proves the loop will only ever execute 35 times, it will not break the build. It will intelligently truncate the unroll to exactly 35, emit a compiler warning (a sigh) about the developer's math, and proceed with the optimized emission. 

*   **`[[vectorize]]` (Explicit Opt-In SIMD):** By default, Lizard assumes standard scalar execution to guarantee predictable memory controller behavior and prevent SIMD-starvation from Gather/Scatter cache misses. Auto-vectorization is disabled. Applying `[[vectorize]]` is a strict command instructing the JIT to compile the loop using the host CPU's widest available SIMD extensions (e.g., AVX-512). If the geometry cannot be vectorized, the compiler falls back to scalar, sighs, and proceeds.

*   **`[[prefetch(lines=3, locality=1)]]`:** Modern CPUs attempt to guess memory access patterns. In complex, non-linear models or high-velocity lock-free queues, the hardware prefetcher fails, resulting in catastrophic cache misses. This tag allows the developer to explicitly emit a software prefetch instruction (e.g., x86 `PREFETCHT0`, `PREFETCHT2`). 
    *   `lines`: Dictates exactly how many 64-byte hardware cache lines ahead the compiler should instruct the memory controller to fetch data. *Defaults to 3 cache lines (192 bytes), the empirical mathematical sweet-spot to cover DDR5 RAM retrieval latency before execution.*
    *   `locality`: Exposes explicit control over the cache hierarchy.
        *   `1` = Hint to L1 cache (Data needed immediately, highest priority). *Default.*
        *   `2` = Hint to L2 cache (Data needed soon).
        *   `3` = Hint to L3 cache (Data needed in the future. Prevents L1 pollution in ultra-high-throughput SPMC queues).

**Example: The 35M/s Ring Buffer Hot Path**
```python
# The compiler verifies the logic can be executed purely via CMOV.
[[branchless]]
def process_queue_node(current_index: int64) -> int64:
    # Explicitly prefetch data 8 cache lines ahead, but restrict it to 
    # the L3 cache. This guarantees the data is off the main RAM bus 
    # without violently evicting the immediate working set from L1.
    [[prefetch(8, 3)]]
    future_node: int64 = ring_buffer[current_index + 8]
    
    return process_node(ring_buffer[current_index])
```

*   **`[[branchless]]`:** A strict architectural mandate. Instructs the compiler to mathematically eliminate conditional branches (`Jcc` instructions) within a block, replacing them entirely with bitwise arithmetic and conditional moves (e.g., x86 `CMOV`). If the logic fundamentally cannot be resolved without a jump, the compiler emits a fatal compile-time error. This guarantees a code block is mathematically immune to branch misprediction latency.

**Example: Explicit Loop Mandates**
```python
# The compiler will attempt to AVX-512 vectorize this loop.
# It will physically write the instructions 64 times in a row,
# completely eliminating the loop counter and branch prediction penalty.
[[vectorize]]
[[unroll(64)]]
for unit in dead_units:
    unit.render_flag = 0
```

**Example: The HFT Hot Path**
```python
# The compiler verifies the logic can be executed via CMOV and bitwise math.
# If a hidden branch is required, the build violently fails. 
[[branchless]]
def process_bbo_update(bid: int32, ask: int32) -> int32:
    # A software prefetch instruction is injected here, grabbing the 
    # next array element 64-bytes ahead directly into L1.
    [[prefetch(64)]]
    next_price: int32 = lookup_table[bid]
    
    # Executed purely via CMOV. Zero branch prediction penalty.
    return ask if bid > ask else bid
```

*   **`[[<arch>:<register>]]` (Register Allocation Mandate):** The ultimate expression of the "Not a Nanny" philosophy. Developers can bypass the compiler's SSA Register Graph Coloring algorithm and manually bind a variable to a specific physical hardware transistor. 
    *   Unlike dropping into raw `asm` blocks (which blinds the compiler and forces stack-spilling), this tag keeps the variable fully visible within the AST. The compiler simply hardwires the allocation and maps the rest of the surrounding logic around it. 
    *   If the requested register is mathematically impossible to allocate (e.g., it is required by the System V ABI for a function call occurring in the same scope), the compiler will emit a fatal error.

**Example: Physical Transistor Binding**
```python
def hyper_optimized_math(b: int32, c: int32) -> int32:
    # Forces the result directly into the x86 EAX accumulator register,
    # or the PowerPC r0 register, depending on the compilation target.
    # The AST completely bypasses dynamic graph-coloring for this variable.
    [[x86:eax]] [[powerpc:r0]] 
    a: int32 = b + c
    
    return a
```



### Why this is the absolute peak of the language:
You can throw away `CustomIntrinsics.hpp` because the compiler *is* the intrinsics library now. 

You no longer have to write unreadable `_mm512_loadu_si512()` Intel intrinsic spaghetti code just to get the CPU to do what you want. You write clean, Pythonic math, slap `[[vectorize]]` or `[[branchless]]` on top of it, and the Lizard JIT physically manipulates the silicon to match your will. 

## E. Hardware Status Flag Introspection
Standard high-level languages abstract away the CPU's internal condition codes (`Carry`, `Overflow`, `Zero`, `Sign`), forcing developers to write slow, redundant mathematical bounds-checking. The physical Arithmetic Logic Unit (ALU) already calculates and stores this exact state for free during the instruction.

Lizard provides native, zero-cost introspection of the physical ALU via compiler-injected, **architecture-specific intrinsic namespaces**. 

Lizard intentionally refuses to provide a "generic" cross-platform flag wrapper (e.g., `CPU.overflow`). Different architectures handle mathematical states differently (e.g., PowerPC's `OV` vs. `SO` Summary Overflow). Providing a generic alias would force the compiler to guess the developer's intent. Instead, developers must explicitly query the hardware they are targeting.

*   **Explicit Silicon Queries:** Developers read the physical flags using raw architecture namespaces: `CPU.x86.flags.OF`, `CPU.arm.flags.V`, `CPU.ppc.flags.SO`. 
*   **The Volatility Capture:** Because the physical flags register is overwritten by almost every mathematical operation, developers must capture the flag state immediately after the target operation into a standard boolean register.

**Example: Zero-Cost x86 Overflow Detection**
```python
def secure_math_x86(b: int64, c: int64) -> int64:
    # 1. The hardware executes the ADD instruction. 
    # The CPU natively updates its internal EFLAGS register for free.
    a: int64 = b + c
    
    # 2. We instantly capture the physical silicon state before it is overwritten.
    # The compiler emits SETO (Set if Overflow) and SETC (Set if Carry).
    # If this script is executed on an ARM processor, the JIT emits a fatal error.
    did_overflow: bool = CPU.x86.flags.OF
    did_carry: bool = CPU.x86.flags.CF

    if did_overflow or did_carry: [[unlikely]]
        gtfo abort_math
        
    return a
**Cross-Platform Support:**
If a developer requires a math function to run across multiple architectures, they do not rely on generic flag aliases. They utilize Lizard's compile-time evaluation to conditionally emit the correct physical block:

```python
def secure_math(b: int64, c: int64) -> int64:
    a: int64 = b + c
    
    # The JIT compiler statically strips the dead paths at compile-time.
    [[constexpr_if]] CPU.arch == "x86_64":
        if CPU.x86.flags.OF: gtfo abort_math
    [[constexpr_elif]] CPU.arch == "aarch64":
        if CPU.arm.flags.V: gtfo abort_math
    [[constexpr_else]]:
        # Fallback to slow software math for unknown architectures
        if (b > 0 and c > int64.max - b): gtfo abort_math

    return a

    # The targeted unwind destination, safely scoped within the function.
    label abort_math:
        log_hardware_fault("Catastrophic integer overflow detected.")
        return 0
```

### F. Absolute Memory Addressing (MMIO & GPIOs)
To fully replace C in embedded systems, custom robotics, and FPGA programming, a language must allow developers to intentionally read and write to raw, hardcoded physical memory addresses (Memory-Mapped I/O). 

Lizard achieves this without requiring unsafe `reinterpret_cast` logic or external C wrappers. Developers can instantiate a hardware pointer by directly casting an integer literal to a typed pointer, combining it with the `[[volatile]]` and `[[writethrough]]` tags defined in Section 8. 

**Example: Embedded GPIO Pin Toggling**
```python
# The developer manually defines the hardware registry address for the GPIO bus.
constexpr GPIO_BASE: int64 = 0x3F200000

def toggle_hardware_pin():
    # Statically cast the integer address to a volatile 32-bit hardware pointer.
    # The compiler guarantees this memory read/write will never be cached or optimized away.
    [[volatile]]
    gpio_register: ptr[uint32] = (ptr[uint32])GPIO_BASE
    
    # Flip the 5th bit to send a physical electrical signal to the hardware pin.
```

### Why this is the perfect closer:
It proves the compiler scales all the way down. 
*   **HFT Quants:** They use `[[prefetch(3,3)]]` and `StockQuote._members` reflection.
*   **AI Researchers:** They use `float4` and `[[vectorize]]`.
*   **Embedded Hackers (Barry):** They use `ptr[uint32] = (ptr[uint32])0x3F200000` to literally flip electricity on and off. 

## 15. The Bare-Metal Escape Hatch: Allocators & FFI

Lizard’s primary directive is to provide deterministic, zero-overhead memory physics natively within the language. However, to truly serve as a foundational orchestration layer for enterprise systems, Lizard must permit power users to entirely override the compiler's memory manager and interface cleanly with legacy ecosystems. 

### A. Custom Memory Allocators (The Arena Override)
Standard OS-level memory allocators (`malloc`, `mmap`) often introduce global kernel spinlocks, creating catastrophic thread-contention in high-frequency architectures. 

Lizard allows developers to explicitly override the underlying memory manager for specific types, arrays, or entire architectural blocks by binding them to custom allocator functions (e.g., Thread-Local Slab Allocators, `jemalloc`, or `tcmalloc`).

**Example: The Custom Arena**
```python
# The developer manually defines a bare-metal memory allocator interface
[[c_abi]] def custom_slab_allocate(size: int64) -> ptr[uint8]
[[c_abi]] def custom_slab_free(p: ptr[uint8])

# The struct is physically bound to the custom allocator.
# When the JIT instantiates this object, it bypasses the OS 
# and directly invokes `custom_slab_allocate`.
[[allocator(custom_slab_allocate, custom_slab_free)]]
struct HighFrequencyNode:
    data: int64
    next: ptr[HighFrequencyNode]
```

### B. The C/C++ Foreign Function Interface (FFI)
Lizard operates on a Universal Recompilation Directive for its native code, but it acknowledges the reality of the legacy world. Developers can seamlessly invoke pre-compiled C/C++ shared libraries (`.so`, `.dll`) with mathematically zero overhead. 

*   **Zero-Cost Context:** Because Lizard natively generates x86_64 machine code, crossing the FFI boundary does not require stack-switching, Pythonic GIL management, or intermediate JNI wrappers. The compiler simply loads the exact hardware registers specified by the System V AMD64 ABI (or Windows x64 ABI) and executes a raw `CALL` instruction. 
*   **Struct Parity:** Because Lizard structs default to strict, literal C-style memory layouts (Section 6A), developers can pass memory pointers (`ptr[T]`) directly into C++ libraries without requiring costly serialization or layout translation.

### C. Language Interoperability (e.g., Python `nanobind`)
Lizard is designed to be the hyper-fast execution core that sits underneath higher-level analysis languages. Because Lizard's structs are perfectly aligned POD (Plain Old Data) and lack Object-Oriented `vptr` bloat, they mathematically perfectly align with 1-copy integration frameworks like C++ `nanobind` or `pybind11`. 

A 64-byte `StockQuote` struct generated by Lizard can be passed instantly across a memory boundary into a Python data-science environment without requiring any translation layer, allowing analysts to interact with bare-metal data streams inside a Jupyter Notebook.

## 16. Enterprise Distribution & Pre-Compilation

Lizard’s foundational architecture is built entirely upon the **Universal Recompilation Directive** (JIT compiling directly on the host silicon to guarantee perfect ABI alignment and microarchitecture targeting). 

However, we acknowledge the reality of enterprise software distribution. Not every company wants to distribute their proprietary algorithms in raw plaintext source code, and not every end-user wants to install the Lizard toolchain to execute a program. To support enterprise deployment without resorting to the architectural anti-pattern of code obfuscation, Lizard provides two explicit distribution compilation targets.

### A. The Cryptographic JIT Payload (`liz build --secure`)
For organizations that want to maintain the absolute performance benefits of Just-In-Time microarchitecture targeting while protecting their intellectual property, Lizard offers the Secure Payload target.

*   The compiler bundles the entire dependency graph (source code) and encrypts it using standard AES cryptography.
*   It generates a standalone, self-extracting executable (`.exe` / ELF binary). 
*   **Execution:** When run on the target machine, the executable prompts for the AES decryption key (via `stdin`, environment variable, or secure keystore). The payload decrypts directly into memory, executes the standard Lizard JIT compiler against the host silicon, and jumps the instruction pointer. The source code never touches the target machine's physical hard drive.

### B. Raw Ahead-Of-Time (AOT) Emission (`liz build --aot`)
For situations where a JIT engine is strictly prohibited (e.g., Apple iOS, game console environments, or heavily restricted corporate servers), Lizard provides a raw Ahead-Of-Time compilation escape hatch.

*   The developer explicitly defines a target architecture (e.g., `--target=zen4` or `--target=x86_64_generic`).
*   The compiler executes its standard AST and SSA passes, but instead of writing opcodes to a `memfd` execution block, it emits a standard binary artifact (`.exe`, `.so`, `.dll`). 
*   **The Pragmatic Warning:** By executing an AOT build, the developer intentionally forfeits Lizard's dynamic microarchitecture targeting and `constexpr` runtime I/O capabilities. The code will execute as a standard, static binary. Lizard maintainers take absolutely zero responsibility for ABI mismatches or generic-silicon latency penalties incurred by using this legacy distribution model. Lizard is a JIT compiler first; AOT is provided strictly as a pragmatic fallback.



Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.
