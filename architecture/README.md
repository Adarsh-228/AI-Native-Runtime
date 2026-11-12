# System Architecture

This directory contains high-level and detailed architectural documentation, subsystem designs, and data/control plane interaction models.

## Architectural Layers

```
+-------------------------------------------------------------+
| 1. Intent Layer                                             |
|    Declarative Tasks, Autonomous Agents, User Workflows     |
+-------------------------------------------------------------+
                              |
+-------------------------------------------------------------+
| 2. Application Runtime                                      |
|    Lifecycle, Tool Sandboxes, Local State Management        |
+-------------------------------------------------------------+
                              |
+-------------------------------------------------------------+
| 3. Distributed Execution Engine                             |
|    Capability Profiling, Cost/Privacy Scheduling, Placement |
+-------------------------------------------------------------+
                              |
+-------------------------------------------------------------+
| 4. Data & Context Fabric                                    |
|    Local Relational DB, CRDT Convergence, Vector Indexes    |
+-------------------------------------------------------------+
                              |
+-------------------------------------------------------------+
| 5. Transport & Network Fabric                               |
|    Multiplexed QUIC/WebRTC/TLS, Peer Discovery, Zero-Copy   |
+-------------------------------------------------------------+
```

## Control Plane vs. Data Plane
- **Control Plane:** Node discovery, trust attestation, telemetry, and placement negotiations. Built to withstand network partitions without halting the local data plane.
- **Data Plane:** High-throughput streaming, zero-copy serialization, tensor execution, and state replication.

Detailed design: [`control-plane-data-plane.md`](control-plane-data-plane.md)


