
---

## Phase 1: อัปเดตระบบ (System Update)

> สิ่งแรกที่ต้องทำหลังติดตั้ง Ubuntu Server คือการอัปเดตแพตช์ความปลอดภัย

```bash
# ─── อัปเดต Package List & Upgrade ทุก Package ───────────────────────────
sudo apt update && sudo apt upgrade -y

# ─── เปิดใช้งาน Auto Security Updates ────────────────────────────────────
sudo dpkg-reconfigure --priority=low unattended-upgrades

# ─── ติดตั้ง unattended-upgrades (Security Patch อัตโนมัติ) ──────────────
sudo apt install -y unattended-upgrades apt-listchanges
```

**เหตุผล**

- ปิดช่องโหว่ด้านความปลอดภัยที่ถูกค้นพบล่าสุด
- แพตช์ kernel และ package สำคัญ
- ลบ package ที่ไม่จำเป็นออกเพื่อลด attack surface

---

## Phase 2: จัดการผู้ใช้และสิทธิ์ (User & Privilege Management)

```bash
# ─── สร้าง User ใหม่ ──────────────────────────────────────────────────────
# หลีกเลี่ยงการใช้ root โดยตรง
sudo adduser adminadmin

# ─── เพิ่มเข้ากลุ่ม sudo ──────────────────────────────────────────────────
sudo usermod -aG sudo adminadmin

# ─── ตรวจสอบสิทธิ์ ────────────────────────────────────────────────────────
grep sudo /etc/group
su - digitaldmb
sudo whoami
# ผลลัพธ์: root

# ─── ลบ User ใหม่ ──────────────────────────────────────────────────────
sudo userdel -f -r adminadmin

# ─── เพิ่ม User ใหม่ ──────────────────────────────────────────────────────
sudo adduser digitaldmb
sudo usermod -aG sudo digitaldmb
su - digitaldmb

# ─── เปลี่ยน Pass User ใหม่ ──────────────────────────────────────────────────────
sudo passwd root


```

---

## Phase 3: Harden SSH ให้ปลอดภัย

### 3.1 แก้ไข SSH Config บน Server

```bash
# ─── Backup Config เดิมก่อน ───────────────────────────────────────────────
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# ─── แก้ไข SSH Config ─────────────────────────────────────────────────────
sudo nano /etc/ssh/sshd_config
```

เปลี่ยนค่าต่อไปนี้:

```ini
# ปรับ Port หนี Bot
Port 717

# ความปลอดภัยพื้นฐาน (บังคับ)
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no

# จำกัด User (ป้องกันคนอื่นแอบสร้าง User แล้วเข้าเครื่องได้)
AllowUsers adminadmin

# ป้องกันการเดารหัสซ้ำๆ ใน Session เดียว
MaxAuthTries 3
```

```bash
# ─── ตรวจสอบ Syntax ก่อน Restart ─────────────────────────────────────────
sudo sshd -t
echo "Syntax OK"

# ─── Restart SSH ──────────────────────────────────────────────────────────
# ⚠️ อย่าปิด Terminal เดิมก่อนทดสอบ Terminal ใหม่!
sudo systemctl restart sshd
```

### 3.2 สร้าง SSH Key บนเครื่อง Local

```bash
# ─── รันบนเครื่อง Local ของคุณ ────────────────────────────────────────────
ssh-keygen -t ed25519 -C "your@email.com"

# ─── Copy Public Key ไปยัง Server ─────────────────────────────────────────
ssh-copy-id -p 2222 adminadmin@your-server-ip

# ─── ทดสอบ Login ด้วย Key ─────────────────────────────────────────────────
ssh -p 2222 adminadmin@your-server-ip
```

---

## Phase 4: ตั้งค่า UFW Firewall

```bash
# ─── ติดตั้ง UFW ──────────────────────────────────────────────────────────
sudo apt install -y ufw

# ─── ตรวจสอบ Rules ────────────────────────────────────────────────────────
sudo ufw status verbose

# ─── เปิดใช้งาน UFW ───────────────────────────────────────────────────────
sudo ufw enable

# ─── ตั้งค่า Default Policy ───────────────────────────────────────────────
sudo ufw default deny incoming
sudo ufw default allow outgoing

# ─── อนุญาต SSH Port ───────────────────────────────────────────────────────
sudo ufw allow 2222/tcp comment 'SSH Custom Port'

# ─── อนุญาต HTTP และ HTTPS ────────────────────────────────────────────────
sudo ufw allow 80/tcp  comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'




```

---

## Phase 5: ป้องกัน Brute Force ด้วย Fail2ban

```bash
# ─── ติดตั้ง Fail2Ban ─────────────────────────────────────────────────────
sudo apt install -y fail2ban

# ─── สร้าง Local Config ────────────────────────────────────────────────────
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
# ─── เริ่มใช้งาน Fail2Ban ─────────────────────────────────────────────────
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# ─── ตรวจสอบสถานะ ─────────────────────────────────────────────────────────
sudo systemctl status fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

## Phase 6: เปิด Mandatory Access Control (AppArmor)

```bash
# ─── ตรวจสอบสถานะ ─────────────────────────────────────────────────────────
sudo aa-status

# ─── เปิด AppArmor ────────────────────────────────────────────────────────
sudo systemctl enable apparmor
sudo systemctl start apparmor
sudo systemctl status apparmor
# ─── ตรวจสอบ ──────────────────────────────────────────────────────────────
sudo aa-status | less
sudo dmesg | grep -i apparmor
```

---

## Phase 7: เปิด Audit Logging (Security Audit)

**เหตุผล**
- บันทึก security event
- ใช้ forensic หลังถูกโจมตี

```bash
# ─── ติดตั้ง Audit System ─────────────────────────────────────────────────
sudo apt install -y auditd audispd-plugins

# ─── เปิดใช้งาน ───────────────────────────────────────────────────────────
sudo systemctl enable auditd
sudo systemctl start auditd

# ─── ตรวจสอบ ──────────────────────────────────────────────────────────────
sudo systemctl status auditd

# ─── เพิ่ม rule สำคัญ ─────────────────────────────────────────────────────
sudo nano /etc/audit/rules.d/hardening.rules
```

```ini
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
```

```bash
# ─── โหลด rule ────────────────────────────────────────────────────────────
sudo augenrules --load

# ─── ดูว่าใครแอบแก้ไฟล์ SSH Config: ────────────────────────────────────────
sudo ausearch -k ssh_changes

# ─── ดูรายงานสรุปประจำวัน (ใคร Login, รันคำสั่งอะไรบ้าง): ────────────────────────────────────────
sudo aureport -f -i
```

---

## Phase 8: File Integrity Monitoring

**เหตุผล**
- ตรวจสอบไฟล์ระบบถูกแก้ไข
- ป้องกัน backdoor

```bash
# ─── ติดตั้ง AIDE ─────────────────────────────────────────────────────────
sudo apt install -y aide
# เลือก Internet Site ที่ควรใส่: misp-wazuh.local หรือ saran.j@dms.mail.go.th
# ─── สร้าง database ───────────────────────────────────────────────────────
sudo aideinit
# ─── ตรวจสอบระบบ ──────────────────────────────────────────────────────────
sudo aide --check
```

---

## Phase 9: Rootkit & Malware Detection

**เหตุผล**
- ตรวจ rootkit
- ตรวจ backdoor

```bash
# ─── ติดตั้ง ──────────────────────────────────────────────────────────────
sudo apt install -y rkhunter chkrootkit
sudo rkhunter --update

# ─── ตรวจสอบระบบ ──────────────────────────────────────────────────────────
sudo rkhunter --check
sudo chkrootkit
```

---

## Phase 10: Kernel Hardening (sysctl)

**เหตุผล**
- ป้องกัน network spoofing
- ลด attack surface

```bash
# ─── สร้างไฟล์ config ─────────────────────────────────────────────────────
sudo nano /etc/sysctl.d/99-security.conf
```

```ini
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
# ─── Apply ────────────────────────────────────────────────────────────────
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

---

## Phase 11: จำกัดการใช้ USB Storage

**เหตุผล**
- ป้องกันการขโมยข้อมูลผ่าน USB

```bash
# ─── ป้องกัน Data Leakage ─────────────────────────────────────────────────
sudo nano /etc/modprobe.d/disable-usb-storage.conf
# เพิ่ม
install usb-storage /bin/true

# ─── reload ───────────────────────────────────────────────────────────────
sudo update-initramfs -u
```

---

## Phase 12: Secure Log Files (Immutable Logs)

**เหตุผล**
- log จะ append ได้อย่างเดียว attacker ลบไม่ได้

```bash
# ─── ป้องกัน attacker ลบ log ──────────────────────────────────────────────
sudo chattr +a /var/log/auth.log
sudo chattr +a /var/log/syslog

# ─── ตรวจสอบ ──────────────────────────────────────────────────────────────
lsattr /var/log/auth.log
```

---

## Phase 13: จำกัด Cron Access

**เหตุผล**
- ป้องกัน malware ใช้ cron persistence

```bash
# ─── อนุญาตเฉพาะ adminadmin ──────────────────────────────────────────────
sudo bash -c 'echo "adminadmin" > /etc/cron.allow'

# ─── บล็อก user อื่นทั้งหมด ───────────────────────────────────────────────
# cron.deny ต้องมีอยู่และว่างเปล่า (หรือระบุ user ที่ต้องการบล็อก)
sudo bash -c '> /etc/cron.deny'
```

---

## Phase 14: ตรวจสอบ Port และ Service

**เหตุผล**
- ปิด service ที่ไม่ใช้

```bash
# ─── ตรวจ port ────────────────────────────────────────────────────────────
sudo ss -tulnp

# ─── ตรวจ service ─────────────────────────────────────────────────────────
systemctl list-unit-files --type=service

# ─── ปิด service ที่ไม่ใช้ ────────────────────────────────────────────────
sudo systemctl disable --now cups
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now bluetooth
```

---

## Phase 15: Secure Sudo Logging

**เหตุผล**
- log ทุก command ที่ใช้ sudo

```bash
# ─── แก้ไข sudoers ────────────────────────────────────────────────────────
sudo visudo
# เพิ่มบรรทัดนี้:
# Defaults logfile="/var/log/sudo.log"

# ─── ตรวจสอบ ──────────────────────────────────────────────────────────────
sudo tail /var/log/sudo.log
```
