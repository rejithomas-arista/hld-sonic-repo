<!-- omit from toc -->
# IPIP Tunnel Support for SONiC #

<!-- omit from toc -->
## Table of Content

<!-- TOC -->

- [1. Revision](#1-revision)
- [2. Definitions/Abbreviations](#2-definitionsabbreviations)
- [3. Scope](#3-scope)
- [4. Overview](#4-overview)
  - [4.1. Context](#41-context)
    - [4.1.1. Current Architecture](#411-current-architecture)
    - [4.1.2. Dual-ToR IPIP vs. Generic IPIP](#412-dual-tor-ipip-vs-generic-ipip)
    - [4.1.3. Control Plane Scaling: Interface-Based vs. Nexthop-Based Tunnels](#413-control-plane-scaling-interface-based-vs-nexthop-based-tunnels)
    - [4.1.4. Gap Summary](#414-gap-summary)
  - [4.2. Solution](#42-solution)
- [5. High-Level Design](#5-high-level-design)
  - [5.1. Tunnel Operational Modes](#51-tunnel-operational-modes)
    - [5.1.1. Mode 1: Interface Mode (Dual-Path)](#511-mode-1-interface-mode-dual-path)
    - [5.1.2. Mode 2: Decap-Only Mode](#512-mode-2-decap-only-mode)
    - [5.1.3. Mode 3: Encap-Only Mode (Lightweight Tunnel Nexthop)](#513-mode-3-encap-only-mode-lightweight-tunnel-nexthop)
    - [5.1.4. Mode 4: Bidirectional Mode (Hybrid)](#514-mode-4-bidirectional-mode-hybrid)
  - [5.2. Existing IPIP Config Flows](#52-existing-ipip-config-flows)
    - [5.2.1. Existing Decap Flow (TunnelMgr → TunnelDecapOrch)](#521-existing-decap-flow-tunnelmgr--tunneldecaporch)
    - [5.2.2. Existing Encap Flow (MuxOrch — Dual-ToR Only)](#522-existing-encap-flow-muxorch--dual-tor-only)
    - [5.2.3. Gap: Routes via Kernel IPIP Interface Do Not Reach SAI](#523-gap-routes-via-kernel-ipip-interface-do-not-reach-sai)
  - [5.3. Proposed Design: Mode 1 Route Flow (Interface Mode)](#53-proposed-design-mode-1-route-flow-interface-mode)
    - [5.3.1. fpmsyncd: IPIP Tunnel Interface Detection](#531-fpmsyncd-ipip-tunnel-interface-detection)
    - [5.3.2. RouteOrch: IPIP Tunnel Nexthop Resolution](#532-routeorch-ipip-tunnel-nexthop-resolution)
    - [5.3.3. Complete Mode 1 Data Flow](#533-complete-mode-1-data-flow)
  - [5.4. Phase 1: VPP SAI Decapsulation](#54-phase-1-vpp-sai-decapsulation)
    - [5.4.1. Accept IPIP Tunnel Type in TunnelManager](#541-accept-ipip-tunnel-type-in-tunnelmanager)
    - [5.4.2. IPIP Tunnel Creation via VPP API](#542-ipip-tunnel-creation-via-vpp-api)
    - [5.4.3. DSCP/ECN/TTL Header Handling](#543-dscpecnttl-header-handling)
    - [5.4.4. IPIP Tunnel Deletion](#544-ipip-tunnel-deletion)
  - [5.5. Phase 2: IPIP Encapsulation with BFD Integration](#55-phase-2-ipip-encapsulation-with-bfd-integration)
    - [5.5.1. TunnelIpipMgr — New Orchestration Component](#551-tunnelipipmgr--new-orchestration-component)
    - [5.5.2. Dual-Path Architecture](#552-dual-path-architecture)
    - [5.5.3. BFD-Based Tunnel State Management](#553-bfd-based-tunnel-state-management)
    - [5.5.4. CONFIG_DB Schema](#554-config_db-schema)
      - [5.5.4.1. Field Definitions](#5541-field-definitions)
      - [5.5.4.2. How Fields Determine the Operational Mode](#5542-how-fields-determine-the-operational-mode)
      - [5.5.4.3. Validation Rules](#5543-validation-rules)
      - [5.5.4.4. Configuration Examples](#5544-configuration-examples)
    - [5.5.5. NhgOrch and RouteOrch Integration](#555-nhgorch-and-routeorch-integration)
  - [5.6. Phase 3: FRR Patch for Kernel Nexthop IPIP Encapsulation](#56-phase-3-frr-patch-for-kernel-nexthop-ipip-encapsulation)
    - [5.6.1. Root Cause — FRR Only Handles MPLS Encapsulation](#561-root-cause--frr-only-handles-mpls-encapsulation)
    - [5.6.2. Required FRR Changes](#562-required-frr-changes)
    - [5.6.3. Impact on Tunnel Modes](#563-impact-on-tunnel-modes)
  - [5.7. Implementation Considerations](#57-implementation-considerations)
    - [5.7.1. YANG Model](#571-yang-model)
    - [5.7.2. Kernel Interface Naming (Mode 1)](#572-kernel-interface-naming-mode-1)
    - [5.7.3. Kernel Interface IP Assignment (Mode 1)](#573-kernel-interface-ip-assignment-mode-1)
    - [5.7.4. Coexistence with Existing Dual-ToR Tunnel Path](#574-coexistence-with-existing-dual-tor-tunnel-path)
    - [5.7.5. Warm Restart](#575-warm-restart)
- [6. Expected Impact](#6-expected-impact)
- [7. Testing](#7-testing)
- [8. Links to Code PRs](#8-links-to-code-prs)

<!-- /TOC -->

## 1. Revision

| Rev  |    Date    |       Author       | Change Description |
| :--: | :--------: | :----------------: | :----------------: |
| 0.1  | 2026-04-10 |    Reji Thomas     | Initial version    |

## 2. Definitions/Abbreviations

| Definitions/Abbreviation | Description                                                     |
| ------------------------ | --------------------------------------------------------------- |
| BFD                      | Bidirectional Forwarding Detection (RFC 5880/5881)              |
| CONFIG_DB                | SONiC Configuration Database (Redis DB 4)                       |
| DSCP                     | Differentiated Services Code Point                              |
| ECN                      | Explicit Congestion Notification                                |
| ECMP                     | Equal-Cost Multi-Path routing                                   |
| FRR                      | Free Range Routing                                              |
| IPIP                     | IP-in-IP tunneling (RFC 2003)                                   |
| LWTUNNEL                 | Linux kernel lightweight tunnel infrastructure                  |
| NHG                      | NextHop Group                                                   |
| P2MP                     | Point-to-Multipoint                                             |
| P2P                      | Point-to-Point                                                  |
| SAI                      | Switch Abstraction Interface                                    |
| TTL                      | Time To Live                                                    |
| VPP                      | Vector Packet Processing (fd.io)                                |
| ZMQ                      | ZeroMQ (high-performance asynchronous messaging)                |

## 3. Scope

Implement generic IPIP (IP-in-IP) tunnel support for SONiC, covering decapsulation, encapsulation, optional BFD-based tunnel state tracking, and the FRR patch required to support kernel nexthop objects with IPIP encapsulation. The design is platform-agnostic — all orchestration, CONFIG_DB schema, BFD integration, and FRR changes use standard SAI APIs and work on any data plane that implements `SAI_TUNNEL_TYPE_IPINIP`. The initial implementation and validation target is the VPP data plane, but the architecture applies equally to hardware ASIC platforms with SAI IPIP support.

## 4. Overview

### 4.1. Context

#### 4.1.1. Current Architecture

SONiC's tunnel data path has four layers. This section describes the current state of each layer for IPIP specifically. The first three layers — orchagent, SAI API definitions, and FRR — are platform-agnostic and apply to any SAI-compliant data plane. The fourth layer, the SAI provider, is platform-specific; VPP is described here as the initial target.

**SONiC Orchagent (platform-agnostic) — Decap path exists.** The generic IPIP decapsulation control plane is already in place:

- `tunneldecaporch` processes `TUNNEL_DECAP_TABLE` and `TUNNEL_DECAP_TERM_TABLE` entries from APP_DB
- `ipinip.json.j2` auto-generates decap configuration from device metadata
- `sairedis` includes full SAI API definitions for `SAI_TUNNEL_TYPE_IPINIP`

No encapsulation orchestration exists for generic IPIP. There is no component that creates IPIP tunnels for encap, manages tunnel nexthops, or tracks tunnel state. The new `TunnelIpipMgr` component proposed in this HLD (§5.5) is entirely SAI-based and platform-agnostic.

**SAI Provider (platform-specific) — VPP blocks IPIP.** The VPP SAI provider (`TunnelManager.cpp`, line 111) rejects any tunnel type other than VXLAN with `SAI_STATUS_NOT_IMPLEMENTED`. All upstream control plane processing completes successfully — APP_DB entries are created, orchagent calls SAI — but the SAI provider drops the request. No IPIP operations reach the data plane. Other SAI providers (e.g., Memory/hardware ASIC vendors) may already support `SAI_TUNNEL_TYPE_IPINIP`; the orchestration changes in this HLD would work on those platforms without modification.

**Data Plane (platform-specific) — VPP ready, no changes needed.** VPP natively supports IPIP tunnels via its `ipip_add_tunnel` / `ipip_del_tunnel` APIs. Manual testing has confirmed both encapsulation and decapsulation at line rate. Tunnels can be P2P (fixed remote endpoint) or P2MP (accept from any source). Both IPv4 and IPv6 transport are supported. On hardware ASIC platforms, the equivalent IPIP forwarding is handled by the switching silicon.

**FRR (platform-agnostic) — No IPIP nexthop support.** FRR 10.3 supports kernel nexthop objects but only parses MPLS encapsulation (`LWTUNNEL_ENCAP_MPLS`). IPIP encapsulation (`LWTUNNEL_ENCAP_IP`) is silently ignored, causing FRR to misinterpret routes as "directly connected." This limits control plane scalability on all platforms, as discussed in §4.1.3.

#### 4.1.2. Dual-ToR IPIP vs. Generic IPIP

SONiC already has one IPIP implementation: the **dual-ToR** use case in `muxorch.cpp`. It uses IPIP tunnels between paired ToR switches for server redundancy. MuxOrch creates a SAI tunnel with `SAI_TUNNEL_TYPE_IPINIP` and both `ENCAP_SRC_IP`/`ENCAP_DST_IP` attributes, then creates a `SAI_NEXT_HOP_TYPE_TUNNEL_ENCAP` nexthop and installs routes pointing to it. It also creates a parallel kernel tunnel interface (`tun0`) for control plane traffic. So dual-ToR does perform both encapsulation and decapsulation, but in a narrow, purpose-built way:

- Fixed tunnel between two specific ToR switches (single src/dst pair)
- P2MP decap (accept from any source to local endpoint)
- No tunnel state tracking — tunnels are always UP
- No BFD, no dynamic failure detection
- ~2 tunnels per device
- Tunnel lifecycle tightly coupled to MuxOrch's active/standby state machine

The dual-ToR implementation established a proven **dual-path architecture**: a SAI tunnel for data plane forwarding and a parallel kernel tunnel interface (`tun0`) for control plane traffic. This separation is necessary because the SAI tunnel cannot be addressed by kernel routing, and the kernel tunnel cannot handle line-rate traffic.

**Generic IPIP support** — the subject of this HLD — extends beyond dual-ToR to cover general-purpose overlay networking: site-to-site connectivity, traffic steering, overlay/underlay separation. It requires capabilities that dual-ToR does not provide:

| Capability | Dual-ToR | Generic IPIP (this HLD) |
|------------|----------|-------------------------|
| Encap | Fixed single peer (MuxOrch-managed) | Arbitrary P2P, P2MP, ECMP nexthops |
| Decap modes | P2MP only | P2P, P2MP, MP2P, MP2MP |
| Tunnel state tracking | Always UP | Optional BFD (sub-second failover) |
| Route withdrawal on failure | No | Yes (BFD → tun down → FRR withdraws) |
| Tunnel lifecycle management | MuxOrch (tied to active/standby) | TunnelIpipMgr (general-purpose, NEW) |
| Scale target | ~2 tunnels | Hundreds to thousands |

#### 4.1.3. Control Plane Scaling: Interface-Based vs. Nexthop-Based Tunnels

For the control plane to participate in encapsulation — so that FRR can install routes via a tunnel and advertise/withdraw them — the kernel must have a representation of the tunnel. There are two approaches, each with different scale characteristics.

**Approach 1: Interface-based tunnels (kernel `tun0` interface)**

Each tunnel gets a dedicated kernel interface created via `ip tunnel add tun0 mode ipip local <src> remote <dst>`. FRR sees `tun0` as a regular interface and can assign addresses, run BGP sessions over it, and react to interface up/down events.

```
TunnelIpipMgr
  ├── SAI tunnel  (data plane, line-rate)
  └── kernel tun0 (control plane, FRR-visible)
         ├── ip address add 10.1.0.1/32 dev tun0
         ├── FRR: interface tun0 → BGP neighbor
         └── BFD down → ip link set tun0 down → FRR withdraws routes
```

This is the dual-ToR pattern generalized. It works today without any FRR changes. The limitation is **~1000 tunnels** — each tunnel consumes a kernel network interface, and Linux has practical limits on interface count.

**Approach 2: Nexthop-based tunnels (kernel nexthop objects with `encap ip`)**

Instead of creating a kernel interface per tunnel, the tunnel encapsulation is encoded in a kernel nexthop object. Routes reference the nexthop object by ID rather than an interface name.

```
TunnelIpipMgr
  ├── SAI tunnel     (data plane, line-rate)
  └── kernel nexthop (control plane, FRR-visible)
         ├── ip nexthop add id 999 encap ip dst 10.2.0.1 src 10.1.0.1 dev Ethernet0
         ├── ip route add 172.16.0.0/16 nhid 999
         └── No interface limit — nexthop objects are lightweight
```

Kernel nexthop objects with IPIP encapsulation have been verified working on Linux 5.14. However, FRR currently ignores IPIP encapsulation in nexthop objects (it only handles MPLS), misinterpreting these routes as "directly connected." A **FRR patch** is required to unblock this approach (see §5.6).

**When to use which:**

| Requirement | Interface-Based | Nexthop-Based |
|-------------|:---------------:|:-------------:|
| Run routing protocols (BGP/OSPF) over the tunnel | **Yes** — FRR sees `tun0` as a routable interface | No — nexthop objects are not addressable interfaces |
| Assign IP addresses to the tunnel endpoint | **Yes** — `ip addr add` on `tun0` | No |
| BFD-driven tunnel state (up/down → route withdrawal) | **Yes** — `ip link set tun0 down` triggers FRR | Not directly — no interface to bring down |
| Hundreds or thousands of tunnels | No — ~1000 kernel interface limit | **Yes** — nexthop objects are lightweight |
| Static route steering (route X via tunnel Y) | Yes | **Yes** |
| No FRR patch required | **Yes** — works today | No — requires FRR patch (§5.6) |

**Use interface-based tunnels** when the tunnel needs to be a first-class network interface — running BGP/OSPF sessions over it, assigning addresses, or using BFD to drive interface state for route withdrawal. This is the right choice for site-to-site connectivity with a moderate number of tunnels.

**Use nexthop-based tunnels** when scale is the primary concern — hundreds or thousands of tunnels where each tunnel simply steers traffic to a remote endpoint without needing to run routing protocols over the tunnel itself. Typical use cases include large-scale overlay fabrics or programmatic tunnel provisioning.

The two approaches are complementary and the implementation supports both. Interface-based tunnels (Mode 1) provide the richest control plane integration and work today without FRR changes. Nexthop-based tunnels (Modes 3 and 4) remove the scale limit but require the FRR patch (§5.6).

#### 4.1.4. Gap Summary

| Layer | Component | Platform Scope | Status | Gap |
|-------|-----------|----------------|--------|-----|
| Orchagent | Decap orchestration (`tunneldecaporch`) | Agnostic | Ready | None |
| Orchagent | Encap orchestration (tunnel lifecycle) | Agnostic | Missing | Need new `TunnelIpipMgr` |
| Orchagent | BFD-to-tunnel state binding | Agnostic | Missing | Need `BfdOrch::Observer` integration |
| Orchagent | IPIP nexthop in NhgOrch/RouteOrch | Agnostic | Missing | Need nexthop resolution path |
| CONFIG_DB | `TUNNEL_IPIP` table | Agnostic | Missing | Need new schema for encap config |
| FRR | IPIP encap in kernel nexthop objects | Agnostic | Missing | Only handles MPLS; need patch |
| Kernel | Nexthop objects with `encap ip` | Agnostic | Ready | None (verified on kernel 5.14) |
| SAI Provider | VPP `TunnelManager.cpp` IPINIP handling | VPP-specific | Missing | Rejects IPIP; need to implement |
| Data Plane | VPP IPIP encap/decap | VPP-specific | Ready | None |

### 4.2. Solution

Implement IPIP tunnel support in three phases. Phases 2 and 3 are platform-agnostic (SAI-based orchagent changes, FRR patch). Phase 1 requires a SAI provider change per platform; VPP is the initial target.

1. **Phase 1 — SAI Provider Decapsulation (VPP first)**: Accept `SAI_TUNNEL_TYPE_IPINIP` in the VPP SAI provider (`TunnelManager.cpp`), translate to VPP `ipip_add_tunnel` API calls, and handle P2P/P2MP decapsulation with DSCP/ECN/TTL modes. This unblocks the existing SONiC control plane decap path. On other platforms, this phase requires the respective SAI provider to implement `SAI_TUNNEL_TYPE_IPINIP` — the orchagent path is already in place.

2. **Phase 2 — Encapsulation with BFD (Interface-Based, platform-agnostic)**: Introduce `TunnelIpipMgr` as the central tunnel lifecycle manager. For each tunnel, it creates a SAI tunnel (data plane) and a kernel tunnel interface (control plane), following the dual-path architecture from §4.1.2. Optional BFD sessions track tunnel reachability; BFD failure triggers `ip link set tun0 down`, causing FRR to withdraw routes. This phase uses the interface-based approach (§4.1.3, Approach 1) and works on any platform with SAI IPIP support, without FRR changes.

3. **Phase 3 — FRR Patch (Nexthop-Based Scaling, platform-agnostic)**: Add `LWTUNNEL_ENCAP_IP` support to FRR's `netlink_nexthop_process_nh()`, enabling kernel nexthop objects with IPIP encapsulation to be correctly interpreted by FRR. This unblocks the nexthop-based approach (§4.1.3, Approach 2), removing the ~1000 tunnel limit and enabling Modes 3 and 4. This patch benefits all SONiC platforms.

## 5. High-Level Design

### 5.1. Tunnel Operational Modes

IPIP tunnels support four distinct operational modes, each with different control plane and data plane configurations:

| Mode | DP Encap | DP Decap | CP Encap | CP Decap | Scale Limit | FRR Patch |
|------|----------|----------|----------|----------|-------------|-----------|
| **1. Interface** | VPP SAI | VPP SAI | Kernel `tun0` | Kernel `tun0` | ~1000 tunnels | No |
| **2. Decap-only** | N/A | VPP SAI | N/A | N/A | Unlimited | No |
| **3. Encap-only** | VPP SAI | N/A | lwtunnel nexthop | N/A | Unlimited | **Yes** |
| **4. Bidirectional** | VPP SAI | VPP SAI | lwtunnel nexthop | None | Unlimited | **Yes** |

#### 5.1.1. Mode 1: Interface Mode (Dual-Path)

Following the proven dual-ToR IPIP implementation:

- **Path 1 — SAI Tunnel (Data Plane)**: VPP creates `ipip0` interface for line-rate forwarding
- **Path 2 — Kernel Tunnel (Control Plane)**: `ip tunnel add tun0 mode ipip` for control plane traffic
- **Why both?**: SAI tunnel cannot be used by kernel routing; kernel tunnel cannot handle line-rate traffic

This mode supports full bidirectional tunneling with control plane participation. The limitation is ~1000 tunnels due to the kernel interface limit.

#### 5.1.2. Mode 2: Decap-Only Mode

- VPP SAI creates decap tunnel via `SAI_TUNNEL_TYPE_IPINIP`
- No kernel interface, no control plane encap/decap needed
- Use case: terminating underlay tunnels for data plane traffic only
- This is the existing dual-ToR decap use case

#### 5.1.3. Mode 3: Encap-Only Mode (Lightweight Tunnel Nexthop)

- SAI creates encap tunnel via `SAI_TUNNEL_TYPE_IPINIP`
- Control plane encap: FRR nexthop-group with `encap ip src <tunnel-src>`, creating kernel nexthop object with `LWTUNNEL_ENCAP_IP`
- No kernel interface created — uses kernel nexthop objects (unlimited scale)
- Routes are configured via FRR static routes (`ip route <prefix> nexthop-group <NAME>`) or PBR policies (`set nexthop-group <NAME>`)
- **Requires FRR patch** to support `encap ip` in nexthop-group CLI and `ip route ... nexthop-group` in staticd (§5.6)
- Use cases:
  - **Static route steering**: FRR static routes via nexthop-group — routes are redistributable via BGP
  - **PBR service chaining**: PBR policy matches traffic by src/dst IP and steers it through IPIP tunnel to a middlebox (firewall, DPI, load balancer) that strips the outer header; return traffic takes a different (non-tunnel) path
  - **Large-scale programmatic provisioning**: hundreds or thousands of tunnel endpoints managed via CONFIG_DB, with frrcfgd generating FRR config

#### 5.1.4. Mode 4: Bidirectional Mode (Hybrid)

- VPP SAI creates bidirectional tunnel (both encap and decap)
- Control plane encap: kernel nexthop object with `encap ip`
- Control plane decap: **none** — data plane handles all decap, control plane cannot receive overlay traffic
- **Requires FRR patch** to support `LWTUNNEL_ENCAP_IP`

**VPP behavior note**: VPP IPIP tunnels are always bidirectional at the data plane level. The "mode" controls only what SAI objects (nexthop for encap, term entry for decap) are created and whether kernel infrastructure (interface or nexthop object) is provisioned.

### 5.2. Existing IPIP Config Flows

Before describing the new design, this section documents the two IPIP config flows that exist in upstream SONiC today.

#### 5.2.1. Existing Decap Flow (TunnelMgr → TunnelDecapOrch)

This is the generic IPIP decap path used by dual-ToR and any deployment with IPIP decapsulation:

```
CONFIG_DB                        TunnelMgr (cfgmgr)                  APP_DB                          TunnelDecapOrch (orchagent)
─────────                        ──────────────────                  ──────                          ──────────────────────────
CFG_TUNNEL_TABLE                 doTunnelTask()                      APP_TUNNEL_DECAP_TABLE          addDecapTunnel()
  "MuxTunnel0" {                   1. Read tunnel_type, dst_ip         "MuxTunnel0" {                  1. Create overlay loopback RIF
    tunnel_type: IPINIP            2. Get peer IP from                   tunnel_type: IPINIP           2. Create SAI tunnel:
    dst_ip: 10.1.0.32                CFG_PEER_SWITCH_TABLE               dscp_mode: uniform              SAI_TUNNEL_TYPE_IPINIP
    dscp_mode: uniform             3. Create kernel tun0:                ecn_mode: ...                   SAI_TUNNEL_ATTR_ENCAP_SRC_IP
    ...                               ip tunnel add tun0              }                                  SAI_TUNNEL_ATTR_DECAP_ECN_MODE
  }                                   mode ipip                                                          SAI_TUNNEL_ATTR_DECAP_TTL_MODE
                                      local <dst_ip>                APP_TUNNEL_DECAP_TERM_TABLE       addDecapTunnelTermEntry()
CFG_PEER_SWITCH_TABLE                 remote <peer_ip>                "MuxTunnel0:10.1.0.32" {         3. Create SAI tunnel term:
  address_ipv4: <peer>             4. ip link set tun0 up               term_type: P2MP                  SAI_TUNNEL_TERM_TYPE_P2P/P2MP
                                   5. Assign Loopback3 IP             }                                  SAI_TUNNEL_TERM..DST_IP
                                      to tun0                                                             SAI_TUNNEL_TERM..SRC_IP (P2P)
                                   6. Write to APP_DB
```

**Key points:**
- `TunnelMgr` runs in the `cfgmgr` process (swss container), not in orchagent
- `TunnelMgr` does **two independent things**: (a) creates the kernel `tun0` interface, and (b) writes to APP_DB for orchagent to consume
- `TunnelDecapOrch` creates the SAI tunnel and term entries — this is where the SAI provider (VPP `TunnelManager.cpp`) is called and currently rejects IPIP
- `TunnelDecapOrch` also exposes a public `createNextHopTunnel()` API that other orchs can call to create `SAI_NEXT_HOP_TYPE_TUNNEL_ENCAP` nexthops against the tunnel

#### 5.2.2. Existing Encap Flow (MuxOrch — Dual-ToR Only)

MuxOrch handles IPIP encapsulation for dual-ToR. It **bypasses** fpmsyncd and RouteOrch entirely:

```
CONFIG_DB                        MuxOrch (orchagent)                     SAI / VPP
─────────                        ──────────────────                      ─────────
MUX_CABLE_TABLE                  1. Create SAI tunnel:                   SAI tunnel object
  "Ethernet4" {                     SAI_TUNNEL_TYPE_IPINIP                 (encap src/dst IPs)
    state: standby                  ENCAP_SRC_IP = loopback IP
    server_ipv4: 192.168.0.2       ENCAP_DST_IP = peer switch IP        SAI tunnel encap nexthop
  }                              2. Create SAI nexthop:                    (TUNNEL_ENCAP type)
                                    SAI_NEXT_HOP_TYPE_TUNNEL_ENCAP
                                    pointing to tunnel OID               SAI route entry
                                 3. sai_route_api->create_route_entry()    server_ip → tunnel NH
                                    server_ip → tunnel nexthop
```

**Key points:**
- MuxOrch creates its **own** SAI tunnel object (separate from TunnelDecapOrch's)
- MuxOrch calls `sai_route_api->create_route_entry()` **directly** — it does not go through RouteOrch
- This means routes programmed by MuxOrch are invisible to RouteOrch and fpmsyncd
- The approach works for dual-ToR's narrow use case (~2 tunnels, fixed server IPs) but does not scale or integrate with FRR

#### 5.2.3. Gap: Routes via Kernel IPIP Interface Do Not Reach SAI

When FRR installs a route via the kernel `tun0` interface, the route does **not** result in IPIP encapsulation in the data plane. The exact flow:

```
Step 1: FRR installs route
   FRR:  ip route 172.16.0.0/16 via 10.2.0.1 dev tun0

Step 2: FRR → fpmsyncd (via FPM netlink)
   fpmsyncd receives RTM_NEWROUTE:
     prefix: 172.16.0.0/16
     RTA_GATEWAY: 10.2.0.1
     RTA_OIF: ifindex of tun0

   getNextHopList() resolves:
     gw_list = "10.2.0.1"
     intf_list = "tun0"       ← just the interface name, no tunnel metadata

   Writes to APP_ROUTE_TABLE:
     ROUTE_TABLE:172.16.0.0/16 → nexthop=10.2.0.1, ifname=tun0

Step 3: RouteOrch processes the route (routeorch.cpp:963)
     if (alsv[i] == "tun0" && !(IpAddress(ipv[i]).isZero()))
         alsv[i] = gIntfsOrch->getRouterIntfsAlias(ipv[i]);

   RouteOrch REPLACES "tun0" with the underlay interface that can reach
   10.2.0.1 (e.g., "Ethernet0"). The tunnel association is lost.

Step 4: SAI route created
     SAI route 172.16.0.0/16 → nexthop 10.2.0.1 via Ethernet0 RIF
     → Plain IP forwarding. NO IPIP encapsulation.
```

**Root cause:** fpmsyncd treats `tun0` as an opaque interface name. It does not inspect the interface type or extract tunnel parameters. The APP_ROUTE_TABLE entry contains no tunnel metadata. RouteOrch's existing code actively strips the `tun0` reference and resolves to the underlay.

**Contrast with VXLAN:** For VXLAN/EVPN routes, fpmsyncd has a separate code path (`getEvpnNextHop()`) that parses `RTA_ENCAP_TYPE = NH_ENCAP_VXLAN` and `RTA_ENCAP` containing VNI/MAC. It writes `vni_label` and `router_mac` fields to APP_ROUTE_TABLE. RouteOrch sees these fields, sets `overlay_nh = true`, and dispatches to `VxlanTunnelOrch::createNextHopTunnel()` to create a SAI tunnel encap nexthop. The tunnel metadata travels explicitly in the route entry.

### 5.3. Proposed Design: Mode 1 Route Flow (Interface Mode)

The new design follows the VXLAN pattern: fpmsyncd detects IPIP tunnel interfaces, adds tunnel metadata to route entries, and RouteOrch resolves tunnel nexthops via TunnelIpipMgr.

#### 5.3.1. fpmsyncd: IPIP Tunnel Interface Detection

fpmsyncd already maintains a kernel link cache (`m_link_cache`) and can query interface properties via libnl. When processing a route, after resolving the interface name via `getIfName()`, fpmsyncd checks the interface type:

```cpp
rtnl_link *link = getLinkByName(if_name);      // already exists in fpmsyncd
const char *type = rtnl_link_get_type(link);   // returns "ipip" for IPIP tunnels
```

The kernel `IFLA_INFO_KIND` for IPIP tunnel interfaces is `"ipip"`. From the link object, fpmsyncd can also extract the tunnel parameters (`local`, `remote`) from `IFLA_INFO_DATA`.

When the outgoing interface is an IPIP tunnel, fpmsyncd adds tunnel metadata fields to the APP_ROUTE_TABLE entry:

```
ROUTE_TABLE:172.16.0.0/16 →
    nexthop    = "10.2.0.1"
    ifname     = "tun0"
    tunnel_type = "IPINIP"
    tunnel_src  = "10.1.0.32"
    tunnel_dst  = "10.2.0.1"
```

**Files to modify:** `sonic-swss/fpmsyncd/routesync.cpp` — extend `getNextHopList()` and/or `onRouteMsg()`.

#### 5.3.2. RouteOrch: IPIP Tunnel Nexthop Resolution

RouteOrch is extended to recognize IPIP tunnel routes, following the VXLAN `overlay_nh` pattern:

```
Step 1: RouteOrch receives route with tunnel_type=IPINIP
   → Sets ipip_nh = true (analogous to overlay_nh for VXLAN)

Step 2: RouteOrch resolves nexthop via TunnelIpipMgr
   → TunnelIpipMgr::getTunnelNexthop("tun0") returns SAI tunnel encap nexthop OID
   → The SAI nexthop is SAI_NEXT_HOP_TYPE_TUNNEL_ENCAP pointing to the SAI IPIP tunnel

Step 3: RouteOrch creates SAI route
   → SAI route 172.16.0.0/16 → SAI tunnel encap nexthop
   → VPP/SAI receives the route and knows to IPIP-encapsulate packets
```

**The existing `tun0` stripping code (line 963) is modified:** Instead of replacing `tun0` with an underlay interface, RouteOrch checks whether the interface is a known IPIP tunnel. If yes, it takes the tunnel nexthop path. If no (unknown tunnel), the existing behavior is preserved.

**Files to modify:** `sonic-swss/orchagent/routeorch.cpp` — modify `tun0` handling, add `ipip_nh` path.

#### 5.3.3. Complete Mode 1 Data Flow

```
┌───────────┐    ┌─────────────────┐    ┌─────────────────────┐    ┌──────────────────┐
│    FRR    │    │   fpmsyncd      │    │    RouteOrch         │    │   SAI / VPP      │
│           │    │                 │    │                      │    │                  │
│ ip route  │───►│ Detects tun0    │───►│ Sees tunnel_type=    │───►│ SAI route with   │
│ 172.16./16│    │ is type "ipip"  │    │ IPINIP               │    │ tunnel encap NH  │
│ via 10.2. │    │                 │    │                      │    │                  │
│ dev tun0  │    │ Adds fields:    │    │ Looks up tunnel NH   │    │ VPP encaps pkts  │
│           │    │  tunnel_type    │    │ from TunnelIpipMgr   │    │ with outer IP    │
│           │    │  tunnel_src     │    │                      │    │ src/dst headers  │
│           │    │  tunnel_dst     │    │ Programs SAI route   │    │                  │
│           │    │                 │    │ → tunnel NH OID      │    │                  │
└───────────┘    └─────────────────┘    └──────────────────────┘    └──────────────────┘

     CP traffic (BGP, BFD):          DP traffic (line-rate):
     kernel tun0 handles             SAI tunnel handles
     encap/decap directly            encap/decap via VPP
```

**Control plane traffic** (BGP keepalives, BFD probes) flows through the kernel `tun0` interface — the kernel does its own IPIP encap/decap for these packets. This works because `TunnelMgr` (or `TunnelIpipMgr` in the new design) creates the kernel tunnel interface with the same src/dst IPs as the SAI tunnel.

**Data plane traffic** flows through VPP/SAI. Packets matching the SAI route hit the SAI tunnel encap nexthop, VPP adds the outer IP header, and sends the encapsulated packet via the underlay. Decap works via the SAI tunnel term entry — incoming IPIP packets matching the term entry have their outer header stripped.

### 5.4. Phase 1: VPP SAI Decapsulation

**Files to modify:** `sonic-buildimage/platform/vpp/saivpp/src/TunnelManager.cpp`, `sonic-buildimage/platform/vpp/saivpp/src/TunnelManager.h`, `sonic-buildimage/platform/vpp/saivpp/vppxlate/SaiVppXlate.cpp`

**No changes to:** orchagent, tunneldecaporch, fpmsyncd, APP_DB schema.

#### 5.4.1. Accept IPIP Tunnel Type in TunnelManager

The existing VXLAN-only guard in `TunnelManager::create_tunnel()` (line 111) is extended to accept `SAI_TUNNEL_TYPE_IPINIP`:

```cpp
// Before:
if (attr.value.s32 != SAI_TUNNEL_TYPE_VXLAN) {
    SWSS_LOG_ERROR("Unsupported tunnel encap type %d", attr.value.s32);
    return SAI_STATUS_NOT_IMPLEMENTED;
}

// After:
if (attr.value.s32 != SAI_TUNNEL_TYPE_VXLAN &&
    attr.value.s32 != SAI_TUNNEL_TYPE_IPINIP) {
    SWSS_LOG_ERROR("Unsupported tunnel type %d", attr.value.s32);
    return SAI_STATUS_NOT_IMPLEMENTED;
}
```

The `create_tunnel()` dispatch then branches to the new IPIP path when `tunnel_type == SAI_TUNNEL_TYPE_IPINIP`.

#### 5.4.2. IPIP Tunnel Creation via VPP API

A new function `create_vpp_ipip_decap()` translates SAI tunnel attributes to a VPP `ipip_add_tunnel` API call:

1. Extract destination IP from `SAI_TUNNEL_TERM_TABLE_ENTRY_ATTR_DST_IP` — this is the local tunnel endpoint (maps to VPP's `src_address`)
2. For P2P tunnels, extract source IP from `SAI_TUNNEL_TERM_TABLE_ENTRY_ATTR_SRC_IP` — this is the remote tunnel endpoint (maps to VPP's `dst_address`)
3. For P2MP tunnels, set VPP `dst_address` to `0.0.0.0` (accept from any source)
4. Call `vpp_ipip_tunnel_add_del()` with `is_add=1`

```
SAI Attribute                              VPP Field
──────────────────────────────────────────────────────────
TUNNEL_TERM..DST_IP (local endpoint)  →    tunnel.src
TUNNEL_TERM..SRC_IP (remote endpoint) →    tunnel.dst
TUNNEL_TERM..TYPE (P2P/P2MP)          →    tunnel.mode
IP address family                     →    tunnel.transport (IP4/IP6)
Auto-assigned                         →    tunnel.instance = ~0
```

The VPP API wrapper `vpp_ipip_tunnel_add_del()` in `SaiVppXlate.cpp` allocates a `vapi_msg_ipip_add_tunnel`, fills the tunnel parameters, and sends via the VAPI context. On success, it returns the `sw_if_index` of the created tunnel interface.

#### 5.4.3. DSCP/ECN/TTL Header Handling

IPIP tunnel header handling follows the SAI tunnel attribute model. These attributes are read from the tunnel object and translated to VPP tunnel flags:

| SAI Attribute | Mode | Behavior |
|---------------|------|----------|
| `DECAP_DSCP_MODE` | Uniform | Inner DSCP updated from outer |
| `DECAP_DSCP_MODE` | Pipe | Inner DSCP preserved |
| `DECAP_ECN_MODE` | Copy from outer | Inner ECN set to outer value |
| `DECAP_ECN_MODE` | Standard | RFC 3168 compliant ECN processing |
| `DECAP_TTL_MODE` | Pipe | Inner TTL preserved (decremented by 1) |
| `DECAP_TTL_MODE` | Uniform | Inner TTL updated to min(outer, inner) |
| `ENCAP_DSCP_MODE` | Uniform | Outer DSCP copied from inner |
| `ENCAP_DSCP_MODE` | Pipe | Outer DSCP set to configured value |
| `ENCAP_TTL_MODE` | Pipe | Outer TTL set to configured value |
| `ENCAP_TTL_MODE` | Uniform | Outer TTL copied from inner |

#### 5.4.4. IPIP Tunnel Deletion

Tunnel deletion mirrors creation: `remove_vpp_ipip_tunnel()` calls `vpp_ipip_tunnel_add_del()` with `is_add=0` using the stored `sw_if_index`. The dispatch in `remove_tunnel()` branches on tunnel type identically to `create_tunnel()`.

### 5.5. Phase 2: IPIP Encapsulation with BFD Integration

**New files:** `sonic-buildimage/src/sonic-swss/orchagent/tunnelipipmgr.h`, `sonic-buildimage/src/sonic-swss/orchagent/tunnelipipmgr.cpp`

**Files to modify:** `sonic-buildimage/src/sonic-swss/orchagent/bfdorch.h`, `sonic-buildimage/src/sonic-swss/orchagent/bfdorch.cpp`, `sonic-buildimage/src/sonic-swss/orchagent/nhgorch.cpp`, `sonic-buildimage/src/sonic-swss/orchagent/routeorch.cpp`

#### 5.5.1. TunnelIpipMgr — New Orchestration Component

`TunnelIpipMgr` is the central manager for all IPIP tunnel modes (Modes 1–4). It runs inside the **orchagent process** and subscribes directly to `CONFIG_DB:TUNNEL_IPIP`. This follows the established SONiC pattern — multiple orchs already subscribe to CONFIG_DB directly, including CrmOrch (`CFG_CRM`), QosOrch (`CFG_DSCP_TO_TC_MAP`, etc.), NvgreTunnelOrch (`CFG_NVGRE_TUNNEL`), VNetCfgRouteOrch (`CFG_VNET_RT`), Srv6Orch (`CFG_SRV6_MY_SID`), and others.

Running inside orchagent gives TunnelIpipMgr direct access to:
- SAI APIs for tunnel and nexthop creation
- `BfdOrch` for BFD state change notifications (Observer pattern)
- `RouteOrch` / `NhgOrch` for tunnel nexthop resolution callbacks
- Kernel operations via `swss::exec()` (same approach as MuxOrch for `ip tunnel` / `ip link` commands)

`TunnelIpipMgr` orchestrates per-mode operations:

| Operation | Mode 1 (Interface) | Mode 2 (Decap) | Mode 3 (Encap) | Mode 4 (Bidir) |
|-----------|-------------------|----------------|----------------|----------------|
| SAI tunnel | Yes | Yes | Yes | Yes |
| SAI encap nexthop | Yes | No | Yes | Yes |
| SAI decap term entry | Yes | Yes | No | Yes |
| Kernel `tun0` interface | Yes | No | No | No |
| Kernel nexthop object | No | No | Yes | Yes |
| BFD session (optional) | Yes | No | No | No |
| STATE_DB update | Yes | Yes | Yes | Yes |

**Mode 1 flow (symmetric + interface):**

```
┌────────────────┐  CONFIG_DB   ┌─────────────────────────────────────────────┐
│   config_db    │──TUNNEL_IPIP─►            TunnelIpipMgr                   │
└────────────────┘              │            (in orchagent)                    │
                                │                                             │
                                │  1. createSaiTunnel()                       │
                                │     └─► SAI tunnel + encap nexthop          │
                                │         + decap term entry                  │
                                │  2. createKernelTunnel()                    │
                                │     └─► swss::exec("ip tunnel add tun0")   │
                                │  3. createBfdSession() (if bfd_enable=true) │
                                │     └─► SAI BFD session via BfdOrch         │
                                └──────────────────┬──────────────────────────┘
                                                   │
                                  BFD state change │ (BfdOrch::Observer)
                                                   ▼
                                ┌──────────────────────────────────────────────┐
                                │             onBfdStateChange()               │
                                │                                              │
                                │  BFD UP   → swss::exec("ip link set tun0 up")
                                │  BFD DOWN → swss::exec("ip link set tun0 down")
                                │                                              │
                                │  FRR sees interface state change →           │
                                │  withdraws/advertises routes accordingly     │
                                └──────────────────────────────────────────────┘
```

**Mode 2 flow (decap-only):**

Mode 2 is the simplest flow. TunnelIpipMgr creates only the SAI tunnel and decap term entry — no kernel interface, no encap nexthop, no BFD.

```
┌────────────────┐  CONFIG_DB   ┌─────────────────────────────────────────────┐
│   config_db    │──TUNNEL_IPIP─►            TunnelIpipMgr                   │
└────────────────┘              │            (in orchagent)                    │
                                │                                             │
                                │  1. createSaiTunnel()                       │
                                │     └─► SAI tunnel (IPINIP, with src_ip)    │
                                │  2. createDecapTermEntry()                  │
                                │     └─► SAI tunnel term (P2P or P2MP)       │
                                │  3. Update STATE_DB with tunnel status      │
                                │                                             │
                                │  No kernel interface                        │
                                │  No encap nexthop                           │
                                │  No BFD                                     │
                                └─────────────────────────────────────────────┘

Data plane:
  IPIP packet arrives → outer dst matches term entry → VPP strips outer header → inner packet forwarded
```

This is functionally equivalent to what `TunnelDecapOrch::addDecapTunnel()` + `addDecapTunnelTermEntry()` do today, but driven from `CONFIG_DB:TUNNEL_IPIP` instead of `APP_DB:TUNNEL_DECAP_TABLE`, and managed by TunnelIpipMgr for consistency across all modes.

**Mode 3 flow (encap-only + nexthop):**

Mode 3 uses FRR nexthop-groups with IPIP encapsulation parameters. FRR owns both the nexthop definition and the route, providing full control plane integration (redistribution, withdrawal) without kernel tunnel interfaces.

**FRR nexthop-group architecture:** FRR's `nexthop-group` CLI creates a named config object (`nexthop_group_cmd`) in lib — a config-level grouping mechanism, not a kernel object. When a daemon (PBR, staticd) uses the group, it resolves the name via `nhgc_find()`, expands the individual nexthops, and sends them to zebra. Zebra then creates the kernel nexthop objects and installs the route. This is the established pattern — PBR already uses `set nexthop-group NHGNAME` exactly this way.

For IPIP, the nexthop-group CLI is extended with `encap ip src <IP>` to carry tunnel parameters. When staticd expands the group and sends nexthops to zebra via `zapi_nexthop`, the IPIP encap fields are included. Zebra encodes them as `LWTUNNEL_ENCAP_IP` + `LWTUNNEL_IP_SRC` + `LWTUNNEL_IP_DST` in the kernel nexthop object — the same encoding infrastructure that already exists for EVPN VNI encapsulation.

**FRR configuration (generated by frrcfgd from CONFIG_DB):**

```
nexthop-group IPIP-TO-REMOTE
  nexthop 10.2.0.1 Ethernet0 encap ip src 10.1.0.1
!
ip route 172.16.0.0/16 nexthop-group IPIP-TO-REMOTE
```

ECMP with mixed tunnel and non-tunnel nexthops works naturally:

```
nexthop-group ECMP-MIX
  nexthop 10.0.0.1 Ethernet0
  nexthop 10.2.0.1 Ethernet0 encap ip src 10.1.0.1
  nexthop 10.0.0.3 Ethernet4
!
ip route 172.16.0.0/16 nexthop-group ECMP-MIX
```

**Complete Mode 3 flow:**

```
CONFIG_DB              frrcfgd             FRR (patched)        zebra              fpmsyncd         RouteOrch
─────────              ───────             ─────────────        ─────              ────────         ─────────
TUNNEL_IPIP            Generates           nexthop-group
  mode: encap          FRR config            IPIP-TO-REMOTE
  src_ip: 10.1.0.1    via vtysh              nexthop 10.2.0.1
  dst_ip: 10.2.0.1                           Ethernet0
                                             encap ip
                                             src 10.1.0.1

STATIC_ROUTE           Generates           ip route 172.16/16
  172.16.0.0/16        FRR config          nexthop-group
  → IPIP-TO-REMOTE     via vtysh            IPIP-TO-REMOTE

                                           staticd expands      Creates kernel
                                           NHG → sends          NH with encap ip
                                           zapi_nexthop         + installs route
                                           with IPIP encap      with nhg_id
                                           fields to zebra
                                                                                  Receives route
                                                                Sends route       with nhg_id
                                                                + NHG via FPM     Resolves NH
                                                                                  detects IPIP
                                                                                  encap metadata
                                                                                  adds tunnel_type
                                                                                  to APP_ROUTE
                                                                                                   Sees IPIP
                                                                                                   route →
                                                                                                   TunnelIpipMgr
                                                                                                   → SAI tunnel
                                                                                                   encap NH
```

TunnelIpipMgr creates only the SAI tunnel + encap nexthop (data plane). FRR handles all control plane via frrcfgd-generated config. Clean separation.

**Required changes for Mode 3 (beyond Mode 1):**

- **FRR patch (§5.6)** — extended scope:
  1. Add `encap ip src <IP>` parameter to nexthop-group submode CLI (`lib/nexthop_group.c`)
  2. Add IPIP encap fields to `struct nexthop` and `zapi_nexthop` (following the SRv6 `nh_srv6` pattern)
  3. Encode IPIP encap as `LWTUNNEL_ENCAP_IP` in zebra kernel nexthop messages (`zebra/rt_netlink.c`)
  4. Parse `LWTUNNEL_ENCAP_IP` on read-back in `netlink_nexthop_process_nh()` (already planned)
  5. Add `ip route <prefix> nexthop-group <NAME>` to staticd CLI (following PBR's `set nexthop-group` pattern)
  6. Expand nexthop-group in staticd route install (replicate PBR's `route_add_helper()` pattern)
  7. Extend PBR's `route_add_helper()` (`pbrd/pbr_zebra.c`) to copy IPIP encap fields from `struct nexthop` to `zapi_nexthop` — PBR already uses `set nexthop-group`, only the encap field copy is missing
- **frrcfgd** — listen to `CONFIG_DB:TUNNEL_IPIP` and generate FRR nexthop-group + static route (or PBR policy) config
- **fpmsyncd** — extend `onNextHopMsg()` to parse `NHA_ENCAP` / `NHA_ENCAP_TYPE` and store IPIP encap metadata (tunnel_src, tunnel_dst) alongside the nexthop in `m_nh_groups`. When a route referencing this nexthop arrives, fpmsyncd adds `tunnel_type=IPINIP` and tunnel endpoint fields to the APP_ROUTE_TABLE entry.
- **RouteOrch** — same IPIP nexthop resolution path as Mode 1 (via TunnelIpipMgr). No additional changes.

**Data plane:** Same as Mode 1 encap — packets matching the SAI route hit the SAI tunnel encap nexthop, outer IP header added by VPP/SAI.

**Route sources:** Mode 3 supports two route sources via the same nexthop-group:
- **Static routes** (`ip route <prefix> nexthop-group <NAME>`) — destination-based, redistributable via BGP
- **PBR policies** (`set nexthop-group <NAME>` in pbr-map) — policy-based, match on src/dst IP, protocol, ports; useful for service chaining where traffic steering is based on flow classification rather than destination prefix

**Control plane difference from Mode 1:** No kernel tunnel interface. FRR cannot run routing protocols (BGP/OSPF) *over* the tunnel. But FRR fully owns the route and can redistribute it via BGP (`redistribute static`). No BFD-driven interface state. Failure detection relies on underlay routing convergence or FRR NHT (nexthop tracking) marking the nexthop unreachable.

**Mode 4 flow (symmetric + nexthop):**

Mode 4 combines Mode 3 encap (FRR nexthop-group with IPIP encap) with Mode 2 decap (SAI tunnel term entry). It provides full bidirectional data plane tunneling at unlimited scale, with FRR-managed encap routes.

```
CONFIG_DB              frrcfgd             FRR (patched)          SAI / VPP
─────────              ───────             ─────────────          ─────────
TUNNEL_IPIP            Generates           nexthop-group          SAI tunnel
  mode: symmetric      FRR config            IPIP-TO-REMOTE        + encap NH
  cp_mode: nexthop     for NHG +            nexthop 10.2.0.1       + decap term
  src_ip: 10.1.0.1    static route          encap ip src 10.1.0.1
  dst_ip: 10.2.0.1
                                           ip route 172.16/16
                                           nexthop-group
                                             IPIP-TO-REMOTE

Encap: FRR static route → staticd → zebra → kernel NH + route → FPM → fpmsyncd → RouteOrch → SAI
Decap: IPIP packet arrives → SAI term entry matches → VPP strips outer header (same as Mode 2)
```

Mode 4 has the same FRR, fpmsyncd, and RouteOrch requirements as Mode 3 — no additional changes beyond what Mode 3 requires. The only difference is that TunnelIpipMgr also creates a SAI decap term entry.

**Control plane limitation:** Decap is data-plane only. The control plane cannot receive traffic destined to overlay IPs because there is no kernel tunnel interface to deliver decapsulated packets to the kernel. This is acceptable when the DUT is a transit/forwarding node for decapsulated traffic, not a destination.

**SAI and Control Plane Coordination (Modes 3/4):**

In Modes 3 and 4, the SAI tunnel encap nexthop (data plane) and the FRR route (control plane) are created by separate components — TunnelIpipMgr and frrcfgd/FRR respectively. RouteOrch connects the two when the route arrives.

```
CONFIG_DB:TUNNEL_IPIP entry arrives
          │
          ├──────────────────────────────┐
          │                              │
          ▼                              ▼
   TunnelIpipMgr (orchagent)      frrcfgd (separate daemon)
          │                              │
          │ 1. Create SAI tunnel         │ 3. Generate FRR config:
          │    (IPINIP, src/dst)         │    nexthop-group + static route
          │ 2. Create SAI encap NH       │    Push to FRR via vtysh
          │    (TUNNEL_ENCAP type)       │
          │ Store: tunnel_name →         │
          │   SAI NH OID mapping         │
          │                              ▼
          │                       FRR → zebra → kernel NH + route
          │                              │
          │                       zebra sends via FPM
          │                              │
          │                              ▼
          │                       fpmsyncd receives route
          │                       detects IPIP encap metadata
          │                       writes APP_ROUTE_TABLE with
          │                         tunnel_type, tunnel_src, tunnel_dst
          │                              │
          │                              ▼
          │                       RouteOrch receives IPIP route
          │                              │
          └──────────────────────────────┘
                                         │
                                         ▼
                                  RouteOrch matches tunnel_src/dst
                                  to TunnelIpipMgr's SAI tunnel NH
                                         │
                                         ▼
                                  SAI route → SAI tunnel encap NH
                                  VPP encapsulates packets
```

**Ordering:** TunnelIpipMgr processes `CONFIG_DB:TUNNEL_IPIP` immediately in orchagent. The route takes a longer path: frrcfgd → FRR → zebra → kernel → FPM → fpmsyncd → RouteOrch. This naturally ensures TunnelIpipMgr creates the SAI tunnel nexthop before RouteOrch needs it.

**Race handling:** If a route arrives before TunnelIpipMgr has processed the tunnel (e.g., during startup or rapid config changes), RouteOrch fails to find the SAI nexthop. RouteOrch retries using the existing unresolved nexthop retry mechanism — the route stays in `m_toSync` and is reprocessed on the next cycle when TunnelIpipMgr signals tunnel creation.

`TunnelIpipMgr` implements `BfdOrch::Observer` to receive BFD state change notifications. When BFD transitions to DOWN, it sets the kernel tunnel interface down (`ip link set tun0 down`). FRR detects the interface state change and withdraws routes. When BFD recovers, the reverse occurs. BFD is only applicable to Mode 1 (interface-based tunnels).

#### 5.5.2. Dual-Path Architecture

For Mode 1 (Interface Mode), each IPIP tunnel has two parallel paths:

```
              ┌──────────────────────────────────────────┐
              │             TunnelIpipMgr                 │
              └──────────┬──────────────────┬────────────┘
                         │                  │
            SAI Path     │                  │   Kernel Path
         (Data Plane)    │                  │  (Control Plane)
                         ▼                  ▼
              ┌────────────────┐  ┌────────────────────┐
              │  SAI ipip0     │  │  Kernel tun0       │
              │  Line-rate     │  │  CP traffic        │
              │  forwarding    │  │  FRR routing       │
              └────────────────┘  └────────────────────┘
```

The SAI tunnel handles all data plane traffic at line rate. The kernel tunnel enables control plane participation — FRR can assign IP addresses to `tun0`, run BGP sessions over it, and advertise routes via the tunnel interface. Both paths share the same tunnel endpoints but operate independently.

#### 5.5.3. BFD-Based Tunnel State Management

Neither VPP nor Linux IPIP tunnels automatically track destination reachability — a tunnel interface stays UP even when the remote endpoint is unreachable. BFD provides active probing to detect failures.

**BFD configuration is optional per tunnel**:

- **BFD enabled**: Tunnel kernel interface state tracks BFD session state. BFD DOWN causes `ip link set tun0 down`, which triggers FRR route withdrawal. Sub-second convergence.
- **BFD disabled**: Tunnel always UP (legacy behavior, like Linux default). Suitable for simple/test environments.

BFD was chosen over VPP FIB tracking because FIB-based tracking would incorrectly mark tunnels UP when routes point to drop/reject nexthops. BFD uses active end-to-end probing and correctly detects all failure modes including firewall blocks and path MTU issues.

| Feature | BFD | Static (No Monitoring) |
|---------|-----|------------------------|
| **Detects peer failure** | Yes (active probing) | No |
| **Detects route loss** | Yes | No |
| **Detects firewall block** | Yes | No |
| **Convergence time** | Sub-second | N/A (never converges) |
| **Overhead** | Low (3–5 small UDP packets/sec) | None |
| **Standard** | RFC 5880/5881 | N/A |

#### 5.5.4. CONFIG_DB Schema

**New table:** `TUNNEL_IPIP|<tunnel_name>`

##### 5.5.4.1. Field Definitions

| Field | Type | Required | Values | Default | Description |
|-------|------|----------|--------|---------|-------------|
| `tunnel_type` | String | Yes | `IPINIP` | — | Tunnel encapsulation type. Always `IPINIP` for this table. Included for consistency with existing SONiC tunnel tables. |
| `src_ip` | IP address | Yes | IPv4 or IPv6 | — | Local tunnel endpoint. Maps to `SAI_TUNNEL_ATTR_ENCAP_SRC_IP`. |
| `dst_ip` | IP address | Conditional | IPv4 or IPv6 | — | Remote tunnel endpoint. Maps to `SAI_TUNNEL_ATTR_ENCAP_DST_IP`. Required when `peer_mode=P2P`. Omitted when `peer_mode=P2MP` (remote determined per-nexthop). |
| `peer_mode` | String | Yes | `P2P`, `P2MP` | — | SAI peer mode. `P2P` = fixed remote endpoint. `P2MP` = remote endpoint varies per route nexthop. Maps to `SAI_TUNNEL_ATTR_PEER_MODE`. |
| `mode` | String | Yes | `encap`, `decap`, `symmetric` | — | Tunnel direction. `encap` = encapsulation only (Mode 3). `decap` = decapsulation only (Mode 2). `symmetric` = both directions (Mode 1 or 4). |
| `cp_mode` | String | No | `interface`, `nexthop` | `interface` | Control plane approach (see §4.1.3). `interface` = kernel `tun0` per tunnel (Mode 1). `nexthop` = kernel nexthop object with `encap ip` (Modes 3/4, requires FRR patch §5.6). |
| `dscp_mode` | String | No | `uniform`, `pipe` | `uniform` | DSCP handling for encap and decap. `uniform` = copy between inner/outer. `pipe` = use fixed value (see `dscp`). |
| `dscp` | Integer | No | 0–63 | `0` | Fixed outer DSCP value for encap when `dscp_mode=pipe`. Ignored when `dscp_mode=uniform`. |
| `ecn_mode` | String | No | `copy_from_outer`, `standard` | `standard` | ECN decap handling. `copy_from_outer` = inner ECN set to outer. `standard` = RFC 3168 compliant. |
| `encap_ecn_mode` | String | No | `standard` | `standard` | ECN encap handling. Only `standard` is supported currently. |
| `ttl_mode` | String | No | `uniform`, `pipe` | `pipe` | TTL handling for encap and decap. `uniform` = copy between inner/outer. `pipe` = use fixed value (see `ttl`). |
| `ttl` | Integer | No | 1–255 | `255` | Fixed outer TTL value for encap when `ttl_mode=pipe`. Ignored when `ttl_mode=uniform`. |
| `bfd_enable` | String | No | `true`, `false` | `false` | Enable BFD tunnel state tracking. Only valid when `cp_mode=interface`. |
| `bfd_min_tx` | Integer | No | 50000–30000000 | `300000` | BFD minimum TX interval in microseconds. Ignored when `bfd_enable=false`. |
| `bfd_min_rx` | Integer | No | 50000–30000000 | `300000` | BFD minimum RX interval in microseconds. Ignored when `bfd_enable=false`. |
| `bfd_detect_mult` | Integer | No | 1–255 | `3` | BFD detection multiplier. Ignored when `bfd_enable=false`. |
| `tunnel_ip` | IP address | No | IPv4 or IPv6 | `src_ip/32` | IP address to assign to the kernel tunnel interface (`iptun<N>`). Only valid when `cp_mode=interface`. Defaults to `src_ip/32` if omitted. Override when a different routable address is needed on the tunnel interface. |

##### 5.5.4.2. How Fields Determine the Operational Mode

The combination of `mode` and `cp_mode` determines which operational mode (§5.1) is used, what kernel objects are created, and what SAI objects are programmed:

| `mode` | `cp_mode` | Operational Mode | Kernel Object | SAI Objects Created |
|--------|-----------|------------------|---------------|---------------------|
| `symmetric` | `interface` | **Mode 1** (Interface) | kernel `tun0` | Tunnel + encap nexthop + decap term entry |
| `decap` | *(ignored)* | **Mode 2** (Decap-only) | none | Tunnel + decap term entry |
| `encap` | `nexthop` | **Mode 3** (Encap-only) | kernel nexthop obj | Tunnel + encap nexthop |
| `symmetric` | `nexthop` | **Mode 4** (Bidirectional) | kernel nexthop obj | Tunnel + encap nexthop + decap term entry |
| `encap` | `interface` | **Mode 1 variant** | kernel `tun0` | Tunnel + encap nexthop *(no decap term)* |

##### 5.5.4.3. Validation Rules

- `src_ip` and `dst_ip` must be the same address family (both IPv4 or both IPv6)
- `dst_ip` is required when `peer_mode=P2P`, must be omitted when `peer_mode=P2MP`
- No duplicate `src_ip` + `dst_ip` pairs across tunnel entries
- `cp_mode=nexthop` requires the FRR patch (§5.6); TunnelIpipMgr logs a warning if used without it
- `bfd_enable=true` is only valid when `cp_mode=interface` (nexthop-based tunnels have no interface to bring down)
- `dscp` field is only meaningful when `dscp_mode=pipe`
- `ttl` field is only meaningful when `ttl_mode=pipe`
- `bfd_min_tx` / `bfd_min_rx`: 50,000–30,000,000 microseconds (50ms–30s)
- `bfd_detect_mult`: 1–255

##### 5.5.4.4. Configuration Examples

**Example 1 — P2P symmetric tunnel with interface mode and BFD (Mode 1):**

```bash
redis-cli HSET "TUNNEL_IPIP|Tunnel1" \
    "tunnel_type" "IPINIP" \
    "src_ip" "10.1.0.1" \
    "dst_ip" "10.2.0.1" \
    "peer_mode" "P2P" \
    "mode" "symmetric" \
    "cp_mode" "interface" \
    "dscp_mode" "uniform" \
    "ttl_mode" "pipe" \
    "ttl" "64" \
    "bfd_enable" "true" \
    "bfd_min_tx" "300000" \
    "bfd_min_rx" "300000" \
    "bfd_detect_mult" "3"
```

This creates:
- SAI tunnel with encap + decap
- Kernel `tun0` interface (FRR can run BGP over it)
- BFD session monitoring 10.2.0.1 (tun0 goes down on BFD failure)

**Example 2 — Decap-only tunnel, no BFD (Mode 2):**

```bash
redis-cli HSET "TUNNEL_IPIP|DecapTunnel" \
    "tunnel_type" "IPINIP" \
    "src_ip" "10.1.0.32" \
    "peer_mode" "P2MP" \
    "mode" "decap" \
    "dscp_mode" "pipe" \
    "ecn_mode" "copy_from_outer" \
    "ttl_mode" "pipe"
```

This creates:
- SAI tunnel with decap term entry only (accept from any source)
- No kernel interface, no BFD

**Example 3 — P2P encap-only tunnel with nexthop mode (Mode 3):**

```bash
redis-cli HSET "TUNNEL_IPIP|EncapTunnel" \
    "tunnel_type" "IPINIP" \
    "src_ip" "10.1.0.1" \
    "dst_ip" "10.4.0.1" \
    "peer_mode" "P2P" \
    "mode" "encap" \
    "cp_mode" "nexthop" \
    "ttl_mode" "pipe" \
    "ttl" "64"
```

This creates:
- SAI tunnel with encap nexthop
- Kernel nexthop object: `ip nexthop add id <auto> encap ip dst 10.4.0.1 src 10.1.0.1 dev <underlay>`
- No kernel interface (unlimited scale)
- Requires FRR patch (§5.6)

**Example 4 — Simple symmetric tunnel, no BFD (Mode 1, minimal config):**

```bash
redis-cli HSET "TUNNEL_IPIP|SimpleTunnel" \
    "tunnel_type" "IPINIP" \
    "src_ip" "10.1.0.1" \
    "dst_ip" "10.2.0.1" \
    "peer_mode" "P2P" \
    "mode" "symmetric"
```

This uses all defaults: `cp_mode=interface`, `dscp_mode=uniform`, `ttl_mode=pipe`, `ttl=255`, `bfd_enable=false`. Tunnel is always UP.

#### 5.5.5. NhgOrch and RouteOrch Integration

IPIP tunnel nexthops are integrated into the existing nexthop resolution path, following the VXLAN pattern:

1. `NhgOrch::isValidNexthop()` checks `TunnelIpipMgr` for tunnel existence when `nh.isTunnelNexthop()` returns true
2. `NhgOrch::getNexthopOid()` retrieves the SAI nexthop OID from `TunnelIpipMgr::getTunnelNexthop()`
3. `RouteOrch::resolveNexthop()` resolves tunnel nexthop names to SAI OIDs via `TunnelIpipMgr`

Routes can reference IPIP tunnels as nexthops:

```bash
redis-cli HSET "ROUTE_TABLE:172.16.0.0/16" \
    "nexthop" "Tunnel1" \
    "ifname" "Tunnel1"
```

### 5.6. Phase 3: FRR Patch for Kernel Nexthop IPIP Encapsulation

**Files to modify:** `sonic-buildimage/src/sonic-frr/frr/zebra/rt_netlink.c`, `sonic-buildimage/src/sonic-frr/frr/zebra/rt_netlink.h`, `sonic-buildimage/src/sonic-frr/frr/zebra/zebra_nhg.h`, `sonic-buildimage/src/sonic-frr/frr/zebra/zebra_vty.c`

#### 5.6.1. Root Cause — FRR Only Handles MPLS Encapsulation

Kernel nexthop objects with IPIP encapsulation work correctly:

```bash
$ sudo ip nexthop add id 999 encap ip dst 10.2.0.1 src 10.1.0.1 dev Ethernet0
$ sudo ip nexthop show id 999
id 999 encap ip id 0 src 10.1.0.1 dst 10.2.0.1 ttl 0 tos 0 dev Ethernet0

$ sudo ip route add 1.2.3.0/24 nhid 999
$ sudo ip -j route show 1.2.3.0/24
[{"dst":"1.2.3.0/24","nhid":999,"encap":"ip","src":"10.1.0.1","dst":"10.2.0.1",...}]
```

However, FRR 10.3 misinterprets these routes:

```bash
$ docker exec bgp vtysh -c 'show ip route 1.2.3.0/24'
Routing entry for 1.2.3.0/24
  Known via "kernel", distance 0, metric 0, best
  * directly connected, Ethernet0
```

FRR shows "directly connected, Ethernet0" — the IPIP encapsulation is completely lost. The root cause is in `netlink_nexthop_process_nh()` in `zebra/rt_netlink.c` (lines 3563–3575):

```c
if (tb[NHA_ENCAP] && tb[NHA_ENCAP_TYPE]) {
    uint16_t encap_type = *(uint16_t *)RTA_DATA(tb[NHA_ENCAP_TYPE]);
    int num_labels = 0;
    mpls_label_t labels[MPLS_MAX_LABELS] = {0};

    if (encap_type == LWTUNNEL_ENCAP_MPLS)  // ← ONLY HANDLES MPLS
        num_labels = parse_encap_mpls(tb[NHA_ENCAP], labels);

    // NO HANDLING FOR LWTUNNEL_ENCAP_IP

    if (num_labels)
        nexthop_add_labels(&nh, ZEBRA_LSP_STATIC, num_labels, labels);
}
```

FRR has had nexthop object support since version 7.3, but only for MPLS encapsulation. `LWTUNNEL_ENCAP_IP` is silently ignored.

#### 5.6.2. Required FRR Changes

The FRR patch has two scopes: a **core scope** required for all nexthop-based modes (Modes 3/4), and an **extended scope** that enables FRR-native IPIP nexthop management.

**Core scope (zebra — kernel nexthop IPIP parsing):**

1. **Add `parse_encap_ip()` function** (`zebra/rt_netlink.c`): Parse `LWTUNNEL_IP_SRC` and `LWTUNNEL_IP_DST` attributes from the `NHA_ENCAP` netlink attribute, storing tunnel source/destination addresses.

2. **Extend `netlink_nexthop_process_nh()`** (`zebra/rt_netlink.c`): Add an `else if (encap_type == LWTUNNEL_ENCAP_IP)` branch that calls `parse_encap_ip()` and sets `nh.encap.encap_type = ENCAP_TYPE_IPIP`.

3. **Extend nexthop encap structure** (`zebra/zebra_nhg.h`): Add `ENCAP_TYPE_IPIP` to the `encap_type_e` enum and add `tunnel_src`/`tunnel_dst` fields (`struct in6_addr`) to store tunnel endpoints.

4. **Update display code** (`zebra/zebra_vty.c`): When `encap_type == ENCAP_TYPE_IPIP`, display tunnel encapsulation info in `show ip route` output.

**Extended scope (nexthop-group + staticd — FRR-native IPIP routes):**

5. **Add `encap ip src <IP>` to nexthop-group CLI** (`lib/nexthop_group.c`): Extend the `ecmp_nexthops_cmd` optional parameter list (which already supports `label`, `vni`, `weight`) with `encap ip src A.B.C.D$encap_src`. Store in a new IPIP encap field in `struct nexthop` (following the `nh_srv6` pattern for SRv6).

6. **Add IPIP encap fields to `zapi_nexthop`** (`lib/zclient.h`): Add `ZAPI_NEXTHOP_FLAG_IPIP_ENCAP` flag and `ipip_src`/`ipip_dst` fields. Encode/decode in zapi message serialization.

7. **Encode IPIP encap in zebra kernel nexthop messages** (`zebra/rt_netlink.c`): When a nexthop has IPIP encap fields, encode `NHA_ENCAP_TYPE = LWTUNNEL_ENCAP_IP` + `NHA_ENCAP` containing `LWTUNNEL_IP_SRC` / `LWTUNNEL_IP_DST`. The encoding infrastructure already exists for EVPN VNI (`LWTUNNEL_ENCAP_IP`) and MPLS (`LWTUNNEL_ENCAP_MPLS`).

8. **Add `ip route <prefix> nexthop-group <NAME>` to staticd** (`staticd/static_vty.c`): New CLI variant for static routes referencing nexthop-groups by name. Staticd resolves the name via `nhgc_find()`, expands individual nexthops, and sends to zebra — following the established pattern used by PBR's `set nexthop-group NHGNAME`.

9. **Expand nexthop-group in staticd route install** (`staticd/static_zebra.c`): Replicate PBR's `route_add_helper()` pattern — iterate nexthop-group members, fill `zapi_nexthop` array including IPIP encap fields, send to zebra via `zclient_route_send()`.

**Expected result after patch:**

```
! Define tunnel nexthop group
nexthop-group IPIP-TO-REMOTE
  nexthop 10.2.0.1 Ethernet0 encap ip src 10.1.0.1

! ECMP with mixed tunnel and plain nexthops
nexthop-group ECMP-MIX
  nexthop 10.0.0.1 Ethernet0
  nexthop 10.2.0.1 Ethernet0 encap ip src 10.1.0.1
  nexthop 10.0.0.3 Ethernet4

! Static route via nexthop-group
ip route 172.16.0.0/16 nexthop-group IPIP-TO-REMOTE

! PBR policy steering traffic through IPIP tunnel
pbr-map STEER-TO-FW seq 10
  match src-ip 192.168.0.0/16
  set nexthop-group IPIP-TO-REMOTE
!
interface Ethernet0
  pbr-policy STEER-TO-FW

! show ip route displays tunnel info
$ show ip route 172.16.0.0/16
Routing entry for 172.16.0.0/16
  Known via "static", distance 1, metric 0, best
  * via 10.2.0.1, Ethernet0, encap ip src 10.1.0.1, weight 1
```

**Upstream feasibility:** Each change has a working precedent in FRR — VNI in nexthop-group (existing), SRv6 encap in nexthop struct (existing), LWTUNNEL_ENCAP_IP encoding in zebra (existing for EVPN), PBR nexthop-group expansion (existing). The design is generic and extends to other tunnel types (GRE, SRv6) by adding additional encap parameters to the same CLI framework.

#### 5.6.3. Impact on Tunnel Modes

| Mode | FRR Patch Required? | Reason |
|------|---------------------|--------|
| **1. Interface** | No | Uses kernel `tun0` interface, not nexthop objects |
| **2. Decap-only** | No | No control plane encap involvement |
| **3. Encap-only** | **Yes** | Uses kernel nexthop objects with `encap ip` |
| **4. Bidirectional** | **Yes** | CP encap uses kernel nexthop objects with `encap ip` |

**Workaround**: Use Mode 1 (Interface Mode) for initial deployment without the FRR patch. Mode 1 works with standard kernel tunnel interfaces but is limited to ~1000 tunnels.

### 5.7. Implementation Considerations

#### 5.7.1. YANG Model

A new `sonic-tunnel-ipip.yang` model is required for CONFIG_DB validation of the `TUNNEL_IPIP` table. This is separate from the existing `sonic-tunnel.yang` which is dual-ToR specific (key pattern `MuxTunnel[0-9]+`, `src_ip` is a leafref to `PEER_SWITCH`).

The new YANG model defines:
- Key: tunnel name matching pattern `[a-zA-Z][a-zA-Z0-9_-]+` (excludes `MuxTunnel*` to avoid conflict with dual-ToR)
- All fields from §5.5.4.1 with types, ranges, and defaults
- Validation constraints from §5.5.4.3 (address family match, conditional `dst_ip`, BFD constraints)

**File:** `sonic-yang-models/yang-models/sonic-tunnel-ipip.yang`

#### 5.7.2. Kernel Interface Naming (Mode 1)

For Mode 1, TunnelIpipMgr creates kernel IPIP tunnel interfaces. The naming scheme is `iptun<N>` where N is auto-incremented starting from 0 (e.g., `iptun0`, `iptun1`, `iptun2`).

TunnelIpipMgr maintains the mapping between CONFIG_DB tunnel name and kernel interface name, and publishes it in STATE_DB:

```
STATE_TUNNEL_IPIP_TABLE|Tunnel1 → kernel_ifname=iptun0, state=up, ...
STATE_TUNNEL_IPIP_TABLE|Tunnel2 → kernel_ifname=iptun1, state=up, ...
```

#### 5.7.3. Kernel Interface IP Assignment (Mode 1)

For FRR to run routing protocols over the tunnel interface or for the interface to be routable, it needs an IP address. TunnelIpipMgr assigns an IP to the kernel interface after creation:

- **Default:** `src_ip/32` is assigned to `iptun<N>` — the tunnel's local endpoint becomes the interface address, matching the Cisco/Arista convention where the tunnel source is the routable address
- **Override:** If `tunnel_ip` is specified in `TUNNEL_IPIP`, that address is used instead

TunnelIpipMgr executes: `swss::exec("ip addr add <tunnel_ip> dev iptun<N>")`

#### 5.7.4. Coexistence with Existing Dual-ToR Tunnel Path

The new `TUNNEL_IPIP` table and the existing `CFG_TUNNEL_TABLE` serve different use cases and must not overlap:

| Table | Owner | Use Case | Key Pattern |
|-------|-------|----------|-------------|
| `CFG_TUNNEL_TABLE` | TunnelMgr + TunnelDecapOrch | Dual-ToR | `MuxTunnel[0-9]+` |
| `TUNNEL_IPIP` | TunnelIpipMgr | Generic IPIP | `[a-zA-Z][a-zA-Z0-9_-]+` (excludes `MuxTunnel*`) |

The YANG models enforce disjoint key patterns. Users configure generic IPIP tunnels exclusively via `TUNNEL_IPIP`. Dual-ToR tunnels continue using `CFG_TUNNEL_TABLE`. No runtime conflict detection is needed.

#### 5.7.5. Warm Restart

Warm restart handling for TunnelIpipMgr is deferred to a future revision. Initial deployment supports cold restart only. Key considerations for future warm restart support:

- SAI tunnel and nexthop objects must be reconciled with pre-existing ASIC state after restart
- Kernel tunnel interfaces (`iptun<N>`) and nexthop objects must be preserved across restart
- BFD sessions must be re-established without triggering tunnel state flap
- FRR-driven routes (Modes 3/4) are handled by FRR's own warm restart mechanism

## 6. Expected Impact

### Phase 1 — Decapsulation

Unblocks IPIP decap use cases on VPP with minimal changes (single file, `TunnelManager.cpp`). The existing SONiC control plane (`tunneldecaporch`, `ipinip.json.j2`) works unmodified. All four protocol combinations (IPv4-in-IPv4, IPv6-in-IPv6, IPv4-in-IPv6, IPv6-in-IPv4) are supported.

### Phase 2 — Encapsulation with BFD

Enables full IPIP tunnel encapsulation with:

| Capability | Impact |
|------------|--------|
| Tunnel lifecycle management | Centralized via TunnelIpipMgr |
| Tunnel state tracking | Sub-second failure detection via BFD |
| FRR route withdrawal on failure | Automatic via kernel interface state |
| ECMP via tunnel nexthops | NhgOrch integration |
| Platform independence | SAI-based, works on VPP and hardware ASICs |

### Phase 3 — FRR Patch

Removes the ~1000 tunnel scale limit by enabling Modes 3 and 4, which use kernel nexthop objects instead of kernel interfaces. No upper bound on tunnel count.

## 7. Testing

### Decapsulation Tests (Phase 1)

- **P2P decap**: Create tunnel term entry with specific `src_ip`/`dst_ip`, send encapsulated packets, verify decapsulation and inner packet forwarding
- **P2MP decap**: Create tunnel term entry with only `dst_ip`, verify packets from any source are decapsulated
- **Protocol matrix**: IPv4-in-IPv4, IPv6-in-IPv6, IPv4-in-IPv6, IPv6-in-IPv4
- **QoS header handling**: All DSCP (uniform/pipe), ECN (copy/standard), TTL (pipe/uniform) modes
- **Existing test suite**: `pytest decap/test_decap.py` from sonic-mgmt

### Encapsulation Tests (Phase 2)

- **P2P encap**: Create tunnel with fixed `dst_ip`, add route via tunnel, verify encapsulated packets on egress
- **P2MP encap**: Create tunnel without `dst_ip`, add multiple routes with different nexthops, verify correct outer dst per route
- **BFD integration**: Create tunnel with BFD, simulate peer failure, verify kernel interface goes down and FRR withdraws routes
- **BFD recovery**: Restore peer, verify kernel interface comes up and FRR re-advertises routes
- **Tunnel without BFD**: Verify tunnel stays UP regardless of reachability (legacy behavior)
- **ECMP**: Multiple tunnel nexthops, verify load balancing

### FRR Patch Tests (Phase 3)

- **Nexthop object**: Create kernel nexthop with `encap ip`, verify `show ip route` displays tunnel info
- **Route redistribution**: Verify routes with IPIP nexthops are correctly redistributed
- **IPv4 and IPv6**: Test with both address families

### Scale and Negative Tests

- **Scale**: 256+ tunnels, 1000+ routes per tunnel
- **Error handling**: Invalid configuration rejection, tunnel deletion with active routes, MTU exceeded, loop prevention via TTL

### Regression

- **VXLAN**: Verify no regression in existing VXLAN tunnel functionality
- **Warm restart**: Verify tunnels survive warm reboot
- **Config reload**: Verify tunnels restored after config reload

## 8. Links to Code PRs

TBD
