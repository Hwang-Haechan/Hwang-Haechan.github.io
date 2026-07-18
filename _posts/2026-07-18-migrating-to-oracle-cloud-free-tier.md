---
title: "Migrating a Local VM Server to Oracle Cloud Free Tier"
date: 2026-07-18 00:00:00 +0900
categories: [Infrastructure, Cloud]
tags: [oracle-cloud, vmware, ssh, migration, nodejs, mysql]
---

## Why Migrate?

Until now, the server had been running inside a VMware VM on a local laptop. That setup worked fine for development, but it had one fundamental limitation: **the server was only reachable while the laptop itself was powered on and connected to the network.** If the laptop slept, lost Wi-Fi, or was simply shut down, the server went down with it — and so did any hope of accessing it remotely from a phone on mobile data.

To get a server that's actually reachable 24/7 regardless of what happens to the local machine, the natural next step was moving to a cloud provider. Oracle Cloud's Free Tier was the choice here, mainly because it offers a genuinely free (not just trial-period) ARM-based compute instance with a decent amount of CPU and memory.

## Setting Up the VM Instance

Creating the instance itself was mostly straightforward through Oracle Cloud's console — select an OS image (Ubuntu), pick a shape, attach an SSH key, and launch. A couple of snags came up along the way:

- **ARM shape unavailable**: The free ARM-based shape (`VM.Standard.A1.Flex`) returned an "out of capacity" error in the selected availability domain. This is a known limitation — free ARM instances are in high demand and availability fluctuates. Switching to the free AMD-based shape (`VM.Standard.E2.1.Micro`) resolved it immediately, at the cost of lower specs (1 OCPU, 1GB RAM).
- **Public IP required a public subnet**: assigning a public IPv4 address to the instance required the subnet itself to be explicitly created as a "public subnet" — the default flow didn't surface this clearly until a VCN and subnet were created manually first.

## The Missing Internet Gateway

After the instance launched with a public IP attached, SSH connections still timed out completely — no rejection, just silence. Working through it systematically:

1. Confirmed the instance was in a running state.
2. Confirmed the Security List had an ingress rule allowing TCP port 22 from anywhere.
3. Still nothing. 

The actual root cause turned out to be one level up: the VCN had **no Internet Gateway attached at all**. Without one, the VCN has no path in or out to the public internet — a public IP assigned to an instance is meaningless if there's no gateway routing traffic to it in the first place.

The fix:
1. Create an Internet Gateway inside the VCN.
2. Add a route rule in the route table: destination `0.0.0.0/0` → target the Internet Gateway.

Once that route was in place, SSH connected immediately. This is easy to overlook if you're used to consumer routers, where "internet access" is assumed by default — in a cloud VPC, every hop has to be explicit.

## SSH Key Confusion

A separate wrinkle: after generating a new SSH key pair and copying it to the server, `git` operations against GitHub kept failing with `Permission denied (publickey)` — even though the exact same key worked fine for logging into the server itself.

The cause was simple in hindsight: the key registered on GitHub was an **older key from a previous setup**, not the one currently being used. Comparing key fingerprints (`ssh-keygen -lf`) on both the client and against what GitHub had listed made the mismatch obvious. Registering the current key's public half with GitHub resolved it — old keys don't need to be removed, multiple keys can coexist on an account.

## Rebuilding the Environment

With SSH and Git access sorted, the rest was reconstructing the same stack that had been running on the local VM:

- Node.js (upgraded via NodeSource, since the distro-provided version was too old for the frontend build tooling)
- MySQL, with a fresh database/table/user set up from scratch
- Environment variables via a `.env` file
- The frontend (React + Vite) rebuilt from source

### Memory Constraints on a Free-Tier Instance

The free AMD shape's 1GB of RAM turned out to be a real constraint — `npm install` for the frontend dependencies simply hung indefinitely. Checking memory usage confirmed the instance was nearly out of free RAM with zero swap configured.

Adding a 2GB swap file resolved it:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

After that, package installs and the production build completed without issue — swap gave the system enough breathing room to get through memory-heavy operations, even on constrained hardware.

### Node.js Package Manager Conflicts

Upgrading Node.js via `apt` initially failed with a file conflict against an old `libnode-dev` package left over from the distro's bundled Node.js version. Removing that package first cleared the way for the new Node.js install to proceed cleanly.

## Firewalling the Application Port

Getting the app itself reachable from a browser required opening its port in two separate places — a detail that's easy to miss coming from a simpler home-router setup:

1. **Security List** (cloud-level firewall): add an ingress rule for the app's port.
2. **iptables** (instance-level firewall): Oracle's default Ubuntu image ships with an iptables ruleset that rejects everything not explicitly allowed. The application's port had to be explicitly inserted into the `INPUT` chain *before* the catch-all reject rule, using `iptables -I INPUT <position> ...` rather than appending it to the end.

Once both were in place, the app became reachable from the public internet.

## Result

At the end of this process, the entire stack — API server, database, and built frontend — was running independently of any local machine, reachable via a public IP from anywhere, including mobile networks. The next logical step is pointing a domain name at it and automating deployment so that a `git push` updates the live server without manual SSH steps each time.

## Key Takeaways

- A public IP alone doesn't guarantee internet reachability — the VCN needs an Internet Gateway and a route pointing to it.
- Cloud firewalls are usually layered: a network-level security list *and* an instance-level firewall (iptables/ufw) both need to allow the same port.
- Free-tier compute shapes can be memory-constrained; a swap file is a quick, effective mitigation for memory-heavy build processes.
- When SSH auth fails against a *service* (like GitHub) but works fine for the server itself, comparing key fingerprints is the fastest way to catch a stale/wrong key registration.
