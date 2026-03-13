# 🛡️ Linux Hardening
---
## Ubuntu 24.04 LTS

> Server — Step by Step
```bash
ssh root@172.17.1.227
sudo su
sudo apt update && sudo apt upgrade -y
timedatectl set-timezone Asia/Bangkok
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo apt install -y unattended-upgrades apt-listchanges

sudo adduser digitalsaran
sudo usermod -aG sudo digitalsaran
exit

ssh digitalsaran@172.17.1.227
grep sudo /etc/group

# ─── Authentication Key-Pair ───────────────────────────
mkdir ~/.ssh && chmod 700 ~/.ssh
logout
ssh-keygen -t ed25519
cd .ssh
ls
# ─── Trenfers .pub to Server ───────────────────────────
scp $env:USERPROFILE/.ssh/id_ed25519.pub digitalsaran@172.17.1.227:~/.ssh/authorized_keys
ssh digitalsaran@172.17.1.227

# ─── แก้ไข SSH Config ─────────────────────────────────────────────
sudo nano /etc/ssh/sshd_config
# ปรับ Port หนี Bot
Port 717
# Disable ipv6
AddressFamily inet

# ความปลอดภัยพื้นฐาน (บังคับ)
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no

# จำกัด User (ป้องกันคนอื่นแอบสร้าง User แล้วเข้าเครื่องได้)
AllowUsers digitalsaran

# ป้องกันการเดารหัสซ้ำๆ ใน Session เดียว
MaxAuthTries 3

# ─── Check Syntax: (ถ้าไม่มี Error แสดงว่าผ่าน) ────────────────────────────
sudo sshd -t

# ─── Restart ──────────────────────────────────────────────────────────
systemctl list-units --type=service | grep ssh
sudo systemctl restart ssh
# ─── ติดตั้ง UFW ──────────────────────────────────────────────────────────

sudo apt install -y ufw
# ─── เปิดใช้งาน UFW ───────────────────────────────────────────────────────
sudo ufw enable

# ─── ตรวจสอบ Rules ────────────────────────────────────────────────────────
sudo ufw status verbose

# ─── ตั้งค่า Default Policy ───────────────────────────────────────────────
sudo ufw default deny incoming
sudo ufw default allow outgoing

# ─── อนุญาต SSH Port ───────────────────────────────────────────────────────
sudo ufw allow 717/tcp comment 'SSH Custom Port'

# ─── ทดสอบ SSH connect ───────────────────────────────────────────────────
sudo ss -tulnp | grep 717

# ─── อย่าเพิ่ง Logout ให้เปิด Terminal ใหม่ เข้าไปดูว่าเข้าได้จริงไหม ─────────────
ssh digitalsaran@172.17.1.227 -p 717
ssh -p 717 digitalsaran@172.17.1.227

```

# กรณีไม่ได้ ss -tulnp ไม่เห็น 717
 ```bash
sudo ss -tulnp | grep 717          # ตรวจสอบ
sudo systemctl stop ssh.socket     # หยุดตัวคุม Port 22
sudo systemctl stop ssh            # หยุด Service
sudo systemctl edit ssh.socket
# ─── เปลี่ยน port  ────────────────────────────
[Socket]
ListenStream=
ListenStream=0.0.0.0:717

sudo systemctl daemon-reload       # โหลดค่าที่แก้ใหม่
sudo systemctl start ssh.socket    # เริ่ม Socket ใหม่ (จะไปจับ Port 717 แทน)
sudo systemctl start ssh           # เริ่ม Service
sudo ss -tulnp | grep 717          # ตรวจสอบ และลอง login again
```
## ตั้งค่า UFW Firewall
```bash
# ─── อนุญาต HTTP และ HTTPS ────────────────────────────────────────────────
sudo ufw allow 80/tcp  comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# ─── ปิด IPV6 ────────────────────────────────────────────────
sudo nano /etc/default/ufw
# เปลี่ยน IPV6=yes
IPV6=no
# reload
sudo ufw reload
# ตรวจสอบหมายเลขบรรทัด
sudo ufw status numbered
# ลบบรรทัดที่เป็น (v6) ออก เช่น ถ้า v6 อยู่บรรทัดที่ 4, 5, 6
sudo ufw delete 6
sudo ufw delete 5
sudo ufw delete 4
# ตรวจสอบสถานะ
sudo ufw status
```

## การตรวจสอบ Log การเข้าใช้งาน (Auth & SSH)
```bash
# ─── ดูประวัติการ Login ล่าสุด ────────────────────────────
last -a
# ─── ดู Log การพยายามเข้าใช้งาน (Real-time) ────────────
# ดูความพยายาม Login ทั้งหมด รวมถึง SSH และการใช้ sudo
sudo tail -f /var/log/auth.log
# ─── ดูเฉพาะคนที่ Login ผิด (Failed Password) ──────────
sudo grep "Failed password" /var/log/auth.log
sudo tail -f /var/log/syslog
vim /var/log/fail2ban.log
```
## ป้องกัน Brute Force ด้วย Fail2ban

```bash
# ─── ติดตั้ง Fail2Ban ─────────────────────────────────────────────────────
sudo apt install -y fail2ban
systemctl status fail2ban
# ─── สร้าง Local Config ────────────────────────────────────────────────────
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
sudo fail2ban-client status
```
เปลี่ยนค่าต่อไปนี้:
```ini
sudo nano /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 717
filter = sshd
logpath = %(sshd_log)s
backend = %(sshd_backend)s
# จำนวนครั้งที่ลอง login ผิด
maxretry = 3
# ระยะเวลาที่นับการโจมตี
findtime = 10m
# ระยะเวลา ban
bantime = 24h
# โหมดตรวจจับรุนแรงขึ้น
mode = aggressive
# ใส่ IP เครื่องคุณที่ใช้ Login ประจำ (White-list) ไม่ให้โดนแบน
# [Localhost] [IPv6-Local] [IP-WiFi] [วง-LAN]
ignoreip = 127.0.0.1/8 ::1 122.155.142.197/32     
```

```bash
# ─── เริ่มใช้งาน Fail2Ban ─────────────────────────────────────────────────
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo systemctl restart fail2ban

# ─── ตรวจสอบสถานะ ─────────────────────────────────────────────────────────
sudo systemctl status fail2ban
sudo fail2ban-client status

# จะเห็นรายการ IP ในส่วน Banned IP list
sudo fail2ban-client status sshd

# ─── The "Fake Attack" Test  ──────────────────────────────────────────────────
# พิมพ์รหัสผ่านมั่วๆ ให้ครบตามจำนวน maxretry (ที่คุณตั้งไว้คือ 3 ครั้ง)
ssh -p 717 attack@172.17.1.227

# ตรวจสอบผลบน Server Currently banned จะเปลี่ยนจาก 0 เป็น 1
sudo fail2ban-client status sshd

# ใน Banned IP list จะปรากฏ เลข IP ของคุณ [sshd] Ban 1.2.3.4
sudo tail -f /var/log/fail2ban.log

# ดู Log ของ Fail2ban เพื่อดูว่ามีใครพยายามบุกรุก
sudo tail -f /var/log/fail2ban.log

# ถ้าเผลอทำตัวเองโดนแบน (Unban)
sudo fail2ban-client set sshd unbanip [เลข_IP_ที่โดนแบน]

# เริ่มเห็น IP แปลกๆ โดนแบน แล้วอยากรู้ว่าเป็นใคร (เช่น มาจากประเทศไหน) ให้ใช้คำสั่ง geoiplookup (ถ้าติดตั้งไว้) หรือใช้ curl
curl ipinfo.io/[เลข_IP_ที่โดนแบน]
```

## เปิด Mandatory Access Control (AppArmor)

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

## เปิด Audit Logging (Security Audit)

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

# ─── ลอง "แตะ" ไฟล์ SSH Config (แค่เปลี่ยนเวลาไฟล์ ไม่ต้องแก้เนื้อหา): ───────────────
sudo touch /etc/ssh/sshd_config

# ─── ดูว่าใครแอบแก้ไฟล์ SSH Config: ────────────────────────────────────────
sudo ausearch -k ssh_changes

# ─── ดูรายงานสรุปประจำวัน (ใคร Login, รันคำสั่งอะไรบ้าง): ────────────────────────────────────────
sudo aureport -f -i
```

# 🛡️🛡️ Linux Hardening ความปลอดภัยขั้นสุด
---

## ปิด Service ที่ไม่จำเป็น
```bash
# ─── Check Service  ─────────────────────────────────────────────────────
systemctl list-unit-files --type=service
systemctl list-units --type=service --state=running
netstat -tulpn

# ─── ปิด service ที่ไม่ใช้ ────────────────────────────────────────────────────
sudo systemctl disable bluetooth
sudo systemctl stop bluetooth

## ตรวจสอบ Port และ Service

# ─── ตรวจ port ────────────────────────────────────────────────────────────
sudo ss -tulnp

# ─── ตรวจ service ─────────────────────────────────────────────────────────
systemctl list-unit-files --type=service

# ─── ปิด service ที่ไม่ใช้ ────────────────────────────────────────────────
sudo systemctl disable --now cups
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now bluetooth
```


## File Integrity Monitoring

**เหตุผล**
- ตรวจสอบไฟล์ระบบถูกแก้ไข
- ป้องกัน backdoor

```bash
# ─── ติดตั้ง AIDE ─────────────────────────────────────────────────────────
sudo apt install -y aide
# เลือก Internet Site ที่ควรใส่: misp-wazuh.local หรือ saran.j@dms.mail.go.th
# ─── สร้าง database ───────────────────────────────────────────────────────
sudo aideinit
sudo ls -l /var/lib/aide/
# ───  ย้ายไฟล์ฐานข้อมูลให้ระบบเรียกใช้ได้ ──────────────────────────────────
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
# ─── ตรวจสอบระบบ ──────────────────────────────────────────────────────────
sudo aide --config /etc/aide/aide.conf --check
```

## Rootkit & Malware Detection

**เหตุผล**
- ตรวจ rootkit
- ตรวจ backdoor

```bash
# ─── ติดตั้ง ──────────────────────────────────────────────────────────────
sudo apt install -y rkhunter chkrootkit

# ─── ตรวจสอบระบบ ──────────────────────────────────────────────────────────
# 1. อัปเดตฐานข้อมูลมัลแวร์
sudo rkhunter --update
# 2. จดจำค่า Hash ของไฟล์ระบบปัจจุบัน (เพื่อไม่ให้มันแจ้งเตือนมั่ว)
sudo rkhunter --propupd
# 3. เริ่มทำการสแกน
sudo rkhunter --check --sk
sudo rkhunter --check
sudo chkrootkit
```
## Kernel Hardening (sysctl)

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
```
# ─── Apply ────────────────────────────────────────────────────────────────
sudo sysctl -p /etc/sysctl.d/99-security.conf

# ─── ldd /bin/ls | grep libc (การป้องกันหน่วยความจำ - ASLR) ───────────────────────────────────
ldd /bin/ls | grep libc

# ─── icmp_echo_ignore_broadcasts = 1 (ป้องกัน Smurf Attack) ───────────────────────────────────
cat /proc/sys/net/ipv4/icmp_echo_ignore_broadcasts

# ─── net.ipv4.tcp_syncookies = 1 (ป้องกัน SYN Flood)───────────────────────────────────
sudo sysctl net.ipv4.tcp_syncookies
```


---

## จำกัดการใช้ USB Storage

**เหตุผล**
- ป้องกันการขโมยข้อมูลผ่าน USB

```bash
# ─── ป้องกัน Data Leakage ─────────────────────────────────────────────────
sudo nano /etc/modprobe.d/disable-usb-storage.conf
# เพิ่ม
install usb-storage /bin/true

# ─── reload ───────────────────────────────────────────────────────────────
sudo update-initramfs -u

# วิธีทดสอบว่า USB ถูกปิดจริงไหม
lsmod | grep usb_storage
```

---

## Secure Log Files (Immutable Logs)
**เหตุผล**
- log จะ append ได้อย่างเดียว attacker ลบไม่ได้
```bash
# ─── ป้องกัน attacker ลบ log ──────────────────────────────────────────────
sudo chattr +a /var/log/auth.log
sudo chattr +a /var/log/syslog

# ─── ตรวจสอบ ──────────────────────────────────────────────────────────────
sudo lsattr /var/log/auth.log
# ลองเขียนต่อท้าย (ต้องผ่าน)
echo "Test append" | sudo tee -a /var/log/auth.log
# ลองลบเนื้อหาทิ้ง (ต้องพัง/Denied)
sudo truncate -s 0 /var/log/auth.log
# ลองลบไฟล์ทิ้ง (ต้องพัง/Denied)
sudo rm /var/log/auth.log
```
## จำกัด Cron Access
**เหตุผล**
- ป้องกัน malware ใช้ cron persistence
```bash
# ─── อนุญาตเฉพาะ digitalsaran@ ──────────────────────────────────────────────
sudo bash -c 'echo "digitalsaran" > /etc/cron.allow'

# ─── บล็อก user อื่นทั้งหมด ───────────────────────────────────────────────
# cron.deny ต้องมีอยู่และว่างเปล่า (หรือระบุ user ที่ต้องการบล็อก)
sudo bash -c '> /etc/cron.deny'
# เช็คสถานะปัจจุบัน
crontab -l
# ลองใช้ User อื่น (ถ้ามี)
sudo -u nobody crontab -e
```

## Secure Sudo Logging

**เหตุผล**
- log ทุก command ที่ใช้ sudo

```bash
# ─── แก้ไข sudoers ────────────────────────────────────────────────────────
sudo visudo
# เพิ่มบรรทัดนี้:
Defaults logfile="/var/log/sudo.log"

# ─── ตรวจสอบ ──────────────────────────────────────────────────────────────
sudo tail /var/log/sudo.log
```


