---
title: "Networking Basics: From the Default Gateway to Cloud NAT Gateways"
date: 2026-09-18 00:00:00 +0900
categories: [Network, Cloud]
tags: [gateway, subnet, availability-zone, nat-gateway, vpc, aws]
---

Starting from the familiar "default gateway" concept at home, this post builds up the concepts needed to understand cloud infrastructure — subnets, Availability Zones (AZs), and cloud NAT gateways — in order. Each concept builds on the one before it, so it's recommended to read in the order: home gateway → subnet → Availability Zone → cloud NAT gateway.

## 1. What Is the Default Gateway?

When you run `ipconfig` in a terminal, the "Default Gateway" address shown is, in most home setups, your router's IP address.

### Why is a gateway needed?
A gateway acts as **the "doorkeeper" connecting your local network to the outside internet**. Devices within the same network (e.g., your home) don't need to go through the gateway to talk to each other. But to reach an external server like Google or Naver, the packet has to be handed off to something — and that's the gateway's job.

### Why is the router the gateway?
```
[My PC]     --- [Router] --- [Internet (Modem/ISP)]
[My Phone]  ---   |
[Printer]   ---   |
```
The router has two "faces":
- **Inside face (LAN IP)**: e.g., `192.168.0.1` — the address devices inside the home see
- **Outside face (WAN IP)**: the public IP assigned by the ISP — the address the internet sees

No device in the home connects to the internet directly; they can only go through the router. So the "exit point to the internet" is the router, and its LAN IP naturally becomes the gateway address.

### How it's set automatically (DHCP)
1. When a device connects via Wi-Fi/Ethernet, it broadcasts "I need an IP address!"
2. The router, acting as a DHCP server, responds and provides:
   - An IP address for the device (e.g., `192.168.0.15`)
   - A subnet mask
   - **Default Gateway = the router's own IP**
   - DNS server address

In other words, the router hands out IPs while registering itself as the gateway.

## 2. What Is a Subnet?

Subnet = Sub + Network — a large network split into smaller network units.

### Why split it up?
Just like dividing an apartment complex into sections, splitting an IP address range by purpose or security level lets you:
- Apply different security rules per section
- Isolate just the affected section when something goes wrong
- Manage traffic more easily

### Seeing it with IP ranges
`192.168.0.0/24` (256 addresses) can be split like this:

| Subnet | IP Range | Purpose |
|---|---|---|
| Subnet A | 192.168.0.0 – 0.63 | Web servers (Public) |
| Subnet B | 192.168.0.64 – 0.127 | DB servers (Private) |
| Subnet C | 192.168.0.128 – 0.191 | Office PCs |

### What CIDR notation (`/24`) means
An IP address (32 bits) is split into a fixed network portion and a portion that can be assigned to hosts.

```
192.168.0.   0
└────────┘ └┘
Network part    Host part (assignable)
 (24 bits)       (8 bits = up to 256, 254 usable)
```
The larger the number (`/24` → `/28`, etc.), the longer the network portion — meaning each subnet becomes smaller.

### Subnets in the cloud (AWS)
```
[VPC: 10.0.0.0/16]  ← the entire virtual network
  ├─ Public Subnet:  10.0.1.0/24  → web servers, NAT gateway
  └─ Private Subnet: 10.0.2.0/24  → DB servers, internal API servers
```
- **VPC**: your own large virtual network inside the cloud (the whole apartment complex)
- **Subnet**: a section of it divided by purpose

Whether a subnet is Public or Private is determined by its **route table**:
- Public Subnet → has a route to an Internet Gateway → reachable from outside
- Private Subnet → has no route to an Internet Gateway; it can only reach the internet through a NAT gateway

## 3. What Is an Availability Zone (AZ)?

### The hierarchy
```
[Region] e.g., Seoul (ap-northeast-2)
   ├─ [AZ-a] — a physically separate cluster of data centers
   ├─ [AZ-b] — another physically separate cluster
   └─ [AZ-c] — another physically separate cluster
```
- **Region**: a large geographic unit, like Seoul or Tokyo
- **Availability Zone (AZ)**: a physically isolated group of data centers within a region

### Why it exists: fault isolation
If a region had only one data center, a single incident — fire, power outage, flooding — could take down the entire region. So AWS builds each AZ with its **own power supply, cooling, and network links**, physically separating them so that a failure in one AZ doesn't affect the others.

### How AZs connect to each other
AZs within the same region are linked by high-speed, low-latency dedicated fiber connections (typically under 1–2ms), so spreading workloads across multiple AZs barely affects performance.

### Example usage
```
[Load Balancer]
   ├─ Web Server 1 → AZ-a
   ├─ Web Server 2 → AZ-b
   └─ Web Server 3 → AZ-c
```
If AZ-a goes down, the load balancer routes traffic only to AZ-b and AZ-c, keeping the service running. For the same reason, it's recommended to deploy one NAT gateway per AZ rather than sharing a single one (see Section 4).

> Note: Multi-AZ distributes across data centers within the same region (city area), so it's still vulnerable to a disaster affecting the whole city, like an earthquake. For stronger disaster recovery, you'd replicate across entirely different regions — e.g., Seoul + Tokyo — known as Multi-Region, though this adds significant complexity and cost. For most services, Multi-AZ alone covers the vast majority of failures (power, hardware, network equipment).

## 4. Cloud NAT Gateway

### A quick recap of NAT
Network Address Translation (NAT) is a technology that translates between private and public IP addresses. In fact, the home router in Section 1 is already doing NAT internally — meaning **a home router is both a gateway and a NAT device combined into one**.

### Why the cloud needs a separate NAT gateway
For security, cloud servers are typically split like this:
```
[VPC]
 ├─ Public Subnet  (directly reachable from the internet — web servers, etc.)
 └─ Private Subnet (not directly reachable from the internet — DB, internal API servers, etc.)
```
A DB server in the Private Subnet must block incoming connections from outside, but it still needs to make **outgoing** connections — for example, downloading security patches or calling external APIs. Handling this "outbound only" requirement is exactly what a NAT gateway does.

### How it works
```
[Private Subnet server] --- [NAT Gateway (sits in a Public Subnet)] --- [Internet]
```
1. The private server sends a request out to the internet
2. The NAT gateway translates it to its own public IP and forwards it
3. When the response comes back, it's routed back to the server that made the request
4. **Inbound connections initiated from outside can never reach the private server directly** — only responses to connections the private server itself initiated are allowed. It's strictly one-way traffic.

## 5. Home Gateway vs. Cloud NAT Gateway

| Aspect | Home Router (Gateway) | Cloud NAT Gateway |
|---|---|---|
| Role | Gateway + NAT + router + firewall, all in one device | NAT only (routing is handled separately by route tables) |
| Inbound access from outside | Possible with port forwarding | Structurally impossible for a Private Subnet |
| Redundancy / failover | If the router dies, the entire home loses internet | Can be made redundant by deploying one per AZ |
| Scalability | Fixed capacity | Bandwidth scales automatically with traffic |
| Managed by | The user manages a single physical device | Provided as a managed service by AWS |
| Cost | Only the cost of the device | Hourly charge + per-GB data processing charge |

### Advantages of a NAT gateway
1. **Security**: Private Subnet servers can never be directly attacked from the internet (SSH, DB ports, etc. are never exposed)
2. **Ease of management**: As a managed service, AWS handles redundancy, patching, and scaling automatically
3. **Availability**: Deploying one per AZ lets traffic reroute if one AZ fails

### Disadvantages of a NAT gateway
1. **Cost**: Hourly charges plus per-GB traffic charges can add up quickly for high-traffic services — often a surprise in practice
2. **True high availability requires one per AZ**: A single NAT gateway in one AZ means that if that AZ fails, private servers in other AZs can also lose internet access — so deploying one per AZ (which increases cost) is the standard recommendation
3. **Outbound only**: Since it's strictly for outbound traffic, any service that needs inbound access must use a different architecture entirely (e.g., a Public Subnet with a load balancer)

## TL;DR

- **Default gateway**: the sole path from your local network out to the internet. At home, the router plays both this role and the NAT role, and announces itself automatically to devices via DHCP.
- **Subnet**: a large network split into smaller ones by purpose or security level. CIDR notation like `/24` marks the boundary between the network portion and the assignable host portion. A subnet always belongs to exactly one AZ.
- **Availability Zone (AZ)**: a group of data centers within a region that are physically separated in power, cooling, and networking. Its purpose is fault isolation, and distributing servers and NAT gateways across AZs is how high availability is achieved.
- **NAT gateway**: a managed, outbound-only exit point that lets Private Subnet servers make outgoing connections. The concept is the same as NAT on a home router, but the cloud version structurally enforces one-way traffic to preserve security isolation — that's the key difference.
