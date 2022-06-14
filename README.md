An implementation of Molecule's delegated driver to leverage an existing
libvirt daemon. It's meant to be used with preconfigured  custom images or
generic cloud images
[such as these](https://docs.openstack.org/image-guide/obtain-images.html).

It is necessary to install the included **delegated.py** script since the
delegated driver does not currently support the ansible_become variables
required for testing scenarios where escalation is needed. 
[PR was submitted and approved](https://github.com/ansible/molecule/pull/1738)
though not yet merged.


Assumptions
-----------

- Functioning libvirt daemon on the default qemu:///system uri.
- The libguestfs-tools package providing virt-customize is installed.
- Appropriate permissions for the user running Molecule on the libvirt daemon, 
directories, etc.


Common Variables
-----------------

These can be set at the driver and platform levels. At the driver level they
apply to every platform, unless set also at the platform level, which takes
precendence.

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
| extra_vol       | hash   | none, optional    | extra disk(s) to be attached to the VM with size including unit  |
| become_pass     | string | none, optional    | password for privilege escalation if using preconfigured image   |
| upload_files    | dict   | []                | list with upload items with `local_path` and `remote_path`       |


Example molecule.yml
--------------------

The example below shows:

- The minimum required configuration for the driver.
- One platform with minimum required configuration using all the defaults (debian9).
- One platform with all the possible configurable parameters (centos7).

```
scenario:
  name: default
driver:
  name: delegated
  qemu_path: /usr/bin/qemu-system-x86_64  
  images_path: /var/lib/libvirt/images
platforms:
  - name: debian9
    base_image: https://cdimage.debian.org/cdimage/openstack/current-9/debian-9-openstack-amd64.qcow2
  - name: centos7
    ram: 1024
    cpu: 2
    extra_vol: {'vdb':'1G','vdc':'512M'}
    user: cloud-user
    config_vm: false
    become_method: sudo
    become_pass: password
    selinux_relabel: true
    ssh_pub: /tmp/centos_key.pub
    base_image: ~/Downloads/CentOS-7-x86_64-GenericCloud.qcow2
    upload_files:
      - local_path: files/somefile
        remote_path: /some/destination/folder/filename
scenario:
  playbooks:
    create: create.yml
    destroy: destroy.yml
    converge: converge.yml
provisioner:
  name: ansible
lint: |
    yamllint .
    ansible-lint .
    flake8 .
verifier:
  name: testinfra
  env:
    PYTHONWARNINGS: "ignore:.*U.*mode is deprecated:DeprecationWarning"
```

Changelog
---------

**2022.06.14:**

- Copy file(s) to virtual machine image option has been added: e.g. Canonical
[cloud images](https://cloud-images.ubuntu.com/releases/) (like Ubuntu18.04 or
20.04) ships without netplan config to bring up ethernet interfaces, so you can
upload them.
- Minor code style changes according to ansible-lint, removed deprecated
options (especially molecule.yml old structure).
