# 🛡️ Linux Hardening Guide
## Ubuntu 24.04 LTS

Server Security Baseline — Step by Step

This guide provides a **practical security baseline** before installing services such as Docker, databases, or file‑sharing platforms.

Goal:
- Reduce attack surface
- Prevent unauthorized access
- Detect system tampering
- Protect sensitive data

---

# Phase 1: System Update

Update the system immediately after installation.

```bash
sudo apt update && sudo apt upgrade -y
```

Install automatic security updates.

```bash
sudo apt install -y unattended-upgrades apt-listchanges
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Reason

- Apply security patches
- Reduce vulnerability exposure
- Maintain patched kernel and libraries

---

# Phase 2: Remove Unnecessary Packages

Reduce attack surface by removing insecure or unused services.

```bash
sudo apt purge -y \
 telnet \
 rsh-client \
 rsh-redone-client \
 nis \
 talk \
 ftp

sudo apt autoremove -y
sudo apt autoclean
```

Reason

- Many legacy protocols transmit credentials in plaintext.

---

# Phase 3: User and Privilege Management

Create a non‑root administrative account.

```bash
sudo adduser adminadmin
sudo usermod -aG sudo adminadmin
```

Verify

```bash
grep sudo /etc/group
su - adminadmin
sudo whoami
```

Expected output

```
root
```

Reason

- Avoid direct root login
- Maintain accountability through user logs

---

# Phase 4: Password Policy

Install password complexity enforcement.

```bash
sudo apt install -y libpam-pwquality
```

Edit configuration

```bash
sudo nano /etc/security/pwquality.conf
```

Example

```
minlen = 12
minclass = 3
maxrepeat = 3
```

Reason

- Prevent weak passwords

---

# Phase 5: SSH Hardening

Backup configuration

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Edit configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Recommended settings

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
PubkeyAuthentication yes
AllowUsers adminadmin
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 60
MaxAuthTries 3
MaxSessions 5
```

Check configuration

```bash
sudo sshd -t
```

Restart SSH

```bash
sudo systemctl restart ssh
```

---

# Phase 6: SSH Key Authentication

Create key on local machine

```bash
ssh-keygen -t ed25519
```

Copy key

```bash
ssh-copy-id adminadmin@SERVER_IP
```

Test

```bash
ssh -p 2222 adminadmin@SERVER_IP
```

---

# Phase 7: Firewall Configuration

Install firewall

```bash
sudo apt install -y ufw
```

Configure policy

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow required ports

```bash
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Enable firewall

```bash
sudo ufw enable
```

Verify

```bash
sudo ufw status verbose
```

---

# Phase 8: Brute Force Protection

Install Fail2Ban

```bash
sudo apt install -y fail2ban
```

Create local config

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Enable service

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check

```bash
sudo fail2ban-client status
```

---

# Phase 9: Mandatory Access Control

Verify AppArmor status

```bash
sudo aa-status
```

Enable if needed

```bash
sudo systemctl enable apparmor
sudo systemctl start apparmor
```

Reason

- Restricts application privileges

---

# Phase 10: Audit Logging

Install audit framework

```bash
sudo apt install -y auditd audispd-plugins
```

Enable

```bash
sudo systemctl enable auditd
sudo systemctl start auditd
```

Example rule file

```bash
sudo nano /etc/audit/rules.d/hardening.rules
```

Example rules

```
-w /etc/passwd -p wa -k user_changes
-w /etc/shadow -p wa -k password_changes
-w /etc/sudoers -p wa -k sudo_changes
-w /etc/ssh/sshd_config -p wa -k ssh_changes
```

Load rules

```bash
sudo augenrules --load
```

---

# Phase 11: File Integrity Monitoring

Install AIDE

```bash
sudo apt install -y aide
```

Initialize

```bash
sudo aideinit
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

Check

```bash
sudo aide --check
```

---

# Phase 12: Rootkit Detection

Install tools

```bash
sudo apt install -y rkhunter chkrootkit
```

Update database

```bash
sudo rkhunter --update
```

Scan

```bash
sudo rkhunter --check
sudo chkrootkit
```

---

# Phase 13: Kernel Hardening

Create configuration

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

Example

```
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
kernel.randomize_va_space = 2
fs.suid_dumpable = 0
```

Apply

```bash
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

---

# Phase 14: Secure Log Files

Protect logs from deletion.

```bash
sudo chattr +a /var/log/auth.log
sudo chattr +a /var/log/syslog
```

Verify

```bash
lsattr /var/log/auth.log
```

---

# Phase 15: Restrict Cron Access

Create allow list

```bash
sudo nano /etc/cron.allow
```

Example

```
adminadmin
```

Remove deny file

```bash
sudo rm /etc/cron.deny
```

---

# Phase 16: Disable USB Storage

Prevent removable device data exfiltration.

```bash
sudo nano /etc/modprobe.d/disable-usb-storage.conf
```

Add

```
install usb-storage /bin/true
```

Update initramfs

```bash
sudo update-initramfs -u
```

---

# Phase 17: Secure Sudo Logging

Edit sudo configuration

```bash
sudo visudo
```

Add

```
Defaults logfile="/var/log/sudo.log"
```

---

# Phase 18: Service and Port Audit

Check open ports

```bash
sudo ss -tulnp
```

List services

```bash
systemctl list-unit-files --type=service
```

Disable unused services

```bash
sudo systemctl disable --now cups
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now bluetooth
```

---

# Phase 19: Time Synchronization

Accurate system time is important for logs and security monitoring.

```bash
sudo timedatectl set-ntp true
```

Verify

```bash
timedatectl
```

---

# Phase 20: Final Security Check

Run a quick system audit.

```bash
echo "Firewall"
sudo ufw status

echo "Fail2ban"
sudo fail2ban-client status

echo "Open Ports"
sudo ss -tulnp

echo "Audit"
sudo systemctl status auditd
```

---

# Hardening Summary

Security layers implemented

1. System patching
2. Access control
3. SSH hardening
4. Firewall protection
5. Brute force prevention
6. Mandatory access control
7. File integrity monitoring
8. Rootkit detection
9. Kernel security
10. Log protection

This baseline prepares the system securely before installing Docker, databases, or application platforms.

