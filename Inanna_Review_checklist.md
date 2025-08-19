Inanna Full Review Playbook (File-by-File)

0) Pre-flight (one-time sanity checks)
- **Repo & names**
  - Confirm filename: you listed `patch_app.sls`. Decide now: use **`patch_apps.sls`** everywhere (recommended).
  - If you rename:
    - `git mv inanna/patch_app.sls inanna/patch_apps.sls`
    - Update any `include:`/`require:`/`state.apply` references in `init.sls`.

- **Salt mode**
  - Are you running **minions** or **salt-ssh** (or both)? Avoid minion-only modules in ssh paths.

- **Lint/Jinja**
  - `salt-call --local --log-level=warning --retcode-passthrough state.show_sls inanna.init test=True` to validate rendering.

1) pillar/top.sls
**Goal:** Ensure pillar data is assigned to the right targets and environments.

**Checklist**
- `base:` env exists and targets are correct.
- No stray whitespace/indent errors.

**Verify**
- `salt '*' pillar.items | grep -A2 inanna` (minions)
- `salt-ssh '*' pillar.items | grep -A2 inanna` (ssh)

**Common fixes**
- Confirm `/srv/pillar/top.sls` path and pillar_roots in master config.

2) pillar/inanna.sls
**Goal:** Every key used by states exists, has sane types, and safe defaults.

**Checklist (suggested keys & types)**
```bash
dry_run:            false
skip_snapshot:      false
auto_reboot:        true
os_update_mode:     'safe'
scan_ranges:
  - 10.0.0.0/24
discovery_ports:    [22, 3389]
targets:            []
exclude_hosts:      []
ssh_default_user:   'ec2-user'
critical_services:
  - sshd
vmware:
  host:             vcenter.example.com
  user:             '{{ pillar.get("secret_vmware_user") | default("changeme") }}'
  password:         '{{ pillar.get("secret_vmware_pass") | default("changeme") }}'
vm_names:           []
cleanup_retention_days: 30
```

**Verify**
- `salt-call --local pillar.items | jq '.inanna'`

3) init.sls
**Goal:** The run graph matches the lifecycle, filenames match, and dry-run flows through.

**Expected order**
```bash
1. discover
2. snapshot
3. patch_os
4. patch_apps
5. reboot
6. validate
7. cleanup
```
**Checklist**
- References match filenames.
- Requisites link stages.
- Honor dry-run.

**Verify**
- `salt-call --local state.show_lowstate inanna.init | less`

4) discover.sls
**Goal:** Produce a timestamped alive-hosts list in a predictable path with correct perms.

**Checklist**
- Inputs: `scan_ranges`, `discovery_ports`.
- Output file path predictable, timestamped.
- Ensure directory exists and perms set.

**Verify**
- `salt-call --local state.apply inanna.discover test=True`

5) snapshot.sls
**Goal:** Create pre-patch snapshots when enabled.

**Checklist**
- Respect `skip_snapshot`.
- Consistent snapshot naming.
- Log success/failure.

**Verify**
- Test both skip_snapshot true/false.

6) patch_os.sls
**Goal:** Apply OS updates consistently.

**Checklist**
- Honors `dry_run`, `os_update_mode`.
- Correct platform-specific modules.
- Target filtering works.

**Verify**
- `salt '*' grains.item os`
- `salt '*' state.apply inanna.patch_os test=True`

7) patch_apps.sls
**Goal:** Mirror OS patching patterns for applications.

**Checklist**
- Same guards as OS.
- Explicit app managers per platform.

**Verify**
- `salt '*' state.apply inanna.patch_apps test=True`

8) reboot.sls
**Goal:** Only reboot when changes occurred and allowed.

**Checklist**
- Guard with auto_reboot and change detection.
- Wait for node to return before continuing.

**Verify**
- Toggle auto_reboot true/false in tests.

9) validate.sls
**Goal:** Confirm critical_services are running.

**Checklist**
- Iterate `critical_services`.
- Log human-readable and JSON outputs.

**Verify**
- Stop a service to test failure path.

10) cleanup.sls
**Goal:** Prune old snapshots/logs/temp files.

**Checklist**
- Respect cleanup_retention_days.
- Safe path guards.

**Verify**
- Test with small retention days on mock files.

Cross-cutting audit
- Naming consistency across includes and calls.
- Requisites: prefer onchanges.
- Dry-run propagation.
- salt-ssh compatibility.
- Secrets remain in pillar.
- Idempotency: no-op on rerun without changes.

Smoke test run order
1. Pillar load check.
2. Render graph.
3. Discover (test).
4. Snapshot (test both ways).
5. Patch OS / Apps (test).
6. Reboot (test).
7. Validate (real on lab box).
8. Cleanup (test).