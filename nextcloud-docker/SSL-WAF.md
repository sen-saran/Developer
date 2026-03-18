### Step 1: วาง SSL Certificate
## รวมไฟล์ใน Windows CMD
```
cd C:\Users\admin\Desktop\owncloud-stack\nginx\ssl
dir
# CMD
type certificate.crt Intemediate.crt > fullchain.pem

#  PowerShell 
Get-Content certificate.crt, Intemediate.crt | Set-Content fullchain.pem

# Copy ไปยัง Server ด้วย SCP
scp -P 717 fullchain.pem digitalsaran@159.138.255.6:~/nextcloud/nginx/ssl/
scp -P 717 privateKey.key digitalsaran@159.138.255.6:~/nextcloud/nginx/ssl/privkey.pem
ls -la ~/nextcloud/nginx/ssl/
chmod 600 ~/nextcloud/nginx/ssl/*.pem
ls -la ~/nextcloud/nginx/ssl/
# ผลที่ควรได้:

-rw------- fullchain.pem
-rw------- privkey.pem
```
## รวมไฟล์ใน Linux
```
# copy และ rename ให้ตรง
cp /path/to/your.crt ~/nextcloud/nginx/ssl/fullchain.pem
cp /path/to/your.key ~/nextcloud/nginx/ssl/privkey.pem

# รวม Certificate + Intermediate
cat certificate.crt Intermediate.crt > ~/nextcloud/nginx/ssl/fullchain.pem

# Copy Private Key
cp privateKey.key ~/nextcloud/nginx/ssl/privkey.pem

# ล็อค Permission
chmod 600 ~/nextcloud/nginx/ssl/*.pem
ls -la ~/nextcloud/nginx/ssl/
ผลที่ควรได้:
-rw------- fullchain.pem
-rw------- privkey.pem
```

### Step 2: เปิดใช้ SSL Config
```
ls nano ~/nextcloud/nginx/
docker exec nextcloud_nginx rm /etc/nginx/conf.d/default.conf
แก้ domain ใน ssl.conf:
nano ~/nextcloud/nginx/ssl.conf 
ตรวจสอบบรรทัด server_name ให้ตรงกับ domain จริง:
server_name nextcloud.dms.go.th;  ← แก้ให้ตรง
กรณี domain ไม่ได้ต่อให้เข้าผ่าน ip จริง
server_name dmbcloud.dms.go.th 159.138.255.6;
```

### Step 3: แก้ docker-compose.yml
```
nano ~/nextcloud/docker-compose.yml
เพิ่ม mount ssl.conf เข้าไปใน nginx volumes:
nginx:
    volumes:
      - ./nginx/ssl.conf:/etc/nginx/conf.d/ssl.conf
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - /data/nextcloud/files:/var/www/html:ro
```

### Step 4: แก้ Nextcloud Config
```
# แก้ให้ชี้ domain และ https
docker exec -u www-data nextcloud_app php occ config:system:set overwrite.cli.url --value="https://nextcloud.dms.go.th"
docker exec -u www-data nextcloud_app php occ config:system:set overwritehost --value="nextcloud.dms.go.th"
docker exec -u www-data nextcloud_app php occ config:system:set overwriteprotocol --value="https"
docker exec -u www-data nextcloud_app php occ config:system:set trusted_domains 0 --value="nextcloud.dms.go.th"
# ReCheck เข้า container
docker exec -it nextcloud_app bash

# เปิดไฟล์
apt update && apt install nano -y
# path จริงของ Nextcloud ใน Docker
nano /var/www/html/config/config.php

แก้ใน .env ด้วย:
nano ~/nextcloud/.env
NEXTCLOUD_DOMAIN=nextcloud.dms.go.th
OVERWRITEPROTOCOL=https
OVERWRITEHOST=nextcloud.dms.go.th
```

### Step 5: แก้ docker-compose.yml environment
```
nano ~/nextcloud/docker-compose.yml
environment:
      - OVERWRITEHOST=${NEXTCLOUD_DOMAIN}
      - OVERWRITEPROTOCOL=https
      - NEXTCLOUD_TRUSTED_DOMAINS=${NEXTCLOUD_DOMAIN}
```

### Step 6: Reload nginx และ Restart
```
cd ~/nextcloud

# reload nginx ก่อน (ไม่ต้อง down)
docker compose exec nginx nginx -t          # ทดสอบ config
docker compose exec nginx nginx -s reload   # reload

# ถ้า config ผิดค่อย down แล้ว up ใหม่
docker compose down
docker compose up -d
```
### Step 7: ทดสอบ
```
ถ้าเกินปิด port 80 เลยให้เข้าได้แต่ 443 ได้ไหม
**# ทดสอบ SSL
curl -I https://nextcloud.dms.go.th

# ทดสอบ redirect HTTP → HTTPS
curl -I http://dmbcloud.dms.go.th
curl -I https://dmbcloud.dms.go.th
หรือเปิด browser: https://dmbcloud.dms.go.th
ผลที่ควรได้:

http:// → redirect 301 ไป https:// อัตโนมัติ
https:// → 200 OK ไม่มี browser warning
# ลบ default.conf ออกจาก container
docker exec nextcloud_nginx rm /etc/nginx/conf.d/default.conf

# reload
docker compose exec nginx nginx -s reload

# ทดสอบ
curl -I http://dmbcloud.dms.go.th


ผลที่ควรได้:
HTTP/1.1 301 Moved Permanently
Location: https://dmbcloud.dms.go.th

HTTP/2 200
```
### Web Application Firewall
```
# สถานการณ์ตอนนี้
dmbcloud.dms.go.th → A Record → 159.138.255.6 (IP ตรง)
# Huawei WAF ต้องการให้เปลี่ยนเป็น:
dmbcloud.dms.go.th → CNAME → f78169e8xxxxxxxxxxc12c41a042.vip1.huaweicloudwaf.com
                                        ↓
                              159.138.255.6 (ผ่าน WAF)
# ขั้นตอนเปลี่ยนเป็น CNAME เพื่อใช้ WAF
Step 1: Backup DNS Record เดิมก่อน
จดไว้ก่อนลบ: dmbcloud.dms.go.th  A  159.138.255.6
Step 2: ลบ A Record เดิม 
เข้า DNS Management ของ dms.go.th แล้ว: หา record dmbcloud ที่เป็น Type A ลบออก
Step 3: เพิ่ม CNAME Record ใหม่
Field = ค่า
Name = dmbcloud
Type = CNAME
Value = f78169e8xxxxxxxxxxc12c41a042.vip1.huaweicloudwaf.com
TTL = 300
Step 4: ตั้งค่า WAF บน Huawei Console
เข้า Huawei Cloud Console → WAF
เพิ่ม domain dmbcloud.dms.go.th
ตั้ง Origin Server → 159.138.255.6
เปิด HTTPS → upload cert fullchain.pem และ privkey.pem.
Step 5: แก้ nginx ให้รับ traffic จาก WAF
nano ~/nextcloud/nginx/ssl.conf
```
```
server {
    listen 80;
    server_name _;
    
    # รับเฉพาะจาก WAF เท่านั้น
    allow 103.252.24.0/24;    # Huawei WAF IP range
    deny all;
    
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name dmbcloud.dms.go.th;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # รับ real IP จาก WAF
    set_real_ip_from 103.252.24.0/24;
    real_ip_header X-Forwarded-For;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;

    client_max_body_size 10G;
    proxy_read_timeout 3600;

    location / {
        proxy_pass http://nextcloud_app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }
    location /.well-known/caldav {
        return 301 $scheme://$host/remote.php/dav;
    }
}
```
```
Step 6: ปิด port 443 จาก public เปิดเฉพาะ WAF
บน Huawei Security Group:
Port Source               หมายเหตุ
443  Huawei WAF IP range แทน 0.0.0.0/0
80   Huawei WAF IP range แทน 0.0.0.0/0
717  122.155.142.197/32  SSH admin เท่านั้น

Step 7: ตรวจสอบ
# รอ DNS propagate 5-30 นาที แล้วเช็ค
nslookup dmbcloud.dms.go.th

# ผลที่ควรได้
# dmbcloud.dms.go.th → CNAME → f78169e839d94ceab0981dc12c41a042.vip1.huaweicloudwaf.com

# ทดสอบ
curl -I https://dmbcloud.dms.go.th
```
⚠️ ก่อนลบ A Record ต้องแน่ใจว่า WAF ตั้งค่าชี้มาที่ 159.138.255.6 แล้วครับ ไม่งั้น site จะ down ระหว่าง propagate
