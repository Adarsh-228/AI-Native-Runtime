# Domain Terminology & Definitions

To maintain consistency across specifications, research documents, and codebases, the following terms are rigorously defined:

- **Node:** An independent compute unit participating in the runtime network (e.g., mobile handset, embedded microcontroller, workstation, edge gateway, cloud container).
- **Control Plane:** The subsystem responsible for node discovery, capability registration, placement policy resolution, and cluster lifecycle.
- **Data Plane:** The execution path carrying task inputs, outputs, direct message streams, state synchronization payloads, and tensor buffers.
- **Task:** An atomic or composite unit of executable computation with defined inputs, deterministic or probabilistic outputs, resource prerequisites, and policy invariants.
- **Execution Placement Policy:** A set of constraints governing where a task is permitted or preferred to execute (`LocalOnly`, `LocalFirst`, `EdgeFirst`, `CloudFirst`, `DynamicCostOptimal`).
- **Capability Profile:** A hardware and software descriptor exported by a node, specifying CPU architectures, GPU compute APIs (Vulkan, Metal, WebGPU), NPU capabilities, available RAM/VRAM, network bandwidth, and energy state.
- **Local-First:** An architectural paradigm ensuring data reads and writes occur against local persistent storage, computations execute locally by default, and network synchronization occurs asynchronously without blocking the user.
- **AI-Native Runtime:** An execution environment providing first-class primitives for tensor computation, model weight management, tool invocation sandboxes, and context lifecycle tracking.
- **Intent Layer:** The top-level developer abstraction where applications express desired outcomes and constraints rather than imperatively binding to specific network sockets or remote servers.

