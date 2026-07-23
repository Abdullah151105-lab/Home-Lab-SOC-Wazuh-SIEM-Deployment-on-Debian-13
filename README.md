A self-hosted Security Information and Event Management (SIEM) lab built to learn Blue Team fundamentals — log analysis, threat detection, file integrity monitoring, and MITRE ATT&CK mapping — using [Wazuh](https://wazuh.com/) on a Debian 13 virtual machine.

## Overview

This project documents the process of standing up an all-in-one Wazuh deployment (Indexer, Manager, Dashboard) from scratch, including the real-world troubleshooting that came with running it on an unsupported, ARM64-based Linux distribution. The goal was to gain hands-on experience with a production-grade SIEM rather than just reading about one.

## Environment

| Component        | Detail                                  |
|-------------------|------------------------------------------|
| Host              | macOS (Apple Silicon)                   |
| Virtualization    | Oracle VirtualBox                       |
| Guest OS          | Debian 13 (Trixie), ARM64/aarch64       |
| SIEM              | Wazuh 4.14.6 (all-in-one: Indexer, Manager, Dashboard) |

## Architecture

```
┌─────────────────────────────────────┐
│           Debian 13 VM               │
│                                       │
│  ┌───────────────┐  ┌──────────────┐ │
│  │ Wazuh Indexer  │  │ Wazuh Manager│ │
│  │ (OpenSearch)   │◄─┤              │ │
│  └───────┬────────┘  └──────────────┘ │
│          │                            │
│  ┌───────▼────────┐                   │
│  │ Wazuh Dashboard │◄── Browser (HTTPS)
│  └────────────────┘                   │
└─────────────────────────────────────┘
```

## Setup Steps

1. Provisioned a Debian 13 VM in VirtualBox (ARM64).
2. Installed Wazuh via the official all-in-one install script:
   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
   sudo bash wazuh-install.sh -a
   ```
3. Verified all three services (`wazuh-indexer`, `wazuh-manager`, `wazuh-dashboard`) were active via `systemctl status`.
4. Confirmed the Indexer API responded correctly before touching the web UI:
   ```bash
   sudo curl -k -u 'admin:<password>' https://localhost:9200
   ```
5. Logged into the Wazuh Dashboard over HTTPS and explored the built-in modules: Threat Hunting / Security Events, Vulnerability Detection, File Integrity Monitoring, and MITRE ATT&CK mapping.

## Challenges & Fixes

Running Wazuh on an unsupported ARM64 host surfaced several real issues — documenting them was as valuable as the install itself.

**1. Install script rejected the system as "not 64-bit"**
The pinned `4.9` install script had an outdated architecture check that misidentified `aarch64` (ARM64) systems. Fixed by using the current script version (`4.14`), which supports ARM64 properly.

**2. Dashboard installation failed: "no space left on device"**
`df -h` showed plenty of free space *inside* the guest OS, which was misleading. The root cause was the VirtualBox virtual disk (`.vdi`) itself hitting its allocated maximum (17.78 GB of 20 GB), since dynamically-allocated VDIs don't reclaim space automatically.
- Resized the VDI from the host via `VBoxManage modifymedium disk <path> --resize 40960`.
- Discovered the swap partition sat directly after the root partition, blocking `growpart` from extending it.
- Removed the swap partition, extended the root partition to the full disk with `parted`, and replaced it with a swapfile instead of a swap partition.

**3. Wazuh Agent installation failed on the same host as the Manager**
`wazuh-agent` and `wazuh-manager` packages conflict and cannot be installed on the same system — the Manager already monitors itself. A second VM (or container) is required to deploy an agent and generate cross-host telemetry.

**4. Dashboard appeared "frozen" — no new events showing**
Not a bug: the dashboard's time-range filter (top right) defaults to a short window (e.g. "Last 15 minutes"). Events outside that window simply don't render, even though the underlying data is current. Widening the range and refreshing resolved it.

## What the Dashboard Revealed

- **Threat Hunting / Security Events** — real-time alert feed with severity levels and rule IDs, generated even without external attacks (the Manager monitors its own host by default).
- **Vulnerability Detection** — automatic CVE scanning against installed packages.
- **File Integrity Monitoring (FIM)** — detects changes to monitored system paths on an interval-based scan.
- **MITRE ATT&CK mapping** — alerts are automatically correlated to known adversary tactics and techniques where applicable.

## Limitations Investigated

- Wazuh has no official mobile agent (Android/iOS) — endpoint monitoring is limited to traditional OS platforms. True visibility into mobile devices on a home network would require network-level tooling (e.g. Suricata/Zeek on a router/switch with port mirroring) rather than a host-based agent.

## Next Steps

- [ ] Deploy a second VM (Kali or plain Debian) and install a Wazuh agent to generate real cross-host telemetry.
- [ ] Simulate basic attacks (SSH brute-force, Nmap port scans) and verify detection + MITRE ATT&CK mapping.
- [ ] Add Suricata for network-level visibility, correlated into Wazuh.
- [ ] Explore Wazuh on a Raspberry Pi as a low-power, always-on home-network monitoring node.

## Key Takeaways

- Hands-on experience deploying and troubleshooting a real SIEM stack, not just following a tutorial end-to-end.
- Practical debugging across the VM/host boundary (VirtualBox disk provisioning, partition layout, systemd services).
- Understanding of Wazuh's architecture (Indexer/Manager/Dashboard separation) and its scope (host-based, not network-based) — including where it needs to be paired with other tools.
