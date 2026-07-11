# Session 5 — Suricata NIDS Integration

## Objective
Detect the port scan that went unnoticed in Session 2 by adding
network-based detection (NIDS) alongside Wazuh's host-based detection.

## Setup
- Suricata installed via PPA on Ubuntu_principal
- Interface: enp0s3
- Custom local rule written to detect port scan patterns
- Wazuh configured to read Suricata's eve.json log

## Steps

### 1. Install Suricata
```bash
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
sudo apt install suricata -y
sudo suricata-update
```

### 2. Configure interface
Edit /etc/suricata/suricata.yaml and set all interface references to enp0s3.

### 3. Add Wazuh localfile
In /var/ossec/etc/ossec.conf:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### 4. Custom detection rule
Created /var/lib/suricata/rules/local.rules:
alert tcp any any -> $HOME_NET any (msg:"Possible Port Scan Detected";

flags:S; threshold:type both, track by_src, count 20, seconds 10;

sid:1000001; rev:1;)
### 5. Repeat nmap scan
```bash
sudo nmap -sS -p 1-1000 192.168.100.16
```
Result: alert visible in Wazuh dashboard under Threat Hunting.

## Troubleshooting resolved

### Problem 1 — apt blocked by unattended-upgrades
Process 9004 held the dpkg lock indefinitely.
```bash
sudo systemctl stop unattended-upgrades
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo dpkg --configure -a
```

### Problem 2 — Suricata service crashing on startup
Error: `af-packet: eth0: failed to find interface type: No such device`

Root cause: suricata.yaml contains multiple interface declarations
across different sections. Only editing the primary af-packet section
was not enough — a leftover eth0 reference on line 856 caused the
service to crash.

Found via:
```bash
grep -n "interface" /etc/suricata/suricata.yaml
```

Fix: changed all eth0 references to enp0s3.

### Problem 3 — Default ruleset has no port scan detection
ET Open ruleset does not include generic port scan detection rules.
Solution: wrote a custom rule using threshold by_src, count 20, seconds 10.

### Problem 4 — Suricata detected but Wazuh showed nothing
Wazuh manager needed a restart to re-read the updated eve.json file.
```bash
sudo systemctl restart wazuh-manager
```

## Comparison: Wazuh native rule vs custom Sigma rule

| | Wazuh native (5763) | Custom Sigma rule |
|---|---|---|
| Threshold | 8 attempts | 10 attempts |
| Time window | 120 seconds | 300 seconds |
| Grouping | same_source_ip | count() by src_ip |
| Cooldown | 60 seconds | not defined |
| MITRE | T1110 | T1110.001 |

## What I learned
Network-based detection (NIDS) and host-based detection (HIDS) are
complementary. Wazuh alone could not see the port scan. Suricata alone
would not connect to the broader SIEM correlation Wazuh provides.
Integrating both closes the blind spot identified in Session 2.

The default ET Open ruleset does not include generic port scan
detection — custom rules are necessary for basic network reconnaissance
detection.
