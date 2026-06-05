# vpn-3xui-ansible

Ansible wrapper for [Torotin/3x-ui_pro_Docker](https://github.com/Torotin/3x-ui_pro_Docker).

The repository prepares a fresh Debian/Ubuntu server and runs the Torotin installer in batch mode. It is a small Ansible project with roles, inventory examples, variables, and a repeatable launch flow.

## Detailed Guide

Russian step-by-step setup guide:

[docs/SETUP_RU.md](docs/SETUP_RU.md)

It covers:

- fresh VPS setup;
- WSL quirks;
- root password bootstrap;
- SSH key login;
- SSH port `22022`;
- service URLs;
- VPN client import;
- troubleshooting.

## What It Does

- Creates a deploy user with SSH public key access.
- Gives that user passwordless sudo, if enabled.
- Prepares base packages needed before the installer Docker step.
- Clones `Torotin/3x-ui_pro_Docker`.
- Passes installer variables through environment variables.
- Runs Torotin installer steps in a reproducible way.
- Fixes executable permissions for Torotin helper scripts where needed.
- Validates the compose stack after installation.
- Writes installer output on the server to `/var/log/torotin-3xui-install.log`.

Torotin's installer owns Docker installation/reinstall, firewall, SSH, sysctl, compose, and final checks. This Ansible project does not reimplement that logic; it makes it repeatable.

The installer repository under `torotin_project_src` is treated as disposable source code. By default, Ansible force-updates it before running installer steps.

## Requirements

Control machine:

- Ansible.
- SSH access to the target server.
- `ansible.posix` collection.

Target server:

- Debian/Ubuntu.
- `systemd` and `apt`.
- Root or sudo access for the first bootstrap.
- A domain DNS record pointing to the server.
- Recommended: 4 GB RAM, or 2 GB RAM with swap.

Install Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Quick Start

Copy examples:

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
cp group_vars/vpn.yml.example group_vars/vpn.yml
```

Edit:

```bash
vim inventory/hosts.ini
vim group_vars/vpn.yml
```

First run, usually as `root` with the provider password:

```bash
ANSIBLE_ROLES_PATH="$PWD/roles" \
ansible-playbook -i inventory/hosts.ini playbooks/00-bootstrap-user.yml --ask-pass
```

Then update `inventory/hosts.ini` to use `ansible_user=deployer` and your private key.

Second run, key-only:

```bash
ANSIBLE_ROLES_PATH="$PWD/roles" \
ansible-playbook -i inventory/hosts.ini playbooks/10-install-3xui.yml
```

After installation, SSH usually moves to `torotin_ssh_port`, default `22022`.

## Important Variables

`group_vars/vpn.yml` is ignored by git because it contains secrets.

```yaml
bootstrap_user: deployer
bootstrap_ssh_public_key: "ssh-ed25519 CHANGE_ME your_email@example.com"

torotin_webdomain: "vpn.example.com"
torotin_user_ssh: "{{ bootstrap_user }}"
torotin_pass_ssh: "CHANGE_ME_TEMP_PASSWORD"
torotin_ssh_port: 22022
torotin_ssh_public_key: "{{ bootstrap_ssh_public_key }}"

torotin_user_web: "admin"
torotin_pass_web: "CHANGE_ME_STRONG_PASSWORD"
```

For public repositories, prefer Ansible Vault for real secrets:

```bash
ansible-vault encrypt group_vars/vpn.yml
ansible-playbook playbooks/10-install-3xui.yml --ask-vault-pass
```

## Destructive Flags

Review these before running on a real server:

```yaml
torotin_destroy_docker_data: true
torotin_apply_firewall: true
torotin_apply_ssh: true
torotin_apply_network: true
```

Use a fresh VPS unless you know exactly what is already running on the host.

## Troubleshooting

Installer log:

```bash
ssh -p 22022 deployer@YOUR_SERVER_IP
sudo tail -n 200 /var/log/torotin-3xui-install.log
```

Summary and service URLs:

```bash
sudo cat /srv/3x-ui_pro_Docker/script/install-state/install.summary
```

Compose status:

```bash
sudo /srv/docker-proxy/compose.d/run-compose.sh ps
sudo /srv/docker-proxy/compose.d/run-compose.sh validate
```

## Project Layout

```text
vpn-3xui-ansible/
|-- ansible.cfg
|-- requirements.yml
|-- docs/
|   `-- SETUP_RU.md
|-- inventory/
|   `-- hosts.ini.example
|-- group_vars/
|   `-- vpn.yml.example
|-- playbooks/
|   |-- 00-bootstrap-user.yml
|   `-- 10-install-3xui.yml
`-- roles/
    |-- bootstrap_user/
    |-- docker_host/
    |-- ssh_hardening/
    `-- torotin_3xui/
```
