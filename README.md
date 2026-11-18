# kickstart

CentOS 7/8 and AlmaLinux 8/9 Kickstart provisioning for Libvirt.

## provision-vm.yml (shared)

`provision-vm.yml` is the generic VM builder. It loads `vars/defaults.yml`
then `vars/<vm>.yml` (the host file wins), renders `templates/<vm>.j2` to a
private temp directory, injects it into the installer initrd with
`virt-install`, and deletes it again once the build has started.

```bash
ansible-playbook provision-vm.yml -e vm=podman                  # new VM
ansible-playbook provision-vm.yml -e vm=podman -e rebuild=true  # wipe + reinstall
```

### Rebuilding an existing VM

The playbook refuses to touch a VM whose libvirt domain **or** disk image
already exists unless you pass `-e rebuild=true`. With it, the playbook:

1. destroys and undefines the domain,
2. deletes the old disk image (`diskFile`), so a changed `diskSize` actually
   takes effect,
3. reinstalls from scratch.

This protects running hosts from a mistyped `-e vm=...`.

### Kickstart handling and secrets

The rendered kickstart contains the root password hash and the IPA admin
password. It is written with mode `0600` to a temp directory, passed to
`virt-install --initrd-inject` (the installer reads it via
`inst.ks=file:/<vm>.ks`), and removed in an `always:` block even if the build
fails. It is **not** published to the web root any more; any leftover
`<ksLoc>/<vm>.ks` from older runs is deleted on each run.

### Disk I/O tuning

Set in `vars/defaults.yml` and overridable per host in `vars/<vm>.yml`. They
are appended to `virt-install --disk`; set a value to `""` to leave it out
and fall back to the virt-install/QEMU default.

| Variable           | Default  | Options                                             |
|--------------------|----------|-----------------------------------------------------|
| `diskCache`        | `none`   | `none`, `writeback`, `writethrough`, `directsync`, `unsafe` |
| `diskIo`           | `native` | `native`, `threads`, `io_uring`                     |
| `diskDiscard`      | `unmap`  | `unmap`, `ignore` — pass guest TRIM to the host     |
| `diskDetectZeroes` | `unmap`  | `unmap`, `on`, `off` — free zeroed blocks in the qcow2 |

`io=native` requires `cache=none` (or `directsync`). If you set
`diskCache: writeback`, also set `diskIo: ""` or `threads`.

These settings only apply when a VM is built or rebuilt. To change a running
VM, edit its `<driver>` line with `virsh edit <vm>` and restart it.

### Adding a host

1. Add a DNS entry (or `/etc/hosts` line) and an `inventory` entry.
2. Create `vars/<name>.yml` with **only** the values that differ from
   `vars/defaults.yml`. At minimum: `macAddress`, `netIp`, `hostName`. Both
   defaults belong to the foreman host, so a new VM that inherits them will
   clash with it.
3. Create `templates/<name>.j2` (copy `templates/podman.j2`, which has the
   fixes below).
4. `ansible-playbook provision-vm.yml -e vm=<name>`

Logical volumes in `logVol` accept `grow: true`. That LV then takes all
remaining space in the volume group, and `lvSize` becomes its minimum size.

## podman host

`vars/podman.yml` and `templates/podman.j2` describe an AlmaLinux 9 Podman
container host (192.168.0.47, 8 GiB RAM, 80 GiB disk on `/virt4`).

* **Disk layout:** no swap. `/var` is 8 GiB, and `/var/lib/containers` (for
  podman image and container storage) grows to fill the rest of the VG,
  about 50 GiB. Rootless podman stores images under `~/.local/share/containers`
  on `/home`, which is only 1 GiB.
* **Firewall:** one kickstart `firewall` line built from `fwService` and
  `fwPort`. Ports must use the `port:proto` form, e.g. `53:udp` or
  `1024-1300:tcp`; `53/udp` is not valid kickstart syntax.
* **IPA:** `ipa-client` is installed from `%packages`. EL9 has no
  `idm:client` module stream.
* **TRIM:** `fstrim.timer` is enabled, so freed blocks are released to the
  host weekly (works with `diskDiscard: unmap`).

## Changes (2026-11-18)

Shared (`provision-vm.yml`, `vars/defaults.yml`) — affects every VM:

* Kickstart rendered to a private `0600` temp dir and deleted after the
  build, instead of being published world-readable in `/var/www/html`.
* `-e rebuild=true` required to destroy an existing domain or disk.
* Old disk image deleted on rebuild, so `diskSize` changes take effect.
* `stat` checks on the ISO and disk image skip checksums. Hashing those files took about 170 seconds of a
  187-second run.
* New disk I/O tuning variables (see above).

podman only (`vars/podman.yml`, `templates/podman.j2`):

* Firewall rules emitted as one `firewall` line. Kickstart's `firewall` is
  a single-use command, so with one line per rule only the last service and
  port took effect.
* Ports converted from `port/proto` to `port:proto`.
* `ipa-client` added to `%packages`; EL8-only `idm:client` module commands
  and the deprecated `--force-ntpd` option removed.
* `vars/podman.yml` trimmed to overrides only; `/var` raised from 4 to
  8 GiB and a growable `/var/lib/containers` LV added.
* `fstrim.timer` enabled.

The other `templates/*.j2` files still have the old one-line-per-rule
firewall loop and the EL8 IPA commands. Copy the fixes from `podman.j2` when
you next rebuild those hosts.

## Legacy

`create-vm-<name>.yml`: the older one-file-per-VM playbooks, copy-pasted per
host and replaced by `provision-vm.yml`.
