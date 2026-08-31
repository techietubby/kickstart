# kickstart

CentOS 7/8 and AlmaLinux 8/9 Kickstart provisioning for Libvirt.

## provision-vm.yml (shared)

`provision-vm.yml` is the generic VM builder. It loads `vars/defaults.yml`
then `vars/<vm>.yml` (the host file wins), renders `templates/<vm>.j2` to the
local HTTP root, and rebuilds the domain with `virt-install`.

```bash
ansible-playbook provision-vm.yml -e vm=foreman
ansible-playbook provision-vm.yml -e vm=foreman2
ansible-playbook provision-vm.yml -e vm=foreman3
```

Add a host by dropping a `vars/<name>.yml` with only the values that differ
from `vars/defaults.yml`, plus a `templates/<name>.j2` kickstart template.

This replaces the per-host `create-vm-foreman*.yml` playbooks (kept for now
for reference; safe to delete once the shared path is confirmed).

## Legacy

`create-vm-<name>.yml` — the older one-file-per-VM playbooks. Same idea,
copy-pasted per host.
