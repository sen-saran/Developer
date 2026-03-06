# 🧹 ลบ MISP และล้างเครื่อง Ubuntu 24 ให้เหมือนใหม่

> คู่มือลบ MISP ที่ติดตั้งแบบ Native (ไม่ใช่ Docker) บน Ubuntu 24  
> และล้างระบบให้พร้อมสำหรับการติดตั้งใหม่ Step by Step

---

## 📋 สารบัญ

| Phase | หัวข้อ |
|:---:|:---|
| **0** | [Backup ข้อมูลก่อนลบ](#phase-0-backup-ก่อนเริ่ม) |
| **1** | [หยุด MISP Services ทั้งหมด](#phase-1-หยุด-misp-services-ทั้งหมด) |
| **2** | [ลบ MISP Application](#phase-2-ลบ-misp-application) |
| **3** | [ลบ Database](#phase-3-ลบ-database) |
| **4** | [ลบ Packages ที่ติดตั้งสำหรับ MISP](#phase-4-ลบ-packages-ที่ติดตั้งสำหรับ-misp) |
| **5** | [ลบ SSL Certificates](#phase-5-ลบ-ssl-certificates) |
| **6** | [ล้าง System](#phase-6-ล้าง-system) |
| **7** | [ล้าง Firewall Rules](#phase-7-ล้าง-firewall-rules) |
| **8** | [ตรวจสอบว่าล้างหมดแล้ว](#phase-8-ตรวจสอบว่าล้างหมดแล้ว) |
| **9** | [อัปเดตระบบให้พร้อมใช้งานใหม่](#phase-9-อัปเดตระบบให้พร้อมใช้งานใหม่) |
| **10** | [Reboot และตรวจสอบสุดท้าย](#phase-10-reboot-และตรวจสอบสุดท้าย) |

---

## ⚠️ คำเตือนก่อนเริ่ม

```
🔴 Backup ข้อมูลสำคัญก่อนทุกครั้ง
🔴 ขั้นตอนนี้ลบข้อมูลถาวร กู้คืนไม่ได้
🔴 ทำบน Server ที่ต้องการล้างจริงเท่านั้น
```

---

## Phase 0: Backup ก่อนเริ่ม

```bash
# ─── Backup MISP Database ────────────────────────────
sudo mysqldump -u root misp > ~/misp_backup_$(date +%Y%m%d).sql

# ─── Backup MISP Config ──────────────────────────────
sudo cp -r /var/www/MISP/app/Config ~/misp_config_backup

# ─── Backup MISP Files ───────────────────────────────
sudo cp -r /var/www/MISP/app/files ~/misp_files_backup

# ─── ตรวจสอบว่า Backup สำเร็จ ───────────────────────
ls -lh ~/misp_backup_*.sql
ls -lh ~/misp_config_backup
echo "✅ Backup เสร็จแล้ว"
```

---

## Phase 1: หยุด MISP Services ทั้งหมด

```bash
# ─── หยุด MISP Workers ───────────────────────────────
sudo systemctl stop misp-workers 2>/dev/null || true
sudo systemctl disable misp-workers 2>/dev/null || true

# ─── หยุด MISP Modules (ถ้าติดตั้ง) ─────────────────
sudo systemctl stop misp-modules 2>/dev/null || true
sudo systemctl disable misp-modules 2>/dev/null || true

# ─── หยุด Apache2 ────────────────────────────────────
sudo systemctl stop apache2
sudo systemctl disable apache2

# ─── หยุด Redis ──────────────────────────────────────
sudo systemctl stop redis-server
sudo systemctl disable redis-server

# ─── หยุด MySQL / MariaDB ────────────────────────────
sudo systemctl stop mysql 2>/dev/null || \
sudo systemctl stop mariadb 2>/dev/null || true
sudo systemctl disable mysql 2>/dev/null || \
sudo systemctl disable mariadb 2>/dev/null || true

# ─── ตรวจสอบว่าหยุดแล้ว ─────────────────────────────
sudo systemctl status apache2 mysql redis-server 2>/dev/null || true
echo "✅ Services หยุดทั้งหมดแล้ว"
```

---

## Phase 2: ลบ MISP Application

```bash
# ─── ลบโฟลเดอร์ MISP ─────────────────────────────────
sudo rm -rf /var/www/MISP

# ─── ลบ MISP Systemd Services ────────────────────────
sudo rm -f /etc/systemd/system/misp-workers.service
sudo rm -f /etc/systemd/system/misp-modules.service
sudo rm -f /etc/systemd/system/misp-scheduler.service

# ─── ลบ Apache Virtual Host Config ──────────────────
sudo rm -f /etc/apache2/sites-enabled/misp.conf
sudo rm -f /etc/apache2/sites-enabled/misp-ssl.conf
sudo rm -f /etc/apache2/sites-available/misp.conf
sudo rm -f /etc/apache2/sites-available/misp-ssl.conf

# ─── ลบ MISP Cron Jobs ───────────────────────────────
sudo crontab -r -u www-data 2>/dev/null || true
sudo rm -f /etc/cron.d/misp*

# ─── ลบ MISP Log Files ───────────────────────────────
sudo rm -rf /var/log/misp*
sudo rm -f /var/log/apache2/misp*

# ─── ลบ MISP User (ถ้าสร้างไว้) ─────────────────────
sudo userdel -r misp 2>/dev/null || true

# ─── ลบ MISP Modules Source ──────────────────────────
sudo rm -rf /usr/local/src/misp-modules 2>/dev/null || true

# ─── Reload Systemd ──────────────────────────────────
sudo systemctl daemon-reload

echo "✅ ลบ MISP Application แล้ว"
```

---

## Phase 3: ลบ Database

```bash
# ─── ลบ MISP Database และ User ───────────────────────
sudo mysql -u root << 'EOF'
DROP DATABASE IF EXISTS misp;
DROP USER IF EXISTS 'misp'@'localhost';
FLUSH PRIVILEGES;
SHOW DATABASES;
EOF

# ─── ตรวจสอบว่า Database ถูกลบแล้ว ──────────────────
sudo mysql -u root -e "SHOW DATABASES;"
# 'misp' ต้องไม่อยู่ในรายการ

echo "✅ ลบ Database แล้ว"
```

---

## Phase 4: ลบ Packages ที่ติดตั้งสำหรับ MISP

### 4.1 ลบ Apache2

```bash
sudo apt purge -y \
  apache2 \
  apache2-utils \
  apache2-bin \
  libapache2-mod-php*

sudo rm -rf /etc/apache2
sudo rm -rf /var/log/apache2
```

### 4.2 ลบ PHP และ Extensions

```bash
sudo apt purge -y \
  php* \
  libapache2-mod-php* \
  php-mysql \
  php-redis \
  php-xml \
  php-mbstring \
  php-curl \
  php-gd \
  php-zip \
  php-intl \
  php-bcmath \
  php-json \
  php-pear

sudo rm -rf /etc/php
sudo rm -rf /var/lib/php
```

### 4.3 ลบ MySQL / MariaDB

```bash
sudo apt purge -y \
  mysql-server \
  mysql-client \
  mysql-common \
  mariadb-server \
  mariadb-client \
  mariadb-common

sudo rm -rf /etc/mysql
sudo rm -rf /var/lib/mysql
sudo rm -rf /var/log/mysql
```

### 4.4 ลบ Redis

```bash
sudo apt purge -y redis-server redis-tools

sudo rm -rf /etc/redis
sudo rm -rf /var/lib/redis
sudo rm -rf /var/log/redis
```

### 4.5 ลบ Packages อื่นๆ

```bash
sudo apt purge -y \
  supervisor \
  ssdeep \
  libfuzzy-dev \
  libxml2-dev \
  libxslt1-dev \
  zlib1g-dev \
  libssl-dev 2>/dev/null || true
```

### 4.6 ลบ Python Packages ของ MISP

```bash
sudo pip3 uninstall -y pymisp misp-modules 2>/dev/null || true
sudo rm -rf /var/www/.cache
```

### 4.7 Autoremove Dependencies

```bash
sudo apt autoremove -y
sudo apt autoclean -y

echo "✅ ลบ Packages ทั้งหมดแล้ว"
```

---

## Phase 5: ลบ SSL Certificates

```bash
# ─── ลบ Self-signed Certificates ────────────────────
sudo rm -f /etc/ssl/private/misp.key
sudo rm -f /etc/ssl/certs/misp.crt
sudo rm -f /etc/ssl/certs/misp*

# ─── ลบ Let's Encrypt (ถ้าใช้) ───────────────────────
sudo certbot delete --cert-name misp.yourdomain.com 2>/dev/null || true
sudo rm -rf /etc/letsencrypt/live/misp* 2>/dev/null || true
sudo rm -rf /etc/letsencrypt/archive/misp* 2>/dev/null || true
sudo rm -rf /etc/letsencrypt/renewal/misp* 2>/dev/null || true

echo "✅ ลบ SSL Certificates แล้ว"
```

---

## Phase 6: ล้าง System

```bash
# ─── ล้าง Package Cache ───────────────────────────────
sudo apt clean
sudo apt autoremove -y
sudo apt autoclean

# ─── ล้าง Systemd Journal Logs ───────────────────────
sudo journalctl --vacuum-time=1d
sudo journalctl --vacuum-size=100M

# ─── ล้าง Temp Files ─────────────────────────────────
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*

# ─── ล้าง Old Kernels ────────────────────────────────
sudo apt purge -y $(dpkg -l 'linux-image-*' | \
  awk '/^ii/{print $2}' | \
  grep -v "$(uname -r)" | \
  grep -v "linux-image-generic" | \
  head -n -1) 2>/dev/null || true

# ─── ล้าง Bash History ───────────────────────────────
history -c
> ~/.bash_history

# ─── ดู Disk หลังล้าง ────────────────────────────────
echo ""
echo "=== Disk Usage หลังล้าง ==="
df -h
echo ""
du -sh /* 2>/dev/null | sort -rh | head -15

echo "✅ ล้าง System แล้ว"
```

---

## Phase 7: ล้าง Firewall Rules

```bash
# ─── ดู Rules ปัจจุบัน ────────────────────────────────
sudo ufw status numbered

# ─── Reset UFW ทั้งหมด ───────────────────────────────
sudo ufw reset

# ─── ตรวจสอบ Port ที่ยังเปิดอยู่ ─────────────────────
echo ""
echo "=== Port ที่ยังเปิดอยู่ ==="
sudo ss -tlnp

echo "✅ Reset Firewall แล้ว"
```

---

## Phase 8: ตรวจสอบว่าล้างหมดแล้ว

```bash
echo "======================================"
echo "  ตรวจสอบผลการล้างระบบ"
echo "======================================"

# ─── ตรวจสอบ Services ────────────────────────────────
echo ""
echo "=== Services ที่ยังรันอยู่ ==="
sudo systemctl list-units --type=service --state=running | \
  grep -iE "misp|apache|php|mysql|mariadb|redis" || \
  echo "✅ ไม่มี MISP-related services รันอยู่"

# ─── ตรวจสอบ Processes ───────────────────────────────
echo ""
echo "=== Processes ที่ยังรันอยู่ ==="
ps aux | grep -iE "apache|php|mysql|redis|misp" | \
  grep -v grep || \
  echo "✅ ไม่มี MISP-related processes"

# ─── ตรวจสอบ Files ───────────────────────────────────
echo ""
echo "=== ตรวจสอบ Files ==="
[ -d /var/www/MISP ] && \
  echo "❌ ยังมีโฟลเดอร์ /var/www/MISP อยู่!" || \
  echo "✅ ลบโฟลเดอร์ MISP แล้ว"

[ -d /etc/apache2 ] && \
  echo "❌ ยังมี /etc/apache2 อยู่!" || \
  echo "✅ ลบ Apache Config แล้ว"

[ -d /var/lib/mysql ] && \
  echo "❌ ยังมี /var/lib/mysql อยู่!" || \
  echo "✅ ลบ MySQL Data แล้ว"

# ─── ตรวจสอบ Port ────────────────────────────────────
echo ""
echo "=== Port ที่เปิดอยู่ ==="
sudo ss -tlnp | grep -E ":80 |:443 |:3306 |:6379 " || \
  echo "✅ ไม่มี Port ที่เกี่ยวข้องเปิดอยู่"

# ─── Disk Usage ──────────────────────────────────────
echo ""
echo "=== Disk Usage ==="
df -h /

echo ""
echo "======================================"
echo "  ตรวจสอบเสร็จสิ้น"
echo "======================================"
```

---

## Phase 9: อัปเดตระบบให้พร้อมใช้งานใหม่

```bash
# ─── อัปเดต Package List ─────────────────────────────
sudo apt update

# ─── Upgrade ทุก Package ─────────────────────────────
sudo apt upgrade -y

# ─── ติดตั้ง Essential Tools กลับมา ──────────────────
sudo apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  net-tools \
  ufw \
  ca-certificates \
  gnupg \
  lsb-release \
  unzip \
  software-properties-common \
  build-essential

# ─── ตั้งค่า UFW ใหม่ ────────────────────────────────
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH Custom Port'
sudo ufw enable
sudo ufw status verbose

echo "✅ ระบบพร้อมใช้งานใหม่แล้ว"
```

---

## Phase 10: Reboot และตรวจสอบสุดท้าย

```bash
# ─── Reboot ──────────────────────────────────────────
sudo reboot
```

หลัง Reboot เข้ามาตรวจสอบ:

```bash
# ─── ตรวจสอบ OS ──────────────────────────────────────
cat /etc/os-release
uname -a

# ─── ตรวจสอบ Services ────────────────────────────────
sudo systemctl list-units --type=service --state=running

# ─── ตรวจสอบ Port ────────────────────────────────────
sudo ss -tlnp

# ─── ตรวจสอบ Disk ────────────────────────────────────
df -h

# ─── ตรวจสอบ RAM ─────────────────────────────────────
free -h

# ─── ตรวจสอบ Uptime ──────────────────────────────────
uptime
```

---

## ✅ Checklist สรุป

| Phase | รายการ | สถานะ |
|:---:|:---|:---:|
| 0 | Backup Database + Config + Files | ☐ |
| 1 | หยุด Services: MISP, Apache, Redis, MySQL | ☐ |
| 2 | ลบ MISP Application + Cron + Logs | ☐ |
| 3 | ลบ Database `misp` + User | ☐ |
| 4 | ลบ Apache, PHP, MySQL, Redis, Python packages | ☐ |
| 5 | ลบ SSL Certificates | ☐ |
| 6 | ล้าง Cache, Temp, Journal Logs | ☐ |
| 7 | Reset UFW Firewall Rules | ☐ |
| 8 | ตรวจสอบว่าล้างหมดแล้ว | ☐ |
| 9 | อัปเดต OS + ติดตั้ง Essential Tools | ☐ |
| 10 | Reboot + ตรวจสอบสุดท้าย | ☐ |

---

## 🚀 ขั้นตอนถัดไป

หลังจากล้างเครื่องเสร็จ พร้อมติดตั้งสิ่งต่อไปนี้ได้เลย:

```bash
# ติดตั้ง Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

> 📝 **หมายเหตุ:** ถ้า MISP ติดตั้งด้วย Script อัตโนมัติ เช่น  
> `MISP-Vagrant` หรือ `misp-docker` อาจมี Path แตกต่างกันเล็กน้อย  
> ให้ตรวจสอบด้วย `find / -name "MISP" -type d 2>/dev/null` ก่อนลบ
