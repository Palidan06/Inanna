# Inanna – Deployment Checklist
*What to edit, where it lives, and how to verify it*

---

## 0) Paths & One‑Time Setup
- **States →** `/srv/salt/inanna/`: `init.sls`, `discover.sls`, `roster_build.sls`, `snapshot.sls`, `patch_os.sls`, `patch_apps.sls`, `reboot.sls`, `validate.sls`, `cleanup.sls`
- **Pillar →** `/srv/pillar/`: `top.sls`, `inanna.sls`
- **Salt master roots:** `/etc/salt/master` → `file_roots: /srv/salt`, `pillar_roots: /srv/pillar` (restart salt-master if changed)

## 1) Pillar edits (your environment)

### A) Discovery / IP spaces
- `inanna.discovery.scan_ranges` → **EDIT** to your CIDRs (e.g., `'10.0.10.0/24'`)
- `inanna.discovery.port_list` → **EDIT** ports (default: `22, 5985, 3389`)
- `inanna.discovery.paths.log_dir` → keep `/var/log/inanna` or set your standard

### B) Targeting
- `inanna.targeting.whitelist` → optional: restrict to these hosts/IPs
- `inanna.targeting.blacklist` → NEVER touch these hosts/IPs

### C) Salt‑SSH access & roster
- `inanna.ssh.roster.default_user` → **EDIT** SSH user (NOPASSWD sudo)
- `inanna.ssh.roster.ssh_port` → **EDIT** if not 22
- `inanna.ssh.roster.priv_key` → **EDIT** controller’s private key path
- `inanna.ssh.roster.output_path/name_prefix` → optional adjustments

### D) VMware (if using snapshots)
- `inanna.vmware.host/user/password/protocol/port` → **EDIT** (use secure pillar for password)
- `inanna.vm_names` → **EDIT** to exact vCenter/ESXi inventory names
- `inanna.snapshot.name_prefix` → keep or adjust

### E) Execution & patching
- `inanna.dry_run` (default false in prod; override per-run for tests)
- `inanna.skip_snapshot` (true to bypass snapshots)
- `inanna.auto_reboot` (true to allow reboots)
- `inanna.update_mode` → `'security'` or `'all'` (preferred; legacy: `os_update_mode`)

### F) Applications
- `inanna.apps.pkgs.RedHat / Debian` → **EDIT** package lists per OS family
- `inanna.apps.snaps` → optional snap names to refresh
- `inanna.apps.restart_services` → services to ensure/restart if pkgs changed

### G) Validation
- `inanna.validation.critical_services` → **EDIT** services that must be running
- `inanna.validation.mode` → `strict` or `permissive`
- `inanna.validation.enforce_start` → `true` to auto-start missing services

### H) Cleanup & safety rails
- `inanna.cleanup.retention_days` → days to keep logs/reports/snapshots
- `inanna.safety.allowed_log_root` → hard guard (default `/var/log/inanna`)

## 2) State files – what they read & verify
- **discover.sls** → reads `discovery.*`, writes in `/var/log/inanna`; verify `*_latest.txt` exists
- **roster_build.sls** → reads `ssh.roster.*`, `discovery.paths.*`; check `/etc/salt/roster`
- **snapshot.sls** → reads `vmware.*`, `vm_names`, `snapshot.*`; set `skip_snapshot` if unused
- **patch_os.sls** → `update_mode`, `targeting.*`, writes logs; Debian security‑only note
- **patch_apps.sls** → `apps.pkgs/snaps/restart_services`, `targeting.*`; logs in `/var/log/inanna`
- **reboot.sls** → `auto_reboot`, `dry_run`; logs with latest symlink
- **validate.sls** → `validation.critical_services/mode/enforce_start`; writes JSON + log
- **cleanup.sls** → `cleanup.retention_days`, `safety.allowed_log_root`; prunes logs/snapshots

## 3) First run – cookbook
- Pillar sanity: `salt '*' pillar.get inanna`
- Render: `salt-call --local state.show_sls inanna.init`
- Dry‑run pipeline: `salt 'lab-host' state.apply inanna.init test=True`
- Live run (single host): `salt 'lab-host' state.apply inanna.init pillar='{...}'`
- Check artifacts: `ls -l /var/log/inanna/*latest*`; `tail -n +1 /var/log/inanna/inanna_*_latest.log`

## 4) Quick “what to edit” cheat sheet
- IP subnets → `inanna.discovery.scan_ranges`
- Discovery ports → `inanna.discovery.port_list`
- Log path root → `inanna.discovery.paths.log_dir` (+ `safety.allowed_log_root`)
- Include/exclude hosts → `inanna.targeting.whitelist / blacklist`
- SSH roster → `inanna.ssh.roster.default_user/ssh_port/priv_key/output_path/name_prefix`
- vCenter/ESXi & creds → `inanna.vmware.*` (secure pillar)
- VM inventory names → `inanna.vm_names`
- Snapshot prefix → `inanna.snapshot.name_prefix`
- OS patching mode → `inanna.update_mode`
- App pkgs/snaps/restarts → `inanna.apps.*`
- Critical services → `inanna.validation.critical_services` (+ mode/enforce_start)
- Auto reboot → `inanna.auto_reboot`
- Retention → `inanna.cleanup.retention_days`

---

_Tip: Override per‑run values safely with `pillar=` on the CLI (e.g., `dry_run=false`, `update_mode=security`)._
