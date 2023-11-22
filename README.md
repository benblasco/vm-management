# How to use

### Accept all defaults

```
ansible-playbook libvirt-newvm.yml --ask-become-pass
```

Note: By default, the VM hostname will match the "distro_version" (ie. distro/version) parameter:

### Accept all defaults but have a different VM name

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "vm_name=<vm name>"
```

### Accept all defaults but have a different OS version

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "distro_version=<rhel version>"
```

Note: Please refer to vars/vm_image for all the available OS versions

### Delete a VM

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "vm_name=<vm name>" -e vm_state=absent"
```

### Override parameters

```
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "distro_version=<rhel version>" -e "disk_size=<size<>GB" -e "memory_mb=<memory in MB>" -e "vcpus=<number of vcpus>"
```
# Default variables

See `defaults/main.yml` for default values of:
- Linux OS/version
- Disk size
- Memory
- vCPUs

# Available distributions

See `vars/vm_image.yml`

Note that the QCOW2 image must be available under `/var/lib/libvirt/images`

# Cloud-init

I have hard coded a cloud init seed qcow2 image at `image: '/var/lib/libvirt/images/generic-seed.qcow2'`. This code needs to be updated to give the user some flexibility not to supply a cloud-init, or to provide a path to a different qcow2 file.
