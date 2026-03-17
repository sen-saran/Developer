# 🛡️ Production Docker Deployment Guide
## Linux Hardening → Docker → Deploy → Security

> คู่มือฉบับสมบูรณ์สำหรับการ Deploy WordPress บน Production Server  
> ตั้งแต่ติดตั้ง OS จนถึง Real-World Security

---

## 📋 สารบัญ

| ขั้นตอน | หัวข้อ |
|:---:|:---|
| **Phase 1** | [Linux Server Hardening](#phase-1-linux-server-hardening) |
| **Phase 2** | [ติดตั้ง Docker บน Production](#phase-2-ติดตั้ง-docker-บน-production) |
| **Phase 3** | [Docker Security Hardening](#phase-3-docker-security-hardening) |
| **Phase 4** | [Deploy WordPress Stack](#phase-4-deploy-wordpress-stack) |
| **Phase 5** | [Domain + SSL + Nginx](#phase-5-domain--ssl--nginx) |
| **Phase 6** | [Firewall & Network Security](#phase-6-firewall--network-security) |
| **Phase 7** | [Monitoring & Logging](#phase-7-monitoring--logging) |
| **Phase 8** | [Backup & Disaster Recovery](#phase-8-backup--disaster-recovery) |
| **Phase 9** | [Production Checklist](#phase-9-production-checklist) |

---

## Phase 1: Linux Server Hardening

> เริ่มต้นด้วย Ubuntu 22.04 LTS (แนะนำสำหรับ Production)

### 1.1 อัปเดตระบบ

```bash
# อัปเดต package ทั้งหมด
sudo apt update && sudo apt upgrade -y

# ติดตั้ง unattended-upgrades (อัปเดต Security Patch อัตโนมัติ)
sudo apt install -y unattended-upgrades apt-listchanges

# เปิดใช้งาน Auto Security Updates
sudo dpkg-reconfigure --priority=low unattended-upgrades

# ตรวจสอบสถานะ
sudo systemctl status unattended-upgrades
```

### 1.2 สร้าง Non-Root User

```bash
# สร้าง User ใหม่ (แทนที่ 'deploy' ด้วยชื่อที่ต้องการ)
sudo adduser deploy

# เพิ่มเข้ากลุ่ม sudo
sudo usermod -aG sudo deploy

# สลับไปใช้ User ใหม่
su - deploy

# ตรวจสอบสิทธิ์
sudo whoami
# ผลลัพธ์: root
```

### 1.3 ตั้งค่า SSH Key Authentication

```bash
# บนเครื่อง Local ของคุณ — สร้าง SSH Key (ถ้ายังไม่มี)
ssh-keygen -t ed25519 -C "your@email.com"

# Copy Public Key ไปยัง Server
ssh-copy-id deploy@your-server-ip

# ทดสอบ Login ด้วย Key
ssh deploy@your-server-ip
```

### 1.4 Harden SSH Configuration

```bash
# สำรอง config เดิมก่อน
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# แก้ไข SSH Config
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

# ปิด Protocol เก่า
Protocol 2
```

```bash
# Restart SSH (อย่าปิด Terminal เดิมก่อน!)
sudo systemctl restart sshd

# เปิด Terminal ใหม่ทดสอบ Login ด้วย Port ใหม่
ssh -p 2222 deploy@your-server-ip
```

### 1.5 ติดตั้ง Fail2Ban (ป้องกัน Brute Force)

```bash
# ติดตั้ง Fail2Ban
sudo apt install -y fail2ban

# สร้าง Local Config
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Ban IP เป็นเวลา 1 ชั่วโมง
bantime  = 3600
# ช่วงเวลาตรวจสอบ 10 นาที
findtime = 600
# พยายาม Login ผิด 5 ครั้ง = Ban
maxretry = 5
# Email แจ้งเตือน
destemail = your@email.com
action = %(action_mwl)s

[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled = true

[nginx-limit-req]
enabled = true
```

```bash
# เริ่มใช้งาน Fail2Ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# ตรวจสอบสถานะ
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### 1.6 Kernel Security Parameters (sysctl)

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

```ini
# ปิด IP Forwarding (ถ้าไม่ใช่ Router)
net.ipv4.ip_forward = 0

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

# เปิด Log Martian Packets
net.ipv4.conf.all.log_martians = 1

# ป้องกัน SMURF Attack
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# ป้องกัน Time-Wait Assassination
net.ipv4.tcp_rfc1337 = 1

# Shared Memory Security
kernel.shmmax = 68719476736
kernel.randomize_va_space = 2
```

```bash
# Apply ทันที
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

### 1.7 ติดตั้ง Intrusion Detection (AIDE)

```bash
# ติดตั้ง AIDE (ตรวจสอบ File System Integrity)
sudo apt install -y aide

# สร้าง Database ครั้งแรก
sudo aideinit
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# ตั้ง Cron ตรวจสอบทุกวัน
echo "0 5 * * * root /usr/bin/aide --check" | sudo tee /etc/cron.d/aide
```

### 1.8 ลบ Package ที่ไม่จำเป็น

```bash
# ลบ Service ที่ไม่ใช้
sudo apt purge -y telnet rsh-client rsh-redone-client ftp

# ปิด Service ที่ไม่จำเป็น
sudo systemctl disable --now bluetooth avahi-daemon cups

# ตรวจสอบ Port ที่เปิดอยู่
sudo ss -tlnp
sudo netstat -tlnp
```

---

## Phase 2: ติดตั้ง Docker บน Production

### 2.1 ติดตั้ง Docker Engine

```bash
# Step 1: ลบ Docker เก่า (ถ้ามี)
sudo apt purge -y docker docker-engine docker.io containerd runc

# Step 2: ติดตั้ง Dependencies
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# Step 3: เพิ่ม Docker GPG Key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step 4: เพิ่ม Repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: ติดตั้ง Docker
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# Step 6: เพิ่ม User เข้ากลุ่ม docker
sudo usermod -aG docker deploy
newgrp docker

# Step 7: เปิด Auto-start
sudo systemctl enable docker
sudo systemctl start docker

# ทดสอบ
docker run hello-world
docker compose version
```

### 2.2 ตั้งค่า Docker Daemon Security

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
# Restart Docker
sudo systemctl restart docker

# ตรวจสอบ
docker info | grep -E "Storage|Logging|Security"
```

---

## Phase 3: Docker Security Hardening

### 3.1 ตั้งค่า Docker Bench Security

```bash
# รัน Docker Security Audit
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /lib/systemd/system:/lib/systemd/system:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security
```

### 3.2 สร้าง Dedicated Docker Network

```bash
# สร้าง Network แยกสำหรับแต่ละ Stack
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/24 \
  --opt com.docker.network.bridge.name=wp_bridge \
  wp_network

# ตรวจสอบ
docker network inspect wp_network
```

### 3.3 ตรวจสอบ Image ก่อน Deploy

```bash
# ติดตั้ง Trivy (Image Vulnerability Scanner)
sudo apt install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install -y trivy

# สแกน Image ก่อน Deploy
trivy image wordpress:6.7.1-php8.3-apache
trivy image mysql:5.7.44
trivy image nginx:1.27-alpine

# สแกนหา Secret ที่รั่วไหลใน Image
trivy image --scanners secret wordpress:6.7.1-php8.3-apache
```

---

## Phase 4: Deploy WordPress Stack

### 4.1 โครงสร้างโฟลเดอร์

```bash
# สร้างโครงสร้างทั้งหมด
mkdir -p /opt/wordpress/{nginx/conf.d,certbot/{conf,www},wordpress_files,logs}
cd /opt/wordpress

# กำหนด Permission
sudo chown -R deploy:deploy /opt/wordpress
chmod 750 /opt/wordpress
```

```
/opt/wordpress/
├── docker-compose.yml
├── docker-compose.override.yml   ← Local overrides (ไม่ commit)
├── .env                          ← Secrets (ไม่ commit)
├── .gitignore
├── nginx/
│   └── conf.d/
│       └── wordpress.conf
├── certbot/
│   ├── conf/                     ← SSL Certificates
│   └── www/                      ← ACME Challenge
├── wordpress_files/              ← WordPress Data
└── logs/                         ← Application Logs
```

### 4.2 สร้างไฟล์ .env

```bash
nano /opt/wordpress/.env
```

```env
# === Database ===
MYSQL_ROOT_PASSWORD=Str0ng!R00t#Pass2024
MYSQL_DATABASE=wordpress_prod
MYSQL_USER=wp_user
MYSQL_PASSWORD=Str0ng!WP#Pass2024

# === WordPress ===
WORDPRESS_TABLE_PREFIX=wp9x_
WP_SITEURL=https://yourdomain.com
WP_HOME=https://yourdomain.com

# === Domain ===
DOMAIN=yourdomain.com
LETSENCRYPT_EMAIL=admin@yourdomain.com

# === Security Keys (สร้างจาก https://api.wordpress.org/secret-key/1.1/salt/)
WP_AUTH_KEY=put-random-string-here
WP_SECURE_AUTH_KEY=put-random-string-here
WP_LOGGED_IN_KEY=put-random-string-here
WP_NONCE_KEY=put-random-string-here
```

```bash
# จำกัดสิทธิ์ .env เฉพาะเจ้าของเท่านั้น
chmod 600 /opt/wordpress/.env

# สร้าง .gitignore
cat > /opt/wordpress/.gitignore << 'EOF'
.env
.env.*
certbot/conf/
wordpress_files/
logs/
*.log
EOF
```

### 4.3 สร้าง docker-compose.yml (Production)

```bash
nano /opt/wordpress/docker-compose.yml
```

```yaml
networks:
  wp_frontend:
    name: wp_frontend
    driver: bridge
  wp_backend:
    name: wp_backend
    driver: bridge
    internal: true      # ← ไม่มี Internet Access โดยตรง

services:

  # ─── Database ────────────────────────────────────────────
  db:
    image: mysql:5.7.44
    container_name: wp_mysql
    restart: unless-stopped
    networks:
      - wp_backend
    volumes:
      - mysql_data:/var/lib/mysql
      - ./logs/mysql:/var/log/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # ─── WordPress ────────────────────────────────────────────
  wordpress:
    image: wordpress:6.7.1-php8.3-apache
    container_name: wp_app
    restart: unless-stopped
    networks:
      - wp_frontend
      - wp_backend
    volumes:
      - ./wordpress_files:/var/www/html
      - ./logs/wordpress:/var/log/apache2
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=${MYSQL_USER}
      - WORDPRESS_DB_PASSWORD=${MYSQL_PASSWORD}
      - WORDPRESS_DB_NAME=${MYSQL_DATABASE}
      - WORDPRESS_TABLE_PREFIX=${WORDPRESS_TABLE_PREFIX}
      - WORDPRESS_CONFIG_EXTRA=
          define('WP_HOME', '${WP_HOME}');
          define('WP_SITEURL', '${WP_SITEURL}');
          define('FORCE_SSL_ADMIN', true);
          define('DISALLOW_FILE_EDIT', true);
          define('WP_AUTO_UPDATE_CORE', true);
          define('WP_DEBUG', false);
          define('WP_DEBUG_LOG', false);
          define('WP_DEBUG_DISPLAY', false);
          define('DISALLOW_FILE_MODS', false);
          @ini_set('upload_max_size', '64M');
          @ini_set('post_max_size', '64M');
          @ini_set('memory_limit', '256M');
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
      - NET_BIND_SERVICE
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/wp-login.php"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ─── Nginx Reverse Proxy ──────────────────────────────────
  nginx:
    image: nginx:1.27-alpine
    container_name: wp_nginx
    restart: unless-stopped
    networks:
      - wp_frontend
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./logs/nginx:/var/log/nginx
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
      - SETUID
      - SETGID
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 128M
    depends_on:
      - wordpress
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ─── Certbot (SSL) ────────────────────────────────────────
  certbot:
    image: certbot/certbot:latest
    container_name: wp_certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  mysql_data:
    name: wp_mysql_data
```

---

## Phase 5: Domain + SSL + Nginx

### 5.1 สร้าง Nginx Config (HTTP ก่อน)

```bash
nano /opt/wordpress/nginx/conf.d/wordpress.conf
```

```nginx
# Rate Limiting Zones
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
limit_req_zone $binary_remote_addr zone=api:10m rate=20r/m;
limit_conn_zone $binary_remote_addr zone=addr:10m;

# ─── HTTP → HTTPS Redirect ────────────────────────────────
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;

    # ACME Challenge (Certbot)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# ─── HTTPS Main Server ────────────────────────────────────
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # ─── SSL Configuration ──────────────────────────────
    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/yourdomain.com/chain.pem;

    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # ─── Security Headers ───────────────────────────────
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:;" always;

    # ซ่อน Nginx version
    server_tokens off;

    # ─── WordPress Security Rules ───────────────────────
    # ป้องกัน wp-config.php
    location ~* wp-config.php {
        deny all;
    }

    # ป้องกัน .htaccess และ dotfiles
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # ป้องกัน xmlrpc.php (Brute Force)
    location = /xmlrpc.php {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Rate Limit wp-login.php
    location = /wp-login.php {
        limit_req zone=login burst=3 nodelay;
        limit_conn addr 5;
        proxy_pass http://wp_app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Rate Limit REST API
    location /wp-json/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://wp_app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Block Bad Bots
    location /wp-admin/install.php { deny all; }
    location /wp-admin/upgrade.php { deny all; }

    # Static Files Cache
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|txt|woff|woff2|ttf|svg)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
        proxy_pass http://wp_app:80;
        proxy_set_header Host $host;
    }

    # ─── Main Proxy ─────────────────────────────────────
    location / {
        proxy_pass http://wp_app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;

        proxy_buffer_size   128k;
        proxy_buffers       4 256k;
        proxy_busy_buffers_size 256k;

        proxy_connect_timeout 60s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        client_max_body_size 64M;
    }

    # Logging
    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log warn;
}
```

### 5.2 รัน Stack และขอ SSL

```bash
cd /opt/wordpress

# Step 1: รัน Nginx ก่อน (ยังไม่มี SSL)
# Comment บล็อก HTTPS ใน nginx config ก่อนชั่วคราว
docker compose up -d nginx

# Step 2: ขอ SSL Certificate
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email ${LETSENCRYPT_EMAIL} \
  --agree-tos \
  --no-eff-email \
  --force-renewal \
  -d yourdomain.com \
  -d www.yourdomain.com

# Step 3: เปิด HTTPS Config แล้ว Reload
docker compose exec nginx nginx -s reload

# Step 4: รัน Stack ทั้งหมด
docker compose up -d

# ตรวจสอบสถานะ
docker compose ps
docker compose logs --tail=20
```

### 5.3 ทดสอบ SSL Grade

```bash
# ทดสอบออนไลน์ที่ https://www.ssllabs.com/ssltest/
# หรือใช้ curl
curl -I https://yourdomain.com

# ตรวจสอบ Security Headers
curl -I https://yourdomain.com | grep -E "Strict|X-Frame|X-Content|X-XSS"
```

---

## Phase 6: Firewall & Network Security

### 6.1 ตั้งค่า UFW Firewall

```bash
# ติดตั้ง UFW
sudo apt install -y ufw

# ตั้งค่า Default Policy
sudo ufw default deny incoming
sudo ufw default allow outgoing

# อนุญาต SSH Port (ใช้ Port ที่ตั้งไว้ใน Phase 1)
sudo ufw allow 2222/tcp comment 'SSH Custom Port'

# อนุญาต HTTP และ HTTPS
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# เปิดใช้งาน UFW
sudo ufw enable

# ตรวจสอบ Rules
sudo ufw status verbose
```

### 6.2 ป้องกัน DDoS ด้วย UFW Rules

```bash
# จำกัด Connection ต่อ IP
sudo ufw limit ssh

# Block IP ที่ Scan Port
sudo ufw deny from 0.0.0.0/0 to any port 23 comment 'Block Telnet'
sudo ufw deny from 0.0.0.0/0 to any port 3306 comment 'Block MySQL External'

# ตั้ง Rate Limit ด้วย iptables
sudo iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# บันทึก iptables Rules
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### 6.3 ติดตั้ง CrowdSec (Modern IPS)

```bash
# ติดตั้ง CrowdSec
curl -s https://install.crowdsec.net | sudo sh

# ติดตั้ง Nginx Bouncer
sudo apt install -y crowdsec-nginx-bouncer

# เพิ่ม Collections
sudo cscli collections install crowdsecurity/nginx
sudo cscli collections install crowdsecurity/wordpress

# ตรวจสอบสถานะ
sudo cscli decisions list
sudo cscli alerts list
```

---

## Phase 7: Monitoring & Logging

### 7.1 ติดตั้ง Monitoring Stack

```bash
mkdir -p /opt/monitoring
nano /opt/monitoring/docker-compose.yml
```

```yaml
networks:
  monitoring:
    driver: bridge

services:
  # ─── Prometheus (Metrics Collection) ─────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    networks:
      - monitoring
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  # ─── Grafana (Dashboard) ──────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    networks:
      - monitoring
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=StrongGrafanaPass!
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_DOMAIN=yourdomain.com

  # ─── Node Exporter (System Metrics) ──────────────────
  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    restart: unless-stopped
    networks:
      - monitoring
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'

  # ─── cAdvisor (Container Metrics) ────────────────────
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    networks:
      - monitoring
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus_data:
  grafana_data:
```

```bash
cd /opt/monitoring
docker compose up -d

# เข้า Grafana ผ่าน SSH Tunnel
# ssh -L 3000:localhost:3000 deploy@your-server-ip -p 2222
# แล้วเปิด http://localhost:3000
```

### 7.2 Log Management

```bash
# ดู Log แบบ Real-time
docker compose -f /opt/wordpress/docker-compose.yml logs -f

# ดู Log เฉพาะ Service
docker compose -f /opt/wordpress/docker-compose.yml logs -f nginx
docker compose -f /opt/wordpress/docker-compose.yml logs -f wordpress
docker compose -f /opt/wordpress/docker-compose.yml logs -f db

# ดู System Log
sudo tail -f /var/log/syslog
sudo tail -f /var/log/auth.log

# ดู Fail2Ban Log
sudo fail2ban-client status
sudo tail -f /var/log/fail2ban.log
```

---

## Phase 8: Backup & Disaster Recovery

### 8.1 สร้าง Backup Script

```bash
nano /opt/wordpress/backup.sh
```

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════
# WordPress Docker Backup Script
# ═══════════════════════════════════════════════════════

set -euo pipefail

# ─── Config ─────────────────────────────────────────────
BACKUP_DIR="/opt/backups/wordpress"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="${BACKUP_DIR}/${TIMESTAMP}"
KEEP_DAYS=30
COMPOSE_DIR="/opt/wordpress"

# Load .env
source "${COMPOSE_DIR}/.env"

# ─── Functions ──────────────────────────────────────────
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"; }
error() { echo "[ERROR] $1" >&2; exit 1; }

# ─── Main ───────────────────────────────────────────────
log "Starting backup: ${TIMESTAMP}"
mkdir -p "${BACKUP_PATH}"

# 1. Backup Database
log "Backing up MySQL database..."
docker exec wp_mysql mysqldump \
  -u root \
  -p"${MYSQL_ROOT_PASSWORD}" \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  2>/dev/null | gzip > "${BACKUP_PATH}/database.sql.gz"

# 2. Backup WordPress Files
log "Backing up WordPress files..."
tar -czf "${BACKUP_PATH}/wordpress_files.tar.gz" \
  -C "${COMPOSE_DIR}" wordpress_files \
  --exclude="wordpress_files/wp-content/cache/*" \
  2>/dev/null

# 3. Backup Config Files
log "Backing up configuration..."
tar -czf "${BACKUP_PATH}/config.tar.gz" \
  -C "${COMPOSE_DIR}" \
  docker-compose.yml \
  nginx/ \
  2>/dev/null

# 4. Verify Backup
log "Verifying backup..."
for f in database.sql.gz wordpress_files.tar.gz config.tar.gz; do
  if [ -f "${BACKUP_PATH}/${f}" ]; then
    SIZE=$(du -sh "${BACKUP_PATH}/${f}" | cut -f1)
    log "  ✓ ${f} (${SIZE})"
  else
    error "Backup file missing: ${f}"
  fi
done

# 5. Cleanup Old Backups
log "Cleaning up backups older than ${KEEP_DAYS} days..."
find "${BACKUP_DIR}" -maxdepth 1 -type d -mtime +${KEEP_DAYS} -exec rm -rf {} \;

# 6. Show Summary
TOTAL_SIZE=$(du -sh "${BACKUP_PATH}" | cut -f1)
log "Backup complete! Total size: ${TOTAL_SIZE}"
log "Location: ${BACKUP_PATH}"
```

```bash
# ให้สิทธิ์รัน
chmod +x /opt/wordpress/backup.sh

# ทดสอบ
sudo /opt/wordpress/backup.sh

# ตั้ง Cron — Backup ทุกวันตี 2
echo "0 2 * * * deploy /opt/wordpress/backup.sh >> /var/log/wp-backup.log 2>&1" | \
  sudo tee /etc/cron.d/wp-backup
```

### 8.2 Restore จาก Backup

```bash
# ─── Restore Database ───────────────────────────────────
BACKUP_DATE="20240101_020000"   # เปลี่ยนเป็นวันที่ต้องการ

# Stop WordPress ก่อน
docker compose -f /opt/wordpress/docker-compose.yml stop wordpress

# Restore DB
zcat /opt/backups/wordpress/${BACKUP_DATE}/database.sql.gz | \
  docker exec -i wp_mysql mysql \
  -u root \
  -p"${MYSQL_ROOT_PASSWORD}"

# ─── Restore Files ──────────────────────────────────────
tar -xzf /opt/backups/wordpress/${BACKUP_DATE}/wordpress_files.tar.gz \
  -C /opt/wordpress/

# Start WordPress
docker compose -f /opt/wordpress/docker-compose.yml start wordpress

echo "Restore complete!"
```

---

## Phase 9: Production Checklist

### ✅ Linux Hardening

- [ ] อัปเดต OS ครบถ้วน
- [ ] สร้าง Non-root User
- [ ] เปิดใช้ SSH Key Authentication
- [ ] ปิด Password SSH Login
- [ ] เปลี่ยน SSH Port จาก 22
- [ ] ติดตั้ง Fail2Ban
- [ ] ตั้งค่า sysctl Security Parameters
- [ ] ลบ Package ที่ไม่จำเป็น

### ✅ Docker Security

- [ ] ตั้งค่า daemon.json
- [ ] ใช้ `no-new-privileges: true`
- [ ] Drop unnecessary Capabilities
- [ ] ตั้ง Resource Limits
- [ ] สแกน Image ด้วย Trivy ก่อน Deploy
- [ ] ใช้ Non-root User ใน Container

### ✅ Application Security

- [ ] ใช้ `.env` file (ไม่ Hardcode secrets)
- [ ] `.env` อยู่ใน `.gitignore`
- [ ] Pin Image Versions
- [ ] เปิด Health Checks
- [ ] Database ไม่ expose Port ออก Public
- [ ] เปลี่ยน WordPress Table Prefix
- [ ] ปิด File Editor ใน WordPress
- [ ] Block xmlrpc.php

### ✅ Network & SSL

- [ ] UFW Firewall เปิดเฉพาะ Port ที่จำเป็น
- [ ] SSL Grade A หรือ A+
- [ ] HSTS เปิดใช้งาน
- [ ] Security Headers ครบ
- [ ] Rate Limiting บน wp-login.php
- [ ] HTTP → HTTPS Redirect

### ✅ Monitoring & Backup

- [ ] Log ทุก Service
- [ ] Backup อัตโนมัติทุกวัน
- [ ] ทดสอบ Restore แล้ว
- [ ] Alert เมื่อ Disk ใกล้เต็ม
- [ ] Monitor Container Health

---

## 🚨 Emergency Commands

```bash
# หยุด Stack ฉุกเฉิน
docker compose -f /opt/wordpress/docker-compose.yml down

# Block IP ฉุกเฉิน
sudo ufw deny from ATTACKER_IP

# ดู Active Connection
sudo ss -tnp | grep ESTABLISHED
sudo netstat -an | grep :443 | wc -l

# ดู Top Process
docker stats --no-stream
htop

# ดู Disk Usage
df -h
docker system df

# ล้าง Docker Cache (กรณี Disk เต็ม)
docker system prune -f
```

---

> 🔒 **Security Note:** คู่มือนี้เป็น Baseline สำหรับ Production  
> ควรทำ Security Audit เพิ่มเติมและปรับตาม Compliance ขององค์กร (PCI-DSS, ISO27001 ฯลฯ)  
> อัปเดต Dependencies และ Patch ทุก OS สม่ำเสมอ
