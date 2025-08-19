Changes to Inanna 14AUG2025

```bash
pillar/inanna.sls
```
What changed and why (high-impact improvements):
- Key normalization: scan_ranges (plural) is now canonical under inanna.discovery. States should read pillar.get('inanna', {}).get('discovery', {}).get('scan_ranges', []).
- Safer defaults: dry_run: false (production) with advice to override per-run; auto_reboot: true to keep fleets aligned after patching; update_mode: security to minimize risk.
- Stronger structure: Grouped sections (execution, patching, discovery, targeting, ssh, vmware, validation, cleanup). Clear paths under discovery.paths and cleanup to keep logs predictable.
- Compatibility retained: os_update_mode retained and pre-synced. States should prefer update_mode, then fall back to os_update_mode if needed.
- Validation behavior switch: validation.mode lets you choose strict vs permissive without editing code.
- Safety rails: Optional safety.enforce_types and cleanup.allowed_log_root give you a place to toggle hard-fail checks and guard cleanup paths.
- Secrets hygiene: Password field is a placeholder with explicit warning—move real credentials to external pillar or a secret backend (e.g., sops, vault, git-crypt + ext_pillar).

```bash
inanna/init.sls
```
Key improvements:
- This version enforces deterministic step-by-step order (via module.run: state.apply).
- It uses our the pillar structure (with backward-compat to your old keys).
- Honors the snapshot and reboot gates.
- Passes through dry-run cleanly.

```bash
inanna/discover.sls
```
Key improvements:
- Reads canonical pillars at inanna.discovery.* (with fallbacks to your current keys).
- Logs go to /var/log/inanna; creates directory if needed.
- Timestamped output files + a *_latest.txt symlink for downstream use.
- Package updates section works on RHEL/Debian (dnf or apt).
- Keeps your ICMP + TCP dual-scan approach unchanged.

```bash
inanna/roster_build.sls
```
Key improvements:
- Reads canonical pillars (inanna.ssh.roster, inanna.discovery.paths) with fallbacks to your current keys.
- Prefers the inanna_alive_mgmt_latest.txt symlink; otherwise falls back to newest match from the glob.
- Safely applies blacklist/exclude_hosts.
- Ensures /etc/salt/roster exists with correct 0640 perms.
- Keeps the managed block behavior unless overwrite: true.

```bash
inanna/snapshot.sls
```
Key improvements:
- Honors inanna.skip_snapshot (so running this state directly won’t create snapshots when disabled).
- Reads canonical pillars (inanna.vmware, inanna.vm_names) with optional config under inanna.snapshot.
- Uses a unique, timestamped snapshot name (prefix configurable).
- Avoids leaking secrets; writes a simple log in /var/log/inanna plus a snapshot_latest.log symlink.
- Uses module.run to call vmware.vm_snapshot_create reliably.

```bash
inanna/patch_os.sls
```
Key improvements:
- Whitelist/blacklist aware (canonical + legacy).
- Uses pkg.uptodate (idempotent) with security: True on RHEL-family.
- Debian: logs a note if security was requested, then performs standard updates.
- Logs to /var/log/inanna/inanna_patching_*.log plus inanna_patching_latest.log.
- Relies on init.sls for snapshot sequencing (no fragile marker files).
- Plays nicely with test=True from init.sls for dry-run previews.

```bash
inanna/patch_apps.sls
```
Key improvements:
- Uses canonical pillars under inanna.apps.* (with legacy fallbacks for targets/exclusions).
- Focuses on application package lists per OS family via pkg.latest (idempotent).
- Optional snap refreshes (only if snap exists and the snap is installed).
- Optional service restarts when app packages change.
- Logging to /var/log/inanna/inanna_apps_*.log + inanna_apps_latest.log.
- Snapshot gating is handled in init.sls (so no fragile local marker file).

```bash
inanna/reboot.sls
```
Key improvements:
- Dry-run aware: Honors inanna.dry_run. When true, it logs what would happen and doesn’t reboot.
- Canonical pillars & paths: Uses inanna.auto_reboot and inanna.discovery.paths.log_dir (defaults to /var/log/inanna), aligning with the rest of the stack.
- Structured logging: Writes to timestamped files inanna_reboot_YYYYMMDD_HHMM.log and maintains a stable inanna_reboot_latest.log symlink. (Replaces the old single /var/log/inanna_reboot.log.)
- Deterministic reboot call: Uses system.reboot with a short delay (in_seconds: 5) and a clear message, so it’s predictable and traceable.
- Clear skip vs. action logs: Whether it reboots, dry-runs, or skips (auto_reboot=false), you get an explicit log entry.
- Idempotent + safe ordering: Ensures the log directory exists first and keeps the state rerunnable without side effects.
- Project-standard default: auto_reboot defaults to true per our production baseline (you can override per-run or in pillar).

```bash
inanna/validate.sls
```
Key improvements:
- If you want this to start services automatically, `set inanna.validation.enforce_start: true`. In dry-run it won’t change anything.
- `mode: strict` makes the state fail when any listed service isn’t running; permissive only logs.
- Canonical pillars + fallback: Reads inanna.validation.critical_services (and mode, enforce_start) with legacy fallback to inanna:critical_services. Keeps your style but standardizes inputs.
- Strict vs. permissive: mode: strict fails the state if any listed service isn’t running; permissive only logs. Clear pass/fail semantics after patching.
- No surprise restarts: By default it doesn’t change service state. You can opt in with inanna.validation.enforce_start: true. Honors dry_run so nothing mutates in test mode.
- Clean logging structure: Timestamped logs under /var/log/inanna plus a stable inanna_validate_latest.log symlink—consistent with the rest of Inanna.
- Machine-readable output: Writes a JSON summary (inanna_validate_*.json) for dashboards/automation while also keeping human-friendly log lines.
- Idempotent and ordered: Ensures the log dir first; logs after any optional service enforcement; deterministic, re-runnable with no noise.
- Type safety rails: Optional guard that fails fast if critical_services isn’t a list, preventing Jinja/render surprises.
- Portable checks: Uses service.status to evaluate each item, so results are consistent across distros.
- Clear messages: Per-service lines like “Service X is running/NOT running” and a final strict/permissive result line make post-run triage quick.
- Aligned paths: Moves from a single flat /var/log/inanna_validation.log to timestamped files in /var/log/inanna, matching snapshot/patch logs.

```bash
inanna/validate.sls
```
Key improvements:
- Canonical pillars (inanna.cleanup.retention_days with fallback to cleanup_retention_days)
- Safe log pathing under /var/log/inanna (via pillar), with a per-run log + stable latest symlink
- Hard path guard so we only delete under the allowed log root
- Best-effort VMware snapshot pruning by prefix + age
- Prunes old timestamped logs/JSON/discovery files; leaves *_latest.* symlinks alone
- Keeps (but marks deprecated) the old local snapshot marker removal