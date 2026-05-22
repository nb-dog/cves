# Behavior Matrix

Status model used by all playbooks:

- `FIXED`: Running kernel is at or above the CVE fixed threshold.
- `MITIGATED`: Running kernel is not fixed, but host-level mitigations are active.
- `VULNERABLE`: Neither fixed nor mitigated.
- `N/A`: Check does not apply to this host or package is not present.

## Modes

### audit

- No system changes.
- Computes CVE status from running kernel and mitigation signals.
- Writes `reports/cveoverview_DDMMYYYY.csv`.
- Writes `reports/unfixedhosts.yml` for non-fully-fixed and unreachable hosts.

### prepare

- Installs latest kernel packages.
- Applies mitigations when host is not yet fixed.
- Recomputes post-action status and recommends reboot only when needed.

### reboot

- Re-checks installed vs running kernel.
- Reboots only when a newer kernel is installed.
- Recomputes status after reboot.
- Removes temporary mitigations only after host is confirmed fixed.

### mitigate_sshkeysign

- Applies only CVE-2026-46333 mitigations.
- Persists and applies runtime `kernel.yama.ptrace_scope=2`.
- For CloudLinux also sets `kernel.user_ptrace=0`.
