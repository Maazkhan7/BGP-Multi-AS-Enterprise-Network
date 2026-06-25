# 🌍 Cisco Lab: BGP Multi-AS Enterprise Network

> **Tools:** Cisco Packet Tracer · BGP (eBGP) · AS 64100 · AS 64200 · AS 64300 · Router-on-a-Stick · 6 VLANs · DHCP Server · HTTP Server · Port Security (Sticky MAC) · TCP Port 179

---

## 📌 Project Overview

**Project** is the most advanced routing lab in this series — implementing **BGP (Border Gateway Protocol)**, the routing protocol that powers the entire internet. Three **Autonomous Systems** (BRANCH-A: AS 64100, ISP: AS 64200, BRANCH-B: AS 64300) are interconnected using **eBGP (External BGP)** over serial WAN links. Each branch has its own BGP AS number, advertises its local VLANs into BGP, and learns remote prefixes through the ISP as the transit AS.

A centralized **DHCP server** (10.1.1.2) serves all 6 VLANs via DHCP relay, an **HTTP server** (20.1.1.2) hosts the "BGP CONFIG" web page reachable from any VLAN, and **Port Security with Sticky MAC** protects all access ports. This lab bridges the gap between CCNA-level IGPs and real-world internet routing.

---

## 🗺️ Network Topology

```
                    ┌──────────────────────────────────────────────┐
                    │                ISP (Router0)                 │
                    │              AS 64200                        │
                    │   10.1.1.1/24 ──── DHCP-SERVER (10.1.1.2)  │
                    │   20.1.1.1/24 ──── HTTP-SERVER (20.1.1.2)  │
                    └──────────┬────────────────────┬─────────────┘
                          Serial2/0             Serial3/0
                         1.1.1.0/24            2.1.1.0/24
                    eBGP AS64100          eBGP AS64300
                              │                    │
                    ┌─────────┴──────┐   ┌─────────┴──────┐
                    │  BRANCH-A      │   │  BRANCH-B      │
                    │  Router2       │   │  Router1       │
                    │  AS 64100      │   │  AS 64300      │
                    └────────┬───────┘   └────────┬───────┘
                             │                    │
                          Switch0 (SW-1)       Switch1
                        /    |    \           /    |    \
                    Fa0/1  Fa0/2  Fa0/3   Fa0/1  Fa0/2  Fa0/3
                     PC0    PC1    PC2     PC3    PC4    PC5

BRANCH-A (AS 64100): VLAN 10 (192.168.1.0/24) · VLAN 20 (192.168.2.0/24) · VLAN 30 (192.168.3.0/24)
BRANCH-B (AS 64300): VLAN 40 (192.168.4.0/24) · VLAN 50 (192.168.5.0/24) · VLAN 60 (192.168.6.0/24)
```

### IP Addressing Table

| Device      | Interface        | IP Address      | AS    | Purpose               |
|-------------|------------------|-----------------|-------|-----------------------|
| ISP         | Serial 2/0       | 1.1.1.2/24      | 64200 | eBGP link to BRANCH-A |
| ISP         | Serial 3/0       | 2.1.1.2/24      | 64200 | eBGP link to BRANCH-B |
| ISP         | FastEthernet 0/0 | 10.1.1.1/24     | 64200 | DHCP Server segment   |
| ISP         | FastEthernet 1/0 | 20.1.1.1/24     | 64200 | HTTP Server segment   |
| BRANCH-A    | Serial 2/0       | 1.1.1.1/24      | 64100 | eBGP WAN to ISP       |
| BRANCH-A    | Fa0/0.10         | 192.168.1.1/24  | 64100 | VLAN 10 gateway       |
| BRANCH-A    | Fa0/0.20         | 192.168.2.1/24  | 64100 | VLAN 20 gateway       |
| BRANCH-A    | Fa0/0.30         | 192.168.3.1/24  | 64100 | VLAN 30 gateway       |
| BRANCH-B    | Serial 3/0       | 2.1.1.1/24      | 64300 | eBGP WAN to ISP       |
| BRANCH-B    | Fa0/0.40         | 192.168.4.1/24  | 64300 | VLAN 40 gateway       |
| BRANCH-B    | Fa0/0.50         | 192.168.5.1/24  | 64300 | VLAN 50 gateway       |
| BRANCH-B    | Fa0/0.60         | 192.168.6.1/24  | 64300 | VLAN 60 gateway       |
| DHCP-SERVER | FastEthernet 0   | 10.1.1.2/24     | —     | Centralized DHCP      |
| HTTP-SERVER | FastEthernet 0   | 20.1.1.2/24     | —     | Web "BGP CONFIG"      |

---

## 🌍 BGP Deep Dive — Full Explanation

### What is BGP?

**BGP (Border Gateway Protocol)** is the **Exterior Gateway Protocol (EGP)** that routes traffic between **Autonomous Systems** on the internet. Defined in **RFC 4271**, it is the only EGP in use today — literally running the global internet. Every ISP, cloud provider, and large enterprise uses BGP.

| Feature | BGP | OSPF | EIGRP | RIP |
|---|---|---|---|---|
| Type | Path-Vector EGP | Link-State IGP | Hybrid IGP | DV IGP |
| Scope | Between AS | Within AS | Within AS | Within AS |
| Metric | AS-PATH + attributes | Cost (BW) | BW+Delay | Hop count |
| Admin Distance | **20 (eBGP)** / 200 (iBGP) | 110 | 90 | 120 |
| Transport | **TCP Port 179** | IP Proto 89 | IP Proto 88 | UDP 520 |
| Scalability | Internet-scale (900K+ routes) | Large enterprise | Medium | Small |
| Neighbor | **Manually configured** | Auto-discovered | Auto-discovered | Auto |

> **Critical:** BGP neighbors are **manually defined** with `neighbor x.x.x.x remote-as` — no automatic discovery. You explicitly choose your peers, making BGP fully policy-controlled.

---

### BGP Autonomous System Numbers

| AS Type | Range | Usage |
|---|---|---|
| Public ASN (16-bit) | 1 – 64511 | IANA/RIR assigned for internet |
| **Private ASN (16-bit)** | **64512 – 65535** | Private use (this lab) |
| Public ASN (32-bit) | 65536 – 4294967295 | Extended range |

**This lab:** AS 64100 (BRANCH-A) · AS 64200 (ISP) · AS 64300 (BRANCH-B) — all private range.

---

### eBGP vs iBGP

| | eBGP (External BGP) | iBGP (Internal BGP) |
|---|---|---|
| Between | **Different AS** — this lab | Same AS number |
| Admin Distance | **20** | 200 |
| TTL | 1 (must be directly connected) | 255 |
| Next-hop | Changed at AS boundary | Unchanged |

---

### BGP Path Attributes

BGP selects paths using **attributes**, not simple metrics:

| Attribute | Description | In This Lab |
|---|---|---|
| **AS-PATH** | List of ASes route traversed — shorter wins | 64200 i = 1 hop, 64200 64300 i = 2 hops |
| **NEXT-HOP** | Next router IP | 1.1.1.2 (ISP) for remote routes |
| **WEIGHT** | Cisco-local, not advertised — higher wins | 32768 for locally originated routes |
| **LOCAL-PREF** | iBGP preference — higher wins | 0 (default, no iBGP here) |
| **MED** | Metric to neighbor AS — lower wins | 0 in this lab |
| **ORIGIN** | i=IGP, e=EGP, ?=incomplete | i (all via network command) |

---

### BGP 6 Session States — Full Explanation

BGP uses **TCP port 179** and progresses through 6 states:

#### ✅ State 1 — IDLE
Initial state. BGP process started but no connection attempted. Waiting for a start event. Router enters this state on boot or after session error.

**Leave condition:** BGP process enabled + `neighbor` command entered → moves to CONNECT

#### ✅ State 2 — CONNECT
BGP is initiating a **TCP three-way handshake** to neighbor on **port 179**. ConnectRetry timer running. If TCP succeeds → OPEN SENT. If TCP fails → ACTIVE.

**In this lab:** Serial link (1.1.1.0/24) provides L3 reachability for TCP.

```cisco
! TCP must reach neighbor before BGP can connect
! Serial2/0: 1.1.1.1 ←→ 1.1.1.2 (ISP)
```

#### ✅ State 3 — ACTIVE
TCP connection failed. BGP actively retrying. **Most common troubleshooting state.**

Common causes of being stuck in ACTIVE:
- Wrong neighbor IP in `neighbor` command
- AS number mismatch (`remote-as` wrong)
- ACL blocking TCP port 179
- No IP route to neighbor
- Interface down

#### ✅ State 4 — OPEN SENT
TCP connected. BGP sends **OPEN message** containing:
- BGP version (4)
- Local AS number
- Hold time (180 sec default)
- BGP Router ID (highest IP or loopback)
- Capabilities (route refresh, IPv4 unicast)

Waits for neighbor's OPEN.

**From `show ip bgp neighbors`:**
```
BGP version 4, remote router ID 20.1.1.1
```

#### ✅ State 5 — OPEN CONFIRM
Both OPEN messages validated. BGP sends **KEEPALIVE** to confirm. Parameters verified:
- BGP version matches ✅
- `remote-as` matches neighbor's actual AS ✅
- Hold time negotiated (minimum of both) ✅

If any mismatch → NOTIFICATION sent → back to IDLE.

#### ✅ State 6 — ESTABLISHED ← THE GOAL
Session fully up. Both KEEPALIVEs exchanged. Routers begin sending **UPDATE messages** with prefixes and path attributes. Routes installed with `B` prefix (AD 20 for eBGP).

**Confirmed in this lab:**
```
%BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up   ← BRANCH-A ✅
%BGP-5-ADJCHANGE: neighbor 2.1.1.1 Up   ← BRANCH-B ✅

BRANCH-A#show ip bgp summary
Neighbor   V    AS    MsgRcvd  MsgSent  Up/Down    State/PfxRcd
1.1.1.2    4  64200    42       36      00:34:58       4         ✅
```

---

### BGP Message Types

| # | Message | Purpose | In This Lab |
|---|---|---|---|
| 1 | **OPEN** | Establish session — AS, version, hold time, router ID | Sent 1, Rcvd 1 |
| 2 | **UPDATE** | Advertise/withdraw prefixes with path attributes | Sent 3, Rcvd 6 |
| 3 | **NOTIFICATION** | Report error, close session | Sent 0, Rcvd 0 ✅ |
| 4 | **KEEPALIVE** | Maintain session (every 60 sec) | Sent 37, Rcvd 37 |

Keepalive = Hold time / 3 = 180 / 3 = **60 seconds**. If no keepalive in 180 sec → session drops.

---

### BGP TCP Session Detail

```
Connection state is ESTAB
Local host:   1.1.1.1, Local port: 179
Foreign host: 1.1.1.2, Foreign port: 1025
SRTT: 259 ms, RTTO: 579 ms
minRTT: 16 ms, maxRTT: 300 ms
Connections established 1; dropped 2
```

BGP rides on **reliable TCP** — unlike OSPF/EIGRP which run directly over IP. TCP guarantees ordered delivery. Port 179 is well-known for BGP.

---

## ⚙️ Full Configuration

### ISP Router (Router0) — AS 64200

```cisco
ISP(config)#router bgp 64200
ISP(config-router)#network 10.1.1.0
ISP(config-router)#network 20.1.1.0
ISP(config-router)#network 10.1.1.0 mask 255.255.255.0
ISP(config-router)#network 20.1.1.0 mask 255.255.255.0
ISP(config-router)#neighbor 1.1.1.1 remote-as 64100
ISP(config-router)#neighbor 2.1.1.1 remote-as 64300
```

### BRANCH-A (Router2) — AS 64100

```cisco
BRANCH-A(config)#router bgp 64100
BRANCH-A(config-router)#network 192.168.1.0
BRANCH-A(config-router)#network 192.168.2.0
BRANCH-A(config-router)#network 192.168.3.0
BRANCH-A(config-router)#neighbor 1.1.1.2 remote-as 64200
```

### BRANCH-B (Router1) — AS 64300

```cisco
BRANCH-B(config)#router bgp 64300
BRANCH-B(config-router)#network 192.168.4.0
BRANCH-B(config-router)#network 192.168.5.0
BRANCH-B(config-router)#network 192.168.6.0
BRANCH-B(config-router)#neighbor 2.1.1.2 remote-as 64200
```

### BRANCH-A — Router-on-a-Stick

```cisco
BRANCH-A(config)#interface fastEthernet 0/0
BRANCH-A(config-if)#no ip address
BRANCH-A(config-if)#no shutdown

BRANCH-A(config)#interface fastEthernet 0/0.10
BRANCH-A(config-subif)#encapsulation dot1Q 10
BRANCH-A(config-subif)#ip address 192.168.1.1 255.255.255.0
BRANCH-A(config-subif)#ip helper-address 10.1.1.2

BRANCH-A(config)#interface fastEthernet 0/0.20
BRANCH-A(config-subif)#encapsulation dot1Q 20
BRANCH-A(config-subif)#ip address 192.168.2.1 255.255.255.0
BRANCH-A(config-subif)#ip helper-address 10.1.1.2

BRANCH-A(config)#interface fastEthernet 0/0.30
BRANCH-A(config-subif)#encapsulation dot1Q 30
BRANCH-A(config-subif)#ip address 192.168.3.1 255.255.255.0
BRANCH-A(config-subif)#ip helper-address 10.1.1.2

BRANCH-A(config)#interface serial 2/0
BRANCH-A(config-if)#clock rate 64000
BRANCH-A(config-if)#ip address 1.1.1.1 255.255.255.0
BRANCH-A(config-if)#no shutdown
```

### SW-1 — VLANs + Trunk + Port Security

```cisco
Switch(config)#hostname SW-1
SW-1(config)#vlan 10
SW-1(config-vlan)#name 10
SW-1(config)#vlan 20
SW-1(config-vlan)#name 20
SW-1(config)#vlan 30
SW-1(config-vlan)#name 30

SW-1(config)#interface fastEthernet 0/1
SW-1(config-if)#switchport mode access
SW-1(config-if)#switchport access vlan 10
SW-1(config)#interface fastEthernet 0/2
SW-1(config-if)#switchport mode access
SW-1(config-if)#switchport access vlan 20
SW-1(config)#interface fastEthernet 0/3
SW-1(config-if)#switchport mode access
SW-1(config-if)#switchport access vlan 30

SW-1(config)#interface fastEthernet 0/24
SW-1(config-if)#switchport mode trunk
SW-1(config-if)#switchport trunk allowed vlan 10,20,30

SW-1(config)#interface range fastEthernet 0/1-3
SW-1(config-if-range)#switchport port-security
SW-1(config-if-range)#switchport port-security mac-address sticky
SW-1(config-if-range)#switchport port-security maximum 1
SW-1(config-if-range)#switchport port-security violation shutdown
```

---

### 📊 BGP Routing Table — BRANCH-A

```
BRANCH-A#show ip bgp

Network          Next Hop     Metric  LocPrf  Weight  Path
*> 10.1.1.0/24   1.1.1.2       0        0       0    64200 i        ← DHCP server
*> 20.1.1.0/24   1.1.1.2       0        0       0    64200 i        ← HTTP server
*> 192.168.1.0   0.0.0.0       0        0     32768   i             ← own VLAN 10
*> 192.168.2.0   0.0.0.0       0        0     32768   i             ← own VLAN 20
*> 192.168.3.0   0.0.0.0       0        0     32768   i             ← own VLAN 30
*> 192.168.4.0   1.1.1.2       0        0       0    64200 64300 i  ← BRANCH-B VLAN 40
*> 192.168.5.0   1.1.1.2       0        0       0    64200 64300 i  ← BRANCH-B VLAN 50
*> 192.168.6.0   1.1.1.2       0        0       0    64200 64300 i  ← BRANCH-B VLAN 60
```

### ISP `show ip route` — B Routes

```
B  192.168.1.0/24 [20/0] via 1.1.1.1, 00:00:00  ← BRANCH-A
B  192.168.2.0/24 [20/0] via 1.1.1.1, 00:00:00
B  192.168.3.0/24 [20/0] via 1.1.1.1, 00:00:00
B  192.168.4.0/24 [20/0] via 2.1.1.1, 00:00:00  ← BRANCH-B
B  192.168.5.0/24 [20/0] via 2.1.1.1, 00:00:00
B  192.168.6.0/24 [20/0] via 2.1.1.1, 00:00:00
```

---

## ✅ End-to-End Verification

### Cross-AS Pings from PC2 (VLAN 30, BRANCH-A)

```
ping 192.168.1.9  → VLAN 10 same branch     0% loss ✅
ping 192.168.5.9  → VLAN 50 BRANCH-B AS64300  0% loss ✅
ping 20.1.1.2     → HTTP Server ISP AS64200   0% loss ✅
```

### HTTP from PC4 (VLAN 50, BRANCH-B)

```
http://20.1.1.2  →  "BGP CONFIG" page loaded ✅
```

### DHCP Pools (6 VLANs, DNS 8.8.8.8)

| Pool | Gateway | DNS | Start IP |
|---|---|---|---|
| VLAN 10 | 192.168.1.1 | 8.8.8.8 | 192.168.1.1 |
| VLAN 20 | 192.168.2.1 | 8.8.8.8 | 192.168.2.1 |
| VLAN 30 | 192.168.3.1 | 8.8.8.8 | 192.168.3.1 |
| VLAN 40 | 192.168.4.1 | 8.8.8.8 | 192.168.4.1 |
| VLAN 50 | 192.168.5.1 | 8.8.8.8 | 192.168.5.1 |
| VLAN 60 | 192.168.6.1 | 8.8.8.8 | 192.168.6.1 |

---

## 🔍 BGP Verification Command Reference

```cisco
show ip bgp                              ! Full BGP table — prefixes, AS-PATH, best path
show ip bgp summary                      ! Neighbor states, prefixes received, uptime
show ip bgp neighbors                    ! Full TCP+BGP session detail
show ip bgp neighbors x.x.x.x advertised-routes  ! What we send to neighbor
show ip bgp neighbors x.x.x.x received-routes    ! What we receive from neighbor
show ip route bgp                        ! Only BGP routes in routing table
show ip bgp x.x.x.x                     ! Detail on specific prefix
show ip protocols                        ! BGP process, AS number, neighbors
```

---

## 📁 Repository Structure

```
📦 Project-119-BGP-MultiAS/
├── 📄 README.md
├── 📁 screenshots/
│   ├── topology.png
│   ├── branch-a-bgp-summary.png
│   ├── branch-a-show-ip-bgp.png
│   ├── branch-a-show-ip-route.png
│   ├── branch-a-bgp-neighbors-detail.png
│   ├── branch-a-bgp-tcp-session.png
│   ├── isp-show-ip-route.png
│   ├── cross-as-ping.png
│   ├── http-bgp-config-page.png
│   └── dhcp-server-pools.png
└── 📄 Project119.pkt
```

---

## 👤 Author

**Maaz Khan**
CCNA Certified · Network Engineer · NOC Engineer
📍 Lower Dir, KPK, Pakistan
🔗 [LinkedIn](https://linkedin.com/in/maazkhanms) 

---

> *"BGP is not just a routing protocol — it IS the internet."*
