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

## Kali Attacker Network Configuration
<img width="1465" height="893" alt="WhatsApp Image 2026-09-07 at 07 07 08" src="https://github.com/user-attachments/assets/ef59d49c-1e12-428d-a3e7-3af78213c3aa" />

## Ubuntu Network Configuration
<img width="1477" height="865" alt="WhatsApp Image 2026-09-07 at 07 30 38" src="https://github.com/user-attachments/assets/9f3f8d66-b7c0-4af2-af4f-fd60f8adbc1c" />

## Pfsense LAN/WAN Network Configuration
<img width="1470" height="870" alt="WhatsApp Image 2026-09-07 at 07 30 02" src="https://github.com/user-attachments/assets/2f4617a4-f538-4fa6-b983-9dd60b7bd199" />

<img width="1481" height="871" alt="WhatsApp Image 2026-09-07 at 07 30 16" src="https://github.com/user-attachments/assets/0e5da458-e439-4990-933f-e5266037c957" />


### 2. pfSense Configuration

1. Install pfSense with default settings.
2. At the console menu, assign interfaces: `vtnet0` → WAN, `vtnet1` → LAN.
3. Leave WAN on DHCP — it will pull an address automatically (e.g. `172.20.10.2`).

<img width="936" height="592" alt="WhatsApp Image 2026-09-07 at 07 33 21" src="https://github.com/user-attachments/assets/5ffaa325-476e-4362-9d63-a5755ec3868b" />

**Enable temporary WAN GUI access** (console):
```
pfSense-d
```
Then log into `https://<WAN_IP>` with the default credentials (`admin` / `pfsense`) and immediately lock it down with a proper rule.

<img width="1537" height="1020" alt="WhatsApp Image 2026-09-07 at 07 35 24" src="https://github.com/user-attachments/assets/5448a49e-8805-422f-9a9a-39b097c6b610" />


**Firewall ▸ Rules ▸ WAN ▸ Add**

| Field       | Value                          |
|-------------|---------------------------------|
| Action      | Pass                            |
| Protocol    | TCP                              |
| Source      | Kali IP (`172.20.10.3`)         |
| Destination | WAN address                     |
| Port        | 443                              |
| Description | Allow GUI access from Kali      |


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

<img width="1015" height="757" alt="WhatsApp Image 2026-09-07 at 07 44 35" src="https://github.com/user-attachments/assets/eba4f898-187f-40bf-ad13-3668976f2a91" />

<img width="917" height="736" alt="WhatsApp Image 2026-09-07 at 07 45 59" src="https://github.com/user-attachments/assets/0fc97031-9b91-47c5-91ab-b94cf939afd4" />

### 4. Ubuntu (Victim)

Ubuntu receives `192.168.1.100/24` via pfSense DHCP automatically. Confirm:

<img width="1373" height="630" alt="WhatsApp Image 2026-09-07 at 07 47 46" src="https://github.com/user-attachments/assets/a84d9b16-6123-4e58-a8e9-8cb779b17ea8" />

```bash
ip a
ping -c3 google.com
```

## Attack Simulation

Install and start Wireshark on Ubuntu, capturing on `enp0s3`:

<img width="1600" height="848" alt="WhatsApp Image 2026-09-07 at 07 49 46" src="https://github.com/user-attachments/assets/89a58d1f-12a7-438c-95fc-c94ff84d59c6" />


```bash
sudo apt update && sudo apt install -y wireshark
sudo wireshark
```

<img width="1566" height="982" alt="WhatsApp Image 2026-09-07 at 07 50 10" src="https://github.com/user-attachments/assets/927b5dcc-3e60-4de2-9a22-f01a80796996" />

From Kali, launch a flood:

```bash
# ICMP flood
sudo hping3 -1 --flood 192.168.1.100

# or SYN flood
sudo hping3 --flood -S -p 80 192.168.1.100
```
<img width="960" height="702" alt="WhatsApp Image 2026-09-07 at 07 53 11" src="https://github.com/user-attachments/assets/67209cd3-a5f8-4035-8cf8-97d2327e2108" />


> **Tip:** a flood generates thousands of packets/sec, which can be hard to read raw. Use a display filter (`ip.addr == 172.20.10.3` or `tcp.flags.syn == 1`) or **Statistics ▸ I/O Graph** for a clean visual.

<img width="1550" height="986" alt="WhatsApp Image 2026-09-07 at 07 54 00" src="https://github.com/user-attachments/assets/601d0cb6-83d1-4e59-af1e-dd56f7b66de8" />


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

<img width="1516" height="805" alt="WhatsApp Image 2026-09-07 at 07 35 46" src="https://github.com/user-attachments/assets/30c02913-617a-4eef-94a4-33371427c3dc" />


## Verification

- **Wireshark:** flood traffic stops appearing on Ubuntu's capture.
- **pfSense logs:** *Status ▸ System Logs ▸ Firewall* (or the rule's log icon) shows the blocked packets in real time.

<img width="1562" height="1007" alt="WhatsApp Image 2026-09-07 at 08 02 25" src="https://github.com/user-attachments/assets/562c5b75-f363-4b3b-9d4d-53e5cb1de356" />


## Key Takeaways

- Network segmentation with bridged vs. internal VirtualBox adapters
- pfSense rule ordering and its effect on traffic evaluation
- Recognizing flood traffic in Wireshark
- Practical firewall-based DoS mitigation and log-based verification

## Disclaimer

This lab is for **educational purposes only**, run entirely on isolated virtual machines within a personal home lab. Do not run flood tools (`hping3` or similar) against any network, host, or device you do not own or have explicit permission to test.

---
