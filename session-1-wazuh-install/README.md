# Session 1 — Wazuh Installation

## Objective
Get Wazuh running on a VM using the official Quickstart install,
with no detection configuration yet.

## Setup
- VirtualBox VM, Ubuntu Server
- Wazuh installed via the official Quickstart curl command

## Steps
1. Created Ubuntu VM in VirtualBox
2. Ran the official Quickstart install command
3. Accessed the dashboard via browser using the VM's IP

## Findings
Dashboard came up with default modules visible: Endpoint Security,
Threat Intelligence, Security Operations, Cloud Security.
No agents were registered initially — the Quickstart install only
sets up the manager/indexer/dashboard core, not a separate agent.

## Issue encountered
Attempted to install `wazuh-agent` on the same machine as the manager.
This caused `apt` to automatically remove `wazuh-manager`, since the
manager already monitors itself internally as "agent 000" — having
both packages on the same host creates a conflict. Reinstalled
`wazuh-manager` to fix it.

## What I learned
The Wazuh "core" (manager + indexer + dashboard) is separate from the
agent package. The manager already monitors its own host by default;
a separate agent is only needed to extend monitoring to *other*
machines, not the manager's own host.
