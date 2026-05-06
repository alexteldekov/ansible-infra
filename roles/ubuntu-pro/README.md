Ansible Role: Ubuntu Pro Subscription
=========

Ansible role to configure Ubuntu Pro subscription settings on Linux Servers.

Requirements
------------

The role does not require anything to run on Ubuntu servers.

Role Variables
--------------

Available variables are listed below, along with default values (see ```defaults/main.yml```):

``` yaml
ubuntu_token: "changeme"
```

```ubuntu_token```  The activation token to use with Ubuntu Pro

Role variables can be stored with the hosts.yaml file, or in the main variables file.

Dependencies
------------

None.

Example Playbook
----------------

``` yaml
    - hosts: servers
      roles:
         - role: mikepruett3.ubuntu-pro
```

License
-------

MIT

Author Information
------------------

Role created by [mikepruett3](https://github.com/mikepruett3) on [Github.com](https://github.com/mikepruett3/ansible-role-ubuntu-pro)
