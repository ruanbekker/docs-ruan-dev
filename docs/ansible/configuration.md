# Ansible: Configuration

This page will show how to configure Ansible.

## Config File

You can find more information in their [documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_configuration.html), but for a short summary the `ansible.cfg` configuration file can be located in the current directory, `/etc/ansible/ansible.cfg`, `~/.ansible.cfg` or instructed with the environment variable `ANSIBLE_CONFIG`, see [this](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings-locations) for more instructions.

For this demonstration I will have in under my current working directory saved as `ansible.cfg`

This is what I usually store:

```
# https://github.com/ansible/ansible/blob/stable-2.9/examples/ansible.cfg
[defaults]
inventory = inventory.ini
deprecation_warnings=False
```

## Inventory

The inventory file is what is used to inform ansible where to reach your hosts, as `ansible.cfg` points to a file called `inventory.ini` in my current directory, the contents look lke this:

```
local]
my.laptop  ansible_connection=local

[webservers]
10.0.2.10  ansible_host=10.0.2.10 
10.0.2.11  ansible_host=10.0.2.11

[local:vars]
deprecation_warnings = False

[webservers:vars]
ansible_connection=ssh
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/ansible
```

To ensure connectivity:

```bash
ansible all -m ping
```

