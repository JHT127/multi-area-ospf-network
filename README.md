# multi-area-ospf-network

> 5-area OSPF enterprise network designed from scratch in Cisco Packet Tracer — manual VLSM subnetting, dynamic routing, and a full datacenter service stack (DHCP, DNS, SMTP/POP3, HTTP/HTTPS).

---

## What Was Built

A simulated enterprise network spanning five areas interconnected through OSPF dynamic routing, with a complete service stack deployed in the datacenter area. The design covers IP planning, device configuration, protocol setup, wireless/cellular connectivity, and end-to-end verification across all segments.

---

## Architecture Overview

**5-area topology:**

```
                  ┌─────────────────────────────────────────┐
                  │            AREA 0 (Core)                │
                  │   R0 ─────── R1 ─────── R2             │
                  │  (NET0-A)  (NET0-B)  (NET0-C)          │
                  └────┬────────────┬────────────┬──────────┘
                       │            │             │
             ┌─────────┤      ┌─────┤       ┌────┤
             ▼         ▼      ▼     ▼       ▼    ▼
          AREA 1      AREA 2  AREA 3        AREA 4
        University    Street   Home       Datacenter
        (NET1-A/B)   (NET2)  (NET3)        (NET4)
        60+60 hosts  30 hosts 20 hosts    15 hosts
        DHCP Server  Cell    Wi-Fi AP    Web Server
                     Tower              Mail Server
                                        DNS Server
```

- **Area 0** is the OSPF backbone; all inter-area traffic routes through it
- **Areas 1–4** are stub areas connected to Area 0 via ABRs (R0, R1, R2)
- Serial links used for core router interconnects (NET0-A/B/C as /30 point-to-point links)
- FastEthernet links for LAN segments within each area

---

## IP Addressing Scheme

**Base network: `102.28.8.0/23`** (510 usable addresses total)

Subnets were allocated using VLSM — sorted by host requirement descending to minimise address waste:

| Subnet | CIDR | Network Address | Usable Range | Usable Hosts |
|---|---|---|---|---|
| NET1-A (University) | /26 | 102.28.8.0 | .1 – .62 | 62 |
| NET1-B (University) | /26 | 102.28.8.64 | .65 – .126 | 62 |
| NET2 (Street/Router) | /28 | 102.28.8.128 | .129 – .143 | 14 |
| NET2 (Cell Tower) | /28 | 102.28.8.144 | .145 – .158 | 14 |
| NET3 (Home) | /27 | 102.28.8.160 | .161 – .190 | 30 |
| NET4 (Datacenter) | /27 | 102.28.8.192 | .193 – .222 | 30 |
| NET0-A (Core link) | /30 | 102.28.8.224 | .225 – .226 | 2 |
| NET0-B (Core link) | /30 | 102.28.8.228 | .229 – .230 | 2 |
| NET0-C (Core link) | /30 | 102.28.8.232 | .233 – .234 | 2 |

**Subnetting method:** CIDR block selected as smallest power-of-two that fits `hosts_needed + 2`. Binary bit-boundary splitting used to avoid overlap. See `docs/network-topology-report.pdf` for full binary allocation walkthrough.

---

## Component Breakdown

### OSPF Routing (Areas 0–4)

- Protocol: OSPF (Open Shortest Path First), using Dijkstra's SPF algorithm
- All routers configured with `router ospf 1`
- Each network advertised with wildcard mask and area ID:
  ```
  router ospf 1
    network <subnet-ip> <wildcard-mask> area <area-id>
  ```
- Area 0 backbone connects R0, R1, R2 via serial point-to-point links
- ABRs propagate routes between areas; end devices have no static routes

### Router Interface Summary

**Router 0** (connects Area 0 ↔ Area 1):

| Interface | IP Address | Subnet Mask | Connected To |
|---|---|---|---|
| FastEthernet0/0 | 102.28.8.1 | 255.255.255.192 | NET1-A |
| FastEthernet1/0 | 102.28.8.65 | 255.255.255.192 | NET1-B |
| Serial2/0 | 102.28.8.230 | 255.255.255.252 | NET0-B |
| Serial3/0 | 102.28.8.225 | 255.255.255.252 | NET0-A |

**Router 1** (connects Area 0 ↔ Areas 2 & 3):

| Interface | IP Address | Subnet Mask | Connected To |
|---|---|---|---|
| FastEthernet0/0 | 102.28.8.161 | 255.255.255.224 | NET3 (Home) |
| FastEthernet1/0 | 102.28.8.129 | 255.255.255.240 | NET2 (Street) |
| Serial2/0 | 102.28.8.233 | 255.255.255.252 | NET0-C |
| Serial3/0 | 102.28.8.226 | 255.255.255.252 | NET0-A |

### Datacenter Services (Area 4)

| Service | Hostname | Protocol | Static IP |
|---|---|---|---|
| Web Server | www.coe.birzeit.edu | HTTP / HTTPS | 102.28.8.193 |
| Mail Server | mail.coe.birzeit.edu | SMTP + POP3 | 102.28.8.196 |
| DNS Server | dns.coe.birzeit.edu | DNS (A + CNAME records) | 102.28.8.197 |
| DHCP Server | — | DHCP | 102.28.8.198 |

**DNS resource records configured:**
- A record: `www.coe.birzeit.edu` → `102.28.8.193`
- A record: `mail.coe.birzeit.edu` → `102.28.8.196`
- CNAME record: alias pointing to canonical domain

**DHCP scope:** Dynamic address assignment for University (Area 1) and Home (Area 3) end devices. Datacenter servers use static IPs.

### Wireless & Cellular (Areas 2 & 3)

- **Wi-Fi (Area 3 — Home):** AccessPoint-PT with WPA2/WPA3, SSID configured; laptops and mobile devices connect wirelessly
- **Cellular (Area 2 — Street):** Cell Tower connected to a Central Office (CO) server, simulating MSC/BSC mobile switching; CO server required its own /28 subnet, separate from the street router subnet (see Limitations)

---

## Test & Verification

| Test | Method | Result |
|---|---|---|
| Inter-area connectivity | `ping` between all area pairs | ✅ |
| Multi-hop routing path | `tracert` from University PC to Datacenter | ✅ confirmed via Area 0 backbone |
| DNS resolution | `nslookup www.coe.birzeit.edu` from all segments | ✅ → 102.28.8.193 |
| Web access by IP | Browser from University, Home, Street | ✅ |
| Web access by domain | Browser using DNS name | ✅ |
| DHCP assignment | End devices in Area 1 & 3 booted | ✅ dynamic IPs assigned |
| Email Home → University | SMTP send + POP3 receive | ✅ |
| Email University → Home (reply) | SMTP send + POP3 receive | ✅ |
| All DNS resource records | `nslookup` for each record | ✅ 3/3 records resolved |

---

## Known Limitations

**Street network subnet revision mid-design**
- Root cause: Initial plan allocated a single /27 for NET2 (30 hosts). During device configuration, the CO server required separate network segmentation from the router to interface correctly with the Cell Tower infrastructure
- Resolution: /27 split into two /28 subnets — one for the router-side segment, one for the Cell Tower/CO side. This introduced a brief routing inconsistency between NET2 and NET1/NET4 during the transition period
- Lesson: service dependencies between devices (CO server ↔ Cell Tower) should be identified and mapped before subnetting, not discovered during configuration

**Packet Tracer simulation constraints**
- Cellular network is simplified; real LTE/5G control plane components (MME, SGW, PGW) are not modelled
- Wireshark capture missed some frames during initial testing due to local interface caching

**No redundancy or failover**
- Single path between areas; if a core router or serial link fails, the connected area loses connectivity
- Real enterprise designs would use OSPF with multiple ABRs and redundant links

---

## How to Open

```
1. Open Cisco Packet Tracer (v8.x recommended)
2. File → Open → select network-topology/enterprise-network.pkt
3. Switch to Simulation mode to step through packet flows
4. Use any end-device CLI to verify:
   ping 102.28.8.193             # reach web server
   nslookup www.coe.birzeit.edu  # DNS resolution
   tracert 102.28.8.193          # confirm OSPF path through Area 0
```

---

## Skills Demonstrated

| What Was Built | Technical Domain |
|---|---|
| VLSM subnetting from first principles | IP addressing, network planning |
| Multi-area OSPF configuration | Dynamic routing, inter-area route propagation |
| DHCP, DNS, SMTP/POP3, HTTP/HTTPS service stack | Enterprise network services integration |
| Wireless (Wi-Fi + cellular) connectivity | Layer 2/3 access network design |
| Root-cause documented subnet redesign | Iterative infrastructure debugging |
| End-to-end cross-segment verification | Network observability, protocol testing |

---

## Repository Structure

```
multi-area-ospf-network/
├── network-topology/
│   └── enterprise-network.pkt
├── docs/
│   └── network-topology-report.pdf
└── README.md
```

---

*Computer Networks (ENCS3320) — Electrical and Computer Engineering, Birzeit University, 2024–2025*