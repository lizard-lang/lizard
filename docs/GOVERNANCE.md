# Lizard Ecosystem Governance

*This document outlines the governance model for the Lizard compiler, Crucible, and the Lizard on Tails framework.*

Copyright (c) 2026 Adderalin & lizard-lang under MIT LICENSE. SEE LICENSE file.

## 1. The BDFL Model (Benevolent Dictator for Life)
The Lizard ecosystem currently operates under a strict BDFL governance model. 
The project was created to enforce specific, uncompromising architectural physics (bare-metal execution, explicit memory geometry, and cognitive privacy). To ensure these core philosophies are never diluted by design-by-committee compromise, **Adderalin** holds ultimate executive authority over the architecture, direction, and inclusion of all features across the ecosystem.

## 2. Core Tenets of Contribution
We welcome and encourage community contributions, bug fixes, and hardware optimizations, provided they adhere to the following non-negotiable tenets:

1. **Latency is King:** Pull requests that improve developer ergonomics but introduce branching penalties, background thread scheduling, or non-deterministic latency spikes will be rejected.
2. **Zero Bloat:** Proposals to add massive abstractions (e.g., HTTP clients, JSON parsers) to the core compiler will be rejected. Build it as a decentralized library. 
3. **No Security Theater:** Proposals to add native source-code obfuscation to the compiler will be rejected.
4. **Transparent Telemetry:** Any PR to Crucible that attempts to introduce unauthorized network calls, "anonymous" analytics, or cloud syncing will be rejected and the contributor banned. 

## 3. The Path to a 501(c)(3) Foundation
While currently operating under the BDFL model to ensure architectural purity during the foundational phase, the long-term intent is to transition the `lizard-lang` organization into a registered 501(c)(3) non-profit foundation. 

Once the core architecture reaches stable maturity, governance will expand to a **Core Steering Committee** comprised of lead maintainers, with the mandate to protect the ecosystem's sovereignty, fund bare-metal research, and ensure the tooling remains forever free of corporate capture.

## 4. Forking (The Ultimate Freedom)
If the community fundamentally disagrees with the BDFL's architectural decisions, they possess the ultimate open-source right: **The Fork**. 

The code is licensed under the MIT License. If you believe the compiler needs a Garbage Collector, a Global Interpreter Lock, or a massive standard library, you are free to fork the repository and build it. We just ask that you change the name. 
