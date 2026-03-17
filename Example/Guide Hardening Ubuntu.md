# 🛡️Linux Hardening Guide
## Ubuntu 24.04 LTS

> Server — Step by Step

---

## Phase 1: อัปเดตระบบ (System Update)
- สิ่งแรกที่ต้องทำหลังติดตั้ง Ubuntu Server คือการอัปเดตแพตช์ความปลอดภัย
> สิ่งแรกที่ต้องทำหลังติดตั้ง Ubuntu Server คือการอัปเดตแพตช์ความปลอดภัย
```bash
# ─── อัปเดต Package List Upgrade ทุก Package ─────────────────────────────
sudo apt update && sudo apt upgrade -y

# ─── ติดตั้ง unattended-upgrades (Security Patch อัตโนมัติ) ─
sudo apt install -y unattended-upgrades apt-listchanges

# ─── เปิดใช้งาน Auto Security Updates ───────────────
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

**เหตุผล**

- ปิดช่องโหว่ด้านความปลอดภัยที่ถูกค้นพบล่าสุด
- แพตช์ kernel และ package สำคัญ
- ลบ package ที่ไม่จำเป็นออกเพื่อลด attack surface

---

## Phase 2: จัดการผู้ใช้และสิทธิ์ (User & Privilege Management)

```bash
# ─── สร้าง User ใหม่ ─────────────────────────────────
# หลีกเลี่ยงการใช้ root โดยตรง
sudo adduser adminadmin

# ─── เพิ่มเข้ากลุ่ม sudo ─────────────────────────────
sudo usermod -aG sudo adminadmin

# ─── ตรวจสอบสิทธิ์ ───────────────────────────────────
grep sudo /etc/group
su - adminadmin
sudo whoami
# ผลลัพธ์: root

```

---

## Phase 3: Harden SSH ให้ปลอดภัย

### 3.1 แก้ไข SSH Config บน Server

```bash
# ─── Backup Config เดิมก่อน ──────────────────────────
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# ─── แก้ไข SSH Config ────────────────────────────────
sudo nano /etc/ssh/sshd_config
```

เปลี่ยนค่าต่อไปนี้:

```ini
# เปลี่ยน Port จาก 22 (ลด Bot Scanning attack )
Port 2222

# ปิด Root Login
PermitRootLogin no

# ปิด Password Login (ใช้ Key เท่านั้น)
PasswordAuthentication no
PermitEmptyPasswords no

# เปิด Key Authentication
PubkeyAuthentication yes

# จำกัด User ที่ SSH ได้
AllowUsers adminadmin

# ปิด X11 Forwarding
X11Forwarding no

# Timeout Settings
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 60

# ใช้ Protocol 2 เท่านั้น
Protocol 2

# จำกัดจำนวน Auth ที่ทำได้
MaxAuthTries 3
MaxSessions 5
```

```bash
# ─── ตรวจสอบ Syntax ก่อน Restart ────────────────────
sudo sshd -t
echo "Syntax OK"

# ─── Restart SSH ─────────────────────────────────────
# ⚠️ อย่าปิด Terminal เดิมก่อนทดสอบ Terminal ใหม่!
sudo systemctl restart sshd
```

### 3.2 สร้าง SSH Key บนเครื่อง Local

```bash
# ─── รันบนเครื่อง Local ของคุณ ───────────────────────
ssh-keygen -t ed25519 -C "your@email.com"

# ─── Copy Public Key ไปยัง Server ────────────────────
ssh-copy-id adminadmin@your-server-ip

# ─── ทดสอบ Login ด้วย Key ────────────────────────────
ssh adminadmin@your-server-ip

# ─── เปิด Terminal ใหม่ทดสอบ Login ──────────────────
ssh -p 2222 deploy@your-server-ip
```


---
## Phase 4: ตั้งค่า UFW Firewall

```bash
# ─── ติดตั้ง UFW ─────────────────────────────────────
sudo apt install -y ufw

# ─── ตั้งค่า Default Policy ──────────────────────────
sudo ufw default deny incoming
sudo ufw default allow outgoing

# ─── อนุญาต SSH Port ──────────────────────────────────
sudo ufw allow 2222/tcp comment 'SSH Custom Port'

# ─── อนุญาต HTTP และ HTTPS ───────────────────────────
sudo ufw allow 80,443/tcp
sudo ufw allow 80/tcp  comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# ─── เปิดใช้งาน UFW ──────────────────────────────────
sudo ufw enable

# ─── ตรวจสอบ Rules ───────────────────────────────────
sudo ufw status verbose

```

---

## Phase 5: ป้องกัน Brute Force ด้วย Fail2ban

```bash
# ─── ติดตั้ง Fail2Ban ─────────────────────────────────
sudo apt install -y fail2ban

# ─── สร้าง Local Config ───────────────────────────────
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

เปลี่ยนค่าต่อไปนี้:

```ini
[DEFAULT]
# Ban IP เป็นเวลา 1 ชั่วโมง
bantime  = 3600
# ช่วงเวลาตรวจสอบ 10 นาที
findtime = 600
# พยายาม Login ผิด 5 ครั้ง = Ban
maxretry = 5
# Backend
backend = systemd

[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 5
bantime  = 7200

[nginx-http-auth]
enabled = true

[nginx-limit-req]
enabled = true
```

```bash
# ─── เริ่มใช้งาน Fail2Ban ────────────────────────────
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# ─── ตรวจสอบสถานะ ────────────────────────────────────
sudo systemctl status fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---
## Phase 6: เปิด Mandatory Access Control (AppArmor)

```bash
# ─── ตรวจสอบสถานะ ───────────────────────────────────
sudo aa-status

# ─── เปิด AppArmor ───────────────────────────────────
sudo systemctl enable apparmor
sudo systemctl start apparmor

# ─── ตรวจสอบ  ──────────────────────────────────────
sudo aa-status | less
```

## Phase 7: เปิด Audit Logging (Security Audit)
**เหตุผล**
- บันทึก security event
- ใช้ forensic หลังถูกโจมตี

```bash
# ─── ติดตั้ง Audit System ───────────────────────────
sudo apt install -y auditd audispd-plugins

# ─── เปิดใช้งาน ─────────────────────────────────────
sudo systemctl enable auditd
sudo systemctl start auditd

# ─── ตรวจสอบ ──────────────────────────────────────
sudo systemctl status auditd

# ─── เพิ่ม rule สำคัญ ───────────────────────────────
sudo nano /etc/audit/rules.d/hardening.rules

# ตรวจสอบไฟล์ user
-w /etc/passwd -p wa -k user_changes
-w /etc/shadow -p wa -k password_changes
-w /etc/group -p wa -k group_changes

# ตรวจสอบ sudo
-w /etc/sudoers -p wa -k sudo_changes

# ตรวจสอบ ssh
-w /etc/ssh/sshd_config -p wa -k ssh_changes

# ตรวจสอบ docker
-w /usr/bin/docker -p x -k docker_usage

# ─── โหลด rule สำคัญ ─────────────────────────────
sudo augenrules --load
```

## Phase 8: File Integrity Monitoring
**เหตุผล**
- ตรวจสอบไฟล์ระบบถูกแก้ไข
- ป้องกัน backdoor

```bash
# ─── ติดตั้ง aide  System ───────────────────────────
sudo apt install -y aide

# ─── สร้าง database  ─────────────────────────────────────
sudo aideinit

# ─── ตรวจสอบระบบ ──────────────────────────────────────
sudo aide --check
```

## Phase 9: Rootkit & Malware Detection
**เหตุผล**
- ตรวจ rootkit
- ตรวจ backdoor

```bash
# ─── ติดตั้ง ───────────────────────────
sudo apt install -y rkhunter chkrootkit
sudo rkhunter --update

# ─── ตรวจสอบระบบ ──────────────────────────────────────
sudo rkhunter --check
sudo chkrootkit
```

## Phase 10: Kernel Hardening (sysctl)
**เหตุผล**
- ตรวจ ป้องกัน network spoofing
- ตรวจ ลด attack surface

```bash
# ─── สร้างไฟล์ config ───────────────────────────
sudo nano /etc/sysctl.d/99-security.conf
```
```INI
# ป้องกัน SYN Flood
net.ipv4.tcp_syncookies = 1

# ป้องกัน IP Spoofing
net.ipv4.conf.all.rp_filter = 1

# ปิด ICMP redirect
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# ปิด source routing
net.ipv4.conf.all.accept_source_route = 0

# ป้องกัน SMURF attack
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Memory protection
kernel.randomize_va_space = 2

# ปิด SUID core dump
fs.suid_dumpable = 0
```
```bash
# ─── Apply ───────────────────────────
sudo sysctl -p /etc/sysctl.d/99-security.conf
```
## Phase 11: จำกัดการใช้ USB Storage
**เหตุผล**
- ป้องกันการขโมยข้อมูลผ่าน USB

```bash
# ─── ป้องกัน Data Leakage ───────────────────────────
sudo nano /etc/modprobe.d/disable-usb-storage.conf
# เพิ่ม
install usb-storage /bin/true
# reload
sudo update-initramfs -u
```

## Phase 12: Secure Log Files (Immutable Logs)
**เหตุผล**
- log จะ append ได้อย่างเดียว attacker ลบไม่ได้

```bash
# ─── ป้องกัน attacker ลบ log ───────────────────────────
sudo chattr +a /var/log/auth.log
sudo chattr +a /var/log/syslog
ตรวจสอบ
lsattr /var/log/auth.log
```


## Phase 13: จำกัด Cron Access
**เหตุผล**
- ป้องกัน malware ใช้ cron persistence

```bash
sudo nano /etc/cron.allow
# เพิ่ม
adminadmin
# ปิด user อื่น
sudo rm /etc/cron.deny
```

## Phase 14: ตรวจสอบ Port และ Service
**เหตุผล**
- ปิด service ที่ไม่ใช้
```bash
# ตรวจ port
sudo ss -tulnp
# ตรวจ service
systemctl list-unit-files --type=service
# ปิด service ที่ไม่ใช้
sudo systemctl disable --now cups
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now bluetooth
```

## Phase 15: Secure Sudo Logging
**เหตุผล**
- log ทุก command ที่ใช้ sudo

```bash
# แก้ไข
sudo visudo
# เพิ่ม
Defaults logfile="/var/log/sudo.log"
# ตรวจสอบ
sudo tail /var/log/sudo.log
```
