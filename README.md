# BootcBlade

Ansible automation for deploying a KVM hypervisor using bootc and CentOS Stream.

![BootcBlade](docs/images/logo.png)

This Ansible automation uses bootc to create "the perfect" KVM hypervisor with ZFS, NFS + Samba, Cockpit, and Sanoid + Syncoid.

## Usage - deploy on top of existing system
1. Install a fresh CentOS Stream to the desired host - use the latest minimal install to save disk space on the resulting deployed machine
2. Install ```podman``` on the host
3. Generate an SSH key
4. Create inventory using the example in the ```docs``` folder
5. Make sure you have passwordless sudo or root access to desired host using your new SSH key
6. Install needed Ansible collections ```ansible-galaxy install -r collections/requirements.yml```
7. Run ```ansible-playbook deploy.yml -i inventories/<your_inventory.yml>```

## Usage - create ISO for automatic installation
1. You will need an existing machine with Podman and Ansible
2. The Ansible can be run locally or remotely - just be sure to have root or passwordless sudo working
3. Create inventory for a local or remote run using the example in the ```docs``` folder
4. Run ```ansible-playbook iso.yml -i inventories/<your_inventory.yml>```
5. ISO will be created on the local or remote machine, ```/root/bootcblade-output/bootiso/install.iso```


## Credentials
### Deploy
- The SSH key (for the user) is added for the root user and baked into the image generation - **the resulting BootcBlade image will include your SSH key**
- The user, ssh key, and password (if defined) are configured after deployment via Ansible

### ISO
- The user, SSH key, and password (if defined) are baked into the ISO - **the resulting ISO will include your user, SSH key, and hashed password (if defined)**


## How It Works
### Deploy
1. A new or existing system must exist. This system should be as small as possible because its filesystem will persist in the resulting deployed machine
2. ```bootcblade.containerfile``` is copied to the existing system, then ```podman build``` is used to build the image
3. Once the image is built, the BootcBlade image is deployed to the system - then it is rebooted
4. Ansible creates the user with (or without) the password and adds the SSH key

### ISO
1. An existing system must exist to build the ISO on, no changes to this system are made
2. Running the Ansible will create the files necessary to generate the ISO - including the user with (or without) password and the SSH key
3. Resulting ISO is stored in the /root directory
4. Configuration files and all used container images are deleted

## Updating
Updates happen in two ways. When ```deploy.yml``` is used, there is a systemd unit created (```bootcblade-rebuild.service```) that is created and will run once a week.
This service depends on ```/root/bootcblade.containerfile``` to exist, as the container is rebuilt. If the build is successful, then the new container is staged.
The default update service ```bootc-fetch-apply-updates.service``` is masked as well so it will not run.

If ```deploy.yml``` is not used (perhaps the system was installed using an ISO created by ```iso.yml```), then there is no automated update process
and the default ```bootc-fetch-apply-updates.service``` is still enabled. If you want the automatic update method, then ```ansible-playbook deploy.yml --tags configure-bootcblade```
will need to be run, either remotely or as localhost, and the required variables will need to be defined.

## Troubleshooting
### ```/root/bootcblade.containerfile``` is gone:
You can use ```update.yml``` to recreate this, assuming you have the correct inventory.

### BootcBlade will no longer build
The default tag used for ```centos-bootc``` is referenced in ```templates/bootcblade.containerfile.j2``` - its possible that there was a kernel update, or a release update, that breaks ZFS. Usually these issues are transient and resolve on their own. If you need a build now (perhaps for a fresh system) you can try and see if there is an older release (tag) from the upstream repo, and adjust it using the ```bootc_image_tag``` variable.

[https://quay.io/repository/centos-bootc/centos-bootc?tab=tags&tag=latest](https://quay.io/repository/centos-bootc/centos-bootc?tab=tags&tag=latest)

## Variable Usage
This is a description of each variable, what it does, and a table to determine when it is needed.

- ```create_user```: This user will be created during ```deploy.yml``` and ```iso.yml```
- ```create_user_password```: This password will be used for the created user
- ```create_user_ssh_pub```: This is a SSH pubkey that will be added to the created user during ```deploy.yml``` and ```iso.yml```, also it is applied to the root user in ```deploy.yml```
- ```create_user_shell```: This shell setting will be used for the created user only during ```deploy.yml```
- ```bootc_image_tag```: Override the source image tag for ```deploy.yml```, ```iso.yml```, and ```update.yml```
- ```bootc_acknowledge```: This setting is only effective when setting it to ```false```, newer versions of ```bootc``` require an acknowledgment during ```deploy.yml``` but older versions break
if this is defined - so this can override the default and remove that
- ```ansible_user``` - This is an Ansible variable, useful for connecting to the initial machine with a different user during ```deploy.yml```
- ```ansible_connection``` - This is an Ansible variable, useful when running Ansible locally with ```iso.yml``` and ```update.yml```

### deploy.yml
| Variable | Used | Required |
| -------- | ---- | -------- |
| create_user          | X | - |
| create_user_password | X | - |
| create_user_ssh_pub  | X | X |
| create_user_shell    | X | - |
| bootc_image_tag      | X | - |
| bootc_acknowledge    | X | - |

### iso.yml
| Variable | Used | Required |
| -------- | ---- | -------- |
| create_user          | X | X|
| create_user_password | X | - |
| create_user_ssh_pub  | X | X |
| create_user_shell    | - | - |
| bootc_image_tag      | X | - |
| bootc_acknowledge    | - | - |

### update.yml
| Variable | Used | Required |
| -------- | ---- | -------- |
| create_user          | - | - |
| create_user_password | - | - |
| create_user_ssh_pub  | - | - |
| create_user_shell    | - | - |
| bootc_image_tag      | X | - |
| bootc_acknowledge    | - | - |

## Hacks / Workarounds
### ZFS
I ran into a few problems with ZFS - the ZFS on Linux packages for CentOS didn't quite work for CentOS Stream.
Yes the older kernel was still in use and didn't change major versions however sometimes Red Hat backported changes from newer kernels.
I considered using Fedora Server (instead of CentOS Stream) however the problem was reverse then, sometimes Fedora changes major kernel versions mid-release.
So I settled with using CentOS Stream for the base and the Fedora ZoL release packages. I may tweak the code or the exact release being used but this seems
to be the most stable so far.

### Cockpit
I ran into a problem where the ```cockpit-ws``` package would not install onto the base image [https://github.com/containers/bootc/issues/571](https://github.com/containers/bootc/issues/571).
There was some advice in that thread about using the containerized version of ```cockpit-ws``` so that is what I am doing, however this is being applied after deployment via Ansible
and not baked into the image.
[https://quay.io/repository/cockpit/ws](https://quay.io/repository/cockpit/ws)

Using this containerized version of ```cockpit-ws``` also brought problems, using the privileged container caused mount points to be held inside the container.
This meant once the container started, ZFS datasets could not be deleted since they were still "mounted" inside the container. To workaround this bastion mode
is being used instead. That means to login to Cockpit you have to use the host ```host.containers.internal```. SSL certificates can still be added to the
```/etc/cockpit/ws-certs.d``` directory - it is mounted into the container.

This also explains why I'm using rpm vs dnf to install the 45Drives Cockpit packages - they have a dependency on ```cockpit-ws``` that I need to override.
Once the official ```cockpit-files``` package is released I will be using that instead of ```cockpit-navigator```.
