# ADR 0003: Policy-Aware Execution Model and Local-First Invariance

- **Status:** Accepted
- **Deciders:** Core Architecture Group
- **Date:** 2026-11-13
- **PR:** #1

## Context and Problem Statement
Traditional distributed runtimes (e.g., Ray, Kubernetes schedulers) assume a trusted, high-bandwidth cluster where compute can be scheduled arbitrarily to balance resource utilization. In edge-to-cloud and multi-device consumer computing, data sovereignty, user privacy, energy budgets, and intermittent connectivity make arbitrary offloading unacceptable.

## Decision Outcome
We establish a **Policy-Aware Execution Model** where every task execution request carries declarative invariants:
1. `LocalOnly`: The task and its dependent data must never leave the originating node.
2. `LocalFirst`: Attempt local execution; if local resources are insufficient or degraded, fall back to permitted peers or edge nodes.
3. `EdgeFirst`: Prioritize nearby low-latency edge gateways; fall back to cloud or local.
4. `CloudFirst`: Offload heavy batch compute to centralized infrastructure while caching results locally.
5. `DynamicOptimal`: Let the scheduler compute the optimal placement based on latency, energy, cost, and bandwidth.

Security and privacy policies are absolute invariants: the scheduler can never override a `LocalOnly` constraint for performance gains.

