# How to use

Use [`libvirt-createvm.yml`](libvirt-createvm.yml) to create a VM with an auto-generated cloud-init seed ISO in a single playbook. The playbook:

1. Defines the VM (without starting it) using the `ansible-role-libvirt-vm` role
2. Retrieves the auto-assigned MAC address from the domain XML
3. Generates a per-VM cloud-init seed ISO (with static network config when `vm_ip_address` is set)
4. Starts the VM when `vm_start` is truthy

The libvirt domain name is `{vm_hostname}.{vm_domain}` (e.g. `rhel101.nuc.blasco.id.au` on `nuc.lan` with defaults). The disk volume in `vm-pool` is named after the short `vm_hostname` only (e.g. `rhel101`).

### Accept all defaults

```
ansible-playbook libvirt-createvm.yml --ask-become-pass
```

By default, the VM hostname matches the `distribution_version` parameter (distro/version key).

### Accept all defaults but have a different VM name

```
ansible-playbook libvirt-createvm.yml --ask-become-pass -e "vm_hostname=<vm name>"
```

### Accept all defaults but have a different OS version

```
ansible-playbook libvirt-createvm.yml --ask-become-pass -e "distribution_version=<rhel version>"
```

See [Available distributions](#available-distributions) for all supported `distribution_version` keys.

### DHCP networking (no static IP)

When `vm_ip_address` is omitted or empty, the integrated `generate-cloud-init-iso` role builds a seed ISO with **user-data only** (no `network-config`). The guest NIC obtains an address via **DHCP on the libvirt network** specified by `vm_host_network`. `vm_ip_gateway`, `vm_ip_nameservers`, and `vm_ip_prefix` are ignored unless `vm_ip_address` is set.

```
ansible-playbook libvirt-createvm.yml --ask-become-pass \
  -e hypervisor_host=hex.lan \
  -e vm_hostname=myvm
```

### Static IP networking

When `vm_ip_address` is set, `vm_ip_gateway` and `vm_ip_nameservers` are required (the playbook fails if either is empty). `vm_ip_prefix` is optional (default `24`). Do not pass `vm_mac_address`.

```
ansible-playbook libvirt-createvm.yml --ask-become-pass \
  -e hypervisor_host=hex.lan \
  -e "vm_host_network=vm-network-vlan140" \
  -e vm_hostname=satellite1 \
  -e vm_domain=savage.test \
  -e vm_ip_address=192.168.140.12 \
  -e vm_ip_prefix=22 \
  -e vm_ip_gateway=192.168.140.1 \
  -e 'vm_ip_nameservers=["192.168.140.5"]' \
  -e vm_start=yes \
  -e "distribution_version=rhel98" \
  -e vcpus=8 \
  -e disk_size=120GB \
  -e memory_mb=24576
```

**Nameservers quoting:** use the JSON array form shown above. Ansible's `-e key=value` syntax does not YAML-parse the value, so `-e 'vm_ip_nameservers=[192.168.1.3, 192.168.1.7]'` passes a literal string and will not work as intended.

### Override parameters

```
ansible-playbook libvirt-createvm.yml --ask-become-pass \
  -e "distribution_version=<rhel version>" \
  -e "disk_size=<size>GB" \
  -e "memory_mb=<memory in MB>" \
  -e "vcpus=<number of vcpus>" \
  -e "boot_mode=<boot mode>" \
  -e "vm_host_network=<libvirt network name>" \
  -e "vm_start=<yes|no>" \
  -e "vm_autostart=<yes|no>"
```

### Attach a VM to a different libvirt network

By default, VMs use the `vm-network-routed` libvirt network (NAT/routed). To attach a VM to VLAN 140 instead, override `vm_host_network`:

```
ansible-playbook libvirt-createvm.yml --ask-become-pass \
  -e "vm_host_network=vm-network-vlan140"
```

The same variable applies to `libvirt-isoinstall.yml`. The `vm-network-vlan140` network must exist on the hypervisor (defined by `config-libvirt-hpc.yml`).

### Delete a VM

```
ansible-playbook libvirt-deletevm.yml --ask-become-pass -e "vm_hostname=<vm name>"
```

Boot firmware (EFI vs BIOS, and whether `virsh undefine --nvram` is used) is detected automatically from the domain XML when the VM exists.

Optional overrides:

```
ansible-playbook libvirt-deletevm.yml --ask-become-pass \
  -e "vm_hostname=<vm name>" \
  -e "vm_domain=<domain>" \
  -e "hypervisor_host=<host>"
```

Pass `vm_domain` only if the VM was created with a non-default domain. `vm_domain` affects the libvirt domain name, not the disk volume name.

# Configurable parameters

Override any of these with `-e` at the command line. Default values are in [`defaults/main.yml`](defaults/main.yml).

| Variable | Purpose |
|----------|---------|
| `hypervisor_host` | Inventory host to target (limits `hosts:`); omit to run against all hypervisors |
| `distribution_version` | Key into `vm_image` dict — selects QCOW2 base image (see [Available distributions](#available-distributions)) |
| `vm_hostname` | Short hostname; if omitted, defaults to `distribution_version` |
| `vm_domain` | Domain suffix; FQDN = `{vm_hostname}.{vm_domain}` |
| `vm_host_network` | Libvirt network the VM NIC attaches to |
| `vm_autostart` | Whether libvirt autostarts the domain on hypervisor boot |
| `vm_start` | Whether to power on the VM after seed ISO generation |
| `disk_size` | Root disk capacity (e.g. `120GB`) |
| `memory_mb` | RAM in megabytes |
| `vcpus` | Virtual CPU count |
| `boot_mode` | `efi` or `bios` (RHEL 7 images require `bios`) |
| `cloud_init_seed_iso` | Seed ISO filename; auto-derived as `{vm_hostname}.{vm_domain}-seed.iso` unless overridden |
| `vm_ip_address` | Static IPv4; omit for DHCP. When set, `vm_ip_gateway` and `vm_ip_nameservers` are required. |
| `vm_ip_prefix` | Optional CIDR prefix when using static IP (default `24`); ignored on DHCP. |
| `vm_ip_gateway` | Required with `vm_ip_address`; ignored on DHCP. |
| `vm_ip_nameservers` | Required with `vm_ip_address`; ignored on DHCP. |
| `vm_mac_address` | Auto-set by playbook from domain XML; do not pass manually under normal use |
| `vm_image_path` | Directory on the hypervisor where QCOW2 images and seed ISOs live; defined in [`vars/vm_image.yml`](vars/vm_image.yml) |

# Resolving JSON errors on Fedora 43+

From Fedora 43 onwards I have been getting errors when creating and destroying VMs via the playbook, which look like:
```
TASK [ansible-role-libvirt-vm : Ensure the VM disk volumes exist] **************
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: json.decoder.JSONDecodeError: Extra data: line 2 column 1 (char 19)
fatal: [nuc.lan]: FAILED! => {"msg": "Unexpected failure during module execution: Extra data: line 2 column 1 (char 19)", "stdout": ""}
```

This is resolved by setting the following in the inventory variables for each hypervisor:
```
ansible_ssh_common_args: '-o SetEnv=\"TERM=dumb\"'
```

Note: The character escaping shown above is required. Please do not confuse this for a typo or a formatting error in this README.

# Available distributions

See [`vars/vm_image.yml`](vars/vm_image.yml) for the full `vm_image` dictionary. Each key is a valid `distribution_version` value.

The QCOW2 image must be available under the path described by `vm_image_path` in that file.

# Cloud-init

[`libvirt-createvm.yml`](libvirt-createvm.yml) generates the cloud-init seed ISO automatically as part of VM creation. You do not need to build or copy a seed ISO beforehand.

- **DHCP:** omit `vm_ip_address` — the seed ISO contains user-data only and the guest uses DHCP on `vm_host_network`.
- **Static IP:** set `vm_ip_address`, `vm_ip_gateway`, and `vm_ip_nameservers` (and optionally `vm_ip_prefix`). The playbook retrieves `vm_mac_address` from the domain XML before rendering network config.

For standalone seed ISO generation (without creating a VM), see [`README.generate-cloud-init-iso-role.md`](README.generate-cloud-init-iso-role.md).
