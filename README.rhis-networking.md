# RHIS VLAN 140 networking

Summary of VLAN 140 L2 passthrough work for RHIS VM deployment across the homelab hypervisors (July 2026).

## Goal

Attach RHIS VMs to VLAN 140 at layer 2 while keeping hypervisor management on untagged DHCP on the physical NIC. VM traffic is untagged on `br140`; the host tags it on `<phys>.140`. Neither `br140` nor the VLAN sub-interface carries a host IP address.

## Architecture

```mermaid
flowchart LR
  mgmt["Physical NIC\nuntagged DHCP"]
  vlan["BASE_IFACE.140\n802.1Q tag 140"]
  br140["br140\nL2 bridge, no IP"]
  libvirt["vm-network-vlan140\nlibvirt bridge network"]
  vm["RHIS VM\nens2 on VLAN 140"]

  mgmt --> vlan
  vlan --> br140
  br140 --> libvirt
  libvirt --> vm
```

| Layer | Component | Purpose |
|-------|-----------|---------|
| Host management | Physical NIC (`eno1`, `enp2s0f0`, `eno2`) | Untagged DHCP (unchanged) |
| Host VLAN | `<iface>.140` | 802.1Q tag 140, bridge port |
| Host bridge | `br140` | L2 passthrough, no IP |
| Libvirt | `vm-network-vlan140` | Bridge mode to `br140` |
| VM | Guest NIC | Static or DHCP on VLAN 140 |

## Hypervisor host mapping

| Host | Physical NIC | VLAN interface | Bridge |
|------|--------------|----------------|--------|
| nuc.lan | `eno1` | `eno1.140` | `br140` |
| micro.lan | `enp2s0f0` | `enp2s0f0.140` | `br140` |
| opti.lan | `eno1` | `eno1.140` | `br140` |
| hex.lan | `eno2` | `eno2.140` | `br140` |

## Host changes (NetworkManager)

Applied manually on all four hypervisors via `nmcli` (tested on nuc.lan with reboot; fleet rollout on micro, opti, and hex without reboot).

Each host has persistent NetworkManager profiles:

- VLAN connection: `<BASE_IFACE>.140` on VLAN ID 140, enslaved to `br140`
- Bridge connection: `br140`, STP disabled, IPv4 disabled, IPv6 ignored
- Both connections set to autoconnect

Example (nuc.lan, `BASE_IFACE=eno1`):

```bash
BASE_IFACE=eno1
VLAN_IFACE="${BASE_IFACE}.140"

nmcli connection add type vlan con-name "${VLAN_IFACE}" dev "${BASE_IFACE}" id 140
nmcli connection add type bridge con-name br140 ifname br140 stp no
nmcli connection modify "${VLAN_IFACE}" master br140 slave-type bridge
nmcli connection modify br140 ipv4.method disabled ipv6.method ignore
nmcli connection modify "${VLAN_IFACE}" connection.autoconnect yes
nmcli connection modify br140 connection.autoconnect yes
```

Management DHCP on the physical NIC was left untouched.

## vm-management repo changes

### Libvirt hypervisor config (`config-libvirt-hpc.yml`)

- Added `vm-network-vlan140` libvirt network (`mode: bridge`, `bridge: br140`)
- Set `libvirt_host_install_daemon: false` and `libvirt_host_install_client: false` for Fedora bootc hypervisors (libvirt/QEMU must be pre-installed in the bootc image)
- Upgraded `stackhpc.libvirt-host` role to v1.16.0 (Ansible 2.20 compatibility)

Playbook applied to nuc.lan first, then `--limit micro.lan,opti.lan,hex.lan --skip-tags cloud-init`.

### VM creation playbooks

- Added `vm_host_network` default in `defaults/main.yml` (`vm-network-routed`)
- `libvirt-newvm.yml` and `libvirt-isoinstall.yml` now attach VMs to `{{ vm_host_network }}` instead of a hardcoded network name
- `README.md` documents the override

Create a VM on VLAN 140:

```bash
ansible-playbook libvirt-newvm.yml --ask-become-pass \
  -e "vm_host_network=vm-network-vlan140" \
  -e "vm_hostname=my-rhis-vm"
```

ISO-based install on VLAN 140:

```bash
ansible-playbook libvirt-isoinstall.yml --ask-become-pass \
  -e "vm_host_network=vm-network-vlan140"
```

### Validation

A test VM (`test140`) was created on nuc.lan on `vm-network-vlan140` (RHEL 10.1). Guest IP was configured manually on `ens2` after boot (`192.168.140.99/22`).

## Kickstart changes (fedora-server-bootc repo)

To persist `br140` across hypervisor reinstalls, a `%post` nmcli block was added to all four homelab kickstart files. The existing management line is unchanged:

```kickstart
network --bootproto=dhcp --device=link --activate --onboot=on
```

| Kickstart file | Host | `BASE_IFACE` |
|----------------|------|--------------|
| `config-nuc-kickstart.ks` | nuc.lan | `eno1` |
| `config-micro-kickstart.ks` | micro.lan | `enp2s0f0` |
| `config-opti-kickstart.ks` | opti.lan | `eno1` |
| `config-hex-kickstart.ks` | hex.lan | `eno2` |

Each `%post` block uses `nmcli --offline connection` to write NM profiles during install. Kickstart files were validated with `ksvalidator` via the Fedora 44 toolbox:

```bash
toolbox run -c fedora-toolbox-44 bash -c '
  sudo dnf install -y pykickstart
  cd ~/git/fedora-server-bootc
  for f in config-*-kickstart.ks; do ksvalidator "$f"; done
'
```

## Git commits

| Repo | Branch | Commit message |
|------|--------|------------------|
| vm-management | `main` | `feat: added option to use VLAN 140 host network for RHIS deployment` |
| fedora-server-bootc | `fedora-44-testing` | `feat: Added VLAN 140 with bridge for RHIS VMs` |

## Not yet automated

- Host VLAN/bridge NM config is not codified in an Ansible playbook in this repo (applied manually via nmcli)
- Cloud-init seed ISO does not configure guest networking for VLAN 140 VMs; IP must be set post-boot or via a dedicated seed image
- Libvirt network exists after running `config-libvirt-hpc.yml`; fresh installs need that playbook (or equivalent) in addition to kickstart bridge setup
