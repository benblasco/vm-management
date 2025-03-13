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
ansible-playbook libvirt-newvm.yml --ask-become-pass -e "distribution_version=<rhel version>" -e "disk_size=<size<>GB" -e "memory_mb=<memory in MB>" -e "vcpus=<number of vcpus>" -e "cloud_init_seed_iso=<iso file name>" -e "boot_mode=<boot mode>"
```
# Default variables

See `defaults/main.yml` for default values of:
- Linux OS/version
- Disk size
- Memory
- vCPUs
- Boot mode (which can be `efi` or `bios`). Note: RHEL 7 hosts must be overridden to `bios`, else they will not boot
- Cloud-init seed ISO file

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

