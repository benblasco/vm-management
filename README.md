# How to use

### Accept all defaults

```
ansible-playbook libvirt-newvm.yml --ask-become-pass
```

Note: By default, the VM hostname will match the "distribution_version" (ie. distro/version) parameter:

### Accept all defaults but have a different VM name

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "vm_name=<vm name>"
```

### Accept all defaults but have a different OS version

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "distribution_version=<rhel version>"
```

Note: Please refer to vars/vm_image for all the available OS versions

### Delete a VM

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "vm_name=<vm name>" -e vm_state=absent"
```

### Override parameters

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "distribution_version=<rhel version>" -e "disk_size=<size<>GB" -e "memory_mb=<memory in MB>" -e "vcpus=<number of vcpus>" -e "cloud_init_seed_iso=<iso file name>" -e "boot_mode=<boot mode>" -e "vm_host_network=<libvirt network name>"
```

### Attach a VM to a different libvirt network

By default, VMs use the `vm-network-routed` libvirt network (NAT/routed). To attach a VM to VLAN 140 instead, override `vm_host_network`:

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "vm_host_network=vm-network-vlan140"
```

The same variable applies to `libvirt-isoinstall.yml`. The `vm-network-vlan140` network must exist on the hypervisor (defined by `config-libvirt-hpc.yml`).

# Default variables

See `defaults/main.yml` for default values of:
- Linux OS/version
- Disk size
- Memory
- vCPUs
- Boot mode (which can be `efi` or `bios`). Note: RHEL 7 hosts must be overridden to `bios`, else they will not boot
- Cloud-init seed ISO file
- Libvirt network (`vm_host_network`; default `vm-network-routed`)

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

See `vars/vm_image.yml`

Note that the QCOW2 image must be available under the path described by `vm_image_path` defined in `vars/vm_image.yml`

# Cloud-init

There's a default cloud-init seed iso image defined in `defaults/main.yml`

You can refer use the `cloud_init_seed_iso` variable to change to a different cloud-init seed image, but that image must be located in /var/lib/libvirt/images

# How to generate a cloud-init seed image

To generate a seed file to be used in a VM, you need the `cloud-utils-cloud-localds` package installed. Then you can use the following command to generate the image from your config file:

```
cloud-localds -v <SEED ISO FILE NAME> <CLOUD-INIT CONFIG FILE NAME>
```

e.g.
```
cloud-localds -v generic-seed.iso cloud_init.cfg 
```

