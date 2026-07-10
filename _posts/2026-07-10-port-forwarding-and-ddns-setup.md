---
title: "Remote SSH Access via Port Forwarding and DDNS"
date: 2026-07-10 00:00:00 +0900
categories: [Network, Server]
tags: [ssh, port-forwarding, ddns, duckdns, vmware]
---

# Remote SSH Access via Port Forwarding and DDNS

A guide to enabling SSH access from anywhere — including mobile data networks — using router port forwarding and DDNS.

## Background

When your server runs inside a VM on a home network, its IP address (e.g. `192.168.13.250`) is a private address only reachable from within the same Wi-Fi network. To access it from outside (mobile data, a different network, etc.), you need to set up port forwarding through your router and optionally DDNS to handle dynamic public IPs.

## Network Architecture

```
External (mobile data / different network)
       ↓
Public IP (e.g. 182.215.225.241) : Port 2222
       ↓
Router Port Forwarding
       ↓
Laptop (192.168.219.101) : Port 2222
       ↓
VMware NAT Port Forwarding
       ↓
Ubuntu VM Server (192.168.13.250) : Port 22
```

## Step 1 — VMware NAT Port Forwarding (already configured)

This was set up earlier to allow SSH into the VM from the host machine:

- Open VMware → Edit → Virtual Network Editor
- Select **VMnet8** (NAT type) → NAT Settings
- Add a port forwarding rule:
  - **Host Port**: `2222`
  - **Type**: `TCP`
  - **Virtual Machine IP**: `192.168.13.250`
  - **Virtual Machine Port**: `22`

This lets traffic arriving at the laptop on port 2222 reach the VM's SSH port (22).

## Step 2 — Windows Firewall Rule

Allow inbound connections on port 2222 through the Windows firewall. Run in PowerShell as Administrator:

```powershell
New-NetFirewallRule -DisplayName "Public SSH 2222" -Direction Inbound -Protocol TCP -LocalPort 2222 -Action Allow
```

## Step 3 — Router Port Forwarding

Access the router admin page (typically at the router's gateway IP, e.g. `192.168.219.1`) and log in.

Navigate to **NAT Settings → Port Forwarding** and add a new rule:

| Field | Value |
|---|---|
| Service Port | `2222` ~ `2222` |
| Protocol | TCP |
| Internal IP Address | `192.168.219.101` (laptop's local IP) |
| Internal Port | `2222` |

Save and apply the settings. This forwards external traffic on port 2222 to the laptop, which then passes it to the VM via VMware's NAT forwarding.

## Step 4 — DDNS Setup with DuckDNS

Home internet connections typically use a **dynamic public IP** — meaning the IP can change after a router reboot or over time. DDNS (Dynamic DNS) solves this by mapping a fixed domain name to your current public IP, updating automatically whenever it changes.

### 4-1. Create a DuckDNS domain

1. Go to [duckdns.org](https://www.duckdns.org) and sign in with a Google account.
2. Enter a subdomain name (e.g. `hhc-server`) and click **add domain**.
3. Your domain will be: `hhc-server.duckdns.org`
4. Note the **token** value shown at the top of the page — you'll need it for router configuration.

### 4-2. Configure DDNS on the router

In the router admin page, find the **DDNS Settings** section and fill in:

| Field | Value |
|---|---|
| DDNS Server | Duck DNS |
| Username | Your DuckDNS token |
| Password | Your DuckDNS token |
| Host Domain | `hhc-server` (subdomain only, without `.duckdns.org`) |

Save the settings. The router will now automatically update DuckDNS with your current public IP whenever it changes.

### 4-3. Verify

On the DuckDNS dashboard, confirm that your domain shows the correct public IP:

```
hhc-server    182.215.225.241    updated X minutes ago
```

## Connecting via SSH

Once everything is set up, connect from anywhere using the domain name instead of the raw IP:

```bash
ssh hwang@hhc-server.duckdns.org -p 2222
```

Or with a key file specified explicitly:

```bash
ssh -i ~/.ssh/id_ed25519 hwang@hhc-server.duckdns.org -p 2222
```

For mobile SSH clients (e.g. Terminus on iOS), update the host settings:

- **Host**: `hhc-server.duckdns.org`
- **Port**: `2222`
- **Username**: `hwang`
- **Key**: `id_ed25519`

## Notes

- The public IP shown in these examples (`182.215.225.241`) is dynamic and may change. DDNS handles this automatically.
- If the connection suddenly stops working, check whether the public IP has changed and whether the DuckDNS record has been updated.
- Some ISPs block certain ports by default. If a port isn't working, try an alternative like `443` or `8022`.
- SSH key authentication is strongly recommended over password authentication for any publicly exposed SSH port.
