# Architecture Deep-Dive: Control Plane and Data Plane Separation

- **Document ID:** `ARCH-0001`
- **Subsystem:** Architectural Core & Distributed Systems
- **Status:** Approved
- **Authors:** AI-Native Runtime Research Group

---

## 1. Executive Design Thesis

Traditional distributed runtimes (e.g., Kubernetes, Ray) conflate control signals with data streams, creating monolithic bottlenecks and high synchronization overhead over untrusted, intermittent networks.

In consumer edge, multi-device, and peer-to-peer computing, nodes operate over asymmetric bandwidth, experience abrupt network disconnections, and maintain varying trust boundaries.

`ai-native-runtime` strictly decouples the **Control Plane** from the **Data Plane**:

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                              CONTROL PLANE                             │
 │    Node Discovery  •  Trust Attestation  •  Capability Profiling       │
 │            Scheduling Negotiations  •  Telemetry & Heartbeats          │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │ (Out-of-band policy metadata)
                                     ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │                               DATA PLANE                               │
 │    Zero-Copy IPC  •  Tensor Pointers  •  Local CRDT State Storage      │
 │           Multiplexed QUIC Streams  •  Hardware Accelerator Drivers    │
 └────────────────────────────────────────────────────────────────────────┘
```

---

## 2. The Control Plane

The Control Plane manages coordination, node membership, capability discovery, and task placement decisions.

### Responsibilities:
1. **Peer Discovery & Membership:** Discovers neighboring devices via mDNS, Bluetooth LE, and local multicast without centralized coordinating servers.
2. **Cryptographic Identity & Trust Attestation:** Authenticates candidate nodes using ed25519 public key pairs and capability tokens.
3. **Hardware & Resource Profiling:** Periodically measures node health:
   - Free memory and battery charge state
   - Available acceleration backends (Apple Neural Engine, Vulkan, CUDA, OpenVINO)
   - Round-trip ping latency and packet loss rates
4. **Placement Arbitration:** Evaluates task requirements against candidate profiles via the cost optimization function.

### Partition Tolerance Invariant:
A total failure or network partition of the Control Plane **MUST NOT** interrupt active local data plane execution or halt local-first storage.

---

## 3. The Data Plane

The Data Plane executes computation, streams data buffers, coordinates tensor operations, and syncs application state.

### Responsibilities:
1. **Zero-Copy Memory Transport:** Utilizes shared memory (IPC) for intra-machine task execution and memory-mapped buffers for inter-process tensor transfers.
2. **Multiplexed Transport:** Over the network, uses encrypted QUIC byte-streams with flow-control priority given to interactive UI events over background state convergence.
3. **Hardware Acceleration Execution:** Routes inference workloads directly to low-level hardware drivers (Metal Performance Shaders, Vulkan/DirectML, ONNX Runtime, libtorch).
4. **Local-First State Materialization:** Commits state mutations directly to local embedded SQLite/CRDT instances, emitting gossip deltas lazily to peer nodes.

---

## 4. Interaction Model

When an application invokes an offloadable AI task:
1. **Control Phase (Out-of-band):** The local scheduler negotiates placement with an eligible edge peer via lightweight protocol messages.
2. **Handshake Phase:** The edge node responds with a signed execution voucher indicating reserved capacity.
3. **Data Phase:** The local node streams serialized inputs directly to the edge worker over a dedicated QUIC data stream.
4. **Return & Convergence:** The result buffer returns over the data stream and commits atomically to local state.
