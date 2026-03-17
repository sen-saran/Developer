# 🗄️ ownCloud Infinite Scale (oCIS) + Portainer + Docker Hub

> การติดตั้งและรัน ownCloud Infinite Scale ด้วย Docker  
> พร้อม Portainer Web UI และการจัดการ Image บน Docker Hub

---

## 🏗️ โครงสร้างโปรเจ็กต์

```
ocis-docker/
├── docker-compose.yml              # สภาพแวดล้อม Development
├── docker-compose.prod.yml         # สภาพแวดล้อม Production
├── .env                            # ตัวแปรสภาพแวดล้อม (Secrets)
├── .dockerignore                   # ไฟล์ Docker ignore
├── scripts/
│   ├── create_users.sh             # สร้าง Users พร้อม Quota
│   └── backup.sh                   # Backup อัตโนมัติ
└── docker/
    ├── nginx/
    │   └── ocis.conf               # Nginx Reverse Proxy Config
    ├── rclone/
    │   └── rclone.conf             # NT Object Storage 2 Keys
    └── ocis/
        └── ocis.yaml               # oCIS Config (optional)
```

---

## ขั้นตอนที่ 1: สร้างโฟลเดอร์โปรเจ็กต์

```bash
mkdir ocis-docker && cd ocis-docker

# สร้างโครงสร้างโฟลเดอร์ทั้งหมด
mkdir -p scripts
mkdir -p docker/nginx
mkdir -p docker/rclone
mkdir -p docker/ocis
mkdir -p data/ocis
mkdir -p data/redis
mkdir -p certbot/conf
mkdir -p certbot/www
mkdir -p logs/nginx
mkdir -p logs/ocis
```

---

## ขั้นตอนที่ 2: สร้างไฟล์ .env

สร้างไฟล์ `.env` ในโฟลเดอร์หลักของโปรเจ็กต์:

```env
# ─── Domain & SSL ─────────────────────────────────────
OCIS_DOMAIN=owncloud.dms.go.th
LETSENCRYPT_EMAIL=admin@dms.mail.go.th

# ─── oCIS Admin ───────────────────────────────────────
OCIS_ADMIN_USER=admin
OCIS_ADMIN_PASSWORD=Str0ng!Admin#2024

# ─── oCIS Security Keys ───────────────────────────────
# สร้างด้วยคำสั่ง: openssl rand -hex 32
OCIS_JWT_SECRET=change-this-to-random-64-char-string-here
OCIS_MACHINE_AUTH_API_KEY=change-this-to-another-random-string

# ─── NT Object Storage Key 1 (Primary 5TB) ────────────
S3_KEY1_ACCESS_KEY=YOUR_NT_ACCESS_KEY_1
S3_KEY1_SECRET_KEY=YOUR_NT_SECRET_KEY_1
S3_KEY1_BUCKET=ocis-primary-bucket
S3_KEY1_ENDPOINT=https://object.storage.nt.th

# ─── NT Object Storage Key 2 (Secondary 5TB) ──────────
S3_KEY2_ACCESS_KEY=YOUR_NT_ACCESS_KEY_2
S3_KEY2_SECRET_KEY=YOUR_NT_SECRET_KEY_2
S3_KEY2_BUCKET=ocis-secondary-bucket
S3_KEY2_ENDPOINT=https://object.storage.nt.th

# ─── Redis ────────────────────────────────────────────
REDIS_PASSWORD=Str0ng!Redis#2024

# ─── Portainer ────────────────────────────────────────
PORTAINER_PORT=9553
```

```bash
# จำกัดสิทธิ์ไฟล์ .env ให้เฉพาะเจ้าของเท่านั้น
chmod 600 .env
```

---

## ขั้นตอนที่ 3: สร้าง Rclone Config สำหรับ NT Object Storage

สร้างไฟล์ `docker/rclone/rclone.conf`:

```ini
# ─── NT Object Storage Key 1 (Primary 5TB) ────────────
[nt_key1]
type = s3
provider = Other
access_key_id = YOUR_NT_ACCESS_KEY_1
secret_access_key = YOUR_NT_SECRET_KEY_1
endpoint = https://object.storage.nt.th
region = auto
acl = private

# ─── NT Object Storage Key 2 (Secondary 5TB) ──────────
[nt_key2]
type = s3
provider = Other
access_key_id = YOUR_NT_ACCESS_KEY_2
secret_access_key = YOUR_NT_SECRET_KEY_2
endpoint = https://object.storage.nt.th
region = auto
acl = private
```

```bash
# ทดสอบการเชื่อมต่อ Key 1
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  lsd nt_key1:

# ทดสอบการเชื่อมต่อ Key 2
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  lsd nt_key2:

# สร้าง Bucket ถ้ายังไม่มี
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  mkdir nt_key1:ocis-primary-bucket

docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  mkdir nt_key2:ocis-secondary-bucket
```

---

## ขั้นตอนที่ 4: สร้าง Nginx Config

สร้างไฟล์ `docker/nginx/ocis.conf`:

```nginx
# Rate Limiting Zones
limit_req_zone $binary_remote_addr zone=ocis_login:10m rate=10r/m;
limit_req_zone $binary_remote_addr zone=ocis_api:10m   rate=60r/m;

# ─── HTTP → HTTPS Redirect ────────────────────────────
server {
    listen 80;
    listen [::]:80;
    server_name owncloud.dms.go.th;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# ─── HTTPS Main Server ────────────────────────────────
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name owncloud.dms.go.th;

    ssl_certificate     /etc/letsencrypt/live/owncloud.dms.go.th/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/owncloud.dms.go.th/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options            "SAMEORIGIN"    always;
    add_header X-Content-Type-Options     "nosniff"       always;
    add_header Referrer-Policy            "no-referrer"   always;
    server_tokens off;

    # oCIS ต้องการ unlimited upload size
    client_max_body_size  0;
    proxy_request_buffering off;

    # Rate Limit บน Login
    location /signin/ {
        limit_req zone=ocis_login burst=5 nodelay;
        proxy_pass https://ocis:9200;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Main Proxy
    location / {
        proxy_pass https://ocis:9200;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket สำหรับ Real-time
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_connect_timeout 300s;
        proxy_send_timeout    300s;
        proxy_read_timeout    300s;
        proxy_buffering       off;
    }

    access_log /var/log/nginx/ocis_access.log;
    error_log  /var/log/nginx/ocis_error.log warn;
}
```

---

## ขั้นตอนที่ 5: สร้างไฟล์ docker-compose.yml

```yaml
networks:
  ocis_net:
    name: ocis_net
    driver: bridge
  portainer_net:
    name: portainer_net
    driver: bridge

services:

  # ─── oCIS Core ──────────────────────────────────────
  ocis:
    image: owncloud/ocis:latest
    container_name: ocis
    restart: unless-stopped
    networks:
      - ocis_net
    entrypoint:
      - /bin/sh
    command: ["-c", "ocis init || true && ocis server"]
    environment:
      # URL
      - OCIS_URL=https://${OCIS_DOMAIN}
      - OCIS_INSECURE=false
      - PROXY_HTTP_ADDR=0.0.0.0:9200
      # Security
      - OCIS_JWT_SECRET=${OCIS_JWT_SECRET}
      - OCIS_MACHINE_AUTH_API_KEY=${OCIS_MACHINE_AUTH_API_KEY}
      # Admin
      - IDM_ADMIN_PASSWORD=${OCIS_ADMIN_PASSWORD}
      # S3 Storage (NT Object Storage Key 1)
      - STORAGE_USERS_DRIVER=s3ng
      - STORAGE_USERS_S3NG_ENDPOINT=${S3_KEY1_ENDPOINT}
      - STORAGE_USERS_S3NG_ACCESS_KEY=${S3_KEY1_ACCESS_KEY}
      - STORAGE_USERS_S3NG_SECRET_KEY=${S3_KEY1_SECRET_KEY}
      - STORAGE_USERS_S3NG_BUCKET=${S3_KEY1_BUCKET}
      - STORAGE_USERS_S3NG_REGION=auto
      # Redis Cache
      - CACHE_STORE=redis
      - REDIS_ADDR=redis:6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      # Log
      - OCIS_LOG_LEVEL=warn
      - OCIS_LOG_FILE=/var/log/ocis/ocis.log
    volumes:
      - ./docker/ocis:/etc/ocis
      - ./data/ocis:/var/lib/ocis
      - ./logs/ocis:/var/log/ocis
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 3500M
        reservations:
          cpus: '1.0'
          memory: 2G
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-fk", "https://localhost:9200/healthz"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─── Redis (Cache & Sessions) ────────────────────────
  redis:
    image: redis:7-alpine
    container_name: ocis_redis
    restart: unless-stopped
    networks:
      - ocis_net
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 400mb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
    volumes:
      - ./data/redis:/data
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 256M
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 15s
      timeout: 5s
      retries: 3

  # ─── Nginx Reverse Proxy ─────────────────────────────
  nginx:
    image: nginx:1.27-alpine
    container_name: ocis_nginx
    restart: unless-stopped
    networks:
      - ocis_net
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/ocis.conf:/etc/nginx/conf.d/ocis.conf:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./logs/nginx:/var/log/nginx
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 128M
    depends_on:
      - ocis

  # ─── Certbot (SSL Auto-Renew) ─────────────────────────
  certbot:
    image: certbot/certbot:latest
    container_name: ocis_certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: >
      /bin/sh -c "trap exit TERM;
      while :; do
        certbot renew;
        sleep 12h & wait $${!};
      done;"

  # ─── Rclone Sync (Key1 → Key2 ทุก 6 ชม.) ─────────────
  rclone_sync:
    image: rclone/rclone:latest
    container_name: ocis_rclone_sync
    restart: unless-stopped
    volumes:
      - ./docker/rclone:/config/rclone:ro
    entrypoint: >
      /bin/sh -c "
      while true; do
        echo '[Rclone] Syncing Key1 → Key2...';
        rclone sync nt_key1:${S3_KEY1_BUCKET} nt_key2:${S3_KEY2_BUCKET}
          --progress --log-level INFO;
        echo '[Rclone] Done. Next sync in 6h.';
        sleep 21600;
      done"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  # ─── Portainer (Docker Web UI) ───────────────────────
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    networks:
      - portainer_net
    ports:
      - "8000:8000"
      - "9553:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

volumes:
  portainer_data:
    name: portainer_data
```

---

## ขั้นตอนที่ 6: สร้างไฟล์ .dockerignore

```
# Secrets
.env
.env.*

# Data & Logs (ไม่ใส่ใน Image)
data/
logs/
certbot/conf/

# Git
.git/
.gitignore

# OS
.DS_Store
Thumbs.db

# Docs
*.md
README*
```

---

## ขั้นตอนที่ 7: ขอ SSL Certificate

```bash
# Step 1: รัน Nginx ชั่วคราว (HTTP เท่านั้น)
# comment บล็อก server 443 ใน ocis.conf ออกก่อน แล้วรัน
docker compose up -d nginx

# Step 2: ขอ Certificate
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email admin@dms.mail.go.th \
  --agree-tos \
  --no-eff-email \
  -d owncloud.dms.go.th

# Step 3: เปิดบล็อก 443 ใน ocis.conf แล้ว Reload
docker compose exec nginx nginx -s reload
```

---

## ขั้นตอนที่ 8: รันแอปพลิเคชัน

```bash
# รัน Services ทั้งหมด
docker compose up -d

# ตรวจสอบสถานะ
docker compose ps

# ดู Log แบบ Real-time
docker compose logs -f

# ดู Log เฉพาะ Service
docker compose logs -f ocis
docker compose logs -f nginx
docker compose logs -f redis
```

ผลลัพธ์ที่ควรเห็น:

```
NAME                STATUS          PORTS
ocis                Up (healthy)
ocis_redis          Up (healthy)
ocis_nginx          Up              0.0.0.0:80->80, 0.0.0.0:443->443
ocis_certbot        Up
ocis_rclone_sync    Up
portainer           Up              0.0.0.0:9553->9443
```

---

## ขั้นตอนที่ 9: ทดสอบแอปพลิเคชัน

```bash
# ทดสอบ HTTP → HTTPS Redirect
curl -I http://owncloud.dms.go.th
# ควรได้ 301 Moved Permanently

# ทดสอบ HTTPS
curl -I https://owncloud.dms.go.th
# ควรได้ 200 OK

# ทดสอบ Health Check
curl -fk https://owncloud.dms.go.th/healthz

# ทดสอบ Security Headers
curl -sk -D - https://owncloud.dms.go.th -o /dev/null | \
  grep -iE "strict-transport|x-frame|x-content"
```

เปิดเบราว์เซอร์และไปที่:
- **oCIS Web UI:** `https://owncloud.dms.go.th`
- **Login:** `admin` / ตาม `OCIS_ADMIN_PASSWORD` ใน `.env`

---

## ขั้นตอนที่ 10: สร้าง Users พร้อม Quota

สร้างไฟล์ `scripts/create_users.sh`:

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════
# สร้าง Users พร้อม Quota สำหรับ oCIS
# ═══════════════════════════════════════════════════════
set -euo pipefail

source "$(dirname "$0")/../.env"

OCIS_URL="https://${OCIS_DOMAIN}"
ADMIN="${OCIS_ADMIN_USER}"
PASS="${OCIS_ADMIN_PASSWORD}"

# Quota Presets (bytes)
QB_10GB=10737418240
QB_20GB=21474836480
QB_50GB=53687091200

# รูปแบบ: "username|displayname|email|password|quota"
USERS=(
  "user01|ชื่อ นามสกุล 01|user01@dms.mail.go.th|Pass@User01!|${QB_10GB}"
  "user02|ชื่อ นามสกุล 02|user02@dms.mail.go.th|Pass@User02!|${QB_10GB}"
  "user03|ชื่อ นามสกุล 03|user03@dms.mail.go.th|Pass@User03!|${QB_20GB}"
  # เพิ่มต่อจนครบ 32 คน
)

create_user() {
  local USERNAME="$1" DISPLAY="$2" EMAIL="$3" PASSWORD="$4" QUOTA="$5"
  echo "Creating: ${USERNAME}..."

  HTTP=$(curl -sk -o /tmp/resp.json -w "%{http_code}" \
    -X POST -u "${ADMIN}:${PASS}" \
    -H "Content-Type: application/json" \
    "${OCIS_URL}/graph/v1.0/users" \
    -d "{
      \"onPremisesSamAccountName\": \"${USERNAME}\",
      \"displayName\": \"${DISPLAY}\",
      \"mail\": \"${EMAIL}\",
      \"passwordProfile\": {\"password\": \"${PASSWORD}\"}
    }")

  [ "$HTTP" != "201" ] && echo "  ✗ Failed (HTTP $HTTP)" && return
  echo "  ✓ Created"

  sleep 2
  DRIVE_ID=$(curl -sk -u "${ADMIN}:${PASS}" \
    "${OCIS_URL}/graph/v1.0/users/${USERNAME}/drives" \
    | grep -o '"id":"[^"]*"' | head -1 | cut -d'"' -f4)

  [ -z "$DRIVE_ID" ] && echo "  ✗ Drive ID not found" && return

  curl -sk -X PATCH -u "${ADMIN}:${PASS}" \
    -H "Content-Type: application/json" \
    "${OCIS_URL}/graph/v1.0/drives/${DRIVE_ID}" \
    -d "{\"quota\": {\"total\": ${QUOTA}}}" > /dev/null

  echo "  ✓ Quota: $(( QUOTA / 1073741824 )) GB"
}

for USER_DATA in "${USERS[@]}"; do
  IFS='|' read -r USERNAME DISPLAY EMAIL PASSWORD QUOTA <<< "$USER_DATA"
  create_user "$USERNAME" "$DISPLAY" "$EMAIL" "$PASSWORD" "$QUOTA"
  sleep 0.5
done

echo "Done! Created ${#USERS[@]} users."
```

```bash
chmod +x scripts/create_users.sh
bash scripts/create_users.sh
```

---

## ขั้นตอนที่ 11: ใช้งาน Portainer (Docker Web UI)

### เข้าสู่ระบบ Portainer
- เปิดเว็บเบราว์เซอร์ไปที่ `https://localhost:9553`
- หรือผ่าน SSH Tunnel: `ssh -L 9553:localhost:9553 deploy@202.139.208.51 -p 2222`
- ตั้งค่า Admin Password ครั้งแรก
- เลือก **"Get Started"** → **"local"** เพื่อจัดการ Docker บนเครื่องนี้

### สิ่งที่ทำได้ใน Portainer

```
Containers  → ดู/Start/Stop/Restart Container ทุกตัว
Images      → ดู Image ที่ดาวน์โหลดมา / ลบ Image เก่า
Volumes     → ดูขนาด Volume / Backup
Networks    → ดู Network ที่สร้างไว้
Logs        → ดู Log แบบ Real-time ผ่านหน้าเว็บ
Stats       → ดู CPU/RAM/Network ของแต่ละ Container
```

---

## ขั้นตอนที่ 12: Push Image ขึ้น Docker Hub

```bash
# Step 1: Login Docker Hub
docker login
# ป้อน Username และ Password ของ Docker Hub

# Step 2: Tag Image
docker tag owncloud/ocis:latest \
  your_dockerhub_username/ocis-dms:latest

# Step 3: Push ขึ้น Docker Hub
docker push your_dockerhub_username/ocis-dms:latest

# ตัวอย่าง
docker push dmsgovth/ocis-dms:1.0.0

# Step 4: ดึงกลับมาใช้งาน (บน Server อื่น)
docker pull your_dockerhub_username/ocis-dms:latest
```

---

## ขั้นตอนที่ 13: เพิ่ม NT Object Storage Key ในอนาคต

```bash
# 1. เพิ่ม Key 3 ใน .env
nano .env
# เพิ่ม:
# S3_KEY3_ACCESS_KEY=YOUR_KEY_3
# S3_KEY3_SECRET_KEY=YOUR_SECRET_3
# S3_KEY3_BUCKET=ocis-bucket-key3
# S3_KEY3_ENDPOINT=https://object.storage.nt.th

# 2. เพิ่มใน rclone.conf
nano docker/rclone/rclone.conf

# 3. สร้าง Bucket ใหม่
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  mkdir nt_key3:ocis-bucket-key3

# 4. Reload rclone
docker compose up -d --no-deps rclone_sync
```

---

## คำสั่งที่ใช้บ่อย

```bash
# ─── จัดการ Services ─────────────────────────────────
docker compose up -d                          # Start ทั้งหมด
docker compose down                           # Stop ทั้งหมด
docker compose restart ocis                   # Restart oCIS
docker compose up -d --pull always            # Update Images ใหม่

# ─── Debug ───────────────────────────────────────────
docker compose logs -f ocis                   # Log oCIS
docker compose logs -f nginx                  # Log Nginx
docker compose exec ocis sh                   # เข้า Shell oCIS
docker compose exec redis \
  redis-cli -a ${REDIS_PASSWORD} ping         # ทดสอบ Redis

# ─── Monitor ─────────────────────────────────────────
docker stats --no-stream                      # CPU/RAM snapshot
docker system df                              # Disk usage
df -h                                         # VM Disk overview
```

---

## Quota Reference

| ขนาด | Bytes |
|:---|:---|
| 5 GB | `5368709120` |
| 10 GB | `10737418240` |
| 20 GB | `21474836480` |
| 50 GB | `53687091200` |
| 100 GB | `107374182400` |
| ไม่จำกัด | `0` |

---

## Architecture สรุป

```
User Browser
     ↓ HTTPS 443
Nginx (Reverse Proxy + SSL)
     ↓
oCIS Core  ←→  Redis (Cache)
     ↓
NT Object Storage
├── Key 1: 5 TB  ← Primary (ไฟล์จริง)
└── Key 2: 5 TB  ← Backup  (Rclone Sync ทุก 6h)

Portainer :9553  ← Web UI จัดการ Docker ทั้งหมด
```

---

> 📝 **หมายเหตุ:**
> - แทนที่ `your_dockerhub_username` ด้วย Username Docker Hub จริง
> - Portainer ควรเข้าถึงผ่าน SSH Tunnel เท่านั้น ไม่ควร expose Port 9553 ออก Internet โดยตรง
> - ไฟล์ `.env` ต้องไม่ commit เข้า Git เด็ดขาด
