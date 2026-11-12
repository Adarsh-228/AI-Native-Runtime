# ADR 0002: Core Runtime Language Selection

- **Status:** Accepted
- **Deciders:** Core Architecture Group
- **Date:** 2026-11-13
- **PR:** #1

## Context and Problem Statement
The runtime core requires high throughput, deterministic memory behavior, low-latency execution, zero garbage collection pauses, rich asynchronous I/O primitives, cross-platform portability (mobile, desktop, embedded edge, cloud servers), and seamless foreign function interface (FFI) bindings for high-level language SDKs (Dart, TypeScript, Python).

## Considered Options
1. **C++ (C++20/23):** Ubiquitous in systems and inference runtimes, but lacks modern memory safety guarantees without heavy sanitizers.
2. **Go:** Excellent concurrency and networking model, but runtime garbage collection pauses and C-FFI overhead make it suboptimal for embedded on-device runtimes and mobile embedding.
3. **Rust:** Memory safety without garbage collection, modern concurrency models, zero-cost abstractions, first-class WebAssembly target support, and seamless C-FFI generation for Dart, Python, and Node.js.

## Decision Outcome
Chosen option: **Rust** for the foundational runtime core (`crates/`), while exposing native bindings to higher-level application languages (Dart/Flutter, TypeScript, Python).

