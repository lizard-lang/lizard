# Contributing to the Lizard Ecosystem 🦎

*Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.*

We welcome contributions from systems engineers, compiler architects, and cypherpunks. However, because this ecosystem is built on strict architectural purity and uncompromising latency constraints, all contributions must adhere to the following rules:

### 1. Do Not Waste Cycles
If your Pull Request introduces a background thread, a garbage collection sweep, an unnecessary memory allocation on the hot path, or an asynchronous stack unwind, it will be closed immediately. 

### 2. Prove the Physics
If you submit an optimization or a feature, you must prove its performance impact. PRs modifying compiler output must include objective, hardware-level telemetry (e.g., branch prediction hit rates, cache-miss deltas, or cycle counts on modern silicon). "It looks cleaner" is not a valid reason to merge code.

### 3. Read the Manifesto
Before opening an issue or a PR, you must read the `ARCHITECTURE.md` and `PHILOSOPHY.md`. If you open an issue requesting standard C++ exception handling, dynamic polymorphism, or cloud-synced telemetry for Crucible, you will be directed to the philosophy documents and the issue will be closed.

### 4. The `lizard.req` Boundary
Do not submit massive libraries (JSON parsers, HTTP clients, cryptography wrappers) to the core compiler repository. The core remains mathematically pure. Build your library in a separate repository and publish the Git hash for the community to ingest via the `lizard.req` manifest.

If you understand the physics, we welcome your code.
