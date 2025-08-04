---

title: 'SMB Multichannel Implementation Experience on Linux and Windows'
date: 2025-08-04
permalink: /_notes/2025/08/linux_windows_smb_multichannel/
tags:

* Linux
* Windows
* SMB
* Multichannel
* Network Optimization

---

**Read this in other languages: [English](https://cc5103.github.io/ZHOUYUNHAO.github.io///_notes/2025/08/linux_windows_smb_multichannel/), [中文](https://cc5103.github.io/ZHOUYUNHAO.github.io///_notes_zh/2025/08/linux_windows_smb_multichannel/), [日本語](https://cc5103.github.io/ZHOUYUNHAO.github.io///_notes_jp/2025/08/linux_windows_smb_multichannel/).**

# Introduction

My newly purchased server came preinstalled with three Gigabit Ethernet ports — for reasons unknown. At first, I assumed they were intended for gateway purposes or enterprise functions like network isolation and failover. However, I soon realized they were of no help to my actual goal: **improving local network file transfer speeds**. So I started exploring how to make use of these otherwise idle ports.

After some research, I found two mainstream solutions:

1. **Link Aggregation (LACP)**
2. **SMB Multichannel**

Both aim to improve bandwidth utilization by leveraging multiple network interfaces, but they work in different scenarios:

* **LACP is ideal for multiple users accessing the server simultaneously**, ensuring users don't slow each other down. However, it **doesn’t allow a single user to exceed the speed of a single NIC**.
* **SMB Multichannel benefits a single user**, enabling **parallel transfers across multiple physical links**, effectively boosting single-connection speed — exactly what I needed.

This article documents how I set up SMB Multichannel between Linux (Debian) and Windows, with real-world implementation details and pitfalls. Most online resources focus on NAS or all-Windows setups, but this is about building your own DIY Linux server from scratch — more relevant for personal self-hosting scenarios.

---

# Table of Contents

- [Introduction](#introduction)
- [Table of Contents](#table-of-contents)
  - [Difference Between LACP and SMB Multichannel](#difference-between-lacp-and-smb-multichannel)
    - [Link Aggregation (LACP)](#link-aggregation-lacp)
    - [SMB Multichannel](#smb-multichannel)
    - [Comparison Summary](#comparison-summary)
  - [Implementing SMB](#implementing-smb)
    - [System Environment](#system-environment)
    - [Sample Samba Configuration](#sample-samba-configuration)
    - [Applying the Configuration](#applying-the-configuration)
  - [Realizing SMB Multichannel](#realizing-smb-multichannel)
    - [Connection Topology Examples](#connection-topology-examples)
    - [Windows Configuration and Issues](#windows-configuration-and-issues)
      - [Checking Multichannel Status](#checking-multichannel-status)
      - [USB NICs and Multichannel Issues](#usb-nics-and-multichannel-issues)
  - [Linux Configuration and Issues](#linux-configuration-and-issues)
    - [Linux Defaults to Weak Host Model](#linux-defaults-to-weak-host-model)
    - [Solution: Strong Host Model + Policy Routing](#solution-strong-host-model--policy-routing)
    - [Routing Configuration Explained](#routing-configuration-explained)
      - [Checking Interface Names and IPs](#checking-interface-names-and-ips)
      - [Example Policy Routing Script (for 3 NICs)](#example-policy-routing-script-for-3-nics)
      - [Enabling the Script](#enabling-the-script)
  - [Conclusion](#conclusion)

---

## Difference Between LACP and SMB Multichannel

### Link Aggregation (LACP)

LACP (IEEE 802.3ad) bonds multiple physical NICs into a single logical interface to achieve redundancy and bandwidth aggregation:

* Ideal for high-concurrency, multi-user server environments.
* **Single-user speed still limited to one physical link (e.g., 1Gbps).**
* Requires LACP-capable switches and more complex configuration.

### SMB Multichannel

Introduced in SMB 3.0, this feature allows a single SMB session to use multiple network paths simultaneously:

* **No switch or special hardware required**
* **Single-user bandwidth can be aggregated**
* Supports various types of NICs — even USB ones
* Limitation: **only works with the SMB protocol**

Perfect for cases like mine — improving **local single-user LAN copy speeds**.

### Comparison Summary

| Feature           | LACP              | SMB Multichannel          |
| ----------------- | ----------------- | ------------------------- |
| Protocol Support  | All network types | Only SMB                  |
| Switch Required   | Yes               | No                        |
| Configuration     | Complex           | Simple (just Samba setup) |
| Single-User Boost | No                | Yes (aggregated links)    |
| Multi-User Eff.   | High              | Moderate                  |
| Fault Tolerance   | Strong            | Depends on manual routing |

---

## Implementing SMB

### System Environment

* **Server OS**: Linux (Debian 12)
* **Client OS**: Windows 10 / 11
* **Samba Version**: Recommended 4.15+ (must support `server multi channel support`)

### Sample Samba Configuration

```ini
[global]
   server min protocol = SMB3
   server max protocol = SMB3

   aio read size = 1
   aio write size = 1
   vfs objects = aio_pthread
   server multi channel support = yes

[SharedFolder]
   path = /mnt/your_shared_directory         ; Replace with your actual path
   browseable = yes
   read only = no
   force user = your_linux_username          ; Replace with your local Linux username
   create mask = 0777
   directory mask = 0777
   guest ok = no
   valid users = your_linux_username         ; Same as above
   use sendfile = yes
   printable = no
```

> **Notes:**
>
> * Ensure the shared directory exists and is accessible (use `chmod -R 777` or `chown`).
> * Username must be a valid local Linux user (check with `whoami`).
> * Open TCP port 445 on your firewall if needed.

### Applying the Configuration

```bash
sudo systemctl restart smbd
```

To check if the share is accessible:

```bash
smbclient -L localhost -U your_linux_username
```

---

## Realizing SMB Multichannel

### Connection Topology Examples

There are three common topologies for SMB Multichannel:

| Topology | Description                                                       |
| -------- | ----------------------------------------------------------------- |
| **EX.1** | All NICs connect to the same switch/subnet                        |
| **EX.2** | Each NIC uses a different subnet; client connects directly to one |
| **EX.3** | NICs connect to different switches but same subnet                |

![alt text](../images/Network_topology_diagram.png)

### Windows Configuration and Issues

#### Checking Multichannel Status

```powershell
Get-SmbMultichannelConnection
```

If you see multiple `Client IP` and `Server IP` entries, multichannel is active.

#### USB NICs and Multichannel Issues

In my experience, multichannel **does not work** with USB NICs, even if properly connected and recognized. After investigation:

* Windows **prefers NICs with RSS (Receive Side Scaling) support**
* Most USB NICs **do not support RSS**
* As a result, only one NIC is used and others are ignored

**Fix:**

1. In Device Manager, **disable RSS for NICs that support it**:

   * Right-click NIC → Properties → Advanced → `Receive Side Scaling` → Set to `Disabled`
2. Reboot and recheck the connection

Also ensure:

* Windows version >= Windows 8
* SMB multichannel enabled in registry (`EnableMultiChannel=1`)
* TCP 445 not blocked
* Avoid mixing NICs with vastly different speeds

---

## Linux Configuration and Issues

In the EX.1 topology, I encountered a frustrating issue:

**Upload/download speeds were highly asymmetric.**

* Windows → Linux: Full speed, \~340MB/s (\~3Gbps)
* Linux → Windows: Sluggish, only \~120MB/s (\~1Gbps)

This issue does **not** occur in EX.2, where each link uses a separate subnet — but I wasn’t satisfied with that. EX.2 **doesn’t allow automatic failover** and **can disrupt internet access stability**, so I dug deeper.

Eventually I confirmed:

### Linux Defaults to Weak Host Model

This means that even if data is received on different interfaces, **the response is sent via the default route**, creating a bottleneck that limits upload speeds.

### Solution: Strong Host Model + Policy Routing

To ensure responses go out the same interface they came in, we need:

**Per-interface routing policies**.

---

### Routing Configuration Explained

#### Checking Interface Names and IPs

```bash
ip link show       # List all NICs
ip addr show       # Show each NIC's IP address
```

Record:

* Interface names (`eno1`, `enp1s0f0`, `enp1s0f1`)
* IP addresses (`192.168.0.10`, etc.)
* Subnet (`192.168.0.0/24`, etc.)

#### Example Policy Routing Script (for 3 NICs)

Edit: `sudo nano /etc/NetworkManager/dispatcher.d/99-policy-routing`

```bash
#!/bin/bash

IFACE=$1
STATUS=$2

# This example is for 3 NICs — replace names with your actual NICs
TABLE_ENO1=100
TABLE_ENP1S0F0=101
TABLE_ENP1S0F1=102

if [ "$STATUS" = "up" ]; then
    # Replace 192.168.0.0/24 with your actual subnet
    ip route show table $TABLE_ENO1 | grep -q "192.168.0.0/24" || ip route add 192.168.0.0/24 dev eno1 table $TABLE_ENO1
    ip route show table $TABLE_ENP1S0F0 | grep -q "192.168.0.0/24" || ip route add 192.168.0.0/24 dev enp1s0f0 table $TABLE_ENP1S0F0
    ip route show table $TABLE_ENP1S0F1 | grep -q "192.168.0.0/24" || ip route add 192.168.0.0/24 dev enp1s0f1 table $TABLE_ENP1S0F1

    # Replace these IPs with actual IPs of each interface
    ip rule show | grep -q "from 192.168.0.10 lookup $TABLE_ENO1" || ip rule add from 192.168.0.10 lookup $TABLE_ENO1
    ip rule show | grep -q "from 192.168.0.11 lookup $TABLE_ENP1S0F0" || ip rule add from 192.168.0.11 lookup $TABLE_ENP1S0F0
    ip rule show | grep -q "from 192.168.0.12 lookup $TABLE_ENP1S0F1" || ip rule add from 192.168.0.12 lookup $TABLE_ENP1S0F1
fi
```

> ✳️ Replacement Summary:
>
> * Replace `eno1`, `enp1s0f0`, `enp1s0f1` with actual NIC names (`ip link show`)
> * Replace `192.168.0.0/24` with your subnet (`ip route show`)
> * Replace IPs `192.168.0.10/11/12` with your NICs’ actual IPs (`ip addr show`)
> * If you only have 1 or 2 NICs, just keep the relevant parts

#### Enabling the Script

```bash
sudo chmod +x /etc/NetworkManager/dispatcher.d/99-policy-routing
```

Restart the network:

```bash
sudo systemctl restart NetworkManager
sudo ip route flush cache
```

Verify:

```bash
ip rule show
ip route show table 100
ip route show table 101
```

---

## Conclusion

SMB Multichannel is an efficient and low-cost way to **improve single-user network performance** using existing multi-NIC devices. Compared to LACP, it doesn’t require special switches and is easy to configure — perfect for home labs or small server setups.

In my case, I ran into:

* **USB NICs failing due to lack of RSS support**
* **Linux weak host model causing upload bottlenecks**

I resolved these by disabling RSS on Windows and enabling strong host model + policy routing on Linux. The result: **SMB Multichannel now maxes out bandwidth in both directions**.

If you have a multi-NIC server and want faster LAN transfer speeds, I hope this guide proves helpful.
