
# 🐳 Docker for Developers — คู่มือติดตั้งและใช้งาน

> คู่มือติดตั้ง Docker แบบ Step by Step พร้อมคำสั่งที่จำเป็นสำหรับการพัฒนาซอฟต์แวร์

---

## 📋 สารบัญ

- [1. ติดตั้ง Docker](#1-ติดตั้ง-docker)
- [2. ตรวจสอบการติดตั้ง](#2-ตรวจสอบการติดตั้ง)
- [3. คำสั่งจัดการ Container](#3-คำสั่งจัดการ-container)
- [4. คำสั่งจัดการ Image](#4-คำสั่งจัดการ-image)
- [5. คำสั่งจัดการ Network](#5-คำสั่งจัดการ-network)
- [6. คำสั่งจัดการ Volume](#6-คำสั่งจัดการ-volume)
- [7. Dockerfile](#7-dockerfile)
- [8. Docker Compose](#8-docker-compose)
- [9. Workshop: WordPress + MySQL + phpMyAdmin](#9-workshop-wordpress--mysql--phpmyadmin)
- [10. Security Best Practices](#10-security-best-practices)
- [11. Domain + SSL (Nginx + Let's Encrypt)](#11-domain--ssl-nginx--lets-encrypt)

---

## 1. ติดตั้ง Docker

### 🍎 macOS

```bash
# ดาวน์โหลด Docker Desktop จาก
# https://www.docker.com/products/docker-desktop/

# หรือติดตั้งผ่าน Homebrew
brew install --cask docker

# เปิด Docker Desktop แล้วรอให้ whale icon ใน menu bar หยุดกระพริบ
```

### 🐧 Linux (Ubuntu/Debian)

```bash
# Step 1: อัปเดต package index
sudo apt update

# Step 2: ติดตั้ง dependencies
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# Step 3: เพิ่ม Docker GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Step 4: เพิ่ม Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: ติดตั้ง Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Step 6: ให้ user ปัจจุบันใช้ Docker ได้โดยไม่ต้อง sudo
sudo usermod -aG docker $USER
newgrp docker
```

### 🪟 Windows

```powershell
# ดาวน์โหลด Docker Desktop จาก
# https://www.docker.com/products/docker-desktop/

# หรือติดตั้งผ่าน winget
winget install Docker.DockerDesktop

# รีสตาร์ทเครื่องหลังติดตั้ง
```

---

## 2. ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Docker version
docker --version
# ผลลัพธ์: Docker version 26.x.x, build xxxxxxx

# ตรวจสอบ Docker Compose version
docker compose version
# ผลลัพธ์: Docker Compose version v2.x.x

# ทดสอบรัน Container แรก
docker run hello-world

# ตรวจสอบว่า Docker daemon กำลังทำงาน
docker info
```

---

## 3. คำสั่งจัดการ Container

### 🚀 รัน Container

```bash
# รัน Container พื้นฐาน
docker run nginx

# รัน Foreground พร้อม Port Mapping  host:container
docker run -p 8800:80 nginx

# รัน Background (Detached mode)
docker run -d -p 9900:80 nginx

# รัน พร้อมตั้งชื่อ Container
docker run --name mynginx -d -p 7700:80 nginx

# รัน พร้อม Environment Variable
docker run -d -e MY_VAR=hello nginx

# รัน แบบลบทิ้งเมื่อหยุด
docker run --rm nginx
```

### 📋 ดูสถานะ

```bash
# ดู Container ที่กำลังรันอยู่
docker ps

# ดู Container ทั้งหมด (รวมที่หยุดแล้ว)
docker ps -a

# ดูรายละเอียด Container
docker inspect <container_id>

# ดูการใช้ Resource แบบ Real-time
docker stats
```

### ⏯️ หยุด / เริ่ม / ลบ

```bash
# หยุด Container (Graceful)
docker stop <container_id>

# หยุดทุก Container ที่รันอยู่
docker stop $(docker ps -q)

# เริ่ม Container ที่หยุดอยู่
docker start <container_id>

# Restart Container
docker restart <container_id>

# ลบ Container (ต้องหยุดก่อน)
docker rm <container_id>

# ลบ Container ทันที (Force)
docker rm -f <container_id>

# ลบ Container ที่หยุดแล้วทั้งหมด
docker container prune
```

### 🔍 Debug Container

```bash
# ดู Log
docker logs <container_id>

# ดู Log แบบ Follow (Real-time)
docker logs -f <container_id>

# ดู Log 50 บรรทัดล่าสุด
docker logs --tail 50 <container_id>

# รันคำสั่งใน Container ที่กำลังทำงาน
docker exec <container_id> ls

# เข้า Shell ของ Container
docker exec -it <container_id> sh

# เข้า Bash (ถ้ามี)
docker exec -it <container_id> bash

# Copy ไฟล์จาก Container มายัง Host
docker cp <container_id>:/path/in/container ./local-path
```

---

## 4. คำสั่งจัดการ Image

```bash
# ดู Image ทั้งหมดในเครื่อง
docker images

# ดึง Image จาก Docker Hub
docker pull nginx
docker pull mysql:5.7
docker pull node:18-alpine

# ลบ Image
docker rmi <image_id>

# ลบ Image ที่ไม่ได้ใช้ทั้งหมด
docker image prune

# Build Image จาก Dockerfile
docker build -t myapp:1.0 .

# Build พร้อม Build Args
docker build --build-arg NODE_ENV=production -t myapp:1.0 .

# Tag Image
docker tag myapp:1.0 myusername/myapp:1.0

# Push ขึ้น Docker Hub
docker push myusername/myapp:1.0
```

---

## 5. คำสั่งจัดการ Network

```bash
# ดู Network ทั้งหมด
docker network ls

# สร้าง Network แบบ Bridge
docker network create mynetwork

# ดูรายละเอียด Network
docker network inspect mynetwork

# เชื่อม Container เข้า Network
docker network connect mynetwork <container_id>

# ตัด Container ออกจาก Network
docker network disconnect mynetwork <container_id>

# ลบ Network
docker network rm mynetwork

# ลบ Network ที่ไม่ได้ใช้
docker network prune
```

---

## 6. คำสั่งจัดการ Volume

```bash
# ดู Volume ทั้งหมด
docker volume ls

# สร้าง Volume
docker volume create myvolume

# ดูรายละเอียด Volume
docker volume inspect myvolume

# รัน Container พร้อม Mount Volume
docker run -v myvolume:/app/data nginx

# Mount โฟลเดอร์จาก Host (Bind Mount)
docker run -v $(pwd)/data:/app/data nginx

# ลบ Volume
docker volume rm myvolume

# ลบ Volume ที่ไม่ได้ใช้
docker volume prune
```

---

## 7. Dockerfile

### คำสั่งหลักใน Dockerfile

| คำสั่ง | คำอธิบาย | ตัวอย่าง |
|:---|:---|:---|
| `FROM` | Base Image ตั้งต้น | `FROM node:18-alpine` |
| `WORKDIR` | กำหนด Working Directory | `WORKDIR /app` |
| `COPY` | คัดลอกไฟล์จาก Host | `COPY . .` |
| `ADD` | คัดลอกไฟล์ (รองรับ URL และ .tar.gz) | `ADD data.tar.gz /app` |
| `RUN` | รันคำสั่งตอน Build | `RUN npm install` |
| `ENV` | กำหนด Environment Variable | `ENV NODE_ENV=production` |
| `EXPOSE` | เปิด Port | `EXPOSE 3000` |
| `CMD` | คำสั่งเริ่มต้นตอน Run | `CMD ["npm", "start"]` |
| `ENTRYPOINT` | คำสั่งหลักที่ตายตัว | `ENTRYPOINT ["node", "app.js"]` |

> ⚠️ **ข้อควรรู้:** `RUN` ทำงานตอน **Build** (สร้าง Image), ส่วน `CMD` ทำงานตอน **Run** (สร้าง Container)

### ตัวอย่าง Dockerfile — Node.js App

```dockerfile
# ใช้ Node.js 18 Alpine (ขนาดเล็ก)
FROM node:18-alpine

# กำหนด Working Directory
WORKDIR /app

# Copy package.json ก่อน (ใช้ประโยชน์จาก Layer Cache)
COPY package*.json ./

# ติดตั้ง dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# กำหนด Environment
ENV NODE_ENV=production
ENV PORT=3000

# เปิด Port
EXPOSE 3000

# คำสั่งรัน App
CMD ["node", "server.js"]
```

### Build และรัน

```bash
# Build Image
docker build -t myapp:1.0 .

# รัน Container
docker run -d -p 3000:3000 --name myapp myapp:1.0

# ดู Log
docker logs -f myapp
```

---

## 8. Docker Compose

### คำสั่ง Docker Compose

```bash
# ตรวจสอบ Syntax ของ docker-compose.yml
docker compose config

# รัน Services ทั้งหมด (Background)
docker compose up -d

# รัน พร้อม Build ใหม่
docker compose up -d --build

# ดูสถานะ Services
docker compose ps

# ดู Log ทุก Services
docker compose logs

# ดู Log เฉพาะ Service
docker compose logs -f wordpress

# หยุด Services (ไม่ลบ Container)
docker compose stop

# หยุดและลบ Container + Network
docker compose down

# หยุดและลบทุกอย่างรวม Volume
docker compose down -v

# Restart Service เดียว
docker compose restart wordpress

# รันคำสั่งใน Service
docker compose exec wordpress sh

# Scale Service
docker compose up -d --scale wordpress=3
```

---

## 9. Workshop: WordPress + MySQL + phpMyAdmin

### Step 1: สร้างโฟลเดอร์โปรเจกต์

```bash
mkdir docker-wordpress-app
cd docker-wordpress-app
mkdir -p nginx/conf.d certbot/conf certbot/www
touch docker-compose.yml .env
```

### Step 2: สร้างไฟล์ .env

```env
MYSQL_ROOT_PASSWORD=S3cur3R00tP@ss!
MYSQL_DATABASE=wordpress_db
MYSQL_USER=wordpress
MYSQL_PASSWORD=W0rdPr3ss@Pass!
```

### Step 3: สร้าง docker-compose.yml

```yaml
networks:
  wp_network:
    name: wp_network
    driver: bridge

services:
  db:
    image: mysql:5.7.44
    container_name: mysql_db
    restart: always
    networks:
      - wp_network
    volumes:
      - mysql_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    security_opt:
      - no-new-privileges:true

  wordpress:
    depends_on:
      - db
    image: wordpress:6.7.1-php8.3-apache
    container_name: wordpress_app
    restart: always
    networks:
      - wp_network
    volumes:
      - ./wordpress_files:/var/www/html
    ports:
      - 8800:80
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=${MYSQL_USER}
      - WORDPRESS_DB_PASSWORD=${MYSQL_PASSWORD}
      - WORDPRESS_DB_NAME=${MYSQL_DATABASE}
      - WORDPRESS_CONFIG_EXTRA=define('FORCE_SSL_ADMIN', true);
    security_opt:
      - no-new-privileges:true

  phpmyadmin:
    depends_on:
      - db
    image: phpmyadmin:5.2.1
    container_name: phpmyadmin_tool
    restart: always
    networks:
      - wp_network
    environment:
      - PMA_ARBITRARY=1
      - PMA_HOST=db
    ports:
      - 127.0.0.1:7770:80

volumes:
  mysql_data:
    name: mysql_data
```

### Step 4: รัน

```bash
# รัน ทุก Services
docker compose up -d

# ตรวจสอบสถานะ
docker compose ps

# ดู Log
docker compose logs -f
```

### Step 5: เข้าใช้งาน

| Service | URL | Login |
|:---|:---|:---|
| WordPress | http://localhost:8800 | ตั้งค่าครั้งแรกผ่านเว็บ |
| phpMyAdmin | http://localhost:7770 | Server: `db` / User: `root` / Pass: ตาม `.env` |

---

## 10. Security Best Practices

```bash
# ❌ อย่า Hardcode password ใน docker-compose.yml
# ✅ ใช้ .env file และเพิ่มใน .gitignore เสมอ
echo ".env" >> .gitignore

# ตรวจสอบ Container ที่มีช่องโหว่
docker scout cves <image_name>

# ดู User ที่รัน Process ใน Container
docker exec <container_id> whoami

# สแกน Image ด้วย Trivy
docker run --rm aquasec/trivy image nginx:latest
```

### Security Checklist

- [ ] ใช้ `.env` file แทน Hardcode passwords
- [ ] เพิ่ม `.env` ใน `.gitignore`
- [ ] ปิด Port MySQL ไม่ให้เปิดออก Public
- [ ] Pin image version (ไม่ใช้ `latest`)
- [ ] เพิ่ม `no-new-privileges:true`
- [ ] จำกัด Memory/CPU ด้วย `deploy.resources.limits`
- [ ] ซ่อน phpMyAdmin ไว้ใน `127.0.0.1` หรือผ่าน VPN

---

## 11. Domain + SSL (Nginx + Let's Encrypt)

### Step 1: ตรวจสอบ DNS

```bash
# ตรวจสอบว่า domain ชี้มาที่ Server IP แล้ว
nslookup yourdomain.com
dig yourdomain.com A
```

### Step 2: สร้าง Nginx Config

**`nginx/conf.d/wordpress.conf`**

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;

    location / {
        proxy_pass         http://wordpress_app:80;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        client_max_body_size 64M;
    }
}
```
Internet → Nginx (80/443) → WordPress Container
                ↓
         SSL Certificate (Let's Encrypt)

### Step 3: เพิ่ม Nginx + Certbot ใน docker-compose.yml

```yaml
  nginx:
    image: nginx:1.27-alpine
    container_name: nginx_proxy
    restart: always
    networks:
      - wp_network
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    depends_on:
      - wordpress

  certbot:
    image: certbot/certbot:latest
    container_name: certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
```

### Step 4: ขอ SSL Certificate

```bash
# รัน Nginx ก่อน
docker compose up -d nginx

# ขอ Certificate
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email your@email.com \
  --agree-tos \
  --no-eff-email \
  -d yourdomain.com \
  -d www.yourdomain.com

# Reload Nginx
docker compose exec nginx nginx -s reload
```

### Step 5: ตั้ง Auto-Renew

```bash
# เปิด Crontab
crontab -e

# เพิ่มบรรทัดนี้ (ต่ออายุทุกวัน ตี 3)
0 3 * * * docker compose -f /path/to/docker-compose.yml run --rm certbot renew && docker compose -f /path/to/docker-compose.yml exec nginx nginx -s reload
```

---

## 🧹 Cleanup Commands

```bash
# ลบ Container ที่หยุดแล้ว
docker container prune

# ลบ Image ที่ไม่ได้ใช้
docker image prune

# ลบ Volume ที่ไม่ได้ใช้
docker volume prune

# ลบ Network ที่ไม่ได้ใช้
docker network prune

# ลบทุกอย่างที่ไม่ได้ใช้ (รวม Image)
docker system prune -a

# ดูพื้นที่ที่ Docker ใช้
docker system df
```

---

## 📚 Resources

- [Docker Documentation](https://docs.docker.com)
- [Docker Hub](https://hub.docker.com)
- [Docker Compose Reference](https://docs.docker.com/compose/reference/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Let's Encrypt](https://letsencrypt.org)

---

> 📝 **หมายเหตุ:** แทนที่ `yourdomain.com` และ `your@email.com` ด้วยข้อมูลจริงของคุณ  
> 🔒 อย่าลืม commit `.env` เข้า Git และใช้ Secret Manager สำหรับ Production
