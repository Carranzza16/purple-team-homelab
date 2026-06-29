# purple-team-homelab
Hands-on home lab project simulating attacks and building detection
capability using Wazuh SIEM. Built as a practical complement to SOC
analyst work and cybersecurity certification studies (CompTIA CySA+,
Cisco CCNA).

## Goal

Move beyond passive alert monitoring and understand detection from
the other side: how logs are generated, how SIEM rules correlate
events, and why certain attacks are detected while others require
additional tooling.

## Lab Architecture

- **Ubuntu_principal** — Wazuh manager + indexer + dashboard (also
  acts as the monitored endpoint via the built-in agent 000)
- **attack** — Kali Linux VM used to run nmap and Hydra against
  Ubuntu_principal
- Network: VirtualBox internal network between both VMs

## Sessions

| Session | Focus | Status |
|---|---|---|
| [Session 1](./session-1-wazuh-install) | Wazuh installation | ✅ Done |
| [Session 2](./session-2-nmap-scan) | Port scanning detection | ✅ Done |
| [Session 3](./session-3-ssh-bruteforce) | SSH brute force detection | ✅ Done |
| Session 4 | Sigma rule writing | 🔜 Next |

## Key takeaway across sessions

Wazuh's default setup is host-based (log analysis), not network-based.
This means some attacks (like a simple port scan) generate no alert
by default, while others (like SSH brute force) are detected
immediately once the right log source is configured. Understanding
this distinction was the main lesson of this lab.

## Tools used

- Wazuh (manager, indexer, dashboard)
- VirtualBox
- nmap
- Hydra
- Kali Linux / Ubuntu 
