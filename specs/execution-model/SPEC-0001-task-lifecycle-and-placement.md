# SPEC-0001: Task Lifecycle and Policy-Aware Placement Specification

- **Specification ID:** `SPEC-0001`
- **Subsystem:** Execution Model & Scheduler
- **Status:** Draft / Active Review
- **Version:** `0.1.0`
- **Authors:** AI-Native Runtime Research Group
- **RFC Anchor:** [`rfcs/proposed/0001-declarative-task-graph.md`](../../rfcs/proposed/0001-declarative-task-graph.md)

---

## 1. Scope & Purpose

This normative specification establishes the formal state machine, execution lifecycles, and policy-driven placement mechanics for computational tasks within `ai-native-runtime`.

Every computational workload executed across local threads, peer nodes, edge gateways, or cloud environments MUST adhere to the invariants and transitions defined in this document.

The key words **"MUST"**, **"MUST NOT"**, **"REQUIRED"**, **"SHALL"**, **"SHALL NOT"**, **"SHOULD"**, and **"MAY"** in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

---

## 2. Task Lifecycle State Machine

A computational task $T$ progresses through well-defined lifecycle states:

```
                ┌──────────────┐
                │  Submitted   │
                └──────┬───────┘
                       │ (Schema & policy validation)
                       ▼
                ┌──────────────┐
                │   Analyzed   │
                └──────┬───────┘
                       │ (Scheduler cost optimization)
                       ▼
                ┌──────────────┐
       ┌───────►│    Placed    │◄──────┐ (Transient network retry)
       │        └──────┬───────┘       │
       │               │ (Dispatch to local executor or network peer)
       │               ▼
       │        ┌──────────────┐
       │        │  Executing   │
       │        └──────┬───────┘
       │               │
       ├───────────────┼───────────────┐
       │ (Execution    │ (Success)     │ (Non-recoverable failure)
       │  timeout)     ▼               ▼
       │        ┌──────────────┐ ┌──────────────┐
       └────────┤ Materialized │ │    Failed    │
                └──────┬───────┘ └──────────────┘
                       │ (CRDT state convergence & cleanup)
                       ▼
                ┌──────────────┐
                │  Completed   │
                └──────────────┘
```

### State Definitions

1. **`Submitted`**: The task descriptor has entered the local node's intake queue from application code or FFI.
2. **`Analyzed`**: The runtime has parsed required compute resources (memory, accelerators, inputs) and validated privacy policies against available trust domains.
3. **`Placed`**: The scheduler has assigned the task to a specific eligible target node $n^* \in \mathcal{N}_{\text{eligible}}$.
4. **`Executing`**: The task payload is actively running inside a local sandbox or on a remote peer worker.
5. **`Materialized`**: Computation has finished, producing zero-copy result buffers and pending state mutations.
6. **`Completed`**: Mutations have converged into the local-first state store and control return has been delivered to the caller.
7. **`Failed`**: Terminal state reached when execution violates policy, exceeds max retry budget, or encounters unrecoverable panics.

---

## 3. Placement Invariants & Policies

Every task $T$ MUST carry a `PlacementPolicy` attribute:

```rust
#[repr(u8)]
pub enum PlacementPolicy {
    /// Task MUST execute locally on the originating device.
    /// Network egress of input data or weights is strictly forbidden.
    LocalOnly = 0,

    /// Task SHOULD execute locally if resources permit.
    /// If local compute is degraded, may offload to authorized peer/edge nodes.
    LocalFirst = 1,

    /// Task SHOULD execute on low-latency edge gateways (e.g., local server/LAN).
    /// Falls back to local or cloud if edge is unavailable.
    EdgeFirst = 2,

    /// Task is compute-heavy and non-private; offload to cloud clusters by default.
    CloudFirst = 3,

    /// Scheduler dynamically computes argmin of latency, energy, and cost.
    DynamicOptimal = 4,
}
```

### Invariant I-1: Privacy Preservation
If $\text{Policy}(T) = \texttt{LocalOnly}$, the scheduler **MUST NOT** consider any candidate node $n \neq n_{\text{local}}$, regardless of local resource constraints or latency penalties. If local execution is impossible (e.g. out of memory), the task **MUST** transition immediately to `Failed` with reason `ResourceExhaustionLocalOnly`.

### Invariant I-2: Local-First Offline Guarantees
When a node is disconnected from all external networks (air-gapped or offline), any task specifying `LocalFirst` or `DynamicOptimal` **MUST** execute locally rather than queuing indefinitely for connectivity.

---

## 4. Cost Optimization Function

When a task permits offloading ($\text{Policy}(T) \neq \texttt{LocalOnly}$), the scheduler determines target node $n^*$ via:

$$\mathcal{P}^*(T, \mathcal{N}) = \arg\min_{n \in \mathcal{N}_{\text{eligible}}} \Big[ \alpha \cdot C_{\text{latency}}(T, n) + \beta \cdot C_{\text{energy}}(T, n) + \gamma \cdot C_{\text{cost}}(T, n) \Big]$$

Where:
- $\alpha, \beta, \gamma \ge 0$ are normalized weighting coefficients defined in application manifests.
- $C_{\text{latency}}(T, n) = T_{\text{serialization}} + T_{\text{network\_rtt}} + T_{\text{remote\_exec}} + T_{\text{deserialization}}$.
- Candidate set $\mathcal{N}_{\text{eligible}} = \{ n \in \mathcal{N} \mid \text{Policy}(T) \subseteq \text{Trust}(n) \land \text{Caps}(n) \ge \text{Reqs}(T) \}$.

---

## 5. Fault Recovery & Network Partition Semantics

If node $n^*$ disconnects during state `Executing`:
1. If the task is idempotent, the scheduler **SHALL** reschedule it on an alternate candidate node or locally.
2. If the task produced partial mutations, those mutations **MUST** remain uncommitted until verified by CRDT vector clocks.
3. If no candidate nodes remain reachable, the task **MUST** fall back to local execution if permitted by policy.
