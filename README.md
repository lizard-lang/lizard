# Lizard 🦎
**The Cold-Blooded Concurrency Compiler.**

Lizard is a zero-dependency, bare-metal x86_64 JIT compiler designed for high-performance computing, quantitative finance, and data science. It combines the elegant readability of Python with the raw execution speed of silicon.

There is no Virtual Machine. There is no Garbage Collector. There is no Global Interpreter Lock (GIL). 

Lizard lexes your script, compiles it directly into an `mmap` buffer, and jumps the instruction pointer. 

## ⚡ Core Features

* **Speed:** 10,000x faster than Python. Designed to natively saturate the L1 cache.
* **Flexible Syntax:** Pythonic off-side rule (whitespace) and C-style braces `{}` are supported natively and interchangeably. The compiler only cares about execution, not religious formatting wars.
* **Zero Exceptions:** Lizard uses cold-blooded error handling. If a thread encounters a fatal memory fault, it does not unwind the stack or print a polite traceback. It physically severs its own call stack (`XCHG EAX, ESP` 🔥) and terminates instantly to prevent cascading latency failures across your pipeline.

## 🧬 Fearless Concurrency Model

Software locks (`std::mutex`) and interpreter locks (the GIL) artificially bottleneck modern multi-core processors. Lizard takes a different approach: **Aggressive Hardware Bus Locking.**

Instead of relying on the OS scheduler, Lizard's compiler intelligently injects the x86 `LOCK` prefix (`F0`) directly at the instruction level for global memory mutations. You do not need to import threading libraries or write lock-free queues. Lizard forces the processor's Infinity Fabric and memory controller to natively resolve your concurrent mutations at 5.7 GHz. 

### Data Integrity: High-Velocity Eventual Consistency
By relaxing strict, blocking temporal object-boundary synchronization, Lizard achieves unparalleled throughput. In high-frequency environments, stalling a thread to perfectly synchronize a complex struct is an anti-pattern. 

Lizard embraces an **Eventual Consistency** model at the cache-line level. Data structures converge naturally over microscopic time horizons without ever halting the execution pipeline. This makes Lizard the mathematically optimal choice for stochastic simulations, probabilistic modeling, and high-velocity data streams where continuous execution speed is prioritized over atomic struct rigidity.

### ⚙️ Compiler Optimization Flags
Lizard maps optimization flags (`-L`) directly to physical hardware cache-line synchronization, allowing developers to dynamically trade strict data integrity for raw throughput.

*   **`-L1` (Default) — Strict Consistency:** The compiler silently injects an implicit `seqlock` (Sequence Lock) and hidden `uint64_t` counters into all struct/object boundaries. Guarantees 100% tear-free reads and writes across complex data structures without ever blocking the writer thread. 
*   **`-L2` — Eventual Consistency:** Strips all sequence counters and brute-forces the AMD Infinity Fabric. Injects the `F0` (`LOCK`) prefix on every single primitive memory mutation. Accepts torn reads across complex structs in exchange for massive throughput gains. Designed for stochastic simulations where macro-statistical speed outweighs micro-tick precision.
*   **`-L3` — Bare-Metal Anarchy:** Zero locks. Zero sequence counters. Zero `F0` prefixes. The compiler emits pure, raw x86_64 arithmetic instructions. Guarantees catastrophic data races on shared memory. Executes at the absolute physical speed limit of the silicon.

## 📊 Benchmarks

* **Python (GIL):** ~10k ops/sec
* **Standard C++ (Software Mutex):** ~5M ops/sec
* **Lizard (Bare-Metal JIT):** **50,000,000+ ops/sec**

## 🚀 Quick Start

```bash
# Execute a script instantly. No build steps. No linking.
lizard alpha.liz
```

## ⚠️ Warnings & Disclaimers

* **Thermal Constraints:** Lizard was designed to bypass software-level locking overhead by communicating directly with the L1 cache. Executing stochastic simulations or infinite loops under the `-L2` or `-L3` optimization flags will cause modern processors to pull maximum amperage. Ensure your VRMs are adequately cooled. The author is not responsible for catastrophic thermal events, melted silicon, or degraded PCIe bus lifespan.
* **Hardware Errata:** If your system becomes unstable or crashes when executing high-throughput concurrent `Lizard` scripts, please verify your hardware grounding. We highly recommend checking your motherboard for EMI interference or static buildup on unused HDMI/DisplayPort connections. 
* **Data Integrity (Reiterated):** Lizard prioritizes raw execution speed over strict object-boundary synchronization. Do not use `-L2` or `-L3` flags for applications requiring perfect temporal atomicity (e.g., banking ledgers). By using this compiler, you accept that your structs may experience High-Velocity Eventual Consistency (torn reads).

Copyright: Adderalin 
* **Support:** There is no enterprise support. If your pipeline fails, drop your tail (`XCHG EAX, ESP` 🔥) and write faster code.

