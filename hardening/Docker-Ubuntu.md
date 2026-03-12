---
# 🐳 Docker Installation Guide
---
sudo hostnamectl set-hostname owncloud
sudo nano /etc/hosts
แก้ไขเป็น:
127.0.1.1  owncloud
hostnamectl
## Ubuntu 24.04 LTS
### Step 1: Set up Docker's apt repository.
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

```

### Step 2: Install the Docker packages.
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo systemctl enable docker
sudo systemctl start docker
# Verify that the installation is successful by running the hello-world image:
sudo docker run hello-world
# Hello from Docker!
# This message shows that your installation appears to be working correctly.
```
### Step 2: อนุญาตให้ User ใช้ Docker (ไม่ต้อง sudo)
```bash
sudo usermod -aG docker digitalsaran
# เพิ่ม user digitalsaran เข้า group docker
newgrp docker
# โหลด group ใหม่ทันที CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
docker ps
docker run hello-world
docker --version
docker ps -a
```
### Step 3: Docker Security
```bash
docker info
# จำกัด Docker Log Size ตอนนี้ default คือ unlimited
sudo nano /etc/docker/daemon.json
sudo systemctl restart docker
docker info | grep -i live
```
- จำกัดขนาด log รวม 30MB
- max-siz container log 10MB
- max-file เก็บสูงสุด 3 file
- live-restore container จะไม่หยุด
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
```
### Step 4: Docker volume
```bash
docker volume ls
docker volume create nextcloud_volume
docker volume create mariadb_volume
docker volume create redis_volume
docker volume ls
# แยก project container ออกจาก system
# ทำให้ backup / migration ง่าย
mkdir owncloud-docker-server
cd owncloud-docker-server
nano docker-compose.yml
```
owncloud-docker-server
 ├─ docker-compose.yml
 └─ .env
```bash
version: "3"

volumes:
  files:
    driver: local
  mysql:
    driver: local
  redis:
    driver: local

services:
  owncloud:
    image: owncloud/server:${OWNCLOUD_VERSION}
    container_name: owncloud_server
    restart: always
    ports:
      - ${HTTP_PORT}:8080
    depends_on:
      - mariadb
      - redis
    environment:
      - OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN}
      - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_TRUSTED_DOMAINS}
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_NAME=${MYSQL_DATABASE}
      - OWNCLOUD_DB_USERNAME=${MYSQL_USER}
      - OWNCLOUD_DB_PASSWORD=${MYSQL_PASSWORD}
      - OWNCLOUD_DB_HOST=mariadb
      - OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME}
      - OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - OWNCLOUD_MYSQL_UTF8MB4=true
      - OWNCLOUD_REDIS_ENABLED=true
      - OWNCLOUD_REDIS_HOST=redis
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - files:/mnt/data

  mariadb:
    image: mariadb:10.11 # minimum required ownCloud version is 10.9
    container_name: owncloud_mariadb
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MARIADB_AUTO_UPGRADE=1
    command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - mysql:/var/lib/mysql

  redis:
    image: redis:6
    container_name: owncloud_redis
    restart: always
    command: ["--databases", "1"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - redis:/data
```
```bash
cat << EOF > .env
OWNCLOUD_VERSION=10.16
OWNCLOUD_DOMAIN=172.17.1.227:8080
OWNCLOUD_TRUSTED_DOMAINS=172.17.1.227 localhost
# OWNCLOUD_TRUSTED_DOMAINS=cloud.company.go.th

ADMIN_USERNAME=admin
ADMIN_PASSWORD=StrongAdminPassword!2026

HTTP_PORT=8080

MYSQL_ROOT_PASSWORD=StrongRootPassword!2026
MYSQL_USER=owncloud
MYSQL_PASSWORD=StrongDBPassword!2026
MYSQL_DATABASE=owncloud
EOF
```
# Docker container Hardening
```bash
docker ps

```
```bash
ls
ls -la
chmod 600 .env
chmod 644 docker-compose.yml
ls -l
ls -ld .
chmod 750 .

docker compose config

docker compose up -d

docker ps
# owncloud_server   Up (health: starting)
# owncloud_redis    Up (healthy)
# owncloud_mariadb  Up (healthy)




docker network ls
docker network inspect owncloud-docker-server_default

sudo ufw allow 8080/tcp  comment 'HTTP'
ss -tulnp | grep 8080
# reload
sudo ufw reload
http://172.17.1.227:8080

docker volume ls | grep files
docker compose logs --follow owncloud
docker logs owncloud_mariadb
docker logs owncloud_server
docker exec -it owncloud_mariadb mysql -u root -p
SHOW DATABASES;
USE owncloud;
SHOW TABLES;
docker compose down

# วิธีแก้ (แก้ config)
# เข้า container
docker exec -it owncloud_server bash

# หรือเข้า container 
sudo nano /var/lib/docker/volumes/owncloud-docker-server_files/_data/config/config.php

# เปิดไฟล์
apt update && apt install nano -y
nano /var/www/owncloud/config/config.php

# แก้จากแบบนี้
'trusted_domains' =>
array (
  0 => 'localhost',
  1 => '172.17.1.227',
  2 => '172.17.1.227:8080',
),
'overwrite.cli.url' => 'http://172.17.1.227:8080',
'overwritehost' => '172.17.1.227:8080',
'overwriteprotocol' => 'http',

# ออกจาก container
exit

# restart
docker restart owncloud_server
```
docker exec -it owncloud_server php occ user:list
docker ps

docker exec -it owncloud_mariadb mysql -u root -p -e "SHOW DATABASES;"

docker exec -it owncloud_mariadb mysql -u root -p

# ดู DATABASES ทั้งหมด
SHOW DATABASES;

# เลือก owncloud
USE owncloud;

# แสดง TABLES
SHOW TABLES;

# ดู user ทั้งหมด
SELECT * FROM oc_users;

# ดูจำนวน user
SELECT COUNT(*) FROM oc_users;

# ดู groups
SELECT * FROM oc_groups;

# ดู id user
SELECT numeric_id,id 
FROM oc_storages 
WHERE id LIKE 'home::%';

# ดู user quota
SELECT userid, configvalue AS quota 
FROM oc_preferences 
WHERE configkey='quota';

# ดู user file all
SELECT path, ROUND(size/1024/1024,2) AS MB 
FROM oc_filecache 
WHERE storage = 8
ORDER BY size DESC;

# ดู user file files
SELECT path
FROM oc_filecache
WHERE storage=8
AND path LIKE 'files/%';

# ดู 1 user file size
SELECT  ROUND(SUM(size)/1024/1024,2) AS MB 
FROM oc_filecache 
WHERE storage=8;

# ดู ทั้งหมด user file size
SELECT
SUBSTRING_INDEX(s.id,'::',-1) AS username,
ROUND(SUM(f.size)/1024/1024,2) AS MB
FROM oc_filecache f
JOIN oc_storages s
ON f.storage = s.numeric_id
WHERE s.id LIKE 'home::%'
GROUP BY username;


# ดู storage usage
SELECT storage, SUM(size)
FROM oc_filecache
GROUP BY storage;

# ดู mapping storage
SELECT * FROM oc_storages;

# ดูไฟล์ user ทั้งหมด
SELECT path,size
FROM oc_filecache
WHERE storage=8;

# ดูไฟล์ user ที่อัพโหลด
SELECT path,size
FROM oc_filecache
WHERE storage=8
AND path LIKE 'files/%';

# วิธีตรวจว่ามีการแชร์
SELECT
id,
share_type,
uid_owner,
share_with
FROM oc_share;



# ดูไฟล์ทั้งหมดในระบบ
SELECT COUNT(*) FROM oc_filecache;

ดู path ทั้งหมดที่ไม่ซ้ำ
SELECT DISTINCT SUBSTRING_INDEX(path,'/',1) FROM oc_filecache;
SELECT DISTINCT SUBSTRING_INDEX(path,'/files',8) FROM oc_filecache;
Query user + storage ที่ถูกต้อง
SELECT * FROM oc_storages;














### Step 3: เพิ่ม Docker’s Official GPG Key
```bash
# ขั้นตอนนี้เป็นการยืนยันตัวตนว่าซอฟต์แวร์ที่เราจะโหลดมาจาก Docker โดยตรง ไม่ได้ถูกปลอมแปลง
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Step 4: ตั้งค่า Repository
```bash
# เราจะบอกให้ Ubuntu รู้ว่าต้องไปโหลด Docker จากที่ไหน (Official Repo)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 5: ติดตั้ง Docker Engine
```bash
# หลังจากตั้งค่าแหล่งที่มาแล้ว ก็สั่งติดตั้งได้เลยครับ (ชุดนี้รวม Docker Compose มาให้ด้วย)
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
---
# 🛠 การตรวจสอบและตั้งค่าหลังติดตั้ง
---
```bash
# ตรวจสอบสถานะ
sudo systemctl status docker
sudo systemctl enable docker
sudo systemctl start docker
# ทดสอบด้วย Hello World
sudo docker run hello-world
# รัน Docker โดยไม่ต้องพิมพ์ sudo (Optional แต่แนะนำ)
sudo usermod -aG docker $USER
newgrp docker
groups $USER
docker --version
docker info
# Container Management
docker ps
docker ps -a
docker start container_name
docker stop container_name
docker rm container_name
# Image Management
docker images
docker pull image_name
docker rmi image_name
docker build -t image_name .
```
