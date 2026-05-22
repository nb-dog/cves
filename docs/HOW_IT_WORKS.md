# CVE Kernel Playbooks: How They Work

This repository provides four Ansible entrypoint playbooks backed by a shared role.
It is designed to audit, prepare, reboot, and mitigate Linux hosts impacted by:

- CVE-2026-31431 (Copy Fail)
- CVE-2026-43284 (Dirty Frag)
- CVE-2026-46300 (Fragnesia)
- CVE-2026-46333 (ssh-keysign-pwn)
- CVE-2026-42945 (NGINX check marker)

## Repository Layout

- `playbooks/audit.yml`: read-only audit run and report generation
- `playbooks/prepare.yml`: install kernel updates and apply mitigations when needed
- `playbooks/reboot.yml`: reboot when safe and required
- `playbooks/mitigate_sshkeysign.yml`: ssh-keysign mitigation only
- `roles/cve_kernel/defaults/main.yml`: tunables and version thresholds
- `roles/cve_kernel/tasks/main.yml`: shared execution logic
- `roles/cve_kernel/templates/cve_report.csv.j2`: CSV report template
- `roles/cve_kernel/templates/unfixedhosts.yml.j2`: inventory artifact template
- `docs/BEHAVIOR_MATRIX.md`: expected behavior matrix
- `docs/TEST_PLAN.md`: pre-prod validation checklist

## Execution Model

All entrypoint playbooks call the same role and set `cve_kernel_action_mode`:

- `audit`
- `prepare`
- `reboot`
- `mitigate_sshkeysign`

The role runs in these phases:

1. Platform detection and support assertion
2. Threshold resolution (fixed kernel versions)
3. Pre-action signal collection and CVE status computation
4. Mode-specific actions
5. Post-action signal collection and status recomputation
6. Host report assembly and optional report artifact generation

## Status Semantics

Per CVE, status is one of:

- `FIXED`: running kernel is at or above fixed version threshold
- `MITIGATED`: running kernel is below threshold, but mitigation controls are active
- `VULNERABLE`: kernel not fixed and mitigation signals absent
- `N/A`: only used where CVE checks do not apply (for example NGINX absent)

Mitigation signals used by the role:

- `/etc/modprobe.d/disable-dirtyfrag.conf` present
- `kernel.yama.ptrace_scope >= 2`
- `kernel.user_ptrace == 0` (CloudLinux)
- `user.max_user_namespaces == 0` (additional hardening signal)

## Mode Details

## 1) audit

Purpose: read-only assessment and artifact generation.

Actions:

- Detect OS family and currently running kernel
- Compute CVE statuses from kernel + mitigation signals
- Build host report records
- Generate:
  - `reports/cveoverview_DDMMYYYY.csv`
  - `reports/unfixedhosts.yml`

No package install, reboot, or mitigation write is performed.

## 2) prepare

Purpose: pre-maintenance patch-and-mitigate phase.

Actions:

- Update kernel packages:
  - RHEL family: `kernel`, `kernel-core`, `kernel-modules`
  - Ubuntu: `linux-image-generic`
  - Debian: architecture-aware meta package mapping (from defaults)
- Apply Dirty Frag mitigation file when CVE state is vulnerable
- Apply ssh-keysign mitigations when vulnerable:
  - persist `kernel.yama.ptrace_scope = 2`
  - runtime `sysctl -w kernel.yama.ptrace_scope=2`
  - CloudLinux `kernel.user_ptrace = 0` persist + runtime
- Remove SUID bit from exploit-adjacent binaries when vulnerable
- Upgrade nginx package if installed
- Recompute post-action states for accurate reporting

## 3) reboot

Purpose: reboot safely after updates, only when justified.

Reboot gating logic:

- Determine `running_kernel` vs `newest_installed_kernel`
- Validate boot target before reboot
  - RHEL: use `grubby --default-kernel` and compare to newest installed
  - Debian/Ubuntu: verify `/boot/vmlinuz-<newest>` exists
- Reboot only if:
  - running kernel differs from newest installed kernel, and
  - boot validation passes (unless `cve_kernel_require_boot_validation: false`)

Post reboot:

- Refresh facts
- Recompute CVE states and mitigation signals
- Include reboot reason in host report

## 4) mitigate_sshkeysign

Purpose: apply only CVE-2026-46333 mitigation path.

Actions:

- Persist + runtime `kernel.yama.ptrace_scope=2`
- CloudLinux: persist + runtime `kernel.user_ptrace=0`
- Recompute post-action status

## Defaults and Tunables

Key variables in `roles/cve_kernel/defaults/main.yml`:

- `cve_kernel_dirtyfrag_fixed_versions`
- `cve_kernel_fragnesia_fixed_versions`
- `cve_kernel_sshkeysign_fixed_versions`
- `cve_kernel_debian_kernel_meta`
- `cve_kernel_reports_dir`
- `cve_kernel_reboot_timeout_seconds`
- `cve_kernel_require_boot_validation`

Override in inventory/group vars/host vars as needed.

## How To Use

1. Populate `inventory/hosts.yml`.
2. Run one of:

```bash
ansible-playbook playbooks/audit.yml -i inventory/hosts.yml
ansible-playbook playbooks/prepare.yml -i inventory/hosts.yml
ansible-playbook playbooks/reboot.yml -i inventory/hosts.yml
ansible-playbook playbooks/mitigate_sshkeysign.yml -i inventory/hosts.yml
```

3. Review artifacts in `reports/` after `audit`.

Recommended sequence for maintenance window workflows:

1. `audit`
2. `prepare`
3. `reboot`
4. `audit` again (verification pass)

## Safety Notes

- `audit` is read-only.
- `prepare` and `mitigate_sshkeysign` make system changes.
- `reboot` can restart hosts; run during approved maintenance windows.
- Boot validation is enabled by default (`cve_kernel_require_boot_validation: true`).

## Validation Commands

Syntax:

```bash
ansible-playbook --syntax-check playbooks/audit.yml
ansible-playbook --syntax-check playbooks/prepare.yml
ansible-playbook --syntax-check playbooks/reboot.yml
ansible-playbook --syntax-check playbooks/mitigate_sshkeysign.yml
```

Lint:

```bash
ansible-lint playbooks roles
```
