# Project Governance

`ai-native-runtime` is an open-source research and engineering initiative operated under transparent, meritocratic principles inspired by the Rust, Kubernetes, and Apache Software Foundation governance models.

---

## 1. Decision-Making Model

All significant technical and architectural decisions occur in the open:
- **Major Features & Architectural Changes:** Proposed via the [RFC Process](file:///a:/master_project/rfcs/README.md).
- **Architectural Trade-offs:** Codified permanently in [Architecture Decision Records (ADRs)](file:///a:/master_project/adr/README.md).
- **Public Discourse:** Discussions are documented on public issue trackers, PR reviews, and community discussion boards.

Decisions are reached through **rough consensus and running code**. If consensus cannot be achieved, the Project Steering Committee / Maintainers facilitate resolution based on empirical data, benchmarks, and alignment with [`foundational-principles.md`](file:///a:/master_project/research/00-project-charter/foundational-principles.md).

---

## 2. Roles and Responsibilities

- **Contributors:** Anyone who submits research papers, documentation, issues, or pull requests.
- **Reviewers:** Experienced community members who provide code and literature reviews.
- **Maintainers:** Individuals responsible for managing subsystem health, triaging RFCs, ensuring test coverage, and cutting releases.

---

## 3. Subsystem Areas & Working Groups
- **WG-Research:** Literature surveys, experimental benchmarking, research papers.
- **WG-Runtime:** Rust core engine, async reactor, actor system, FFI.
- **WG-Scheduling:** Decentralized scheduler, capability profiling, placement algorithms.
- **WG-Data:** Local SQLite integration, CRDT convergence, state replication.
- **WG-Inference:** Accelerators, ONNX / LiteRT / ExecuTorch drivers, memory sandboxes.
- **WG-SDKs:** Language bindings (Dart/Flutter, TypeScript, Python).

