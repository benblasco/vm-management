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
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "distribution_version=<rhel version>" -e "disk_size=<size<>GB" -e "memory_mb=<memory in MB>" -e "vcpus=<number of vcpus>" -e "cloud_init_seed_iso=<iso file name>"
```
# Default variables

See `defaults/main.yml` for default values of:
- Linux OS/version
- Disk size
- Memory
- vCPUs

# Available distributions

See `vars/vm_image.yml`

Note that the QCOW2 image must be available under the path described by `vm_image_path` defined in `vars/vm_image.yml`

# Cloud-init

There's a default cloud-init seed iso image defined in `defaults/main.yml`

You can refer use the `cloud_init_seed_iso` variable to change to a different cloud-init seed image, but that image must be located in /var/lib/libvirt/images