# Snappi-VPP Traffic Generator High Level Design

**Revision:** 1.0  
**Status:** Draft   

### Revision History

| Revision | Date | Summary |
|---|---|---|
| 1.0 | Jun 2026 | SONiC HLD - Initial draft. |

### Terminology

| Term | Meaning |
|---|---|
| OTG | Open Traffic Generator API model. |
| Snappi | Python client for the OTG API. |
| TGen | Traffic generator. |
| DUT | Device under test. |
| VPP | Vector Packet Processing dataplane. |
| PAPI | VPP Python API client for binary API calls. |
| FRR | Free Range Routing, used to emulate routed peers. |
| LAG | Link Aggregation Group. |
| LACP | IEEE 802.3ad link aggregation control protocol. |
| PFC | Priority Flow Control. |
| vhost-user | Shared-memory virtio dataplane between VPP and QEMU. |

### Background

SONiC uses a modular, microservices-based architecture where network functions run in independent Docker containers. This design makes its components -- including VPP, FRR etc as reusable components. A SONiC device under test can be a physical switch with ASIC hardware, or a virtual SONiC instance backed by VPP for software dataplane testing. `sonic-mgmt` is the test framework used to run functional, protocol, scale, convergence, QoS, and platform tests against those devices.

Many SONiC traffic tests use Snappi, the Python client for the Open Traffic Generator API. A Snappi test describes ports, devices, protocols, flows, captures, metrics, and control actions in a vendor-neutral way. Today, those requests are commonly sent to a hardware traffic-generator chassis or a vendor traffic-generator appliance. The proposal in this HLD is to provide another OTG endpoint: a SONiC-native VPP-OTG server.

VPP-OTG has three major runtime pieces - OTG adapter, VPP and FRR. This HLD covers the overall VPP-OTG design: the VPP traffic-generation dataplane extensions, the OTG adapter, the FRR/control-plane integration, container packaging, and the sonic-mgmt testbed integration.


## Overview

SONiC already has the major building blocks needed for a software traffic generator. `sonic-vpp` provides an actively maintained VPP-based software dataplane, and FRR is part of the SONiC control-plane stack. Snappi-VPP assembles these existing components into an OTG/Snappi-compatible traffic-generator endpoint that `sonic-mgmt` tests can use as a drop-in replacement for a hardware chassis.

The system has three runtime pieces:

1. **OTG adapter** -- a Python process that speaks the Snappi/OTG REST API, translates test intent into backend operations, and returns OTG-compatible metrics.
2. **VPP with flowgen plugin** -- the dataplane that generates, receives, timestamps, and counts packets at configured rates.
3. **FRR** -- emulates routed peers (BGP, OSPF, ISIS) so the DUT can form control-plane sessions with the tester.

All three run inside a single container (`docker-vpp-otg`). From the test framework's perspective, the container is a traffic-generator appliance reachable on a REST port, just like an Ixia chassis.

### Motivation

Using VPP for the traffic-generator dataplane keeps the test infrastructure aligned with SONiC itself. For SONiC testing, features that need OTG coverage are normally features already being developed or maintained in the SONiC ecosystem. When that work adds VPP dataplane support, VPP interface support, packaging, observability, FRR integration, LACP behavior, 802.1X, MACsec, or other protocol/daemon support, the same capability can be consumed by VPP-OTG with adapter-side mapping rather than maintained in a separate traffic-generator fork. This is a lower-maintenance path than importing a separate open-source traffic generator that would need its own lifecycle, feature tracking, packaging model, and SONiC-specific adaptations. If such an external project is not developed and maintained with SONiC's requirements in mind, it can drift away from what SONiC needs as new test coverage and platform features are added.

The hardware landscape also makes a VPP-based OTG server more attractive than it was historically. Commodity SmartNICs and modern interface cards increasingly provide offloads that used to require custom traffic-generator ASICs. A VPP-OTG server can plug into these offloads over time while keeping the same OTG adapter and SONiC test integration. This reduces the gap between a physical Ixia-style chassis and an off-the-shelf server running VPP-OTG with capable NICs, especially for the large set of tests that need rate, counters, protocol behavior, and repeatability more than proprietary chassis features.

The practical workflow benefit is large. A developer should be able to run a `sonic-otg` / `sonic-vpp` style setup on a laptop or lab server and exercise meaningful Snappi tests without reserving a hardware traffic-generator chassis. The same setup can be used in CI/CD for the majority of protocol-correctness, routing, convergence, link-event, counter, and software dataplane cases. Strict hardware-dependent cases, such as tests requiring ASIC timing, ASIC queue behavior, physical optics, or real hardware PFC/ECN enforcement, can still be delegated to hardware chassis and physical DUT labs.

Snappi-VPP therefore focuses on making VPP and FRR look like an OTG traffic generator to `sonic-mgmt`. That requires both southbound dataplane work and northbound API work. VPP must provide the traffic-generator dataplane functions: stream creation, rate control, packet templating, timestamping, RX measurement, counters, capture hooks, and scalable worker/NIC behavior. The adapter must accept Snappi/OTG configuration, map it to VPP dataplane objects and FRR protocol state, and return OTG-compatible metrics and capture results. The design keeps most of the implementation burden inside components already relevant to SONiC, while keeping the adapter purpose-built for SONiC test needs.

## Scope

### Goals

1. Provide an OTG/Snappi-compatible API endpoint that existing `sonic-mgmt/tests/snappi_tests` can use with minimal or no test changes.
2. Reuse SONiC-community components: VPP for dataplane, FRR for control-plane emulation and any other parts as needed.
3. Generate and receive Ethernet/IP traffic through VPP at configured rates (pps, bps, percent-line-rate, fixed-duration, fixed-packet, burst).
4. Support BGP and LACP test coverage as first-class functional targets to start with.
5. Support port metrics, flow metrics, protocol metrics, capture, link state control, traffic start/stop, and route advertise/withdraw.
6. Run as a single container deployable on a developer laptop, lab server, or CI worker.
7. Integrate with `ars vswitch setup --enable-tgen` and the SONiC vlab topology model.
8. Support both sonic-vpp virtual testbeds and physical-DUT testbeds.
9. Keep the dataplane extensible for PFC, ECN, QoS, NIC offloads, and SmartNIC acceleration.

### Non-Goals

1. Snappi-VPP is not an IxNetwork emulator and does not implement IxNetwork-private Snappi extensions.
2. It does not provide ASIC-class timestamp accuracy or hardware latency precision.
3. It does not claim unlimited single-instance throughput. The design targets linear scaling for the throughput required by SONiC test cases by adding VPP workers, NIC queues, ports, SmartNIC/offload support, and additional VPP-OTG servers where needed.
4. It does not emulate every chassis or multi-DUT failure scenario in the first release.
5. It does not implement full L7 stateful traffic generation in the initial phases. This can be taken up later if DASH or other SONiC test areas require it.

## Requirements

### Functional Requirements

| Area | Requirement |
|---|---|
| Northbound API | Accept OTG-style REST requests from the Snappi Python client. |
| Configuration | Accept ports, layer1, devices, LAGs, captures, and flows from an OTG config object. |
| Traffic | Support continuous, fixed-packet, fixed-duration, and burst traffic patterns. |
| Packet templates | Support Ethernet, IPv4, IPv6, TCP, UDP, VLAN, PFC pause, Ethernet pause, custom bytes, and field variations used by the target tests. |
| Protocols | Support BGP IPv4/IPv6 peers and route ranges; support LACP LAGs. Other protocol adapters may be present as partial or smoke support as needed by sonic-mgmt snappi tests. |
| Control state | Support traffic start/stop, link up/down, capture start/stop, protocol start/stop, and route advertise/withdraw. |
| Metrics | Report port, flow, BGP, LACP, capture, and convergence metrics in OTG-compatible response shapes. |
| Capture | Return pcap data for requested ports with adapter-side filter handling. |
| Idempotency | Repeated `set_config` calls must not leave stale VPP streams, stale FRR peers, or duplicate LAG state. |

### Performance Requirements

| Item | Target |
|---|---|
| TX scaling | Streams pinned to VPP workers; independent workers scale close to linearly until CPU or interface backend saturates. |
| RX scaling | RX measurement runs on workers that receive packets on the configured RX interface. |
| vhost-user path | Optimised for vlab correctness and practical throughput, not hardware timestamp precision. |
| DPDK path | Primary physical-DUT backend for higher throughput and lower software overhead. |
| Latency histogram | 1 microsecond bucket width with overflow bucket. |
| Metrics collection | Bulk flow stats preferred over per-flow polling to minimise API overhead. |

### Design Scope

The design has multiple workstreams that must fit together for the system to behave like a useful traffic generator.

| Workstream | Required work | Why it is needed |
|---|---|---|
| VPP traffic-generation dataplane | Rework and extend VPP packet-generation capability into an OTG-facing traffic-generation dataplane for external tester ports. VPP already has packet-generator primitives that can inject basic packets into the graph; VPP-OTG needs stream objects that transmit through DPDK/vhost-user/af_packet interfaces, generate at configured rates, stamp timestamps and stream markers, measure RX, collect latency/jitter/loss, track flow RX, count pause/PFC frames, and expose binary APIs. | Existing VPP packet generation will be used as a foundation,SONiC traffic tests need a tester dataplane that drives external NIC-backed ports and reports traffic-generator metrics. The required work is to remodel the generation path around OTG streams, physical/virtual interface backends, worker scaling, RX measurement, and stable control/stat APIs. |
| VPP interface and backend support | Support DPDK as the primary physical-DUT backend, vhost-user for sonic-vpp virtual testbeds, af_packet for fallback, and future NIC/SmartNIC offload integration. | The same OTG model must run against physical DUTs, virtual DUTs, and developer/lab environments with different dataplane attachment methods. |
| OTG/Snappi adapter | Translate OTG config/control/metrics/capture/state APIs into VPP and FRR operations; own validation, reconciliation, capability reporting, error handling, and idempotency. | `sonic-mgmt` tests speak Snappi/OTG, not VPP PAPI or FRR CLI. |
| FRR control-plane integration | Render and control routed peer state, primarily BGP first, and expose protocol metrics/states in OTG format. | Many Snappi tests require the traffic generator to behave as routed peers, not only as packet ports. |
| LAG/LACP integration | Map OTG LAG objects to VPP BondEthernet and expose LACP state/metrics. | SONiC tests validate LAG behavior and traffic over PortChannels. |
| Packaging and process model | Package VPP, the traffic-generation plugin, FRR, the adapter, and supervisor config into `docker-vpp-otg`. | Deployable unit with clear process ownership and health behavior. |
| Testbed integration | Wire `docker-vpp-otg` into sonic-vpp virtual testbeds, physical-DUT topologies, connection graphs, and CI/CD runner workflows. | sonic-mgmt should be able to deploy and drive it consistently. |
| Validation and conformance | Provide unit, integration, OTG capability, and sonic-mgmt regression coverage. | Make sure that the endpoint behaves predictably as OTG coverage grows. |

---

## High Level Design

### Architecture Overview

#### Current External-TGen Topology

Today, Snappi tests commonly treat a hardware traffic-generator chassis as an external appliance. The SONiC test runner drives it through the Snappi/OTG API, and the chassis ports connect to DUT front-panel ports.

```
                        +---------------+
                        | Test runner   |
                        | sonic-mgmt    |
                        +-------+-------+
                                |
                                | Snappi / OTG REST
                                v
                    +-----------------------+
                    | External TGen chassis |
                    | hardware test ports   |
                    +---+-------+-------+---+
                        |       |       |
                        |       |       | physical links
                        v       v       v
                    +-----------------------+
                    | SONiC DUT             |
                    | front-panel ports     |
                    +-----------------------+
```


#### Proposed VPP-OTG Topology

Snappi-VPP keeps the same Snappi/OTG northbound API and replaces the external chassis with a server running VPP-OTG. The server can attach to a sonic-vpp virtual DUT through vhost-user or to a physical DUT through NIC ports. As NIC and SmartNIC offloads improve, the same topology can use more hardware assistance without changing the Snappi-facing model.

```
                        +---------------+
                        | Test runner   |
                        | sonic-mgmt    |
                        +-------+-------+
                                |
                                | Snappi / OTG REST
                                v
        +------------------------------------------------+
        | VPP-OTG server / docker-vpp-otg               |
        |                                                |
        |  OTG adapter                                  |
        |    -> VPP flowgen dataplane                   |
        |    -> FRR protocol emulation                  |
        |    -> NIC / SmartNIC offload path over time   |
        +-------------+----------------------+-----------+
                      |                      |
                      | vhost-user           | physical NIC links
                      | virtual wiring       | through DPDK
                      v                      v
              +---------------+      +---------------+
              | sonic-vpp DUT |      | Physical DUT  |
              | dev / CI      |      | lab / CI      |
              +---------------+      +---------------+
```

#### System View

```
sonic-mgmt pytest
  |
  | Snappi / OTG REST (port 8443)
  v
+-------------------------------------------------------------+
| docker-vpp-otg                                              |
|                                                             |
|  +----------------------+    +----------------------------+ |
|  | snappi-vpp adapter   |--->| VPP driver (PAPI)          | |
|  | config / control /   |    | interfaces, flowgen, stats | |
|  | metrics / capture    |    +-------------+--------------+ |
|  +----------+-----------+                  |                |
|             |                              v                |
|             |                 +--------------------------+  |
|             |                 | VPP + flowgen plugin     |  |
|             |                 | TX, RX measure, capture  |  |
|             |                 +------------+-------------+  |
|             |                              |                |
|             v                              |                |
|  +----------------------+       control-plane interfaces    |
|  | FRR driver / FRR     |<---- TAP / host-if / Linux view   |
|  | BGP / OSPF / ISIS    |      of DUT-facing VPP ports      |
|  +----------------------+                                   |
|                                                             |
|  +----------------------+                                   |
|  | LAG / LACP           | VPP BondEthernet + LACP plugin    |
|  +----------------------+                                   |
+-----------------------------+-------------------------------+
                              |
                              | DPDK / vhost-user / af_packet
                              v
                         SONiC DUT or DUT VM
```

#### Component Diagram

```
+-------------------+       +----------------------+
| Snappi client     |       | arsonic / testbed    |
| set_config        |       | deployment plumbing  |
| control_state     |       +----------+-----------+
| get_metrics       |                  |
+---------+---------+                  |
          |                            |
          v                            v
+----------------------------------------------------+
| snappi-vpp adapter                                 |
|                                                    |
|  API layer                                         |
|    - request routing                               |
|    - state lock                                    |
|    - health/version/config                         |
|                                                    |
|  Translation and reconcile                         |
|    - OTG config to internal model                  |
|    - diff current/desired state                    |
|    - apply ports, LAGs, L3, protocols, flows       |
|                                                    |
|  Operations                                        |
|    - traffic start/stop                            |
|    - protocol state and route control              |
|    - metrics/state/capture/link operations         |
|                                                    |
|  Drivers                                           |
|    - VPP PAPI driver                               |
|    - FRR vtysh driver                              |
|    - supervisor/sidecar driver                     |
+--------------------+-------------------------------+
                     |
                     v
+----------------------------------------------------+
| Runtime backends                                   |
|                                                    |
|  VPP                                               |
|    - DPDK / vhost-user / af_packet ports           |
|    - BondEthernet LAGs                             |
|    - flowgen plugin                                |
|    - pcap trace capture                            |
|                                                    |
|  FRR                                               |
|    - bgpd peers and route ranges                   |
|    - protocol state and metrics source             |
|                                                    |
|  Linux kernel                                      |
|    - TAP view for control-plane peering            |
|    - bond status where needed                      |
+----------------------------------------------------+
```

The design principle is that each component owns its natural domain: VPP owns dataplane behaviour, FRR owns routed control-plane behaviour, and the adapter owns OTG translation and orchestration. Protocol-specific behaviour is added under the relevant component through a protocol adapter contract, not as cross-cutting branches.

**Component ownership boundaries:**

| Component | Owns |
|---|---|
| VPP traffic generator | Packet generation, rate control, RX measurement, stream counters, latency/jitter, flow tracking, traffic-generator binary APIs. |
| VPP interface and backend support | DPDK, vhost-user, af_packet, interface naming, link state, L3 binding, capture primitives, future NIC/SmartNIC offload attachment. |
| OTG/Snappi adapter | Northbound OTG API, request routing, validation, internal state, translation, reconciliation, backend driver calls, metrics normalization. |
| FRR | Routed peer emulation, initially BGP peers and route ranges. |
| Other protocol or daemon integrations | LACP first; future MACsec, LLDP, DHCP, 802.1X, or other daemon-backed protocol support. |

**Adapter communication architecture:**

The adapter communicates with VPP and FRR through different IPC mechanisms suited to each backend's design. All communication is local to the container -- there is no network-based management plane between the adapter and its backends.

```
+---------------------------------------------------------------+
| docker-vpp-otg                                                |
|                                                               |
|  +-------------------+                                        |
|  | OTG adapter       |                                        |
|  | (Python, port     |                                        |
|  |  8443)            |                                        |
|  +----+---------+----+                                        |
|       |         |                                             |
|       |         +------ vtysh subprocess ----+                |
|       |         |       (stdin pipe)         |                |
|       |         |                            v                |
|       |         |                  +-------------------+      |
|       |         |                  | FRR daemons       |      |
|       |         |                  | bgpd, zebra, ...  |      |
|       |         |                  +-------------------+      |
|       |                                                       |
|       +--- PAPI binary API ---+                               |
|       |    (Unix socket)      |                               |
|       |                       v                               |
|       |             +-------------------+                     |
|       |             | VPP               |                     |
|       |             | flowgen, bonds,   |                     |
|       |             | interfaces, stats |                     |
|       |             +-------------------+                     |
|       |                       ^                               |
|       +--- Stats segment ----+                                |
|            (shared memory)                                    |
+---------------------------------------------------------------+
```

*Adapter to VPP.* The adapter uses two channels to communicate with VPP, both operating over Unix domain sockets on the local filesystem:

| Channel | Transport | Purpose |
|---|---|---|
| Binary API (PAPI) | Unix domain socket | All control operations: stream lifecycle, interface creation, link state, bond management, IP address assignment, capture control. The adapter issues typed RPC calls and receives structured responses. |
| Stats segment | Shared memory | High-performance counter reads. The adapter maps the stats segment into its own process memory and reads per-interface TX/RX packet and byte counters directly, aggregated across all VPP workers. No RPC round-trip per counter read. |

A small number of operations that lack binary API equivalents use CLI-over-PAPI, where the adapter sends a CLI command string through the binary API channel and receives the text output. These include L2 cross-connect wiring for vhost-user deployments, ARP/ND probing via ping, and pcap capture control.

The adapter connects to VPP on the first OTG API call with retry logic (up to 30 attempts at one-second intervals) to tolerate VPP startup time. On successful connection it clears any stale streams left from a previous test run.

*Adapter to FRR.* The adapter communicates with FRR exclusively through the vtysh command-line interface, invoked as a subprocess. There are two interaction patterns:

| Pattern | When used | Mechanism |
|---|---|---|
| Config render and reload | On `set_config` when protocol state changes | The adapter renders daemon-specific configuration from Jinja2 templates (one template per protocol), writes the result to the daemon's config file, and pipes the configuration lines into vtysh via stdin. Stdin piping avoids shell argument-length limits when advertising large route tables (thousands of prefixes). The daemon applies the configuration live without a restart. |
| Targeted runtime commands | On route withdraw/advertise, neighbor control | The adapter sends individual vtysh commands to modify specific state. For route withdrawal, this means toggling a per-peer outbound prefix-list from permit to deny and issuing a soft clear to trigger UPDATE/WITHDRAW messages. No full config re-render is needed. |

Protocol metrics are collected by running vtysh queries that return JSON output (for example, BGP neighbor state), which the adapter parses and normalises into OTG response format.

FRR configuration is idempotent: the adapter computes a signature of the rendered configuration and skips the reload if the signature matches the previous apply. This avoids unnecessary bgpd churn when repeated `set_config` calls do not change protocol intent.

*Process startup and coordination.* All three runtime components start inside the container under supervisord, with staggered priorities to ensure dependencies are ready before dependents initialise:

```
supervisord
  |
  |-- priority 10:   VPP             starts first, opens API and stats sockets
  |-- priority 20:   FRR zebra       waits for VPP, provides kernel/netlink integration
  |-- priority 25:   FRR daemons     bgpd, isisd, ospfd, ospf6d, staticd
  |-- priority 100:  OTG adapter     starts last, connects to VPP and FRR on first API call
```

VPP and FRR daemons are configured for automatic restart on crash. The adapter performs lazy initialisation: it does not attempt to connect to VPP or FRR until the first OTG request arrives, which allows supervisord to bring up the backends at their own pace. Health checks verify that the VPP API socket and the FRR zebra socket are present on the filesystem.

### Deployment Models

Snappi-VPP supports three deployment models.  All three produce the same VPP `sw_if_index` and use the same adapter, flowgen, metrics, and protocol code paths.

#### Physical DUT (DPDK)

The VPP-OTG server replaces a hardware chassis. NIC ports are bound to DPDK (`vfio-pci`) and VPP owns them directly via PCIe DMA. Physical cables connect to DUT front-panel ports. No kernel in the data path.

```
VPP-OTG server                          Physical SONiC switch
  VPP (DPDK)                               DUT
  NIC port 0000:06:00.0  ── fibre ──►  Ethernet0
  NIC port 0000:06:00.1  ── fibre ──►  Ethernet4
```

- Highest throughput (10-100+ Mpps).
- Ports declared in VPP `startup.conf`; adapter finds them at `set_config` time.
- Requires DPDK-compatible NICs (Mellanox CX-5/6, Intel X710/E810) and hugepages.

#### Virtual DUT with vhost-user

VPP creates Unix sockets; the QEMU DUT VM connects as a vhost-user client. Zero-copy shared-memory data path.

```
docker-vpp-otg                          QEMU DUT VM
  VPP (vhost-user plugin)                SONiC-VPP
  VirtualEthernet0/0/0  ── socket ──►  virtio Ethernet0
  VirtualEthernet0/0/1  ── socket ──►  virtio Ethernet4
```

- ~10-14 Mpps. No kernel in data path.
- Requires hugepages for both VPP and QEMU.
- Wire script pre-creates TAP interfaces with L2 xconnects for FRR control plane.
- Selected via `TGEN_PORT_DRIVER=vhost_user`.

#### Virtual DUT with af_packet (PoC / CI)

VPP attaches to veth pairs via `AF_PACKET` raw sockets. The other end of each veth plugs into an OVS bridge reaching the DUT.

```
docker-vpp-otg             Host         DUT (container/VM)
  VPP (af_packet)           OVS           SONiC
  host-veth-otg0  ──veth──► bridge ──►  Ethernet0
  host-veth-otg1  ──veth──► bridge ──►  Ethernet4
```

- ~1-2 Mpps. Every packet crosses the kernel twice.
- Simplest to set up; no hugepages required for data path.
- Default mode (`TGEN_PORT_DRIVER=af_packet`).

### Request Processing Model

Every OTG request follows the same path regardless of deployment model:

```
Snappi client
  |
  | HTTP POST /config, /control/state, /results/metrics, ...
  v
HTTP server (server.py)
  |
  | normalise path, parse JSON body
  v
Api router (api.py)
  |
  | acquire _state_lock (for writes)
  v
+----- set_config -----+---- control_state ----+---- get_metrics ----+
|                       |                       |                     |
| 1. Validate           | Route by choice:      | Route by choice:   |
| 2. Translate to       |   traffic -> TrafficOps|  port -> port ctrs |
|    InternalConfig     |   protocol -> Proto    |  flow -> flow ctrs |
| 3. Plan (diff)        |   port -> LinkOps      |  bgpv4 -> FRR     |
| 4. Apply (ordered)    |        or CaptureOps  |  lacp -> bond     |
| 5. Commit             |                       |  convergence ->   |
|                       |                       |    event log      |
+-----------------------+-----------------------+-------------------+
```

---

## Component Design

### OTG Adapter

The adapter is the northbound component seen by `sonic-mgmt`. It accepts OTG requests, validates them, translates them to an internal model, reconciles backend state, and returns OTG-shaped responses. It does not generate packets and does not implement routing protocols.

**Subcomponents:**

| Subcomponent | Responsibility |
|---|---|
| API router | Route OTG operations (config, control, metrics, states, capture, version, health). Hold current accepted state and serialise writers. |
| Validator | Reject missing references, unsupported choices, and invalid ranges before touching any backend. |
| Translator | Convert nested OTG JSON into flat internal models (ports, devices, flows, LAGs, BGP peers, route ranges, captures). |
| Packet template builder | Use Scapy to craft wire-format packet bytes from OTG headers. Compute per-field byte offsets for runtime field variation. |
| Reconcile engine | Diff current state against desired state. Emit ordered apply steps (ports, LAGs, L3, sidecars, protocols, flows). Roll back on failure. |
| Protocol registry | Central registry of protocol adapters. Adding a new protocol requires one adapter module and one registry entry. |
| Backend drivers | Hide VPP PAPI, FRR vtysh, and supervisord details from the adapter core logic. |
| Metrics coordinator | Fan out metric requests to VPP, FRR, and protocol adapters. Compute rates, noise floor, and loss. Normalise into OTG responses. |
| Traffic ops | Two-pass baseline-then-enable for flow start. Flow stop and auto-stop on duration expiry. |

**Request routing:**

| OTG request | Adapter processing | Backend |
|---|---|---|
| `set_config` | Validate, translate, reconcile (ordered apply) | **Ports:** VPP interface layer; **LAGs:** VPP BondEthernet; **L3:** VPP interface layer; **Protocols:** FRR ; **Flows:** VPP flowgen |
| Traffic control | Route start/stop to selected flows | VPP flowgen (flowgen_enable / flowgen_disable) |
| Protocol control | Route start/stop/withdraw/advertise to protocol adapter | FRR (neighbor shutdown/activate, prefix-list toggle) |
| Port control | Route link up/down or capture start/stop | VPP interface layer (set_flags) or VPP capture (pcap trace) |
| Metrics / states | Fan out to relevant backends, normalise to OTG response | VPP (stream stats, port counters), FRR (BGP state), LACP (bond state) |
| Capture readback | Read pcap file, apply OTG filters | VPP capture pipeline (pcap dispatch trace output) |
| Version / capability | Return supported features and backend health | Adapter protocol registry and VPP/FRR health checks |

**Internal state model:**

Internal state is in-memory for the active test run. The durable intent is the OTG config supplied by the test. Runtime metadata includes mappings such as logical port to VPP interface, flow name to VPP stream id, capture file path, counter baselines, and protocol object mappings.

| Internal object | Purpose |
|---|---|
| Port | Logical Snappi port and backend location. |
| Layer1 | Port speed and settings used for rate conversion and link behavior. |
| Device | Ethernet/IP/protocol identity bound to a port or LAG. |
| LAG | LAG name, member ports, LACP mode, actor settings, and member identity. |
| Flow | TX/RX references, packet headers, rate, duration, burst settings, size, and measurement requirements. |
| Capture | Capture name, ports, filters, and output ownership. |
| Runtime metadata | VPP stream ids (`applied_stream_indices`), interface aliases, sidecar state, capture files (`applied_capture_files`), and counter baselines (`_flow_runtime`). |
| `_last_config` | Stashed OTG dict for `get_config` echo-back. |

**Packet template construction:** The adapter uses Scapy to turn OTG Ethernet/IP/TCP/UDP/VLAN/pause/PFC/custom header intent into packet bytes and header offset metadata. VPP then owns high-rate packet generation from those templates. This keeps packet crafting expressive in Python while keeping runtime generation, rate control, timestamping, and RX measurement inside VPP.

**Alias resolution:** Snappi flows can refer to port names, device names, Ethernet names, IP interface names, BGP peer names, route-range names, or LAG names. The adapter resolves all of these to the component that must act on the request.

**Example request flow:** One IPv4 port-to-port stream at 100 000 pps.

```
  set_config (ports p1/p2, devices tx/rx, flow f1: Eth+IPv4, 100kpps, 128B)
  |
  v
+-----------+     +-----------+     +------------------+
| Validate  |---->| Translate |---->| Reconcile (plan) |
| refs,     |     | OTG JSON  |     | diff current vs  |
| ranges,   |     | to flat   |     | desired; emit    |
| fields    |     | internal  |     | ordered steps    |
+-----------+     | models    |     +--------+---------+
                  +-----------+              |
                                             v
                  +----------------------------------------------+
                  | Apply (sequential, with rollback on failure) |
                  |                                              |
                  |  1. Bind ports    VPP: af_packet / vhost /   |
                  |                        DPDK lookup + UP      |
                  |  2. Assign L3     VPP: VRF + IP addr +       |
                  |                        ARP probe gateway     |
                  |  3. Build packet  Scapy: Eth/IPv4 -> bytes   |
                  |     template           + offset map          |
                  |  4. Create stream VPP: flowgen_create        |
                  |     (disabled)         (rate, size, mode)    |
                  +----------------------------------------------+
                                             |
                                             v
                  +----------------------------------------------+
                  | Commit: _current = applied config            |
                  | Return {"status": "success"}                 |
                  +----------------------------------------------+

  traffic START
  |
  v
+------------------+     +------------------+
| Pass 1: Baseline |---->| Pass 2: Enable   |
| snapshot TX/RX   |     | flowgen_enable   |
| port counters    |     | per flow         |
| BEFORE enable    |     | (VPP starts TX)  |
+------------------+     +------------------+

  get_metrics (flow)
  |
  v
+-------------+     +-------------+     +--------------+     +----------+
| Auto-stop   |---->| Read VPP    |---->| Compute rate |---->| Return   |
| check       |     | stream +    |     | (sliding     |     | OTG flow |
| (duration   |     | port ctrs,  |     |  window),    |     | metrics  |
|  expired?)  |     | subtract    |     | noise floor, |     |          |
|             |     | baselines   |     | loss %       |     |          |
+-------------+     +-------------+     +--------------+     +----------+

  traffic STOP
  |
  v
+------------------+
| flowgen_disable  |
| counters freeze  |
| state = stopped  |
+------------------+
```
**Idempotency guarantees:**

| Operation | Behaviour |
|---|---|
| Repeat same `set_config` | No backend change (diff is empty). |
| Start already-started flow | No-op. |
| Stop already-stopped flow | No-op. |
| Link up when already up | No-op. |
| Capture start while already active | Clear error unless multi-capture support is added. |

**Error model:**

| Failure | HTTP | Behaviour |
|---|---|---|
| Validation error | 400 | No backend touched. |
| Backend error | 502 | Infrastructure failure; retry safe. |
| Apply failure | 502 | Attempt inverse operations for steps already applied; keep previous current state if rollback succeeds. |
| Rollback failure | 502 | Mark health degraded and report diagnostics for testbed cleanup. |
| Backend timeout | 502 | Return backend error or documented partial metrics where the OTG operation allows it. |
| Unimplemented | 501 | Feature explicitly unsupported. |

### VPP Traffic Generator

The flowgen plugin is the core dataplane component that makes VPP behave as a traffic generator. It reworks and extends VPP packet-generation capability into an OTG-facing stream engine that can transmit through external tester ports, measure received packets, and expose traffic-generator counters. It runs entirely in C inside VPP's graph processing loop.

VPP already has packet-generator primitives that can inject basic packets into the graph. VPP-OTG needs following additional behavior for SONiC traffic tests:

| Area | Required behavior |
|---|---|
| Stream lifecycle | Create, delete, enable, disable, and clear named streams. |
| External TX path | Transmit through DPDK, vhost-user, or af_packet interfaces rather than only injecting local graph packets. |
| Rate control | Support pps, bps, percent-line-rate after adapter conversion, fixed-packet, fixed-duration, continuous, and burst semantics. |
| Packet template | Use wire-format packet templates derived from OTG flow headers and generated device-based flows. |
| Field variation | Mutate configured packet fields using offset/width/mask descriptors generated by the adapter. |
| Timestamping | Stamp packets with enough metadata to attribute RX packets to streams and compute latency. |
| RX measurement | Classify received packets, update RX counters, compute latency/jitter, and record sequence/out-of-order information. |
| Flow tracking | Record first/last RX observations for convergence and failover tests. |
| Pause/PFC | Generate pause/PFC frames and count received pause/PFC frames where tests require it. |
| API and stats | Expose stable VPP binary APIs for stream lifecycle, bulk stats, RX flow tracking, and PFC counters. |

**Dataplane graph:**

```
TX path

flowgen-input
  |
  | allocate buffers, copy packet template,
  | apply field variations, set TX interface
  v
flowgen-timestamp
  |
  | stamp timestamp, stream id, sequence marker
  v
interface-output
  |
  v
DPDK / vhost-user / af_packet interface


RX measurement path

interface input
  |
  v
flowgen-measure feature node
  |
  | find marker, recover stream id, compute latency,
  | update counters, optionally update flow tracking
  v
normal VPP feature arc
```

Each OTG flow becomes a VPP stream. The adapter creates the stream, but VPP owns the runtime behavior after the stream is created.

**Stream field groups:**

| Stream field group | Purpose |
|---|---|
| Identity | Stream name and stream index. |
| Interfaces | TX interface, optional RX measurement interface, output node, and worker assignment. |
| Template | Packet bytes, packet size model, protocol mode, and header offsets. |
| Rate | Target packets per second and accumulator state. |
| Duration | Packet limit, burst state, and enable/disable state. |
| Variation | Offset, width, mask, and mutation behavior for variable fields. |
| Counters | TX/RX packets and bytes, latency, jitter, sequence, loss, out-of-order, and histogram state. |
| Pause/PFC | Pause frame parameters and per-priority receive counters. |


**Traffic generation model:**

The flowgen plugin supports two independent and orthogonal axes of configuration: the **traffic pattern** (how packets are paced and when the stream stops) and the **flow identity** (what 5-tuple each packet carries). Any traffic pattern can be combined with any flow identity mode.

*Traffic patterns (rate and duration axis).* Five traffic patterns are supported. The first four correspond directly to OTG duration choices; the fifth is a specialised mode for pause frame injection.

| Traffic pattern | OTG duration choice | Pacing | Stop condition |
|---|---|---|---|
| Continuous rate | continuous | Packet accumulator at configured PPS | Runs until the adapter explicitly disables the stream. |
| Fixed packet count | fixed_packets | Packet accumulator at configured PPS | Input node auto-disables the stream when the running packet counter reaches the configured limit. |
| Fixed seconds | fixed_seconds | Packet accumulator at configured PPS | The adapter monitors wall-clock elapsed time during metrics collection and disables the stream when the configured duration expires. |
| Burst | burst | Wire-speed emission of M packets per burst, inter-burst gap, repeat N times | Auto-disables after the last burst completes. If N is zero, bursts repeat indefinitely until explicitly stopped. |
| PFC / 802.3x pause | continuous (with PFC stream mode) | Packet accumulator at configured PPS | Same as continuous. Template is a raw 64-byte pause or PFC frame copied verbatim with no IP fixup or checksum recomputation. |

All rate modes defined by OTG are accepted by the adapter: packets per second, bits per second (and kilobits, megabits, gigabits variants), and percentage of line rate. The adapter converts all of these to packets per second before passing the value to the plugin, using the average packet size and the TX port's configured line rate for the conversion.

*Flow identity (what 5-tuple each packet carries).* Three modes control how the plugin assigns identity to generated packets. These are independent of the traffic pattern above.

| Flow identity mode | Configuration | Behavior |
|---|---|---|
| Single static flow | Default (no field variation, no flow lifecycle) | Every packet carries the same 5-tuple from the packet template. Most sonic-mgmt tests use this mode today. |
| Field variation | Adapter provides per-field offset, width, mask, and mutation descriptors | The plugin mutates configured packet fields on each packet using increment, decrement, or random patterns. Supported fields include DSCP, ECN, TTL, source and destination ports, source and destination IP addresses, MAC addresses, and arbitrary byte offsets. From the DUT's perspective these appear as many distinct flows, but the plugin treats the stream as a single object with per-packet field mutations. PFC, ECN, and QoS tests use this mode to sweep priority and marking fields. |
| Flow lifecycle | New flows per second > 0, with configured flow duration | The plugin maintains a pool of active flows, each with a distinct 5-tuple and an expiry timestamp. New flows arrive at a configured rate and expire after a configured duration. Each packet is assigned to a randomly selected active flow. This is the most realistic model, simulating thousands of concurrent connections that come and go. Arrival distribution can be uniform or exponential; duration distribution can be uniform or Pareto (heavy-tailed). |

*Combined capabilities.* Any traffic pattern can pair with any flow identity mode:

```
Traffic pattern × Flow identity combinations

                     Single static     Field variation     Flow lifecycle
                   +------------------+-------------------+--------------------+
  Continuous rate  | Basic port-to-   | ECMP hash spread, | Realistic ongoing  |
                   | port traffic     | DSCP/ECN sweep    | traffic simulation |
                   +------------------+-------------------+--------------------+
  Fixed packets    | Exact count      | Counted packets   | N packets across   |
                   | validation       | with varying      | dynamic flows      |
                   |                  | headers           |                    |
                   +------------------+-------------------+--------------------+
  Fixed seconds    | Timed test run   | Timed run with    | Realistic timed    |
                   |                  | field variation   | workload           |
                   +------------------+-------------------+--------------------+
  Burst            | Burst stress     | Burst with        | Burst across       |
                   | testing          | varying fields    | live flows         |
                   +------------------+-------------------+--------------------+
  PFC / pause      | Pause frame      |        N/A        |        N/A         |
                   | injection        | (raw frame copy)  | (raw frame copy)   |
                   +------------------+-------------------+--------------------+
```

**Rate and duration enforcement:**

Rate and duration are enforced in the dataplane where possible so that the adapter is not in the hot path for pacing or stopping traffic.

*Rate pacing -- packet accumulator algorithm.* Continuous-rate streams use a floating-point packet accumulator. On every main-loop iteration the input node computes the elapsed time since the last generation call, multiplies it by the configured packets-per-second target, and adds the result to a running accumulator. The integer part of the accumulator is the number of packets to emit this iteration; the fractional remainder carries forward to the next cycle. Each iteration is capped at one VPP frame (256 packets) to bound per-call work. This design is self-correcting: fractional credits accumulate across iterations, so the long-term average converges precisely to the target rate regardless of how frequently VPP polls the input node. No token-bucket data structure is needed -- a single floating-point counter per stream is sufficient.

```
Rate pacing (per main-loop iteration)

  elapsed = now - last_generate_time
  accumulator += elapsed * target_pps
  packets_this_iteration = floor(accumulator)
  accumulator -= packets_this_iteration          (carry fraction forward)
  packets_this_iteration = min(packets_this_iteration, 256)
  last_generate_time = now
```

*Burst mode.* When the stream uses burst arrival distribution, rate pacing is bypassed. The input node emits a configured number of packets back-to-back at wire speed (one burst), then enforces a configurable inter-burst gap by recording the next allowed burst time. After the gap elapses, the next burst fires. If a finite burst count is configured, the stream auto-disables after the last burst completes. If the burst count is zero, bursts repeat indefinitely until the stream is explicitly stopped.

*Duration enforcement.* Four duration modes are supported, each enforced at a different layer:

| Duration mode | Enforcement | Mechanism |
|---|---|---|
| Continuous | No limit | Stream runs until explicitly disabled by the adapter. |
| Fixed packet count | Dataplane | The input node checks a running packet counter against the configured limit before each generation cycle. When the limit is reached, the stream atomically clears its enabled flag and stops. |
| Fixed seconds | Adapter | The adapter monitors wall-clock elapsed time during metrics polling and disables the stream when the configured duration expires. |
| Burst (N bursts of M packets) | Dataplane | The input node tracks remaining bursts and remaining packets within the current burst. After all bursts complete, the stream auto-disables. |

*Per-flow lifecycle and data structures.* When flow-aware generation is configured (new flows per second > 0), the plugin maintains a pool of active flows. Each flow carries a 5-tuple identity, creation timestamp, and expiry timestamp. Two data structures work together:

```
Flow lifecycle data structures

+--------------------------------------------+
| Active flow index vector (dense, unordered) |
|  [ 3, 7, 1, 12, 5 ]                        |
|    |   |   |   |    |                       |
|    v   v   v   v    v                       |
| Flow pool (sparse, free-list managed)       |
|  [0][1][2][3][ ][5][ ][7]...[12]            |
|       |       |   |       |    |            |
|       v       v   v       v    v            |
|     flow    flow flow   flow  flow          |
|   (5-tuple, start_time, end_time, counters) |
+--------------------------------------------+
```

The **flow pool** is a contiguous array with an embedded free-list bitmap. Allocating a new flow slot and returning an expired slot are both constant-time operations, and lookup by pool index is direct array access. The pool owns the flow state, including each flow's expiry timestamp.

The **active flow index vector** is a dense dynamic array of pool indices. It exists because the pool itself is sparse (freed slots leave gaps). The vector provides two capabilities the pool alone cannot: random flow selection for round-robin packet assignment (pick a random vector position in constant time), and fast unordered removal on expiry (swap the expired entry with the last element and shrink by one).

*Flow creation.* New flows arrive at a configured rate. On each input-node call, if the current time has passed the next scheduled arrival, a flow is allocated from the pool with a randomised 5-tuple and an expiry timestamp. The expiry duration follows one of two distributions: uniform (all flows live exactly the configured duration) or Pareto with shape 1.5 (heavy-tailed: most flows are short-lived, a few persist much longer). The next arrival time advances by the inter-arrival interval, which can itself follow a uniform or exponential distribution.

*Flow expiry.* On each input-node call, the expire routine sweeps the active index vector in reverse order, checking each flow's expiry timestamp against the current time. Expired flows are returned to the pool and removed from the vector via swap-and-pop. Reverse iteration ensures that the swap-and-pop removal does not skip entries. This is a linear scan -- O(n) per call where n is the number of active flows -- which trades algorithmic efficiency for simplicity and cache-friendly memory access during high-rate packet generation.

*Interaction between rate, duration, and flow lifecycle.* These mechanisms compose in a fixed order within the input node:

```
Input node processing order (per main-loop iteration)

1. CHECK PACKET LIMIT -----> stop stream if count reached
          |
2. DETERMINE PACKETS THIS ITERATION
       |-- BURST mode:  emit burst, enforce inter-burst gap
       +-- CONTINUOUS:  accumulator += elapsed * target_pps
          |
3. CREATE NEW FLOWS  -----> if arrival time has passed
          |
4. EXPIRE DEAD FLOWS -----> sweep for end_time <= now
          |
5. SELECT FLOW, BUILD PACKETS -----> round-robin from active flows
          |
6. INCREMENT COUNTERS
```

The packet limit is the outermost gate. Rate or burst mode controls how many packets are produced per iteration. Flow lifecycle determines which flows are available to carry those packets.


**TX path detail (flowgen-input node):**

| Step | Action |
|---|---|
| Rate control | Accumulate `dt * target_pps`, emit `floor(accumulator)` packets per iteration, preserve fractional remainder. |
| Buffer allocation | `vlib_buffer_alloc()` for the batch. |
| Template copy | Copy wire-format template bytes into each buffer. |
| Field variation | Apply per-field mutations (increment, decrement, random) at recorded byte offsets. |
| Size + checksum | Pad to configured size. Recompute IP total_length and checksum (mode-dependent). |
| Metadata stamp | Write TX timestamp and stream index into buffer opaque for RX measurement. |
| Enqueue | Hand buffers to interface-output for the configured TX sw_if_index. |
| Counters | Increment stream `tx_packets` and `tx_bytes`. |

**RX path (flowgen-measure node):**

A feature on the device-input arc of the configured RX interface. Non-intrusive -- all packets pass through to the next node unchanged.

| Step | Action |
|---|---|
| Identify stream | Read `stream_index` from buffer opaque metadata. |
| Latency | Compute `now - tx_timestamp` in nanoseconds. Update min/max/sum/count. |
| Counters | Increment stream `rx_packets` and `rx_bytes`. |
| Flow tracking | Record first/last RX timestamp per 5-tuple (used for convergence metrics). |

**VPP binary API surface:**

| API | Purpose |
|---|---|
| `flowgen_create` | Create a named stream with template, rate, size, mode, interfaces. Returns stream_index. |
| `flowgen_enable` / `flowgen_disable` | Start or stop a named stream. |
| `flowgen_delete` | Remove a named stream. |
| `flowgen_stats_dump` | Bulk read per-stream TX/RX, latency, jitter, enabled state. |
| `flowgen_rx_flows_dump` | Per-5-tuple first/last RX timestamps for convergence. |
| `flowgen_pfc_rx_dump` | Per-port per-priority PFC RX counters. |
| `flowgen_clear_stats` | Reset stream counters. |

**Stream modes:**

| Mode | Headers | Plugin recomputes |
|---|---|---|
| 0 | Ethernet only | Nothing (raw template copy). |
| 1 | Ethernet + IPv4 | `ip.total_length`, `ip.checksum`. |
| 2 | Ethernet + IPv4 + TCP | Above + `tcp.checksum`. |
| 3 | Ethernet + IPv6 | `ipv6.payload_length`. |
| 4 | Ethernet + IPv6 + TCP | Above + `tcp.checksum`. |
| 5 | PFC / 802.3x pause | Nothing (verbatim 64-byte frame copy). |

**Worker scaling:**

Worker scaling is part of the VPP traffic-generator component. Streams are assigned to VPP workers. Independent hot streams across independent workers should scale close to linearly until CPU, memory bandwidth, NIC queues, or the interface backend saturates.

| Deployment pattern | Expected behavior |
|---|---|
| One hot stream on one worker | Limited by one worker and the interface backend. |
| N hot streams across N workers | Scales close to linearly until host or NIC limits are reached. |
| Many streams on one worker | Worker packet rate is shared across streams. |
| vhost-user backend | Limited by QEMU/virtio and host scheduling. |
| DPDK backend | Primary path for physical-DUT performance. |

Flow metrics correctness is also owned here. `flowgen-measure` provides per-stream RX and latency when packets return on the configured RX interface. For ECMP and failover tests where traffic can return on multiple RX ports, the adapter may combine VPP port counters across RX ports to preserve aggregate loss/rate correctness, while per-flow latency attribution remains tied to the measured RX stream path.

### Interface Backend Layer

The interface backend connects the traffic generator to the DUT. It is separate from the stream engine: the stream engine decides what to transmit and measure; the interface layer decides how VPP ports attach to the DUT.

| Backend | VPP interface name | Created by | Use case |
|---|---|---|---|
| DPDK | `TenGigE6/0/0` (NIC-dependent) | VPP at startup via `dpdk {}` config | Physical DUT (primary performance path). |
| vhost-user | `VirtualEthernet0/0/N` | Adapter via PAPI `create_vhost_user_if` | sonic-vpp vlab (zero-copy shared memory), CI, Developer laptops. |
| af_packet | `host-veth-otgN` | Adapter via PAPI `af_packet_create_v3` | PoC, developer laptops (kernel-mediated). |
| NIC/SmartNIC offload | (NIC-dependent) | VPP DPDK with offload flags | Future rate, timestamp, checksum, steering, and scaling improvements. |

The interface layer exposes stable logical ports to the adapter. OTG port locations such as `<tgen-ip>;1;1` are resolved to VPP interfaces. The adapter should not need to know whether the resolved interface is a DPDK port, a vhost-user interface, or an af_packet host interface.

DPDK is the primary physical-DUT backend. vhost-user and af_packet are not the main physical-DUT performance path; they exist so the same architecture works for sonic-vpp virtual testbeds, developer laptops, and fallback setups.

The control-plane view is important for FRR. VPP owns the DUT-facing tester ports and L3 dataplane configuration. FRR does not communicate with the VPP traffic generator directly; FRR sources routed protocol sessions over the Linux/TAP/host-interface view associated with those DUT-facing ports.

**Interface resolution:** OTG tests identify ports using location strings -- either Ixia-style chassis addresses (`10.244.84.23;1;1`) or bare names (`veth-otg0`). VPP identifies interfaces by integer index. The adapter bridges this gap through a two-tier lookup.

*Alias overrides* are checked first. These are explicit name-to-index mappings registered by the adapter during reconciliation -- for example, when a LAG is created, the adapter pins the OTG LAG name to the BondEthernet index. Alias overrides are held in memory and survive interface cache refreshes.

*Candidate expansion* handles all other lookups. The adapter does not track which deployment backend is active. Instead, it extracts the port number from the location string and generates candidate VPP interface names for all backends, returning the first match found in VPP's interface table:

```
OTG location "10.244.84.23;1;1"  -->  port number 1 (1-based)

  Candidate: "host-veth-otg0"          matches if af_packet backend
  Candidate: "VirtualEthernet0/0/0"    matches if vhost-user backend
  Candidate: exact name from VPP dump  matches if DPDK backend
```

Only one candidate will match for any given deployment. This keeps the adapter backend-agnostic without requiring a deployment-mode configuration flag. If no candidate matches, the adapter refreshes the interface cache from VPP and retries once before raising an error.

**Additional interface layer responsibilities:**

| Capability | Implementation |
|---|---|
| Link state | `sw_interface_set_flags` via PAPI. |
| L3 binding | `sw_interface_add_del_address` + per-port VRF isolation via `ip_table_create`. |
| ARP/ND probing | `ping <gateway> repeat 1` CLI trick to trigger proactive neighbour resolution. |
| Capture | `pcap dispatch trace` CLI for port-level pcap. |
| Bond interfaces | `bond_create2` + `bond_add_member` for LACP LAGs (see LACP Integration). |
| Control-plane view | Linux/TAP/host-interface view where FRR or future daemons source protocol sessions. |

### FRR Integration

FRR provides routed peer emulation. It runs as a set of daemons (bgpd, zebra, isisd, ospfd, ospf6d, staticd) inside the same container, managed by supervisord. The adapter communicates with FRR exclusively through `vtysh` subprocesses.

FRR does not send traffic-generator data streams. VPP owns the traffic stream dataplane. FRR interacts with the DUT through the control-plane view of tester interfaces, while VPP owns the tester-facing ports and L3 dataplane setup.

**FRR responsibilities:**

| Area | FRR responsibility |
|---|---|
| BGP sessions | Establish IPv4/IPv6 BGP sessions with the DUT. |
| Route ranges | Advertise configured prefixes toward the DUT. |
| Route control | Support advertise/withdraw actions for selected peers or route ranges. |
| Protocol state | Provide session, neighbor, and route state used for OTG protocol metrics. |
| Routed peer behavior | Use FRR semantics and observability for routed protocol behavior. |

**BGP architecture:**

| Layer | Component | Responsibility |
|---|---|---|
| OTG config | `device.bgp` in set_config JSON | Declares peers, AS numbers, route ranges. |
| Translation | `InternalConfig.from_otg()` | Builds `BgpPeerCfg` / `BgpRouteRangeCfg` Pydantic models. Populates `device.aliases` for flow resolution. |
| Protocol adapter | `BgpAdapter` (protocols/bgp.py) | Caches per-device BGP state. Renders the union of all devices into a single bgpd.conf. Owns route withdrawal via prefix-lists. |
| FRR driver | `FrrDriver` (drivers/frr.py) | Template rendering (Jinja2), vtysh subprocess execution, JSON state parsing. |
| FRR daemon | `bgpd` | Actual BGP session establishment, OPEN/KEEPALIVE/UPDATE exchange, route advertisement. |

**Why one BGP instance for all devices?** FRR supports per-VRF BGP instances, but using separate VRFs would require creating Linux VRF devices, moving TAP interfaces into them, and aligning VPP and Linux VRF numbering on every config apply -- significant plumbing for no protocol-level benefit. BGP identifies peers by TCP connection, not by router-id, so multiple neighbors in a single `router bgp` block with different peer addresses on different interfaces are indistinguishable from separate instances to the DUT. The adapter therefore merges all devices' peers into one default-VRF BGP instance, caching each device's peers and re-rendering the union on each apply. Per-peer outbound prefix-lists preserve independent route control across peers.

**Route range expansion:** OTG route ranges (`{address, prefix, count, step}`) are expanded into individual FRR `network <cidr>` statements. With prefix=24 and step=1, the increment is `1 << (32-24) = 256`, giving consecutive /24s.

**Route withdrawal:** Uses per-peer outbound prefix-lists (not full config reload). `PL_PEER0_OUT` controls outbound filtering. Withdrawal sets the prefix-list to `deny any` and runs `clear bgp <peer> soft out` to trigger FRR to send WITHDRAW messages.

**BGP metrics:** `vtysh -c "show bgp neighbors json"` returns per-peer state. FRR's `bgpState` strings are normalised to OTG's `{up, down}` enum.

**Key FRR/BGP design points:**

| Area | Design |
|---|---|
| IPv4/IPv6 peers | Report per-address-family state in OTG metric responses. |
| Route ranges | Convert OTG route ranges into FRR advertised prefixes. |
| Route withdrawal | Isolate withdraw/advertise actions so one peer or range does not affect unrelated peers. |
| Idempotency | Avoid unnecessary FRR reloads when rendered BGP intent has not changed. |
| Convergence | Correlate FRR route events with VPP RX observations when tests request convergence metrics. |

### LACP Integration

LACP uses VPP's BondEthernet with `lacp_plugin.so`. VPP runs the full LACP state machine -- LACPDU exchange, partner detection, member activation/deactivation, and L3+L4 load balancing.

**LAG creation flow:**

| Step | Action | VPP PAPI |
|---|---|---|
| 1. Resolve members | Map OTG port names to VPP sw_if_indexes (already created by port apply). | `sw_if_index()` lookup |
| 2. Create bond | Create BondEthernet with LACP mode (5) and L3+L4 hash. | `bond_create2(mode=5, lb=1, id=N)` |
| 3. Register alias | Map LAG name to bond sw_if_index so L3/BGP/flow steps resolve it. | `register_alias("lag1", bond_idx)` |
| 4. Add members | Admin-down each member, add to bond, admin-up. | `bond_add_member()` |
| 5. Bond up | Bring bond interface up so LACPDUs flow. | `sw_interface_set_flags()` |
| 6. xconnect re-target | (vhost-user only) Re-wire TAP<->VE to TAP<->Bond so LACPDUs reach the bond LACP state machine. | `cli_inband("set interface l2 xconnect ...")` |
| 7. Wait for LACP | Poll `bond_info().active_members` until > 0, up to 30 seconds. | `sw_bond_interface_dump()` |

**Auto-LACP wrapper (`SNAPPI_VPP_AUTO_LACP=1`):** Wraps every raw port (not in an explicit LAG) in a 1-member passive LACP bond. This mirrors physical IXIA behaviour where ports auto-emit LACPDUs when the DUT has a PortChannel. Passive mode means the TGEN only responds to LACPDUs, avoiding double-initiation when DUT teamd runs active.

**LACP metrics:** Parsed from `vppctl show bond details`. Reports `oper_status`, `active_slaves`, `total_slaves`, `lacp_rate` per LAG.

For vhost-user deployments, the LACP path must ensure that data and LACPDUs traverse the bond rather than bypassing it through individual member wiring. The reconcile engine waits for active bond members when the test requires LAG readiness before traffic starts.

### Future Daemon-Backed Protocols

The same pattern applies to future SONiC-adjacent protocol support such as MACsec, LLDP, DHCP, or 802.1X. These should be added as protocol adapters with a backend driver or supervised sidecar contract. They should not add protocol-specific branches throughout the API router, flow translation, metrics formatting, or unrelated protocol components.

For a future MACsec-style component, the adapter contract would be:

| Responsibility | Extension point |
|---|---|
| Applicability | Detect MACsec objects or security settings in the OTG/internal model. |
| Validation | Check supported cipher suites, keys, replay settings, port/LAG bindings, and capability limits. |
| Apply/revert | Program the selected backend, which may be a VPP feature, Linux/driver facility, or supervised helper daemon. |
| Control state | Start, stop, rekey, or clear MACsec state through the protocol adapter. |
| Metrics/states | Normalize secure-channel, secure-association, packet, error, and replay counters into OTG-shaped responses. |
| Lifecycle | Declare whether an always-on daemon, optional sidecar, or VPP plugin is required. |

---

## Operational Workflows

### Configuration Workflow

```
Test script calls snappi.api.set_config(config)
  |
  v
1. VALIDATE (translation/validation.py)
   - Every flow has a name
   - Flow tx/rx ports exist in config.ports[]
   - Rate is non-negative
   - On failure: HTTP 400, no backend touched
  |
  v
2. TRANSLATE (translation/internal_model.py)
   - OTG JSON -> flat Pydantic models (PortCfg, DeviceCfg, FlowCfg, LagCfg, BgpCfg...)
   - device.aliases populated from BGP peer/route-range names (critical for flow resolution)
   - Rate normalised to PPS/BPS/PERCENT enum
   - Duration normalised to CONTINUOUS/FIXED_PACKETS/FIXED_SECONDS/BURST
  |
  v
3. PLAN (translation/reconcile.py)
   - Diff current config (empty on first call) vs desired
   - Emit ordered steps: ports -> LAGs -> L3 -> sidecars -> protocols -> flows
   - Each step has an inverse for rollback
  |
  v
4. APPLY (translation/reconcile.py)
   - Execute steps sequentially
   - Port: af_packet_create / vhost_user_create / DPDK lookup -> interface_set_state(up)
   - LAG: bond_create -> bond_add_member -> register_alias -> xconnect re-target
   - L3: ip_table_create (VRF) -> interface_set_table -> interface_set_l3_ipv4 -> ARP probe
   - Protocol: adapter.apply(device, vpp, frr, sidecars) -> e.g., render bgpd.conf
   - Flow: build_template_bytes (Scapy) -> resolve ports -> convert rate -> flowgen_create
   - On failure: walk applied steps backward with inverse operations
  |
  v
5. COMMIT
   - _current = applied config (with populated applied_stream_indices)
   - _last_config = original OTG dict
   - _wait_for_lacp (poll bond active_members up to 30s)
   - Return {"status": "success"}
```

### Traffic Lifecycle

```
Test calls set_control_state(traffic.flow_transmit.start, flow_names=["f1"])
  |
  v
PASS 1: BASELINE SNAPSHOT (before enabling any flow)
   For each flow:
   - Read TX port counters: interface_counters(tx_sw_if_index) via VPP stats segment
   - Read RX port counters: interface_counters(rx_sw_if_index) per rx_port
   - Read PFC RX baseline: flowgen_pfc_rx_dump(rx_sw_if_index)
   - Store in _flow_runtime[name] = {started_at, tx_baseline, rx_baselines, ...}
   WHY: Reading after enable causes phantom loss of ~(PAPI_RTT * pps) packets
  |
  v
PASS 2: ENABLE FLOWGEN
   For each baselined flow:
   - flowgen_enable(name) via VPP PAPI
   - VPP sets stream.enabled=1
   - flowgen-input node starts generating on next main-loop iteration
   - Runtime state: {state: "started", started_at: now}
  |
  v
Return {"status": "started"}

---

Test calls set_control_state(traffic.flow_transmit.stop, flow_names=["f1"])
  |
  v
DISABLE FLOWGEN
   - flowgen_disable(name) via VPP PAPI
   - VPP clears enabled bit; flowgen-input skips stream; counters freeze
   - Runtime state: {state: "stopped", stopped_at: now}
  |
  v
Return {"status": "stopped"}
```

### BGP Workflow

```
Test calls set_config with devices containing device.bgp
  |
  v
TRANSLATE BGP CONFIG
   - BgpCfg with router_id, peers[], each peer has peer_address, as_number, peer_as
   - BgpRouteRangeCfg with address, prefix, count, step
   - device.aliases populated with peer names + route-range names
  |
  v
RECONCILE PROTOCOL APPLY
   - BgpAdapter.applies_to(device) -> device.bgp is not None and has peers
   - BgpAdapter.apply(device, vpp, frr, sidecars):
     1. Sanitise router-id (must not collide with any peer_address)
     2. Cache device state in _device_state[device.name]
     3. _render_union(): collect all peers across all cached devices
     4. Expand route ranges: {addr, prefix, count, step} -> CIDR list
     5. Assign per-peer outbound prefix-lists (PL_PEER0_OUT, PL_PEER1_OUT, ...)
     6. Render bgpd.conf via Jinja2 template
     7. Idempotency check: skip reload if rendered signature unchanged
     8. FrrDriver.render_and_reload("bgpd", template, context):
        - Write to /etc/frr/bgpd.conf
        - Pipe through vtysh -d bgpd via stdin
  |
  v
FRR bgpd initiates TCP:179 connections to DUT, sends OPEN, KEEPALIVE, UPDATE

---

Test calls set_control_state(protocol.route.withdraw, names=["routes_peer1"])
  |
  v
ROUTE WITHDRAWAL
   - BgpAdapter.set_route_state(["routes_peer1"], "withdraw", frr):
     1. Toggle rr.withdrawn = True on matching route ranges
     2. Change affected peer's prefix-list: "ip prefix-list PL_PEER0_OUT seq 5 deny any"
     3. Push via vtysh stdin to bgpd
     4. "clear bgp <peer_addr> soft out" -> FRR sends WITHDRAW to DUT
   - EventLog records "route.withdraw" with timestamp (for convergence metrics)

---

Test calls set_control_state(protocol.route.advertise, names=["routes_peer1"])
  |
  v
ROUTE ADVERTISE
   - Same path, but rr.withdrawn = False
   - Prefix-list back to "permit any"
   - "clear bgp <peer_addr> soft out" -> FRR re-advertises prefixes
```

### LACP Workflow

```
Test calls set_config with config.lags[]
  |
  v
TRANSLATE LAG CONFIG
   - LagCfg with name, port_names, mode, actor_rate, actor_system_id, member_macs
   - lacpdu_periodic_time_interval: 1 -> fast, 30 -> slow
  |
  v
RECONCILE APPLY LAG (before L3, before protocols, before flows)
   1. Resolve member ports -> VPP sw_if_indexes
   2. bond_create2(mode=5/LACP, lb=1/L34, id=N, mac=actor_system_id)
   3. register_alias(lag_name, bond_sw_if_index)
   4. For each member:
      - sw_interface_set_flags(member, down)
      - bond_add_member(member, bond, is_passive, is_long_timeout)
      - sw_interface_set_flags(member, up)
   5. sw_interface_set_flags(bond, up)
   6. (vhost-user only) replace_xconnect_for_bond:
      - "set interface l3 <VirtualEthernet>" (remove from L2 xconnect)
      - "set interface l3 <TAP>" (remove from L2 xconnect)
      - "set interface l2 xconnect <TAP> <BondEthernet>"
      - "set interface l2 xconnect <BondEthernet> <TAP>"
  |
  v
WAIT FOR LACP CONVERGENCE
   - Poll bond_info(bond_sw_if_index).active_members every 1s
   - Up to 30s timeout
   - Log warning if LACP does not converge
  |
  v
SUBSEQUENT STEPS USE THE BOND
   - L3: IP address assigned to bond (resolved via alias)
   - BGP: FRR peers DUT over bond's TAP/linux-cp interface
   - Flow: flowgen_create targets bond's sw_if_index for TX
   - Metrics: counters read from bond's sw_if_index
```

### Metrics Workflow

```
Test calls get_metrics({choice: "flow", flow: {flow_names: ["f1"]}})
  |
  v
AUTO-STOP CHECK
   - For ALL flows (not just requested): check if configured duration expired
   - FIXED_SECONDS: elapsed >= value -> flowgen_disable
   - FIXED_PACKETS: elapsed >= (packets / rate) * 1.1 -> flowgen_disable
   - BURST: elapsed >= (M*N/rate + (N-1)*gap) * 1.3 -> flowgen_disable
   - Set runtime state to "stopped"
  |
  v
RATE CACHE PRIMING
   - If no recent sample (>3s old) for any visible running flow:
     1. Snapshot tx/rx counters now
     2. Sleep 2.0s
     3. Proceed with collection (gives ~2s window for instantaneous rate)
  |
  v
COUNTER COLLECTION
   - Bulk: flowgen_stats_dump() -> per-stream {tx_packets, rx_packets, latency, jitter}
   - TX: prefer port counter (interface_counters(tx_idx).tx_packets - baseline)
         over flowgen tx counter (which counts packets handed to graph, not wire)
   - RX: single rx_port + flowgen-measure has data -> per-stream rx (disambiguates shared rx)
         multi rx_port (ECMP) -> sum port counters across all rx_ports
  |
  v
RATE COMPUTATION
   - Sliding window: last 6 samples, samples with > 3s age discarded
   - tx_rate = delta_tx_packets / delta_time
   - rx_rate = delta_rx_packets / delta_time
  |
  v
NOISE FLOOR CORRECTIONS
   - rx_rate < 1% of tx_rate -> snap to 0 (filters ARP, BGP keepalives, ICMP)
   - rx_rate > tx_rate -> cap at tx_rate
   - |tx_rate - rx_rate| < 2% of tx_rate -> snap rx_rate = tx_rate (PAPI counter skew)
   - Same 2% snap for cumulative packet counts
  |
  v
LOSS + RESPONSE
   - loss_pct = max(0, tx - rx) / tx * 100
   - Return OTG flow_metrics with frames_tx/rx, rates, latency, jitter, loss, transmit state

---

Test calls get_metrics({choice: "bgpv4"})
  |
  v
   - Route to BgpAdapter.get_metrics()
   - FRR: vtysh -c "show bgp neighbors json"
   - Normalise bgpState: "Established" -> "up", everything else -> "down"
   - Return bgpv4_metrics with name, session_state, prefixes_received

---

Test calls get_metrics({choice: "lacp"})
  |
  v
   - Route to LagAdapter.get_metrics()
   - Parse vppctl show bond details
   - Map BondEthernet<N> to snappi lag name by enumeration order
   - Return lacp_metrics with name, oper_status, active_slaves, total_slaves
```

## Packaging and Deployment

### Container Contents

`docker-vpp-otg` is a single container that includes:

| Content | Purpose |
|---|---|
| VPP runtime + DPDK | Dataplane, interfaces, graph nodes, stats, capture. |
| `flowgen_plugin.so` | Traffic generation and measurement graph nodes. |
| `lacp_plugin.so` | LACP state machine for BondEthernet LAGs. |
| Python adapter (`snappi_vpp/`) | OTG API translation and orchestration. |
| FRR (bgpd, zebra, isisd, ospfd, ospf6d, staticd) | Routed control-plane emulation. |
| Supervisord | Process supervision for VPP, FRR daemons, and adapter. |
| Startup scripts | Backend selection, hugepages, VPP workers, DPDK device binding. |

### Process Model

```
supervisord
  ├── VPP               (main-core + workers, owns dataplane)
  ├── FRR daemons       (bgpd, zebra, etc., own control plane)
  └── snappi-vpp adapter (HTTP server on port 8443, orchestrates VPP + FRR)
```

### Data-Plane Backends

| Backend | Use case |
|---|---|
| DPDK | Physical-DUT path for higher throughput and lower overhead. |
| vhost-user | Virtual testbed path. Connects VPP interfaces directly to QEMU virtio endpoints through shared-memory sockets. |
| af_packet | Kernel-mediated fallback for veth-based local testing or restricted hosts. |


## Testbed Integration

### Physical DUT Topology

```
                        +---------------------------+
                        | Lab management network    |
                        +---+-----------+-----------+
                            |           |
          +-----------------+           +-----------------+
          v                                       v
+-------------------+                 +-----------------------+
| sonic-mgmt        | Snappi REST    | VPP-OTG server        |
| pytest + ansible  |--------------->| docker-vpp-otg (DPDK) |
+---------+---------+                +-----+-----+-----+-----+
          |                                |     |     |
          | mgmt                           |     |     | DPDK NIC ports
          v                                |     |     | direct cables
+-------------------+                      |     |     |
| Physical SONiC DUT|<---------------------+-----+-----+
| Ethernet0..N      |
+-------------------+
```

The data-plane links are point-to-point cables between VPP-OTG NIC ports and DUT front-panel ports. There is no switch in the data path unless a lab deliberately inserts a fanout switch for shared cabling. The test orchestrator, VPP-OTG server, and DUT communicate over the management network for control traffic.

**Physical deployment characteristics:**

| Item | Requirement |
|---|---|
| VPP-OTG server role | Replaces the traffic-generator chassis from the Snappi test framework's point of view. |
| Data-plane NICs | DPDK-supported NIC ports cabled directly to DUT front-panel ports. |
| Management | VPP-OTG server must be reachable from `sonic-mgmt` on the Snappi API port and over SSH for administration. |
| Testbed file | Physical `testbed.yaml` entry points `ptf` / `ptf_ip` at the VPP-OTG server, similar to an Ixia/Snappi testbed. |
| Inventory | Physical inventory lists the DUT and VPP-OTG server. Virtual-only keys such as vhost-user wiring are not used. |
| Cabling | DAC, AOC, or optics matching the DUT speed: 10/25/40/100/400 GbE as required by the lab DUT. |

**Example logical layout:**

```
DUT Ethernet0   <------------------>  VPP-OTG Card1/Port1
DUT Ethernet4   <------------------>  VPP-OTG Card1/Port2
DUT Ethernet8   <------------------>  VPP-OTG Card1/Port3
DUT Ethernet12  <------------------>  VPP-OTG Card1/Port4

Optional:
DUT PortChannelX <----------------->  VPP BondEthernetX with LACP members
```

This topology is the target for tests that require real DUT hardware behavior: PFC, PFCWD, QoS, ECN, packet trimming, reboot behavior, physical link behavior, and ASIC queue behavior. BGP and LACP also run here, with FRR and VPP providing the tester-side peers.

### Virtual Testbed

```
sonic-mgmt container
  |
  | Snappi REST
  v
docker-vpp-otg  <── af_packet/vhost-user ──>  sonic-vpp DUT VM

Example logical links:

  TGen Port 1  <->  DUT Ethernet0
  TGen Port 2  <->  DUT Ethernet4
  TGen Port 3  <->  DUT Ethernet8
  TGen Port 4  <->  DUT Ethernet12
```

The virtual testbed has two useful wiring modes:

| Mode | Wiring | Use |
|---|---|---|
| af_packet over veth/bridge/tap | Works with existing host kernel and bridge wiring. | Easy bring-up, local development, low-rate correctness checks. |
| vhost-user shared memory | VPP-OTG and the sonic-vpp DUT VM exchange packets over shared virtio rings. | Higher-throughput virtual CI and development runs. |

The virtual topology is not meant to validate ASIC-side queueing, PFC backpressure, ECN marking, optics, or physical timing behavior. This is not a traffic-generator limitation -- VPP-OTG can generate and measure PFC, ECN, and QoS traffic -- but a DUT-side restriction: sonic-vpp uses a software dataplane without ASIC hardware, so it cannot enforce real queue scheduling, hardware PFC pause reactions, or ECN marking thresholds. 
The virtual topology is suited for features that do not depend on physical DUT hardware -- protocol correctness (BGP, OSPF, ISIS, LACP), routing convergence, link-event handling, development and regression coverage.

Integrated via `ars vswitch setup --enable-tgen`.

### Control-Plane Peer Layout

In both physical and virtual topologies, each emulated Snappi device has VPP interface/L3 state and, when routed protocols are configured, FRR protocol state.

```
SONiC DUT                         docker-vpp-otg
---------                         --------------
Ethernet0   <------------------>  VPP port / FRR peer 1
Ethernet4   <------------------>  VPP port / FRR peer 2
Ethernet8   <------------------>  VPP port / FRR peer 3
Ethernet12  <------------------>  VPP port / FRR peer 4

Optional:
PortChannelX <----------------->  VPP BondEthernetX with LACP members
```

The DUT does not need to know that the emulated peers are co-located in one container. The Snappi/OTG model still describes ports, devices, protocols, flows, and metrics in the same way it does for an external traffic-generator chassis.

### Server Sizing

Server sizing is driven mainly by tester port count, per-port speed, and whether the deployment is a small bring-up bench or a shared lab/CI resource. Most SONiC Snappi tests use a small number of ports; larger route-convergence topologies can require more.

| Deployment | Suggested server | NIC guidance | Notes |
|---|---|---|---|
| Initial physical bring-up | 16-core x86 server, 16-32 GB RAM, NVMe storage, 1 GbE mgmt NIC. | 4 x 10/25 GbE or 4 x 100 GbE DPDK-supported ports, depending on DUT speed. | Enough to validate basic Snappi, BGP/LACP, and PFC path on a physical DUT. |
| Dedicated 4-8 port lab bench | 1U/2U x86 server, at least one isolated worker core per active tester port plus cores for VPP main, adapter, FRR, and Linux. 32-64 GB RAM. | ConnectX-5/6/7, Intel X710/E810, or equivalent DPDK-supported NICs. | DPDK is the primary backend. Reserve hugepages and isolate worker cores for repeatability. |


**Practical rules:**

| Rule | Reason |
|---|---|
| Match DUT port speed where possible. | Avoid autonegotiation, breakout, and optics issues. |
| Keep management NIC separate from DPDK data NICs. | DPDK takes exclusive ownership of data-plane ports. |
| Reserve hugepages. | VPP/DPDK needs predictable packet-buffer memory. |
| Prefer one VPP worker or PMD core per active high-rate port. | Reduces scheduling noise and improves scaling. |
| Use direct cabling for strict PFC/QoS validation. | Fanout or shared switching can hide or alter pause and timing behavior. |


## Scalability Considerations

| Dimension | Scaling mechanism |
|---|---|
| TX throughput | Add VPP workers (one per NIC queue). Independent workers scale linearly until CPU or NIC saturates. |
| Port count | Add NICs to the VPP-OTG server, or deploy multiple VPP-OTG servers behind separate Snappi API endpoints. |
| BGP route scale | FRR route range expansion is adapter-side. Large route tables (10k+ prefixes) are pushed via vtysh stdin to avoid ARG_MAX. |
| Flow count | Flowgen streams are pooled in VPP. Each stream is lightweight (template + counters). Hundreds of streams supported. |
| Metrics | Bulk `flowgen_stats_dump` preferred over per-flow polling. Protocol metrics fanned out via ThreadPoolExecutor with per-backend timeout. |


## Future Extensions

| Extension | Path |
|---|---|
| NIC/SmartNIC offload | DPDK backend can leverage NIC TX rate limiting, checksum offload, and timestamp offload without changing the adapter or OTG API. |
| L7 stateful traffic | Can be added by extending the flowgen plugin with stateful packet generation, or by leveraging VPP's built-in session layer and host stack for real TCP connection handling. Targets DASH and stateful firewall tests. |
| Additional protocols | ISIS, OSPF, LLDP, DHCP, BFD, MACsec, 802.1X adapters exist at varying levels of completion. Each follows the `ProtocolAdapter` contract. |

## Testing Strategy

| Level | Coverage |
|---|---|
| Unit tests | Per-module tests for translation, headers, reconcile, metrics, protocol adapters (`adapter/tests/unit/`). |
| Integration tests | End-to-end tests against a real VPP instance: config apply, traffic flow, metrics read, pause frames (`adapter/tests/integration/`). |
| Conformance | OTG capability-matrix compliance runner (`snappi_vpp/conformance/`). |
| sonic-mgmt regression | Run existing `sonic-mgmt/tests/snappi_tests/` against docker-vpp-otg to validate real test compatibility. |
| Physical DUT validation | BGP, LACP, PFC, convergence tests against physical SONiC switches. |


## Summary

Snappi-VPP provides a SONiC-native traffic-generator endpoint for both virtual and physical DUT testbeds. The architecture keeps the OTG API surface in Python, uses VPP for the dataplane (flowgen plugin for TX, flowgen-measure for RX, BondEthernet for LACP), maps LAG behavior to VPP BondEthernet, and uses FRR for routed peer emulation (BGP first). The most important design choice is that traffic generation is native to VPP through `flowgen`, so the adapter can drive packets at configured rates, collect counters, measure latency, handle pause frames, and provide convergence observability without depending on an external TGen chassis for every test. The three deployment models (DPDK for physical DUTs, vhost-user for sonic-vpp vlabs, af_packet for PoC/CI) share the same adapter, translation, reconcile, traffic, and metrics code paths -- they differ only in how VPP attaches to the DUT interface.

The current design is strongest for BGP and LACP protocol-correctness testing and uses sonic-vpp as the fast development and CI vehicle. The same architecture is intended to scale to PFC/QoS, additional protocols, NIC offloads, and physical-DUT performance validation as the SONiC community's test coverage grows, while hardware chassis can remain reserved for cases that truly require proprietary chassis functions or strict physical timing.
