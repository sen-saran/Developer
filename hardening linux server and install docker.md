# 🛡️ Linux Server Hardening & Docker Installation Guide

> คู่มือ Hardening Ubuntu 24 LTS และติดตั้ง Docker  
> สำหรับ Production Server — Step by Step

---

## 📋 สารบัญ

| Phase | หัวข้อ |
|:---:|:---|
| **1** | [อัปเดตระบบ](#phase-1-อัปเดตระบบ) |
| **2** | [สร้าง Non-root User](#phase-2-สร้าง-non-root-user) |
| **3** | [Harden SSH](#phase-3-harden-ssh) |
| **4** | [ติดตั้ง Fail2Ban](#phase-4-ติดตั้ง-fail2ban) |
| **5** | [ตั้งค่า UFW Firewall](#phase-5-ตั้งค่า-ufw-firewall) |
| **6** | [Kernel Security (sysctl)](#phase-6-kernel-security-sysctl) |
| **7** | [ลบ Package ที่ไม่จำเป็น](#phase-7-ลบ-package-ที่ไม่จำเป็น) |
| **8** | [ติดตั้ง Docker](#phase-8-ติดตั้ง-docker) |
| **9** | [ตั้งค่า Docker Security](#phase-9-ตั้งค่า-docker-security) |
| **10** | [ตรวจสอบสุดท้าย](#phase-10-ตรวจสอบสุดท้าย) |

---

## Phase 1: อัปเดตระบบ

```bash
# ─── อัปเดต Package List ─────────────────────────────
sudo apt update

# ─── Upgrade ทุก Package ─────────────────────────────
sudo apt upgrade -y

# ─── ติดตั้ง unattended-upgrades (Security Patch อัตโนมัติ) ─
sudo apt install -y unattended-upgrades apt-listchanges

# ─── เปิดใช้งาน Auto Security Updates ───────────────
sudo dpkg-reconfigure --priority=low unattended-upgrades

# ─── ตรวจสอบสถานะ ────────────────────────────────────
sudo systemctl status unattended-upgrades

echo "✅ อัปเดตระบบเสร็จแล้ว"
```

---

## Phase 2: สร้าง Non-root User

```bash
# ─── สร้าง User ใหม่ ─────────────────────────────────
# แทนที่ 'deploy' ด้วยชื่อที่ต้องการ
sudo adduser deploy

# ─── เพิ่มเข้ากลุ่ม sudo ─────────────────────────────
sudo usermod -aG sudo deploy

# ─── ตรวจสอบสิทธิ์ ───────────────────────────────────
su - deploy
sudo whoami
# ผลลัพธ์: root

exit  # กลับมาเป็น root
echo "✅ สร้าง User deploy แล้ว"
```

---

## Phase 3: Harden SSH

### 3.1 สร้าง SSH Key บนเครื่อง Local

```bash
# ─── รันบนเครื่อง Local ของคุณ ───────────────────────
ssh-keygen -t ed25519 -C "your@email.com"

# ─── Copy Public Key ไปยัง Server ────────────────────
ssh-copy-id deploy@your-server-ip

# ─── ทดสอบ Login ด้วย Key ────────────────────────────
ssh deploy@your-server-ip
```

### 3.2 แก้ไข SSH Config บน Server

```bash
# ─── Backup Config เดิมก่อน ──────────────────────────
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# ─── แก้ไข SSH Config ────────────────────────────────
sudo nano /etc/ssh/sshd_config
```

เปลี่ยนค่าต่อไปนี้:

```ini
# เปลี่ยน Port จาก 22 (ลด Bot Scanning)
Port 2222

# ปิด Root Login
PermitRootLogin no

# ปิด Password Login (ใช้ Key เท่านั้น)
PasswordAuthentication no
PermitEmptyPasswords no

# เปิด Key Authentication
PubkeyAuthentication yes

# จำกัด User ที่ SSH ได้
AllowUsers deploy

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

# ─── เปิด Terminal ใหม่ทดสอบ Login ──────────────────
ssh -p 2222 deploy@your-server-ip

echo "✅ Harden SSH เสร็จแล้ว"
```

---

## Phase 4: ติดตั้ง Fail2Ban

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
maxretry = 3
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

echo "✅ ติดตั้ง Fail2Ban เสร็จแล้ว"
```

---

## Phase 5: ตั้งค่า UFW Firewall

```bash
# ─── ติดตั้ง UFW ─────────────────────────────────────
sudo apt install -y ufw

# ─── ตั้งค่า Default Policy ──────────────────────────
sudo ufw default deny incoming
sudo ufw default allow outgoing

# ─── อนุญาต SSH Port ──────────────────────────────────
sudo ufw allow 2222/tcp comment 'SSH Custom Port'

# ─── อนุญาต HTTP และ HTTPS ───────────────────────────
sudo ufw allow 80/tcp  comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# ─── เปิดใช้งาน UFW ──────────────────────────────────
sudo ufw enable

# ─── ตรวจสอบ Rules ───────────────────────────────────
sudo ufw status verbose

echo "✅ ตั้งค่า Firewall เสร็จแล้ว"
```

---

## Phase 6: Kernel Security (sysctl)

```bash
# ─── สร้าง Security Config ───────────────────────────
sudo nano /etc/sysctl.d/99-security.conf
```

```ini
# ─── Network Security ─────────────────────────────────

# ป้องกัน SYN Flood Attack
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_synack_retries = 2

# ป้องกัน IP Spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# ปฏิเสธ ICMP Redirect
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# ปิด Source Routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log Martian Packets
net.ipv4.conf.all.log_martians = 1

# ป้องกัน SMURF Attack
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# ป้องกัน Time-Wait Assassination
net.ipv4.tcp_rfc1337 = 1

# ─── Memory Security ──────────────────────────────────

# ASLR — สุ่ม Memory Address
kernel.randomize_va_space = 2

# ปิด Core Dump ที่มี SUID
fs.suid_dumpable = 0

# ─── Docker Support ───────────────────────────────────

# จำเป็นสำหรับ Docker networking
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

```bash
# ─── Apply ทันที ─────────────────────────────────────
sudo sysctl -p /etc/sysctl.d/99-security.conf

# ─── ตรวจสอบ ─────────────────────────────────────────
sudo sysctl net.ipv4.tcp_syncookies
sudo sysctl kernel.randomize_va_space

echo "✅ Kernel Security เสร็จแล้ว"
```

---

## Phase 7: ลบ Package ที่ไม่จำเป็น

```bash
# ─── ลบ Service ที่ไม่ใช้ ────────────────────────────
sudo apt purge -y \
  telnet \
  rsh-client \
  ftp \
  nfs-common \
  nis \
  talk \
  finger 2>/dev/null || true

# ─── ปิด Service ที่ไม่จำเป็น ────────────────────────
sudo systemctl disable --now bluetooth 2>/dev/null || true
sudo systemctl disable --now avahi-daemon 2>/dev/null || true
sudo systemctl disable --now cups 2>/dev/null || true

# ─── Autoremove ──────────────────────────────────────
sudo apt autoremove -y
sudo apt autoclean

# ─── ตรวจสอบ Port ที่เปิดอยู่ ────────────────────────
echo ""
echo "=== Port ที่เปิดอยู่ทั้งหมด ==="
sudo ss -tlnp

echo "✅ ลบ Package ที่ไม่จำเป็นแล้ว"
```

---

## Phase 8: ติดตั้ง Docker

### 8.1 ลบ Docker เก่า (ถ้ามี)

```bash
sudo apt purge -y \
  docker \
  docker-engine \
  docker.io \
  containerd \
  runc \
  docker-compose 2>/dev/null || true

sudo apt autoremove -y
```

### 8.2 ติดตั้ง Docker Engine

```bash
# ─── ติดตั้ง Dependencies ────────────────────────────
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# ─── เพิ่ม Docker GPG Key ────────────────────────────
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# ─── เพิ่ม Docker Repository ─────────────────────────
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ─── ติดตั้ง Docker ───────────────────────────────────
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# ─── เพิ่ม User เข้ากลุ่ม docker ────────────────────
sudo usermod -aG docker deploy
newgrp docker

# ─── เปิด Auto-start ─────────────────────────────────
sudo systemctl enable docker
sudo systemctl start docker

# ─── ทดสอบ ────────────────────────────────────────────
docker run hello-world
docker compose version

echo "✅ ติดตั้ง Docker เสร็จแล้ว"
```

---

## Phase 9: ตั้งค่า Docker Security

### 9.1 ตั้งค่า Docker Daemon

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "icc": false,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "storage-driver": "overlay2",
  "features": {
    "buildkit": true
  }
}
```

```bash
# ─── Restart Docker ───────────────────────────────────
sudo systemctl restart docker

# ─── ตรวจสอบ ─────────────────────────────────────────
docker info | grep -E "Storage|Logging|Security|no-new"

echo "✅ ตั้งค่า Docker Security เสร็จแล้ว"
```

### 9.2 รัน Docker Bench Security Audit

```bash
# ตรวจสอบ Docker Config ตาม CIS Benchmark
docker run --rm --net host --pid host \
  --userns host --cap-add audit_control \
  -v /etc:/etc:ro \
  -v /lib/systemd/system:/lib/systemd/system:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security 2>&1 | tee docker-bench.txt

# ดูเฉพาะ WARN และ FAIL
grep -E "\[WARN\]|\[FAIL\]" docker-bench.txt
```

---

## Phase 10: ตรวจสอบสุดท้าย

```bash
echo "============================================"
echo "  ตรวจสอบ Linux Hardening & Docker"
echo "============================================"

# ─── OS Version ───────────────────────────────────────
echo ""
echo "=== OS Version ==="
cat /etc/os-release | grep -E "^NAME|^VERSION"
uname -r

# ─── Users ────────────────────────────────────────────
echo ""
echo "=== Users ที่มี sudo ==="
getent group sudo

# ─── SSH ──────────────────────────────────────────────
echo ""
echo "=== SSH Config ==="
sudo sshd -T | grep -E "^port|^permitrootlogin|^passwordauthentication|^pubkeyauthentication"

# ─── Fail2Ban ─────────────────────────────────────────
echo ""
echo "=== Fail2Ban Status ==="
sudo fail2ban-client status

# ─── UFW ──────────────────────────────────────────────
echo ""
echo "=== UFW Rules ==="
sudo ufw status verbose

# ─── Port ─────────────────────────────────────────────
echo ""
echo "=== Open Ports ==="
sudo ss -tlnp

# ─── Docker ───────────────────────────────────────────
echo ""
echo "=== Docker Version ==="
docker --version
docker compose version

echo ""
echo "=== Docker Info ==="
docker info | grep -E "Server Version|Storage Driver|Logging Driver|Security"

# ─── Services Running ─────────────────────────────────
echo ""
echo "=== Services ที่รันอยู่ ==="
sudo systemctl list-units --type=service --state=running \
  | grep -E "docker|fail2ban|ssh|ufw"

# ─── Disk ─────────────────────────────────────────────
echo ""
echo "=== Disk Usage ==="
df -h /

# ─── RAM ──────────────────────────────────────────────
echo ""
echo "=== Memory ==="
free -h

echo ""
echo "============================================"
echo "  ✅ Hardening & Docker พร้อมใช้งานแล้ว!"
echo "============================================"
```

---

## Hardening Checklist

| หัวข้อ | รายการ | สถานะ |
|:---|:---|:---:|
| **System** | อัปเดต OS ครบถ้วน | ☐ |
| **System** | เปิด Auto Security Updates | ☐ |
| **User** | สร้าง Non-root User | ☐ |
| **User** | ปิด Root Login | ☐ |
| **SSH** | เปลี่ยน Port จาก 22 | ☐ |
| **SSH** | ใช้ SSH Key เท่านั้น | ☐ |
| **SSH** | ปิด Password Authentication | ☐ |
| **SSH** | จำกัด AllowUsers | ☐ |
| **Fail2Ban** | ติดตั้งและเปิดใช้งาน | ☐ |
| **Firewall** | UFW Default Deny | ☐ |
| **Firewall** | เปิดเฉพาะ Port ที่จำเป็น | ☐ |
| **Kernel** | sysctl Security Parameters | ☐ |
| **Package** | ลบ Service ที่ไม่จำเป็น | ☐ |
| **Docker** | ติดตั้ง Docker Engine | ☐ |
| **Docker** | ตั้งค่า daemon.json | ☐ |
| **Docker** | รัน Docker Bench Security | ☐ |

---

## สรุป Port ที่เปิด

| Port | Protocol | Service | หมายเหตุ |
|:---:|:---:|:---|:---|
| 2222 | TCP | SSH | เปลี่ยนจาก 22 |
| 80 | TCP | HTTP | Redirect → HTTPS |
| 443 | TCP | HTTPS | Web Application |

> 📝 **หมายเหตุ:**
> - แทนที่ `deploy` ด้วยชื่อ User จริงของคุณ
> - แทนที่ `your-server-ip` ด้วย IP จริงของ Server
> - ทดสอบ Login ด้วย Terminal ใหม่ **ก่อน** ปิด Session เดิมเสมอ  
>   เพื่อป้องกัน Lock ตัวเองออกจาก Server
