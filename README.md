# CVE Kernel Playbooks

Ansible playbooks for auditing and remediating CVE-2026-31431-related kernel risks,
including Dirty Frag, Fragnesia, ssh-keysign-pwn mitigation paths, and NGINX presence markers.

## Quick Start

1. Add your hosts to `inventory/hosts.yml`.
2. Run:

```bash
ansible-playbook playbooks/audit.yml -i inventory/hosts.yml
ansible-playbook playbooks/prepare.yml -i inventory/hosts.yml
ansible-playbook playbooks/reboot.yml -i inventory/hosts.yml
ansible-playbook playbooks/mitigate_sshkeysign.yml -i inventory/hosts.yml
```

## Recommended Workflow

1. `audit` (baseline)
2. `prepare` (patches + mitigations)
3. `reboot` (safe reboot if needed)
4. `audit` (verification)

## Repository Layout

- `playbooks/`: entrypoint playbooks
- `roles/cve_kernel/`: shared role with mode-driven logic
- `inventory/hosts.yml`: inventory file
- `docs/HOW_IT_WORKS.md`: complete architecture and operations guide
- `docs/BEHAVIOR_MATRIX.md`: mode-by-mode behavior contract
- `docs/TEST_PLAN.md`: validation matrix and assertions

## Output Artifacts

- `reports/cveoverview_DDMMYYYY.csv`
- `reports/unfixedhosts.yml`

## Validation

```bash
ansible-playbook --syntax-check playbooks/audit.yml
ansible-playbook --syntax-check playbooks/prepare.yml
ansible-playbook --syntax-check playbooks/reboot.yml
ansible-playbook --syntax-check playbooks/mitigate_sshkeysign.yml
ansible-lint playbooks roles
```
