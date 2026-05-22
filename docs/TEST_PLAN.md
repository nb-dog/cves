# Scenario Test Plan

Run these against representative hosts before production rollout.

## Host Matrix

- RHEL 8
- RHEL 9
- Ubuntu (supported LTS)
- Debian (supported stable)

## Commands

```bash
ansible-playbook playbooks/audit.yml -i inventory/hosts.yml --limit <host>
ansible-playbook playbooks/prepare.yml -i inventory/hosts.yml --limit <host>
ansible-playbook playbooks/reboot.yml -i inventory/hosts.yml --limit <host>
ansible-playbook playbooks/mitigate_sshkeysign.yml -i inventory/hosts.yml --limit <host>
```

## Assertions

- `audit`: CSV and unfixed inventory are generated in `reports/`.
- `prepare`: kernel packages are updated and mitigations applied when still vulnerable.
- `reboot`: reboot only occurs when installed kernel differs from running kernel.
- `mitigate_sshkeysign`: ptrace mitigations are present and active.
- rerun idempotence: second run should produce minimal changes.
