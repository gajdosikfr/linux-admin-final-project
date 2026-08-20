# Linux Administration Final Project

This project was originally created as the final assignment for a Linux Administration course. The documentation is written in English because the project will also be used as part of my GitHub portfolio and homelab documentation.

## Host Setup

### 1. Create project user

Create a dedicated non-root user for the project:

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
su - deploy
```

### 2. Clone the project repository

Install Git if it is not already available:

```bash
sudo apt update
sudo apt install git -y
```

Clone the project repository as the `deploy` user:

```bash
git clone https://github.com/gajdosikfr/linux-admin-final-project.git
cd linux-admin-final-project
```

### 3. Initial Lynis Audit

Install Lynis and run an initial security audit before applying system hardening:

```bash
sudo apt update
sudo apt install lynis -y
sudo lynis audit system
```
Record the initial Hardening Index for later comparison.

### 4. SSH Hardening

Set up SSH key authentication for the `deploy` user before disabling password authentication.

Create the SSH directory and `authorized_keys` file:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Generate an SSH key and paste the public key into `~/.ssh/authorized_keys`.

For example, when using PuTTY, generate an Ed25519 key with PuTTYgen, paste the public key into `~/.ssh/authorized_keys`, and verify login using the private key in a second PuTTY session before disabling password authentication.

Before applying the SSH hardening configuration, schedule an automatic rollback. If SSH access fails, the new configuration will be removed and the SSH service reloaded after five minutes:

```bash
sudo systemd-run --unit=ssh-rollback --on-active=5m /bin/sh -c 'rm -f /etc/ssh/sshd_config.d/99-hardening.conf && systemctl reload ssh'
```

Copy the provided SSH hardening configuration:

```bash
sudo cp 99-hardening.conf /etc/ssh/sshd_config.d/99-hardening.conf
```

Validate the SSH configuration before applying it:

```bash
sudo sshd -t
```

If no errors are returned, reload the SSH service:

```bash
sudo systemctl reload ssh
```

Verify the active SSH configuration:

```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication'
```

Expected values:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

Before closing the current SSH session, open a second session and verify that authentication using the SSH key works correctly.

After successful login in the second session, cancel the rollback timer:

```bash
sudo systemctl stop ssh-rollback.timer
```

### 5. Firewall

Install UFW and configure a default-deny firewall policy:

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
```

Before enabling the firewall, schedule an automatic rollback. If SSH access is blocked, UFW will be disabled after five minutes:

```bash
sudo systemd-run --unit=ufw-rollback --on-active=5m /usr/sbin/ufw disable
sudo ufw enable
```

Verify the firewall configuration:

```bash
sudo ufw status verbose
```

Before cancelling the rollback timer, open a new SSH session and verify that access still works.

After successful login, cancel the rollback timer:

```bash
sudo systemctl stop ufw-rollback.timer
```

The application web service is bound only to 127.0.0.1, so port 8092 is not publicly accessible. The Uptime Kuma dashboard is intentionally published on all host interfaces on port 3001. Docker manages this published port independently of UFW, and this exposure is intentional so the dashboard can be accessed remotely.

### 6. Fail2Ban

Install Fail2Ban to protect the SSH service against repeated failed authentication attempts:

```bash
sudo apt install fail2ban -y
```

Verify that the Fail2Ban service is enabled and running:

```bash
systemctl status fail2ban --no-pager
```

Check the active jails and verify the status of the SSH jail:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

The output should show one active jail named `sshd`. On this system, the default configuration allows five failed authentication attempts within ten minutes and then bans the offending IP address for ten minutes.
