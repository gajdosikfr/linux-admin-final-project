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

### 7. Final Lynis Audit

Run another Lynis audit after applying the host security configuration:

```bash
sudo lynis audit system
```

Record the final Hardening Index and compare it with the initial audit.

### 8. Install Ansible

Install Ansible on the prepared host:

```bash
sudo apt install ansible -y
```

Ansible runs locally on the target VM and uses privilege escalation to install Docker and deploy the application stack.

## Application Deployment

The Ansible playbook installs Docker and Docker Compose, copies the project files to `/opt/final_project`, builds the custom web image, and starts the complete Compose stack.

Run the playbook from the repository root:

```bash
ansible-playbook ansible/playbook.yml -K
```

The `-K` option prompts for the sudo password required by `become: true`.

Run the same command a second time to verify idempotence:

```bash
ansible-playbook ansible/playbook.yml -K
```

The second run should complete with `changed=0`.

### Verify the deployment

Check the state of the Compose stack:

```bash
sudo docker compose -f /opt/final_project/compose.yaml ps
```

Verify the web service:

```bash
curl -fsS http://127.0.0.1:8092
```

Verify that the persistent Redis volume exists:

```bash
sudo docker volume inspect final_project_redis_data
```

The `deploy` user is intentionally not added to the `docker` group. Manual Docker commands are run with `sudo` because access to the Docker socket effectively provides root-level privileges.

The web service is available only locally at `http://127.0.0.1:8092`. The Uptime Kuma dashboard is intentionally available remotely at:

```text
http://SERVER_IP:3001
```

## Monitoring

Uptime Kuma is included in the Compose stack and is used to monitor the web service. Its dashboard is available at:

```text
http://SERVER_IP:3001
```

Create an HTTP(s) monitor with the following settings:

* **Friendly Name:** `Final Project Web`
* **URL:** `http://web`
* **Heartbeat Interval:** `10` seconds
* **Retries:** `0`

The URL uses the Compose service name `web`, which is resolved through the shared Docker network.

### Test outage detection

First verify that the monitor reports the service as `UP`.

Stop only the web service while keeping Uptime Kuma running:

```bash
sudo docker compose -f /opt/final_project/compose.yaml stop web
```

Uptime Kuma should detect the outage and change the monitor status to `DOWN`.

Start the web service again:

```bash
sudo docker compose -f /opt/final_project/compose.yaml start web
```

After the next successful check, the monitor should return to `UP`. This verifies the complete monitoring sequence:

```text
UP → DOWN → UP
```

In a production environment, Uptime Kuma would be configured to notify the server administrator by email after repeated failed checks.
