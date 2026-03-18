---
title: "Production VPS Setup — Docker + GitLab Runner"
tags: [docker, gitlab-runner, ssh, vps, production]
author: "Imam Nc (Kacoon)"
created: 2026-03-18
---

# Production VPS Setup — Docker & GitLab Runner

Compact checklist for preparing a production VPS with Docker and a GitLab Runner. Commands are for Debian/Ubuntu; run as a user with `sudo` or as `root`.

## 1) Setup Docker

Install required packages, add Docker's GPG key and APT repository, then install Docker Engine.

```bash
# install dependencies
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# add Docker APT repo (architecture + signed-by key)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# update and install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# verify Docker service
sudo systemctl status docker

# add your user to the docker group (log out/in or use newgrp to refresh group)
sudo usermod -aG docker $USER
newgrp docker
```

Notes:

- `systemctl status docker` shows service state; use `sudo journalctl -u docker -f` to tail logs.
- After adding the user to `docker` group, either log out and back in, or run `newgrp docker` in the shell to apply.

## 2) Setup GitLab Runner

Install the official GitLab Runner package and register it with your GitLab instance.

```bash
# add GitLab Runner repository (Debian/Ubuntu)
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner

# register the runner (interactive)
sudo gitlab-runner register
```

Tips:

- During `register` you'll provide the GitLab URL, registration token, runner description, tags, and executor (e.g., `docker`, `shell`).
- For production use, prefer the `docker` executor with proper image and resource limits, or `docker+machine` for autoscaling.

## 3) Generate SSH Key (optional)

Generate an SSH key pair locally and copy the public key to the VPS for passwordless login.

```bash
# on your local machine
ssh-keygen -t rsa -b 4096 -C "efi-ssh-key"

# copy the public key to the VPS (replace user@host)
ssh-copy-id -i ~/.ssh/id_rsa.pub root@your.vps.ip.address
```

Notes:

- Use a descriptive `-C` comment to identify the key.
- After adding the public key to `~/.ssh/authorized_keys` on the VPS, you can SSH in without a password.

## Security and production reminders

- Harden SSH: disable root login if not needed, use key-based auth, change default SSH port, and enable fail2ban.
- Keep system and Docker packages updated (`sudo apt update && sudo apt upgrade`).
- Run GitLab Runner with least privileges and monitor runner logs.
- Consider firewall rules (ufw/iptables) to restrict access to needed ports only.

## 4) UFW (Uncomplicated Firewall)

Example UFW setup to allow SSH, HTTP, and HTTPS and enable basic rate limiting for SSH:

```bash
# install ufw if missing
sudo apt install -y ufw

# allow OpenSSH (replace with custom port if you changed it)
sudo ufw allow OpenSSH

# or to allow a custom port, e.g. 2222:
# sudo ufw allow 2222/tcp

# allow web traffic
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# rate-limit SSH to mitigate brute-force
sudo ufw limit OpenSSH

# enable firewall (confirm rules first)
sudo ufw status numbered
sudo ufw enable

# check status
sudo ufw status verbose
```

Notes:

- Use `ufw status numbered` to remove rules by number (`sudo ufw delete <num>`).
- If you change the SSH port, update both `ufw` rules and any `fail2ban` config.

## 5) fail2ban — basic setup

Install `fail2ban` and enable protections for SSH. This example creates a minimal jail configuration.

```bash
# install
sudo apt install -y fail2ban

# create local jail config (prevents editing packaged defaults)
sudo tee /etc/fail2ban/jail.d/local.conf > /dev/null <<'EOF'
[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 3600
EOF

# restart and check status
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
sudo fail2ban-client status sshd
```

Tips:

- `maxretry` controls how many failures before a ban; `bantime` is seconds banned.
- Use `fail2ban-client set sshd unbanip <IP>` to manually unban an address.
- Monitor `/var/log/fail2ban.log` and `/var/log/auth.log` for events.

---
