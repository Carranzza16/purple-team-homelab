# Session 4 — Writing a Sigma Rule

## Objective
Translate the SSH brute force pattern observed in Session 3 into a
portable Sigma rule, usable across different SIEMs.

## Source pattern
Based on direct observation in `/var/log/auth.log` and Wazuh alerts:
- Failed login attempts contain the string `Failed password`
- Wazuh's own brute force correlation rule fired after multiple
  failed attempts from the same source IP within a short window

## The rule
See [rule.yml](./rule.yml).

Key design decisions:
- **Threshold:** 10 failed attempts in 5 minutes — high enough to
  avoid flagging normal typos, low enough to catch automated tools
  like Hydra
- **Grouping by `src_ip`:** ensures the count is per-attacker, not
  global across all SSH traffic
- **MITRE ATT&CK tag (T1110.001):** maps directly to "Password
  Guessing" under Brute Force, connecting this rule to the broader
  threat framework

## What I learned
Sigma rules separate **detection logic** from the **SIEM that runs
it**. The same rule that could theoretically run on Wazuh could be
translated to Splunk or Elastic without rewriting the core logic —
only the field mappings change. This is the same logic Wazuh used
internally to generate its brute force alert in Session 3, now
written in a vendor-neutral format.

## Next steps
- Test rule conversion using the Sigma CLI (`sigma convert`) to a
  Wazuh-compatible format
- Expand detection to cover the "invalid user" scenario separately
