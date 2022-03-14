# Ansible: Installation

This page will show how to install Ansible using Python pip.

## Virtual Environment

Create and use a python virtual environment:

```bash
python3 -m virtualenv -p python3 .venv
source .venv/bin/activate
```

## Install Ansible

Install Ansible with pip:

```bash
pip install 'ansible==4.4.0'
```

## Verify

Verify that ansible is installed:

```bash
ansible --version
```
