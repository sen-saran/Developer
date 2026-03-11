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

http://172.17.1.227:8080

docker volume ls | grep files
docker compose logs --follow owncloud

docker compose down
```


















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
