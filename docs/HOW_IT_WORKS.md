# CVE Kernel Playbooks: How They Work

This repository provides four Ansible entrypoint playbooks backed by a shared role.
It targets audit and remediation workflows for:

- CVE-2026-31431 (Copy Fail)
- CVE-2026-43284 (Dirty Frag)
- CVE-2026-46300 (Fragnesia)
- CVE-2026-46333 (ssh-keysign-pwn)
- CVE-2026-42945 (NGINX check marker)

## Repository Layout

- `playbooks/audit.yml`: read-only audit and report generation
- `playbooks/prepare.yml`: package updates and mitigations
- `playbooks/reboot.yml`: gated reboot flow
- `playbooks/mitigate_sshkeysign.yml`: CVE-2026-46333 mitigation only
- `roles/cve_kernel/defaults/main.yml`: tunables and fixed-version thresholds
- `roles/cve_kernel/tasks/main.yml`: dispatcher by `cve_kernel_action_mode`
- `roles/cve_kernel/tasks/audit.yml`: audit stage
- `roles/cve_kernel/tasks/prepare.yml`: prepare stage
- `roles/cve_kernel/tasks/reboot.yml`: reboot stage
- `roles/cve_kernel/tasks/mitigate_sshkeysign.yml`: mitigation stage
- `roles/cve_kernel/tasks/common/*.yml`: shared logic (init/signals/status/kernel/report)
- `roles/cve_kernel/templates/cve_report.csv.j2`: CSV report template
- `roles/cve_kernel/templates/unfixedhosts.yml.j2`: unfixed inventory template

## Execution Model

All entrypoint playbooks call the same role and set `cve_kernel_action_mode`.
The dispatcher in `roles/cve_kernel/tasks/main.yml` includes exactly one stage file.

Shared common includes are reused by the stage files:

- `common/init.yml`: platform classification, support check, fixed thresholds, package facts
- `common/signals_pre.yml`: pre-action mitigation signals
- `common/status_pre.yml`: pre-action status computation
- `common/kernel_detect.yml`: family-specific newest installed kernel detection + normalization
- `common/signals_post.yml`: post-action mitigation signals
- `common/status_post.yml`: post-action status computation + host report
- `common/report.yml`: CSV + unfixed/unreachable artifacts

## Status Semantics

Per CVE, status is one of:

- `FIXED`: running kernel is at or above fixed threshold
- `MITIGATED`: kernel not fixed, but mitigation controls are active
- `VULNERABLE`: neither fixed nor mitigated
- `N/A`: check does not apply (for example NGINX absent)

Mitigation signals used:

- `/etc/modprobe.d/disable-dirtyfrag.conf` present
- `kernel.yama.ptrace_scope >= 2`
- `kernel.user_ptrace == 0` (CloudLinux)
- `user.max_user_namespaces == 0`

## Mode Details

### audit

- No host changes.
- Computes statuses from running kernel and mitigation signals.
- Generates:
  - `reports/cveoverview_DDMMYYYY.csv`
  - `reports/unfixedhosts.yml`

### prepare

- Updates kernel packages:
  - RHEL: `kernel`, `kernel-core`, `kernel-modules`
  - Ubuntu: `linux-image-generic`
  - Debian: mapped meta package from defaults
- Applies mitigations where host is still vulnerable.
- Recomputes post-action statuses.
- Does not reboot.

### reboot

Reboot is gated and only happens when all required conditions are true:

1. Newest installed kernel is detected by family-specific logic (`rpm`/`dpkg`).
2. Normalized version comparison confirms `installed > running`.
3. Boot target validation passes (unless `cve_kernel_require_boot_validation: false`).

Boot validation methods:

- RHEL: `grubby --default-kernel` must match newest installed kernel
- Debian/Ubuntu: `/boot/vmlinuz-<newest>` must exist

Debug output prints the decision context:

- running kernel raw/normalized
- newest installed kernel raw/normalized
- boot validation result/reason
- reboot needed/reason

### mitigate_sshkeysign

- Applies only CVE-2026-46333 mitigation path.
- Persists and applies runtime `kernel.yama.ptrace_scope=2`.
- On CloudLinux also persists and applies runtime `kernel.user_ptrace=0`.
- Recomputes post-action status.

## Defaults and Tunables

Key variables in `roles/cve_kernel/defaults/main.yml`:

- `cve_kernel_dirtyfrag_fixed_versions`
- `cve_kernel_fragnesia_fixed_versions`
- `cve_kernel_sshkeysign_fixed_versions`
- `cve_kernel_debian_kernel_meta`
- `cve_kernel_reports_dir`
- `cve_kernel_reboot_timeout_seconds`
- `cve_kernel_require_boot_validation`

Override via inventory/group vars/host vars as needed.

## How To Use

```bash
ansible-playbook playbooks/audit.yml -i inventory/hosts.yml
ansible-playbook playbooks/prepare.yml -i inventory/hosts.yml
ansible-playbook playbooks/reboot.yml -i inventory/hosts.yml
ansible-playbook playbooks/mitigate_sshkeysign.yml -i inventory/hosts.yml
```

If sudo password prompt is required:

```bash
ansible-playbook playbooks/audit.yml -i inventory/hosts.yml -K
```

Recommended sequence:

1. `audit`
2. `prepare`
3. `reboot`
4. `audit` (verification)

## Validation

```bash
ansible-playbook --syntax-check playbooks/audit.yml
ansible-playbook --syntax-check playbooks/prepare.yml
ansible-playbook --syntax-check playbooks/reboot.yml
ansible-playbook --syntax-check playbooks/mitigate_sshkeysign.yml
ansible-lint playbooks roles
```
