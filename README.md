<div align="center">

# 🏠 Home Cybersecurity Lab

### SIEM Administration · Active Directory · Detection Engineering · Attack Simulation

![Status](https://img.shields.io/badge/status-actively%20expanding-brightgreen)
![Proxmox](https://img.shields.io/badge/-Proxmox%20VE-E57000?style=flat&logo=proxmox&logoColor=white)
![Security Onion](https://img.shields.io/badge/-Security%20Onion-2E7D32?style=flat)
![VirtualBox](https://img.shields.io/badge/-VirtualBox-183A61?style=flat&logo=virtualbox&logoColor=white)
![Kali Linux](https://img.shields.io/badge/-Kali%20Linux-557C94?style=flat&logo=kalilinux&logoColor=white)
![Sigma](https://img.shields.io/badge/-Sigma%20Rules-6E44FF?style=flat)

</div>

---

## 📌 Overview

This repo documents a three-node home cybersecurity lab built from scratch on personal hardware to gain hands-on, resume-relevant experience with **SIEM administration, Active Directory, endpoint telemetry, attack simulation, and detection engineering**.

The lab runs a bare-metal **Proxmox** hypervisor hosting **Security Onion** (Suricata, Zeek, Elastic Stack), a **VirtualBox**-based Active Directory environment (Domain Controller + domain-joined Windows client, both instrumented with Sysmon and Elastic Agent), and a **Kali Linux** attacker machine. Building it surfaced a long list of real infrastructure problems — networking, firewall rules, certificate trust, DNS, Windows logging gaps — all diagnosed and documented below. The environment was then used to simulate real attack techniques and validate detection coverage, culminating in three custom Sigma rules.

**Hardware:** 3 laptops (16GB RAM each) + 1 desktop (8GB RAM, management/browsing only) — no cloud, no budget beyond what was already owned.

> 📄 Full build log, architecture detail, and portfolio writeup: [`Home_Lab_Documentation.pdf`](./Home_Lab_Documentation.pdf)

---

## 🗺️ Architecture

| Machine | Hypervisor | Role | VM(s) | Key IP(s) |
|---|---|---|---|---|
| Laptop 2 | Proxmox VE (bare metal) | Detection / SIEM server | Security Onion | 192.168.123.226 (mgmt) |
| Laptop 1 | VirtualBox (hosted) | AD victim environment | DC1 (Domain Controller), WIN11 (domain client) | DC1: 10.10.10.10 / 192.168.123.x · WIN11: 10.10.10.20 / 192.168.123.x |
| Laptop 3 | VirtualBox (hosted) | Attacker machine | Kali Linux | 192.168.123.247 |
| Desktop | — | Management / browsing | — | — |

**Network segments**
- **Bridged home network** (`192.168.123.0/24`) — every machine's WAN-facing adapter connects here; it's the only path devices on different hypervisors (Proxmox vs. VirtualBox) can reach each other.
- **Internal AD network** (`10.10.10.0/24`, VirtualBox internal switch) — isolated segment used only by DC1 and WIN11 for domain authentication.
- **Security Onion monitor interface** (`ens19`) — deliberately unconnected to anything external; sniffs traffic on Proxmox's own virtual switch only.

```
+------------------+   +----------------------------------+   +----------------+
|     Laptop 2     |   |             Laptop 1              |   |    Laptop 3    |
|   Proxmox VE     |   |            VirtualBox             |   |   VirtualBox   |
|                  |   |                                    |   |                |
|  Security Onion  |   |   DC1 <---10.10.10.0/24---> WIN11   |   |      Kali      |
|  192.168.123.226 |   |  (AD, DNS)         (domain client)  |   |   (attacker)   |
+--------+---------+   +---------------+--------------------+   +-------+--------+
         |                             |                                |
         +-----------------------------+--------------------------------+
                              Home network (bridged)
                              192.168.123.0/24
```

---

## 🔨 Build Process

**Proxmox + Security Onion**
- Bare-metal Proxmox VE on Laptop 2; disabled paid enterprise/Ceph repos, enabled `pve-no-subscription`.
- Deployed Security Onion (Eval Mode): 4 vCPU, 8GB RAM, 150GB disk, dual NICs (`ens18` = management, `ens19` = monitor).
- Configured static management IP, gateway, and DNS during setup.

**Active Directory Environment**
- Built DC1 (Windows Server, 4GB RAM, 2 vCPU) and promoted it to Domain Controller for a new forest, `lab.local`.
- Built WIN11 (Windows 11, 5GB RAM, 3 vCPU), pointed DNS at DC1, joined it to the domain.
- Installed Sysmon (SwiftOnSecurity config) on both hosts for high-fidelity process, network, and file telemetry.
- Added bridged adapters to both AD machines so they could reach Security Onion while keeping the internal AD segment intact.
- Enrolled both hosts in Security Onion's Elastic Fleet, collecting Sysmon Operational, PowerShell Operational, and Security event log channels.

**Kali Attacker Machine**
- Imported Kali's official VirtualBox image on Laptop 3 (8GB RAM, 4 vCPU).
- Verified connectivity to all lab hosts over the bridged network.
- Installed Atomic Red Team on DC1 for structured MITRE ATT&CK technique testing.

---

## 🧯 Troubleshooting Journal

Real infrastructure problems encountered and resolved during the build:

| Issue | Root Cause | Resolution |
|---|---|---|
| Proxmox black screen / no Ethernet | Wi-Fi-only laptop; installer needs a NIC driver at install time | Selected the detected Wi-Fi NIC as the management interface and supplied credentials during install |
| Security Onion bonding failure on monitor NIC | `bond0` configured for MTU 9000; VirtIO NIC doesn't support jumbo frames | Set both bond0 and slave interface to MTU 1500 via `nmcli`, cycled connections |
| "IP being routed is not the IP assigned" | `ens19` held a leftover DHCP lease with a competing default route | Flushed the IP from `ens19` and brought it down before re-running `so-setup` |
| Elastic Agent download failed (404) | `/opt/socore/html` artifact directory was never created — a genuine provisioning gap | Downloaded the Elastic Agent installer directly from Elastic's site instead |
| Elastic Agent enrollment x509 error | Fleet Server uses a self-signed certificate | Added `--insecure` to the install/enroll command |
| Agent healthy in Fleet but shipping no data | Agents resolved `securityonion` via public DNS, which has no record for it | Added a manual hosts file entry on each Windows machine |
| Elasticsearch output degraded (separate x509 error) | `--insecure` only covered enrollment, not ongoing Elasticsearch output | Set `ssl.verification_mode: none` on the Fleet output |
| Elasticsearch (9200) unreachable from agents | iptables only allowed traffic from Security Onion's own Docker network | Located the `elasticsearch_rest` hostgroup and added it via the pillar file (`soc_firewall.sls`), re-ran a highstate |
| Suricata/Zeek show zero traffic between Kali and DC1 | Proxmox connects over Wi-Fi; Kali↔DC1 traffic never crosses Proxmox's virtual switch | Documented as a known limitation; relied on host-based telemetry instead |
| No Event ID 4104 for encoded PowerShell | Script Block Logging not enabled by default | Enabled via registry policy (`ScriptBlockLogging`) |
| Sigma rule failed to convert (HTTP 500) | Converter didn't support `count() by field > N` aggregation syntax | Rewrote as a simple field-match (single-event) detection |

---

## ⚔️ Attack Simulation & Detection Results

Each scenario was executed from Kali or on the target Windows host, then confirmed against real ingested data in Security Onion.

| Scenario | Tool / Technique | MITRE ATT&CK | Detected Via | Result |
|---|---|---|---|---|
| Network reconnaissance | Nmap (`-sV -A`) | T1046 – Network Service Discovery | — | Not visible to Suricata/Zeek (Wi-Fi visibility gap); no Sysmon Event ID 3 generated |
| Authenticated file share access | smbclient | T1021.002 – SMB/Windows Admin Shares | Security Event 4624 (Logon Type 3) | Successful logon captured, including source IP and account |
| Brute-force / password guessing | netexec | T1110 – Brute Force | Security Event 4625 | Multiple failed logons captured, same account and source IP |
| Credential dumping attempt | Mimikatz (`sekurlsa::logonpasswords`, `sekurlsa::tickets`) | T1003 – OS Credential Dumping | Sysmon Event ID 1 | Process launch logged; extraction blocked by modern Windows LSA Protection |
| Obfuscated command execution | Encoded PowerShell (`-EncodedCommand`) | T1059.001 / T1027 | PowerShell Operational Event 4104 | Invisible until Script Block Logging enabled; fully visible (decoded) afterward |
| Structured technique test | Atomic Red Team T1082 | T1082 – System Information Discovery | Windows Defender detection log | Simulated technique blocked and logged by Defender |

---

## 🛡️ Custom Detection Rules (Sigma)

Three Sigma rules were authored, converted to EQL, enabled, and validated against live attack traffic.

<details>
<summary><strong>6.1 — Failed Logon Attempt Detected</strong> (T1110)</summary>

```yaml
title: 'Failed Logon Attempt Detected'
id: 7d8f3a1e-4b2c-4e9a-9f1d-2a3b4c5d6e7f
status: 'experimental'
description: |
    Detects a failed logon attempt (Event ID 4625) which may indicate an
    incorrect password or an attempted brute-force attack against a
    Windows account.
author: '@user'
tags:
  - attack.credential_access
  - attack.t1110
logsource:
  category: authentication
  product: windows
detection:
    selection:
        EventID: 4625
    condition: selection
level: 'medium'
```

> **Note:** An initial version used `count() by IpAddress > 3` to require multiple failures from one source before alerting. Security Onion's Sigma-to-EQL converter didn't support that aggregation syntax (HTTP 500 on conversion), so it was replaced with the single-event version above. A proper threshold-based version would be implemented via ElastAlert.

</details>

<details>
<summary><strong>6.2 — Mimikatz Process Execution Detected</strong> (T1003)</summary>

```yaml
title: 'Mimikatz Process Execution Detected'
id: 3f4a5b6c-7d8e-4f9a-b0c1-d2e3f4a5b6c7
status: 'experimental'
description: |
    Detects execution of mimikatz.exe, a well-known credential dumping
    tool used to extract passwords, hashes, and Kerberos tickets from
    memory. Execution of this tool is a strong indicator of credential
    access activity and is rarely, if ever, legitimate.
author: '@user'
tags:
  - attack.credential_access
  - attack.t1003
logsource:
  category: process_creation
  product: windows
detection:
    selection:
        Image|endswith: '\\mimikatz.exe'
    condition: selection
level: 'high'
```

</details>

<details>
<summary><strong>6.3 — Encoded PowerShell Command Execution</strong> (T1059.001 / T1027)</summary>

```yaml
title: 'Encoded PowerShell Command Execution'
id: 8a9b0c1d-2e3f-4a5b-6c7d-8e9f0a1b2c3d
status: 'experimental'
description: |
    Detects execution of PowerShell with the -EncodedCommand (or
    shorthand -enc/-e) parameter, commonly used by attackers to
    obfuscate malicious commands and evade simple text-based
    detection. Legitimate administrative use of this flag is uncommon.
author: '@user'
tags:
  - attack.execution
  - attack.t1059.001
  - attack.defense_evasion
  - attack.t1027
logsource:
  category: process_creation
  product: windows
detection:
    selection_img:
        - Image|endswith: '\\powershell.exe'
        - OriginalFileName: 'PowerShell.EXE'
    selection_cli:
        CommandLine|contains|windash:
            - ' -enc'
            - ' -EncodedCommand'
    condition: all of selection_*
level: 'high'
```

</details>

---

## ⚠️ Known Limitations

- **Network visibility gap:** Suricata/Zeek can't see traffic between other bridged devices (e.g., Kali → DC1). Proxmox connects to the home network over Wi-Fi, so that traffic never crosses Proxmox's own virtual switch and can't be mirrored to the monitor interface. Wired switches share this limitation without a SPAN/mirror port.
- The Security Onion nginx artifact directory (`/opt/socore/html`) was never created by this install; the Fleet-provided one-line agent download fails as a result. Manual download from Elastic's site is the permanent workaround.
- Suricata shows a persistent "rule mismatch" status. The documented cause (Elasticsearch disk watermark) was checked and ruled out — not pursued further since it doesn't affect any currently-used detection path.
- Sysmon (default SwiftOnSecurity config) didn't log the Nmap scan or initial SMB connection as Event ID 3 — the config is tuned toward outbound/anomalous connections rather than every inbound connection to a standard service.

---

## 💡 Lessons Learned

- Defense in depth is real: three detection layers (network, host telemetry, endpoint protection) each caught different things — and each had blind spots the others covered.
- Logging isn't "on" by default — PowerShell Script Block Logging had to be explicitly enabled before encoded commands became visible, mirroring a common real-world enterprise gap.
- Being on the same subnet doesn't mean a monitoring tool can see all traffic on it — genuine visibility requires sitting in the actual physical/virtual path the traffic flows through.
- Splitting a lab across multiple hypervisors on separate machines is beginner-friendly and resource-flexible, but trades away easy network-level visibility.
- Detection rule syntax support varies by platform — valid Sigma logic still needs to be checked against what the specific backend actually supports.

---

<div align="center">

Built and documented by **Chris Poosuntisumpun** — [LinkedIn](https://www.linkedin.com/in/chrispoosuntisumpun) · [chris.poosuntisumpun@gmail.com](mailto:chris.poosuntisumpun@gmail.com)

</div>
