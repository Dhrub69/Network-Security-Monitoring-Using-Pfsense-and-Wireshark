# Network-Security-Monitoring-Using-Pfsense-and-Wireshark
A VirtualBox home lab demonstrating pfSense firewall configuration, network segmentation, DoS traffic simulation, Wireshark analysis, and firewall-based mitigation..

![Topology](screenshots/topology.png)

## 📋 Table of Contents
- [Overview](#overview)
- [Lab Topology](#lab-topology)
- [Tech Stack](#tech-stack)
- [Setup](#setup)
  - [1. Virtual Machines](#1-virtual-machines)
  - [2. pfSense Configuration](#2-pfsense-configuration)
  - [3. Kali Linux (Attacker)](#3-kali-linux-attacker)
  - [4. Ubuntu (Victim)](#4-ubuntu-victim)
- [Attack Simulation](#attack-simulation)
- [Mitigation](#mitigation)
- [Verification](#verification)
- [Key Takeaways](#key-takeaways)
- [Disclaimer](#disclaimer)

## Overview

This project simulates a small two-zone network using VirtualBox: an external ("WAN") segment where an attacker sits, and an internal, firewalled LAN where a victim host lives. A **pfSense** firewall sits between them. The lab demonstrates:

- Building network segmentation using bridged and internal VirtualBox adapters
- Configuring DHCP and firewall rules on pfSense
- Launching an ICMP/SYN flood with `hping3`
- Observing the attack in Wireshark
- Writing and applying a pfSense block rule to stop the attack
- Confirming mitigation through packet capture and firewall logs

## Lab Topology

| Device            | Network            | Interface | IP Address         | Role                     |
|-------------------|---------------------|-----------|---------------------|--------------------------|
| pfSense (WAN)     | Bridged             | vtnet0    | `172.20.10.2`        | Edge firewall            |
| pfSense (LAN)     | Internal (`LabNet`) | vtnet1    | `192.168.1.1/24`     | Gateway / DHCP server    |
| Kali Linux        | Bridged             | eth0      | `172.20.10.3/28`     | Simulated attacker       |
| Ubuntu Desktop    | Internal (`LabNet`) | enp0s3    | `192.168.1.100/24`   | Victim host              |

```
        [ Home Network / Internet ]
                    |
              (Bridged Adapter)
                    |
             ┌──────────────┐
             │   pfSense    │
             │ WAN: .10.2   │
             │ LAN: .1.1    │
             └──────┬───────┘
                    │ (Internal Network: LabNet)
                    │
             ┌──────────────┐
             │   Ubuntu     │
             │ 192.168.1.100│
             └──────────────┘

   Kali (172.20.10.3) reaches Ubuntu
   via a static route through pfSense's WAN IP.
```

## Tech Stack

- **VirtualBox** — virtualization platform
- **pfSense CE** — firewall/router OS
- **Kali Linux** — attacker host, `hping3`
- **Ubuntu Desktop** — victim host, `Wireshark`

## Setup

### 1. Virtual Machines

| VM       | OS Type         | CPU / RAM   | Disk  | Adapter(s)                                |
|----------|-----------------|-------------|-------|--------------------------------------------|
| pfSense  | FreeBSD 64-bit  | 2 vCPU / 2GB| 20 GB | Adapter 1: Bridged · Adapter 2: Internal (`LabNet`) |
| Kali     | Debian 64-bit   | 2 vCPU / 2GB| 15 GB | Adapter 1: Bridged                          |
| Ubuntu   | Ubuntu 64-bit   | 2 vCPU / 2GB| 15 GB | Adapter 1: Internal (`LabNet`)              |

> Create `LabNet` once — type the name into any VM's **Network ▸ Internal Network** dropdown and VirtualBox auto-creates it.

![VM Settings](screenshots/vm-settings.png)

### 2. pfSense Configuration

1. Install pfSense with default settings.
2. At the console menu, assign interfaces: `vtnet0` → WAN, `vtnet1` → LAN.
3. Leave WAN on DHCP — it will pull an address automatically (e.g. `172.20.10.2`).

**Enable temporary WAN GUI access** (console):
```
pfSense-d
```
Then log into `https://<WAN_IP>` with the default credentials (`admin` / `pfsense`) and immediately lock it down with a proper rule.

**Firewall ▸ Rules ▸ WAN ▸ Add**

| Field       | Value                          |
|-------------|---------------------------------|
| Action      | Pass                            |
| Protocol    | TCP                              |
| Source      | Kali IP (`172.20.10.3`)         |
| Destination | WAN address                     |
| Port        | 443                              |
| Description | Allow GUI access from Kali      |

![pfSense WAN Rule](screenshots/pfsense-wan-gui-rule.png)

**Configure LAN DHCP** — *Services ▸ DHCP Server ▸ LAN*

- Enable DHCP on LAN
- Range: `192.168.1.100 – 192.168.1.199`
- DNS: home router or public resolver

**Allow Kali → Ubuntu** — *Firewall ▸ Rules ▸ WAN ▸ Add*

| Field       | Value                        |
|-------------|-------------------------------|
| Action      | Pass                          |
| Protocol    | Any                            |
| Source      | `172.20.10.3`                  |
| Destination | `192.168.1.100`                |
| Description | Allow Kali access to Ubuntu   |

### 3. Kali Linux (Attacker)

Add a static route so Kali knows how to reach the internal LAN through pfSense:

```bash
sudo ip route add 192.168.1.0/24 via 172.20.10.2
ip route   # verify
```

### 4. Ubuntu (Victim)

Ubuntu receives `192.168.1.100/24` via pfSense DHCP automatically. Confirm:

```bash
ip a
ping -c3 google.com
```

## Attack Simulation

Install and start Wireshark on Ubuntu, capturing on `enp0s3`:

```bash
sudo apt update && sudo apt install -y wireshark
sudo wireshark
```

From Kali, launch a flood:

```bash
# ICMP flood
sudo hping3 -1 --flood 192.168.1.100

# or SYN flood
sudo hping3 --flood -S -p 80 192.168.1.100
```

> **Tip:** a flood generates thousands of packets/sec, which can be hard to read raw. Use a display filter (`ip.addr == 172.20.10.3` or `tcp.flags.syn == 1`) or **Statistics ▸ I/O Graph** for a clean visual.

![Wireshark Flood Capture](screenshots/wireshark-flood.png)

## Mitigation

Add a **Block** rule above the existing allow rule (order matters — pfSense evaluates top-down):

**Firewall ▸ Rules ▸ WAN ▸ Add (move to top)**

| Field       | Value                    |
|-------------|----------------------------|
| Action      | Block                      |
| Source      | `172.20.10.3`               |
| Destination | `192.168.1.100`             |
| Logging     | ✅ Enabled                  |
| Description | Block Kali DoS             |

![pfSense Block Rule](screenshots/pfsense-block-rule.png)

## Verification

- **Wireshark:** flood traffic stops appearing on Ubuntu's capture.
- **pfSense logs:** *Status ▸ System Logs ▸ Firewall* (or the rule's log icon) shows the blocked packets in real time.

![pfSense Firewall Logs](screenshots/pfsense-logs.png)

## Key Takeaways

- Network segmentation with bridged vs. internal VirtualBox adapters
- pfSense rule ordering and its effect on traffic evaluation
- Recognizing flood traffic in Wireshark
- Practical firewall-based DoS mitigation and log-based verification

## Disclaimer

This lab is for **educational purposes only**, run entirely on isolated virtual machines within a personal home lab. Do not run flood tools (`hping3` or similar) against any network, host, or device you do not own or have explicit permission to test.

---
