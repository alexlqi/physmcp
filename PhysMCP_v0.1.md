# PhysMCP Specification v0.1

**Physical Model Context Protocol — An Open Standard for Agentic IoT**

> *Every physical device, a native MCP Server.*

---

## Table of Contents

1. [Overview](#1-overview)
2. [Design Principles](#2-design-principles)
3. [Architecture](#3-architecture)
4. [The Three Standard Tools](#4-the-three-standard-tools)
5. [Manifest Schema](#5-manifest-schema)
6. [Policy Schema](#6-policy-schema)
7. [Mesh Layer](#7-mesh-layer)
8. [Peer Selection Algorithm](#8-peer-selection-algorithm)
9. [Semantic Routing for `can_do`](#9-semantic-routing-for-can_do)
10. [Floating Coordinator Protocol](#10-floating-coordinator-protocol)
11. [Network Splits and Recovery](#11-network-splits-and-recovery)
12. [CapabilityReport Format](#12-capabilityreport-format)
13. [Wire Protocol](#13-wire-protocol)
14. [Device Profiles](#14-device-profiles)
15. [Reference Implementation](#15-reference-implementation)
16. [Glossary](#16-glossary)

---

## 1. Overview

PhysMCP is an open standard that turns every physical device into a native **Model Context Protocol (MCP) Server**, enabling direct interaction between AI agents (LLMs) and the physical world without intermediary gateways or proprietary platforms.

The standard defines:

- A canonical tool taxonomy (`read_*`, `do_*`, `can_do`)
- A declarative manifest format binding semantic capabilities to physical interfaces
- A peer-to-peer mesh layer with semantic routing
- A discovery and feasibility-check protocol via `can_do`
- A wire protocol over MCP/JSON-RPC

PhysMCP is **gateway-optional**: the architecture works as a flat peer-to-peer mesh, with gateways becoming a deployment choice rather than a structural requirement.

### What problem PhysMCP solves

Today, connecting an AI agent to the physical world requires custom integration per vendor, per protocol, per deployment. PhysMCP standardizes this connection so that:

- Any agent that speaks MCP can interact with any PhysMCP-compliant device
- Hardware manufacturers implement the SDK once and become universally consumable
- Deployments scale from 3 nodes to 10,000 without re-architecture
- Agents reason about physical capability with semantic context, not raw tags

---

## 2. Design Principles

1. **MCP is the protocol.** No custom transport. No translation layer. Devices speak MCP natively.
2. **Two verbs, one negotiator.** The complete vocabulary is `read_*`, `do_*`, `can_do`.
3. **Specific tools, semantic metadata.** Each capability is a distinct tool with strong typing, not a string-dispatched generic call.
4. **Gateway-optional.** Mesh-first. Gateways are a role, not a tier.
5. **Declarative bindings.** Manifest YAML maps semantic names to physical interfaces.
6. **Sovereignty per device.** Each device enforces its own policy. No central bypass.
7. **Semantic mesh.** Peer connections are organized by capability affinity, not random topology.
8. **Stateless coordination.** Any device can coordinate any query. No leader election.
9. **Open standard, certified ecosystem.** The protocol is open; certification creates the marketplace.

---

## 3. Architecture

```
┌─────────────────────────────────────────────────┐
│  AGENT LAYER                                    │
│  Claude / GPT / Qwen / LangGraph / any          │
│  Multi-MCP-server client                        │
└──────────────────┬──────────────────────────────┘
                   │ MCP / JSON-RPC over WebSocket
┌──────────────────▼──────────────────────────────┐
│  DISCOVERY LAYER                                │
│  mDNS bootstrap + semantic gossip               │
└──────┬──────────┬──────────┬────────────────────┘
       │          │          │
┌──────▼──┐ ┌────▼────┐ ┌───▼─────┐
│ Node A  │ │ Node B  │ │ Node C  │
│ MCP/IP  │ │ MCP/IP  │ │ Serial  │
├─────────┤ ├─────────┤ ├─────────┤
│read_t_0 │ │do_relay │ │read_p_0 │
│can_do   │ │can_do   │ │can_do   │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
   I2C         GPIO        RS-485
     │           │           │
  Sensor      Actuator    Sensor
```

Each node is simultaneously:
- An MCP Server exposing its capabilities
- A gossip mesh participant
- A policy enforcement point for its own actuators
- A potential floating coordinator for `can_do` queries

---

## 4. The Three Standard Tools

PhysMCP defines exactly three tool patterns. No others are part of the standard.

### 4.1 `read_{capability}`

Retrieves a value from the physical world.

**Naming:** lowercase, snake_case, prefixed with `read_`.
**Examples:** `read_temperature_0`, `read_humidity_zone_3`, `read_pressure_main`.

**Schema:**
```json
{
  "name": "read_temperature_0",
  "description": "Read ambient temperature from sensor 0 in Zone 3",
  "inputSchema": {
    "type": "object",
    "properties": {},
    "required": []
  }
}
```

**Response shape:**
```json
{
  "value": 24.7,
  "unit": "celsius",
  "timestamp": "2026-04-27T15:32:11Z",
  "quality": "good"
}
```

The `quality` field MAY be one of: `good`, `stale`, `degraded`, `error`.

### 4.2 `do_{capability}`

Performs an action on the physical world.

**Naming:** lowercase, snake_case, prefixed with `do_`.
**Examples:** `do_relay_0`, `do_fan_speed_0`, `do_valve_main`.

**Schema:**
```json
{
  "name": "do_fan_speed_0",
  "description": "Set fan speed for Zone 3 ventilation. Range 0-100 percent.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "value": {
        "type": "integer",
        "minimum": 0,
        "maximum": 100
      }
    },
    "required": ["value"]
  }
}
```

**Response shape:**
```json
{
  "success": true,
  "applied_value": 70,
  "previous_value": 30,
  "timestamp": "2026-04-27T15:32:11Z"
}
```

### 4.3 `can_do`

Negotiates feasibility before action. The only generic tool in the standard.

**Schema:**
```json
{
  "name": "can_do",
  "description": "Check if a structured intent is feasible in the current physical mesh. Returns candidate devices, blocked candidates, and contextual state. Call before any do_* when topology or state is uncertain.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "object",
        "properties": {
          "semantic_tags": {
            "type": "array",
            "items": {"type": "string"},
            "description": "Tags describing the desired capability category"
          },
          "location": {
            "type": "string",
            "description": "Target location (zone, area, line)"
          },
          "direction": {
            "type": "string",
            "enum": ["increase", "decrease", "set", "toggle"],
            "description": "Action direction"
          },
          "target": {
            "type": "string",
            "description": "Property being targeted (temperature, pressure, etc.)"
          },
          "value": {
            "description": "Optional target value"
          }
        },
        "required": ["semantic_tags"]
      },
      "scope": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Optional list of device_ids to constrain search"
      },
      "ttl": {
        "type": "integer",
        "default": 3,
        "minimum": 1,
        "maximum": 5
      }
    },
    "required": ["query"]
  }
}
```

Response: see [Section 12](#12-capabilityreport-format).

---

## 5. Manifest Schema

Every PhysMCP device MUST declare a `physmcp.manifest.yml` describing its identity, capabilities, and bindings.

### 5.1 Full schema

```yaml
# physmcp.manifest.yml — REQUIRED on every device

physmcp_version: "0.1"

device:
  id: string                    # REQUIRED — globally unique device identifier
                                # format: lowercase, alphanumeric + underscore
                                # convention: <type>_<location>_<seq>
                                # example: node_temp_zone3_001

  type: string                  # REQUIRED — device category
                                # examples: environmental_sensor,
                                #           hvac_controller, valve_actuator

  vendor: string                # OPTIONAL — manufacturer name
  model: string                 # OPTIONAL — hardware model
  firmware_version: string      # OPTIONAL — semver
  hardware_revision: string     # OPTIONAL

  location:                     # REQUIRED — physical location metadata
    zone: string                # REQUIRED — logical zone identifier
    area: string                # OPTIONAL — sub-zone
    coordinates:                # OPTIONAL — physical coordinates
      x: float
      y: float
      z: float
    description: string         # OPTIONAL — human-readable

  certification:                # OPTIONAL — for certified hardware
    physmcp_certified: bool
    cert_id: string
    cert_expiry: string         # ISO 8601 date

network:
  protocols:                    # REQUIRED — list of supported transports
    - type: enum                # ip | serial | lora | ble
      address: string           # transport-specific binding
      mcp_port: integer         # for ip transports
      role: enum                # native | proxy | lite

  mesh:                         # REQUIRED — gossip configuration
    enabled: bool               # default: true
    seed_peers: [string]        # OPTIONAL — bootstrap IPs
    max_peers: integer          # default: 10
    heartbeat_interval_ms: int  # default: 1000

capabilities:                   # REQUIRED — list of all read_* and do_*
  - name: string                # REQUIRED — capability identifier
                                # tool name will be `read_{name}` or `do_{name}`

    kind: enum                  # REQUIRED — read | do

    description: string         # REQUIRED — human and LLM readable

    binding:                    # REQUIRED — physical interface mapping
      type: enum                # i2c | spi | gpio | adc | pwm | uart | modbus
                                # opcua | bacnet | mqtt | virtual
      address: string           # transport-specific
                                # i2c:   "0x48:reg=0x00"
                                # gpio:  "pin=14:mode=digital"
                                # pwm:   "pin=25:freq=25000"
                                # modbus: "slave=1:reg=40001:type=holding"

    data_type: enum             # REQUIRED for read — float | int | bool | string
                                # REQUIRED for do — float | int | bool | enum

    unit: string                # OPTIONAL but RECOMMENDED for read
                                # examples: celsius, percent, pascal, lux

    range:                      # OPTIONAL — value bounds
      min: number
      max: number

    enum_values: [string]       # OPTIONAL — for enum data_type

    precision: number           # OPTIONAL — for float reads (resolution)

    sample_rate_hz: number      # OPTIONAL — for read, max sampling frequency

    semantic_tags: [string]     # REQUIRED — vocabulary for can_do routing
                                # examples:
                                #   read: [temperature, environmental, ambient]
                                #   do:   [cooling, ventilation, energy_consumer]

    side_effects: [string]      # OPTIONAL — for do_*, declared side effects
                                # examples: [noise_increase, energy_consumption,
                                #            zone_3_air_change]

    rate_limit: string          # OPTIONAL — e.g. "1/min", "10/hour"

    requires_permission: enum   # OPTIONAL — none | operator | supervisor | admin
                                # default: none

    requires_quorum: bool       # OPTIONAL — for critical actuators
                                # default: false

    quorum_size: integer        # OPTIONAL — minimum peer count if quorum required
                                # default: 3

    cooldown_ms: integer        # OPTIONAL — minimum time between consecutive do_*

    timeout_ms: integer         # OPTIONAL — max execution time
                                # default: 5000

    cacheable: bool             # OPTIONAL — for read, whether value can be cached
                                # default: true for read, false for do
    cache_ttl_ms: integer       # OPTIONAL — default: 1000

  # Adopted capabilities (proxy mode for constrained peers)
  - name: string
    kind: enum
    adopted_from:               # OPTIONAL — when this device proxies for another
      source_address: string    # serial:/dev/ttyS0:addr=0x10
      source_protocol: enum
    binding:
      type: virtual
    # ... rest as above

audit:
  enabled: bool                 # default: true
  retention_days: integer       # default: 7
  remote_sink:                  # OPTIONAL — external log destination
    type: enum                  # mqtt | http | syslog
    address: string
```

### 5.2 Minimal valid example

```yaml
physmcp_version: "0.1"

device:
  id: node_temp_zone3_001
  type: environmental_sensor
  location:
    zone: zone_3

network:
  protocols:
    - type: ip
      address: "192.168.1.45"
      mcp_port: 3000
      role: native
  mesh:
    enabled: true

capabilities:
  - name: temperature_0
    kind: read
    description: "Ambient temperature sensor in Zone 3"
    binding:
      type: i2c
      address: "0x48:reg=0x00"
    data_type: float
    unit: celsius
    range:
      min: -40
      max: 125
    precision: 0.0625
    semantic_tags: [temperature, environmental, ambient]
```

### 5.3 Full example with actuator

```yaml
physmcp_version: "0.1"

device:
  id: node_hvac_zone3_007
  type: hvac_controller
  vendor: enthalpy_dw
  model: PhysMCP-Node-v1
  firmware_version: "1.0.3"
  location:
    zone: zone_3
    area: north_wing
    description: "HVAC control unit, Line A north"

network:
  protocols:
    - type: ip
      address: "192.168.1.107"
      mcp_port: 3000
      role: native
  mesh:
    enabled: true
    max_peers: 10
    heartbeat_interval_ms: 1000

capabilities:
  - name: temperature_zone_3
    kind: read
    description: "HVAC return air temperature, Zone 3"
    binding:
      type: i2c
      address: "0x48:reg=0x00"
    data_type: float
    unit: celsius
    precision: 0.1
    semantic_tags: [temperature, hvac, zone_3, ambient]
    sample_rate_hz: 1
    cacheable: true
    cache_ttl_ms: 2000

  - name: setpoint_0
    kind: do
    description: "Set HVAC target temperature for Zone 3. Range 18-26C."
    binding:
      type: modbus
      address: "slave=1:reg=40001:type=holding"
    data_type: float
    unit: celsius
    range:
      min: 18
      max: 26
    semantic_tags: [cooling, heating, hvac, zone_3, energy_consumer]
    side_effects:
      - zone_3_climate_change
      - energy_consumption
    rate_limit: "6/hour"
    requires_permission: operator
    cooldown_ms: 30000
    timeout_ms: 2000

  - name: fan_mode_0
    kind: do
    description: "Set HVAC fan mode"
    binding:
      type: modbus
      address: "slave=1:reg=40010:type=holding"
    data_type: enum
    enum_values: [auto, low, medium, high, off]
    semantic_tags: [ventilation, hvac, zone_3]
    side_effects: [noise_change, energy_consumption]
    requires_permission: operator

audit:
  enabled: true
  retention_days: 30
```

---

## 6. Policy Schema

Each device MAY declare a `physmcp.policy.yml` enforcing local rules.

```yaml
# physmcp.policy.yml — OPTIONAL but recommended for actuators

physmcp_policy_version: "0.1"

# Applied to all do_* unless overridden
defaults:
  require_permission: none
  rate_limit: null
  audit_all: true

# Per-capability overrides
capabilities:
  do_setpoint_0:
    rules:
      - type: time_window
        allow_during: ["business_hours"]
        deny_during: ["maintenance_window"]
      - type: permission
        require: [operator, supervisor]
      - type: rate
        max: "6/hour"
      - type: quorum
        min_peers: 3
        on_failure: deny
      - type: range_guard
        min: 19
        max: 25
        on_violation: clamp     # clamp | deny | warn

  do_fan_mode_0:
    rules:
      - type: permission
        require: [operator]
      - type: rate
        max: "12/hour"

# Time window definitions
time_windows:
  business_hours:
    days: [mon, tue, wed, thu, fri]
    start: "07:00"
    end: "19:00"
    timezone: "America/Monterrey"

  maintenance_window:
    days: [sun]
    start: "02:00"
    end: "06:00"
    timezone: "America/Monterrey"

# Identity providers for permission checks
identity:
  providers:
    - type: jwt
      issuer: "https://auth.example.com"
      audience: "physmcp-mesh"
    - type: mtls
      ca_cert_path: "/etc/physmcp/ca.crt"

# Emergency overrides
emergency:
  override_permission: admin
  override_token_ttl_seconds: 300
  audit_all_overrides: true
```

---

## 7. Mesh Layer

PhysMCP devices form a peer-to-peer mesh via semantic gossip.

### 7.1 Bootstrap

1. Device boots, loads manifest, starts MCP Server
2. Performs mDNS query: `_physmcp._tcp.local`
3. Any responding peer returns its IP and `mcp_port`
4. Device connects to first responder via MCP, calls `list_peers`
5. Receives initial member list, populates peer table

If `seed_peers` is configured in the manifest, those are tried before mDNS.

### 7.2 Peer table structure

Each device maintains three categorized slots:

| Slot category    | Count | Selection criterion                          |
|------------------|-------|----------------------------------------------|
| `same_zone`      | 3-5   | Peers with matching `location.zone`          |
| `same_capability`| 3-5   | Peers with overlap in `semantic_tags`        |
| `long_distance`  | 2-3   | Random peers with low semantic affinity      |

Total active links per node: 8-13.

### 7.3 Heartbeat

Each device pings its peers every `heartbeat_interval_ms` (default 1000).
- Three consecutive missed pings → `suspected`
- Five consecutive missed pings → `dead`, slot freed
- Status changes propagate via gossip

---

## 8. Peer Selection Algorithm

Each device evaluates candidates with a weighted score function and rebalances peer slots every 30 seconds.

### 8.1 Score function

```
score(candidate) =
    w_zone * location_match(self, candidate)
  + w_caps * jaccard(self.tags, candidate.tags)
  - w_lat  * normalized_latency(candidate)
  - w_age  * staleness(candidate)
  + w_div  * graph_distance_estimate(candidate)
```

Default weights:

| Weight | Default | Purpose                                   |
|--------|---------|-------------------------------------------|
| w_zone | 1.0     | Reward same-zone peers                    |
| w_caps | 1.0     | Reward semantic overlap                   |
| w_lat  | 0.5     | Penalize high latency                     |
| w_age  | 0.3     | Penalize stale information                |
| w_div  | 0.7     | Reward graph distance (anti-clustering)   |

The weights differ per slot:
- `same_zone` slots: maximize `w_zone`
- `same_capability` slots: maximize `w_caps`
- `long_distance` slots: maximize `w_div`, ignore `w_zone` and `w_caps`

### 8.2 Maintenance loop

Every 30 seconds, each node:

1. Evaluates current peers against fresh candidates
2. If a slot has a strictly better candidate available, replaces it
3. Drops peers that have become redundant (e.g., two peers with identical tag sets in same zone)
4. Triggers a scout if any slot is unfilled
5. Updates routing hints based on observed query latencies

### 8.3 Anti-clustering guarantee

`long_distance` peers are explicitly chosen with **low** semantic affinity to prevent fragmentation. This preserves the small-world property: any node is reachable from any other in O(log n) hops.

---

## 9. Semantic Routing for `can_do`

Queries are routed by **directed semantic forwarding** with TTL, not flood broadcast.

### 9.1 Routing parameters

| Parameter    | Default | Range  | Purpose                              |
|--------------|---------|--------|--------------------------------------|
| TTL          | 3       | 1-5    | Maximum hops from coordinator        |
| Fanout       | 3       | 1-5    | Max peers each node forwards to      |
| Hop timeout  | 150ms   | -      | Per-hop response wait                |
| Total timeout| 500ms   | -      | Coordinator final timeout            |

### 9.2 Forwarding logic

When a node receives a `can_do` query:

```
1. Check query_id in deduplication cache
   - If seen, drop silently
   - If new, record query_id with TTL of 5 seconds

2. Evaluate self:
   - For each local capability:
     - if capability.semantic_tags intersects query.semantic_tags
       AND (no location filter OR location matches)
     - add to local matches

3. If TTL > 0, select peers to forward:
   - Filter peers whose advertised tags intersect query.semantic_tags
   - Filter peers in or adjacent to query.location
   - Exclude visited[]
   - Rank by tag overlap, take top `fanout`

4. Forward query with:
   - TTL decremented by 1
   - visited += [self.id]
   - parent_query_id = original

5. Wait for responses up to hop_timeout

6. Aggregate:
   - local matches
   - responses from forwarded peers
   - deduplicate by device_id

7. Return aggregated CapabilityReport to upstream
```

### 9.3 Coverage estimation

With `fanout=3` and `TTL=3`:
- Hop 1: 3 peers
- Hop 2: 9 peers
- Hop 3: 27 peers
- Total reachable: ~40 peers per query

For specific tags this is over-coverage; for broad queries it ensures candidates are found.

### 9.4 Adaptive TTL (optional v0.2)

Coordinators MAY adjust TTL based on:
- Specificity of `semantic_tags` (more tags → lower TTL)
- Number of candidates already found mid-aggregation
- Historical query success patterns

---

## 10. Floating Coordinator Protocol

The first node receiving a `can_do` from an agent assumes the **coordinator role** for that single query. The role is ephemeral and stateless.

### 10.1 Coordinator responsibilities

1. Generate `query_id` UUIDv4
2. Initialize response buffer with TTL
3. Apply self-evaluation (as per §9.2 step 2)
4. Initiate fanout to selected peers
5. Wait for responses up to `total_timeout` (default 500ms)
6. Apply final filters:
   - Deduplicate by `device_id`
   - Apply liveness check (alive in gossip)
   - Apply policy check (current node's policy)
7. Construct `CapabilityReport`
8. Return MCP response to agent
9. Discard query state

### 10.2 No leader election

Any node can coordinate any query. If a coordinator dies mid-query:
- The agent's MCP call times out
- Agent retries (typically against another peer)
- The new coordinator handles it identically

This makes the protocol resilient: there is no leader to lose.

### 10.3 Late response handling

Responses arriving after `total_timeout`:
- MAY be cached for similar queries (TTL 5s)
- MUST NOT be returned to the agent for the original query
- MUST NOT extend the coordinator's lifetime

### 10.4 Concurrent queries

Each node handles concurrent queries independently. Memory budget per query: ~1-4KB. ESP32-S3 can handle 50+ concurrent queries without strain.

---

## 11. Network Splits and Recovery

### 11.1 Detection

Each node maintains an estimated `mesh_size` derived from gossip member counts.

A node enters `suspected_partition` state when:
- Reachable peer count drops by >40% within 60 seconds
- Multiple `same_zone` peers become unreachable simultaneously
- Heartbeats to long-distance peers fail consistently

### 11.2 Healing mode behavior

When `suspected_partition` is set:
- mDNS scout interval reduced from 30s to 5s
- Member list requests sent to all `long_distance` peers
- Cooldown for new peer adoption is reduced
- Query TTLs are temporarily increased by 1 to compensate for reduced reach

### 11.3 Quorum protection

Capabilities marked `requires_quorum: true` enforce minimum peer visibility before action:

```yaml
- name: emergency_shutdown_0
  kind: do
  requires_quorum: true
  quorum_size: 5
  semantic_tags: [safety_critical, emergency]
```

If alive peer count < `quorum_size`, the device returns:

```json
{
  "permission": "partition_no_quorum",
  "reason": "current alive peers: 2, required: 5"
}
```

This prevents split-brain on safety-critical actuators.

### 11.4 Merge protocol

When partitions reconnect:

1. Each side detects unknown `device_id`s in incoming gossip
2. Both sides exchange complete member lists
3. Conflict resolution by `last_seen` timestamp (most recent wins)
4. `capabilities_hash` mismatches trigger re-fetch of manifest
5. Peer tables rebalance under normal scoring

### 11.5 Exponential healing backoff

After a successful merge, scouting frequency returns to normal exponentially:

```
scout_interval = base_interval * (1 - decay^seconds_since_heal)
```

Default `base_interval=30s`, `decay=0.95`. After ~120s post-heal, behavior is fully normal.

---

## 12. CapabilityReport Format

The canonical response shape for `can_do`.

### 12.1 Full schema

```json
{
  "query_id": "uuid-v4",
  "feasible": true,
  "candidates": [
    {
      "device_id": "node_001",
      "capability": "do_fan_speed_0",
      "confidence": 0.92,
      "match_reasons": [
        "tag_overlap: cooling, ventilation",
        "location_match: zone_3",
        "currently_online"
      ],
      "current_state": {
        "value": 30,
        "unit": "percent",
        "timestamp": "2026-04-27T15:32:11Z"
      },
      "constraints": {
        "min": 0,
        "max": 100,
        "unit": "percent",
        "data_type": "integer"
      },
      "side_effects": [
        "noise_increase",
        "energy_consumption"
      ],
      "rate_limit": {
        "max": "1/min",
        "current_window_remaining": 1
      },
      "permission": "allowed",
      "estimated_latency_ms": 50,
      "estimated_effect_lag_ms": 200
    }
  ],
  "blocked": [
    {
      "device_id": "node_012",
      "capability": "do_chiller_0",
      "reason": "device_offline",
      "details": "last_seen 4 minutes ago",
      "permission": "device_unreachable"
    },
    {
      "device_id": "node_015",
      "capability": "do_window_actuator_0",
      "reason": "policy_denied",
      "details": "actuator locked during production hours",
      "permission": "policy_denied"
    },
    {
      "device_id": "node_023",
      "capability": "do_emergency_stop_0",
      "reason": "quorum_failure",
      "details": "current peers 2, required 5",
      "permission": "partition_no_quorum"
    }
  ],
  "context": {
    "location": "zone_3",
    "related_reads": [
      "node_001.read_temperature_0",
      "node_007.read_temperature_zone_3"
    ],
    "current_observations": {
      "temperature_zone_3": {
        "value": 26.8,
        "unit": "celsius",
        "source": "node_007.read_temperature_zone_3"
      }
    },
    "implied_target": "decrease temperature_zone_3 below current 26.8"
  },
  "metadata": {
    "coordinator_node": "node_042",
    "ttl_used": 3,
    "fanout_used": 3,
    "peers_consulted": 12,
    "responses_received": 8,
    "responses_dropped_late": 1,
    "total_latency_ms": 387,
    "timestamp": "2026-04-27T15:32:11Z",
    "mesh_size_estimate": 184,
    "partition_state": "healthy"
  }
}
```

### 12.2 Field semantics

#### `feasible` (boolean, required)
- `true` if at least one candidate exists with `permission: allowed`
- `false` otherwise

#### `candidates` (array, required)
Each entry MUST include:
- `device_id` — origin device
- `capability` — full tool name (e.g., `do_fan_speed_0`)
- `confidence` — float 0.0-1.0, semantic match strength
- `permission` — one of: `allowed`, `requires_human_approval`, `policy_denied`, `device_unreachable`, `partition_no_quorum`, `rate_limited`, `cooldown_active`

Optional fields enrich agent reasoning but are not strictly required for protocol compliance.

#### `blocked` (array, required, may be empty)
Candidates that match semantically but cannot execute. Helps the agent understand why options are unavailable.

#### `context` (object, optional)
Provides surrounding state to inform agent decisions. Particularly:
- `current_observations`: live reads of related sensors
- `related_reads`: tool names the agent can call for more context

#### `metadata` (object, recommended)
Diagnostics for debugging and protocol observability.

### 12.3 Permission enum

| Value                       | Meaning                                         |
|-----------------------------|-------------------------------------------------|
| `allowed`                   | Action will execute                             |
| `requires_human_approval`   | Action requires human-in-the-loop confirmation  |
| `policy_denied`             | Local policy rejects the action                 |
| `device_unreachable`        | Device offline or unreachable                   |
| `partition_no_quorum`       | Mesh partitioned, quorum not met                |
| `rate_limited`              | Rate limit exceeded for this period             |
| `cooldown_active`           | Cooldown from previous action still active      |
| `permission_insufficient`   | Caller lacks required role                      |
| `out_of_range`              | Requested value outside declared bounds         |

---

## 13. Wire Protocol

PhysMCP uses standard MCP over JSON-RPC 2.0.

### 13.1 Transports

| Transport        | Use case                              |
|------------------|---------------------------------------|
| WebSocket + JSON | IP-capable devices (default)          |
| HTTP + SSE       | IP-capable devices (alternative)      |
| Serial JSON-RPC  | UART/RS-485 for constrained devices   |
| BLE GATT         | Battery-powered peripherals (v0.2)    |

### 13.2 Standard MCP methods

PhysMCP devices implement the standard MCP method set:

- `initialize`
- `tools/list`
- `tools/call`
- `resources/list`
- `resources/read`
- `notifications/*`

PhysMCP-specific methods (extension namespace `physmcp/`):

- `physmcp/list_peers` — return current peer table
- `physmcp/mesh_status` — return mesh health and partition state
- `physmcp/manifest` — return current manifest

### 13.3 Authentication

Devices MAY require authentication for `do_*` calls:

- mTLS for IP transports
- JWT bearer tokens
- API keys (development only)

Authentication is enforced per-policy, declared in `physmcp.policy.yml`.

---

## 14. Device Profiles

PhysMCP defines profiles for different hardware classes:

### 14.1 Profile: Full

- IP stack with WebSocket
- Native MCP Server
- Full gossip participation
- Local policy enforcement
- Audit logging
- Examples: ESP32-S3, RP2040+W5500, Linux SBC, Jetson

### 14.2 Profile: Lite

- Telemetry-only (read-only capabilities)
- May lack IP stack
- Discovered through proxy peer
- Examples: 8-bit MCUs, LoRa sensors, BLE peripherals

### 14.3 Profile: Proxy

- Full profile device that proxies for Lite peers
- Manifest declares `adopted_capabilities`
- Lite peers appear as virtual capabilities on the proxy
- Examples: ESP32 bridging RS-485 sensors

### 14.4 Profile: Aggregator (Gateway, optional)

- Full profile with high compute
- May host local LLM for `can_do` intent parsing
- May offer cross-mesh routing for multi-site deployments
- Not architecturally required — optional convenience role

---

## 15. Reference Implementation

The PhysMCP reference implementation is published as `physmcp-sdk`:

```
physmcp-sdk/
├── firmware/
│   ├── micropython/    # ESP32-S3, RP2040
│   ├── arduino/        # C++ for Arduino-compatible boards
│   └── linux/          # Python for Linux SBCs
├── manifest/
│   ├── schema.json     # JSON Schema for physmcp.manifest.yml
│   └── validator.py    # Manifest validation tool
├── policy/
│   ├── schema.json     # JSON Schema for physmcp.policy.yml
│   └── enforcer.py     # Policy evaluation engine
├── mesh/
│   ├── gossip.py       # Semantic gossip implementation
│   ├── peer_table.py   # Peer slot management
│   └── coordinator.py  # Floating coordinator logic
├── server/
│   └── mcp_server.py   # MCP server bridging tools to bindings
├── client/
│   └── mesh_client.py  # Multi-server MCP client for agents
└── spec/
    └── PhysMCP-Specification-v0.1.md
```

License: Apache 2.0 for SDK, CC-BY-SA 4.0 for the specification document.

### 15.1 Reference hardware

Two reference hardware designs ship with the SDK:

**PhysMCP Node** — ESP32-S3 based, factor industrial DIN, $15-25 BOM
**PhysMCP Aggregator** — CM4-based, optional gateway role, $80-150 BOM

Hardware schematics and PCB layouts are open under CERN-OHL-S.

### 15.2 Certification program

Third-party hardware can be certified PhysMCP-compliant by:

1. Implementing the SDK or equivalent
2. Passing the conformance test suite
3. Declaring valid manifest with semantic_tags from controlled vocabulary
4. Annual recertification fee

Certified devices appear in the PhysMCP marketplace and may use the certification mark.

---

## 16. Glossary

| Term                  | Definition                                                      |
|-----------------------|-----------------------------------------------------------------|
| **Capability**        | A named atomic operation: a `read_*` or `do_*` tool             |
| **Coordinator**       | Ephemeral role assumed by the node receiving a `can_do` query   |
| **Floating role**     | A role any node can assume; not persistent or elected           |
| **Gossip**            | Peer-to-peer membership and metadata propagation                |
| **Manifest**          | YAML declaration of device identity, capabilities, bindings     |
| **Mesh**              | The collective of PhysMCP devices forming a peer-to-peer network|
| **Node**              | A single PhysMCP device                                         |
| **Policy**            | Local rules governing capability execution                      |
| **Quorum**            | Minimum peer count required for critical actions                |
| **Semantic tags**     | Controlled vocabulary describing capability category            |
| **Skill Pack**        | Vertical-specific configuration of capabilities and policies    |
| **Slot**              | A categorized position in the peer table                        |
| **Small-world graph** | Graph with O(log n) diameter and high local clustering          |
| **TTL**               | Time-to-live, here measured in hops                             |

---

## Versioning

This specification is **PhysMCP v0.1**. The standard follows semver:

- **Major**: incompatible protocol changes
- **Minor**: backward-compatible additions
- **Patch**: clarifications and corrections

Devices MUST declare `physmcp_version` in their manifest. Mesh interoperability is guaranteed within the same major version.

---

## Status

This document is a working specification. Implementations and feedback are welcome. The reference implementation tracks this spec.

**Spec authors:** EnthalpyDW
**License:** CC-BY-SA 4.0
**Repository:** TBD
**Discussion:** TBD

---

*End of PhysMCP Specification v0.1*
