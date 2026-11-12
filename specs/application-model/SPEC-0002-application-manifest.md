# SPEC-0002: Application Manifest Specification (`anr.toml`)

- **Specification ID:** `SPEC-0002`
- **Subsystem:** Application Model & Configuration
- **Status:** Draft / Active Review
- **Version:** `0.1.0`
- **Authors:** AI-Native Runtime Research Group

---

## 1. Scope & Purpose

This specification defines the syntax, schema, and semantics of `anr.toml`, the declarative configuration file used by applications building on `ai-native-runtime`.

`anr.toml` establishes application identities, resource requirements, default placement policies, local-first state schemas, on-device AI model profiles, and peer trust domains.

---

## 2. Canonical Manifest Structure

```toml
[application]
id = "org.example.research-app"
name = "Research Assistant"
version = "0.1.0"
authors = ["Research Team <research@example.org>"]
entrypoint = "crates/main.rs"

[runtime]
core_version = "0.1"
default_placement = "LocalFirst"
max_local_memory_mb = 2048
allow_p2p_discovery = true

[scheduler.weights]
alpha_latency = 0.5
beta_energy = 0.3
gamma_monetary = 0.2

[storage]
engine = "sqlite-crdt"
persistence_mode = "DurableLocal"
sync_topology = "PeerGossip"
conflict_resolution = "AutomergeLWW"

[[models]]
alias = "reasoning-core"
source = "local-embedded"
format = "ggml"
path = "models/phi-3-mini-4k-instruct-q4.gguf"
accelerator_preference = ["NPU", "GPU", "CPU"]
max_context_tokens = 4096

[[models]]
alias = "frontier-fallback"
source = "remote-provider"
endpoint = "https://api.openai.com/v1"
placement_policy = "CloudFirst"
privacy_clearance = "AnonymizedOnly"

[[trust_domains]]
name = "local-lan"
allowed_nodes = ["fingerprint:sha256:7f83b165..."]
max_offload_tier = "Edge"
```

---

## 3. Schema Definitions

### 3.1 `[application]` Table
* `id` (String, Required): Globally unique reverse-DNS identifier.
* `name` (String, Required): Human-readable name.
* `version` (String, Required): Semantic version string (`MAJOR.MINOR.PATCH`).
* `entrypoint` (String, Required): Relative path to execution root.

### 3.2 `[runtime]` Table
* `default_placement` (Enum, Required): Default policy (`LocalOnly`, `LocalFirst`, `EdgeFirst`, `CloudFirst`, `DynamicOptimal`).
* `max_local_memory_mb` (Integer, Optional): Maximum memory allocation on host device before triggering memory shedding or offload.
* `allow_p2p_discovery` (Boolean, Default: `true`): Whether local peer discovery via mDNS / Bluetooth LE / LAN is enabled.

### 3.3 `[storage]` Table
* `engine` (Enum, Required): State persistence driver (e.g. `sqlite-crdt`, `libsql-vector`).
* `persistence_mode` (Enum): `EphemeralMemory` or `DurableLocal`.
* `sync_topology` (Enum): `PeerGossip`, `ClientHub`, or `MeshDirect`.

### 3.4 `[[models]]` Array
Defines AI models managed natively by the runtime kernel.
* `alias` (String, Required): Identifier used in application task graphs.
* `source` (Enum): `local-embedded` or `remote-provider`.
* `accelerator_preference` (Array of Strings): Priority order for hardware backend selection.

