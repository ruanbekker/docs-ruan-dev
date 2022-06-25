# Ansible: Ad-Hoc

This page will show how to run ad-hoc commands with Ansible.

## Informational

To ensure connectivity:

```bash
ansible all -m ping
```

Gather Facts:

```bash
$ ansible all -m gather_facts
```

Gather Facts for only one host:

```bash
ansible all -m gather_facts --limit 10.0.0.2
```

List Hosts:

```bash
ansible all --list-hosts
```

## Commands

To run a sigle command (note that ansible runs them on 5 hosts at a time on default):

```bash
ansible all -m command -a hostname
```

If you want to run commands to more than 5 nodes in parallel, you can use for:

```
ansible all -m command -a hostname -f 10
```

When using the shell module when you are using arguments:

```
ansible all -m shell -a "uptime --help"
```

Update index repositories with apt and become the root user and prompt for the sudo password:

```
ansible all -m apt -a update_cache=true --become --ask-become-pass
```

Install packages with apt:

```
ansible all -m apt -a name=vim --become --ask-become-pass
```

Install the latest package with apt:

```
ansible all -m apt -a "name=vim state=latest" --become --ask-become-pass
```

Upgrade dist with apt:

```
ansible all -m apt -a upgrade=dist --become --ask-become-pass
```

Ensure a service is started with systemd:

```
ansible nginx -m systemd -a "name=nginx state=started" --become
```

## More Info

To list all the modules you can use `ansible-doc` and to list for ping:

```
ansible-doc -l | grep -i ping
```

For usage on it:

```bash
ansible-doc ping
```
