# Session 2 — Port Scan Detection

## Objective
Run an nmap scan from a separate attacker VM and check whether
Wazuh detects it.

## Setup
- Attacker VM: Kali Linux
- Target: Ubuntu_principal (Wazuh manager host)
- Network: VirtualBox internal network between both VMs

## Steps
1. Verified connectivity between VMs with `ping`
2. Ran: `nmap -sV -p 1-1000 <target-ip>`
3. Checked Wazuh dashboard (Threat Hunting) for related alerts

## Findings
- Only port 443 (Wazuh dashboard itself) was open
- No alert was generated for the scan itself

## Why no alert appeared
Wazuh's default detection is **host-based** (log analysis), not
**network-based**. A port scan with nmap only performs a TCP
handshake to check if a port responds — it doesn't attempt
authentication or trigger any local log entry. Detecting a scan
like this would require a network-based IDS (e.g., Suricata
integrated with Wazuh) or a specific correlation rule for
connection patterns, neither of which is active by default.

## What I learned
Host-based SIEM detection depends entirely on what gets logged
locally. An action that doesn't generate a local log (like a simple
port scan) is invisible to this setup, even though it clearly
happened. Network-level visibility requires different tooling.
