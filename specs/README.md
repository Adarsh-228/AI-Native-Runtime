# Technical Specifications

Formal, normative specifications defining the interfaces, data models, protocols, and behavioral contracts of `ai-native-runtime`.

## Active Specifications
- [`execution-model/SPEC-0001`](execution-model/SPEC-0001-task-lifecycle-and-placement.md): Task Lifecycle and Policy-Aware Placement Specification.
- [`application-model/SPEC-0002`](application-model/SPEC-0002-application-manifest.md): Application Manifest Specification (`anr.toml`).

## Planned Specifications
- `distributed-scheduling/`: Node capability profiles, placement algorithms, and negotiation.
- `networking-protocols/`: Peer discovery, session handshakes, and binary wire protocols.
- `data-model/`: Local storage engine contracts and CRDT state synchronization.
- `ai-inference/`: Model descriptor formats, memory allocation, and hardware acceleration bridges.
- `agent-interoperability/`: Tool invocation protocol and agent-to-agent messaging.
- `security-model/`: Cryptographic node identity, capability tokens, and sandbox isolation.
