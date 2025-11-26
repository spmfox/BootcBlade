# BootcBlade

Ansible automation for deploying a KVM hypervisor using bootc and Fedora Server.

![BootcBlade](docs/images/logo.png)

This Ansible automation uses bootc to create "the perfect" KVM hypervisor with ZFS, NFS + Samba, Cockpit, and Sanoid + Syncoid.

## How It Works
BootcBlade is split into two Ansible roles:
- ['bootc'](roles/bootc)
  - test
- ['host-configure`](roles/host-configure)
  - test
