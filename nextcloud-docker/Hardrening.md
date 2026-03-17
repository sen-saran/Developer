# Nextcloud Docker Installation Guide
### Ubuntu 24.04 LTS + PostgreSQL + Cloudflare Tunnel

---

## Step 1: Hardening Ubuntu 24.04 LTS

### 1.1 Update System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
timedatectl set-timezone Asia/Bangkok
sudo adduser digitalsaran
sudo usermod -aG sudo digitalsaran
```

### 1.2 SSH Hardening
# ─── แก้ไข SSH Config ──────────────────────────────────
```bash
sudo nano /etc/ssh/sshd_config
```

แก้ไขค่าดังนี้:

```conf
Port 717                        # เปลี่ยน port จาก 22
AllowUsers digitalsaran         # จำกัด User
AddressFamily inet              # Disable ipv6
# ความปลอดภัยพื้นฐาน (บังคับ)
PermitRootLogin no              # ปิด root login
PasswordAuthentication yes      # เปิดไว้ยังไม่ใช้ SSH key
MaxAuthTries 3                  # จำกัดการ login ผิด
LoginGraceTime 20               # timeout 20 วินาที
X11Forwarding no                # ปิด X11
AllowTcpForwarding no           # ปิด TCP forwarding
# ─── SSH key ──────────────────────────────────
PasswordAuthentication no       # ปิดได้เมื่อใช้ SSH key
PermitEmptyPasswords no         # ปิดได้เมื่อใช้ SSH key

```

```bash
# ─── Check Syntax ────────
sudo sshd -t
# ─── Restart ─────────────
systemctl list-units --type=service | grep ssh
sudo systemctl restart ssh
sudo ufw allow 717/tcp
sudo ss -tulnp | grep 717
# ─── อย่าเพิ่ง Logout ให้ทดสอบก่อน ─────────
ssh -p 717 user@server
grep sudo /etc/group
exit
# ─── Authentication Key-Pair ───────────────────────────
mkdir ~/.ssh && chmod 700 ~/.ssh
logout
ssh-keygen -t ed25519
cd .ssh
ls
# ─── Trenfers .pub to Server ───────────────────────────
scp $env:USERPROFILE/.ssh/id_ed25519.pub digitalsaran@172.17.1.227:~/.ssh/authorized_keys
ssh digitalsaran@172.17.1.227
```

> ⚠️ อย่าปิด session เก่าจนกว่าจะทดสอบ SSH port ใหม่ได้สำเร็จ
### กรณีไม่ได้ ss -tulnp ไม่เห็น 717
 ```bash
sudo ss -tulnp | grep 717
sudo systemctl stop ssh.socket     # หยุดตัวคุม Port 22
sudo systemctl stop ssh            # หยุด Service
sudo systemctl edit ssh.socket
sudo systemctl daemon-reload       # โหลดค่าที่แก้ใหม่
sudo systemctl start ssh.socket    # เริ่ม Socket ใหม่ (จะไปจับ Port 717 แทน)
sudo systemctl start ssh           # เริ่ม Service
```

### 1.3 UFW Firewall

```bash
sudo apt install -y ufw

# ค่า default
sudo ufw default deny incoming
sudo ufw default allow outgoing

# อนุญาต port ที่จำเป็น
sudo ufw allow 717/tcp comment 'SSH Custom Port'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# เปิด firewall
sudo ufw enable

# ─── ตรวจสอบ Rules ───────────
sudo ufw status verbose

# ตรวจสอบ
sudo ufw status numbered

# ─── ปิด IPV6 ─────────────
sudo nano /etc/default/ufw
# เปลี่ยน IPV6=yes
IPV6=no

# reload
sudo ufw reload

# ตรวจสอบหมายเลขบรรทัด
sudo ufw status numbered

# ลบบรรทัดที่ไม่ได้ใช้
sudo ufw delete 6
sudo ufw delete 5
sudo ufw delete 4

# ตรวจสอบสถานะ
sudo ufw status
```

### การตรวจสอบ Log การเข้าใช้งาน (Auth & SSH)

```bash
# ─── ดูประวัติการ Login ล่าสุด ────────────
last -a

# ─── ดู Log การพยายามเข้าใช้งาน (Real-time) ────────────
# ดูความพยายาม Login ทั้งหมด รวมถึง SSH และการใช้ sudo
sudo tail -f /var/log/auth.log

# ─── ดูเฉพาะคนที่ Login ผิด (Failed Password) ──────────
sudo grep "Failed password" /var/log/auth.log
sudo tail -f /var/log/syslog
vim /var/log/fail2ban.log

### 1.4 ป้องกัน Brute Force ด้วย Fail2ban

```bash
sudo apt install -y fail2ban
systemctl status fail2ban

# สร้าง config
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
backend  = systemd

[sshd]
enabled  = true
port     = 717
logpath  = %(sshd_log)s

[nextcloud]
enabled  = true
port     = 80,443
logpath  = /data/nextcloud/files/data/nextcloud.log
maxretry = 5
bantime  = 3600
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# ตรวจสอบ
sudo fail2ban-client status

# จะเห็นรายการ IP ในส่วน Banned IP list
sudo fail2ban-client status sshd

###  ─── The "Fake Attack" Test ────────────────
# พิมพ์รหัสผ่านมั่วๆ ให้ครบตามจำนวน maxretry 3 ครั้ง
ssh -p 717 attack@172.17.1.227

#  Server Currently banned จะเปลี่ยนจาก 0 เป็น 1
sudo fail2ban-client status sshd

# Banned IP list [sshd] Ban 1.2.3.4
sudo tail -f /var/log/fail2ban.log

# ดู Log ของ Fail2ban เพื่อดูว่ามีใครพยายามบุกรุก
sudo tail -f /var/log/fail2ban.log

# ถ้าเผลอทำตัวเองโดนแบน (Unban)
sudo fail2ban-client set sshd unbanip [เลข_IP_ที่โดนแบน]

# เริ่มเห็น IP แปลกๆ โดนแบนมาจากประเทศไหน geoiplookup 
curl ipinfo.io/[เลข_IP_ที่โดนแบน]
geoiplookup pinfo.io/[เลข_IP_ที่โดนแบน]
geoiplookup ip
```

### 1.5 Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo apt install -y unattended-upgrades apt-listchanges
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

แก้ให้เปิดการอัปเดต security:

```conf
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
```

```bash
sudo dpkg-reconfigure -plow unattended-upgrades

# ตรวจสอบ
sudo systemctl status unattended-upgrades

# เปิด Mandatory Access Control (AppArmor)
# ─── ตรวจสอบสถานะ ───────────
sudo aa-status
# ─── เปิด AppArmor ────────────
sudo systemctl enable apparmor
sudo systemctl start apparmor
sudo systemctl status apparmor
# ─── ตรวจสอบ ─────────────────
sudo aa-status | less
sudo dmesg | grep -i apparmor

# เปิด Audit Logging (Security Audit)
**เหตุผล บันทึก security event forensic หลังถูกโจมตี**

# ─── ติดตั้ง Audit System ─────────────────
sudo apt install -y auditd audispd-plugins

# ─── เปิดใช้งาน ────────────────────────
sudo systemctl enable auditd
sudo systemctl start auditd

# ─── ตรวจสอบ ────────────────
sudo systemctl status auditd

# ─── เพิ่ม rule สำคัญ ──────────────────────────────
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
# ─── โหลด rule ───────────────────
sudo augenrules --load

# ─── ลอง "แตะ" ไฟล์ SSH Config ───────────
sudo touch /etc/ssh/sshd_config

# ─── ดูว่าใครแอบแก้ไฟล์ SSH Config: ────────────────────────────────────────
sudo ausearch -k ssh_changes

# ─── ดูรายงานสรุปประจำวัน (ใคร Login, รันคำสั่งอะไรบ้าง): ────────────────────────────────────────
sudo aureport -f -i

# ปิด Service ที่ไม่จำเป็น
# ─── Check Service  ─────────────────────────
systemctl list-unit-files --type=service
systemctl list-units --type=service --state=running
netstat -tulpn

# ─── ปิด service ที่ไม่ใช้ ────────────────────────────
sudo systemctl disable bluetooth
sudo systemctl stop bluetooth

## ตรวจสอบ Port และ Service

# ─── ตรวจ port ─────────────────
sudo ss -tulnp

# ─── ตรวจ service ─────────────────────────
systemctl list-unit-files --type=service

# ─── ปิด service ที่ไม่ใช้ ─────────────────────────
sudo systemctl disable --now cups
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now bluetooth

# File Integrity Monitoring ป้องกัน backdoor
# ─── ติดตั้ง AIDE ─────────────────────────────
sudo apt install -y aide
# ─── สร้าง database ─────────────────────────────
sudo aideinit
sudo ls -l /var/lib/aide/
# ───  ย้ายไฟล์ฐานข้อมูลให้ระบบเรียกใช้ได้ ───────────
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
# ─── ตรวจสอบระบบ ─────────────────────────────
sudo aide --config /etc/aide/aide.conf --check

# Rootkit & Malware Detection
# ─── ติดตั้ง ────────────────────────
sudo apt install -y rkhunter chkrootkit

# ─── ตรวจสอบระบบ ────────────────
# 1. อัปเดตฐานข้อมูลมัลแวร์
sudo rkhunter --update
# 2. จดจำค่า Hash ของไฟล์ระบบปัจจุบัน 
sudo rkhunter --propupd
# 3. เริ่มทำการสแกน
sudo rkhunter --check --sk
sudo rkhunter --check
sudo chkrootkit

# Kernel Hardening (sysctl)
# ─── สร้างไฟล์ config ───────────────────────
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
# จำกัดการใช้ USB Storage
# ─── ป้องกัน Data Leakage ──────────────────────────
sudo nano /etc/modprobe.d/disable-usb-storage.conf
# เพิ่ม
install usb-storage /bin/true

# ─── reload ────
sudo update-initramfs -u

# วิธีทดสอบว่า USB ถูกปิดจริงไหม
lsmod | grep usb_storage


# Secure Log Files ลบไม่ได้
# ─── ป้องกัน attacker ลบ log ──────────────
sudo chattr +a /var/log/auth.log
sudo chattr +a /var/log/syslog

# ─── ตรวจสอบ ───────
lsattr /var/log/auth.log
# ลองเขียนต่อท้าย (ต้องผ่าน)
echo "Test append" | sudo tee -a /var/log/auth.log
# ลองลบเนื้อหาทิ้ง (ต้องพัง/Denied)
sudo truncate -s 0 /var/log/auth.log
# ลองลบไฟล์ทิ้ง (ต้องพัง/Denied)
sudo rm /var/log/auth.log

# จำกัด Cron Access ป้องกัน malware ใช้ cron persistence
# ─── อนุญาตเฉพาะ digitalsaran@ ────────────────
sudo bash -c 'echo "digitalsaran@" > /etc/cron.allow'

# ─── บล็อก user อื่นทั้งหมด ───────────────────────
# cron.deny ต้องมีอยู่และว่างเปล่า (หรือระบุ user ที่ต้องการบล็อก)
sudo bash -c '> /etc/cron.deny'
# เช็คสถานะปัจจุบัน
crontab -l
# ลองใช้ User อื่น (ถ้ามี)
sudo -u nobody crontab -e

# Secure Sudo Logging log ทุก command ที่ใช้ sudo
# ─── แก้ไข sudoers ────────
sudo visudo
# เพิ่มบรรทัดนี้:
Defaults logfile="/var/log/sudo.log"

# ─── ตรวจสอบ ──────────────
sudo tail /var/log/sudo.log


# Disable Unused Services
# ดู service ที่รันอยู่
sudo systemctl list-units --type=service --state=running

# ปิด service ที่ไม่จำเป็น (ตัวอย่าง)
sudo systemctl disable --now snapd
sudo systemctl disable --now cups
sudo systemctl disable --now bluetooth

# ตรวจสอบ
sudo systemctl list-units --type=service --state=running
```

---
## Step 2: Setup Disk

### 2.1 ตรวจสอบ Disk

```bash
lsblk
df -h
```

### 2.2 Format Disk 1TB

```bash
sudo mkfs.ext4 /dev/vdb
```

### 2.3 Mount Disk

```bash
sudo mkdir -p /data
sudo mount /dev/vdb /data
```

### 2.4 Auto-mount หลัง Reboot

```bash
# หา UUID
sudo blkid /dev/vdb
```

```bash
sudo nano /etc/fstab
```

เพิ่มบรรทัด (แทน `YOUR-UUID` ด้วยค่าจริง):

```
UUID=YOUR-UUID /data ext4 defaults,nofail 0 2
```

```bash
# ทดสอบ
sudo mount -a
df -h | grep /data
```

```bash
# ทดสอบ
 sudo grep "Failed password" /var/log/auth.log | wc -l
 sudo journalctl -u sshd --since "30 days ago" | grep -i "filed" | head -20
```
