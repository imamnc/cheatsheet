# 🚀 Production VPS Setup — Docker & GitLab Runner

> A step-by-step guide to harden and configure a production VPS with containerization, CI/CD, and security tools.

**Prerequisites:**

- Root or sudo access on a Debian/Ubuntu VPS
- Stable internet connection
- Basic familiarity with command-line tools

**Table of Contents:**

1. [� SSH Configuration](#1-generate-ssh-key-optional)
2. [🔥 UFW Firewall](#2-ufw-uncomplicated-firewall)
3. [🛡️ fail2ban Protection](#3-fail2ban--basic-setup)
4. [🐳 Docker Setup](#4-setup-docker)
5. [🔄 GitLab Runner](#5-setup-gitlab-runner)
6. [📋 Quick Reference](#quick-reference-checklist)

## 1) Generate SSH Key (optional)

🔑 Create a secure SSH key pair for passwordless authentication to your VPS.

**What this does:**

- Generates a 4096-bit RSA key with a descriptive comment
- Copies the public key to `~/.ssh/authorized_keys` on the remote VPS
- Enables secure, password-free SSH access

```bash
# on your local machine
ssh-keygen -t rsa -b 4096 -C "YOUR_SSH_KEY_COMMENT" -f ~/.ssh/id_rsa

# copy the public key to the VPS (replace user@host)
ssh-copy-id -i ~/.ssh/id_rsa.pub root@your.vps.ip.address
```

**📌 Notes:**

- Use a descriptive `-C` comment to identify the key.
- After adding the public key to `~/.ssh/authorized_keys` on the VPS, you can SSH in without a password.
- **⚠️ Security:** Keep your private key (`~/.ssh/id_rsa`) secure and never share it. Use `ssh-agent` for key management in production.

## 2) UFW (Uncomplicated Firewall)

🔥 Configure a stateful firewall to restrict traffic and protect against common attacks.

**What this does:**

- Enables a kernel-level firewall with intuitive rules
- Allows SSH, HTTP, and HTTPS traffic
- Rate-limits SSH to slow brute-force attempts

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

**📌 Notes:**

- Use `ufw status numbered` to remove rules by number (`sudo ufw delete <num>`).
- If you change the SSH port, update both `ufw` rules and any `fail2ban` config.
- **⚠️ Warning:** Always allow SSH before enabling UFW, or you'll lock yourself out. Check `sudo ufw status numbered` before `ufw enable`.

## 3) fail2ban — basic setup

🛡️ Deploy an intrusion-prevention system that monitors logs and bans repeat offenders.

**What this does:**

- Watches system logs (auth.log) for repeated SSH login failures
- Automatically blocks IPs after a set number of failed attempts
- Unbans IPs after a timeout window

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

**📌 Tips:**

- `maxretry` controls how many failures before a ban; `bantime` is ban duration in seconds (3600 = 1 hour).
- Use `fail2ban-client set sshd unbanip <IP>` to manually unban an impacted address.
- Monitor `/var/log/fail2ban.log` and `/var/log/auth.log` for blocked IPs and intrusion attempts.
- Adjust `maxretry` and `bantime` based on your environment; stricter settings = fewer false positives but harder legitimate access.

## 4) Setup Docker

🐳 Install and verify Docker Engine with secure GPG key validation. All commands are for Debian/Ubuntu.

**What this does:**

- Installs Docker CLI, daemon, and container runtime
- Adds Docker's official repository with GPG verification
- Configures user permissions for running Docker without `sudo` (optional)

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

**📌 Notes:**

- `systemctl status docker` shows service state; use `sudo journalctl -u docker -f` to tail logs.
- After adding the user to `docker` group, either log out and back in, or run `newgrp docker` in the shell to apply.
- **⚠️ Security:** Users in the `docker` group have elevated privileges equivalent to `sudo`; restrict membership.

## 5) Setup GitLab Runner

🔄 Deploy a GitLab Runner to execute CI/CD pipelines on this VPS.

**What this does:**

- Installs the GitLab Runner from the official repository
- Registers the runner with your GitLab instance for distributed job execution
- Prepares Docker-based job execution

```bash
# add GitLab Runner repository (Debian/Ubuntu)
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner

# register the runner (interactive)
sudo gitlab-runner register
```

**📌 Tips:**

- During `register` you'll provide the GitLab URL, registration token, runner description, tags, and executor (e.g., `docker`, `shell`).
- For production use, prefer the `docker` executor with proper image and resource limits, or `docker+machine` for autoscaling.
- Store the registration token securely; it grants CI/CD execution on this runner.

---

## Quick Reference Checklist

Use this checklist to verify your VPS is properly hardened:

```
✅ SSH key pair generated and copied to VPS
   - Test: ssh -i ~/.ssh/id_rsa root@your.vps.ip

✅ UFW firewall enabled with rules
   - Command: sudo ufw status verbose
   - Verify SSH, HTTP, HTTPS are allowed

✅ fail2ban service running
   - Command: sudo systemctl status fail2ban
   - Monitor bans: sudo fail2ban-client status sshd

✅ Docker installed and verified
   - Command: docker --version
   - Service: sudo systemctl status docker

✅ GitLab Runner installed and registered
   - Command: gitlab-runner --version
   - Status: gitlab-runner verify
```

---

## Security & Production Reminders

**🔒 SSH Hardening:**

- Disable root login: `PermitRootLogin no` in `/etc/ssh/sshd_config`
- Use SSH key-based auth only; disable password auth: `PasswordAuthentication no`
- Consider changing default SSH port (22) to reduce noise from scanners
- Restart SSH after config changes: `sudo systemctl restart ssh`

**📦 System Maintenance:**

- Keep system and Docker packages updated: `sudo apt update && sudo apt upgrade`
- Scheduled reboots to apply kernel updates and manage system health
- Monitor disk usage: `df -h`

**🏃 GitLab Runner:**

- Run GitLab Runner with least privileges; consider a dedicated user
- Monitor runner logs: `sudo journalctl -u gitlab-runner -f`
- Set resource limits on runner containers to prevent resource exhaustion

**🔥 Firewall & Monitoring:**

- Regularly audit firewall rules: `sudo ufw status numbered`
- Set up centralized log aggregation if running multiple servers
- Monitor fail2ban activity for attack patterns: `grep '\[sshd\]' /var/log/fail2ban.log | tail -20`

---

**Last updated:** 2026-03-18  
**Maintainer:** Imam Nc (Kacoon)
