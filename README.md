# Inanna

**Enterprise-grade Salt automation for discovery, patching, validation, and rollback — with production-ready error handling and comprehensive logging.**

---

## What Inanna Does

Inanna is a Salt-based automation framework that provides a complete lifecycle for infrastructure patching and maintenance:

1. **Discover** our infrastructure (Salt-first + Nmap network scanning)
2. **Protect** with VMware snapshots before changes
3. **Patch** operating systems and applications with selective filtering
4. **Reboot** intelligently (change-aware, maintenance windows, health checks)
5. **Validate** critical services and configurations
6. **Rollback** automatically if validation fails
7. **Clean up** old logs and snapshots
8. **Collect** centralized reports on the Salt master

### Key Features

- ** Concurrency Protection**: Advisory locking prevents simultaneous runs
- ** Selective Patching**: Whitelist/blacklist packages, wildcard patterns, package groups
- ** Intelligent Retry**: Automatic retry with backoff for network and package operations
- ** Timeout Protection**: Configurable timeouts prevent hung operations
- ** Pre-flight Checks**: Disk space, system load, active users, maintenance windows
- ** Rich Logging**: Human-readable logs + machine-readable JSON metrics
- ** Failure Recovery**: Continue-on-error options, automatic rollback triggers
- ** Orchestration**: Batch deployment with canary support
- ** Verification**: Port binding, dependency checks, HTTP health checks, config validation

---

## Table of Contents

- [Before You Start](#before-you-start)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Pillar Configuration](#pillar-configuration)
- [Module Reference](#module-reference)
- [Common Workflows](#common-workflows)
- [Advanced Features](#advanced-features)
- [Troubleshooting](#troubleshooting)
- [Testing Framework](#testing-framework)
- [FAQ](#faq)

---

## Before You Start

### Prerequisites

1. **Salt Master & Minions** (or Salt-SSH setup)
2. **Python 3.7+** with PyYAML
3. **Network Requirements**:
   - Minions can reach master
   - Scanner minions can reach target networks
   - (Optional) vCenter/ESXi for snapshots
4. **Disk Space**: 500MB+ free on `/var/log/inanna`
5. **Permissions**: Root or sudo access on managed hosts

### One-Time Master Setup

Enable file receiving on the Salt master:

```bash
# Edit /etc/salt/master
vim /etc/salt/master
```

```yaml
file_recv: True
file_recv_max_size: 500MB
```

```bash
# Restart Salt services
sudo systemctl restart salt-master
# If the above doesn't work, reboot may be required
```

---

## Project Structure

```
bcds/inanna/
├── init.sls                    # Main orchestration entry point
├── discover.sls                # Salt-first inventory + Nmap scanning
├── roster_build.sls            # Generate Salt-SSH roster
├── snapshot.sls                # VMware pre-patch snapshots
├── patch_os.sls                # OS-level patching
├── patch_apps.sls              # Application patching
├── reboot.sls                  # Intelligent, change-aware reboot
├── validate.sls                # Service & health validation
├── cleanup.sls                 # Log and snapshot cleanup
├── rollback.sls                # VMware snapshot rollback
├── lock.sls                    # Concurrency control
├── collect_master.sls          # Master-side log aggregation
└── orch_inanna.sls             # Batch orchestration runner

pillar/
├── top.sls                     # Standard Salt pillar targeting
└── inanna.sls                  # Central Inanna configuration
```

---

## Quick Start

### 1. Initial Setup

```bash
# Verify Salt connectivity
salt-key -L
salt '*' test.ping

# Check pillar data
salt '*' pillar.get inanna
```

### 2. Configure Pillar

Edit `/srv/pillar/inanna.sls` (see [Pillar Configuration](#pillar-configuration) section):

```yaml
inanna:
  dry_run: true                          # Start with dry-run
  skip_snapshot: true                    # Skip snapshots for testing
  auto_reboot: false                     # Disable reboots initially
  
  discovery:
    scanners: [manager]                  # Our Salt master minion ID
    scan_ranges: ['192.168.100.0/24']    # Our network(s)
  
  validation:
    critical_services: [sshd, nginx]     # Services to verify
```

### 3. First Discovery Run

```bash
# Run discovery (safe, read-only)
salt '*' state.apply bcds.inanna.discover

# Collect results to master
salt 'manager' state.apply bcds.inanna.collect_master

# View results
ls -l /var/log/inanna/hosts/
cat /var/log/inanna/inanna_master_alive_mgmt.txt
```

### 4. Dry-Run Full Workflow

```bash
# Test complete workflow without changes
salt '*' state.apply bcds.inanna pillar='{"inanna":{"dry_run":true}}'

# Check logs
tail -f /var/log/inanna/*_latest.log
```

### 5. Production Run

```bash
# When ready, run for real
salt '*' state.apply bcds.inanna pillar='{
  "inanna":{
    "dry_run":false,
    "skip_snapshot":false,
    "auto_reboot":true
  }
}'
```

---

## Pillar Configuration

### Essential Configuration

```yaml
inanna:
  ###########################################################################
  # 1) Execution Control
  ###########################################################################
  dry_run: false                  # true = test mode (no changes)
  skip_snapshot: false            # true = skip VMware snapshots
  auto_reboot: true               # true = allow reboots after patching
  
  ###########################################################################
  # 2) Patching Modes
  ###########################################################################
  update_mode: security           # 'security' | 'all'
  
  ###########################################################################
  # 3) Discovery Configuration
  ###########################################################################
  discovery:
    scanners: [manager]           # Minions that run Nmap
    scan_ranges:                  # Networks to scan
      - '192.168.100.0/24'
    port_list: [22, 5985, 3389]   # Ports for agentless detection
    
    # Error handling
    scan_timeout: 300             # Seconds per Nmap scan
    scan_retry_attempts: 2        # Retry failed scans
    verify_discovery: true        # Validate results
    min_hosts_threshold: 1        # Warn if below this count
  
  ###########################################################################
  # 4) Targeting (Include/Exclude Hosts)
  ###########################################################################
  targeting:
    whitelist: []                 # Only patch these (empty = all)
    blacklist: []                 # Never patch these
  
  ###########################################################################
  # 5) VMware Snapshots
  ###########################################################################
  vmware:
    host: vcenter.example.com
    user: administrator@vsphere.local
    password: USE_SECRET_STORE     # Use external pillar!
    protocol: https
    port: 443
  
  vm_names:
    - WebServer01
    - DBServer01
  
  snapshot:
    name_prefix: inanna_prepatch_
    timeout: 600                  # Seconds per snapshot
    retry_attempts: 3             # Retry failed snapshots
    continue_on_error: true       # Don't stop on single failure
  
  ###########################################################################
  # 6) OS Patching Configuration
  ###########################################################################
  patching:
    update_timeout: 2700          # 45 minutes (industry standard)
    refresh_retry_attempts: 3
    min_disk_space_mb: 500        # Require 500MB free space
    verify_updates: true
    continue_on_partial_failure: true
  
  ###########################################################################
  # 7) Application Patching
  ###########################################################################
  apps:
    pkgs:
      RedHat: [nginx, docker-ce]
      Debian: [nginx, docker.io]
    snaps: [lxd, kubectl]
    restart_services: [nginx, docker]
    
    # Error handling
    package_timeout: 1800         # 30 minutes per package
    package_retry_attempts: 3
    verify_services: true         # Check services started
  
  ###########################################################################
  # 8) Reboot Configuration
  ###########################################################################
  reboot:
    delay: 5                      # Seconds before reboot
    max_load_threshold: 10.0      # Defer if load > this
    check_active_users: true      # Check for logged-in users
    max_active_users: 0           # Don't reboot if users present
    require_kernel_change: false  # Only reboot for kernel updates
    
    maintenance_window:
      enabled: false              # Restrict reboots to window
      start_hour: 22              # 10 PM
      end_hour: 6                 # 6 AM
  
  ###########################################################################
  # 9) Validation Configuration
  ###########################################################################
  validation:
    critical_services: [sshd, nginx]
    warning_services: [redis]     # Log only, don't fail
    mode: strict                  # 'strict' | 'permissive'
    
    # Advanced validation
    check_ports: true             # Verify port bindings
    check_dependencies: true      # Verify service dependencies
    retry_attempts: 3
    stabilization_delay: 30       # Wait after recent boot
    
    # Health checks
    http_checks:
      nginx_health:
        url: http://localhost/health
        expected_code: 200
        timeout: 5
  
  ###########################################################################
  # 10) Cleanup & Retention
  ###########################################################################
  cleanup:
    retention_days: 30            # Keep logs/snapshots this long
    cleanup_snapshots: true       # Clean old VMware snapshots
    snapshot_retention_days: 7    # Keep snapshots this long
  
  ###########################################################################
  # 11) Concurrency Control
  ###########################################################################
  lock:
    enabled: true                 # Prevent simultaneous runs
    max_lock_age_hours: 24        # Auto-remove stale locks
  
  ###########################################################################
  # 12) Selective Patching (Advanced)
  ###########################################################################
  selective_patching:
    enabled: false
    package_filter:
      mode: exclude               # 'include' | 'exclude' | 'all'
      exclude_packages:
        - kernel*                 # Don't auto-patch kernel
        - docker*
      use_wildcards: true
      case_sensitive: false
    
    package_groups:
      enabled: false
      groups:
        critical_security:
          - kernel
          - openssl
          - openssh
      active_groups: [critical_security]
```

### Orchestration Configuration

```yaml
# Separate from inanna config - used for salt-run orchestrate
orch:
  batch_size: 1                   # Hosts per batch
  batch_delay: 300                # 5 minutes between batches
  fail_fast: true                 # Stop on first failure
  verify_between_batches: true    # Run validation between batches
  canary_batch: false             # First batch as canary
  
  # Custom batch definitions
  batch_definitions:
    - name: canary
      target: web01
      target_type: glob
    - name: production
      target: web0[2-5]
      target_type: glob
```

---

## Module Reference

### init.sls - Main Entry Point

**Purpose**: Orchestrates the complete Inanna workflow with locking.

**Sequence**:
1. Acquire lock (prevent concurrent runs)
2. Discovery (Salt inventory + Nmap scanning)
3. Collect results to master
4. Build Salt-SSH roster (optional)
5. Snapshots (if enabled)
6. OS patching
7. Application patching
8. Reboot (if needed and allowed)
9. Validation
10. Cleanup (always runs)
11. Release lock (always runs)

**Usage**:
```bash
# Standard run
salt '*' state.apply bcds.inanna

# With pillar overrides
salt '*' state.apply bcds.inanna pillar='{
  "inanna":{
    "dry_run":true,
    "update_mode":"all"
  }
}'
```

---

### discover.sls - Infrastructure Discovery

**Purpose**: Salt-first inventory + network discovery for agentless hosts.

**Process**:
1. Enumerate Salt minions (on any host with salt.cmd)
2. Run Nmap from designated scanner minions
3. Detect managed and agentless hosts
4. Generate discovery reports
5. Push results to master via cp.push

**Key Features**:
- Scanner gating (only designated minions run Nmap)
- Timeout protection (configurable scan timeout)
- Retry logic (failed scans retry automatically)
- Discovery verification (checks against thresholds)
- Change detection (compares to previous run)

**Outputs**:
```
/var/log/inanna/inanna_alive_mgmt_TIMESTAMP.txt      # Managed hosts
/var/log/inanna/inanna_alive_agentless_TIMESTAMP.txt # Agentless hosts
/var/log/inanna/inanna_discovery_TIMESTAMP.log       # Discovery log
```

**Master Collection**:
```
/var/log/inanna/inanna_master_alive_mgmt.txt         # Aggregated managed
/var/log/inanna/inanna_master_alive_agentless.txt    # Aggregated agentless
```

---

### roster_build.sls - Salt-SSH Roster Generation

**Purpose**: Generate `/etc/salt/roster` from discovered agentless hosts.

**Features**:
- Automatic backup of existing roster
- Deduplication of entries
- Format validation (YAML syntax check)
- Optional connectivity testing
- Preserves custom entries (if overwrite=false)

**Usage**:
```bash
salt 'manager' state.apply bcds.inanna.roster_build
```

---

### snapshot.sls - VMware Snapshots

**Purpose**: Create pre-patch VMware snapshots for rollback capability.

**Key Features**:
- Explicit gate (must set skip_snapshot: true/false)
- Per-VM timeout and retry
- Continue-on-error option
- Comprehensive logging
- Snapshot name tracking

**Configuration**:
```yaml
inanna:
  skip_snapshot: false            # Must be explicitly set
  snapshot:
    timeout: 600                  # 10 minutes per VM
    retry_attempts: 3
    continue_on_error: true       # Don't fail entire run if one VM fails
```

---

### patch_os.sls - Operating System Patching

**Purpose**: Apply OS-level updates with selective filtering and error handling.

**Key Features**:
- Security-only or all updates
- Selective patching (whitelist/blacklist)
- Package groups support
- Wildcard pattern matching
- Pre-flight checks (disk space, repo connectivity)
- Timeout and retry protection
- Update verification
- Change tracking

**Selective Patching Examples**:

```yaml
# Example 1: Whitelist specific packages
selective_patching:
  enabled: true
  package_filter:
    mode: include
    include_packages: [kernel, openssl, openssh*, systemd]
    use_wildcards: true

# Example 2: Blacklist packages
selective_patching:
  enabled: true
  package_filter:
    mode: exclude
    exclude_packages: [kernel*, docker*]

# Example 3: Package groups
selective_patching:
  enabled: true
  package_groups:
    enabled: true
    groups:
      critical_security: [kernel, openssl, openssh, glibc]
      web_stack: [nginx, apache2, php*]
    active_groups: [critical_security]
```

**Outputs**:
```
/var/log/inanna/inanna_patching_TIMESTAMP.log
/var/log/inanna/inanna_changed_patch_os.stamp      # Change indicator
```

---

### patch_apps.sls - Application Patching

**Purpose**: Update application packages and snaps with service management.

**Key Features**:
- OS-specific package lists
- Snap package refresh
- Service restart coordination
- Selective filtering (same as OS patching)
- Service verification
- Timeout and retry protection

**Configuration**:
```yaml
inanna:
  apps:
    pkgs:
      RedHat: [nginx, docker-ce, postgresql13]
      Debian: [nginx, docker.io, postgresql]
    snaps: [lxd, kubectl, helm]
    restart_services: [nginx, docker, postgresql]
    
    package_timeout: 1800
    verify_services: true
```

**Outputs**:
```
/var/log/inanna/inanna_apps_TIMESTAMP.log
/var/log/inanna/inanna_changed_patch_apps.stamp    # Change indicator
```

---

### reboot.sls - Intelligent Reboot

**Purpose**: Change-aware reboot with safety checks and maintenance windows.

**Key Features**:
- Only reboots if patches actually changed something
- Pre-reboot checks (system load, active users, maintenance window)
- Stamp age validation (don't act on old changes)
- Kernel change detection
- Reboot marker creation
- What-changed summary in logs

**Safety Checks**:
1. System load threshold
2. Active user count
3. Maintenance window compliance
4. Stamp file freshness
5. Kernel/init change detection (optional)

**Configuration**:
```yaml
inanna:
  auto_reboot: true
  reboot:
    max_load_threshold: 10.0
    check_active_users: true
    max_active_users: 0
    stamp_max_age_minutes: 180
    
    maintenance_window:
      enabled: true
      start_hour: 22            # 10 PM
      end_hour: 6               # 6 AM
```

**Outputs**:
```
/var/log/inanna/inanna_reboot_TIMESTAMP.log
/var/log/inanna/inanna_last_reboot.marker
```

---

### validate.sls - Service Validation

**Purpose**: Verify critical services are running with comprehensive health checks.

**Key Features**:
- Strict vs. permissive modes
- Retry logic for service checks
- Port binding verification
- Dependency checking
- Config validation (nginx, apache, sshd, postgres)
- HTTP health checks
- TCP connectivity checks
- Post-reboot stabilization delay
- Machine-readable JSON output

**Validation Modes**:
- **Strict**: Fail state if any critical service is down
- **Permissive**: Log warnings but don't fail state

**Configuration**:
```yaml
inanna:
  validation:
    critical_services: [sshd, nginx, postgresql]
    warning_services: [redis, memcached]    # Log only
    mode: strict
    
    # Advanced checks
    check_ports: true
    check_dependencies: true
    retry_attempts: 3
    stabilization_delay: 30
    
    # HTTP health checks
    http_checks:
      nginx_health:
        url: http://localhost/health
        expected_code: 200
        timeout: 5
    
    # TCP checks
    tcp_checks:
      postgres:
        host: localhost
        port: 5432
        timeout: 5
```

**Outputs**:
```
/var/log/inanna/inanna_validate_TIMESTAMP.log       # Human-readable
/var/log/inanna/inanna_validate_TIMESTAMP.json      # Machine-readable
```

---

### cleanup.sls - Log and Snapshot Cleanup

**Purpose**: Prune old logs, reports, and VMware snapshots.

**Key Features**:
- Configurable retention periods
- Safe path guards (won't delete outside allowed_log_root)
- Log collection retry
- Collection verification
- VMware snapshot pruning
- Temp file cleanup
- Cleanup summary and metrics

**Configuration**:
```yaml
inanna:
  cleanup:
    retention_days: 30
    cleanup_snapshots: true
    snapshot_retention_days: 7
    collect_retry_attempts: 3
```

**Outputs**:
```
/var/log/inanna/inanna_cleanup_TIMESTAMP.log
```

---

### rollback.sls - VMware Snapshot Rollback

**Purpose**: Revert VMs to previous snapshots with safety measures.

**Key Features**:
- Explicit confirmation requirement
- Safety snapshot before rollback
- Auto-select latest snapshot option
- Boot wait and verification
- Post-rollback validation
- Comprehensive logging

**Usage**:
```bash
# Dry-run
salt 'minion' state.apply bcds.inanna.rollback pillar='{
  "inanna":{
    "dry_run":true,
    "rollback":{"snapshot_name":"inanna_prepatch_20250107_1030"}
  }
}'

# Real rollback (requires confirmation)
salt 'minion' state.apply bcds.inanna.rollback pillar='{
  "inanna":{
    "rollback":{
      "snapshot_name":"inanna_prepatch_20250107_1030",
      "require_confirmation":false
    }
  }
}'

# Auto-select latest snapshot
salt 'minion' state.apply bcds.inanna.rollback pillar='{
  "inanna":{
    "rollback":{
      "auto_select":true,
      "require_confirmation":false
    }
  }
}'
```

---

### lock.sls - Concurrency Control

**Purpose**: Advisory locking to prevent simultaneous Inanna runs.

**Actions**:
- `acquire`: Create lock file with PID and metadata
- `release`: Remove lock file
- `status`: Check lock status
- `force-release`: Emergency lock removal

**Features**:
- Automatic stale lock detection (process not running)
- Age-based stale lock removal
- Lock metadata (PID, timestamp, user, Salt job ID)

**Usage**:
```bash
# Check lock status
salt-call state.apply bcds.inanna.lock pillar='{
  "inanna":{"lock":{"action":"status"}}
}'

# Force release stuck lock (emergency only!)
salt-call state.apply bcds.inanna.lock pillar='{
  "inanna":{"lock":{"action":"force-release"}}
}'
```

---

### collect_master.sls - Master-Side Aggregation

**Purpose**: Collect and aggregate discovery files from all minions on the master.

**Process**:
1. Verify running on Salt master
2. Find discovery files in master cache
3. Validate file freshness and format
4. Aggregate into master reports
5. Deduplicate entries
6. Generate collection metrics

**Key Features**:
- Staleness detection
- Format validation
- Deduplication
- Collection health metrics
- Reporting minion tracking

**Usage**:
```bash
# Run on master
salt 'manager' state.apply bcds.inanna.collect_master
```

**Outputs**:
```
/var/log/inanna/inanna_master_alive_mgmt.txt
/var/log/inanna/inanna_master_alive_agentless.txt
/var/log/inanna/inanna_collection_TIMESTAMP.log
/var/log/inanna/inanna_collection_metrics.json
```

---

### orch_inanna.sls - Batch Orchestration

**Purpose**: Orchestrate Inanna across multiple hosts in controlled batches.

**Features**:
- Custom batch definitions
- Canary deployment support
- Configurable delays between batches
- Fail-fast option
- Health checks between batches
- Batch-level timeouts
- Comprehensive orchestration logging

**Usage**:
```bash
# Simple batch orchestration
salt-run state.orchestrate bcds.orch_inanna pillar='{
  "orch":{
    "batch_size":2,
    "batch_delay":300,
    "fail_fast":true
  }
}'

# Canary deployment
salt-run state.orchestrate bcds.orch_inanna pillar='{
  "orch":{
    "canary_batch":true,
    "batch_definitions":[
      {"name":"canary","target":"web01","target_type":"glob"},
      {"name":"production","target":"web0[2-9]","target_type":"glob"}
    ],
    "canary_delay":600,
    "fail_fast":true
  }
}'
```

---

## Common Workflows

### Workflow 1: Discovery Only

```bash
# 1. Run discovery
salt '*' state.apply bcds.inanna.discover

# 2. Collect to master
salt 'manager' state.apply bcds.inanna.collect_master

# 3. Review results
cat /var/log/inanna/inanna_master_alive_mgmt.txt
cat /var/log/inanna/inanna_master_alive_agentless.txt

# 4. Build roster (optional)
salt 'manager' state.apply bcds.inanna.roster_build
cat /etc/salt/roster
```

---

### Workflow 2: Security Updates Only (Dry-Run)

```bash
salt '*' state.apply bcds.inanna pillar='{
  "inanna":{
    "dry_run":true,
    "update_mode":"security",
    "skip_snapshot":true,
    "auto_reboot":false
  }
}'
```

---

### Workflow 3: Full Production Run

```bash
# Run complete workflow with snapshots and reboot
salt '*' state.apply bcds.inanna pillar='{
  "inanna":{
    "dry_run":false,
    "skip_snapshot":false,
    "auto_reboot":true,
    "update_mode":"security"
  }
}'

# Monitor progress
tail -f /var/log/inanna/*_latest.log

# Collect logs to master
salt 'manager' state.apply bcds.inanna.collect_master

# Review results on master
ls -l /var/log/inanna/hosts/
```

---

### Workflow 4: Selective Patching (Critical Only)

```bash
salt '*' state.apply bcds.inanna.patch_os pillar='{
  "inanna":{
    "selective_patching":{
      "enabled":true,
      "package_groups":{
        "enabled":true,
        "groups":{
          "critical":["kernel","openssl","openssh","glibc","systemd"]
        },
        "active_groups":["critical"]
      }
    }
  }
}'
```

---

### Workflow 5: Canary Deployment

```bash
salt-run state.orchestrate bcds.orch_inanna pillar='{
  "orch":{
    "canary_batch":true,
    "batch_definitions":[
      {
        "name":"canary",
        "target":"web01",
        "target_type":"glob"
      },
      {
        "name":"batch-1",
        "target":"web0[2-5]",
        "target_type":"glob"
      },
      {
        "name":"batch-2",
        "target":"web0[6-9]",
        "target_type":"glob"
      }
    ],
    "canary_delay":600,
    "batch_delay":300,
    "fail_fast":true,
    "verify_between_batches":true
  }
}'
```

---

### Workflow 6: Rollback After Failed Validation

```bash
# Check validation results
cat /var/log/inanna/inanna_validate_latest.log
cat /var/log/inanna/inanna_validate_latest.json

# Find snapshot name
ls -l /var/log/inanna/snapshot_latest.log

# Perform rollback
salt 'minion' state.apply bcds.inanna.rollback pillar='{
  "inanna":{
    "rollback":{
      "snapshot_name":"inanna_prepatch_20250107_1030",
      "require_confirmation":false,
      "validate_after_rollback":true
    }
  }
}'
```

---

## Advanced Features

### Selective Patching

**Whitelist Mode** (only patch specific packages):
```yaml
inanna:
  selective_patching:
    enabled: true
    package_filter:
      mode: include
      include_packages:
        - kernel
        - openssl
        - openssh*          # Wildcards supported
        - systemd
```

**Blacklist Mode** (exclude specific packages):
```yaml
inanna:
  selective_patching:
    enabled: true
    package_filter:
      mode: exclude
      exclude_packages:
        - kernel*           # Don't auto-patch kernel
        - docker*
        - postgresql*
```

**Package Groups**:
```yaml
inanna:
  selective_patching:
    enabled: true
    package_filter:
      mode: include
    package_groups:
      enabled: true
      groups:
        critical_security:
          - kernel
          - openssl
          - openssh
          - glibc
          - systemd
        web_stack:
          - nginx
          - apache2
          - php*
        database:
          - postgresql*
          - mysql*
          - mariadb*
      active_groups:
        - critical_security
        - web_stack
```

---

### Maintenance Windows

Restrict reboots to specific time windows:

```yaml
inanna:
  reboot:
    maintenance_window:
      enabled: true
      start_hour: 22        # 10 PM
      end_hour: 6           # 6 AM (overnight window)

# Supports windows that cross midnight
# Example: 22:00 - 06:00 = overnight maintenance
```

---

### HTTP Health Checks

Add HTTP endpoint checks to validation:

```yaml
inanna:
  validation:
    http_checks:
      nginx_health:
        url: http://localhost/health
        expected_code: 200
        timeout: 5
      
      api_status:
        url: http://localhost:8080/api/status
        expected_code: 200
        timeout: 10
```

---

### Batch Orchestration with Custom Targeting

```yaml
orch:
  batch_definitions:
    - name: canary-test
      target: test-minion-01
      target_type: glob
    
    - name: web-servers-east
      target: role:web AND dc:us-east
      target_type: compound
    
    - name: web-servers-west
      target: role:web AND dc:us-west
      target_type: compound
    
    - name: database-servers
      target: role:database
      target_type: grain
  
  canary_batch: true
  canary_delay: 600           # 10 minutes extra after canary
  batch_delay: 300            # 5 minutes between other batches
  fail_fast: true
  verify_between_batches: true
```

---

### Automatic Rollback on Validation Failure

```yaml
inanna:
  validation:
    mode: strict
    trigger_rollback_on_failure: true
    critical_services: [sshd, nginx, postgresql]
```

If validation fails in strict mode, automatically trigger rollback to pre-patch snapshot.

---

## Troubleshooting

### Issue: cp.push not collecting logs to master

**Symptoms**: Logs not appearing in `/var/log/inanna/hosts/` on master

**Solution**:
```bash
# On master: verify file_recv is enabled
grep file_recv /etc/salt/master

# Should show:
# file_recv: True

# If not, add it and restart
sudo systemctl restart salt-master

# Test manually
salt 'minion' cmd.run 'echo "test" > /tmp/test.txt'
salt 'minion' cp.push /tmp/test.txt

# Check master cache
ls -l /var/cache/salt/master/minions/*/files/tmp/
```

---

### Issue: Nmap failed or not found

**Symptoms**: Discovery log shows "Nmap not found" or DNS errors

**Solution**:
```bash
# Option 1: Install nmap on scanner minions
salt 'manager' pkg.install nmap

# Option 2: Disable auto-install, fix manually
# In pillar:
inanna:
  discovery:
    install_nmap: false    # Don't try to auto-install

# Option 3: If DNS is broken, fix networking first
salt 'manager' cmd.run 'ping -c 1 google.com'
```

---

### Issue: Discovery not finding expected hosts

**Symptoms**: Empty or incomplete discovery results

**Diagnosis**:
```bash
# 1. Check scanner configuration
salt 'manager' pillar.get inanna:discovery:scanners

# 2. Verify scan ranges
salt 'manager' pillar.get inanna:discovery:scan_ranges

# 3. Check if threshold is too high
salt 'manager' pillar.get inanna:discovery:min_hosts_threshold

# 4. Review discovery log for errors
tail -100 /var/log/inanna/inanna_discovery_latest.log

# 5. Test nmap manually on scanner
salt 'manager' cmd.run 'nmap -sn 192.168.100.0/24'
```

---

### Issue: Validation failures after patching

**Symptoms**: validate.sls fails in strict mode

**Diagnosis**:
```bash
# 1. Check validation log
cat /var/log/inanna/inanna_validate_latest.log

# 2. Check JSON for details
cat /var/log/inanna/inanna_validate_latest.json | jq '.'

# 3. Check service status manually
systemctl status nginx
journalctl -u nginx -n 50

# 4. Verify port bindings
ss -tunlp | grep :80
```

**Solutions**:
```yaml
# Option 1: Switch to permissive mode temporarily
inanna:
  validation:
    mode: permissive

# Option 2: Move problematic service to warning_services
inanna:
  validation:
    critical_services: [sshd]
    warning_services: [nginx]    # Just warn, don't fail

# Option 3: Increase retry attempts
inanna:
  validation:
    retry_attempts: 5
    retry_interval: 15

# Option 4: Increase stabilization delay
inanna:
  validation:
    stabilization_delay: 60      # Wait 60s after reboot
```

---

### Issue: Lock prevents runs

**Symptoms**: "Inanna is already running" error

**Diagnosis**:
```bash
# Check lock status
salt-call state.apply bcds.inanna.lock pillar='{
  "inanna":{"lock":{"action":"status"}}
}'

# Check if process is actually running
ps aux | grep -i inanna
ps aux | grep -i salt
```

**Solution**:
```bash
# If process is not actually running (stale lock):
salt-call state.apply bcds.inanna.lock pillar='{
  "inanna":{"lock":{"action":"force-release"}}
}'

# Or wait for automatic stale lock removal (24 hours by default)
```

---

### Issue: Rollback fails or times out

**Symptoms**: Rollback doesn't complete or VM doesn't boot

**Diagnosis**:
```bash
# 1. Check rollback log
cat /var/log/inanna/rollback_latest.log

# 2. Verify snapshot exists in vCenter
# (Check vSphere UI or use govc)

# 3. Check network connectivity to vCenter
salt 'minion' cmd.run 'ping -c 3 vcenter.example.com'
```

**Solutions**:
```yaml
# Increase timeouts
inanna:
  rollback:
    boot_timeout: 600         # 10 minutes
  snapshot:
    timeout: 900              # 15 minutes

# Skip boot wait if network boot is slow
inanna:
  rollback:
    wait_for_boot: false
    verify_revert: false
```

---

### Issue: Selective patching not filtering correctly

**Symptoms**: Wrong packages being patched or excluded

**Diagnosis**:
```bash
# Check pillar rendering
salt 'minion' pillar.get inanna:selective_patching

# Check filter report in log
grep -A 20 "Selective Patching Summary" \
  /var/log/inanna/inanna_patching_latest.log
```

**Common Mistakes**:
```yaml
#  WRONG: Wildcards disabled but using wildcards
inanna:
  selective_patching:
    package_filter:
      include_packages: [nginx*]
      use_wildcards: false      # <-- Should be true!

#  CORRECT
inanna:
  selective_patching:
    package_filter:
      include_packages: [nginx*]
      use_wildcards: true

#  WRONG: Case sensitive but wrong case
inanna:
  selective_patching:
    package_filter:
      include_packages: [NGINX]
      case_sensitive: true      # Package is lowercase "nginx"

#  CORRECT
inanna:
  selective_patching:
    package_filter:
      include_packages: [nginx]
      case_sensitive: false     # Or use lowercase
```

---

## Testing Framework

Inanna includes comprehensive test scenarios in the project.

### Test Files

All test files are located in `/mnt/project/` with pattern `*-test*.yaml`:

1. **blacklist_test.sls.yaml** - Package exclusion testing
2. **canary_development-test.sls.yaml** - Canary deployment
3. **case_sensitivity-test.sls.yaml** - Case-sensitive filtering
4. **dryrun.sls.yaml** - Dry-run mode testing
5. **failure_recovery-test.sls.yaml** - Error handling
6. **lock_prevention-test.sls.yaml** - Concurrency control
7. **orchestration-test.sls.yaml** - Batch orchestration
8. **package_groups_sls.yaml** - Package group filtering
9. **rollback-testing_sls.yaml** - Snapshot rollback
10. **selective_patching-test.sls.yaml** - Whitelist filtering
11. **wildcard_pattern_matching-test.sls.yaml** - Wildcard patterns
12. **workflow_full-test.sls.yaml** - Complete workflow

### Running Tests

```bash
# Run individual test
salt 'test-minion-01' state.apply bcds.inanna.patch_os \
  pillar='{"inanna":{"selective_patching":{"enabled":true,...}}}'

# See test files for complete examples
cat /mnt/project/*-test*.yaml
```

---

## FAQ

### Q: Why Salt-first?

**A:** We trust Salt's own minion inventory before anything else. Nmap widens the net to catch non-minion devices (shadow IT, network gear, etc.).

---

### Q: Do I need Nmap everywhere?

**A:** No. Only scanner minions need nmap (typically our Salt master). Set `inanna.discovery.scanners` accordingly.

---

### Q: Where do logs live?

**A:** Locally on each minion under `/var/log/inanna/`. Copies are pushed to the master via cp.push and organized under `/var/log/inanna/hosts/<minion>/`.

---

### Q: Can I run everything from one command?

**A:** Yes:
```bash
# Standard run
salt '*' state.apply bcds.inanna

# Or via orchestration
salt-run state.orchestrate bcds.orch_inanna
```

---

### Q: How do I skip snapshots for testing?

**A:** Set `skip_snapshot: true` in pillar:
```yaml
inanna:
  skip_snapshot: true
```

Or override via CLI:
```bash
salt '*' state.apply bcds.inanna pillar='{"inanna":{"skip_snapshot":true}}'
```

---

### Q: Why didn't my host reboot?

**A:** Reboot only happens when:
1. `auto_reboot: true`
2. Change stamps exist (patches were applied)
3. Pre-reboot checks pass (load, users, maintenance window)

Check `/var/log/inanna/inanna_reboot_latest.log` for details.

---

### Q: How do I test without making changes?

**A:** Use dry-run mode:
```bash
salt '*' state.apply bcds.inanna pillar='{"inanna":{"dry_run":true}}'
```

---

### Q: Can I patch only security updates?

**A:** Yes:
```yaml
inanna:
  update_mode: security
```

Or via CLI:
```bash
salt '*' state.apply bcds.inanna pillar='{"inanna":{"update_mode":"security"}}'
```

---

### Q: How do I exclude kernel updates?

**A:** Use selective patching:
```yaml
inanna:
  selective_patching:
    enabled: true
    package_filter:
      mode: exclude
      exclude_packages: [kernel*]
```

---

### Q: Is there a machine-readable summary?

**A:** Yes. Validation writes JSON:
```bash
cat /var/log/inanna/inanna_validate_latest.json | jq '.'
```

Collection also writes metrics:
```bash
cat /var/log/inanna/inanna_collection_metrics.json | jq '.'
```

---

### Q: How do I roll back if something breaks?

**A:** Use rollback.sls:
```bash
# Find snapshot name
ls -l /var/log/inanna/snapshot_latest.log

# Rollback
salt 'minion' state.apply bcds.inanna.rollback pillar='{
  "inanna":{
    "rollback":{
      "snapshot_name":"inanna_prepatch_20250107_1030",
      "require_confirmation":false
    }
  }
}'
```

---

### Q: Can I patch in batches?

**A:** Yes, use orchestration:
```bash
salt-run state.orchestrate bcds.orch_inanna pillar='{
  "orch":{"batch_size":2,"batch_delay":300}
}'
```

---

### Q: How do I handle secrets (VMware passwords)?

**A:** Use external pillar (recommended):
- **sops** - Encrypted pillar files
- **vault** - HashiCorp Vault integration
- **git-crypt** - Encrypted git repository

Never commit plaintext passwords to version control.

---

## Best Practices

### 1. Start with Discovery Only

```bash
# Phase 1: Just discover
salt '*' state.apply bcds.inanna.discover
salt 'manager' state.apply bcds.inanna.collect_master

# Review results before proceeding
```

### 2. Use Dry-Run First

```bash
# Phase 2: Test workflow with dry-run
salt '*' state.apply bcds.inanna pillar='{"inanna":{"dry_run":true}}'
```

### 3. Test on Lab/Dev First

```bash
# Phase 3: Real run on non-production
salt 'lab*' state.apply bcds.inanna
```

### 4. Use Canary Deployments in Production

```bash
# Phase 4: Canary in production
salt-run state.orchestrate bcds.orch_inanna pillar='{
  "orch":{
    "canary_batch":true,
    "batch_definitions":[
      {"name":"canary","target":"web01",...},
      {"name":"production","target":"web0[2-9]",...}
    ]
  }
}'
```

### 5. Monitor Logs Actively

```bash
# Watch logs in real-time
tail -f /var/log/inanna/*_latest.log

# Check for errors
grep -i error /var/log/inanna/*.log
grep -i fail /var/log/inanna/*.log
```

### 6. Regular Log Cleanup

```yaml
inanna:
  cleanup:
    retention_days: 30              # Adjust based on compliance requirements
```

### 7. Use Selective Patching for Critical Systems

```yaml
inanna:
  selective_patching:
    enabled: true
    package_groups:
      enabled: true
      active_groups: [critical_security]
```

### 8. Configure Maintenance Windows for Production

```yaml
inanna:
  reboot:
    maintenance_window:
      enabled: true
      start_hour: 22
      end_hour: 6
```

### 9. Enable All Safety Checks

```yaml
inanna:
  reboot:
    check_active_users: true
    max_load_threshold: 5.0
  
  patching:
    min_disk_space_mb: 1000
    verify_updates: true
  
  validation:
    verify_services: true
    check_dependencies: true
```

### 10. Keep Snapshots Current

```yaml
inanna:
  skip_snapshot: false              # Always snapshot in production
  snapshot:
    continue_on_error: false        # Don't patch if snapshot fails
  
  cleanup:
    cleanup_snapshots: true
    snapshot_retention_days: 7      # Keep week of snapshots
```

---

## Security Considerations

### Secrets Management

**Never commit plaintext passwords**. Use external pillar:

```yaml
# /srv/pillar/inanna.sls (committed to git)
inanna:
  vmware:
    host: vcenter.example.com
    user: administrator@vsphere.local
    password: "{{ pillar.get('secret_vmware_pass') }}"

# /srv/pillar/secrets.sls (NOT in git, encrypted with sops/vault)
secret_vmware_pass: "actual_password_here"
```

### SSH Key Security

```bash
# Ensure proper permissions
chmod 600 /root/.ssh/id_rsa

# Use dedicated key for automation
ssh-keygen -t ed25519 -f /root/.ssh/inanna_id_ed25519 -C "inanna-automation"
```

### Lockdown Roster File

```bash
# Secure roster file
chmod 600 /etc/salt/roster
chown root:root /etc/salt/roster
```

### Audit Logging

All Inanna operations log to:
- Local: `/var/log/inanna/`
- Syslog: Tagged with `inanna-*`
- Master: `/var/log/inanna/hosts/<minion>/`

Configure central syslog for audit trail.

---

## Performance Tuning

### Large Environments (100+ hosts)

```yaml
# Increase timeouts
inanna:
  discovery:
    scan_timeout: 600             # 10 minutes per scan
  
  patching:
    update_timeout: 3600          # 1 hour for updates
  
  snapshot:
    timeout: 900                  # 15 minutes per VM

# Use batch orchestration
orch:
  batch_size: 5                   # Patch 5 at a time
  batch_delay: 600                # 10 minutes between batches
```

### Network-Constrained Environments

```yaml
inanna:
  discovery:
    scan_retry_attempts: 5        # More retries
  
  patching:
    refresh_retry_attempts: 5
  
  cleanup:
    collect_retry_attempts: 5
```

### Low-Resource Systems

```yaml
inanna:
  patching:
    min_disk_space_mb: 200        # Reduce if necessary
  
  reboot:
    max_load_threshold: 20.0      # Higher threshold
```

---

## Contributing

### Reporting Issues

1. Collect relevant logs:
```bash
tar czf inanna-logs.tar.gz /var/log/inanna/
```

2. Include:
   - Salt version: `salt --version`
   - OS details: `cat /etc/os-release`
   - Pillar config (sanitized)
   - Error messages from logs

### Development

```bash
# Test state rendering without running
salt-call --local state.show_sls bcds.inanna.init test=True

# Validate pillar
salt-call --local pillar.items | jq '.inanna'

# Check for Jinja errors
salt-call --local --log-level=warning --retcode-passthrough \
  state.show_sls bcds.inanna.init test=True
```

---

## License

[Our License Here]

---

## Support

- Documentation: This README
- Review Checklist: `Inanna_Review_checklist.md`
- Usage Requirements: `Inanna_Usage_reqs.md`
- Improvements Log: `Inanna_Improvements_14AUG2025.md`

---

## Version History

### Version 2.0 (January 2026)
- Enhanced error handling and retry logic
- Selective patching with package groups
- Concurrency control with advisory locking
- Maintenance window support
- Comprehensive validation with health checks
- Batch orchestration with canary deployment
- Automatic rollback triggers
- Master-side log aggregation with verification
- Timeout protection for all long-running operations
- Pre-flight checks (disk space, load, active users)
- Discovery verification and comparison
- Roster backup and syntax validation
- Service dependency checking
- Config validation (nginx, apache, sshd, postgres)
- HTTP and TCP health checks
- Stabilization delays after reboot
- Comprehensive metrics and reporting

### Version 1.0 (August 2025)
- Initial release
- Basic discovery, patching, validation workflow

---

**Built with Strength for reliable infrastructure automation**
