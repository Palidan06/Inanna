# Inanna

**Automated Host Discovery, Patching, and Validation with SaltStack**

---

## Table of Contents

1. [Overview](#overview)  
2. [Prerequisites](#prerequisites)  
3. [Project Structure](#project-structure)  
4. [Pillar Configuration](#pillar-configuration)  
5. [Module Descriptions](#module-descriptions)  
6. [Usage](#usage)  
7. [Logging and Reports](#logging-and-reports)  
8. [Sample Outputs](#sample-outputs)  
9. [Next Steps](#next-steps)  

---

## Overview

`Inanna` is a SaltStack-based automation framework designed to:

- **Discover** hosts, including non-minions, by network scan and service probe.  
- **Snapshot** VMs on ESXi/vCenter before patching.  
- **Patch** operating systems and applications (full or security-only).  
- **Reboot** hosts based on configuration.  
- **Validate** post-patch service health.  
- **Cleanup** old snapshots, logs, and temporary files.  
- **Build Salt SSH rosters** automatically for agentless management.

All modules are orchestrated via a single entry point (`init.sls`), enabling an end-to-end, hands-off workflow for both Salt minions and SSH targets.

---

## Prerequisites

- **Salt Master** and **Salt Minions**, or access via **salt-ssh**.  
- **Python SDK for VMware** (`pyVmomi`) installed on the Salt Master or proxy:  
  ```bash
  pip install pyvmomi
  ```

- **Pillar data** defined for `inanna` (see [Pillar Configuration](#pillar-configuration)).

---

## Project Structure
```bash
inanna_salt/
├── init.sls
├── discover.sls
├── roster_build.sls
├── snapshot.sls
├── patch_os.sls
├── patch_apps.sls
├── reboot.sls
├── validate.sls
├── cleanup.sls
├── rollback.sls (planned)
└── pillar/
    ├── top.sls
    └── inanna.sls
```
---

## Pillar Configuration
```bash
inanna:
## Simulation & control flags
  dry_run:        true
  skip_snapshot:  false
  auto_reboot:    false
  os_update_mode: all                 # all | security_only

## Discovery & targeting
  scan_ranges:                        # Multi-subnet scan support
    - '192.168.1.0/24'
    - '192.168.2.0/24'
    - '192.168.3.0/24'
  targets:                            # Whitelisted known hosts
    - 192.168.1.10
    - 192.168.1.11
  exclude_hosts:                      # Never manage these
    - 192.168.1.50
  discovery_ports: [22, 80, 443]      # Probe ports for "alive" detection

## Services to verify post-patch
  critical_services:
    - sshd
    - nginx
    - netbox

## VMware snapshot credentials
  vmware:
    host:       "vcenter.example.local"
    user:       "administrator@vsphere.local"
    password:   "YourPassword"
    protocol:   "https"
    port:       443

## VMs to snapshot
  vm_names:
    - LabHost1
    - LabHost2

## Cleanup retention (days)
  cleanup_retention_days: 7
```
---

## Module Descriptions

### init.sls
**Purpose**: Orchestrates the entire pipeline.  
**Sequence**:  
1. `discover.sls`     – Scan subnets, find live hosts (minion or SSH), collect system data.  
2. `roster_build.sls` – Auto-generate Salt SSH roster from discovered hosts.  
3. `snapshot.sls`     – Take VMware snapshots if configured.  
4. `patch_os.sls`     – Apply OS updates.  
5. `patch_apps.sls`   – Apply application updates.  
6. `reboot.sls`       – Reboot if configured.  
7. `validate.sls`     – Check critical services.  
8. `cleanup.sls`      – Remove old snapshots/logs/temp files.  

---

### discover.sls
**Purpose**: Network scan + data collection.  
**Features**:  
- Nmap ping scan across all `scan_ranges`.  
- Optional port probes (`discovery_ports`).  
- Identifies both Salt minions and agentless hosts.  
- Writes host-specific discovery reports.  
- Generates `inanna_alive_mgmt_<timestamp>.txt` with reachable hosts.  

---

### roster_build.sls
**Purpose**: Create `/etc/salt/roster` entries for agentless hosts.  
**Logic**:  
- Reads alive host list from `discover.sls`.  
- Assigns default SSH user (can be overridden in pillar).  
- Adds to Salt SSH roster for easy targeting.  

---

### snapshot.sls
**Purpose**: VMware VM snapshots before patching.  
**Snapshot Name**: `inanna_prepatch_YYYYMMDD_HHMM`.  

---

### patch_os.sls
**Purpose**: OS-level patching.  
**Respects**:  
- `dry_run`  
- `os_update_mode`  
- `exclude_hosts`  
**Logs to**: `/var/log/inanna_patching.log`.  

---

### patch_apps.sls
**Purpose**: Application patching.  
- Same conditional logic as `patch_os.sls`.  

---

### reboot.sls
**Purpose**: Conditional reboot.  
**Logs to**: `/var/log/inanna_reboot.log`.  

---

### validate.sls
**Purpose**: Ensure `critical_services` are running post-patch.  

---

### cleanup.sls
**Purpose**: Delete snapshots, logs, and temp files older than retention period.  

---

### rollback.sls (Planned)
**Purpose**: Restore VM to pre-patch snapshot if validation fails.  

---

## Usage
```bash
# Run on all hosts (Minions or not (SSH)):

```bash
salt '*' state.apply inanna
salt-ssh '*' state.apply inanna --roster=/etc/salt/roster
```

# For just running Discovery:

```bash
salt '*' state.apply inanna.discover
```

# To build the SSH roster after running Discovery

```bash
salt-call state.apply inanna.roster_build
```
```
---

## Logging and Reports

| Log Type   | Path                                         |
|------------|----------------------------------------------|
| Discovery  | `/var/log/inanna_discover_<minion>.txt`      |
| Alive Hosts| `/var/log/inanna_alive_mgmt_<timestamp>.txt` |
| Patching   | `/var/log/inanna_patching.log`               |
| Reboot     | `/var/log/inanna_reboot.log`                 |
| Validation | `/var/log/inanna_validation.log`             |
| Cleanup    | `/var/log/inanna_cleanup.log`                |

---

## Sample Outputs

Discovery Alive Hosts File:

```bash
192.168.1.10
192.168.1.11
192.168.2.20
```

## Discovery Alive Hosts File:

```bash
[2025-08-13 14:00:12] lab01: Service sshd is running
[2025-08-13 14:00:13] lab01: Service nginx is NOT running
```

## Snapshot Name Example:

```bash
`inanna_prepatch_20250730_1425`
```