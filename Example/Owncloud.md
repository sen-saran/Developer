# 🗄️ ownCloud Infinite Scale (oCIS) + NT Object Storage

> การติดตั้งและรัน ownCloud Infinite Scale ด้วย Docker  
> รองรับ NT Object Storage 2 Keys + 32 Users พร้อม Quota

---

## 🏗️ โครงสร้างโปรเจ็กต์

```
ocis-docker/
├── docker-compose.yml          # สภาพแวดล้อม Development
├── docker-compose.prod.yml     # สภาพแวดล้อม Production
├── .env                        # ตัวแปรสภาพแวดล้อม (Secrets)
├── .dockerignore
├── scripts/
│   ├── create_users.sh         # สร้าง 32 Users พร้อม Quota
│   └── check_disk.sh           # Monitor Disk Usage
├── docker/
│   ├── nginx/
│   │   └── ocis.conf           # Nginx Reverse Proxy Config
│   ├── rclone/
│   │   └── rclone.conf         # NT Object Storage 2 Keys
│   └── ocis/
│       └── ocis.yaml           # oCIS Config (optional override)
├── data/
│   ├── ocis/                   # oCIS temp data
│   └── redis/                  # Redis persistence
├── certbot/
│   ├── conf/                   # SSL Certificates
│   └── www/                    # ACME Challenge
└── logs/
    ├── nginx/
    └── ocis/
```

---

## ขั้นตอนที่ 1: สร้างโฟลเดอร์โปรเจ็กต์

```bash
mkdir ocis-docker && cd ocis-docker

# สร้างโครงสร้างโฟลเดอร์ทั้งหมด
mkdir -p scripts docker/nginx docker/rclone docker/ocis
mkdir -p data/ocis data/redis
mkdir -p certbot/conf certbot/www
mkdir -p logs/nginx logs/ocis
```

---

## ขั้นตอนที่ 2: สร้างไฟล์ .env

สร้างไฟล์ `.env` ในโฟลเดอร์หลักของโปรเจ็กต์:

```env
# ─── Domain & SSL ─────────────────────────────────────
OCIS_DOMAIN=cloud.yourdomain.com
LETSENCRYPT_EMAIL=admin@yourdomain.com

# ─── oCIS Admin ───────────────────────────────────────
OCIS_ADMIN_USER=admin
OCIS_ADMIN_PASSWORD=Str0ng!Admin#2024

# ─── oCIS Security Keys ───────────────────────────────
# สร้างด้วย: openssl rand -hex 32
OCIS_JWT_SECRET=change-this-to-random-64-char-string-here-now
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
```

```bash
# จำกัดสิทธิ์ไฟล์ .env
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
no_check_bucket = false

# ─── NT Object Storage Key 2 (Secondary 5TB) ──────────
[nt_key2]
type = s3
provider = Other
access_key_id = YOUR_NT_ACCESS_KEY_2
secret_access_key = YOUR_NT_SECRET_KEY_2
endpoint = https://object.storage.nt.th
region = auto
acl = private
no_check_bucket = false
```

---

## ขั้นตอนที่ 4: ทดสอบการเชื่อมต่อ NT Object Storage

```bash
# ทดสอบ Key 1
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  lsd nt_key1:

# ทดสอบ Key 2
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  lsd nt_key2:

# สร้าง Bucket (ถ้ายังไม่มี)
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  mkdir nt_key1:ocis-primary-bucket

docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  mkdir nt_key2:ocis-secondary-bucket

# ตรวจสอบ Bucket ที่สร้าง
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  ls nt_key1:ocis-primary-bucket
```

---

## ขั้นตอนที่ 5: สร้าง Nginx Config

สร้างไฟล์ `docker/nginx/ocis.conf`:

```nginx
# Rate Limiting
limit_req_zone $binary_remote_addr zone=ocis_login:10m rate=10r/m;
limit_req_zone $binary_remote_addr zone=ocis_api:10m rate=60r/m;

# ─── HTTP → HTTPS Redirect ────────────────────────────
server {
    listen 80;
    listen [::]:80;
    server_name cloud.yourdomain.com;

    # Certbot ACME Challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# ─── HTTPS Main ───────────────────────────────────────
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name cloud.yourdomain.com;

    # SSL
    ssl_certificate     /etc/letsencrypt/live/cloud.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cloud.yourdomain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer" always;
    server_tokens off;

    # oCIS ต้องการ unlimited upload size
    client_max_body_size 0;
    proxy_request_buffering off;

    # Rate Limit Login
    location /signin/ {
        limit_req zone=ocis_login burst=5 nodelay;
        proxy_pass https://ocis:9200;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Rate Limit API
    location /api/ {
        limit_req zone=ocis_api burst=30 nodelay;
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

        # WebSocket สำหรับ Real-time notifications
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

## ขั้นตอนที่ 6: สร้างไฟล์ docker-compose.yml (Development)

```yaml
networks:
  ocis_net:
    name: ocis_net
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

      # Logging
      - OCIS_LOG_LEVEL=warn
      - OCIS_LOG_FILE=/var/log/ocis/ocis.log

    volumes:
      - ./docker/ocis:/etc/ocis
      - ./data/ocis:/var/lib/ocis
      - ./logs/ocis:/var/log/ocis
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
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 128M
    depends_on:
      - ocis

  # ─── Certbot SSL ─────────────────────────────────────
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

  # ─── Rclone Sync (Key1 → Key2) ───────────────────────
  rclone_sync:
    image: rclone/rclone:latest
    container_name: ocis_rclone_sync
    restart: unless-stopped
    volumes:
      - ./docker/rclone:/config/rclone:ro
    environment:
      - S3_KEY1_BUCKET=${S3_KEY1_BUCKET}
      - S3_KEY2_BUCKET=${S3_KEY2_BUCKET}
    entrypoint: >
      /bin/sh -c "
      while true; do
        echo '[Rclone] Starting sync: Key1 → Key2';
        rclone sync nt_key1:${S3_KEY1_BUCKET} nt_key2:${S3_KEY2_BUCKET}
          --progress --log-level INFO;
        echo '[Rclone] Sync complete. Next in 6h.';
        sleep 21600;
      done"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
```

---

## ขั้นตอนที่ 7: สร้างไฟล์ .dockerignore

สร้างไฟล์ `.dockerignore`:

```
# Secrets
.env
.env.*

# Data volumes (ไม่ใส่ใน Image)
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

## ขั้นตอนที่ 8: ขอ SSL Certificate

```bash
# Step 1: รัน Nginx ก่อน (ชั่วคราว HTTP เท่านั้น)
# Comment บล็อก server 443 ใน ocis.conf ออกก่อน แล้ว:
docker compose up -d nginx

# Step 2: ขอ Certificate จาก Let's Encrypt
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email ${LETSENCRYPT_EMAIL} \
  --agree-tos \
  --no-eff-email \
  -d cloud.yourdomain.com

# Step 3: เปิดบล็อก 443 ใน ocis.conf แล้ว Reload
docker compose exec nginx nginx -s reload
```

---

## ขั้นตอนที่ 9: รันแอปพลิเคชัน

```bash
# รัน ทุก Services
docker compose up -d

# ตรวจสอบสถานะ
docker compose ps

# ดู Log แบบ Real-time
docker compose logs -f

# ดู Log เฉพาะ Service
docker compose logs -f ocis
docker compose logs -f redis
docker compose logs -f nginx
```

ผลลัพธ์ที่ควรเห็น:

```
NAME              STATUS          PORTS
ocis              Up (healthy)
ocis_redis        Up (healthy)
ocis_nginx        Up              0.0.0.0:80->80, 0.0.0.0:443->443
ocis_certbot      Up
ocis_rclone_sync  Up
```

---

## ขั้นตอนที่ 10: ทดสอบแอปพลิเคชัน

```bash
# ทดสอบ HTTP → HTTPS Redirect
curl -I http://cloud.yourdomain.com

# ทดสอบ HTTPS
curl -I https://cloud.yourdomain.com

# ทดสอบ Health Check
curl -fk https://cloud.yourdomain.com/healthz

# ทดสอบ oCIS โดยตรง (bypass Nginx)
docker exec ocis curl -sk https://localhost:9200/healthz
```

เปิดเบราว์เซอร์และไปที่:
- **oCIS Web UI:** `https://cloud.yourdomain.com`
- **Login:** admin / ตาม `OCIS_ADMIN_PASSWORD` ใน `.env`

---

## ขั้นตอนที่ 11: สร้าง Users พร้อม Quota

สร้างไฟล์ `scripts/create_users.sh`:

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════
# สร้าง Users พร้อม Quota สำหรับ oCIS
# ═══════════════════════════════════════════════════════

set -euo pipefail

# โหลด .env
source "$(dirname "$0")/../.env"

OCIS_URL="https://${OCIS_DOMAIN}"
ADMIN="${OCIS_ADMIN_USER}"
PASS="${OCIS_ADMIN_PASSWORD}"

# ─── Quota Presets ──────────────────────────────────────
QB_10GB=10737418240    #  10 GB
QB_20GB=21474836480    #  20 GB
QB_50GB=53687091200    #  50 GB
QB_100GB=107374182400  # 100 GB

# ─── รายชื่อ Users ─────────────────────────────────────
# รูปแบบ: "username|displayname|email|password|quota_bytes"
USERS=(
  "user01|สมชาย ใจดี|user01@yourdomain.com|Pass@User01!|${QB_10GB}"
  "user02|สมหญิง ดีใจ|user02@yourdomain.com|Pass@User02!|${QB_10GB}"
  "user03|นายทดสอบ หนึ่ง|user03@yourdomain.com|Pass@User03!|${QB_20GB}"
  "user04|นางทดสอบ สอง|user04@yourdomain.com|Pass@User04!|${QB_50GB}"
  # เพิ่มต่อจนครบ 32 คน...
)

# ─── Functions ──────────────────────────────────────────
log()   { echo "[$(date '+%H:%M:%S')] $1"; }
ok()    { echo "  ✓ $1"; }
fail()  { echo "  ✗ $1" >&2; }

create_user() {
  local USERNAME="$1" DISPLAY="$2" EMAIL="$3" PASSWORD="$4" QUOTA="$5"

  log "Creating: ${USERNAME} (${DISPLAY})"

  # สร้าง User ผ่าน Graph API
  HTTP_CODE=$(curl -sk -o /tmp/ocis_resp.json -w "%{http_code}" \
    -X POST \
    -u "${ADMIN}:${PASS}" \
    -H "Content-Type: application/json" \
    "${OCIS_URL}/graph/v1.0/users" \
    -d "{
      \"onPremisesSamAccountName\": \"${USERNAME}\",
      \"displayName\": \"${DISPLAY}\",
      \"mail\": \"${EMAIL}\",
      \"passwordProfile\": { \"password\": \"${PASSWORD}\" }
    }")

  if [ "$HTTP_CODE" != "201" ]; then
    fail "Create failed (HTTP ${HTTP_CODE}): ${USERNAME}"
    return
  fi
  ok "User created: ${USERNAME}"

  # ดึง Drive ID ของ Personal Space
  sleep 2  # รอ oCIS สร้าง Space
  DRIVE_ID=$(curl -sk \
    -u "${ADMIN}:${PASS}" \
    "${OCIS_URL}/graph/v1.0/users/${USERNAME}/drives" \
    | grep -o '"id":"[^"]*"' | head -1 | cut -d'"' -f4)

  if [ -z "$DRIVE_ID" ]; then
    fail "Cannot get Drive ID for: ${USERNAME}"
    return
  fi

  # ตั้ง Quota
  curl -sk -X PATCH \
    -u "${ADMIN}:${PASS}" \
    -H "Content-Type: application/json" \
    "${OCIS_URL}/graph/v1.0/drives/${DRIVE_ID}" \
    -d "{\"quota\": {\"total\": ${QUOTA}}}" > /dev/null

  QUOTA_GB=$(( QUOTA / 1073741824 ))
  ok "Quota set: ${QUOTA_GB} GB → ${USERNAME}"
}

# ─── Main ────────────────────────────────────────────────
echo "======================================"
echo "  oCIS User Creation Script"
echo "  Target: ${OCIS_URL}"
echo "======================================"
echo ""

for USER_DATA in "${USERS[@]}"; do
  IFS='|' read -r USERNAME DISPLAY EMAIL PASSWORD QUOTA <<< "$USER_DATA"
  create_user "$USERNAME" "$DISPLAY" "$EMAIL" "$PASSWORD" "$QUOTA"
  sleep 0.5
  echo ""
done

echo "======================================"
echo "  Done! Created ${#USERS[@]} users."
echo "======================================"
```

```bash
chmod +x scripts/create_users.sh

# รัน Script
bash scripts/create_users.sh
```

---

## ขั้นตอนที่ 12: เพิ่ม/ลด/แก้ไข User Quota ภายหลัง

```bash
# ─── ดู Users ทั้งหมด ────────────────────────────────
curl -sk -u admin:password \
  https://cloud.yourdomain.com/graph/v1.0/users \
  | python3 -m json.tool | grep -E "displayName|mail|id"

# ─── ดู Drive/Space ของ User ─────────────────────────
curl -sk -u admin:password \
  "https://cloud.yourdomain.com/graph/v1.0/users/user01/drives" \
  | python3 -m json.tool

# ─── เปลี่ยน Quota เป็น 50GB ─────────────────────────
curl -X PATCH \
  -u admin:password \
  -H "Content-Type: application/json" \
  "https://cloud.yourdomain.com/graph/v1.0/drives/{DRIVE_ID}" \
  -d '{"quota": {"total": 53687091200}}'

# ─── ลบ User ─────────────────────────────────────────
curl -X DELETE \
  -u admin:password \
  "https://cloud.yourdomain.com/graph/v1.0/users/user01"

# ─── Reset Password ───────────────────────────────────
curl -X PATCH \
  -u admin:password \
  -H "Content-Type: application/json" \
  "https://cloud.yourdomain.com/graph/v1.0/users/user01" \
  -d '{"passwordProfile": {"password": "NewPass@2024!"}}'
```

---

## ขั้นตอนที่ 13: เพิ่ม NT Object Storage Key ในอนาคต

```bash
# 1. เพิ่ม Key 3 ใน .env
nano .env
```

```env
# เพิ่ม Key 3 ใหม่
S3_KEY3_ACCESS_KEY=YOUR_NT_ACCESS_KEY_3
S3_KEY3_SECRET_KEY=YOUR_NT_SECRET_KEY_3
S3_KEY3_BUCKET=ocis-bucket-key3
S3_KEY3_ENDPOINT=https://object.storage.nt.th
```

```bash
# 2. เพิ่ม rclone config
nano docker/rclone/rclone.conf
```

```ini
[nt_key3]
type = s3
provider = Other
access_key_id = YOUR_NT_ACCESS_KEY_3
secret_access_key = YOUR_NT_SECRET_KEY_3
endpoint = https://object.storage.nt.th
region = auto
acl = private
```

```bash
# 3. สร้าง Bucket ใหม่
docker run --rm \
  -v $(pwd)/docker/rclone:/config/rclone \
  rclone/rclone:latest \
  mkdir nt_key3:ocis-bucket-key3

# 4. Reload rclone container
docker compose up -d --no-deps rclone_sync
```

---

## คำสั่งที่ใช้บ่อย

```bash
# ─── จัดการ Services ─────────────────────────────────
docker compose up -d                          # Start ทั้งหมด
docker compose down                           # Stop ทั้งหมด
docker compose restart ocis                   # Restart oCIS
docker compose pull && docker compose up -d   # Update Images

# ─── Debug ───────────────────────────────────────────
docker compose logs -f ocis                   # Log oCIS
docker compose logs -f nginx                  # Log Nginx
docker compose exec ocis sh                   # เข้า Shell oCIS
docker compose exec redis redis-cli -a $REDIS_PASSWORD ping

# ─── Monitor Resource ─────────────────────────────────
docker stats --no-stream                      # CPU/RAM usage
docker system df                              # Disk usage
df -h                                         # VM Disk

# ─── oCIS Maintenance ────────────────────────────────
docker exec ocis ocis storage-users           # ดู Storage info
docker exec ocis ocis idm                     # User management CLI
```

---

## Quota Reference Table

| ขนาด | bytes |
|:---|:---|
| 5 GB | `5368709120` |
| 10 GB | `10737418240` |
| 20 GB | `21474836480` |
| 50 GB | `53687091200` |
| 100 GB | `107374182400` |
| 200 GB | `214748364800` |
| ไม่จำกัด | `0` |

---

## Architecture สรุป

```
User Browser
     ↓ HTTPS
Nginx (Port 443)
     ↓ Proxy
oCIS Core ←→ Redis (Cache)
     ↓
NT Object Storage
├── Key 1: 5 TB (Primary — ไฟล์จริง)
└── Key 2: 5 TB (Backup — Rclone Sync ทุก 6h)
```

> 📝 **หมายเหตุ:** แทนที่ `cloud.yourdomain.com` และ `yourdomain.com`  
> ด้วยโดเมนจริงของคุณในทุกไฟล์ก่อน Deploy
