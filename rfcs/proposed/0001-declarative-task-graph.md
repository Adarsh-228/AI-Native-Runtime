# RFC 0001: Declarative Task Graph and Zero-Copy Task Descriptor Protocol

- **RFC ID:** `RFC-0001`
- **Author(s):** AI-Native Runtime Research Group
- **Status:** Proposed
- **Created:** 2026-11-13
- **Target Subsystem:** Runtime / Scheduler / Networking

---

## 1. Summary

This RFC proposes the format and execution protocol for the **Declarative Task Graph (DTG)**, the foundational unit of computation within `ai-native-runtime`. A Task Graph is a directed acyclic graph (DAG) of task descriptors where nodes specify atomic computational units (e.g., model inference, state mutations, data transformations) and edges represent data dependencies and policy invariants.

The wire protocol is formalized using zero-copy schema buffers (`rkyv` / FlatBuffers), eliminating serialization penalties across IPC and local network transports.

---

## 2. Motivation

Traditional application frameworks rely on imperative, deeply nested callback chains or client-server RPCs (REST, gRPC) to coordinate multi-step workflows involving AI models and database queries. This introduces two major problems:
1. **Opaque Scheduling:** The runtime cannot inspect future task requirements, preventing intelligent pre-fetching or co-locating dependent tasks near their data.
2. **Serialization Tax:** In high-frequency agent loops, repeatedly serializing and deserializing JSON or Protobuf payloads between UI threads, inference engines, and remote servers incurs up to 40% latency overhead.

---

## 3. Detailed Technical Design

### 3.1 Task Descriptor Schema

```rust
pub struct TaskDescriptor {
    pub task_id: u128,
    pub graph_id: u128,
    pub task_name: String,
    pub placement_policy: PlacementPolicy,
    pub resource_requirements: ResourceRequirements,
    pub input_refs: Vec<BufferRef>,
    pub execution_payload: ExecutionPayload,
    pub timeout_ms: u32,
    pub retry_policy: RetryPolicy,
}

pub struct ResourceRequirements {
    pub min_ram_bytes: u64,
    pub required_accelerator: Option<AcceleratorType>, // NPU, GPU, Any
    pub min_vram_bytes: Option<u64>,
}

pub enum ExecutionPayload {
    LocalWasm { bytecode_hash: [u8; 32], entry: String },
    NativeKernel { symbol: String },
    ModelInference { model_alias: String, prompt_tokens: BufferRef },
}
```

### 3.2 Zero-Copy Buffer References (`BufferRef`)

Inputs and outputs are passed via memory-mapped slices or shared memory handles (`BufferRef`). Over the network, buffers are transmitted with zero serialization overhead: the receiving node maps the bytes directly into memory without parsing.

---

## 4. Drawbacks & Trade-offs

- **Memory Safety Rigor:** Zero-copy formats require strict memory layout alignment and pointer validation to prevent memory corruption when parsing buffers from untrusted remote peers.
- **Protocol Rigidity:** Binary schemas are less forgiving of field drift than schema-less JSON, requiring strict semantic versioning in message headers.

---

## 5. Prior Art & Alternatives Considered

- **Ray Core Tasks:** Ray uses Plasma Store for zero-copy object sharing, but Plasma is designed for high-memory server clusters rather than resource-constrained mobile and edge devices.
- **Dapr Workflows:** Dapr workflows orchestrate microservices via gRPC, which carries significant serialization overhead for large tensor arrays.
- **LangGraph:** LangGraph models stateful agent DAGs in Python, but lacks hardware accelerator primitives, offline local-first storage, or cross-node wire protocols.

---

## 6. Security & Privacy Implications

Task descriptors declare explicit data boundaries. Payloads marked with `LocalOnly` are rejected by the transport layer if directed towards an outgoing socket. All cross-node peer tasks are encrypted in-flight using QUIC/TLS 1.3 with peer certificate verification.

---

## 7. Migration & Compatibility

This is the foundational execution protocol for `0.1.0-alpha`. Forward compatibility will be maintained via schema versioning fields embedded in the root message frame.

---

## 8. Unresolved Questions

- What is the optimal eviction policy for intermediate graph buffers on battery-constrained mobile nodes?
- Should dynamic DAG branch expansion (loops, conditional jumps) be resolved purely on the scheduler node or delegated to local sub-graphs?

