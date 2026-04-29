# PhysMCP

**An open standard for Agentic IoT**

> *Every physical device, a native MCP Server.*

[![Status](https://img.shields.io/badge/status-working%20draft-orange)](./PHYSMCP_V0.1.md)
[![Version](https://img.shields.io/badge/spec-v0.1-blue)](./PHYSMCP_V0.1.md)
[![License (Spec)](https://img.shields.io/badge/spec%20license-CC--BY--SA%204.0-green)](./LICENSE)
[![License (SDK)](https://img.shields.io/badge/sdk%20license-Apache%202.0-green)](./LICENSE)

---

## What is PhysMCP?

PhysMCP defines how physical devices — sensors, actuators, controllers — expose themselves as native **Model Context Protocol (MCP) Servers**, enabling AI agents to read from and act upon the physical world without intermediary gateways, proprietary platforms, or custom translation layers.

The standard defines:

- A **canonical tool taxonomy** — three verbs: `read_*`, `do_*`, `can_do`
- A **declarative manifest format** — YAML binding semantic capabilities to physical interfaces
- A **peer-to-peer mesh layer** — semantic gossip, no central broker required
- A **feasibility protocol** — `can_do` negotiates physical reality before action
- A **wire protocol** — MCP / JSON-RPC over WebSocket, native on IP-capable devices

**The key architectural decision:** PhysMCP is mesh-first and gateway-optional. Devices form a peer-to-peer network using semantic gossip. Any device can coordinate any query. There is no broker, no hub, no single point of failure.

---

## Current Status

> ⚠️ **This is a working draft.** The specification is complete and coherent. The reference implementation does not yet exist. This repository is the canonical home of the spec and the starting point for the SDK.

| Component | Status |
|---|---|
| Specification v0.1 | ✅ Complete (working draft) |
| Manifest schema (YAML) | ✅ Defined in spec |
| Policy schema (YAML) | ✅ Defined in spec |
| CapabilityReport format | ✅ Defined in spec |
| Mesh protocol (gossip) | ✅ Defined in spec |
| Reference firmware (MicroPython) | 🔲 Not started |
| Reference firmware (Arduino C++) | 🔲 Not started |
| Reference firmware (Linux Python) | 🔲 Not started |
| Conformance test suite | 🔲 Not started |
| Reference hardware designs | 🔲 Not started |

**Authors:** [EnthalpyDW](https://enthalpy.mx) · Monterrey, Mexico  
**Spec started:** April 2026  
**Repository created:** April 2026

---

## The Three Standard Tools

PhysMCP defines exactly three tool patterns. No others are part of the standard.

```
read_{capability}     →  retrieve a value from the physical world
do_{capability}       →  act upon the physical world
can_do(query)         →  negotiate feasibility before committing action
```

Every PhysMCP-compliant device exposes its capabilities as specific MCP tools:

```yaml
# Example: a temperature sensor + fan controller
tools:
  - read_temperature_0      # returns {value: 24.7, unit: celsius}
  - do_fan_speed_0          # accepts {value: 0-100}
  - can_do                  # semantic feasibility across the mesh
```

The LLM agent selects tools by reading their schemas and descriptions — no custom integration code required.

---

## Why Gateway-Optional Matters

Every existing IoT-to-AI integration requires a centralized broker, hub, or gateway:

```
Today:    Sensor → MQTT → Broker → Gateway → MCP → Agent
PhysMCP:  Sensor ⇌ Agent   (direct, via mesh)
```

Centralized architectures create:
- Single points of failure
- Vendor lock-in at the broker layer
- $50,000+ gateway costs per industrial site
- Re-architecture when scaling beyond initial deployment

PhysMCP devices form a **small-world semantic mesh**. Peer connections are organized by capability affinity and physical location — not random topology. Any device can coordinate any `can_do` query. The coordinator role is ephemeral, stateless, and assumed per-query.

---

## The `can_do` Mechanism

`can_do` is PhysMCP's unique contribution to the MCP ecosystem. Before an agent commits any physical action, it can query the mesh for feasibility:

```json
// Agent asks: "can I lower the temperature in Zone 3?"
{
  "query": {
    "semantic_tags": ["cooling", "zone_3"],
    "direction": "decrease",
    "target": "temperature"
  }
}

// Mesh responds with a CapabilityReport:
{
  "feasible": true,
  "candidates": [
    {
      "device_id": "node_hvac_zone3_007",
      "capability": "do_fan_speed_0",
      "confidence": 0.92,
      "current_state": {"value": 30, "unit": "percent"},
      "constraints": {"min": 0, "max": 100},
      "side_effects": ["noise_increase", "energy_consumption"],
      "permission": "allowed"
    }
  ],
  "blocked": [
    {
      "device_id": "node_chiller_001",
      "reason": "device_offline",
      "permission": "device_unreachable"
    }
  ]
}
```

No other IoT+MCP framework provides pre-action feasibility negotiation with side-effect declaration and policy enforcement. This is the mechanism that makes PhysMCP safe for industrial deployment.

---

## Device Manifest

Every PhysMCP device declares a `physmcp.manifest.yml`:

```yaml
physmcp_version: "0.1"

device:
  id: node_hvac_zone3_007
  type: hvac_controller
  location:
    zone: zone_3

capabilities:
  - name: temperature_zone_3
    kind: read
    binding:
      type: i2c
      address: "0x48:reg=0x00"
    unit: celsius
    semantic_tags: [temperature, hvac, zone_3]

  - name: setpoint_0
    kind: do
    binding:
      type: modbus
      address: "slave=1:reg=40001"
    range: [18, 26]
    semantic_tags: [cooling, hvac, zone_3]
    requires_permission: operator
```

The manifest is the contract between hardware and agents. The SDK reads it and generates the MCP Server automatically.

---

## Design Philosophy

> **Spec for capability, not constraint.**
>
> The hardware of 2024 is a temporary embarrassment. The hardware of 2028 will run this standard without effort. TCP/IP was specified for hardware that didn't exist. MQTT was built for "millions of connected devices" when there were 50,000. Matter was criticized as too heavy for cheap chips — three years later, every smart home chip ships with Matter support by default.
>
> PhysMCP is designed for the silicon of 2028, deployed on the silicon of 2026.

Every constraint cited as a reason to simplify the protocol is a constraint that dissolves on a 5-year hardware cycle. The specification does not compromise to fit current low-end hardware. Current low-end hardware will catch up.

---

## Roadmap

### v0.1 — Mesh foundation *(current)*
**Target hardware:** ESP32-S3, RP2040, Linux SBCs  
**Status:** Specification complete. Implementation not started.

- [x] Three-tool vocabulary (`read_*`, `do_*`, `can_do`)
- [x] Semantic gossip mesh (SWIM-based, small-world topology)
- [x] Floating coordinator protocol
- [x] Manifest schema (YAML declarative bindings)
- [x] Policy schema (local per-device enforcement)
- [x] CapabilityReport format
- [x] Quorum protection for critical actuators
- [x] Network split detection and recovery
- [ ] Reference firmware — MicroPython (ESP32-S3)
- [ ] Reference firmware — Arduino C++
- [ ] Reference firmware — Linux Python
- [ ] Conformance test suite v0.1
- [ ] Reference hardware design (PhysMCP Node)

---

### v0.2 — On-device semantics *(planned, 2027)*
**Target hardware:** ESP32-P4, NPU-equipped MCUs

- [ ] Quantized embeddings on-device for local `can_do` matching
- [ ] Multi-site mesh federation
- [ ] Capability composition (chaining `do_*` across devices)
- [ ] PhysMCP/BLE transport profile

---

### v0.3 — Local reasoning *(planned, 2028)*
**Target hardware:** $20 BoM with native NPU

- [ ] Small language model on-device for intent parsing
- [ ] Multi-modal capabilities (vision, audio)
- [ ] Fully offline operation as default mode
- [ ] Federated learning across mesh

---

### v1.0 — Standard ratification *(planned, 2029)*

- [ ] Frozen protocol surface
- [ ] Backward compatibility guarantee
- [ ] Certification program launch
- [ ] Multi-vendor interoperability testing

---

## Comparison with Related Work

PhysMCP is not the first project to explore MCP + IoT. Here is an honest comparison:

| | MCP over MQTT (EMQ) | IoT-MCP (arXiv) | IoRT+MCP (IEEE) | **PhysMCP** |
|---|---|---|---|---|
| Devices as native MCP Servers | ✓ | ✗ | ✓ | ✓ |
| **No centralized broker/gateway** | ✗ | ✗ | ✗ | **✓** |
| **Peer-to-peer mesh** | ✗ | ✗ | ✗ | **✓** |
| **Semantic gossip routing** | ✗ | ✗ | ✗ | **✓** |
| **`can_do` feasibility protocol** | ✗ | ✗ | ✗ | **✓** |
| **Floating coordinator** | ✗ | ✗ | ✗ | **✓** |
| **Quorum + split protection** | ✗ | ✗ | ✗ | **✓** |
| Declarative manifest | partial | ✗ | partial | **✓** |
| Open spec + certification program | ✓ | ✗ | ✗ | **planned** |

**MCP over MQTT (EMQ)** is the most mature adjacent project. It uses a centralized MQTT broker as the coordination layer. PhysMCP's architectural bet is that the broker becomes unnecessary as edge hardware matures, and that peer-to-peer mesh is more resilient, more scalable, and more aligned with how physical deployments actually grow.

---

## Contributing

The specification is in active development. Contributions are welcome.

**What we need most right now:**

1. **Technical review** of the spec — is the gossip protocol correct? Is the `can_do` routing realistic?
2. **Implementation attempts** — any firmware that implements even one section
3. **Use case feedback** — does the manifest schema cover your hardware?
4. **Semantic tag vocabulary** — proposals for a controlled vocabulary of `semantic_tags`

**How to contribute:**

- Open an issue to propose changes to the spec
- Open a PR with corrections, clarifications, or additions
- Open a discussion for architectural questions

Please read [CONTRIBUTING.md](./CONTRIBUTING.md) (coming soon) before submitting.

**Note on authorship:** PhysMCP is currently authored by EnthalpyDW. There is no foundation or consortium yet. The spec license (CC-BY-SA 4.0) ensures that contributions remain open regardless of the organizational structure's future evolution.

---

## Related Projects

- [Model Context Protocol](https://modelcontextprotocol.io/) — Anthropic's open standard that PhysMCP builds upon
- [MCP over MQTT (EMQ)](https://www.emqx.com/mqtt-for-ai/mcp-over-mqtt/) — centralized-broker approach to MCP+IoT
- [IoT-MCP (arXiv 2510.01260)](https://arxiv.org/abs/2510.01260) — academic framework for LLM-IoT integration
- [MQTT Sparkplug B](https://sparkplug.eclipse.org/) — semantic context layer over MQTT (inspiration for manifest approach)
- [Matter](https://csa-iot.org/developer-faqs/what-is-matter/) — smart home standard (precedent for hardware certification model)

---

## License

The **PhysMCP Specification** is licensed under [CC-BY-SA 4.0](./LICENSE).  
The **PhysMCP Reference SDK** (when released) will be licensed under [Apache 2.0](./LICENSE).

© 2026 EnthalpyDW · Monterrey, Mexico  
[physmcp.org](https://physmcp.org) · [iotmcp.org](https://iotmcp.org)
