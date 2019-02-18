An implementation of Molecule's delegated driver to leverage an existing libvirt daemon. It's meant to be used with preconfigured  KVM images or generic cloud images such as these:

https://docs.openstack.org/image-guide/obtain-images.html

It's currently required to install the included **delegated.py** script since the delegated driver does not support ansible_become variables. PR was submitted and approved though not merged yet:

https://github.com/ansible/molecule/pull/1738


Assumptions
-----------

- Functioning libvirt daemon on the default qemu:///system uri.
- Appropriate permissions for the user running Molecule on the libvirt daemon, directories, etc.


Common Variables
-----------------

These can be set at the driver and platform levels. At the driver level they apply to every platform, unless set also at the platform level, which takes precendence.

| Variable      | Type   | Default                 | Comment                                                        |
|---------------|--------|-------------------------|----------------------------------------------------------------|
| ssh_pub       | string | $HOME/.ssh/id_rsa.pub   | public key used to access the VM                               |
| become_method | string | su                      | method for privilege escalation if using a preconfigured image |


Driver Variables
----------------

| Variable      | Type   | Default           | Comment                                    |
|---------------|--------|-------------------|--------------------------------------------|
| qemu_path     | string | none, must be set | path for qemu-kvm or qemu-system-x86_64    |
| images_path   | string | none, must be set | path for the created VM image files        |
| download_path | string | $HOME             | downloaded base images will be placed here |
| virt_network  | string | default           | name of the libvirt network to be used     |


Platform Variables
------------------

| Variable        | Type   | Default           | Comment                                                          |
|-----------------|--------|-------------------|------------------------------------------------------------------|
| name            | string | none, must be set | name of the instance, will be prefixed with mlcl-                |
| base_image      | string | none, must be set | can be a filesystem path or a URL                                |
| user            | string | $USER             | user to access the instance                                      |
| config_vm       | bool   | true              | adds user to the VM, injects SSH keys, sets random root password |
| selinux_relabel | bool   | false             | required for SELinux-enabled images if config_vm is done         | 
| ram             | int    | 512               | memory in MB as positive integer                                 |
| cpu             | int    | 1                 | number of vCPU as positive integer                               |
| become_pass     | string | none, optional    | password for privilege escalation if using preconfigured image   |


molecule.yml
------------

The molecule.yml file included shows an example with:

- The minimum required configuration for the driver.
- One platform with minimum required configuration using all the defaults (debian9).
- One platform showing all the possible configurable parameters (centos7).

```
scenario:
  name: default
driver:
  name: delegated
  qemu_path: /usr/bin/qemu-system-x86_64  
  images_path: /var/lib/libvirt/images
platforms:
  - name: debian9
    base_image: http://cdimage.debian.org/cdimage/openstack/9.7.0/debian-9.7.0-openstack-amd64.qcow2
  - name: centos7
    ram: 1024
    cpu: 2
    user: cloud-user
    config_vm: false
    become_method: sudo
    become_pass: password
    selinux_relabel: true
    ssh_pub: /tmp/centos_key.pub
    base_image: ~/Downloads/CentOS-7-x86_64-GenericCloud.qcow2
provisioner:
  name: ansible
  playbooks:
    create: files/create.yml
    destroy: files/destroy.yml
    converge: playbook.yml
  lint:
    name: ansible-lint
lint:
  name: yamllint
scenario:
  name: default
verifier:
  name: testinfra
  directory: ./test
  env:
    PYTHONWARNINGS: "ignore:.*U.*mode is deprecated:DeprecationWarning"
  lint:
    name: flake8
  options:
    v: 1
```
