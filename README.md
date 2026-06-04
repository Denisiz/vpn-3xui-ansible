# vpn-3xui-ansible

Ansible wrapper for [Torotin/3x-ui_pro_Docker](https://github.com/Torotin/3x-ui_pro_Docker).

The repository prepares a fresh Debian/Ubuntu server and runs the Torotin installer in batch mode. It is intentionally a small Ansible project with roles, inventory examples, variables, and a repeatable launch flow.

## What It Does

- Creates a deploy user with SSH public key access.
- Gives that user passwordless sudo, if enabled.
- Prepares base packages needed before the installer Docker step.
- Clones `Torotin/3x-ui_pro_Docker`.
- Passes installer variables through environment variables.
- Runs `doctor`, `apt`, `env`, `docker`, `user`, `firewall`, `ssh`, `network`, `compose`, and `final`.
- Validates the compose stack after installation.
- Writes installer output on the server to `/var/log/torotin-3xui-install.log` by default.

Torotin's installer owns Docker installation/reinstall, firewall, SSH, sysctl, compose, and final checks. This Ansible project does not reimplement that logic; it makes it reproducible.

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

First run, usually with the provider's root password:

```bash
ansible-playbook playbooks/00-bootstrap-user.yml --ask-pass --ask-become-pass
```

Then update `inventory/hosts.ini` to use `ansible_user=deployer` and your private key.

Second run, key-only:

```bash
ansible-playbook playbooks/10-install-3xui.yml
```

## Important Variables

`group_vars/vpn.yml` is intentionally ignored by git because it contains secrets.

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

Torotin documents these as explicit host-changing actions. Use a fresh VPS unless you know exactly what is already running on the host.

## Troubleshooting

If the installer step fails while Ansible output is hidden, inspect the server-side log:

```bash
ssh -p 22 deployer@YOUR_SERVER_IP
sudo tail -n 200 /var/log/torotin-3xui-install.log
```

After SSH hardening has changed the port, use `-p 22022` or your configured `torotin_ssh_port`.

## Project Layout

```text
vpn-3xui-ansible/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   └── hosts.ini.example
├── group_vars/
│   └── vpn.yml.example
├── playbooks/
│   ├── 00-bootstrap-user.yml
│   └── 10-install-3xui.yml
└── roles/
    ├── bootstrap_user/
    ├── docker_host/
    ├── ssh_hardening/
    └── torotin_3xui/
```

## Notes

- If `torotin_ssh_public_key` is set, the Torotin installer switches SSH to public key auth.
- After SSH hardening, reconnect using `torotin_ssh_port`.
- The Torotin install summary is usually written on the server to `{{ torotin_project_src }}/script/install-state/install.summary`.
