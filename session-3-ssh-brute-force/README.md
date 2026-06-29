# Session 3 — SSH Brute Force Detection

## Objective
Simulate an SSH brute force attack with Hydra and analyze how
Wazuh logs and classifies it.

## Setup
- Attacker VM: Kali Linux, using Hydra
- Target: Ubuntu_principal, with SSH service enabled
- Wazuh manager configured to read `/var/log/auth.log`
  (not enabled by default — added manually in `ossec.conf`)

## Steps
1. Enabled SSH on the target: `sudo apt install openssh-server`
2. Confirmed Wazuh was not reading `auth.log` by default
3. Added a `<localfile>` entry in `ossec.conf` pointing to
   `/var/log/auth.log` and restarted `wazuh-manager`
4. Verified detection worked with a manual failed SSH login first
5. Ran Hydra against a non-existent user:
   `hydra -l adriana -P wordlist.txt ssh://<target-ip>`
6. Ran Hydra against a real, existing user with wrong passwords:
   `hydra -l <real-user> -P wordlist.txt ssh://<target-ip>`

## Findings
- Attempts against the non-existent user were logged as
  **invalid user** authentication failures
- Attempts against the real user were logged as **authentication
  failed** events, and Wazuh additionally raised a dedicated
  **brute force attack** alert — a correlation rule triggered by
  multiple failed logins from the same source in a short time
  window

## What I learned
Wazuh distinguishes between different failure types in `auth.log`
(invalid user vs. wrong password for an existing user), and brute
force detection isn't just "one alert per failed log" — it's a
correlation rule that activates only after a pattern (volume +
timeframe + same source) is met. This is the same logic later
captured into the Sigma rule written in Session 4.
