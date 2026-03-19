# Nextcloud Docker Installation Guide
### Ubuntu 24.04 LTS + PostgreSQL + Cloudflare Tunnel
---
## Step 3: Create Environment Nextcloud Docker
### 3.1 สร้าง Project Structure

```bash
sudo mkdir -p /data/nextcloud/{db,files,backup}
mkdir -p ~/nextcloud/{nginx/ssl}
# เจ้าของเป็น user ปัจจุบัน
sudo chown -R $USER:$USER /data/nextcloud   
ls -la /data/nextcloud
ls -la
```

```bash
cd ~/nextcloud
# สร้างโฟลเดอร์
mkdir -p ~/nextcloud/nginx/ssl

# copy ไฟล์ cert ของหน่วยงานมาวางที่นี่
cp /path/to/your/fullchain.pem ~/nextcloud/nginx/ssl/
cp /path/to/your/privkey.pem   ~/nextcloud/nginx/ssl/
# ล็อค permission
chmod 600 ~/nextcloud/nginx/ssl/*.pem

# สร้างไฟล์ default.conf
nano ~/nextcloud/nginx/default.conf
# เช็คว่าไฟล์มีแล้ว
ls -la ~/nextcloud/nginx/

echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3.2 สร้าง .env
```bash
nano ~/nextcloud/.env
chmod 600 ~/nextcloud/.env
cat ~/nextcloud/.env
# เช็คว่าไฟล์มีอยู่จริง
ls -la ~/nextcloud

# ทดสอบว่า docker compose อ่านได้ไหม
docker compose  config | grep POSTGRES_DB
*กรณีมีปัญหา*
# ลบ container และข้อมูล db เก่า
docker compose down
sudo rm -rf /data/nextcloud/db/*

# รันใหม่
cd ~/nextcloud
docker compose  up -d

# ตรวจสอบ
docker compose  logs db
```

### 3.3 สร้าง docker-compose.yml
```bash
nano ~/nextcloud/docker-compose.yml
```
---

## Step 4: Install Docker

### 4.1 ติดตั้ง Docker

```bash
sudo apt update
sudo apt install -y curl

curl -fsSL https://get.docker.com | sh

# เพิ่ม user เข้า docker group
sudo usermod -aG docker $USER
newgrp docker
```

### 4.2 ติดตั้ง Docker Compose Plugin

```bash
sudo apt install -y docker-compose-plugin

# ตรวจสอบ version
docker --version
docker compose version
```

### 4.3 ตั้งให้ Docker Auto-start

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

### 4.4 รัน Nextcloud

```bash
cd ~/nextcloud
docker compose --env-file secrets/.env up -d

# ดู logs
docker compose logs -f app
```

รอจนเห็น:
```
nextcloud_app | Nextcloud was successfully installed
```

### 4.5 ตรวจสอบ

```bash
docker compose ps

sudo ufw delete allow 8080/tcp
sudo ufw delete allow 8081/tcp

# ตรวจสอบ
sudo ufw status numbered

# ผลที่ควรได้:
[ 1] 717/tcp    ALLOW IN    Anywhere    # SSH Custom Port
[ 2] 80/tcp     ALLOW IN    Anywhere    # HTTP
[ 3] 443/tcp    ALLOW IN    Anywhere    # HTTPS

เปิด browser: `http://159.138.255.6`
80✅ เปิดredirect → 443 ถ้าใครพิมพ์ http:// จะถูกส่งไป https:// อัตโนมัติ443✅ เปิดHTTPS จริง
```
---

## กรณียังใช้งานไม่ได้
```bash
grep -i protocol ~/nextcloud/docker-compose.yml
### เช็ค container ทุกตัว
docker compose  ps

### เช็ค nginx logs
docker compose  logs nginx

### เช็ค app logs
docker compose  logs app

# วิธีแก้ (แก้ config)
# เช็ค config ปัจจุบัน
docker exec -u www-data nextcloud_app php occ config:system:get overwrite.cli.url
docker exec -u www-data nextcloud_app php occ config:system:get overwritehost
# เข้า container
docker exec -it nextcloud_app bash


# เปิดไฟล์
apt update && apt install nano -y
# path จริงของ Nextcloud ใน Docker
nano /var/www/html/config/config.php

# แก้จากแบบนี้
'trusted_domains' =>
array (
  0 => '159.138.255.6',
  1 => 'dmbcloud.dms.go.th',
),
'overwrite.cli.url' => 'http://159.138.255.6',
'overwritehost' => '159.138.255.6',
'overwriteprotocol' => 'http',

# ออกจาก container
exit

# restart
docker restart nextcloud_app


# restart app
docker compose restart app
```

## ทางที่ 1: เปลี่ยนชื่อ user ใน Nextcloud (แนะนำ)
```bash
### เปลี่ยน display name
docker exec -u www-data nextcloud_app php occ user:modify admin displayname "ชื่อใหม่"
### หรือสร้าง admin user ใหม่แล้วลบเก่า
docker exec -u www-data nextcloud_app php occ user:add --password-from-env --group="admin" ชื่อใหม่

## ทางที่ 2: ลบข้อมูลแล้วติดตั้งใหม่ (ถ้ายังไม่มีข้อมูลสำคัญ)
docker compose down
sudo rm -rf /data/nextcloud/db/*
sudo rm -rf /data/nextcloud/files/*

### แก้ .env ก่อน
nano ~/nextcloud/.env

docker compose up -d
```
---

## Post-Install Configuration

### ตั้งค่า Background Job (Cron)

```bash
crontab -e
```

```cron
*/5 * * * * docker exec -u www-data nextcloud_app php -f /var/www/html/cron.php
```
### ตั้งค่า Nextcloud
# login เข้าหน้าเว็บไป overview เพื่อแก้ไข error ของ nextcloud
```
cd etc/php/7.4/apache2/
ls
sudo nano php.ini
# ctrl+w
memory_limit = 1028M
sudo service apache2 restart

cd /var/www/nextcloud/config
ls
sudo nano config.php
'default_phone_region' => 'TH',
```
# login เข้าหน้าเว็บไป system เพื่อแก้ไขร upload ไฟล์ ของ nextcloud
```
cd etc/php/7.4/apache2/
ls
sudo nano php.ini
# ctrl+w
upload_max_filesize = 5G
post_max_size = 5G
sudo service apache2 restart

sudo apt remove imagemagick-6-common php-imagick
sudo apt install imagemagick php-imagick
```
# การเซ็ทเพื่อเพิ่ม domain ของ nextcloud
```
cd /var/www/nextcloud/config
ls
sudo nano config.php
array (
  0 => '159.138.255.6',
  1 => 'dmbcloud.dms.go.th:443',
),
sudo service apache2 restart
```
community document server
onlyoffice
adress 
### ตั้งค่า Nextcloud
```
วิธีแก้ปัญหา Docker Container "หลับ" หรือหยุดทำงาน (Sleep/Exit) คือการทำให้ Container มี Process หลักทำงานอยู่ตลอดเวลา โดยใช้คำสั่ง
-it (Interactive + TTY) เพื่อรันแบบ Interactive mode หรือใช้ -d ร่วมกับคำสั่งที่ไม่จบการทำงาน เช่น tail -f /dev/null หรือใช้ restart policy เพื่อสั่งให้คอนเทนเนอร์ทำงานใหม่ทันทีหากหลุด
```
```bash
# เช็ค Docker auto-start
sudo systemctl is-enabled docker

# เช็ค restart policy ของทุก container
docker inspect --format='{{.Name}} → {{.HostConfig.RestartPolicy.Name}}' $(docker ps -aq)

# เปิด Docker auto-start
sudo systemctl enable docker

# ทุก container ใน docker-compose.yml ต้องมี
restart: always

docker ps -a && docker images && docker volume ls
# ตั้ง background job
docker exec -u www-data nextcloud_app php occ background:cron

# ตั้ง phone region
docker exec -u www-data nextcloud_app php occ config:system:set default_phone_region --value=TH

# ตั้ง maintenance window (backup ตอนกลางคืน)
docker exec -u www-data nextcloud_app php occ config:system:set maintenance_window_start --value=1
```
